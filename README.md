# reporting-pipelines

Documentation and sync tooling for Cansun's sales reporting pipeline: ERP MySQL → `reporting-db` (MySQL, on **athena**) → Metabase dashboards.

## Contents

- `docs/server-architecture.md` — containers, cron schedule, and data flow on athena
- `docs/credentials.md` — what each credential is for and where it lives (no secret values)
- `docs/tables.md` — schema of the `reporting` database tables
- `docs/monitoring.md` — silent-failure job monitoring: registry format, log schema, Telegram report
- `scripts/sync_engine.py` — scaffold for a future consolidated sync script (not yet adopted; per-table scripts remain the live pattern)
- `scripts/configs/` — non-secret config files for sync tooling
- `session-notes.md` — running log of work sessions

## Server

- Host: `athena` (`10.20.52.43`), SSH key auth as `arslan`
- Reporting stack lives in `~/metabase-stack/` (docker-compose)
- Sync scripts live in `~/reporting-scripts/`: `refresh_sales_snapshot.py` (sales_snapshot, incremental stage-then-merge) and `refresh_customer_last_price.py` (customer_last_price, full replace)
- Both sync scripts log a structured run record (ok/fail, rows, duration) to `~/reporting-scripts/job_runs.csv`; `report_job_status.py` (cron'd 07:00 daily) reads `monitored_jobs.yml` and sends a Telegram summary — flags jobs that didn't run at all as well as ones that failed. See `docs/monitoring.md`.

See `docs/server-architecture.md` for details.
