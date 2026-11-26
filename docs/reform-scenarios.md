# Reform scenarios (#165)

A reform scenario answers one question: *if this bill became law as introduced, what would it do to
my contract?* The first one is `hr9544_reform`, modelled on H.R. 9544, the Saving MEDICARE Act
(Doggett, introduced June 30 2026). It is **hypothetical**. It is built from the sponsor's release
and the bill summary, not from enacted law or final bill text, and every view that shows it carries
the label "Hypothetical: H.R. 9544 as introduced".

## How it fits the policy library

Each engine reads one policy chain: net risk reads the RADV chain (`pre_crush`, `crush_*`), the
bid bridge reads MA payment (`ma_payment_2027`), and QBP and rebates come from the Stars chain
(`stars_*`). A reform bill touches all three, so a reform set is applied as an **overlay** on the
run, not as one more link in a chain:

```
ScenarioSpec.reform_policy_set_id = "hr9544_reform"

PolicySet(store, "crush_proposed_2026", overlay="hr9544_reform")
   = crush_proposed_2026 resolved through its parents
   + hr9544_reform's OWN params and code lists on top (its parents are not applied)
```

* With no overlay, every engine behaves exactly as before. The new parameters are read with
  today's behaviour as the default.
* The overlay is part of the run's policy version, so a reform run has its own fingerprint.
  Baseline runs keep the fingerprints they had before this change.
* A reform set is any policy set that defines `reform.label`. The Console and Brief list those
  sets under "Reform scenario", not among the RADV rules.
* `hr9544_reform` inherits `ma_payment_2027` so that its payment factors resolve when it is read on
  its own. To apply the reform, always pass it as `reform_policy_set_id`, never as the run's policy set.

## What is modelled

| Bill provision (summary) | Parameter | What Vantage does |
|---|---|---|
| Diagnoses from chart reviews or health risk assessments only don't count toward risk scores | code list `excluded_dx_sources`: `in_home_hra`, `chart_review_linked` | **Net risk:** HCC cells from those sources leave the risk score. Their value (V24 coefficient × base rate × 12 per HCC) comes off risk revenue. They also leave the RADV exposure, because nothing is paid on them. The output is the risk-score and revenue change by contract and HCC family (`output.reform_exclusion`). **Settlement:** the group's average risk score falls by the same first-order amount. |
| End the MA Quality Bonus Program | `ma.qbp_enabled: false` | **Bid bridge:** the benchmark uses the rate book's no-bonus column whatever the rating (the new-contract 3.5% column ends too). **Stars value:** the QBP is $0, and P(quality bonus) is 0. **Studio and Program ROI:** the quality-bonus domain is $0, including the premium a percent-of-premium contract pays the group. The **rebate share still follows the Star Rating**, because the summary does not change it. |
| Prohibit percent-of-premium provider contracts that reward adding codes | `ma.pct_premium_coding_ban: true` | **Settlement and Program ROI:** `ma_pct_premium` term sets are flagged as non-compliant under the scenario. This is a flag only: the settlement still shows today's terms (with the reform's lower risk scores and no QBP). Vantage does not model what the contract would turn into. |

### Assumptions (basis: assumption)

* **"Only" support.** The aggregate input counts each HCC once, under the diagnosis source that
  supports it (`hcc_support.dx_source`). Dropping the `in_home_hra` and `chart_review_linked` cells
  therefore approximates "supported only by an HRA or a chart review". A plan whose extract
  assigns an HCC to its strongest source fits this approximation best.
* **First order.** Each dropped HCC is valued at its CMS-HCC V24 coefficient for a full year.
  Hierarchies and interaction terms are not re-run on aggregate input, and normalization and the
  coding pattern adjustment are not re-applied. This matches how net risk already values revenue.
* **Rebate share.** It is unchanged by the bill summary, so it still follows the rating.
* **Timing.** The set says `effective_year: 2028`. Nothing in the engines gates on it: a reform run
  shows the provisions applied to whatever years the run covers.

## What is left out

* **County quartiles and the favorable-selection adjustment.** The rate book carries only
  quartile-adjusted rates, so this needs county FFS base rates and a selection assumption. It
  becomes v2 only if a prospect asks for it.
* Studio's lever effects (for example, the in-home reach lever) are not re-scored for excluded
  sources. The Studio and Program ROI reform runs change only the QBP.
* The multi-contract provider P&L does not take a reform overlay yet.
* Medicare for All, a public option, or Medicare at 60/50 enrollment shifts.
* Any claim about the bill's final form or its chance of passing.

## Running it

| Front door | How |
|---|---|
| Console | Net Risk Position: pick a **Reform scenario**, or press **Compare with reform scenario** to get today's rules and the reform side by side on the same seed and draws, plus the dropped HCCs by family and contract. Bid Bridge also has a Reform scenario picker. |
| Brief | On a net risk card, **What would H.R. 9544 as introduced do?** opens the reform run as a second card next to the first. Net risk, bid, Stars value and settlement cards have a **Reform scenario** chip. |
| REST | `reform_policy_set_id` on `POST /api/net-risk`, `/api/bid/benchmark`, `/api/bid/pack`, `/api/stars/value`, `/api/studio`, `/api/program-roi`, `/api/settlement`. `GET /api/runs/{id}` returns `reform` (label, status, provisions). `POST /api/runs/compare` lists `reform_policy_set_id` in `differs_by`. |
| MCP | The same optional `reform_policy_set_id` argument on `net_risk_position`, `bid_benchmark`, `bid_assumption_pack`, `stars_value`, `scenario_studio`, `program_roi` and `forward_adjudication`. |
| CLI | `--reform hr9544_reform` on `net-risk`, `stars-value`, `bid-pack`, `settlement` and `program-roi`. |

```bash
uv run vantage net-risk --dataset demo-harborview-38k --policy pre_crush --seed 7 --draws 500
uv run vantage net-risk --dataset demo-harborview-38k --policy pre_crush --seed 7 --draws 500 --reform hr9544_reform
```

## How to talk about it

The scenario is about **accuracy and policy exposure**: which share of today's risk revenue rests on
diagnoses that a live bill would stop paying for, and how a contract's economics change if it
passed. It is not a coding-revenue tool, and it should never be presented as one.
