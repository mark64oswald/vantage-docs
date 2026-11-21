# Deploy the Vantage Console on Railway (U-06, P-06)

The Console, its REST API and the **Ask Vantage** chat run as one container
(`Dockerfile`, `railway.json`). It reads and writes runs in MotherDuck
(`md:vantage`), the same database as the hosted MCP server, so a run started in
the Console shows up in Claude Desktop and the other way round.

```
 browser ──► Railway: vantage-api (FastAPI)
               ├── /console        static Console (U-01..U-05)
               ├── /api/*          REST over vantage-services
               ├── /api/chat       Ask Vantage: Claude + the MCP tools ──► Anthropic API
               ├── /auth/*         WorkOS AuthKit sign-in (P-06)
               └── store ──────────────────────────────────────────────► MotherDuck md:vantage
```

Data stays aggregate and synthetic: Railway, MotherDuck and the Anthropic API
see no PHI because Vantage holds none. Keep it that way; none of these services
is covered by a BAA in this setup.

## What you need

| Thing | Where it comes from |
|---|---|
| Anthropic API key | Claude Console → API keys. Pay per use. |
| MotherDuck token | the one already in GitHub secrets / Horizon (`MOTHERDUCK_TOKEN`) |
| WorkOS API key and client ID | WorkOS dashboard → API keys (same WorkOS account as P-05) |
| Cookie password | generate locally, see below |
| Railway project | Hobby plan; deploy from the GitHub repo |

Generate the cookie password (a Fernet key) on your machine:

```bash
uv run python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

## Environment variables (Railway service → Variables)

| Variable | Value |
|---|---|
| `ANTHROPIC_API_KEY` | your key |
| `MOTHERDUCK_TOKEN` | your token |
| `WORKOS_API_KEY` | `sk_...` |
| `WORKOS_CLIENT_ID` | `client_...` |
| `WORKOS_COOKIE_PASSWORD` | the generated key |
| `VANTAGE_PUBLIC_URL` | the Railway URL, e.g. `https://vantage-console-production.up.railway.app` |
| `VANTAGE_ALLOWED_EMAILS` | optional, comma list of addresses or `@domain`s allowed in |
| `VANTAGE_CHAT_MODEL` | optional; default `claude-opus-5-5` |

The Dockerfile already sets `VANTAGE_DB=md:vantage`, `VANTAGE_MOUNT_MCP=0` (the
MCP server lives on Horizon) and `VANTAGE_REQUIRE_AUTH=1`: until the `WORKOS_*`
variables are set, every page except `/health` returns 503, so the chat can't
spend API credit for strangers. Railway sets `PORT`; the app reads it.

## WorkOS settings

Register the callback with the WorkOS API (no dashboard): copy your WorkOS API key, then

```bash
pbpaste | uv run python scripts/workos_setup.py <VANTAGE_PUBLIC_URL>
```

Sign-out only clears the Console's cookie, so no sign-out redirect is needed.
AuthKit's default sign-in methods (email) work as they are; add others in WorkOS if you want them.

Use `VANTAGE_ALLOWED_EMAILS` if AuthKit sign-up is open, so only the people you
list get past the callback.

## How it deploys

The Railway service `vantage-console` (project `vantage`, environment `production`) is
linked to this repo and redeploys on every push to `main`, so a merged PR is live a few
minutes later. Railway health-checks `/health` before switching traffic. Sign-in uses
the WorkOS staging environment for now (Console and hosted MCP); moving to WorkOS
production means swapping `WORKOS_API_KEY`/`WORKOS_CLIENT_ID` (and `AUTHKIT_DOMAIN` on
Horizon) and re-running `scripts/workos_setup.py` for the callback.

The hosted MCP server on Horizon uses the same allowlist: set `VANTAGE_ALLOWED_EMAILS`
and `WORKOS_API_KEY` there too (see docs/deploy-mcp-horizon.md).

## Check it

```bash
curl https://<url>/health            # {"status":"ok",...}
curl https://<url>/api/config        # 401 until signed in (sign-in is on)
```

Open `https://<url>/`: you land on the AuthKit page, then the Console with the
chat docked on the right (a button on narrow screens). Ask one of the suggested
questions; each tool call shows as a row, and runs link to Runs & Library.

## Cost notes

- Railway Hobby covers one small always-on service; the app idles at about 200 MB.
- Each chat question is a few Claude requests (one per tool round). The system
  prompt and tool list are cached, so repeat questions cost mostly output tokens.
  Set a monthly spend limit in the Claude Console.
- Monte Carlo runs execute in the container and write to MotherDuck; draws are
  capped by the tools (20,000 net risk, 5,000 Stars).

## Local

```bash
ANTHROPIC_API_KEY=... uv run vantage-api      # http://127.0.0.1:8000, chat on, no sign-in
```

Without `ANTHROPIC_API_KEY` the chat panel stays hidden and `/api/chat` returns 503.
