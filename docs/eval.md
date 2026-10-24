# Evaluation

Two layers. An offline harness runs frozen scenarios against a real model and gates the build. An online ledger records what every call cost and how long it took, and grades every dated claim when it resolves. This page says how each works and how the numbers in the README were produced.

## Offline harness

`seeker eval --tier cheap|mid|deep` loads the scenarios registered for that tier and runs them one at a time, with a pause between calls because the cheap tier is rate limited. Each scenario yields cases, a scorer turns the model's answers into a score in `[0, 1]`, and the report compares the score with a baseline recorded by hand from a run somebody looked at.

Gate arithmetic:

- A scenario fails when its score is more than 0.05 below its baseline.
- A scenario with no baseline runs ungated and the report says so.
- Answers lost to HTTP 429, 503 or a timeout count as not answered, not as wrong. If fewer than 80% of cases were answered the scenario is reported as throttled and not scored, so a bad quota day cannot be read as a regression.
- The suite exits non-zero only on a regression.

A guard refuses to build a model transport when the test runner is loaded, so `pytest` cannot reach a model by accident. The eval suite is a separate CI job that runs only when a model key is present as a secret.

| Scenario | Tier | What it measures |
|---|---|---|
| Price-talk classifier | cheap | 22 short passages, half written as recommendations (some with injection framing), half as ordinary filing and call prose with financial vocabulary. Accuracy, refusal rate on recommendations, allowance rate on research. |
| Price-talk hold-out | cheap | A second 22-passage set written independently and kept back, run through both classifiers side by side with the disagreements listed. |
| Exposure trap | cheap | Twenty hand-labelled rows mixing real exposures with plausible companies that have no segment evidence. Accuracy, trap rejection rate, precision, recall. |
| Causal chain | deep | Five frozen themes, each targeting one failure: a present bottleneck, an honest "none", the hop cap, an unsupported common-knowledge hop, the wrong bottleneck kind. A chain is clean when it has one to three hops, every hop cites a source, and the bottleneck question is answered. |
| Analyst grounding | deep | Five synthetic companies. Share of the three claim paragraphs whose citations resolve to documents the model was shown. |
| Agreement stability | deep | The same five cases three times; the agreement label must not change between runs. |
| Ablation | mixed | The dossier with and without a stage (today: the debate), paired on the same evidence, measured on conviction, cost and thesis survival with an exact sign test. |

Honest state of the harness: the price-talk scenarios run against a real model today. The other four and the ablation have their cases and scorers written but no adapter between a frozen case and the graph node it names, so they report as skipped. They are listed because they define what "better" means for each stage, and because the gate arithmetic is the same for all of them.

### Rules-only measurement

The price-talk rules filter is also measured without any model, in a deterministic unit test. Two lists were written: the phrasings the rules were tuned on, which must be caught at 100%, and a held-out list the rules had never seen. The held-out result is asserted exactly so that it cannot drift silently: 17 of 40 recommendations refused, 0 of 40 research sentences refused. The 23 misses are enumerated in the fixture. That number is why the classifier is mandatory and the rules are only allowed to add refusals.

## Online ledger

Every model call writes one row with the run, the graph node, the tier, the model, input tokens (including cache writes), cached tokens, output tokens, the cost in USD at the vendor's list price for that model, and the wall-clock latency around the call. Every tool call writes a row with the node, the tool, an argument hash, whether it was a cache hit, and latency. A run row carries its start and finish and its total cost.

Two endpoints read those rows: a per-run trace that interleaves model calls, tool calls and checkpoints, which is how a run's spend is attributed rather than guessed; and a costs endpoint that groups spend by node, model, day, tool or run against the weekly budget. Percentiles are not computed by an endpoint; they come from SQL over the same rows.

Caveats a reader should hold: latency includes rate-limiter wait and retries, so it is what a caller experiences and not what the vendor took; the price-talk classifier's calls are not recorded, so server-side cost is a floor; a model call that raised writes no row; the run total is accumulated in graph state and can differ from the sum of its rows by rounding.

## Forecast ledger

A forecast is a set of dated claims, each with a probability, a horizon and a resolution criterion. When a claim comes due a resolver checks it: a macro series against its vintage at the horizon, an XBRL concept against the filing, a transcript statement against the call by regex, an event or a manual claim through a human queue. The outcome and a Brier score are written to the ledger, which is append-only except for that one resolution update.

A scorecard endpoint returns the mean Brier score over resolved claims, a ten-bin calibration table (stated probability against observed frequency), and two baselines to beat: uniform 0.25 and persistence. Nothing resolves against the share price, by design.

## How the README numbers were produced

Measured on 2026-10-24 on a single machine running the Compose stack: Postgres 16, MinIO, the seeker API, the quant service. Cheap tier: Gemini Flash-Lite on the free tier. Deep-tier reasoning over MCP was done by the connected client's model. No deep-tier call was made by the server.

1. `seeker eval --tier cheap --json` and `seeker eval price-talk --holdout --json`, outputs saved. Accuracy and rates are read from the JSON.
2. `open_analysis` and `submit_forecast` over MCP on two companies already in the store, each opened at two cutoffs for which no forecast existed, plus `review_portfolio` on one imported book of 28 holdings. The client was a small stdio driver inside the API container that timed each call with a monotonic clock. The drafts were written by the connected model from the passages each open returned. Three of the four were stored on the first submission; the fourth was refused three times on its counter-view and is reported as refused.
3. SQL over `llm_calls` and `tool_calls`, joined to `runs` for the window:

```sql
select r.graph, c.node, c.model, count(*) as calls,
       percentile_cont(0.5)  within group (order by c.latency_ms) as p50_ms,
       percentile_cont(0.95) within group (order by c.latency_ms) as p95_ms,
       avg(c.cost_usd) as usd_per_call, sum(c.cost_usd) as usd
from llm_calls c join runs r on r.id = c.run_id
where c.ts between :since and :until
group by 1, 2, 3 order by usd desc;
```

4. Per-run cost from `runs.cost_usd` for the `mcp_analysis` rows in the window.

The company used for timing is not named here and the forecast content is not reproduced; the synthetic dataset and transcript in this repository show the shapes, not that run.

## What is not measured yet

Forecast accuracy and calibration need claims to come due; the ledger has the columns and the endpoint but the first horizon has not passed. The four skipped scenarios need their adapters. There is no load test; the latency rows are from one process on one machine.
