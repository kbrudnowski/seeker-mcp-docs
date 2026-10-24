# Synthetic sample data

Everything in this folder is invented. The companies do not exist, the filings were never filed, the figures were typed by hand, and the portfolio belongs to nobody. The files exist to show the shapes the system works with, so that the transcript in `examples/` and the demo storyboard can be read without any real data.

Security ids use the prefix `SYN:` so they cannot collide with the `US:` ids the real system uses.

| File | What it shows |
|---|---|
| `companies.json` | Three synthetic companies as the security master would hold them |
| `evidence.jsonl` | Numbered quotable passages as `open_analysis` returns them, one per line |
| `portfolio.csv` | A broker-style holdings export, the input to `seeker portfolio import` |
| `forecast_ledger.csv` | Dated claims with probabilities and, where due, outcomes and Brier scores |
| `quant_score_response.json` | What the quant service returns for the three names at a cutoff, including one with partial coverage |
