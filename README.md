# reporting-pipelines

Documentation and sync tooling for Cansun's sales reporting pipeline: ERP MySQL → `reporting-db` (MySQL, on **athena**) → Metabase dashboards.

## Contents

- `docs/server-architecture.md` — containers, cron schedule, and data flow on athena
- `docs/credentials.md` — what each credential is for and where it lives (no secret values)
- `docs/tables.md` — schema of the `reporting` database tables
- `docs/monitoring.md` — silent-failure job monitoring: registry format, log schema, Telegram report; also the home for every sync/maintenance job's own schedule and alerting notes
- `docs/metabase-permissions.md` — Metabase groups, collections, and permission model (Director/Sales-Purchase/Other)
- `docs/adding-a-new-report.md` — standing playbook for the "new ERP view → synced table → Metabase question" pattern
- `scripts/sync_engine.py` — scaffold for a future consolidated sync script (not yet adopted; per-table scripts remain the live pattern)
- `scripts/configs/` — non-secret config files for sync tooling
- `session-notes.md` — running log of work sessions

## Server

- Host: `athena` (`10.20.52.43`), SSH key auth as `arslan`
- Reporting stack lives in `~/metabase-stack/` (docker-compose)
- Sync scripts live in `~/reporting-scripts/`, one per synced table (per-table-script pattern, not a shared consolidated script): `refresh_sales_snapshot.py` (sales_snapshot, incremental stage-then-merge-then-prune — the prune step, added 2026-08-25, removes stale rows left behind when an ERP document is corrected/reissued under a new ID), `refresh_customer_last_price.py` (customer_last_price, full replace), `refresh_supplier_last_purchase.py` (supplier_last_purchase, upsert-then-prune), `refresh_cheque_bond_maturity.py` (cheque_bond_maturity, full-replace-by-key), `refresh_vault_status.py` (vault_status, hourly), `refresh_vault_movements_hourly.py` / `refresh_vault_movements_daily.py` (vault_movements, two jobs for two freshness needs), `refresh_cek_senet_portfoy.py` (cek_senet_portfoy, every 2 hours), `refresh_stock_details.py` (stock_details, nightly full-catalog replace). Plus one non-sync monthly job, `bump_filter_defaults.py`, which re-points Metabase dashboard filter defaults at the current year/month via the API. See `docs/tables.md` for what each table holds and `docs/monitoring.md` for each script's exact schedule.
- All of the above log a structured run record (ok/fail, rows, duration) to `~/reporting-scripts/job_runs.csv`; `report_job_status.py` (cron'd 07:00 daily) reads `monitored_jobs.yml` and sends a Telegram summary for the nightly-batch jobs — flags jobs that didn't run at all as well as ones that failed. Jobs with an intraday/monthly cadence that doesn't fit a daily-freshness check (`vault_status`, `vault_movements_*`, `cek_senet_portfoy`, `bump_filter_defaults.py`) are deliberately excluded from that registry and instead alert on failure (or, for the monthly job, every run) directly to Telegram. See `docs/monitoring.md`.

See `docs/server-architecture.md` for details.
