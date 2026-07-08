# Claude-SSH-reporter

Automated daily/weekly reporting pipelines. Two independent pipelines live side by side in this repo, each with its own data, reports, and scripts:

- [`Monnit/`](Monnit/README.md) — 16-room Monnit home sensor network (temperature, humidity, dewpoint, condensation risk, current/power draw).
- [`GivenEnergy/`](GivenEnergy/README.md) — GivenEnergy inverter energy flows (solar generation, battery charge/discharge, grid import/export, home consumption).

Both follow the same shape: a scheduled GitHub Actions workflow fetches new data, dedups it into a JSONL history file, generates a self-contained HTML report (charts embedded as base64 PNGs), and commits/pushes the result. See each pipeline's own README for specifics.

No ongoing manual intervention or Claude API usage is required for the mechanical pipelines — they run autonomously. AI-narrative commentary (where present) is added by a locally scheduled routine, not a GitHub Action — see the relevant pipeline's CLAUDE.md.
