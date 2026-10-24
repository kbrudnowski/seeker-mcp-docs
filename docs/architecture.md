# Architecture walkthrough

This is the longer version of the diagram in the README. It follows one request through the system, then describes the pieces the request touched.

## One request, end to end

A client connected over MCP calls `open_analysis` with a ticker and a cutoff date.

1. **Budget check.** Before anything is fetched, the server counts this week's questions and this week's model spend. Either cap exhausted is a refusal that says so and does nothing else.
2. **Ingest to the cutoff.** The security master resolves the reference to one company. A provider fetches the filings published on or before the cutoff that the store does not yet hold; a provider cannot return anything published after `before`, which is a mandatory argument on every provider method. Raw bytes go to the object store, write-once. Each new document is normalised once and stored with the normalisation every later byte offset is measured against. A cheap-tier model reads each new document once and emits facts, reconciled against the XBRL figures, and events. This is the only place the pipeline spends money on the server side for this tool.
3. **Evidence hash.** The set of visible documents and facts is hashed. The profile and forecast caches are keyed by that hash, so a second open over unchanged evidence costs no model call and a changed store produces a different key rather than a stale answer.
4. **Snippet cut.** Up to 12 documents and 120 quotable passages are selected, each at most 200 characters, and numbered. A second digest is taken over the ordered cut. Both digests are kept on an in-memory handle with the company id and the cutoff.
5. **Hand-over.** The client receives an `analysis_id`, the prompt blocks the server's own deep-tier node would have used, and the numbered passages. The client's model does the reasoning. The server made no deep-tier call.

The client calls `submit_forecast` with the handle and a draft.

6. **Handle and digest.** The handle must exist. The evidence and the cut are recomputed; if the store gained a document since the open, the keys no longer name the same sentences and the client is told to reopen.
7. **Shape.** The draft is a Pydantic model: exactly one bull, base and bear scenario, probabilities summing to one, confidence in `[0, 1]`, a counter-view, and resolution criteria with a kind, a target and a date. A schema failure is rejected before any model is consulted.
8. **Citations.** Each cited key is resolved to a document id and a byte range. A key the open did not show is dropped with its claim. The quote at that range must match the stored text exactly.
9. **Criteria.** A `macro_series` target must be a known series id, a `filing_field` a known XBRL concept, a `transcript_statement` a regex that compiles, is under 200 characters and cannot match everything. Prose kinds (`event`, `manual`) go to a human queue when they come due.
10. **Price talk.** Every visible string, including each scenario narrative, each driver, each assumption and each prose criterion target, passes the rules filter and then the classifier. A judged recommendation closes the handle. A classifier that did not answer, because of a timeout or a transport error, is reported as such, and the same draft can be resubmitted unchanged.
11. **Store.** What survives is written to the forecast cache under the evidence hash. The cache has no delete; a second answer over the same evidence is checked and discarded, and the client is told which row stands.

## Components

### MCP server

Built on the official Python SDK. The server registers all seven tools and removes the ones a transport is not allowed to serve, so an allow-list naming a tool that does not exist is a startup error rather than a silently smaller server. Stdio serves all seven; streamable HTTP serves the four that neither ingest, nor fetch arbitrary URLs, nor write to a tenant-less cache. HTTP is stateless JSON, with a DNS-rebinding guard on the `Host` and `Origin` headers. The server refuses to start on either transport if the price-talk classifier has no model, because the forecast cache has no delete and a rules-only write would be served for ever.

### REST and SSE API

FastAPI, eleven routers under `/v1`, each closed by a bearer dependency so a new route is authenticated by construction. A dossier run streams over SSE with sequence ids; a dropped stream reconnects with `Last-Event-ID` and replays from the broker's backlog or, failing that, from the graph checkpoints. A long run writes a heartbeat every sixty seconds; thirty missed beats mark it abandoned, which is how a killed driver is told apart from a slow one.

### Graphs

Four LangGraph graphs compile against one Postgres checkpointer. Nodes are pure async functions of `(state, deps)`; a layering contract enforced by import-linter forbids them from importing FastAPI or a database session.

- **dossier**: ingest → quant score → screen → profile → environment forecast → analyst review → bull and bear in parallel → judge → persist. The screen can stop the run before the deep tier is paid for.
- **sweep**: load world model → scan deterministic signals → propose theme changes → build cited causal chains per theme → map and verify exposures → price check → update world → interrupt for a person → emit candidates. The MCP tools `list_candidates` and `approve_candidates` are the two sides of that interrupt.
- **coherence**: cluster a set of theses by theme overlap and feature cosine, name the clusters, find shared falsifiers. `review_portfolio` runs it per review.
- **monitor**: load a thesis, fetch new evidence, check falsifiers, propose a stage change, commit through a fixed rule table, persist an event.

### Model tiers

Three tiers, each a vendor model family and a token ceiling, overridable by environment. The cheap tier does extraction, verification, cluster naming and the price-talk classifier. The mid tier does the company profile, the bull and bear cases and stage proposals. The deep tier does forecasts, analyst reviews, the judge, theme deltas and causal chains. Transports exist for the Anthropic Messages API and any OpenAI-compatible endpoint, with retries on 408, 429 and 5xx, a per-host request-per-minute bucket, and a price table that raises on an unknown model rather than billing zero.

The model never function-calls. Graph nodes call a six-tool belt themselves (web search, page fetch, EDGAR full-text search, procurement, news volume, short interest), each bounded by the run's cutoff and a per-node call cap, and each call writes a `tool_calls` row.

### Validators

- `SpanValidator`: the cited document is in the evidence, the range is inside the text, the quote equals the slice.
- `CitationValidator`: a claim with no span or an unknown document id is rejected.
- `CriterionValidator`: targets resolve to something a program can check; regexes are safe.
- `NoPriceTalkValidator`: 26 rules over text that has been Unicode-folded, look-alike-mapped and de-spaced, followed by the classifier. The rules can only add refusals the classifier would not have made; the classifier is what the gate rests on.
- Structured output: one repair attempt, then a result flagged partial rather than one that looks complete.

### Storage

Postgres holds about 35 tables across 15 migrations. Three are append-only by trigger: `thesis_events`, `journal`, and `forecast_ledger`, where exactly one update is allowed, the resolution. Three caches are keyed by evidence hash. `llm_calls` and `tool_calls` carry tokens, cost and latency per call with the run and node that made it. The object store holds raw bytes only, write-once, under a key that includes the content hash. A `quant_ro` role can SELECT four point-in-time tables (`facts`, `prices`, `index_membership`, `corporate_actions`) and nothing else, with no default privileges, so a table added later stays unreadable until granted by name.

### Providers

Thirteen providers behind one base class. Each declares a source kind, the exchanges it covers and a licence from a fixed vocabulary (`public_domain`, `own`, `free_noncommercial`, `commercial_internal`, `commercial_display`). Only the first, second and last may be shown to a user. A `relicense` command reconciles stored documents when a vendor plan changes: unchanged text moves licence in place and keeps every span; changed text becomes a new version. `SEEKER_OFFLINE` closes every socket and answers from recordings, which is how provider health runs with no keys in CI.

### Quant service

A separate FastAPI service in its own repository, reached over HTTP with a bearer token. It serves factor scores and percentiles for named securities or a whole universe at a cutoff, the feature panel and its schema, a model registry with measured reports, point-in-time index membership, enumerated data-quality flags, and descriptive portfolio analytics (concentration, exposures, risk contribution). It returns numbers and enumerated flags only; a contract test walks the response schemas and fails on any field named like advice. An unknown security is never a 404 on the scoring routes, because a 404 cannot tell a typo from a gap in coverage; it comes back with `coverage_ok=false` and a reason. The OpenAPI document is committed and CI fails when it drifts. The service image installs only the serving dependencies; a test fails if the research stack leaks in.

### Contracts

One Pydantic package, `frozen` and `extra="forbid"`, shared by both services and released by git tag. Forty-one JSON Schemas are generated from it and checked for drift. A weekly job installs the newest tag against the quant service's contract tests, so a pin can move with evidence.

## Point in time, everywhere

Every stored fact carries the date the world could have known it, separate from the date it happened. Every provider takes `before` and cannot return anything later. Every graph run has `as_of`. Macro series come from real-time vintages, so a revised figure returns the value known then. Index membership is served by effective date. The quant service applies the same cutoffs in SQL when it reads the seeker's tables directly. Without this, every historical evaluation would be fiction, and the forecast ledger would be grading claims against evidence the model could not have had.
