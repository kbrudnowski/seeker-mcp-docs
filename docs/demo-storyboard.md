# Two-minute demo storyboard

Screen recording, no voice-over required; captions carry the explanation. Synthetic company throughout. Target 120 seconds. Times are cumulative.

| Time | Screen | Caption | What the viewer sees |
|---|---|---|---|
| 0:00 | Terminal: `docker compose ps` | Six containers. Postgres, object store, the seeker API, the quant service. | Everything healthy; `/ready` answers 200 after a `curl`. |
| 0:10 | Terminal: `curl /ready` with the model key unset, then set | The server refuses to be ready without the classifier that stops price talk. | A 503 naming the missing setting, then a 200. Eight seconds. |
| 0:18 | MCP client (Claude Desktop or Code): tool list | Seven tools. Three of them only on stdio, because they ingest, fetch or write. | The tool panel, briefly. |
| 0:25 | `open_analysis` on the synthetic ticker, cutoff 2026-06-30 | Evidence is loaded to a cutoff. Nothing filed after it exists. | The numbered passages E1 to E8 scroll past; the `analysis_id` is visible. Keep the response on screen four seconds. |
| 0:40 | Claude drafts; `submit_forecast` with a cited key that was never shown | First submission. | The refusal: the key was not shown, the claim was dropped, the driver has no citation. Hold three seconds. |
| 0:50 | Claude repairs and resubmits | Second submission. The handle stayed open; a citation error is repairable, price talk is not. | `Stored as …`, then the forecast JSON with spans resolved to byte offsets and exact quotes. Zoom on one span. |
| 1:05 | Claude is asked to add "so the stock should re-rate" to the counter-view and resubmit | What happens when the draft recommends. | Refusal naming the field and the rule; the handle is closed. Hold three seconds. |
| 1:15 | `list_portfolios`, then `review_portfolio` for seven days | A book, reviewed. What moved, what is due, and whether three holdings are really three bets. | The digest; cursor pauses on `thesis_stage_moves`, `catalysts_due`, `clusters`. |
| 1:30 | Terminal: `psql` running the p50/p95 query over `llm_calls` and `tool_calls` | Every call wrote tokens, cost and latency. This is where the numbers come from. | The query result, a dozen rows. |
| 1:42 | README results table | Measured, dated, with n. | Scroll the table slowly; end on it. |
| 1:55 | Still on the table | Source is private; the docs, the schemas and the eval method are not. Link in the description. | Hold to 2:00. |

Preparation before recording:

- Import `data/synthetic/portfolio.csv` into a fresh tenant so the book shown is the synthetic one.
- Seed the synthetic company's documents (the eight passages) into a scratch store, or record against a real public-domain company and blur the name; the first is cleaner.
- Have the three drafts ready in the client so the typing does not eat the clock: the bad citation, the repair, the price-talk attempt.
- Set `MCP_ANALYSES_PER_WEEK` high enough that the demo's opens are not refused by the question cap.
- Run the SQL once before recording so Postgres has the plan cached and the result appears at once.

Why this order: the refusal and repair is the point of the system, so it sits in the middle where attention is highest; the price-talk refusal comes right after so the viewer sees the two kinds of refusal side by side; the review shows the read-only surface; the SQL and the table close the loop from "it did something" to "here is what it cost and how long it took".
