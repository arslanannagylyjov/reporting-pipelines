# Server Architecture — athena (10.20.52.43)

Inspected live via SSH on 2026-08-10. Reflects actual server state, not assumptions.

## Containers

All defined in `~/metabase-stack/docker-compose.yml` on athena, on a shared `metanet` bridge network.

| Container | Image | Ports | Restart policy | Purpose |
|---|---|---|---|---|
| `reporting-db` | mysql:8.0 | 127.0.0.1:3306→3306 (host-local only) | unless-stopped | Holds the `reporting` database (`sales_snapshot`, `sales_staging`) |
| `metabase` | metabase/metabase:latest | 0.0.0.0:3000→3000 | unless-stopped | Dashboards, reads from `reporting-db` |
| `metabase-postgres` | postgres:16 | internal only (5432) | unless-stopped | Metabase's own app DB (users, dashboards, saved questions) — **not** reporting data |

Two stopped `hello-world` containers also exist (`optimistic_moser`, `gracious_booth`), leftover test runs — not part of the pipeline, safe to ignore/prune.

## Cron (crontab -l, user `arslan`)

```
30 2 * * * /usr/bin/python3 /home/arslan/reporting-scripts/refresh_sales_snapshot.py >> /home/arslan/reporting-scripts/refresh.log 2>&1
```

Runs daily at **02:30** server time. This is the only scheduled job on athena.

## Data flow

1. **ERP MySQL** (`10.20.52.11:13989`, database `cansun`, read-only user `metabase_ro`) — source of truth for sales data.
2. `refresh_sales_snapshot.py` reads from ERP in batches, stages rows into `reporting.sales_staging` on `reporting-db` (via `reporting_writer`).
3. Staged rows are merged into `reporting.sales_snapshot`.
4. `sales_staging` is cleared after each run (it's a transient landing table — 0 rows at rest is expected).
5. **Metabase** (container `metabase`, port 3000) queries `reporting-db` directly to build dashboards.

Last recorded run (from `refresh.log`): staged and merged 64,451 rows successfully, staging table cleared afterward — pipeline is healthy as of last run.

## Tables

See `docs/tables.md` for full schema. Summary:

| Table | Rows (as of 2026-08-10) | Role |
|---|---|---|
| `sales_snapshot` | 675,810 | Durable synced sales data, read by Metabase |
| `sales_staging` | 0 | Transient landing table for each sync run |

## Grants — `reporting_writer`

```
GRANT USAGE ON *.* TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DELETE ON `reporting`.`sales_snapshot` TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DELETE, DROP ON `reporting`.`sales_staging` TO `reporting_writer`@`%`
```

`reporting_writer` is scoped to the `reporting` database only — no access elsewhere on the MySQL instance. No `UPDATE` grant on `sales_snapshot`; the sync logic relies on stage-then-merge rather than in-place updates.

## Open item

The `reporting-db` root password currently set in `~/metabase-stack/.env` (`REPORTING_DB_ROOT_PASSWORD`) did **not** authenticate against the running container during this inspection. MySQL only applies `MYSQL_ROOT_PASSWORD` on first volume initialization, so the container's data volume was likely created with an earlier password before the current `.env` value was written. Root access via that `.env` value should not be assumed to work — verify before relying on it.
