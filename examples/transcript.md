# Example: Claude calling the tools

Synthetic throughout. The company, the passages and the portfolio are from [data/synthetic](../data/synthetic/README.md). Tool inputs and outputs follow the real schemas; the text the server returns is abridged where marked. The exchange shows the one thing the transcript is for: the server refuses a draft that cites a key it did not show, the model repairs it, and the second draft is stored.

---

**User:** What does the environment look like for Northwind Grid over the next year? Use the seeker.

**Claude:** I'll open an analysis at today's cutoff and work from the passages it returns.

```
tool: open_analysis
input: {"ref": "NWGC", "as_of": "2026-06-30", "depth": "quick"}
```

```
analysis_id: 01K8SYN0000000000000000001
SYN:NWGC as of 2026-06-30T23:59:59-04:00, 8 quotable passages

## System
[abridged: the forecasting role, the rule that nothing may discuss the share
price, the three-scenario shape, and that every claim must cite a key below]

## Context
Company: Northwind Grid Components Inc. (SYN:NWGC). Electrical equipment.
Segments from the latest 10-K: utility transformers and switchgear (68% of
revenue), industrial controls (32%).

## Quotable evidence
E1 [10-K 2026-02-27, Item 1] "Transformer and switchgear sales to utility
   customers were 68% of revenue in fiscal 2025, up from 61% ..."
E2 [10-K 2026-02-27, Item 7] "Backlog at year end was $412 million, of which
   approximately 70% is scheduled for delivery within twelve months."
E3 [10-K 2026-02-27, Item 1A] "We source grain-oriented electrical steel from
   three suppliers, two of which are outside the United States ..."
E4 [10-Q 2026-05-08, Item 2] "Gross margin was 27.4% compared with 25.9% ..."
E5 [call 2026-05-08] "We expect lead times on large power transformers to
   remain above 80 weeks through the end of the calendar year."
E6 [call 2026-05-08] "The second plant line is on schedule for commissioning
   in the fourth quarter; we have not yet booked orders against that capacity."
E7 [8-K 2026-06-15] "... five-year supply agreement with a regional
   transmission operator for distribution transformers, with minimum annual
   volumes."
E8 [10-Q 2026-05-08, Note 9] "Two customers each accounted for more than 10%
   of revenue in the quarter."

## Task
Forecast the environment this company operates in over the next 12 months ...

TRANSPORT: reply with submit_forecast and this analysis_id. Cite only the E
keys above. A key not shown here is dropped together with its claim.
```

**Claude:** The evidence supports a demand driver (utility hardening, E1, E2, E7), a capacity driver (E5, E6) and two risks (steel supply, E3; customer concentration, E8). I'll draft the forecast.

```
tool: submit_forecast
input: {
  "analysis_id": "01K8SYN0000000000000000001",
  "draft": {
    "drivers": [
      {
        "driver": "Utility grid-hardening programmes keep transformer demand above the company's delivery capacity",
        "direction": "up",
        "horizon_months": 12,
        "confidence": 0.7,
        "evidence": ["E1", "E2", "E5"],
        "resolution_criterion": {
          "check_kind": "filing_field",
          "target": "RevenueFromContractWithCustomerExcludingAssessedTax",
          "comparator": "direction_up",
          "resolves_at": "2027-02-28"
        }
      },
      {
        "driver": "Second plant line is commissioned and starts taking orders",
        "direction": "up",
        "horizon_months": 6,
        "confidence": 0.8,
        "evidence": ["E6", "E9"],
        "resolution_criterion": {
          "check_kind": "transcript_statement",
          "target": "commission(ed|ing).{0,40}(second|new) line",
          "comparator": "exists",
          "resolves_at": "2026-12-31"
        }
      }
    ],
    "scenarios": [
      {
        "name": "bull", "probability": 0.3,
        "narrative": "Backlog converts on schedule and the new line is sold out within two quarters of commissioning; pricing holds.",
        "demand_delta_pct": [12, 20], "margin_pressure": "easing",
        "key_assumptions": ["No steel allocation event", "Second line commissioned in Q4"],
        "horizon_months": 12,
        "resolution_criterion": {"check_kind": "filing_field", "target": "GrossProfit", "comparator": ">=", "threshold": 0.29, "resolves_at": "2027-02-28"}
      },
      {
        "name": "base", "probability": 0.5,
        "narrative": "Deliveries track the twelve-month backlog schedule; the new line adds capacity but orders against it arrive gradually.",
        "demand_delta_pct": [6, 12], "margin_pressure": "flat",
        "key_assumptions": ["Lead times stay above 80 weeks"],
        "horizon_months": 12,
        "resolution_criterion": {"check_kind": "filing_field", "target": "GrossProfit", "comparator": ">=", "threshold": 0.27, "resolves_at": "2027-02-28"}
      },
      {
        "name": "bear", "probability": 0.2,
        "narrative": "A tariff or allocation change on electrical steel delays deliveries and the two large customers defer.",
        "demand_delta_pct": [-5, 3], "margin_pressure": "tightening",
        "key_assumptions": ["Steel supply disruption"],
        "horizon_months": 12,
        "resolution_criterion": {"check_kind": "event", "target": "tariff or allocation notice affecting grain-oriented electrical steel", "comparator": "exists", "resolves_at": "2026-12-31"}
      }
    ],
    "counter_view": "Backlog is a lagging indicator; if utility budgets were cut the company would still report a full year of deliveries before new orders showed the change.",
    "counter_view_evidence": ["E2"],
    "external_risks": ["Steel supply concentration outside the United States", "Two customers above 10% of revenue"],
    "bottlenecks_owned": ["Large power transformer capacity until the second line is commissioned"],
    "bottlenecks_faced": ["Grain-oriented electrical steel supply"]
  }
}
```

```
error: the forecast was refused and nothing was stored
- drivers[1].evidence: E9 was not among the keys this analysis showed; the
  claim it supported was dropped
- drivers[1]: no evidence survived, so the driver has no citation
```

**Claude:** I cited a key that does not exist. The commissioning claim rests on E6 alone, so I'll cite that and resubmit; the handle is still open because the refusal was a citation problem, not price talk.

```
tool: submit_forecast
input: { "analysis_id": "01K8SYN0000000000000000001",
         "draft": { ...same, with drivers[1].evidence: ["E6"] } }
```

```
Stored as 01K8SYN0000000000000000002.
{
  "id": "01K8SYN0000000000000000002",
  "security_id": "SYN:NWGC",
  "as_of": "2026-06-30T23:59:59-04:00",
  "evidence_hash": "3f1c…9a",
  "drivers": [
    {
      "driver": "Utility grid-hardening programmes keep transformer demand above the company's delivery capacity",
      "direction": "up", "horizon_months": 12, "confidence": 0.7,
      "spans": [
        {"document_id": "syn-doc-10k-2025", "start": 1184, "end": 1361, "quote": "Transformer and switchgear sales to utility customers were 68% ..."},
        {"document_id": "syn-doc-10k-2025", "start": 20417, "end": 20532, "quote": "Backlog at year end was $412 million ..."},
        {"document_id": "syn-doc-call-q1-2026", "start": 3302, "end": 3410, "quote": "We expect lead times on large power transformers ..."}
      ],
      "resolution_criterion": {"check_kind": "filing_field", "target": "RevenueFromContractWithCustomerExcludingAssessedTax", "comparator": "direction_up", "threshold": null, "resolves_at": "2027-02-28"}
    },
    { "driver": "Second plant line is commissioned and starts taking orders", "...": "spans resolved from E6" }
  ],
  "scenarios": [ "...as submitted" ],
  "counter_view": "Backlog is a lagging indicator; ...",
  "spans": [ {"document_id": "syn-doc-10k-2025", "start": 20417, "end": 20532, "quote": "Backlog at year end was $412 million ..."} ]
}
```

Note what the server did: it resolved `E1`, `E2`, `E5`, `E6` to byte offsets in the stored documents and attached the exact quotes; it filled `security_id`, `as_of` and `evidence_hash` itself; and it ran the price-talk classifier over every narrative, driver, assumption and prose target before writing the row.

---

**User:** And how does my book look this week?

```
tool: list_portfolios
input: {}
```

```
01K8SYNPF00000000000000001  synthetic-1  3 holdings
```

```
tool: review_portfolio
input: {"portfolio_id": "01K8SYNPF00000000000000001", "as_of": "2026-06-30", "period_days": 7}
```

```json
{
  "schema_version": 1,
  "portfolio": {"id": "01K8SYNPF00000000000000001", "name": "synthetic-1", "as_of": "2026-06-30"},
  "period": {"start": "2026-06-23", "end": "2026-06-30"},
  "coverage": {
    "holdings": 3, "resolved": 3, "unresolved": 0, "unresolved_share_by_count": "0",
    "value_basis": "cost",
    "by_currency": [{"currency": "USD", "total": "15720.00", "resolved": "15720.00", "unresolved": "0", "unresolved_share": "0", "holdings_without_cost": 0}],
    "holdings_without_cost": 0, "unresolved_holdings": []
  },
  "changes": {
    "thesis_stage_moves": [
      {"thesis_id": "01K8SYNTH0000000000000001", "security_id": "SYN:NWGC", "stage": {"from_stage": "forming", "to_stage": "confirming"}, "rationale": "supply agreement with minimum volumes matched a confirming driver", "ts": "2026-06-15T21:10:00Z"}
    ],
    "catalysts_due": [
      {"catalyst_id": "01K8SYNCAT000000000000001", "security_id": "SYN:VRSL", "kind": "regulatory", "due_at": "2026-06-28", "severity": "high", "handled": false, "triggers": []}
    ],
    "forecasts_resolved": []
  },
  "coherence": {
    "as_of": "2026-06-30",
    "security_ids": ["SYN:NWGC", "SYN:HLCP", "SYN:VRSL"],
    "theme_concentration": {"grid-capex": 0.33},
    "clusters": [
      {"label": "grid capex beneficiaries", "security_ids": ["SYN:NWGC"], "shared_themes": ["grid-capex"], "shared_falsifiers": []}
    ],
    "shared_falsifiers": [],
    "diversifying": ["SYN:HLCP", "SYN:VRSL"],
    "flags": [],
    "features_version": "feat-2026.10.1"
  },
  "flags": []
}
```

**Claude:** One thesis moved from forming to confirming after the supply agreement, one catalyst on Verisol came due two days ago and nobody has handled it, and the three names cluster into three separate bets. The digest reports what moved; it does not say what to do, and nothing I conclude here is written back.
