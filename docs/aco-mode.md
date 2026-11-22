# ACO mode: MSSP and LEAD track choice

ACO mode answers one question for a Medicare ACO: **which track, and what range of shared
savings or losses over the agreement?** It compares the Medicare Shared Savings Program
(BASIC A–E and ENHANCED) and the LEAD model (Professional and Global, from 2027) on the same
seeded futures, so the difference between tracks is the terms, not the luck.

Surfaces: Console **ACO Track Choice** (`#aco`), the Brief starter "Which ACO track: MSSP or
LEAD?", MCP tools `list_aco_options`, `aco_settlement`, `aco_backtest`, REST
`/api/aco/catalog`, `/api/aco/settlement`, `/api/aco/backtest`, and the CLI (`vantage aco`,
`aco-options`, `aco-backtest`, `load-aco`). The back-test is written up in
[notes/aco-backtest-note.html](notes/aco-backtest-note.html).

## 1. Inputs

| Input | Source |
|---|---|
| Public ACO results, PY2013–2024 | data.cms.gov "Performance Year Financial and Quality Results", fetched by `scripts/fetch_aco_results.py` and normalized by `scripts/parse_aco_results.py` into `reference-data/vbc/mssp_aco_results.csv` (`reference.aco_result`). ACO level only. Pre-2018 ACO ids are carried to CMS's 2018 numbering by name, one-to-one only. |
| An ACO | Either a public ACO id (its latest year and history come from the public file) or an ACO profile in `inputs/aco/*.yaml` (`input.aco_profile`). See [loading-datasets.md](loading-datasets.md#aco-profiles-aco-mode). |
| Terms | Term sets in `policy/contracts/` (`mssp_*`, `lead_*`), families `mssp` and `lead`, each term tagged with its basis. |
| Assumptions | FFS trend P10/P50/P90 2.5% / 4.5% / 6.5%; ACPT 4.0%; risk-score growth 0 / 0.5% / 1.5%; quality score SD 6 points; ACO revenue as a share of spending 25% (low revenue) or 60% (high revenue). Overridable per run. |

## 2. The savings-rate forecast

Savings rate SR = (benchmark − expenditure) / benchmark. For each ACO with a prior year:

    SR_t ~ Normal(a + b·SR_(t−1) + c·rebase + d·rebase·SR_(t−1),  s0² + k / N)

fitted by least squares on every ACO year-pair in the three most recent public years
(`FIT_WINDOW = 3`), where `rebase` marks a new agreement period and N is assigned
beneficiaries. ACOs with no prior year use the mean and spread of first-year ACOs in the
window. The same model drives both the run and the back-test, which fits only on years
before the one it forecasts.

A run draws SR for the gap years between the ACO's data year and the first performance
year, then for each year of the horizon. Each year also draws the benchmark update
((1 − w)·FFS trend + w·ACPT, w = `acpt_weight`), risk-score growth against the cap, and a
quality score.

## 3. Settlement rules

Every draw satisfies **ACO result + Medicare result = benchmark − expenditure**.

| Rule | MSSP | LEAD |
|---|---|---|
| Sharing rate | BASIC A–B 40%, C–E 50%, ENHANCED 75% (42 CFR 425.605, 425.610) | Professional 50%, Global 100% (CMS model overview) |
| Savings cap | 10% of benchmark (BASIC), 20% (ENHANCED) | 25% (assumption) |
| Minimum savings / loss rate | One-sided: size-based MSR table (verified against published MSRs). Two-sided: 2% symmetric (the ACO's elective choice; assumption) | None (assumption) |
| Losses | BASIC C–E 30% of losses, limited to 2/4/8% of ACO revenue or 1/2/4% of benchmark; ENHANCED 1 − 0.75 × quality, floor 40%, ceiling 75%, limit 15% of benchmark | Shared at the sharing rate, limit 25% (assumption) |
| Quality | Below the 40th-percentile standard, sharing slides with the score; ENHANCED loss rate rises to 75% | 3% benchmark withhold, earned back in full at score 80 (schedule is an assumption) |
| Benchmark | Risk-score growth capped at 3% | Global: discount 1.75%, rising 0.25 points a year to 3%; higher-spending ACOs get a 1.5% add-on. Lower-spending Global: 3% discount, no add-on |
| Agreement | 5 years | Up to 10 years (2027–2036) |

The size-based MSR knots (assigned beneficiaries, MSR): 5,000 3.9%, 6,000 3.6%, 7,000 3.4%,
8,000 3.2%, 9,000 3.1%, 10,000 3.0%, 15,000 2.7%, 20,000 2.5%, 50,000 2.2%, 60,000+ 2.0%,
interpolated between.

## 4. Outputs

`output.aco_settlement_summary` (per track and year, plus `horizon_year = 0` for the
cumulative result: mean, P10/P50/P90, P(savings), P(loss), Medicare's result, gross savings),
`output.aco_settlement_driver` (each driver at its P10 then P90, ranked by swing) and
`output.aco_settlement_draw`. The back-test writes `output.aco_backtest_forecast` and
`output.aco_backtest_metric` (per year, `performance_year = 0` pooled, plus calibration bins).

## 5. How good is the forecast

Fitted on the three prior years only, 2017–2024 (run in the note's appendix):

| | Model | Last year's rate | Prior years' mean |
|---|---|---|---|
| MAE, all ACOs | 2.62 pts | | 3.39 pts |
| MAE, ACOs with a prior year | 2.62 pts | 2.75 pts | |
| Brier, P(earns shared savings) | 0.174 | base rate 0.241 | |
| Share inside P10–P90 | 85% | | |

Misses: 2020 (COVID) had 68% band coverage and ACOs earned savings far more often than
forecast; in 2023 and 2024 last year's rate was marginally better; and ACOs with a 40–80%
forecast chance of earning savings earned them about 6 points more often than forecast,
because savings rates rose almost every year and a three-year fit lags that drift.

## 6. Limits

- No ACO detail beyond the public file or a profile: no market, attribution, or spending mix.
- LEAD is settled on the ACO's MSSP-basis benchmark with LEAD's discount, withhold and
  add-on; LEAD's own baseline weights, regional efficiency and prior-savings adjustments are
  not modelled. Its MSR, caps, risk-score cap and earn-back schedule are stated assumptions
  until CMS publishes them. The LEAD fact sheet's Global bullet says 50% sharing where the
  model overview says 100%; Vantage uses 100% and flags it.
- ACO REACH is not modelled (it ends in 2026).
- PY2025 public results publish in late summer 2027. Re-fetch with the "Refresh reference
  data" workflow, then `vantage load-aco` and `vantage aco-backtest --rerun`.
