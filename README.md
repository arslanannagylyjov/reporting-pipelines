# reporting-pipelines

Documentation and sync tooling for Cansun's sales reporting pipeline: ERP MySQL → `reporting-db` (MySQL, on **athena**) → Metabase dashboards.

## Contents

- `docs/server-architecture.md` — containers, cron schedule, and data flow on athena
- `docs/credentials.md` — what each credential is for and where it lives (no secret values)
- `docs/tables.md` — schema of the `reporting` database tables
- `scripts/sync_engine.py` — scaffold for a future consolidated sync script
- `scripts/configs/` — non-secret config files for sync tooling
- `session-notes.md` — running log of work sessions

## Server

- Host: `athena` (`10.20.52.43`), SSH key auth as `arslan`
- Reporting stack lives in `~/metabase-stack/` (docker-compose)
- Current sync script lives in `~/reporting-scripts/refresh_sales_snapshot.py`

See `docs/server-architecture.md` for details.
