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
30 2 * * *      /usr/bin/python3 /home/arslan/reporting-scripts/refresh_sales_snapshot.py >> /home/arslan/reporting-scripts/refresh.log 2>&1
45 2 * * *      /usr/bin/python3 /home/arslan/reporting-scripts/refresh_customer_last_price.py >> /home/arslan/reporting-scripts/refresh_customer_last_price.log 2>&1
0 3 * * *       /usr/bin/python3 /home/arslan/reporting-scripts/refresh_supplier_last_purchase.py >> /home/arslan/reporting-scripts/refresh_supplier_last_purchase.log 2>&1
15 3 * * *      /usr/bin/python3 /home/arslan/reporting-scripts/refresh_cheque_bond_maturity.py >> /home/arslan/reporting-scripts/refresh_cheque_bond_maturity.log 2>&1
30 3 * * *      /usr/bin/python3 /home/arslan/reporting-scripts/refresh_vault_movements_daily.py >> /home/arslan/reporting-scripts/refresh_vault_movements_daily.log 2>&1
30 3 1 * *      /usr/bin/python3 /home/arslan/reporting-scripts/bump_filter_defaults.py >> /home/arslan/reporting-scripts/bump_filter_defaults.log 2>&1
0 7 * * *       /usr/bin/python3 /home/arslan/reporting-scripts/report_job_status.py >> /home/arslan/reporting-scripts/report_job_status.log 2>&1
0 9-19 * * *    /usr/bin/python3 /home/arslan/reporting-scripts/refresh_vault_status.py >> /home/arslan/reporting-scripts/refresh_vault_status.log 2>&1
5 9-19 * * *    /usr/bin/python3 /home/arslan/reporting-scripts/refresh_vault_movements_hourly.py >> /home/arslan/reporting-scripts/refresh_vault_movements_hourly.log 2>&1
0 9-19/2 * * *  /usr/bin/python3 /home/arslan/reporting-scripts/refresh_cek_senet_portfoy.py >> /home/arslan/reporting-scripts/refresh_cek_senet_portfoy.log 2>&1
```

Ten scheduled jobs on athena as of 2026-08-24, grown from the original two — each was added incrementally alongside a new synced table; see `docs/monitoring.md` and `session-notes.md` for the history and reasoning behind each addition's cron time. Grouped by cadence:

- **Nightly batch (02:30–03:30):** `sales_snapshot` → `customer_last_price` → `supplier_last_purchase` → `cheque_bond_maturity` → `vault_movements_daily`, each offset ~15 minutes past the previous based on that job's own observed run duration (all finish in well under a minute). `bump_filter_defaults.py` shares the 03:30 slot but only actually fires on the 1st of the month.
- **07:00 daily digest:** `report_job_status.py` sends one Telegram summary covering only the four jobs registered in `monitored_jobs.yml` (`sales_snapshot`, `customer_last_price`, `supplier_last_purchase`, `cheque_bond_maturity`) — see `docs/monitoring.md` for why every other job below is deliberately excluded from that registry (their schedules don't fit a "ran once overnight" daily-freshness check) and instead use failure-only or per-run Telegram alerting of their own.
- **Intraday, 09:00–19:00:** `vault_status` (hourly), `vault_movements_hourly` (hourly, offset 5 minutes past `vault_status` so their ERP connections don't fire in the same instant), `cek_senet_portfoy` (every 2 hours, same top-of-hour slots as `vault_status` — confirmed harmless, since the two hit entirely different source views/destination tables).

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

**Newer pipelines** (`supplier_last_purchase`, `cheque_bond_maturity`, `vault_status`, `vault_movements`, `cek_senet_portfoy`, added 2026-08-14 through 2026-08-23) follow the same ERP-view → replace-strategy-per-table → Metabase pattern, each with its own dedicated script on athena — see `docs/tables.md` for each table's sourcing and replace strategy, and `docs/monitoring.md` for each script's schedule and alerting.

## Tables

See `docs/tables.md` for full schema and current row counts (each table's own "as of" verification date). Summary:

| Table | Rows (as of) | Role |
|---|---|---|
| `sales_snapshot` | 675,810 (2026-08-10) | Durable synced sales data, read by Metabase |
| `sales_staging` | 0 | Transient landing table for each sync run |
| `customer_last_price` | 65,143 (2026-08-10) | Durable synced last-price-per-customer data, read by Metabase (full replace each run) |
| `supplier_last_purchase` | 29,351 (2026-08-14) | Each supplier's last purchase per product, all three companies (upsert-then-prune) |
| `cheque_bond_maturity` | 222 (2026-08-19) | Cheque/bond maturity schedule, Cansun + Almer (full-replace-by-key) |
| `vault_status` | 3 (2026-08-21) | Live snapshot of the three cash vault balances, hourly `REPLACE INTO` |
| `vault_movements` | 1,306+ (2026-08-21 backfill, grows daily) | Vault transaction history behind `vault_status`, two sync jobs (hourly window + nightly full replace) |
| `cek_senet_portfoy` | 10 (2026-08-23) | Outstanding çek/senet portfolio (full-replace-by-key with a pre-flight duplicate check) |

## Grants — `reporting_writer`

```
GRANT USAGE ON *.* TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DELETE ON `reporting`.`sales_snapshot` TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DELETE, DROP ON `reporting`.`sales_staging` TO `reporting_writer`@`%`
GRANT SELECT, INSERT, DROP ON `reporting`.`customer_last_price` TO `reporting_writer`@`%`
```

`reporting_writer` is scoped to the `reporting` database only — no access elsewhere on the MySQL instance. No `UPDATE` grant on `sales_snapshot`; the sync logic relies on stage-then-merge rather than in-place updates. `customer_last_price` needs `DROP` (not `DELETE`) because `TRUNCATE TABLE` checks the `DROP` privilege in MySQL — this grant was applied by Arslan directly on athena on 2026-08-10 (root access, not seen or handled by Claude).

**Every table added since (`supplier_last_purchase`, `cheque_bond_maturity`, `vault_status`, `vault_movements`, `cek_senet_portfoy`) has `SELECT, INSERT, UPDATE, DELETE` but no `DROP`** — confirmed live for each one via a failed `TRUNCATE` attempt (`1142 (42000): DROP command denied`) before its sync script was written to use `DELETE`+`INSERT`, `REPLACE INTO`, or upsert-then-prune instead (see `docs/tables.md` for which strategy each table uses). No grants have been changed to work around this on any of them.

## Root account — resolved (2026-08-10)

The `reporting-db` root password set in `~/metabase-stack/.env` (`REPORTING_DB_ROOT_PASSWORD`) previously did not authenticate against the running container. Investigation found why: `root`@`%` had **no password set at all** at the time. This has since been addressed directly by Arslan on athena — root access is now restricted to `root`@`localhost` (the `root`@`%` account is no longer usable) and protected with a real password. `docker-compose.yml` has been returned to its normal (non-recovery) state.
