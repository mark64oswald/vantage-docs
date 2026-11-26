# Multi-tenancy: separate storage per organization

Status: **built behind a switch** (`VANTAGE_TENANCY`, off by default; PR #172). With the
switch off, everything runs on the one `$VANTAGE_DB` database as before. See
[Running it](#running-it) for the commands, the cutover and the rollback.

## Why

Today Vantage has one tenant. Sign-in is WorkOS AuthKit plus an email allow list
(`VANTAGE_ALLOWED_EMAILS`), and every signed-in user, in the Console, REST, chat and the
hosted MCP server, opens the same database (`md:vantage`). A second user would see every
run, proposal and dataset, and could delete what the screens allow. The only separation is
the demo upload screen's (#169): uploads (`up-<workspace>-…`) are hidden from other users in
the Console's dataset list. That is a display filter, not a security boundary: Runs & Library,
the MCP `list_datasets` tool and a known dataset id still reach them.

A paying customer needs three things that one shared database cannot give cleanly:

1. **Isolation**: one organization can never read or change another's datasets, outcomes,
   runs or proposals, through any front door.
2. **A clean exit**: a customer's data can be exported and deleted completely.
3. **Shared public data**: CMS reference data, policy sets and public weights releases stay one
   copy that every organization reads, so a rule change (a new policy set on Rate
   Announcement day) reaches everyone at once.

## What WorkOS gives natively, and what Vantage builds

| WorkOS (configure, don't build) | Vantage (build) |
|---|---|
| **Organizations** and **memberships**; an org picker when a user belongs to more than one | Map `org_id` to that org's database |
| The **`org_id` and `role`** in the AuthKit session, and in the MCP access token | Read them on every request; refuse a request with no org |
| **Roles and permissions** (for example `admin`, `analyst`, `viewer`) | Check permissions on write routes (load, remove, proposals) |
| **SSO** (SAML/OIDC) per organization, **directory sync** (SCIM), **admin portal** | Nothing, unless a customer asks for it (priced per connection) |
| | Org provisioning, the demo org, isolation tests |

WorkOS answers *who* is asking and *for which org*. Keeping one org's rows away from
another's queries is Vantage's job.

## Options considered

| | A. One database, `org_id` column | **B. One database per org (chosen)** |
|---|---|---|
| Enforcement | Every query must filter by org. DuckDB and MotherDuck have no row-level security, so one missed `WHERE` leaks data | The request's store opens only that org's database, so another org's rows aren't reachable |
| Code change | Hundreds of queries and every table's key | Mostly one place: which store a request gets |
| Delete a customer | DELETEs across every table | Drop one database |
| Cost | One database | MotherDuck databases are cheap; storage is the same |
| Cross-org analytics (benchmarks) | Easy, and easy to leak | Deliberate: an explicit job that reads several databases |

Also considered: one *schema* per org in one database (no better isolation than A in
DuckDB, and it multiplies schema names), and the customer's own warehouse (Snowflake or
Databricks through the DataStore interface). The second stays the answer for a customer
that wants data in its own cloud account or member-level work; it is not this design.

## Design

```
            WorkOS AuthKit  ──>  session / MCP token: user, org_id, role
                                      │
   Console / REST / chat / MCP  ──────┤
                                      ▼
                     Store router (org_id -> store, cached per process)
                 ┌────────────────────┴───────────────────┐
   md:vantage_shared   (read-only to apps)      md:vantage_org_<slug>  (read-write)
   reference.*   public CMS data                 input.*        datasets, uploads, outcomes
   policy.*      policy sets, codes              run.*          runs, proposals
   contract.*    term sets, portfolios           output.*       results
   calibration.* public weights releases         calibration.*  the org's own releases
```

**Two kinds of database.**

- `md:vantage_shared`: everything built from public data or the repo: `reference`, `policy`,
  `contract`, public `calibration` releases, `scenario` and `population` definitions. Written
  only by the publish job (CI `run_motherduck`) and the startup sync of policy sets, term
  sets and portfolios (`VANTAGE_SYNC_REPO_DATA`), which moves from per-process to that job.
- `md:vantage_org_<slug>`: one per organization: `input`, `run`, `output`, and the org's own
  calibration releases from its outcomes feed.

**Keeping the query text unchanged.** The engine writes schema-qualified names
(`reference.hcc_family`, `policy.code_list`). An org database is opened as the default
catalog with the shared database attached read-only as `shared`, and provisioning creates
**shim views** in the org database (`CREATE VIEW reference.hcc_family AS SELECT * FROM
shared.reference.hcc_family`, one per shared table). Existing SQL then runs as is, and
writes can only land in the org's own database. The view list is regenerated whenever the
shared schema changes (a step in the publish job). The read-only attach is a MotherDuck
share, so even a bug cannot write to shared data.

**Store router.** `ops.open_store()` takes an `org` and returns a cached store for that org's
database. The FastAPI `call()` helper resolves the org from `request.state.user`
(`org_id` from the session), and the MCP server does the same from the access token's
claims. Chat already goes through the MCP tools, so it inherits the org. A request with no
org gets 403. The store lock becomes per store.

**Calibration.** `weights_version: calibrated` resolves to the org's newest release when it
has one, else the newest public release. An org's outcomes never update the public release
unless the org opts in to pooling (as the Integration Guide already promises).

**Roles.**

| Permission | admin | analyst | viewer |
|---|---|---|---|
| Run any module, view runs | ✓ | ✓ | ✓ |
| Create or edit proposals | ✓ | ✓ | |
| Upload or load datasets and outcomes | ✓ | ✓ | |
| Remove datasets, outcomes or runs | ✓ | | |
| Invite users (WorkOS admin portal) | ✓ | | |

Uploads become shared within the org (the `up-<workspace>` prefix goes away). Within-org
per-user privacy is not a goal.

**Demo org.** Bayside, Harborview, Cedar Valley and the Kestrel sample move to a "Vantage Demo"
organization. Prospects get demo-org access only. A customer never sees demo data unless it
is also a member of the demo org.

**Provisioning.** `vantage org create <slug> --workos-org <id>` creates the database, the
shim views and the org row in a small `shared.tenant.org` table (slug, WorkOS id, database,
created). `vantage org export|delete <slug>` dumps to Parquet or drops the database.

## Build plan

| Step | Work | Days |
|---|---|---|
| 1 | WorkOS: create organizations, turn on roles; read `org_id` and `role` from the Console session and the MCP token | 2–3 |
| 2 | Split the database: shared vs org schemas, the publish job writes `vantage_shared`, shim views, store router with per-org cache and lock | 5–7 |
| 3 | `vantage org create/export/delete`, migrate today's `md:vantage` into the demo org | 2 |
| 4 | Role checks on write routes; calibrated-weights resolution per org | 1–2 |
| 5 | Isolation tests: org A can never read or write org B through REST, MCP, chat or the CLI; docs (Data Model, Integration Guide, Walkthrough) | 3–4 |

## Not in scope

- **PHI.** This is separate storage for aggregates and outcomes, not a HIPAA platform.
  Member-level data still needs a BAA with the storage vendor (or the customer's own
  warehouse), audit logging and the production-platform work (#32).
- **Separate credentials per org.** One MotherDuck service token reaches every org database,
  so the store router is the security boundary and the isolation tests guard it. A customer
  that needs its own credentials or cloud account gets the customer-warehouse option.
- **A public upload API.** The upload screen loads into the org's database once this lands;
  an API for scheduled feeds is a separate item.

## Open questions

- Does a consultant who serves several plans get one org per client (recommended) or one
  org with several datasets?
- Pooling of outcomes across orgs for public calibration: opt-in per org, recorded where?
- MotherDuck plan limits on database count and concurrent connections at, say, 20 orgs.

## Running it

Code: `vantage.db.tenancy` (DDL split, shim views, read-only attach) and
`vantage_services.tenancy` (switch, router, roles, org admin). Isolation tests:
`tests/test_tenancy.py`; end-to-end check: `scripts/tenancy_check.py [--motherduck]`.

**Settings** (all optional except the switch):

| Variable | Default | Meaning |
|---|---|---|
| `VANTAGE_TENANCY` | off | `1` turns multi-org on |
| `VANTAGE_SHARED_DB` | `md:vantage_shared` (when `VANTAGE_DB` is `md:`), else `data/vantage_shared.duckdb` | the shared database |
| `VANTAGE_SHARED_ATTACH` | `VANTAGE_SHARED_DB` | what org connections attach read-only as `vantage_shared` (a MotherDuck share URL also works) |
| `VANTAGE_ORG_DB_PREFIX` | `md:vantage_org_`, else `data/orgs/` | org databases are `<prefix><slug>` |
| `VANTAGE_DEMO_ORG` | `demo` | the org the repository's demo portfolios sync into at startup |
| `VANTAGE_DEV_ORG` | none | local runs without sign-in: every request uses this org as admin (ignored when sign-in is required) |

**Commands** (with `VANTAGE_TENANCY=1`):

```bash
uv run vantage org publish-shared --from md:vantage          # or data/vantage.duckdb
uv run vantage org create demo --name "Vantage Demo" --workos-org org_...
uv run vantage org import demo --from md:vantage             # copy datasets, runs, calibration
uv run vantage org list
uv run vantage org link demo --workos-org org_...            # (re)map to a WorkOS organization
uv run vantage org export demo --out exports/demo            # Parquet, one file per table
uv run vantage org delete demo --confirm demo                # drops the database
uv run vantage --org demo datasets                           # any command, in one org
```

**Who gets which org.** The Console session and the MCP access token carry the WorkOS
`org_id` and `role`. A session or token issued before the user joined an organization has
no `org_id`; then the user's one registered WorkOS membership is used (several, with no
choice made, is refused). WorkOS's default `member` role counts as analyst.

**Cutover** (hosted): publish the shared database from `md:vantage`, create the demo org and
import `md:vantage` into it, create the WorkOS organizations and memberships, link them, set
`VANTAGE_TENANCY=1` on the Railway service and redeploy. `md:vantage` is left as it is.

**Rollback**: remove `VANTAGE_TENANCY` (or set it to `0`) and redeploy. The service goes back
to `md:vantage`; runs made while the switch was on stay in the org databases.

**What differs from the design above**

- Public calibration releases are *copied* into each new org (and today's releases come
  over with the demo org), not read through shim views, because an org writes its own
  releases into the same tables. `vantage calibrate --org <slug>` rebuilds an org's.
- Uploads keep their per-user `up-<workspace>` naming inside the org for now.
- A user in several organizations needs AuthKit's org picker at sign-in; there is no
  in-Console org switch yet.
