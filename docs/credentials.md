# Credentials — Inventory and Locations

No secret values are recorded in this file or anywhere in this repo, ever — location pointers only. Verified live on athena; last updated 2026-08-10.

## `reporting_writer` (MySQL user, `reporting-db` container)

- **For:** write access used by the sync script to stage and merge ERP sales data into the `reporting` database on athena.
- **Located:** `~/reporting-scripts/.env` on athena, as `REPORTING_DB_PASSWORD`, loaded via `os.environ` in `/home/arslan/reporting-scripts/refresh_sales_snapshot.py` and (as of 2026-08-10) also `refresh_customer_last_price.py` — same `.env`, same loader pattern, no new credential added. `.env` permissions are `600` (owner-only); a `.gitignore` in that directory excludes it in case the folder is ever git-tracked separately.
- Not present in `~/metabase-stack/docker-compose.yml` or `~/metabase-stack/.env` — this user was created directly in MySQL (via `GRANT`), not provisioned through compose env vars.
- **Caveat:** `refresh_sales_snapshot.py.bak` (a pre-existing backup, older than this migration) may still hold this credential hardcoded in plaintext — pending Arslan's decision on deletion.
- **Pending:** rotation — Arslan will perform this himself.

## ERP MySQL — `metabase_ro` (host `10.20.52.11:13989`, db `cansun`)

- **For:** read-only source connection the sync script uses to pull sales data from the ERP database.
- **Located:** `~/reporting-scripts/.env` on athena, as `ERP_DB_PASSWORD`, loaded via `os.environ` in the same script (and `refresh_customer_last_price.py`, as of 2026-08-10). Same file/permissions as above.
- **Caveat:** same `refresh_sales_snapshot.py.bak` note as `reporting_writer`, above.
- **Pending:** rotation — Arslan will perform this himself.

## `metabase_reporting_ro` (MySQL user, `reporting-db` container — Metabase's own read connection)

- **For:** the credential Metabase itself uses to connect to `reporting-db` and query `sales_snapshot`/`customer_last_price` for dashboards and questions. Distinct from the ERP `metabase_ro` above (different host, different account). Renamed from `metabase_ro` to `metabase_reporting_ro` on 2026-08-18 to end the shared-name coincidence with the ERP account.
- **Located:** inside Metabase's own encrypted app-DB (the `metabase-postgres` container), configured via the Metabase admin UI (`Admin → Databases → metabase_reporting_db → Edit connection details`), not in any `.env` file in this repo or in `~/reporting-scripts/`. Discovered/documented on 2026-08-10 while wiring up `customer_last_price` — previously undocumented here.
- **Grants (on `reporting-db`):** `SELECT` on `sales_snapshot` (pre-existing) and, as of 2026-08-10, `SELECT` on `customer_last_price` (added by Arslan directly, root access, after the new table wasn't visible in Metabase post-sync). No grant on `sales_staging` — intentional, that table is transient/internal and shouldn't be Metabase-visible.
- **Pending:** exact host mask (`%` vs specific) and password not verified/rotated by Claude — same "Arslan handles this himself" pattern as the other credentials in this file.

## `reporting-db` MySQL root — `root`@`localhost`

- **For:** root/admin access to the `reporting-db` MySQL container.
- **Located:** `~/metabase-stack/docker-compose.yml` on athena, `reporting-db` service → `environment` section, as `${REPORTING_DB_ROOT_PASSWORD}`. Actual value is in `~/metabase-stack/.env` on athena.
- **Status:** now password-protected. Previously `root`@`%` existed with no password set at all (see `docs/server-architecture.md`). Arslan fixed this directly on athena on 2026-08-10 — a real password is now set, root access is restricted to `root`@`localhost`, and `docker-compose.yml` is back to its normal (non-recovery) state. Value was set directly on the server and was not seen or handled by Claude.

## Metabase app-DB Postgres password

- **For:** Postgres password backing Metabase's own application database (dashboards, saved questions, users) — distinct from the reporting sales data.
- **Located:** `~/metabase-stack/docker-compose.yml` on athena, used by both the `postgres` service (`POSTGRES_PASSWORD`) and the `metabase` service (`MB_DB_PASS`), as `${POSTGRES_PASSWORD}`. Actual value is in `~/metabase-stack/.env` on athena.
