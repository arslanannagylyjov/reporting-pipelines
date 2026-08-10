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

### Credential hardening (same day, later)

- Moved the two application credentials (ERP `metabase_ro`, `reporting_writer`) out of the hardcoded script into `~/reporting-scripts/.env` on athena. Values were written file-to-file on the server and never printed to terminal. Script now loads them via `os.environ` (stdlib-only loader; skipped `python-dotenv` since athena's Python is externally-managed and installing it would've needed `sudo`). Added a `.gitignore` for `.env` in that directory.
- Verified with a connectivity-only dry run (real connections to both DBs with the new `.env` values, no data mutated) — both succeeded.
- Found a pre-existing `refresh_sales_snapshot.py.bak` (predates this work) that still has both credentials hardcoded in plaintext. Not deleted — pending Arslan's decision.
- Investigated the stale root password: `.bash_history` showed `root`@`localhost` had been set to an **empty password**; confirmed live with a read-only test. Reported to Arslan without changing anything, per instruction not to guess and overwrite.
- **Root password: fixed and confirmed complete.** Arslan set a real root password directly on athena (Claude never saw or handled the value). `root`@`%` (previously blank) is no longer usable; root access is now `root`@`localhost` only, with a real password. `docker-compose.yml` restored to normal (non-recovery) state. Docs updated accordingly.
- **ERP (`metabase_ro`) and `reporting_writer` password rotations are still pending** — Arslan will do these himself. Not to be touched by Claude.
- Deleted `refresh_sales_snapshot.py.bak` from `~/reporting-scripts/` on athena — confirmed removed (`ls` on the path now errors "No such file or directory"). That was the last place the old hardcoded credentials existed in plaintext outside of `.env`.

### Next steps (superseded — see below)

- Arslan to rotate the ERP and `reporting_writer` passwords himself; update `docs/credentials.md`/`docs/tables.md` context only if something structural changes, no values.
- Decide whether `scripts/sync_engine.py` should replace `refresh_sales_snapshot.py` going forward — still undecided; second pipeline below followed the existing per-table-script pattern instead.

## 2026-08-10 (continued) — customer_last_price pipeline

Added a second synced table: `customer_last_price` (each customer's last transaction price per product), sourced from a pre-built, pre-deduplicated ERP view `aa_customer_last_price` — no ranking/dedup logic needed in the sync script, unlike `sales_snapshot`.

- **Blocker found and resolved:** `customer_last_price` already existed on `reporting-db`, but `reporting_writer` had zero grants on it (confirmed via error 1142 "SELECT command denied," distinguishing "ungranted" from "doesn't exist" — 1146 would've meant the latter). Reported to Arslan with the exact `GRANT` statement needed; he applied it himself directly on athena (root access, not seen/handled by Claude): `GRANT SELECT, INSERT, DROP ON reporting.customer_last_price TO 'reporting_writer'@'%';`. `DROP` (not `DELETE`) is required because MySQL's `TRUNCATE TABLE` checks the `DROP` privilege.
- Verified ERP side first, independent of the blocker: view exists, `metabase_ro` already had SELECT, 65,143 rows, column order matches what was specified.
- Wrote `~/reporting-scripts/refresh_customer_last_price.py` on athena, following the same per-table-script pattern as `refresh_sales_snapshot.py` (same `.env` loader, same `SOURCE`/`DEST` connection dicts and credentials — no new secrets). Full replace, not incremental: `TRUNCATE` + chunked `INSERT` (5,000 rows/batch), no staging table (unnecessary at ~65k rows, per Arslan). Used an explicit column list on the source `SELECT` (rather than literal `SELECT *`) to avoid relying on positional column order — matches how `refresh_sales_snapshot.py` itself is written, and turned out to matter: the target table's physical column order doesn't match the view's.
- Pre-flight check before the real run: confirmed zero NULLs across all three source PK columns (`Firma`, `HesapKodu`, `StokKodu`) — the target table declares them `NOT NULL` as the primary key, so a NULL would have failed the insert.
- **Manual run verified end-to-end:** 65,143 rows fetched and inserted (14 batches), matching the ERP view count exactly. `TRUNCATE` succeeded cleanly (first real test of the new `DROP` grant). Spot-checked rows across all three companies (Almer, Cansun, Karacan) — Turkish-character fields, prices, dates, invoice numbers, and `FaturaD_ID` all matched source exactly (an initial mojibake reading was a `mysql` client charset display artifact from omitting `--default-character-set=utf8mb4`, not real corruption — confirmed by re-querying with the flag set). Target schema confirmed sane: PK `(Firma, HesapKodu, StokKodu)`, all 12 view columns present, `FaturaD_ID` correctly widened from source `int` to target `bigint`.
- Added cron entry: `45 2 * * *` (customer_last_price), kept separate from the `30 2 * * *` sales_snapshot job per instruction — not merged or chained.
- Updated `docs/tables.md` (new table schema), `docs/server-architecture.md` (cron, data flow, grants, row counts), `docs/credentials.md` (noted the new script reuses the same `.env` credentials, no new secrets), `README.md` (script inventory).

### Next steps

- Metabase UI wiring for `customer_last_price` — Arslan will kick this off himself (Playwright MCP) after reviewing the synced data.
- ERP/`reporting_writer` password rotation still pending, Arslan's own action item (unchanged from above).
- `scripts/sync_engine.py` consolidation still undecided — both pipelines now live as separate per-table scripts on athena (`refresh_sales_snapshot.py`, `refresh_customer_last_price.py`), not in this repo.
