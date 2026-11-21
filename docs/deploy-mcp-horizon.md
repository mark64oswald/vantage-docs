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

1. MotherDuck (app.motherduck.com) → your name/avatar, top left → **Settings** →
   **Access Tokens** → **Create token**. Name it `vantage-github`, type
   Read/Write, no expiry (or a long one). Copy it now; it is shown once.
2. GitHub → `mark64oswald/vantage` → **Settings** → **Secrets and variables** →
   **Actions** → **New repository secret**. Name `MOTHERDUCK_TOKEN`, paste the
   token, **Add secret**.
3. GitHub → **Actions** → **CI** (left list) → **Run workflow** (right) → branch
   `main`, tick **run_motherduck** → **Run workflow**. The `motherduck` job
   builds the database, publishes it to `md:vantage_ci`, runs the service, REST
   and MCP tests against it, and if they pass publishes `md:vantage`, the
   database the hosted server reads. No local tools needed.

## 2. Create the Horizon server

In Horizon (signed in with GitHub), create a server from `mark64oswald/vantage`:

| Setting | Value |
|---|---|
| Server name | `vantage` (sets the URL, `https://vantage.fastmcp.app/mcp`) |
| Entrypoint | `packages/mcp/src/vantage_mcp/server.py:mcp` |
| Authentication | On (Horizon managed OAuth; only your organization can connect) |
| Environment variables | `VANTAGE_DB` = `md:vantage`; `MOTHERDUCK_TOKEN` = the same token (mark it secret if offered) |
| PR previews | On |

Horizon installs dependencies from the root `pyproject.toml`. The repo is a uv
workspace (core, services, api, mcp). If the first build cannot resolve the
workspace members, tell Claude the build log line and it will add a
Horizon-specific requirements file.

## 3. Smoke test

From Claude Code (the most reliable client to test with):

```bash
claude mcp add --scope user --transport http vantage https://vantage.fastmcp.app/mcp
```

`--scope user` matters: the default scope registers the server only for the
folder the command ran in. Restart Claude Code, run `/mcp` and sign in, then ask:

1. "Use the vantage MCP to list the policy sets."
2. "Use vantage to run net_risk_position on demo-bayside-100k under
   crush_proposed_2026, seed 11, 500 draws." The answer carries a `run_id`;
   `VANTAGE_DB=md:vantage uv run vantage run <run_id>` returns the same headline.

Horizon's server page also has its own chat for a quick check. Claude Desktop and
claude.ai connector menus change often; look for a custom connector option there.

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
| `list_runs`, `get_run`, `compare_runs` | stored runs and provenance | no |

`forward_adjudication` (Module 2, after June 7) joins when that module exists.
Export a bid pack with `GET /api/runs/{run_id}/bid-pack?format=csv|xlsx|memo` or
`vantage bid-export <run_id> --format xlsx --out pack.xlsx`.

## Switching to WorkOS sign-in (P-05, Horizon Pro)

The server uses WorkOS AuthKit when `AUTHKIT_DOMAIN` is set, and otherwise has
no auth of its own (local runs, or Horizon's managed OAuth). Never use both on the
hosted server.

1. In WorkOS, these are the settings FastMCP's AuthKit guide calls for. Its
   dashboard path was "Applications → Configuration", but check it on screen.
   - Turn on **Dynamic Client Registration**, so Claude and other MCP clients
     can register themselves.
   - Turn on **Resource Indicators**, and list `https://vantage.fastmcp.app/mcp`
     as the resource, so tokens carry that audience.
   - Copy the AuthKit domain. It looks like `https://<name>-<id>.authkit.app`.
2. In Horizon, on the vantage server:
   - add the environment variables `AUTHKIT_DOMAIN` = that domain and
     `BASE_URL` = `https://vantage.fastmcp.app`
   - to limit who can connect, add `VANTAGE_ALLOWED_EMAILS` (comma list of
     addresses or `@domain`s, same format as the Console) and `WORKOS_API_KEY`
     (the server looks up each signed-in user's email with it). Without
     `VANTAGE_ALLOWED_EMAILS`, anyone who can sign up in AuthKit gets in.
   - turn Horizon's own authentication off
   - redeploy
3. Reconnect each client. Claude Code: run `/mcp`, pick vantage and sign in
   again. The sign-in page is now WorkOS.

To roll back, remove `AUTHKIT_DOMAIN` and turn Horizon auth back on.
