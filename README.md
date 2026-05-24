# 547trends

Daily production dashboard for HI-A547-C, sourced from Arena Energy DPRs.

- `index.html` — dashboard (live at https://clayh53.github.io/547trends/)
- `arena_production.csv` / `.json` — extracted daily allocated volumes
- Data refreshes via a local Mac launchd job that pulls new PDFs from Gmail,
  extracts the values, and pushes here.

## Pipeline overview

```
Arena email  ──▶  Gmail API  ──▶  Dropbox folder (PDFs only)
                                          │
                                          ▼
                              extract_arena.py  ──▶  ~/code/arena-pipeline/output/  (CSV / JSON)
                                          │
                                          ▼
                              git push  ──▶  this repo  ──▶  GitHub Pages
```

Trigger: launchd fires at login and every 4 hours of uptime.

## File locations

| What | Where |
|---|---|
| Scripts and venv | `~/code/arena-pipeline/` |
| PDFs (input) | `~/Library/CloudStorage/Dropbox/Arena Production Reports/` |
| CSV / JSON outputs | `~/code/arena-pipeline/output/` |
| Git working copy of this repo | `~/code/547trends/` |
| launchd plist | `~/Library/LaunchAgents/com.clay.arena-daily-pull.plist` |
| Main log | `~/Library/Logs/arena_daily_pull.log` |
| stderr log | `~/Library/Logs/arena_daily_pull.stderr.log` |
| OAuth credentials | `~/.config/arena-gmail/` |

Scripts AND CSV/JSON outputs live outside `~/Library/CloudStorage/` because
macOS TCC blocks launchd-spawned bash processes from accessing CloudStorage
paths. PDFs themselves still live in Dropbox — Python reads them fine, only
bash is restricted.

## Troubleshooting

### 1. Did the latest run succeed?

```bash
launchctl list | grep arena
```

Format: `<PID>  <exit-code>  <label>`.
A `0` exit code is success. Anything else points at a failure — see step 3.

### 2. What does the log say?

```bash
tail -40 ~/Library/Logs/arena_daily_pull.log
```

Look for the most recent `arena_daily_run.sh starting` line and read down.
A clean run ends in either `Pushed update for latest production date YYYY-MM-DD`
or `No data changes — nothing to push.`.

### 3. Did the script crash before logging?

```bash
tail -20 ~/Library/Logs/arena_daily_pull.stderr.log
```

This catches errors that happen before the Python script can write its own
log — e.g. missing executable bit, TCC denial, wrong interpreter.

### 4. Did a new PDF actually land?

```bash
ls -lt "/Users/clay/Library/CloudStorage/Dropbox/Arena Production Reports/"*.pdf | head -3
```

If the latest PDF date doesn't match yesterday's email date, the Gmail pull
isn't reaching new mail — re-check OAuth or the Gmail search query in
`arena_daily_pull.py`.

### 5. Is the launchd agent still registered?

```bash
launchctl print gui/$(id -u)/com.clay.arena-daily-pull 2>&1 | head -30
```

Shows last exit time, last exit code, current state. If the agent isn't
listed at all, reload it:

```bash
launchctl unload ~/Library/LaunchAgents/com.clay.arena-daily-pull.plist
launchctl load   ~/Library/LaunchAgents/com.clay.arena-daily-pull.plist
```

## Manual recovery

If automation is broken and you just need today's data on the dashboard:

```bash
# Run the whole pipeline by hand
~/code/arena-pipeline/arena_daily_run.sh
tail -20 ~/Library/Logs/arena_daily_pull.log
```

If the extractor needs to run independently of the Gmail pull
(e.g. you dropped PDFs in manually):

```bash
~/code/arena-pipeline/.venv/bin/python ~/code/arena-pipeline/extract_arena.py
~/code/arena-pipeline/arena_daily_run.sh   # picks up the CSV diff and pushes
```

## Re-authenticating with Gmail

If the OAuth token gets revoked (e.g. password change, security review,
extended inactivity), delete the cached token and run the pull script
interactively so it can prompt for a fresh browser sign-in:

```bash
rm ~/.config/arena-gmail/token.json
~/code/arena-pipeline/.venv/bin/python ~/code/arena-pipeline/arena_daily_pull.py
```

## Common failure modes

| Symptom in stderr / log | Likely cause | Fix |
|---|---|---|
| `Operation not permitted` reading from CloudStorage path | macOS TCC blocked launchd | Scripts must live outside `~/Library/CloudStorage/` |
| `Missing Google client libs` | venv missing deps | `~/code/arena-pipeline/.venv/bin/pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib pypdf pdfplumber` |
| `Extractor not found at /path/...` | Path drift after a move | Update the `EXTRACTOR` path in `arena_daily_pull.py` |
| `git push` fails | SSH agent not loaded under launchd | The plist runs as you — keys in `~/.ssh/` should work; if not, run `ssh-add ~/.ssh/id_ed25519` and try again |
| Dashboard shows stale data | `git push` succeeded but Pages hasn't rebuilt | Wait 1–2 min; check the **Actions** tab on GitHub for a green check on "pages build and deployment" |
| `launchctl list \| grep arena` shows non-zero exit, but manual run works | Loaded plist points at old path | Edit `~/Library/LaunchAgents/com.clay.arena-daily-pull.plist`, then `launchctl unload && load` it |

## What gets shared publicly

The repo is **public** so GitHub Pages can serve it without authentication.
This is fine because the data isn't confidential — but be mindful that
`arena_production.csv` and the dashboard URL are world-readable to anyone
who guesses or is told the URL `https://clayh53.github.io/547trends/`.
