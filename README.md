# P&L Dashboard

A static, single-page dashboard showing current / month-to-date / year-to-date trades and
P&L. It reads **only** `snapshot.json` in this folder via `fetch()` — it has no database
connection, no API calls, and no build step. That means it can be deployed to any free static
host, completely independent of where the trading application itself runs.

## Regenerating the snapshot

From the repo root, with the app's venv set up (see [../OPERATIONS.md](../OPERATIONS.md)):

```bash
weekly-trader export-snapshot
```

This overwrites `dashboard/snapshot.json` with current data from the live SQLite database
(realized P&L on closed positions, open positions, trade fills, and the equity curve from
account snapshots — see `src/weekly_trader/services/dashboard_export.py` for exactly what's
computed and how). Nothing else in this folder needs to change; just redeploy or re-upload
`snapshot.json` afterwards. There's no live connection between the dashboard and the app, so
the numbers are only as fresh as the last time you ran this command.

Optional: write it elsewhere with `weekly-trader export-snapshot --out path/to/file.json`.

## Deploying (pick any free static host)

The whole deployable unit is this `dashboard/` folder (`index.html` + `snapshot.json`).

**Netlify (drag-and-drop, no account setup needed beyond signup)**
1. Go to [app.netlify.com/drop](https://app.netlify.com/drop)
2. Drag the `dashboard/` folder onto the page
3. You get a live URL immediately; re-drag the folder any time you regenerate `snapshot.json`

**GitHub Pages**
1. Commit `dashboard/` to a GitHub repo (this one or a separate one)
2. Repo Settings → Pages → deploy from branch → set the folder to `/dashboard`
3. Push updates to `snapshot.json` and Pages redeploys automatically

**Cloudflare Pages**
1. `npx wrangler pages deploy dashboard` (or connect the repo in the dashboard UI, build
   output directory = `dashboard`, no build command)
2. Re-run the same deploy command after regenerating `snapshot.json`

Any other static host (Vercel, Surge, S3+CloudFront, etc.) works the same way — there is
nothing server-side to configure.

## Auto-publishing from a standalone repo

To host only the dashboard on a public URL, with the app auto-committing `snapshot.json`
every time it updates so your static host redeploys, split this folder into its own git
repo and point the app at it:

1. **Create the standalone repo checkout** somewhere outside this repo, e.g.:
   ```bash
   mkdir ../rhood_trader_dashboard
   cp dashboard/index.html dashboard/README.md ../rhood_trader_dashboard/
   cd ../rhood_trader_dashboard
   git init -b main
   git add -A && git commit -m "Initial dashboard"
   ```
2. **Create the GitHub repo** (via the GitHub web UI, or `gh repo create` if you have the
   CLI installed) and add it as the remote:
   ```bash
   git remote add origin git@github.com:<you>/rhood-trader-dashboard.git
   ```
3. **Generate a deploy key** so the trading app can push here unattended, without using your
   personal SSH key or a broad personal access token:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/rhood_dashboard_deploy -N ""
   ```
   Add `~/.ssh/rhood_dashboard_deploy.pub` under the new repo's **Settings → Deploy keys**
   with **write access**. Then make the private key usable for this specific repo without
   affecting your other git pushes, e.g. in `~/.ssh/config`:
   ```
   Host github-rhood-dashboard
       HostName github.com
       User git
       IdentityFile ~/.ssh/rhood_dashboard_deploy
       IdentitiesOnly yes
   ```
   and set the remote to use that host alias: `git remote set-url origin
   git@github-rhood-dashboard:<you>/rhood-trader-dashboard.git`.
4. **Point the app at it** in `config/default.yaml` (or a local override):
   ```yaml
   dashboard:
     output_dir: /absolute/path/to/rhood_trader_dashboard
     publish_git: true
     publish_branch: main
   ```
   Every `write_snapshot` call (Sunday research, Friday review, daily health, execute-approved,
   and manual approvals) will now `git add -A`, commit, and `git push origin main` in that
   directory after writing `snapshot.json` — skipping the commit if nothing changed. Push
   failures are logged, not raised, so a broken deploy key never fails a trading job.
5. **Connect your static host** (Netlify, GitHub Pages, Cloudflare Pages, etc.) to the new
   GitHub repo so it redeploys on every push to `main`.

## Notes

- The page shows a warning banner if the loaded snapshot is more than a week old, so a stale
  deploy is obvious to whoever's looking at it.
- All P&L figures assume long-only positions (this project's `instrument_policy.yaml` denies
  shorting), computed as `(close_price - entry_fill_price) * quantity` per closed position.
- Because it's a flat JSON file, treat it like any other export: don't commit a snapshot that
  contains data you don't want public if you're deploying to a public static host without
  access controls.
