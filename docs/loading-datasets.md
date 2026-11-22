# Loading an aggregate dataset

Vantage takes **aggregate, small-cell-suppressed** inputs only. A dataset is a folder with
three files:

```
my-dataset/
  dataset.yaml              # dataset_id, kind, name, entity_type, premium_share, min_cell_size, description
  contract_enrollment.csv   # contract_id, pbp_id, payment_year, plan_segment, member_months, avg_raf
  hcc_support.csv           # payment_year, contract_id, hcc_family, dx_source, n_hccs [, n_supported | support_rate, rate_basis]
```

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
