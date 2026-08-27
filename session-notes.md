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

### Next steps (superseded — see below)

- Metabase UI wiring for `customer_last_price` — Arslan will kick this off himself (Playwright MCP) after reviewing the synced data.
- ERP/`reporting_writer` password rotation still pending, Arslan's own action item (unchanged from above).
- `scripts/sync_engine.py` consolidation still undecided — both pipelines now live as separate per-table scripts on athena (`refresh_sales_snapshot.py`, `refresh_customer_last_price.py`), not in this repo.

## 2026-08-10 (continued) — Metabase UI wiring for customer_last_price

Wired `customer_last_price` into Metabase (Playwright MCP driving Chrome, per Arslan's explicit instruction to use that instead of the gstack browser skill — gstack needed a one-time Playwright browser binary install that was still running when he redirected).

- **Blocker found and resolved:** after re-syncing the `metabase_reporting_db` schema, `customer_last_price` still didn't appear (neither did `sales_staging`, which normally doesn't need to). Root cause: Metabase's own DB connection to `reporting-db` uses a MySQL user called `metabase_ro` — a distinct, previously undocumented account (not `reporting_writer`, and not the same as the ERP `metabase_ro`, despite the shared name) — which had no grant on the new table. Reported to Arslan with the fix (`GRANT SELECT ON reporting.customer_last_price TO 'metabase_ro'@'%';`); he applied it directly on athena. Re-ran the schema sync afterward and the table appeared (as "Customer Last Price", table id 11). Documented this account in `docs/credentials.md` (new section) — location/purpose only, no values.
- **Access decision (per Arslan):** confirmed via `admin/people` and `admin/people/groups` that Metabase has exactly one real user (Arslan) and no custom groups — only the two defaults (Administrators, All Users), both containing just him. Since there's no sales-team group to scope permissions to yet, built and saved the question in the root **"Our analytics"** collection (not `Boss Dashboard`, not a new/custom collection) — the collection visible under the default "All Users" permission scope. No group created, no one invited, per explicit instruction.
- **Question built as a native SQL question**, not the GUI/MBQL notebook editor — this was a deliberate deviation, not literal to how Step 2 was worded. Reason: Metabase's GUI notebook-editor filters bake a fixed value into the saved query at save time; they can't be left open for a viewer to type into afterward outside of a dashboard context. The only way to get two independent, typed, partial-match filter *widgets* on a **single saved question** (dashboards explicitly out of scope) is native SQL with `{{variable}}` Field Filter template tags. Built by starting from the GUI notebook editor (to get exact column selection/order/sort right), then using Metabase's "Convert this question to SQL," then hand-editing the SQL and configuring two Field Filter variables:
  - `hesap_kodu` → mapped to `HesapKodu`, widget type "String contains", input box, single value, label "Müşteri Kodu"
  - `hesap_aciklamasi` → mapped to `HesapAciklamasi`, same widget type/shape, label "Müşteri"
  - Both left with no default and not required, so either can be used independently — verified by filtering on `hesap_kodu=01730` alone (65,143 rows → 443 rows) and clearing it back to unfiltered before saving.
- Columns: exactly the 7 requested, in the requested order (`HesapKodu, HesapAciklamasi, StokKodu, StokAciklamasi, DvzBirimFiyat, DovizKodu, BelgeTarihi`) — note this required an explicit `SELECT` column list, since the physical table column order differs from the requested display order.
- Column display labels renamed via visualization settings: HesapKodu → "Müşteri Kodu", HesapAciklamasi → "Müşteri", StokKodu → "Stok Kodu", StokAciklamasi → "Ürün", DvzBirimFiyat → "Fiyat", DovizKodu → "Döviz" (BelgeTarihi left as-is, not in the rename list).
- Sort: `HesapAciklamasi ASC, StokAciklamasi ASC`, as specified.
- Saved as **"Müşteri Son Fiyat Sorgusu"** (`/question/56-musteri-son-fiyat-sorgusu`) in "Our analytics."
- Screenshots taken of the final saved result and the open "Contains..." filter widget, sent to Arslan directly (not committed to the repo — they're point-in-time UI captures, not documentation).

### Next steps (superseded — see below)

- Metabase group/permissions work (creating a sales-team group, inviting users, deciding actual access scope) is still fully open — explicitly deferred by Arslan to a separate decision.
- ERP/`reporting_writer` password rotation still pending, Arslan's own action item (unchanged from above).
- `scripts/sync_engine.py` consolidation still undecided.

## 2026-08-11 — add Firma column + DovizKodu data-quality note

- Added `Firma` to "Müşteri Son Fiyat Sorgusu" (`/question/56-musteri-son-fiyat-sorgusu`), positioned right after Müşteri Kodu/Müşteri and before Stok Kodu, per Arslan's request. Edited the SQL editor directly (still a native SQL question, unchanged from how it was built originally), inserted `Firma` into the `SELECT` list in that position, left everything else — the two Field Filter variables (`hesap_kodu`, `hesap_aciklamasi`), the `ORDER BY`, and all existing column renames — untouched.
- Verified both filters still work **independently** after the change: filtered on Müşteri Kodu = "01730" alone → 443 rows; cleared it, filtered on Müşteri = "ABDALLAH" alone → same 443 rows (same customer, filtered by code vs. by name) — confirms each Field Filter resolves correctly on its own with the other left blank.
- Saved via "Replace original question" (not save-as-new), so it's still question id 56, same URL.
- Screenshot taken showing the new column in place; sent to Arslan directly (not committed — same pattern as prior screenshots).
- **Data quality note added to `docs/tables.md`** under `customer_last_price`: `DovizKodu` has inconsistent values for the same currency (`"EUR"` vs at least one `"EURO"`) — sourced from the ERP view as-is, not normalized by the sync pipeline by design. Documented as a known issue for future report-builders to handle at the query layer; explicitly not fixed in the pipeline or table, per Arslan's instruction not to normalize source data quality issues here.

### Next steps

- Metabase group/permissions work (creating a sales-team group, inviting users, deciding actual access scope) is still fully open — explicitly deferred by Arslan to a separate decision.
- ERP/`reporting_writer` password rotation still pending, Arslan's own action item (unchanged from above).
- `scripts/sync_engine.py` consolidation still undecided.
- `DovizKodu` normalization (EUR/EURO and possibly other variants) is an open item for whoever next builds a currency-grouped report — not scheduled, just documented.

## 2026-08-11 (continued) — silent-failure job monitoring

Built cron job monitoring since there was previously no failure alerting at all: a crashed or silently-not-running job would go unnoticed indefinitely.

- Added `~/reporting-scripts/monitored_jobs.yml` on athena — a small registry (name, script path, cron schedule, expected time) that the report script reads dynamically, so adding a future job means one new entry there plus a `run_job()` call in the new script, nothing in the report logic itself.
- Added `~/reporting-scripts/job_logging.py`: a stdlib-only (`csv`, no new dependency) helper. `run_job(job_name, func)` times `func()`, appends one row to `~/reporting-scripts/job_runs.csv` (`timestamp, job_name, status, rows, duration_seconds, error_message`), and on a crash logs `fail` with the exception before re-raising — so the existing per-script cron logs (`refresh.log`, `refresh_customer_last_price.log`) still get the full traceback exactly as before, unchanged.
- Modified both `refresh_sales_snapshot.py` and `refresh_customer_last_price.py` on athena: `main()` now returns its row count instead of nothing, and the `if __name__ == '__main__':` block calls `run_job('sales_snapshot'|'customer_last_price', main)`. No other change to either script's sync logic — same connections, same SQL, same batching.
- Wrote `~/reporting-scripts/report_job_status.py`: loads the registry, loads today's log entries (latest per job if a job ran more than once), and for each job emits `❌ DID NOT RUN` (no entry today — checked first, the priority case), `❌ FAILED — <error>` (latest entry is `fail`), or `✅ ok — <rows> rows in <duration>s` (latest is `ok`). Failures/missing jobs are listed before successes in one Telegram message, sent via a single HTTP POST to the Bot API using `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` from the existing `~/reporting-scripts/.env` (same file, same `_load_env` stdlib loader already used by the sync scripts — both variables were already present in `.env`, nothing new added). Added cron entry `0 7 * * *` for it.
- **All three required scenarios verified live on athena, not just read through:**
  - Forced failure: ran `refresh_sales_snapshot.py` with `ERP_DB_PASSWORD` overridden to a wrong value via the environment (real `.env` on disk untouched) — crashed with exit 1 and full traceback as before, and logged a `fail` row. Report correctly showed `❌ sales_snapshot: FAILED — ProgrammingError: 1045 ...` ahead of the other job's `✅`.
  - DID NOT RUN (the priority case, not skipped): commented out `customer_last_price`'s cron line, confirmed via `crontab -l`, then restored it — proving the disable/re-enable round-trips cleanly. Since cron wouldn't actually skip until the next real night, separately reproduced the exact resulting state (no log entry for that job today) by removing its rows from `job_runs.csv`, ran the report, confirmed `❌ customer_last_price: DID NOT RUN today (no log entry)`, then restored the real log from a backup taken before the test.
  - Clean night: ran both jobs for real (valid credentials), both logged `ok`, report showed both `✅` with correct row counts (65,068 / 65,204) and durations.
  - All test backups (`.pre-monitoring.bak`, `job_runs.csv.verify.bak`, crontab snapshots) deleted after verification; `job_runs.csv` on athena now reflects only real run history.
- Documented the whole system in `docs/monitoring.md` (registry format, log schema, how to add a job, verification summary); updated `README.md` and `docs/server-architecture.md` (cron table, file inventory) accordingly. Followed the existing repo convention of keeping live scripts on athena only — `job_logging.py`, `report_job_status.py`, and `monitored_jobs.yml` are not committed here, same as `refresh_sales_snapshot.py`/`refresh_customer_last_price.py` never have been.

### Next steps

- Metabase group/permissions work, ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, and `DovizKodu` normalization — all unchanged/open from above.
- No monitoring for `report_job_status.py` itself yet (i.e. nothing watches the watcher) — not requested, flagging only as a known gap.

## 2026-08-12 — system timezone fix (athena was on UTC, 3 hours behind Istanbul)

athena's system clock was set to `Etc/UTC`, so every cron job (`30 2`, `45 2`, `0 7`) was firing 3 hours later than intended in real Istanbul time — `sales_snapshot` cron'd for 02:30 was actually running at 05:30 Istanbul, and the 07:00 report fired at 10:00 Istanbul.

- Investigated for UTC dependencies before touching anything, per instruction: the four reporting scripts have no hardcoded UTC/`pytz` logic (they use `datetime.now()`/`date.today()`, which just follow host system time); `reporting-db` has no scheduled MySQL `EVENTS`; the sync scripts never call `NOW()`/`CURDATE()` against the destination DB (the one `CURDATE()` use is against the separate external ERP server); system-level cron/timers on the box are all stock Ubuntu maintenance jobs with no cross-system time coordination. Root's own crontab wasn't checked — needs `sudo`, which Claude doesn't have on this box (consistent with the standing pattern that root-level actions on athena are Arslan's own).
- **Confirmed a real follow-up issue while investigating, not yet fixed:** the three Docker containers don't inherit the host timezone — no `TZ` env var, no `/etc/localtime` mount in `docker-compose.yml`, and all three still show UTC via `docker exec ... date` even after the host fix. `reporting-db`'s MySQL `time_zone=SYSTEM` resolves to the *container's* clock, not the host's. Checked Metabase's own app-level timezone setting via its `/api/setting` endpoint rather than assuming: `report-timezone` is unset and the effective `system-timezone` reads `"GMT"` — Metabase has no independent compensation, so its displayed sync/query timestamps will keep reading 3 hours off from real Istanbul time until the compose file is updated (`TZ=Europe/Istanbul` or `MB_REPORT_TIMEZONE`) and the containers are recreated. That requires a restart, which per Arslan's instruction needs his go-ahead and a low-usage window — not done here.
- Blocked on step 2 (Claude has no sudo on athena) — reported findings and asked Arslan to run the command himself. He ran `sudo timedatectl set-timezone Europe/Istanbul` directly.
- Verified via `timedatectl`: `Time zone: Europe/Istanbul (+03, +0300)`, local time correctly tracking real Istanbul time. Crontab left untouched, as instructed — the existing `30 2`/`45 2`/`0 7` specs now mean what they look like.
- Considered scheduling a cloud routine (via the `/schedule` skill) to verify tomorrow's 07:00–07:15 Istanbul Telegram delivery automatically, but stopped before creating it: cloud routines run in Anthropic's sandboxed cloud infra with no access to local SSH keys or private networks, and athena is only reachable via the SSH key on Arslan's Mac — a cloud agent couldn't have reached it at all. Verification will instead happen the normal way, via SSH from this same environment, next time the conversation resumes after 2026-08-13 07:15 Istanbul.
- Updated `docs/server-architecture.md` with a new "System timezone fix" section covering the problem, the fix, the pre-change investigation, and the unresolved container-timezone follow-up.

### Next steps (superseded — see below)

- **Verify tomorrow's real cron cycle** (2026-08-13, after 07:15 Istanbul): check `job_runs.csv` timestamps for `sales_snapshot`/`customer_last_price` land in the 02:30–03:00ish Istanbul window (not 05:30), check `report_job_status.log` for a ~07:00 Istanbul run, and confirm the Telegram message itself arrived 07:00–07:15 Istanbul, not 10:00. Explicitly deferred — no manual test run, per instruction.
- Container timezone fix (Metabase/MySQL still on UTC internally) — needs `docker-compose.yml` changes and a container restart; needs Arslan's go-ahead and a low-usage window before scheduling.
- Metabase group/permissions work, ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, and `DovizKodu` normalization — all unchanged/open from above.

## 2026-08-12 (continued) — Metabase permission groups: Director/User, Boss Dashboard lockdown, cost-column exposure check

Added the first Metabase permission groups beyond the built-ins (`Administrators`, `All Users`): `Director` (group 5) and `User` (group 6, later renamed `Other`) — full detail in `docs/metabase-permissions.md`.

- `Director`: kept Metabase's default full ("Query builder and native") data access on `sales_snapshot`/`customer_last_price` — already matched the intended scope. `User`: explicitly set to "No" query-building access on both tables (the default would otherwise have been full access).
- Locked down the `Boss Dashboard` collection: `Director` → View, `User` → No access. **Found and fixed a real gap before it mattered:** `All Users` (every account's automatic group, no opt-out) had `Curate` (full edit) by default on this collection — since Metabase permissions are most-permissive-wins across a user's groups, this would have silently overridden `User`'s restriction the moment a real non-director person was added. Set `All Users` to No access.
- **Cost-column exposure check:** confirmed `FabrikaFiyati`/`FabrikaTutarUsd` (the two cost columns that must stay boss-only) aren't reachable from anywhere else in the instance — searched all collections/questions/trash for "Fabrika" (zero hits) and separately pulled every card's actual query definition via `/api/card/<id>`, since a GUI question's column selection wouldn't show up in a text search. All 14 cards then in the collection confirmed clean.
- Pure Metabase-side permission/collection work via Playwright MCP — no `.env` or credential changes.

## 2026-08-12 (continued) — homepage redirect bug found and fixed on first Director login

Director #1 (first real non-admin account, user ID 2) logged in and landed on the `Boss Dashboard` **collection** (a folder listing) instead of the `Cansun Satış Genel Bakış` **dashboard** (ID 3) that lives inside it.

- **Root cause:** `Admin > Settings > General > Homepage` was set to "Default Metabase home," which falls back to whichever collection a user can see — since `Director` only had View on the collection, not the root, that landed them on the folder instead of the dashboard. Confirmed dashboard ID 3 genuinely is `Cansun Satış Genel Bakış` before assuming this was a wrong-ID problem — it wasn't.
- **Fix:** set Homepage to Dashboard → `Cansun Satış Genel Bakış`. Verified via `/api/setting` (`custom-homepage: true`, `custom-homepage-dashboard: 3`) and confirmed live with Director #1's own refreshed session.
- **Terminology correction, same day:** "Boss Dashboard" had been used loosely for two different objects — the collection (folder, ID 6) and the actual dashboard (ID 3) inside it. Documented the distinction explicitly in `docs/metabase-permissions.md`, since the mixup is what caused this bug.
- Director #1 provisioned this same day: `Director` group only (confirmed via `/api/user`, not the admin UI label); temporary password relayed by Arslan directly (no SMTP on this instance, never recorded in this repo).

### Next steps (superseded — see below)

- Second Director account, `Sales/Purchase` group, and the full collection redesign — done next, 2026-08-13.
- ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, and `DovizKodu` normalization — all unchanged/open from above.

## 2026-08-13 — collections & permissions redesign: three collections, Sales/Purchase group, root + database lockdown

Full restructure from the single `Boss Dashboard` model to three purpose-built collections, done via Playwright MCP against the live instance. Full detail in `docs/metabase-permissions.md`.

- Renamed `User` → `Other` (group 6, unchanged zero-access config); created `Sales/Purchase` (group 7, no members yet, real people explicitly out of scope for this phase).
- Created three flat collections under root — `Yöneticiler` (9), `Satış/Satın Alma` (10), `Diğer` (11) — each with `Director`: View, plus the matching department group: View, everyone else: No access. Hit the same `All Users`-defaults-to-Curate gotcha as the Boss Dashboard fix and set it to No access on all three.
- Moved dashboard 3 (`Cansun Satış Genel Bakış`) into `Yöneticiler` and question 56 (`Müşteri Son Fiyat Sorgusu`) into `Satış/Satın Alma`. Deliberately left the old `Boss Dashboard` collection and its other 13 questions in place — dashboards reference cards by ID regardless of which collection the dashboard lives in, so nothing broke, but flagged it as a slightly orphaned leftover for a future cleanup pass.
- Locked the root collection (`Our analytics`) to No access for `All Users` — nothing can be saved loose at root going forward.
- Locked database-level query building to admins only: set `Director`/`Sales/Purchase`/`Other`/`All Users` all to "No" create-queries on `metabase_reporting_db`. This also hides the Databases sidebar entry for everyone non-admin.
- **Incident:** lost the live admin session mid-verification — logging a throwaway test account into the same browser tab invalidated it server-side. Regained access using Metabase's own reset-password CLI tool on the server; no other accounts or data were affected.
- Verified live with throwaway test accounts (created, checked, deactivated afterward) rather than trusting the permission graph alone.

### Same day, second pass — sidebar ordering, Boss Dashboard lockdown, first Sales/Purchase accounts

- Renamed the three collections with numeric prefixes (`1. Yöneticiler`, `2. Satış/Satın Alma`, `3. Diğer`) to force sidebar sort order.
- Revoked `Director`'s access to the old `Boss Dashboard` collection entirely (View → No access) — this **broke 5 of 8 cards** on `Cansun Satış Genel Bakış`, since those cards' questions still physically live there. Fixed same day, see next section.
- Provisioned the first two real `Sales/Purchase` accounts (user IDs 6, 7) and a second Director account (Director #2, user ID 3) — all group memberships confirmed via `/api/user/<id>`, not the admin UI label.

### Same day, third pass — Boss Dashboard lockdown regression fix

- Restored `Director`'s access on the old collection to **View** (not the broken No access, and not full Curate) — cards render correctly again.
- Renamed the collection `Boss Dashboard` → `_Hesaplama Kaynağı` ("calculation source" — deliberately generic, non-hierarchical naming) and nested it as a sub-collection under `1. Yöneticiler` instead of a top-level sidebar item.
- Verified live: all 8 dashboard cards render with zero permission placeholders; edit/move/delete are blocked (Metabase shows an explicit "View-only" badge); `Sales/Purchase`/`Other` confirmed unaffected.
- Separately redacted account-specific details from the permissions doc afterward, same day — a documentation cleanup, not a permissions change.

### Next steps (superseded — see below)

- `Other`/`Diğer` real-report rollout — still deferred; collection/group scaffolding exists but stays empty.
- New sync pipeline (`supplier_last_purchase`) — done next, 2026-08-14.
- ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, and `DovizKodu` normalization — all unchanged/open from above.

## 2026-08-14 — supplier_last_purchase pipeline + new-report playbook

Added a third synced table, `supplier_last_purchase` (each supplier's last purchase per product, across all three companies via `Firma`), following the by-now-established per-table-script pattern.

- Sourced from the pre-built ERP view `aa_supplier_last_purchase` (~29,300 rows). Full replace, but via upsert-then-prune rather than `TRUNCATE` — confirmed live that `reporting_writer` only has `SELECT, INSERT, UPDATE, DELETE` on this table, no `DROP` (which `TRUNCATE` requires in MySQL).
- Added `refresh_supplier_last_purchase.py` to the monitoring registry, cron'd `0 3 * * *` (15 minutes after `customer_last_price`, confirmed non-overlapping from the earlier jobs' observed run durations).
- Added the question `Tedarikçi Son Alış Fiyatı` (card 57) to `2. Satış/Satın Alma`, same two-Field-Filter search pattern as `Müşteri Son Fiyat Sorgusu`. No new collection, no permission changes needed — confirmed via the permissions graph first that the existing `Satış/Satın Alma` grants already covered it.
- Verified `Sales/Purchase`/`Other` group access via the Metabase API directly (session tokens for throwaway test accounts) rather than logging into the shared admin browser session as someone else, to avoid repeating the 2026-08-13 session-invalidation incident.
- Wrote `docs/adding-a-new-report.md` — a standing playbook for the recurring "new ERP view → synced table → Metabase question" pattern, based on how this and the two prior pipelines were actually built.
- This script's per-run Telegram notification (sent on both success and failure) was later found to be an inconsistency with every other job and removed on 2026-08-19 — see below.

### Next steps (superseded — see below)

- ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, container timezone fix, and `Other`/`Diğer` rollout — all unchanged/open from above.

## 2026-08-18 — Metabase's own DB account renamed for clarity

Renamed the Metabase-facing MySQL account on `reporting-db` from `metabase_ro` to `metabase_reporting_ro`, to end the confusing shared name with the *ERP's* separate `metabase_ro` account (different host, different purpose, discovered back on 2026-08-10). Documentation-only change on this side — Arslan performed the actual rename directly on athena; updated `docs/credentials.md`/`docs/tables.md` references accordingly. No grants or application behavior changed.

## 2026-08-19 — cheque_bond_maturity pipeline, consolidated Telegram digest, first Takip tab

**New table, `cheque_bond_maturity`** (Cansun + Almer cheque/bond maturity schedule; Karacan intentionally excluded — not present in the source view). Full-replace-by-key via chunked `REPLACE INTO` (same no-`DROP`-grant reason as before). **Composite-key correction found before shipping:** the originally intended key `(Firma, EvrakNo)` turned out not unique — one document can carry several installments, each needing its own row. Verified live first (222 source rows collapsed to only 70 distinct `(Firma, EvrakNo)` pairs) rather than assuming; Arslan changed the primary key to `(Firma, BelgeNo)` directly on `reporting-db` before the sync script was built. Added to the monitoring registry, cron'd `15 3 * * *`.

**Consolidated Telegram digest:** found `refresh_supplier_last_purchase.py` was the only sync job sending its own standalone Telegram message per run — an inconsistency, not a feature (the other jobs' runs were logged the same way but nothing ever surfaced a completion time anywhere). Removed that script's standalone message and instead taught the shared `job_logging.py`/`report_job_status.py` to capture and display a `completed HH:MM:SS` timestamp for every job in the one 07:00 digest. Net effect: one consolidated message now covers all four registry jobs with per-job timestamps, no separate per-job Telegram sends exist anywhere in the codebase.

**First `Takip` tab (superseded 2026-08-20, see below):** split dashboard 4 (`Firmamızın Senet-Çek Durumu`) into `Genel Bakış` (original 2 cards, untouched) and a new `Takip` tab with 12 monthly calendar cards (`Ocak Takip`…`Aralık Takip`), each a date-series LEFT JOIN against `cheque_bond_maturity`. Weekday/holiday flagging verified against a real calendar before building all 12. Hit and documented a real Metabase API gotcha here: `PUT /api/dashboard/:id/cards` deletes any tabs not included in the same request body, so `tabs` must always be sent alongside `cards` on every call, not just the one that creates them.

### Next steps (superseded — see below)

- `Takip`'s fixed 12-month design doesn't handle a year boundary — rebuilt next, 2026-08-20.
- ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, container timezone fix, and `Other`/`Diğer` rollout — all unchanged/open from above.

## 2026-08-20 — Takip rebuilt with a date-driven range, monthly filter-default bump job, dashboard-component policy

**`Takip` rebuilt and moved:** replaced the fixed Ocak–Aralık design with a data-driven range (`MAX(VadeTarihi)` through the current month — 7 months as of this date, spanning a year boundary cleanly) and moved it from `Firmamızın Senet-Çek Durumu` to `Cansun Satış Genel Bakış` as a new tab, preserving all 13 existing cards' exact positions in the same API call (same "always send all tabs" gotcha as 2026-08-19). Added conditional color-range formatting on totals, confirmed live to exist in this Metabase version before assuming so. Renamed the date column to `Vade Tarihi` and switched its display format to `D/M/YYYY` after confirming live that manual column-width dragging doesn't exist in this version (it reorders columns instead) — the narrower date format, not a width change, is what actually removed the table's need for a horizontal scrollbar. Arslan renamed the tab to `Çek-Senet Takip` shortly after.

**`bump_filter_defaults.py` added** — a monthly (not daily) job that re-points the "Günlük/Haftalık" tab's 9 hardcoded-year/month card defaults plus 2 dashboard filters at the current month via the Metabase API, since those defaults would otherwise silently go stale every month. Discovered this Metabase version (v0.63.2.7) uses a newer MBQL5 "stages" query shape, not the classic dict shape the public API docs describe — found the real template-tag location empirically rather than trusting the docs. Created a scoped `METABASE_API_KEY` for this, since session-token login isn't practical for a once-a-month unattended job. Deliberately excluded from the daily monitoring registry (would falsely cry "DID NOT RUN" on the other ~29 days of the month) — sends its own dedicated Telegram message per run instead. Cron'd `30 3 1 * *`.

**Policy added:** any future dashboard-component question must be created directly inside `_Hesaplama Kaynağı` from the start, not built elsewhere and moved later — avoids repeating the collection-membership confusion hit during earlier permission audits.

### Next steps (superseded — see below)

- Vault balance/movement pipelines — added next, 2026-08-21.
- ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, container timezone fix, and `Other`/`Diğer` rollout — all unchanged/open from above.

## 2026-08-21 — vault_status and vault_movements pipelines (cash vault reporting)

Two new tables covering Cansun's cash vaults ("kasa"): `vault_status` (live balances, 3 rows, hourly `REPLACE INTO`) and `vault_movements` (full transaction history behind those balances).

- `refresh_vault_status.py`: hourly, 09:00–19:00 (`0 9-19 * * *`). Deliberately excluded from the daily digest registry — its first run of the day (09:00) postdates the 07:00 report, so registering it would falsely cry "DID NOT RUN" every single morning. Uses failure-only Telegram alerting instead — a new pattern alongside the existing daily ok/fail digest, since a success ping every hour would flood the channel.
- `vault_movements` gets **two** separate sync scripts rather than one, matching two different freshness needs: `refresh_vault_movements_hourly.py` (last 3 days, scoped `DELETE`+re-insert, offset 5 minutes past `vault_status`'s cron so their ERP connections don't fire in the same instant) and `refresh_vault_movements_daily.py` (full nightly replace, `30 3 * * *`). Both failure-only on Telegram, both excluded from the registry for the same "first run postdates the digest" reason.
- One-time manual backfill of `vault_movements` (1,306 rows) before cron took over.
- Confirmed live (not assumed) that `reporting_writer` has no `DROP` grant on `vault_movements` either — same `DELETE`-not-`TRUNCATE` pattern as `supplier_last_purchase`/`cheque_bond_maturity`.

### Next steps (superseded — see below)

- New çek/senet portfolio pipeline — added 2026-08-23.
- ERP/`reporting_writer` password rotation, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, container timezone fix, and `Other`/`Diğer` rollout — all unchanged/open from above.

## 2026-08-23 — cek_senet_portfoy pipeline (outstanding çek/senet portfolio)

Added `refresh_cek_senet_portfoy.py`, syncing `reporting.cek_senet_portfoy` (10 rows) from ERP view `aa_cek_senet_portfoy`, every 2 hours 09:00–19:00 (`0 9-19/2 * * *`) — deliberately not offset from `vault_status`'s hourly slots, since the two hit unrelated source views/destination tables and the overlap is harmless.

- **Mandatory pre-flight duplicate check, unique to this job:** before any write, groups source rows by `(Firma, CekSiraNo)` — the intended primary key — and aborts loudly if any group has more than one row, since a silent `DELETE`+`INSERT` would otherwise quietly collapse real duplicate rows if that key assumption ever turned out wrong. Not yet exercised against real duplicate data (would require writing to the read-only ERP source); verified by code review only so far.
- Same no-`DROP`-grant discovery as prior tables — switched to `DELETE`+`INSERT`.
- Per the task spec, this job originally sent a Telegram message on **both** success and failure — a deliberate one-off deviation from the failure-only pattern the other intraday jobs use. Changed the next day (below) once the every-2-hours success pings became noise.
- Added three Metabase questions to `_Hesaplama Kaynağı`, placed on dashboard 3's "Kasa Durumu" tab between the vault scalar cards and the `vault_movements` tables.

## 2026-08-24 — vault_movements window widened, cek_senet_portfoy alerting quieted, Sales/Purchase batch 2

- **`refresh_vault_movements_hourly.py`:** widened its rolling window from 3 to 4 days per Arslan's request. `refresh_vault_movements_daily.py` needed no change (already a full nightly replace regardless of window). Verified with a manual run (20 rows under the new 4-day cutoff vs. 9 under the old 3-day one on the prior cron run).
- **`cek_senet_portfoy`:** removed the success-path Telegram message added 2026-08-23 — it now matches the failure-only pattern used by every other intraday job. Re-verified both the now-silent success path and the still-working failure alert.
- **Sales/Purchase batch 2:** added 7 more real accounts (user IDs 20–26), plus one more (ID 27) once Arslan confirmed shortly after that an initially-ambiguous 8th candidate (originally floated for `Other`) actually belonged in `Sales/Purchase` — brings the group to 10 real members. All confirmed via `/api/user/<id>`, group membership exactly `[All Users, Sales/Purchase]`.
- **Password-handling preference confirmed for this batch:** Metabase's admin UI has no "type an arbitrary password" flow for an admin — only "Reset password" (admin sees the generated value) or "Get reset link" (only the user ever sees it). Arslan confirmed he's fine with "Reset password" and wants the resulting table relayed directly in chat at the end (not a standalone file, which was the prior batch's method) — noted as a standing preference to check against, not assumed automatically next time.
- Updated `docs/metabase-permissions.md`, `docs/monitoring.md`, `docs/tables.md` to reflect all of the above; committed and pushed together.

### Next steps (superseded — see below)

- ERP/`reporting_writer` password rotation — still pending, Arslan's own action item, unchanged since 2026-08-10.
- Container timezone fix (Metabase/MySQL still on UTC internally) — still pending Arslan's go-ahead and a low-usage window, unchanged since 2026-08-12.
- `scripts/sync_engine.py` consolidation — still undecided; every pipeline added since has followed the per-table-script pattern instead.
- `DovizKodu` normalization (EUR/EURO variants) — still open, unscheduled.
- `Other`/`Diğer` real-report rollout — still deferred; scaffolding exists, no real members or reports yet.
- No monitoring for `report_job_status.py`, `bump_filter_defaults.py`, or any of the intraday/failure-only jobs' own health (i.e. nothing watches these watchers) — known gap, not requested.

## 2026-08-25 — stock_details pipeline, Manager group and collection, doc-staleness fixes

Ran the full spec end-to-end: a new nightly sync pipeline, a new Metabase group/collection, and a documentation-checklist expansion prompted by the 08-24 backfill gap.

**`stock_details` pipeline:** synced `reporting.stock_details` (40,104 rows, full stock/product catalog with pricing/supplier/dimension columns) from ERP view `cansun.aa_rapor_stok_listesi`. Verified live before assuming the spec's grant claim: `reporting_writer` genuinely has `DROP` on this table (unlike everything added since `cheque_bond_maturity`), so the script uses plain `TRUNCATE` + chunked `INSERT`, matching `customer_last_price`'s pattern. `StkoKodu` confirmed unique on the live source (40,104/40,104 distinct) before skipping a pre-flight duplicate check. Cron'd `45 3 * * *`, registered in `monitored_jobs.yml` as the fifth daily-digest job (no standalone Telegram — covered by the existing 07:00 digest). Manual run verified: 40,104 rows in 6.66s, logged `ok`.

**Manager group and `2. Manager` collection:** hit a real discrepancy before building anything — the task assumed a `Manager` group already existed with Nursiman as a member; live state showed neither was true (no `Manager` group at all, Nursiman a real `Sales/Purchase`-only member). Flagged it and asked rather than guessing; Arslan chose to create `Manager` fresh and add Nursiman to it *in addition to* his existing `Sales/Purchase` membership. Created the group (ID 8), added Nursiman, then created `2. Manager` (collection ID 35) as a flat peer of the other numbered collections — not nested like `_Hesaplama Kaynağı`. This collection's `All Users` defaulted to No access on creation (not the Curate-by-default gotcha every prior collection hit) — set Director/Manager to View explicitly anyway rather than trust that the favorable default will hold next time. Renumbered `2. Satış/Satın Alma`→`3.` and `3. Diğer`→`4.` (same IDs, name only) to make room; confirmed via a before/after `/api/collection/graph` diff that the renames touched only names, not permissions.

Built `Stok Listesi` (card 103), a deliberate full-detail placeholder — every `stock_details` column including the cost column `FabrikaFiyatiUsd`, with the column-exposure decision explicitly deferred to a future pass. Verified live with three throwaway test accounts over independent API sessions (never touching the admin browser session): Director-only → 200, Manager+Sales/Purchase (matching Nursiman's actual real group set, not a naive "Manager only") → 200, Sales/Purchase-only → 403 on both the card and the collection — confirms the cost column doesn't leak. All three throwaway accounts deactivated immediately after.

**Unrelated finding, not acted on:** several older throwaway test accounts (`Test ChequeVerify*`, `Test TabsVerify*`, `Test TakipMoveVerify*`) are still active with real recent logins, despite this project's own deactivate-after-use convention. Flagged in `docs/metabase-permissions.md` for Arslan to decide on separately — out of scope for this task.

**Documentation:** updated all six files per the (now-expanded) Step 13 checklist in `docs/adding-a-new-report.md` — `docs/tables.md`, `docs/monitoring.md`, `docs/metabase-permissions.md`, `docs/server-architecture.md`, `README.md`, and this entry.

### Next steps (superseded — see below)

- Old throwaway test accounts (`Test ChequeVerify*`/`TabsVerify*`/`TakipMoveVerify*`) — still active, flagged but not deactivated; Arslan's call on whether/when to clean up.
- `stock_details` column exposure (which columns `Stok Listesi` should actually show, given `FabrikaFiyatiUsd`) — explicit future decision, not made here.
- ERP/`reporting_writer` password rotation, container timezone fix, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, `Other`/`Diğer` rollout, and monitoring-the-watchers — all unchanged/open from above.

## 2026-08-25 (continued) — sales_snapshot stale-row bug: root cause, one-time cleanup, permanent fix

Arslan reported a manually-confirmed total (1,227,259.37 TL, Almer+Cansun, HesapKodu `S 35831`, August 2026) not matching the dashboard. Investigated as diagnosis-only first, per explicit instruction — no fix until the cause was understood.

**Diagnosis:** ERP source view (`aa_rapor_engin_karacan_satis`) itself totaled 1,224,959.26 for this customer/month (Karacan confirmed genuinely absent, not excluded) — already 2,300.11 TL short of Arslan's figure, a separate and still-unexplained gap unrelated to the sync pipeline. Comparing `sales_snapshot` against that same ERP total surfaced the real bug: 36 Almer rows in `sales_snapshot` where the ERP view has only 12. No `(Firma, ID)` duplicates existed (composite PK intact) — instead, three distinct ID ranges (152446–152457, 152674–152685, 152769–152780) all represented the *same* line items, apparently reissued twice in the ERP under new IDs; only the last range still exists in the live ERP view. Root cause: `refresh_sales_snapshot.py`'s stage-then-`REPLACE INTO` merge has always had no mechanism to remove a row once its key stops appearing in the ERP fetch — every such correction anywhere in the table leaves a permanent stale duplicate behind, forever.

**Table-wide scope, not just S 35831:** a dry-run `SELECT` (fresh ERP pull staged into `sales_staging`, diffed against `sales_snapshot` for the same 4-month window, then `sales_staging` truncated back to empty — no side effects) found **90 stale rows across 3 distinct HesapKodu**, not one: `S 35831` (Almer, 24 rows, 1,881,466.15 TL), `Y 01823` (Cansun, 65 rows, 606,154.56 TL — the largest single contributor), and `34-947` (Cansun, 1 row, 2,759.42 TL). Total: **2,490,380.13 TL** of phantom revenue sitting in the table before cleanup. Shown to Arslan for explicit confirmation before any deletion, per instruction.

**One-time cleanup:** re-fetched a fresh ERP pull (re-confirmed 90/2,490,380.13 matched the earlier dry run before deleting, guarding against drift between the two runs), then ran the equivalent `DELETE` with the identical `WHERE` clause. **Exactly 90 rows deleted.** Re-verified `S 35831`/August immediately after: 12 Almer rows / 1,161,510.03 TL, 4 Cansun rows / 63,449.23 TL — matches the live ERP view exactly.

**Permanent fix:** added a prune step to `refresh_sales_snapshot.py` — `DELETE FROM sales_snapshot WHERE Tarih >= <cutoff> AND NOT EXISTS (matching key in sales_staging)`, run immediately after the merge and before staging is cleared. The window cutoff is now computed **once** (`SELECT DATE_SUB(CURDATE(), INTERVAL 4 MONTH)` against the ERP connection) and passed as a bound parameter to both the fetch and the prune, rather than two independently-evaluated `CURDATE()` calls against two different MySQL servers that could in principle drift apart.

**Logging decision, not a silent deviation:** kept `job_runs.csv`'s `rows` field meaning unchanged (merged-row count only) rather than extending the shared CSV schema `job_logging.py` writes for all eleven jobs to carry a second number — judged that redesigning a schema used by ten other jobs' log history was out of proportion to logging one job's prune count. The prune count is printed to stdout (`refresh.log`) on every run instead, same as every other diagnostic line this script already prints. Documented in `docs/monitoring.md` in case this tradeoff should be revisited.

**Verified (2026-08-25):** manual run after the fix merged 68,007 rows and pruned 0 (correct — the one-time cleanup had already removed everything stale), zero errors, logged `ok`. Table-wide re-check: in-window row count (68,007) exactly equals the fresh ERP fetch's row count — zero orphans anywhere in the table. Rows older than the window are structurally unreachable by either the merge or the prune (both scoped to `Tarih >= cutoff`) — confirmed 618,498 pre-window rows present and spot-checked unchanged. Known-good current IDs (152769/152775/152780, the real still-current `S 35831` Almer invoices) confirmed to survive the prune untouched.

### Next steps (superseded — see below)

- **Issue B, still open:** the 2,300.11 TL gap between the live ERP view and Arslan's manually-confirmed total for `S 35831`/August — not a sync bug (the sync is now provably faithful to the ERP view), needs reconciling against whatever source produced the manual figure.
- Old throwaway test accounts, `stock_details` column exposure, ERP/`reporting_writer` password rotation, container timezone fix, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, `Other`/`Diğer` rollout, and monitoring-the-watchers — all unchanged/open from above.
- Whether the `job_runs.csv` schema should eventually carry a prune count (or similar per-job secondary metrics) as a deliberate, first-class field rather than leaving it to each script's own stdout — flagged, not decided.

## 2026-08-26 — vault_status widened from hourly to every 30 minutes

Per Arslan's request. Manual run first: `status=ok`, 3 rows, fresh values confirmed directly against `reporting-db` (TL/EUR/USD balances all current) — same verification standard as any other manual run, not just "no error." Checked the script's actual runtime history before touching cron, rather than assuming the old cadence being safe meant the new one would be too: a consistent ~0.10s across its entire hourly run history, so halving the interval to 30 minutes leaves an enormous margin. Checked the *full* crontab, not just the neighboring `vault_movements_hourly` job, for anything else scheduled at `:30` inside 09:00–19:00 — found none (`vault_movements_daily`/`bump_filter_defaults.py` both use `:30` but at 03:30, outside this window). Cron changed from `0 9-19 * * *` to `0,30 9-19 * * *`.

### Next steps (superseded — see below)

- **Issue B, still open:** the 2,300.11 TL gap between the live ERP view and Arslan's manually-confirmed total for `S 35831`/August — not a sync bug, needs reconciling against whatever source produced the manual figure.
- Old throwaway test accounts, `stock_details` column exposure, ERP/`reporting_writer` password rotation, container timezone fix, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, `Other`/`Diğer` rollout, and monitoring-the-watchers — all unchanged/open from above.
- Whether the `job_runs.csv` schema should eventually carry a prune count as a first-class field — flagged, not decided.

## 2026-08-27 — "Genel Bakış" tab: Tarih filter inconsistency diagnosed and fixed

A Director-flagged symptom on dashboard 3's default landing tab: "Bu Ay Ciro" read 71.7M while "İhracat/Yurtiçi Dağılımı" and "Firma Karşılaştırması" implied ~83.7M for the same "Previous 30 days" filter selection. Diagnosed first (no changes), reported, then fixed per Arslan's explicit decisions on each card.

**Root cause — two date models on one tab.** Of the 8 cards on the "Genel Bakış" tab (tab id 4), 4 are driven by the dashboard's `Tarih` field-filter (`WHERE {{tarih}}`: cards 45/46 Toplam Satış Adedi / Ortalama Fatura Değeri, 48/49 Firma Karşılaştırması / İhracat-Yurtiçi) and 4 ignore the filter entirely, hardcoding `CURDATE()`-based logic (43 Bu Ay Ciro = `YEAR(Tarih)=YEAR(CURDATE()) AND MONTH(Tarih)=MONTH(CURDATE())`, 44 Bu Yıl Ciro = `YEAR(Tarih)=YEAR(CURDATE())`, 47 Yıllık Karşılaştırma = `YEAR(Tarih) IN (YEAR(CURDATE()), YEAR(CURDATE())-1)`, 55 2026 Satis Ay Bazinda = MBQL builder filter `Tarih > 2026-01-01`). Proof the two cards use identical math and differ only in date window: setting `Tarih` to the current month made "Firma Karşılaştırması" total exactly equal "Bu Ay Ciro" to the kuruş (71,658,715.25). The 83.7M figure is the rolling-30-day window (drops Aug 27, adds Jul 28–31, four higher-volume days).

**Decision (Arslan, explicit — asked before touching anything):**
- **Cards 43 / 44 left as fixed KPIs.** No SQL change, no rename. Treated as intentional "where we stand this month / this year" header tiles, independent of the filter by design. (Had they both been switched to `WHERE {{tarih}}` they would have become byte-identical queries — flagged this, which drove the decision.)
- **Cards 47 / 55 left untouched.** Card 47 is a deliberate this-year-vs-last-year comparison; card 55 a deliberate current-year monthly breakdown by Firma — neither expressible as a single `{{tarih}}` range. Noted but not fixed: card 55's literal `2026-01-01` will silently go stale on 2027-01-01 (card 47 self-updates via `CURDATE()`).

**Part B — widget-type mismatch fixed (cards 45, 46, 48, 49).** Each declared its `{{tarih}}` template tag *and* its card-level `parameters[]` entry as `date/month-year`, while the dashboard `Tarih` param is `date/all-options`. Through the dashboard the mismatch was masked (dashboard value wins), but opening any of the 4 as a standalone question rejected relative ranges: `Invalid parameter value type :date/all-options for parameter "tarih" with widget type :date/month-year`. Changed both the template-tag `widget-type` and the card-level `parameters[].type` to `date/all-options` via `PUT /api/card/:id` with `{dataset_query, parameters}` only. SQL text round-trip-diffed clean (unchanged) on all 4. Verified: a standalone `past30days` query on each now returns data instead of erroring.

**Part C — Tarih filter default set.** Dashboard 3's `Tarih` param (`d147204a`) had no default and no `required` flag, so a fresh load left the filter-aware cards at all-time totals while the hardcoded cards showed current month/year — the tab's own worst inconsistency, visible before any interaction. Set `default: "thisyear"` and `required: true` via `PUT /api/dashboard/3` with `{parameters}` only, matching the `Yıl`/`Ay` convention on the Günlük/Haftalık tab. `thisyear` resolves to `2026-01-01~<today>`. Other params (`firma`, `yıl`, `ay`) confirmed untouched.

**No clobbering.** All edits were card-level `PUT`s and one `parameters`-only dashboard `PUT` — no `/api/dashboard/3/cards` call. Before/after `GET /api/dashboard/3` diff: 39 dashcards, 6 tabs, every per-tab card-id set identical, zero card-name changes. (Dashboard edits log as user `metabase-filter-defaults-cron` — the API key's identity.)

**Verified live (Playwright MCP, `http://10.20.52.43:3000`, admin session):**
- Fresh load (default "This year"): "Bu Yıl Ciro" (fixed) 541.9M = "İhracat/Yurtiçi Dağılımı" 541.9M = "Firma Karşılaştırması" 541.9M — the same-window cards now agree on landing. "Bu Ay Ciro" 71.7M is the current-month subset, clearly labelled.
- `Tarih = Previous 30 days`: filter-aware cards move together to the 83.7M window ("İhracat/Yurtiçi" Total 83.7M, "Toplam Satış Adedi" 76,639, "Ortalama Fatura Değeri" 14,578.45 — all matching direct-API cross-checks); the fixed KPIs stay pinned at 71.7M / 541.9M by design.
- Cards 47 and 55 render unchanged.

**Documentation:** only this file. No table/script/cron added; Part B/C changed filter internals only — no card names or collection placement changed — so `docs/tables.md`, `docs/monitoring.md`, `docs/server-architecture.md`, `docs/metabase-permissions.md`, and `README.md` need no change.

### Next steps

- **Card 55's hardcoded `2026-01-01`** (and the dashboard `Yıl`/`Ay` filter defaults `2026`/`8`) — same class of silent-staleness issue, all roll over wrong on 2027-01-01. Left as-is here per the "leave untouched" decision; worth a deliberate pass to switch to `CURDATE()`-derived boundaries like card 47 uses.
- **Issue B, still open:** the 2,300.11 TL gap between the live ERP view and Arslan's manually-confirmed total for `S 35831`/August — not a sync bug, needs reconciling against whatever source produced the manual figure.
- Old throwaway test accounts, `stock_details` column exposure, ERP/`reporting_writer` password rotation, container timezone fix, `scripts/sync_engine.py` consolidation, `DovizKodu` normalization, `Other`/`Diğer` rollout, and monitoring-the-watchers — all unchanged/open from above.
- Whether the `job_runs.csv` schema should eventually carry a prune count as a first-class field — flagged, not decided.
