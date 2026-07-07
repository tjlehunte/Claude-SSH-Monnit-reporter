# CLAUDE.md

Operational notes for working on this repo — not a description of what the code does (see README.md and the scripts themselves for that), just the gotchas that aren't obvious from reading the code.

## Verifying changes

This pipeline runs unattended, so a change that "looks right" but hasn't actually been run is not verified. After editing any script:

1. Push to `main`.
2. Trigger the relevant workflow manually: `POST /repos/tjlehunte/Claude-SSH-Monnit-reporter/actions/workflows/{daily-report.yml,weekly-report.yml}/dispatches` with `{"ref":"main"}` (add `"inputs":{"backfill_hours":"720"}` for a historical backfill).
3. Poll `GET .../actions/workflows/{name}/runs?per_page=1` until `status: completed`, then check `conclusion` and pull the job logs (`.../actions/jobs/{id}/logs`) for errors — don't assume success from a green checkmark alone if you changed core logic.
4. Pull the result and check the actual output (window dates, which room won a highlight figure, etc.) against what you expected.

## Room exclusion constants (`scripts/sensor_utils.py`)

Different "highlight" figures deliberately exclude different rooms — this is intentional, not inconsistent:

- `PEAK_TEMPERATURE_EXCLUDE` / `HUMIDITY_HIGHLIGHT_EXCLUDE` = `NON_LIVING_ROOMS` = `{Outside, Loft 1, Loft 2, Network}` — these would otherwise always win peak/lowest temperature and humidity, making the figure meaningless.
- `CONDENSATION_HIGHLIGHT_EXCLUDE` = `{Outside}` only — lofts and Network stay eligible for "tightest condensation margin."
- `HOUSE_INTERIOR_EXCLUDE` = `{Loft 1, Loft 2, Outside}` — used for the whole-house mean temperature/humidity; Network stays *included* here (it's indoors, just not a living space).

All of these only affect the mechanical "highlight" figures (Summary bullets, scalar stats.json fields feeding the AI insights). The charts always plot all 16 sensors — the condensation-margin bar chart colors bars by category (room/loft/outside) specifically so viewers know not to compare them directly.

## Weekly AI-insights placeholder

`generate_weekly_report.py` always resets `reports/weekly/latest.html`'s AI-insights section to `<!-- AI_INSIGHTS_PLACEHOLDER --><p><em>Not yet generated.</em></p>` on every run — the actual narrative is filled in separately by a **locally scheduled Claude Desktop task** (`monnit-weekly-ai-insights`, not part of this repo, not a GitHub Action), which runs Monday 09:00 local time after the weekly workflow. If you regenerate the weekly report while testing and want to see the filled-in version, you'll need to either wait for that routine or manually replace the placeholder (the HTML file is too large for line-based diff tools since it embeds base64 PNG charts — use a literal string replace, e.g. `perl -i -0pe 's/\Q$old\E/$new/'`, not the Edit tool).

## Git push on Windows (this dev machine)

Pushing with a token embedded in the URL (`https://<token>@github.com/...`) does **not** avoid Windows Git Credential Manager on this machine — GCM is registered at the system gitconfig level and will still intercept the push and try to open a browser OAuth prompt. Always push with:

```
git -c credential.helper= push "https://<token>@github.com/tjlehunte/Claude-SSH-Monnit-reporter.git" main
```

with `GIT_TERMINAL_PROMPT=0` set first. If a push hangs or fails confusingly, validate the token first: `curl -H "Authorization: token $TOKEN" https://api.github.com/user`.

## Data source specifics

- Monnit gateway: `https://146.87.171.55/json/...`, self-signed cert (`verify=False`), `NetworkID=6` (the "SSH" network), headers `APIKeyID`/`APISecretKey` from the `MONNIT_API_KEY_ID`/`MONNIT_API_SECRET_KEY` repo secrets.
- Gateway date-range queries error on large single-shot windows (returns a plain string instead of a message list) — `fetch_and_append.py` chunks any request into ≤5-day spans (`CHUNK_DAYS`) and defensively skips malformed chunks rather than crashing.
- Falls back to the Render proxy (`https://monnit-plumber-api.onrender.com/data`, no auth) only if the gateway is unreachable — the proxy only ever exposes a ~36h rolling window, so it can't satisfy a `BACKFILL_HOURS` request; backfill mode fails loudly rather than silently returning a too-small window.

## Concurrency

`daily-report.yml` and `weekly-report.yml` share one concurrency group (`monnit-reporter-writes`) so GitHub Actions queues them instead of racing on `git push` — this was a real bug (non-fast-forward push rejection) before the group was unified. Both also do `git pull --rebase` before pushing as defense-in-depth against the local AI-insights routine pushing at an unpredictable time outside GitHub Actions' concurrency control.
