# Deploying the Vantage MCP server to Prefect Horizon (P-03)

Vantage's MCP server is `packages/mcp/src/vantage_mcp/server.py:mcp`. Horizon
runs it from GitHub; it reads its data from MotherDuck. Everything served is
aggregate or public contract-level data (no PHI), which is what the free plans
allow: none of Horizon, Prefect or MotherDuck's plans in use come with a BAA.

```text
 Claude Desktop / claude.ai ──OAuth (Horizon managed)──▶ Horizon: vantage_mcp.server:mcp
                                                              │  VANTAGE_DB=md:vantage
                                                              ▼
                                                        MotherDuck "vantage"
                                                     (reference, policy, input, run, output)
```

## 1. Put the data in MotherDuck (once, then after reference refreshes)

1. In MotherDuck, create an access token (Settings → Access tokens). Treat it as a secret.
2. Add it to the GitHub repo as an Actions secret named `MOTHERDUCK_TOKEN`
   (Settings → Secrets and variables → Actions).
3. Check the round trip: Actions → CI → Run workflow → tick
   "publish the DB to MotherDuck". That builds the DB, publishes it to
   `md:vantage_ci` and runs the service, REST and MCP tests against it.
4. Publish the database the server will use (locally, or as a one-off step):

   ```bash
   uv run python scripts/init_db.py --rebuild
   MOTHERDUCK_TOKEN=... uv run python scripts/publish_motherduck.py --name vantage
   ```

## 2. Create the Horizon server

In Horizon (signed in with GitHub), create a server from `mark64oswald/vantage`:

| Setting | Value |
|---|---|
| Server name | `vantage` (sets the URL, `https://vantage.fastmcp.app/mcp`) |
| Entrypoint | `packages/mcp/src/vantage_mcp/server.py:mcp` |
| Authentication | On (Horizon managed OAuth; only your organization can connect) |
| Environment | `VANTAGE_DB=md:vantage`, `MOTHERDUCK_TOKEN=<token>` |
| PR previews | On |

Horizon installs dependencies from the root `pyproject.toml`. The repo is a uv
workspace (core, services, api, mcp). If the first build cannot resolve the
workspace members, tell Claude the build log line and it will add a
Horizon-specific requirements file.

## 3. Smoke test

1. In Claude Desktop, add a custom connector with the Horizon URL and sign in.
2. Ask: "List Vantage policy sets", then "Run net_risk_position on
   demo-bayside-100k under crush_proposed_2026 with seed 11 and 500 draws".
3. The answer should carry a `run_id`; `get_run` on that id from the CLI
   (`VANTAGE_DB=md:vantage uv run vantage run <run_id>`) returns the same headline.

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
| `list_runs`, `get_run`, `compare_runs` | stored runs and provenance | no |

`forward_adjudication` (Module 2, after June 7) and `bid_assumption_pack`
(Phase 4) join when those modules exist.
