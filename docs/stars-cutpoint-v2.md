# Stars cut-point model v2 (#138)

The Stars forecast needs next year's cut points before CMS publishes them. v1 reclustered simulated
contract scores with CMS's method and added back the data-year calibration offset. It tied, and
mostly lost to, simply reusing last year's published cut points ("persistence"). v2 is the default
from vantage-core 0.3.0 for every module that reads Stars draws (Stars Outlook, $ per measure
point, Studio and Program ROI lifts, settlement).

## What the data showed

CMS caps how far a measure's cut point may move in one year: the guardrail, 5 points on a 0–100
scale (5% of the score range otherwise), for measures with cut points in each of the prior three
star years. The `stars_2026` policy set said guardrails were removed from 2026. The published cut
points say otherwise:

| Star year | Non-CAHPS thresholds | Moved exactly 5 points | Moved more than 5 |
|---|---|---|---|
| 2020–2022 (no guardrail) | 388 | 15 | 73 |
| 2023 | 124 | 20 | 2 |
| 2024 | 124 | 38 | 3 |
| 2025 | 136 | 33 | 3 |
| 2026 | 136 | 25 | 5 |
| 2027 | 136 | 26 | 6 |

Every move over 5 points since 2023 is a measure with fewer than three prior years (CBP, MPF, FMC,
TRC, KED) or a respecified one (COL in 2023 and 2027, MTM in 2023). `stars_2026` now has the
guardrail on, with that evidence in its note, and `stars.cutpoints.guardrail_min_years: 3` is a
policy parameter.

## The model

For each measure and threshold (stars 2–5), per draw:

```
pressure  = recluster(data-year scores, no cap) - recluster(data-year scores, capped)   # 0 where the cap did not bind
move      = drawn industry shift + pressure
move      = clip(move, -cap, +cap)          # only for measures the guardrail covers
cut       = round(published data-year cut + move, published precision)
```

- **Guardrail pressure** is how far the cap held each threshold back in the data year. When the
  data-year scores, clustered without the cap, sit far from the published cut point, the next
  year's cut point keeps moving toward them by up to the cap (Call Center, appeals, care for older
  adults, HbA1c control). Our clustering and CMS's share the same method gap on both sides, so it
  cancels.
- **Industry shift** is the same draw that moves contract scores (the calibrated `stars_dynamics`
  posterior), so cut points and scores stay coherent within a draw.
- **CAHPS** cut points (relative distribution, no guardrail) move by the shift.
- The point forecast is the median of the draws (`output.stars_forecast_cutpoint.cut_p50`); the
  band is P10–P90.

`cut_model: "v1"` on a forecast spec (MCP `stars_forecast`, CLI `--cut-model v1`) runs the old model
for comparison.

## Held-out back-test

Each star year is forecast from data published through the year before. The year being forecast is
in no estimate (drift, regression, posteriors, pressure). Error is the mean absolute error over the
four thresholds, in the measure's units. See `scripts/stars_backtest_note.py` and the reissued
note `docs/notes/stars-backtest-note.html`.

Calibrated release (the default), seed 42, 100 draws, 40 measures a year:

| Star year | Median error, v2 | v1 | Persistence | Mean error, v2 | Persistence | v2 vs persistence (W / T / L) | P10–P90 coverage |
|---|---|---|---|---|---|---|---|
| 2025 | **1.00** | 2.00 | 1.75 | **1.69** | 2.33 | 12 / 23 / 5 | 81% |
| 2026 | **1.25** | 1.62 | 1.75 | **1.61** | 2.04 | 11 / 20 / 9 | 81% |
| 2027 | **0.75** | 1.25 | 1.25 | **1.42** | 1.89 | 13 / 22 / 5 | 80% |

v2's median error is 1.00 against 1.58 for persistence averaged over the three years, lower in every
year, and its mean error is about 23% lower. It beats persistence on 36 measure-years and loses on 19;
the 65 ties are nearly all measures where v2 kept last year's cut points (no guardrail pressure, little
drift), so both forecasts are the same. So the issue's bar,
"beats persistence on a majority of measures", is met among measures where the two forecasts differ
(36 of 55), not among all measures, because of those ties. The P10–P90 band covers 80% of published
thresholds, which is what an 80% band should do. Rating accuracy is unchanged within noise (91% within
half a star; Brier for 4+ 0.131 / 0.162 / 0.178 against persistence's 0.171 / 0.189 / 0.227).

## Limits

- Specification changes that lift the guardrail for a measure (COL in 2027) are not modeled, and
  neither is the early-year climb of new measures (KED, FMC). Both are the largest misses.
- Ties are common: for CAHPS and stable measures v2 usually keeps last year's whole-point cut points,
  the same forecast as persistence.
- Re-run before the 2028 Star Ratings (October 2027) to publish a forecast in advance of the release.
