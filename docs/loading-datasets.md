# Loading an aggregate dataset

Vantage takes **aggregate, small-cell-suppressed** inputs only. A dataset is a folder with
three files, plus a fourth for a risk-bearing group that wants Forward Settlement (Module 2):

```
my-dataset/
  dataset.yaml              # dataset_id, kind, name, entity_type, premium_share, min_cell_size, description
                            #   [, stars_contract: public H-number whose Star Ratings stand in for the plan]
  contract_enrollment.csv   # contract_id, pbp_id, payment_year, plan_segment, member_months, avg_raf
  hcc_support.csv           # payment_year, contract_id, hcc_family, dx_source, n_hccs [, n_supported | support_rate, rate_basis]
  group_financials.csv      # optional: contract_id, payment_year, member_months, medical_cost_pmpm
                            #   [, rx_cost_pmpm, rx_revenue_pmpm]
```

`stars_contract` names the public contract (for example `H2406`) whose Star Ratings the
Stars engine forecasts for this dataset. Scenario Studio, Program ROI and Forward Settlement
use it; without it they fall back to `H2663`, a 3.5-star contract.

`group_financials.csv` holds the group's paid medical and pharmacy cost per member per
month by contract and year. Forward Settlement reads the latest year and trends it to the
payment year. Rows under `min_cell_size` member months are suppressed; negative costs fail
the load.

See `inputs/datasets/demo-harborview-38k/` for a complete example. Member-level columns
(member ids, names, dates of birth and the like) are rejected outright.

## 1. Validate without writing

```bash
uv run vantage load-dataset path/to/my-dataset --dry-run
```

Prints the load report as JSON: rows that would load per table, cells suppressed by reason
(`table.action.reason`), and the content hash. Nothing is written.

`--mode reject` refuses the load (exit code 1) if any cell is under `min_cell_size`, instead
of suppressing it. Use it when the sender has promised pre-suppressed data and you want
to prove it.

## 2. Load locally

```bash
uv run vantage load-dataset path/to/my-dataset
```

Writes to `$VANTAGE_DB` (default `data/vantage.duckdb`), replacing any earlier load of the
same `dataset_id`. Check it with `uv run vantage datasets`.

## 3. Make it visible in the hosted Console and MCP

The hosted Console and the hosted MCP server (both on Railway) read MotherDuck
`md:vantage`. There are two ways to get a dataset there.

**Load straight into MotherDuck** (keeps every run already saved there). From a machine
with `MOTHERDUCK_TOKEN` set:

```bash
VANTAGE_DB=md:vantage uv run vantage load-dataset path/to/my-dataset --dry-run
VANTAGE_DB=md:vantage uv run vantage load-dataset path/to/my-dataset
```

The dataset shows up in the Console's dataset picker on the next page load; no redeploy.

**Republish the whole database** (for demo datasets checked into `inputs/datasets/`).
Run the CI workflow by hand with `run_motherduck` ticked. It rebuilds the database from the
repo, tests it against `md:vantage_ci`, then replaces `md:vantage`. This **drops runs and
proposals saved on the hosted site**, so prefer the direct load for anything else.

Never load customer data into `md:vantage` or any other service without a BAA in place; the
current hosted stack is for synthetic and public data only.

## Schema changes

The API applies the current schema (new tables, `ADD COLUMN IF NOT EXISTS`) to its database
at startup, so a deploy that adds a column no longer needs a republish to keep working.
New *data* (such as a new demo dataset) still needs one of the two loads above.

## Audit results, rates and families

- `n_supported` counts supported HCCs over **all** `n_hccs`, unless you also send
  `n_reviewed`: the number of charts an audit actually reviewed in that cell. Then support is
  `n_supported / n_reviewed`, and the uncertainty is sized to the sample (40 charts reviewed
  carry 40 charts' worth of evidence, however many HCCs the cell holds). Either side of the
  reviewed split under `min_cell_size` is suppressed, like any other small cell.
- A `support_rate` without counts is used as the cell's mean support, with the literature
  prior's strength around it (no counts, so no more certainty than the prior).
- `hcc_family` must be one of `reference.hcc_family`; unknown families fail the load (and
  the dry run) with the list of offenders.

## Contract terms (Module 2)

Forward Settlement settles a group's year under a **term set**: percent of premium, quality
withhold and gate, corridor, RADV pass-through and cap, Part D carve-out, deficit carry-forward
and IBNR. Term sets are YAML under `policy/contracts/` and inherit from a parent, so a
variant states only what differs:

```yaml
term_set_id: ma_pct_withhold_4
parent: ma_pct_base
family: ma_pct_premium
label: "As base, with a 4% withhold"
version: 1
terms:
  withhold_pct: {value: 0.04, unit: share_of_group_premium, basis: illustrative}
```

Each term carries a `basis` (`contract_text`, `market_practice`, `illustrative` or
`assumption`) so a result shows where every number came from. A customer's real term sheet
becomes a new file with `basis: contract_text`. `uv run vantage term-sets` lists what is
loaded. The database build loads every file in `policy/contracts/`; `uv run vantage load-terms`
(re)loads them into an existing database, such as `VANTAGE_DB=md:vantage` for the hosted site.

Run a settlement from the CLI:

```bash
uv run vantage settlement --dataset demo-harborview-38k --seed 42 \
    --terms ma_pct_base ma_pct_withhold_4 --policy crush_proposed_2026
```

## ACO profiles (ACO mode)

ACO Track Choice runs on any public Shared Savings Program ACO by its id (from the public
results file loaded at build), or on an **ACO profile**: an ACO's own aggregate numbers from
its benchmark reports, as YAML under `inputs/aco/`. No beneficiary data.

```yaml
profile_id: my-aco
name: My ACO
kind: customer
data_year: 2024
n_ab: 18400                  # assigned beneficiaries
benchmark_pc: 13450          # updated benchmark per assigned beneficiary ($)
history:                     # savings rate = (benchmark - expenditure) / benchmark
  - {year: 2023, savings_rate: 0.038}
  - {year: 2024, savings_rate: 0.044}
qual_score: 84.2             # MSSP quality score, 0-100
track: BASIC_E
rev_exp_cat: Low Revenue     # or High Revenue
rebase_next: true            # the next agreement rebases the benchmark
```

The database build loads every profile in `inputs/aco/` and the public results in
`reference-data/vbc/mssp_aco_results.csv`. `uv run vantage load-aco` (re)loads both into an
existing database, such as `VANTAGE_DB=md:vantage`; `--profiles DIR` reads another folder.
MSSP and LEAD term sets live with the others in `policy/contracts/` (`load-terms`).

```bash
uv run vantage aco --profile my-aco --terms mssp_basic_e mssp_enhanced lead_global --horizon 5
uv run vantage aco-backtest --rerun      # after loading a new year of public results
```

## Outcomes feed (calibration)

The input feed above is what every run reads. The **outcomes feed** is optional: observed
actuals that update the weights a calibrated run uses ([calibration](calibration.md)). For a data
partner that holds outcomes but not a plan's population, it is the whole integration.

```
inputs/outcomes/<outcomes_id>/
  outcomes.yaml            outcomes_id, applies_to (dataset id), scope (own_runs | shared), source
  audit_outcomes.csv       hcc_family, dx_source, payment_year, n_reviewed, n_supported
  program_outcomes.csv     weight_id, period, n_members, observed_effect, se
  settlement_actuals.csv   contract_id, payment_year, member_months, group_settlement
```

Every file is optional; same rules as the input feed (aggregates only, member-level column names
reject the file, cells under 11 charts or members are dropped).

```bash
uv run vantage load-outcomes inputs/outcomes/sample-harborview-outcomes --dry-run
uv run vantage load-outcomes inputs/outcomes/sample-harborview-outcomes
uv run vantage drop-outcomes sample-harborview-outcomes
```

- `scope: own_runs` (default): only runs on `applies_to` read the evidence. `shared` records the
  customer's consent to pool it into the shared calibrated release (rebuilt by `vantage calibrate`).
- Audit outcomes update each HCC family's support-rate prior for **calibrated** runs: net risk
  position and the settlement that reads it. With the sample file, Harborview's calibrated
  recovery rises about a fifth; a literature-prior run is unchanged. Cells the input dataset
  already counts (`n_supported`) keep their own counts.
- Program outcomes update effect-weight posteriors in the shared release when shared;
  settlement actuals are stored for the settlement back-test.
- Loading outcomes changes the run fingerprint, so an earlier run is not reused.

## Removing a dataset

```bash
uv run vantage drop-dataset sample-kestrel-23k           # refused if runs or proposals use it
uv run vantage drop-dataset sample-kestrel-23k --force   # runs are kept; its proposals are deleted
```

## Demo: sample customer dataset

`inputs/samples/sample-kestrel-23k/` is a fictional customer submission (Kestrel Health
Plan, kind `customer`, every number synthetic) built with the integration kit. Samples are
not part of a database build; they are imported for a demo and removed afterwards, so the
demo can be repeated for each prospect.

- **Console**: Runs & Library → *Sample customer data* → **Import sample customer**. The
  Console switches to it; the header badge reads "Sample customer data". When the demo is
  over, **Remove sample customer** (click twice) deletes it with every run and proposal made
  on it. Bayside and Harborview are never touched.
- **CLI**: `uv run vantage sample list`, `uv run vantage sample import`,
  `uv run vantage sample remove` (default id `sample-kestrel-23k`; add
  `VANTAGE_DB=md:vantage` to act on the hosted site).
- **REST**: `GET /api/samples`, `POST /api/samples/{id}/import`, `DELETE /api/samples/{id}`.

Only ids starting with `sample-` that exist under `inputs/samples/` can be imported or
removed this way. Real customer data uses `load-dataset` and `drop-dataset`.

## Demo: upload screen in the Console

**Upload Data** (left rail, under Trust) takes one aggregate dataset's files in the browser:
`contract_enrollment.csv`, `hcc_support.csv`, and optionally `dataset.yaml` and
`group_financials.csv`. Drop the files or their folder, or stage one of the three demo
sets with one click (`inputs/samples/sample-kestrel-23k/` and
`inputs/upload-examples/{member-level-columns,unsuppressed}/`).

**Check files** runs the loader's own checks in order and reports each one, writing nothing:
files present, `dataset.yaml` readable, **no member-level columns** (a `member_id`, MBI, name,
date of birth, ZIP… refuses the file and stops there), columns match the contract, values
valid (types, diagnosis sources, HCC families, duplicates), small cells (suppressed with a
warning, or refused with **Strict** ticked), and room in the workspace (5 uploads per user).
A file that passes shows a preview (members, contracts, HCC rows by diagnosis source,
observed support rates, cells suppressed, content hash). **Load into my workspace** loads
it as `up-<workspace>-<id>` (the workspace is derived from the signed-in email), selects it
in the dataset picker and every view runs on it.

Uploads are visible only to their owner in `GET /api/datasets` and can be removed by their
owner with their runs and proposals (**Your uploads → Remove**). REST: `GET /api/uploads`,
`GET /api/uploads/examples`, `POST /api/uploads/validate`, `POST /api/uploads/load` (body
`{"files": {name: text}, "strict": false}`), `DELETE /api/uploads/{id}`.

This is demo-grade: uploads sit in the same database as everything else (MotherDuck on the
hosted site), and the MCP server's `list_datasets` is not filtered by user. Use synthetic or
test files only on the hosted Console. Real customer data is loaded with `load-dataset` as
above; a self-serve upload API with separate storage per customer waits for a paying customer.
