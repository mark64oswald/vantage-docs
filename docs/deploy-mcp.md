# The hosted Vantage MCP server

Vantage's MCP server is `packages/mcp/src/vantage_mcp/server.py:mcp`. In production it
runs inside the Railway service `vantage-console`, alongside the Console, and answers at
**`https://mcp.vantage-intelligence.ai/mcp`**. Requests whose `Host` is
`VANTAGE_MCP_HOST` go straight to the MCP app (`vantage_api.app.HostRouter`), so the
Console's cookie sign-in doesn't apply; the MCP server does its own OAuth through WorkOS
AuthKit. It reads MotherDuck `md:vantage`. Everything served is aggregate or public
contract-level data (no PHI): none of Railway, WorkOS or MotherDuck's plans in use come
with a BAA.

```text
 Claude Desktop / claude.ai / Claude Code
        │  OAuth (WorkOS AuthKit, VANTAGE_ALLOWED_EMAILS)
        ▼
 mcp.vantage-intelligence.ai ──▶ Railway vantage-console: HostRouter ──▶ vantage_mcp.server:mcp
                                                                              │ VANTAGE_DB=md:vantage
                                                                              ▼
                                                                    MotherDuck "vantage"
```

Domain, DNS and the Railway variables (`VANTAGE_MCP_HOST`, `AUTHKIT_DOMAIN`, `BASE_URL`,
`VANTAGE_ALLOWED_EMAILS`, `WORKOS_API_KEY`) are in docs/custom-domain.md. Until Nov 22,
2026 the server ran on Prefect Horizon at `vantage.fastmcp.app`; that server is retired.

## Data in MotherDuck

The CI workflow's `motherduck` job (run it from GitHub Actions with **run_motherduck**
ticked on `main`) builds the database, publishes it to `md:vantage_ci`, runs the service,
REST and MCP tests against it, and if they pass publishes `md:vantage`, the database the
hosted server reads. It needs the `MOTHERDUCK_TOKEN` repository secret. To add a dataset
without a rebuild, see docs/loading-datasets.md.

## Auth

The server uses WorkOS AuthKit when `AUTHKIT_DOMAIN` is set, and has no auth of its own
otherwise (local runs). In WorkOS, Dynamic Client Registration is on, so MCP clients
register themselves, and Resource Indicators list `https://mcp.vantage-intelligence.ai/mcp`
so tokens carry that audience. `VANTAGE_ALLOWED_EMAILS` (comma list of addresses or
`@domain`s, as on the Console) limits who gets in, and needs `WORKOS_API_KEY` to look up
each user's email. Without it, anyone who can sign up in AuthKit gets in.

## Connect a client

Claude Code:

```bash
claude mcp add --scope user --transport http vantage https://mcp.vantage-intelligence.ai/mcp
```

`--scope user` matters: the default scope registers the server only for the folder the
command ran in. Restart Claude Code, run `/mcp` and sign in. In Claude Desktop or claude.ai,
add a custom connector with the same URL; the connector menus change often, so look for the
option on screen. Then ask:

1. "Use the vantage MCP to list the policy sets."
2. "Use vantage to run net_risk_position on demo-bayside-100k under
   crush_proposed_2026, seed 11, 500 draws." The answer carries a `run_id`;
   `VANTAGE_DB=md:vantage uv run vantage run <run_id>` returns the same headline.

## Long runs

Clients give up on a tool call after about 80 seconds. `net_risk_position` takes about
10-20 seconds at 1000 draws, and an identical repeat call (same inputs, seed and draws)
returns the stored run at once with `reused: true`. If a call ever times out, find the run
with `list_runs` / `get_run` instead of starting a new one.

## Local equivalents

```bash
uv run vantage-mcp                      # stdio, for Claude Desktop/Code locally
uv run vantage-mcp --http --port 8001   # streamable HTTP at http://127.0.0.1:8001/mcp
uv run vantage-api                      # REST at :8000/api, docs at :8000/docs, MCP at :8000/mcp
uv run fastmcp inspect packages/mcp/src/vantage_mcp/server.py:mcp
```

Claude Code: `claude mcp add vantage -- uv --directory /path/to/vantage run vantage-mcp`.

## Tools

| Tool | What it does | Writes a run |
|---|---|---|
| `list_policy_sets`, `list_datasets` | catalogs | no |
| `net_risk_position` | Module 1 RADV recovery and net revenue | yes |
| `audit_source_exposure` | drivers by HCC family and dx source, review priority | no |
| `stars_forecast` | P(4+) and rating distribution per contract | yes |
| `stars_value` | QBP probability, payment range, $ per measure point | yes |
| `bid_benchmark` | service-area benchmark, effective QBP and rebate from the CMS rate book | no |
| `search_counties` | find a county's SSA code and rates in the CMS rate book | no |
| `search_star_contracts` | published Star Ratings by contract or parent organization | no |
| `bid_assumption_pack` | CY bid assumptions (risk score, QBP, rebate, RADV reserve, net revenue) with P10/P50/P90 and sources | yes |
| `scenario_studio`, `program_roi` | lever settings and named programs to a five-year margin; Stars lifts re-rated on the Stars engine. `program_roi` with `term_set` splits each program's value between group and plan under a named contract, ranks a tornado, and with an ACCESS term set gives the vendor's outcome-payment view | yes |
| `save_proposal`, `list_proposals` | named options per dataset, each tied to its runs | no |
| `list_term_sets` | Module 2 contract term sets with resolved terms and their basis | no |
| `forward_adjudication` | Module 2: a risk-bearing group's settlement on a percent-of-premium MA contract, terms compared on the same draws, swing drivers | yes |
| `list_aco_options` | ACO mode: ACO profiles, public MSSP ACOs matching a search, MSSP and LEAD tracks with resolved terms | no |
| `aco_settlement` | ACO mode: an ACO's shared savings or losses under MSSP BASIC A–E, ENHANCED and LEAD tracks over the agreement, on the same draws, with swing drivers | yes |
| `multi_contract_pnl` | every MA percent-of-premium contract and ACO agreement a provider group holds, settled on the same draws: combined range, offsets, drivers by contract (no `portfolio_id`: the catalog) | yes |
| `aco_backtest` | ACO mode: the savings-rate forecast scored against published MSSP results by year (latest run, or rerun) | yes, on rerun |
| `list_weights_releases` | calibration: weights releases (literature prior, calibrated), parameters moved by evidence, default, held-out back-test | no |
| `list_runs`, `get_run`, `compare_runs` | stored runs and provenance | no |

The run tools (`stars_forecast`, `stars_value`, `scenario_studio`, `program_roi`, `forward_adjudication`) take
`weights_version`: `calibrated`, `prior` (literature prior), `empirical` (raw history, the engine before
calibration) or a `wr-` release id. Left out, the default release applies; `get_run` shows `weights_label`.

`net_risk_position`, `bid_benchmark`, `bid_assumption_pack`, `stars_value`, `scenario_studio`, `program_roi` and
`forward_adjudication` also take `reform_policy_set_id`, a hypothetical reform overlay such as `hr9544_reform`
(H.R. 9544 as introduced). Left out, today's rules apply; `get_run` shows the run's `reform`. See
[reform-scenarios.md](reform-scenarios.md).

Export a bid pack with `GET /api/runs/{run_id}/bid-pack?format=csv|xlsx|memo` or
`vantage bid-export <run_id> --format xlsx --out pack.xlsx`.
