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
45 2 * * * /usr/bin/python3 /home/arslan/reporting-scripts/refresh_customer_last_price.py >> /home/arslan/reporting-scripts/refresh_customer_last_price.log 2>&1
0 7 * * * /usr/bin/python3 /home/arslan/reporting-scripts/report_job_status.py >> /home/arslan/reporting-scripts/report_job_status.log 2>&1
```

Two scheduled jobs on athena, kept deliberately separate (not chained): `sales_snapshot` at **02:30**, `customer_last_price` at **02:45**. A third job, `report_job_status.py` (added 2026-08-11), runs at **07:00** and sends a Telegram summary of the night's runs, flagging any job that failed or never ran at all — see `docs/monitoring.md`.

## System timezone fix (2026-08-12)

**Problem:** athena's system timezone was `Etc/UTC`, three hours behind Istanbul. Every cron time above is a wall-clock local-time spec, so the jobs were firing 3 hours later than intended in real Istanbul time — `sales_snapshot` (cron'd 02:30) actually ran at 05:30 Istanbul, and the 07:00 report fired at 10:00 Istanbul.

**Fix:** changed the *system* timezone rather than offsetting the crontab times, so the existing `30 2`, `45 2`, `0 7` specs now mean what they look like. Arslan ran `sudo timedatectl set-timezone Europe/Istanbul` directly on athena (root-level change, not something Claude has sudo for). Confirmed via `timedatectl`:

```
Time zone: Europe/Istanbul (+03, +0300)
```

Local time now tracks real Istanbul time (Turkey has used a fixed +03 offset with no DST since 2016, so this doesn't need seasonal revisiting). No crontab edit was needed or made.

**Pre-change investigation (before touching anything):** checked every cron/timer source and script on the box for UTC assumptions that a timezone flip could silently break:
- The four reporting scripts (`refresh_sales_snapshot.py`, `refresh_customer_last_price.py`, `job_logging.py`, `report_job_status.py`) have no hardcoded UTC/`pytz`/`tzinfo` logic — they use `datetime.now()`/`date.today()`, which simply follow the host's system timezone. This is exactly the mechanism the fix relies on; no code changes were needed.
- `reporting-db`'s MySQL has no scheduled `EVENTS` that depend on time.
- The sync scripts never call `NOW()`/`CURDATE()` against the destination database; the one `CURDATE()` filter in `refresh_sales_snapshot.py` runs against the separate external ERP server, untouched by anything on athena.
- System-level cron (`/etc/cron.d`, `/etc/crontab`) and systemd timers on the box are all stock Ubuntu maintenance jobs (`e2scrub`, `sysstat`, `apt-daily`, `logrotate`, `fstrim`, `man-db`, etc.) with no cross-system time coordination — a 3-hour local-time shift is harmless for these.
- Not checked: root's own crontab (requires `sudo`, which Claude doesn't have on this box) — low risk given everything else was stock, but not verified empty.

**Known follow-up, not yet fixed:** the Docker containers (`metabase`, `reporting-db`, `metabase-postgres`) do **not** inherit the host timezone — `docker-compose.yml` sets no `TZ` env var and mounts no `/etc/localtime`, and `docker exec ... date` inside all three still shows UTC after the host fix. `reporting-db`'s MySQL `time_zone` is `SYSTEM`, which resolves to the *container's* own OS clock (UTC), not the host's. Checked Metabase's own app-level timezone setting via its API rather than assuming: `report-timezone` is unset, and the effective `system-timezone`/`report-timezone-long`/`report-timezone-short` all read `"GMT"` — Metabase has no independent compensation and will keep displaying sync times, query timestamps, etc. 3 hours off from real Istanbul time. Fixing this would mean adding `TZ=Europe/Istanbul` to the compose file's environment (or `MB_REPORT_TIMEZONE` for Metabase specifically) and recreating the containers — a restart, which per Arslan's instruction needs his sign-off and a low-usage window before it happens. Not done as part of this fix.

**Verification:** rather than manually trigger a test run, waited for the real 02:30/02:45/07:00 cycle. [Pending as of 2026-08-12 — to be confirmed once the next cycle completes and the Telegram report is checked for 07:00–07:15 Istanbul delivery.]

## Data flow

**sales_snapshot** (incremental, stage-then-merge):
1. **ERP MySQL** (`10.20.52.11:13989`, database `cansun`, read-only user `metabase_ro`) — source of truth for sales data.
2. `refresh_sales_snapshot.py` reads from ERP in batches, stages rows into `reporting.sales_staging` on `reporting-db` (via `reporting_writer`).
3. Staged rows are merged into `reporting.sales_snapshot`.
4. `sales_staging` is cleared after each run (it's a transient landing table — 0 rows at rest is expected).

**customer_last_price** (full replace, no staging):
1. **ERP MySQL**, view `aa_customer_last_price` — pre-built, pre-deduplicated (one row per `Firma`/`HesapKodu`/`StokKodu`), granted to `metabase_ro`.
2. `refresh_customer_last_price.py` pulls all rows in one `SELECT`, `TRUNCATE`s `reporting.customer_last_price`, then loads via chunked `INSERT` (5,000 rows/batch) directly — no staging table, since the table is small (~65k rows) and the source is already deduplicated.

**Metabase** (container `metabase`, port 3000) queries `reporting-db` directly to build dashboards for both tables.

Last recorded run (from `refresh.log`): staged and merged 64,451 rows successfully, staging table cleared afterward — pipeline is healthy as of last run. `customer_last_price` first verified run (2026-08-10): 65,143 rows, matching the ERP view count exactly.

## Tables

See `docs/tables.md` for full schema. Summary:

| Table | Rows (as of 2026-08-10) | Role |
|---|---|---|
| `sales_snapshot` | 675,810 | Durable synced sales data, read by Metabase |
| `sales_staging` | 0 | Transient landing table for each sync run |
| `customer_last_price` | 65,143 | Durable synced last-price-per-customer data, read by Metabase (full replace each run) |

## Grants — `reporting_writer`

```
GRANT USAGE ON *.* TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DELETE ON `reporting`.`sales_snapshot` TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DELETE, DROP ON `reporting`.`sales_staging` TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DROP ON `reporting`.`customer_last_price` TO `reporting_writer`@`%`
```

`reporting_writer` is scoped to the `reporting` database only — no access elsewhere on the MySQL instance. No `UPDATE` grant on `sales_snapshot`; the sync logic relies on stage-then-merge rather than in-place updates. `customer_last_price` needs `DROP` (not `DELETE`) because `TRUNCATE TABLE` checks the `DROP` privilege in MySQL — this grant was applied by Arslan directly on athena on 2026-08-10 (root access, not seen or handled by Claude).

## Root account — resolved (2026-08-10)

The `reporting-db` root password set in `~/metabase-stack/.env` (`REPORTING_DB_ROOT_PASSWORD`) previously did not authenticate against the running container. Investigation found why: `root`@`%` had **no password set at all** at the time. This has since been addressed directly by Arslan on athena — root access is now restricted to `root`@`localhost` (the `root`@`%` account is no longer usable) and protected with a real password. `docker-compose.yml` has been returned to its normal (non-recovery) state.
