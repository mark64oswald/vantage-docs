# Program ROI under a contract, ACCESS, and the multi-contract P&L

Three Q2 additions that sit on the engines already in place: Scenario Studio and
Program ROI (Module 3), Forward Settlement (Module 2) and ACO mode.

## Program ROI under a named contract (#137)

Without a contract, Program ROI shows each program from the plan's and the group's
side, with the group paid a flat share of premium. Pass a term set and every year's
program effect goes through that contract's settlement rule instead:

```bash
uv run vantage program-roi --dataset demo-harborview-38k --policy pre_crush --seed 42 \
    --programs chf_transitions pharmacist_htn --terms ma_pct_base --tornado pharmacist_htn
```

- **group_contract** and **plan_contract** perspectives: the NPV each side keeps
  once the corridor, withhold and RADV pass-through apply. The two always add up to
  the program's total effect, by domain (`output.program_capture`).
- A **tornado** for one program (`--tornado`, `tornado_program` in the API and MCP): each library effect moves
  between its low and high, each program cost moves ±25%, everything else stays on
  its draws. Drivers that do not move the NPV are left out.
- It needs the group's own cost (`group_financials`), so it runs on a risk-bearing
  group's dataset such as Harborview, not on a plan dataset.

On Harborview under `ma_pct_base`, the 5% corridor roughly halves what the group
keeps of a program's cost and risk-revenue gains (group share about 0.55 to 0.66),
and about 0.78 of the quality gain because of withhold gate crossings.

## ACCESS outcome payments (#137)

CMS's ACCESS model (July 2026 to June 2036) pays a technology-enabled chronic care
organization a fixed outcome-aligned payment per aligned Original Medicare
beneficiary. Half of Medicare's share is paid monthly. The other half depends on
the share of beneficiaries who meet every outcome target.

Term sets `access_eckm` and `access_ckm` (`policy/contracts/`) hold the allowed
amounts, Medicare share, threshold and model end year with their page cites. Two
terms are illustrative until CMS publishes the reconciliation detail: the rule below
the threshold (proportional) and the substitute-spend haircut.

Pick an ACCESS term set in Program ROI and qualifying programs get a **vendor**
perspective: revenue, cost and margin by year, P(meeting the threshold), and a
tornado on the vendor's own assumptions (panel size, retention, baseline
attainment, cost). `pharmacist_htn` qualifies for eCKM and `diabetes_rpm` for CKM.

## Multi-contract P&L (#136, demo grade)

A provider group usually holds several risk contracts at once. A portfolio
(`inputs/portfolios/*.yaml`) lists them: MA percent-of-premium contracts, each with
its dataset, term set and Stars contract, and one ACO agreement with its profile.

```bash
uv run vantage load-portfolio inputs/portfolios
uv run vantage portfolio demo-harborview-portfolio --seed 42 --draws 1000
```

Every contract settles on the same draws for one payment year. One common Medicare
trend drives MA rates and the ACO benchmark. The group's own cost trend follows it by
`trend_link` (0 to 1, illustrative 0.5). Each MA contract has its own Star Rating
draws. The run reports each contract's range, its share of the combined variance,
its correlation with the rest, what it does in the group's worst tenth of years, and
a driver table that shows where a move offsets and where it stacks.

The Harborview demo (seed 42, 1,000 draws, PY2029): two MA contracts and a LEAD
agreement. The combined P50 is $12.8M, with a 36% chance of an overall loss against
84% that at least one contract loses. The combined P10 sits $5.1M above the sum of
the contracts' P10s. The LEAD agreement is about 2% of the variance and nearly
uncorrelated with the MA book.

Demo grade until a group shares its own term sheets.

## CY2028 bid pack demo (#139)

`scripts/bid_pack_demo.py` builds the pre-announcement CY2028 pack for public
contract H2406. It uses public data for the published 2027 Star Rating (CY2028 QBP),
the CY2027 rate book for Chester and York, PA and New Castle, DE, and the CY2027
factors. The risk score and RADV reserve come from a synthetic stand-in book. It
writes the memo, XLSX and CSV. After the April 2027 Rate Announcement, re-run it
through the drill (docs/rate-announcement-drill.md).
