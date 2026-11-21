# Prefect flows (P-07)

There are two flows on Prefect Cloud (free Hobby tier, 2 of the 5 deployments).
They orchestrate only: GitHub Actions does the fetching, parsing and computing,
so Prefect never holds data, only a GitHub token.

| Deployment | Schedule (Pacific) | What happens |
|---|---|---|
| `ratebook-refresh` | 7 AM weekdays in April, and 7 AM on the 1st of each month | Cuts `refresh/auto-<stamp>` from main. The push runs "Refresh reference data". If data changed, it opens a PR for CI to test; otherwise it deletes the branch. |
| `stars-backtest` | 8 AM on the 2nd of each month | Dispatches "Stars back-test". That rebuilds the DB, re-runs the back-test and keeps the note as a private build artifact. |

```text
Prefect Cloud (schedule) ──GitHub API──▶ refresh/auto-* branch ──push──▶ Refresh workflow ──▶ PR ──▶ CI tests
                         └─GitHub API──▶ Stars back-test workflow ──▶ artifact (not published)
```

## One-time setup (about 10 minutes, on your Mac)

1. Create a fine-grained GitHub token at
   https://github.com/settings/personal-access-tokens/new. Settings:
   - repository access: only `mark64oswald/vantage`
   - permissions: Contents, Pull requests and Actions, each set to Read and write
   - an expiry you're comfortable with
2. In the repo folder, sign in to Prefect Cloud and create the work pool and the
   secret:

   ```bash
   uv run --with prefect prefect cloud login
   uv run --with prefect prefect work-pool create vantage-managed --type prefect:managed
   uv run --with prefect python -c "from prefect.blocks.system import Secret; \
   Secret(value=input('GitHub token: ')).save('github-token', overwrite=True)"
   ```

3. Deploy both flows from `prefect.yaml`:

   ```bash
   uv run --with prefect prefect deploy --all
   ```

4. Run each flow once by hand to finish P-07:

   ```bash
   uv run --with prefect prefect deployment run 'ratebook-refresh/ratebook-refresh'
   uv run --with prefect prefect deployment run 'stars-backtest/stars-backtest'
   ```

If a command's flags differ in your Prefect version, `--help` on that command
shows the current ones.
