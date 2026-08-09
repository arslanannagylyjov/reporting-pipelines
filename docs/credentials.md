# Credentials — Inventory and Locations

No secret values are recorded in this file or anywhere in this repo, ever — location pointers only. Verified live on athena on 2026-08-10.

## `reporting_writer` (MySQL user, `reporting-db` container)

- **For:** write access used by the sync script to stage and merge ERP sales data into the `reporting` database on athena.
- **Located:** hardcoded in the `DEST` connection dict inside `/home/arslan/reporting-scripts/refresh_sales_snapshot.py` on athena. File permissions are `600` (owner-only read/write).
- Not present in `~/metabase-stack/docker-compose.yml` or `~/metabase-stack/.env` — this user was created directly in MySQL (via `GRANT`), not provisioned through compose env vars.

## ERP MySQL — `metabase_ro` (host `10.20.52.11:13989`, db `cansun`)

- **For:** read-only source connection the sync script uses to pull sales data from the ERP database.
- **Located:** hardcoded in the `SOURCE` connection dict inside `/home/arslan/reporting-scripts/refresh_sales_snapshot.py` on athena. Same file/permissions as above.

## `reporting-db` MySQL root password

- **For:** root/admin access to the `reporting-db` MySQL container.
- **Located:** `~/metabase-stack/docker-compose.yml` on athena, `reporting-db` service → `environment` section, as `${REPORTING_DB_ROOT_PASSWORD}`. Actual value is in `~/metabase-stack/.env` on athena.
- **Note:** see `docs/server-architecture.md` "Open item" — this value did not authenticate against the running container during inspection; treat as unverified until confirmed.

## Metabase app-DB Postgres password

- **For:** Postgres password backing Metabase's own application database (dashboards, saved questions, users) — distinct from the reporting sales data.
- **Located:** `~/metabase-stack/docker-compose.yml` on athena, used by both the `postgres` service (`POSTGRES_PASSWORD`) and the `metabase` service (`MB_DB_PASS`), as `${POSTGRES_PASSWORD}`. Actual value is in `~/metabase-stack/.env` on athena.

## Observation

The two MySQL application credentials (`reporting_writer`, ERP `metabase_ro`) are hardcoded directly in the Python script rather than sourced from `~/metabase-stack/.env` like the container-level passwords are. The file is permission-locked to the owner, so this isn't an urgent exposure, but it's inconsistent with how the rest of the stack handles credentials — worth moving into an env-var or config-file pattern if `scripts/sync_engine.py` in this repo ever replaces the current script.
