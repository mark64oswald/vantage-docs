# Custom domain: vantage-intelligence.ai

| Hostname | Serves | Where |
|---|---|---|
| `app.vantage-intelligence.ai` | Console, Brief, REST API | Railway service `vantage-console` |
| `mcp.vantage-intelligence.ai` | MCP server at `/mcp` (AuthKit OAuth) | the same Railway service |

Both hostnames are custom domains on the one Railway service (the Hobby plan allows two
per service), so there is no extra service or cost. Requests whose `Host` is
`VANTAGE_MCP_HOST` go straight to the MCP app (`vantage_api.app.HostRouter`), bypassing the
Console's cookie sign-in; the MCP server does its own OAuth through WorkOS AuthKit, with its
metadata at the host root (`/.well-known/oauth-protected-resource/mcp`). Sign-in stays on
WorkOS's own AuthKit domain (a custom `auth.` domain is a $99/mo WorkOS add-on, declined
Nov 22, 2026).

## Railway variables on `vantage-console`

| Variable | Value |
|---|---|
| `VANTAGE_PUBLIC_URL` | `https://app.vantage-intelligence.ai` |
| `VANTAGE_MCP_HOST` | `mcp.vantage-intelligence.ai` |
| `AUTHKIT_DOMAIN` | the AuthKit domain (same value as on Horizon) |
| `BASE_URL` | `https://mcp.vantage-intelligence.ai` |
| `VANTAGE_ALLOWED_EMAILS`, `WORKOS_API_KEY` | already set for the Console |

## WorkOS

Add `https://app.vantage-intelligence.ai/auth/callback` as a redirect URI (keep the Railway
one until the move is verified). The MCP OAuth flow uses AuthKit's dynamic client
registration and needs no redirect URI of ours.

## DNS (GoDaddy)

Railway shows one CNAME target per custom domain (and a TXT record for verification when it
asks for one). Add each in GoDaddy DNS for `vantage-intelligence.ai`: host `app` and host
`mcp`, type CNAME, value as Railway shows it.

## After the move

- Set repo variables `VANTAGE_CONSOLE_URL=https://app.vantage-intelligence.ai` and
  `VANTAGE_MCP_URL=https://mcp.vantage-intelligence.ai/mcp` so the nightly smoke test
  checks the new hosts.
- Re-add the MCP server in clients:
  `claude mcp add --scope user --transport http vantage https://mcp.vantage-intelligence.ai/mcp`.
- Keep the Horizon server running until clients have moved, then retire it.
