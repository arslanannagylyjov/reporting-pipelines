# Job Monitoring — Silent-Failure Detection

Added 2026-08-11. Detects two failure modes for the nightly sync jobs: an explicit
crash (`status=fail` in the log) and the more dangerous silent case — a job that
never ran at all (no log entry for today), e.g. because its cron entry was
removed, the box rebooted and cron didn't restart, or someone disabled it and
forgot. All files live on **athena** in `~/reporting-scripts/` (not in this
repo), same as the sync scripts themselves — this doc describes their format
and behavior.

## Job registry — `monitored_jobs.yml`

```yaml
jobs:
  - name: sales_snapshot
    script: /home/arslan/reporting-scripts/refresh_sales_snapshot.py
    schedule: "30 2 * * *"
    expected_time: "02:30"
  - name: customer_last_price
    script: /home/arslan/reporting-scripts/refresh_customer_last_price.py
    schedule: "45 2 * * *"
    expected_time: "02:45"
  - name: supplier_last_purchase
    script: /home/arslan/reporting-scripts/refresh_supplier_last_purchase.py
    schedule: "0 3 * * *"
    expected_time: "03:00"
  - name: cheque_bond_maturity
    script: /home/arslan/reporting-scripts/refresh_cheque_bond_maturity.py
    schedule: "15 3 * * *"
    expected_time: "03:15"
```

`supplier_last_purchase` added 2026-08-14, 15 minutes after `customer_last_price` (02:45) — confirmed non-overlapping before adding: both earlier jobs finish in well under a minute (`sales_snapshot` ~19s, `customer_last_price` ~8s per `job_runs.csv`), leaving a wide margin to 03:00.

`cheque_bond_maturity` added 2026-08-19, 15 minutes after `supplier_last_purchase` (03:00) — same margin reasoning: all three earlier jobs finish in well under 20s each, and the new job itself ran in 0.19s on its first live run (222 source rows), so 03:15 leaves a wide margin. Worth reconfirming actual runtime after a few real cron cycles rather than treating 03:15 as permanently final, per usual practice when adding a job here.

`report_job_status.py` reads this file and loops over `jobs` — it never
hardcodes a job name. `name` **must** match the `job_name` string the script
passes to `run_job()` (see below), since that's the join key between the
registry and the log.

**To add a new monitored job:**
1. Add an entry here with `name`, `script`, `schedule`, `expected_time`.
2. In the new script, wrap its `main()` (which should `return` a row count)
   with `job_logging.run_job('<name>', main)` in the `if __name__ ==
   '__main__':` block — matching the `name` used in step 1.
3. Add the job's own cron line, same pattern as the existing two.

Nothing in `report_job_status.py` needs to change.

## Log — `job_runs.csv`

One CSV row appended per job run, shared by all jobs, written by
`job_logging.py`:

| Field | Meaning |
|---|---|
| `timestamp` | Run completion time, `YYYY-MM-DD HH:MM:SS`, Europe/Istanbul (athena host's local clock — see `docs/server-architecture.md`) |
| `job_name` | Must match a `name` in the registry |
| `status` | `ok` or `fail` |
| `rows` | Row count returned by the job's `main()` (blank on failure) |
| `duration_seconds` | Wall-clock time of `main()` |
| `error_message` | `<ExceptionType>: <message>`, truncated to 500 chars (blank on success) |

`job_logging.run_job(job_name, func)` times `func()`, logs `ok` with its
return value as `rows` on success, or logs `fail` with the exception and
**re-raises** on a crash — so the traceback still lands in the script's own
cron log (`refresh.log` etc.) exactly as before; the structured log is
additive, not a replacement for that.

All four sync scripts end with:

```python
if __name__ == '__main__':
    from job_logging import run_job
    run_job('sales_snapshot', main)  # or 'customer_last_price', 'supplier_last_purchase', 'cheque_bond_maturity'
```

`main()` itself is unchanged except that it now `return`s the total row
count instead of nothing. No other sync logic was touched. This is also the
*only* place any of these scripts touch Telegram — as of 2026-08-19, none of
them send a per-job message themselves; see "Per-job Telegram notification
removed" below.

## Report — `report_job_status.py`

Cron'd daily at **07:00** on athena
(`~/reporting-scripts/report_job_status.log` captures its own output). For
every job in the registry, using only today's log entries (most recent one if
a job ran more than once):

- No entry for today → **`❌ <name>: DID NOT RUN today (no log entry)`**
  — checked and reported first; this is the priority signal, since a crash at
  least leaves a trace but a job that silently stopped running leaves nothing.
  No completion timestamp is shown here — there's no log entry to read one from.
- Latest entry is `status=fail` → **`❌ <name>: FAILED — <error_message> —
  completed HH:MM:SS`**
- Latest entry is `status=ok` → **`✅ <name>: ok — <rows> rows in
  <duration_seconds>s — completed HH:MM:SS`**

The completion timestamp is the `HH:MM:SS` portion of that log row's own
`timestamp` field (see the `job_runs.csv` field table above) — Europe/Istanbul,
same clock as the rest of the athena host.

Failures and missing jobs are listed first, successes after. The message is
sent via one HTTP POST to the Telegram Bot API
(`api.telegram.org/bot<token>/sendMessage`), using `TELEGRAM_BOT_TOKEN` and
`TELEGRAM_CHAT_ID` from `~/reporting-scripts/.env` — same file, same
stdlib-only loader (`_load_env`) already used by the sync scripts. No new
`.env` file, no secrets added to this repo.

## Per-job Telegram notification removed, timestamps consolidated into the daily digest (2026-08-19)

Between 2026-08-14 and 2026-08-19, `refresh_supplier_last_purchase.py` sent
its own standalone Telegram message immediately after every run (success or
failure), separate from the 07:00 digest above — see git history of this file
for the exact format that used. This turned out to be an inconsistency, not a
feature: `supplier_last_purchase` was the only one of the (then three) sync
jobs that captured and showed its own completion timestamp at all: the other
two jobs' runs were logged to `job_runs.csv` with a timestamp same as always,
but nothing ever surfaced it anywhere.

Fixed by going the other direction — removed the standalone per-job message
entirely (deleted `refresh_supplier_last_purchase.py`'s own `send_telegram()`
call and its `try`/`except` wrapper; its `__main__` block now matches the
other sync scripts exactly, see above) and instead normalized
`job_logging.py`'s `log_run()` to write `timestamp` in the human-readable
`YYYY-MM-DD HH:MM:SS` format every job already got implicitly, then taught
`report_job_status.py`'s `build_message()` to display it per job. Net effect:
completion timestamps are now captured the same way for every job (via the
shared `job_logging.py` helper, not per-script code), and shown for every job,
but only in the one consolidated 07:00 message — no separate per-job
Telegram send exists anywhere in this codebase anymore.

Verified live 2026-08-19: manually ran `report_job_status.py` and confirmed a
single message with per-job `completed HH:MM:SS` on every line, no leftover
individual message from any sync script.

## Verified (2026-08-11)

- **Forced failure:** ran `refresh_sales_snapshot.py` with a deliberately
  wrong `ERP_DB_PASSWORD` (env override, `.env` on disk untouched) — crashed
  with exit code 1, traceback intact, and logged a `fail` row with the MySQL
  auth error. The next report showed `❌ sales_snapshot: FAILED —
  ProgrammingError: 1045 ...` ahead of the other job's `✅`.
- **DID NOT RUN:** disabled `customer_last_price`'s cron line (commented it
  out, confirmed via `crontab -l`, then restored it) and separately removed
  its log entries for the day to reproduce the exact state that disabled cron
  entry would produce tomorrow morning. The report correctly emitted `❌
  customer_last_price: DID NOT RUN today (no log entry)`.
- **Clean night:** both jobs run for real, both logged `ok`, report showed
  `✅ sales_snapshot: ok — 65068 rows in 17.92s` and `✅ customer_last_price:
  ok — 65204 rows in 7.39s`.

All test artifacts (backup files, injected bad credentials, deleted log rows)
were reverted before finishing; `job_runs.csv` on athena reflects real run
history only.

## Verified (2026-08-19)

- **`cheque_bond_maturity` added to the registry and cron** (03:15, 15
  minutes after `supplier_last_purchase`) — manually ran
  `refresh_cheque_bond_maturity.py` once to validate end-to-end before
  relying on cron: 222 rows loaded (matching the ERP view's row count
  exactly), logged to `job_runs.csv` in the new `YYYY-MM-DD HH:MM:SS` format.
- **Consolidated digest picked up the fourth job automatically** — no code
  change was needed in `report_job_status.py` beyond the timestamp display
  fix above, since it already loops over the registry rather than
  hardcoding job names (see "To add a new monitored job" above). Manually
  ran `report_job_status.py` and confirmed all four jobs on one message,
  each with a real `completed HH:MM:SS`:
  ```
  Daily job report — 2026-08-19
  ✅ sales_snapshot: ok — 66503 rows in 18.86s — completed 02:30:20
  ✅ customer_last_price: ok — 65518 rows in 10.99s — completed 02:45:12
  ✅ supplier_last_purchase: ok — 29353 rows in 11.86s — completed 03:00:13
  ✅ cheque_bond_maturity: ok — 222 rows in 0.19s — completed 17:55:54
  ```
- **No leftover per-job message** — confirmed `refresh_supplier_last_purchase.py`
  no longer sends anything to Telegram itself (removed as part of this same
  change, see "Per-job Telegram notification removed" above), and
  `refresh_cheque_bond_maturity.py` was never given one in the first place.

## `bump_filter_defaults.py` — monthly Metabase filter-default bump (2026-08-20)

Not one of the sync jobs above and **deliberately not in `monitored_jobs.yml`**
— see "Why it's not in the registry" below. Lives on athena in
`~/reporting-scripts/bump_filter_defaults.py` (not in this repo), same
convention as the sync scripts.

**What it does:** the "Günlük/Haftalık" tab on the Cansun Satış dashboard
(Metabase dashboard id 3) has 9 native-SQL questions (card ids `80`–`88`,
confirmed against the tab's actual `dashboard_tab_id` — the `_Hesaplama
Kaynağı` collection those cards live in has 32 cards total, most unrelated to
this tab, so collection membership alone is not a safe way to enumerate the
target set) plus 2 required dashboard filters (Yıl `d28dbc0d`, Ay `ca252e0e`),
all defaulting to the current year/month. Those defaults are hardcoded values,
not computed, so they go stale every month. This job re-points all 11 objects
at `datetime.now()`'s year/month via the Metabase API:
- For each card: `GET /api/card/:id` → find the `year`/`month` entries in
  `dataset_query.stages[0]['template-tags']` (a **list**, not a dict — this
  Metabase version, v0.63.2.7, uses the newer MBQL5 "stages" query shape, not
  the classic `dataset_query.native.template-tags` dict the public docs
  describe) → `PUT /api/card/:id` with `{"dataset_query": <modified>}` only,
  not the full GET response, to avoid round-tripping read-only fields → re-GET
  and diff the SQL string byte-for-byte to confirm nothing but the default
  changed. On any diff/verify failure for a card, the run stops (does not
  attempt the remaining cards) and alerts — a bad response shape is a reason
  to stop and check, not to keep firing 8 more writes blind.
- For the dashboard: same pattern on `dashboard.parameters`, updating only the
  Yıl/Ay entries' `default` and diffing that Tarih/Firma are untouched.

Auth is a scoped API key (`METABASE_API_KEY` in `~/reporting-scripts/.env`,
same file as `METABASE_ADMIN_*`, no new credential store), group
Administrators, created 2026-08-20 — chosen over session-token login because
this version of Metabase supports API keys (`GET /api/api-key` works) and a
key doesn't expire/need re-login handling in a once-a-month cron job.

**Why it's not in the registry:** `monitored_jobs.yml` + the 07:00 digest
(above) assume daily cadence — a job with no log entry for *today* is
reported as `DID NOT RUN`. This job only runs once a month, so registering it
would make the digest cry wolf on the other ~29 days. Instead it sends its
own dedicated Telegram message immediately after each run, prefixed
`[Filtre Varsayılanları]` so it's unmistakably not one of the four digest
jobs, via the same bot (`@cansun_reporting_bot`, `TELEGRAM_BOT_TOKEN`/
`TELEGRAM_CHAT_ID` from the same `.env`). This is a deliberate, narrow
exception to "Per-job Telegram notification removed" above — that removal was
about *daily* jobs whose per-run status belongs in one consolidated morning
message; a monthly job has no daily message to fold into. It still calls
`job_logging.run_job('filter_defaults_bump', main)` so its runs land in
`job_runs.csv` like every other job, for local history/debugging — it's just
excluded from the registry that drives the digest.

Cron: `30 3 1 * *` — 03:30 on the 1st of the month, chosen after checking
`job_runs.csv`: the 03:00 and 03:15 jobs both finish in well under 20s, so
03:30 leaves a wide margin past both, and 07:00 is 3.5 hours later.

## `refresh_vault_status.py` — hourly vault balance sync (2026-08-21)

Lives on athena in `~/reporting-scripts/refresh_vault_status.py` (not in this
repo), same convention as the other sync scripts. Syncs `reporting.vault_status`
(3 rows — see `docs/tables.md`) from the ERP view `aa_vault_status` via a single
unchunked `REPLACE INTO`, no staging table.

Cron: `0 9-19 * * *` — hourly on the hour, 09:00 through 19:00 daily. Confirmed
no overlap with the 02:30/02:45/03:00/03:15/03:30 jobs — entirely different
time window.

**Deliberately not in `monitored_jobs.yml`:** the registry + 07:00 digest
(above) check for *today's* log entry at 07:00 — three hours before this job's
first run of the day (09:00). Registering it would make every single morning's
digest falsely report `❌ vault_status: DID NOT RUN today`, since by 07:00 the
job genuinely hasn't run yet that day. This is a different reason than
`bump_filter_defaults.py`'s exclusion (monthly vs. daily cadence) but the same
underlying issue: the registry's daily-freshness check doesn't fit this job's
schedule.

**Telegram alerting — failure only, not the per-run pattern:** unlike
`bump_filter_defaults.py` (which sends a message on every run), this job sends
**only on failure/exception**, prefixed `[vault_status]` so it's unmistakably
distinct from the other jobs' messages. A success message every run would
flood the channel at this job's hourly cadence. Uses the same bot
(`@cansun_reporting_bot`, `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` from the same
`.env`). Still calls `job_logging.run_job('vault_status', main)` so every run
(success or failure) lands in `job_runs.csv` for local history/debugging, same
as `bump_filter_defaults.py` — it's only excluded from the registry that
drives the 07:00 digest, not from the shared run log.

**Verified (2026-08-21):** manual run loaded 3 rows matching the ERP view
exactly, logged `status=ok` in `job_runs.csv`, no Telegram message sent (correct
— success is silent). Forced-failure test (`ERP_DB_PASSWORD` env override, same
method as the `sales_snapshot` forced-failure test above) crashed with the
expected `ProgrammingError` 1045, logged `status=fail` with the error detail in
`job_runs.csv`, exited non-zero, and `send_telegram()`'s `urllib.request.urlopen`
call did not raise — confirming the `[vault_status]` alert reached the bot
(same verification method as `bump_filter_defaults.py`'s Telegram check above).
`vault_status` table contents were unaffected by the forced failure (crashed
before any write).

## Verified (2026-08-20)

- Round-tripped a test default (`2026` → `2099` → `2026`) on card 80 and on
  the dashboard's Yıl filter via the API, diffing the full object before/after
  each write: zero drift outside the one `default` field touched, on both.
- Created the `METABASE_API_KEY` scoped key and confirmed it authenticates
  (`GET /api/user/current` → `200`).
- Confirmed the 9-card list by reading dashboard 3's actual tab structure
  (`tabs` + each dashcard's `dashboard_tab_id`), not by assuming the
  `_Hesaplama Kaynağı` collection's contents equal the tab's cards — it
  doesn't (32 cards in the collection, 9 on this tab).
- Manually triggered `bump_filter_defaults.py` live ahead of its first
  scheduled run: all 9 cards + the dashboard filters updated (re-verified via
  a fresh, independent `GET` on all 9 cards and the dashboard afterward — all
  read back `year=2026`/`month=8`), SQL diffs clean on all 9,
  `job_runs.csv` logged `status=ok`, and the Günlük/Haftalık tab reloaded
  fresh in-browser afterward with 0 console errors and all 9 card queries
  returning `202`. Telegram delivery was confirmed structurally, not
  visually: `send_telegram()` uses `urllib.request.urlopen`, which raises on
  any non-2xx response and would have been caught and logged as `status=fail`
  — since the run logged `ok`, the POST to the Telegram API succeeded.
  Current month was unchanged at the time of this test (still 2026-08), so
  this run was a no-op on the actual default values — see the script's own
  log for the exact before/after on its first real 1st-of-month run.