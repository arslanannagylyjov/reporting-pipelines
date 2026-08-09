# Session Notes

## 2026-08-10

- Set up repo scaffold: `README.md`, `docs/server-architecture.md`, `docs/credentials.md`, `docs/tables.md`, `scripts/sync_engine.py` (empty), `scripts/configs/` (empty).
- SSH'd into athena (`arslan@10.20.52.43`, key auth) and inspected live state rather than assuming:
  - `crontab -l` → one job, daily 02:30 run of `refresh_sales_snapshot.py`.
  - `docker ps -a` → 3 active containers (`reporting-db`, `metabase`, `metabase-postgres`) defined in `~/metabase-stack/docker-compose.yml`, plus 2 unrelated stopped `hello-world` containers.
  - Queried `reporting-db` directly (mysql client inside the container, `reporting_writer` creds) for table list, schema, row counts, and `SHOW GRANTS FOR CURRENT_USER()`.
  - Located the two application-level DB credentials: both are hardcoded in `~/reporting-scripts/refresh_sales_snapshot.py` on athena (`SOURCE` dict = ERP `metabase_ro`, `DEST` dict = `reporting_writer`), not in the compose `.env`.
  - Located the two container-level credentials (`REPORTING_DB_ROOT_PASSWORD`, `POSTGRES_PASSWORD`) in `~/metabase-stack/.env`, referenced from `docker-compose.yml`.
  - Found that the `reporting-db` root password currently in `.env` does **not** authenticate against the running container — flagged as an open item in `docs/server-architecture.md`, likely stale from before a volume/`.env` change.
- Wrote all findings into `docs/server-architecture.md`, `docs/credentials.md`, `docs/tables.md` — locations only, no secret values anywhere in the repo.
- No `.env` file created in this repo; none planned.

### Next steps

- Verify/rotate the `reporting-db` root password so `.env` on athena is actually correct.
- Decide whether `scripts/sync_engine.py` should replace `refresh_sales_snapshot.py`, and if so, move the two hardcoded DB credentials into an env-var/config pattern consistent with the rest of the stack.
