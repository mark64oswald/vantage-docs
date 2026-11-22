# Calibration (E-11)

Every weight Vantage uses starts as a stated **literature prior**. Observed evidence updates it
with closed-form Bayesian rules, and every run records which **weights release** it used, so a
number in a brief can always be traced to the priors and evidence behind it.

```
reference data / library ──► priors ──┐
                                      ├─► conjugate update ──► posteriors ──► weights release (wr-…)
public CMS data, outcomes feed ──► evidence ┘     (min-evidence floor,                 │
                                                  shrinkage, 95% evidence cap)          ▼
                                                                         run.run.weights_version
```

## Releases

| Release | What it holds |
|---|---|
| `literature_prior` | every parameter at its stated prior; no evidence applied |
| `calibrated` | the same priors updated with the evidence in `calibration.evidence` |
| (none) / `empirical` | the engine before calibration: each Stars measure's raw history, no shrinkage |

A release id is a content hash over its status, method, parameters and the effect-weight library,
so changed evidence makes a new release. `vantage calibrate` builds both releases, runs the
held-out back-test and sets the default (below). `vantage releases` lists them.

Runs pin a release with `weights_version` on the spec (REST bodies, MCP tools and the CLI accept
it): `calibrated`, `prior`, `empirical` or a `wr-` id. Left out, the default release applies. The
run receipt (`get_run`, Console run page, Brief memo) shows `weights_label`: calibrated,
literature prior or uncalibrated.

## Families

| Family | Parameters | Prior | Evidence today |
|---|---|---|---|
| `stars_dynamics` | per Stars measure: industry shift, shift SD, year-over-year slope, residual noise | Normal(0, 0.10·spread), scaled-inv-χ²(0.25·spread, ν=2), Normal(0.80, 0.15), Normal(0.60, 0.15) | public CMS Star Ratings measure data, one observation per star year |
| `effect` | each effect weight in `reference.effect_weights` | Normal with the triangular (low, base, high) mean and SD | program outcomes from the outcomes feed (#151) |
| `radv_support` | support rate per HCC family | the Beta priors in `reference.support_prior` | audit results from the outcomes feed (#151) |

Rules (`vantage/calibration/bayes.py`): at least 3 yearly observations or 11 charts before a
parameter moves; evidence never takes more than 95% of the weight; posteriors are closed-form and
reproducible from their inputs. Carry-forward years (CMS reused prior-year data, r² > 0.99) are
not evidence.

## Held-out years

A forecast from data year D sees only star years ≤ D, in both the raw-history engine and the
calibrated posteriors. Before this release the history used every loaded year, including the year
being forecast, so the earlier published back-test was slightly optimistic. That is fixed.

## Result on public data (seed 42, 100 draws, all rated contracts)

Mean over held-out star years 2025, 2026 and 2027 (forecast from the year before):

| Metric | Literature prior | Raw history | Calibrated |
|---|---|---|---|
| Overall rating within half a star | 91.1% | 90.9% | 90.8% |
| Brier score, P(4+ stars) | 0.158 | 0.157 | **0.156** |
| Mean absolute error (stars) | 0.334 | 0.336 | 0.335 |
| Actual inside P10–P90 | 70.5% | 72.7% | 71.8% |
| Cut-point error, median over measures (score points) | 1.83 | 1.89 | **1.73** |

Before and after: evidence moved 156 of the 172 Stars-dynamics parameters. The 80% bands for
year-over-year slope and residual noise narrowed to about a quarter of the prior width, and the
industry-shift band to about 60% of it.

Rating accuracy is unchanged within noise; the gain is in the cut-point forecast (8% lower error
than raw history) and a slightly better-calibrated P(4+). Calibrated is the default because it
meets the rule: lower cut-point error, and within-half and Brier no worse than raw history beyond
sampling noise (1 point, 0.002).
