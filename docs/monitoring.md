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
  - name: stock_details
    script: /home/arslan/reporting-scripts/refresh_stock_details.py
    schedule: "45 3 * * *"
    expected_time: "03:45"
```

`supplier_last_purchase` added 2026-08-14, 15 minutes after `customer_last_price` (02:45) — confirmed non-overlapping before adding: both earlier jobs finish in well under a minute (`sales_snapshot` ~19s, `customer_last_price` ~8s per `job_runs.csv`), leaving a wide margin to 03:00.

`cheque_bond_maturity` added 2026-08-19, 15 minutes after `supplier_last_purchase` (03:00) — same margin reasoning: all three earlier jobs finish in well under 20s each, and the new job itself ran in 0.19s on its first live run (222 source rows), so 03:15 leaves a wide margin. Worth reconfirming actual runtime after a few real cron cycles rather than treating 03:15 as permanently final, per usual practice when adding a job here.

`stock_details` added 2026-08-25, 30 minutes after `cheque_bond_maturity` (03:15) — checked `job_runs.csv` for `refresh_vault_movements_daily.py`'s recent runs (the job occupying the neighboring 03:30 slot) before picking 03:45: it consistently finishes in ~0.25s, so 15 minutes past it is a wide margin.

`orders_in_transit` (`expected_time: "03:05"`) and `warehouse_stock` (`expected_time: "03:10"`) added 2026-08-28. Both are nightly jobs that had been running since 2026-08-27 / 2026-08-28 but were initially left out of the registry (they each sent their own per-run Telegram message). On 2026-08-28 the per-run *success* pings were removed from both scripts and the jobs were folded into the digest instead — see the `job_runs.csv` section below and each job's own section for the full reasoning. The 03:05 / 03:10 slots were checked against real `job_runs.csv` data before the registry change: `supplier_last_purchase` (03:00) finishes by ~03:00:27, `cheque_bond_maturity` starts 03:15, and the two new jobs themselves run in ~1s (`orders_in_transit`) and ~0.7s (`warehouse_stock`) — no overlap. **The daily digest now covers seven jobs.**

`report_job_status.py` reads this file and loops over `jobs` — it never
hardcodes a job name. `name` **must** match the `job_name` string the script
passes to `run_job()` (see below), since that's the join key between the
registry and the log.

**To add a new monitored job:**
1. Add an entry here with `name`, `script`, `schedule`, `expected_time`.
2. In the new script, wrap its `main()` (which should `return` a row count)
   with `job_logging.run_job('<name>', main)` in the `if __name__ ==
   '__main__':` block — matching the `name` used in step 1.
3. Add the job's own cron line, same pattern as the existing ones.

Nothing in `report_job_status.py` needs to change.

## Log — `job_runs.csv`

One CSV row appended per job run, shared by all jobs, written by
`job_logging.py`:

| Field | Meaning |
|---|---|
| `timestamp` | Run completion time, `YYYY-MM-DD HH:MM:SS`, Europe/Istanbul (athena host's local clock — see `docs/server-architecture.md`) |
| `job_name` | The string passed to `run_job(job_name, func)` — written to this log regardless of whether it appears in `monitored_jobs.yml`. Must match a `name` in the registry only if that job wants to be covered by the 07:00 digest below; the intraday jobs (see further down this file) intentionally use a `job_name` that isn't in the registry. |
| `status` | `ok` or `fail` |
| `rows` | Row count returned by the job's `main()` (blank on failure) |
| `duration_seconds` | Wall-clock time of `main()` |
| `error_message` | `<ExceptionType>: <message>`, truncated to 500 chars (blank on success) |

`job_logging.run_job(job_name, func)` times `func()`, logs `ok` with its
return value as `rows` on success, or logs `fail` with the exception and
**re-raises** on a crash — so the traceback still lands in the script's own
cron log (`refresh.log` etc.) exactly as before; the structured log is
additive, not a replacement for that.

Every sync script calls `job_logging.run_job()` the same way, regardless of
cadence — logging to `job_runs.csv` is universal (13 distinct `job_name`s
write to it as of 2026-08-28, `warehouse_stock` being the newest). **Seven**
jobs are listed in `monitored_jobs.yml` and covered by the 07:00 digest below:
`sales_snapshot`, `customer_last_price`, `supplier_last_purchase`,
`orders_in_transit` (03:05), `warehouse_stock` (03:10), `cheque_bond_maturity`,
`stock_details`. All seven are nightly and complete before 07:00, so the
digest can honestly report on them.

The remaining jobs deliberately are **not** in that registry: the intraday
syncs `vault_status` (hourly), `vault_movements_hourly` (hourly),
`vault_movements_daily` (nightly), `cek_senet_portfoy` (every 2 hours) — their
first daily run *postdates* the 07:00 digest, so registering them would
falsely report "DID NOT RUN" every morning — and the monthly
`bump_filter_defaults.py` (`30 3 1 * *`), which would falsely report "DID NOT
RUN" on the ~29 days a month it isn't scheduled. All of them use failure-only
Telegram alerting of their own instead.

**`orders_in_transit` and `warehouse_stock` were added to the registry on
2026-08-28** (they had been out of it initially). Both run nightly at
03:05 / 03:10 — well before 07:00 — so the false-"DID NOT RUN" problem never
applied to them. Each originally sent its own Telegram message on *both*
success and failure, which is why they were first kept out; on 2026-08-28
that was reconsidered — a nightly success ping per job is the exact noise the
2026-08-19 consolidation removed for the original jobs — so the per-run
success `send_telegram()` call was **removed from both scripts** and they were
folded into the 07:00 digest like every other nightly job. They still send a
`[orders_in_transit]` / `[warehouse_stock]`-prefixed Telegram alert
**immediately on failure** (pre-flight duplicate-check abort or any
unexpected exception) — matching the failure-only convention every sync job
now follows. `report_job_status.py` needed no change (it just loops the
registry).

All non-registry jobs still write to `job_runs.csv` via `run_job()` and do
their own failure-only Telegram alerting (see each job's own section further
down this file).

The five original daily jobs end with the plain form:

```python
if __name__ == '__main__':
    from job_logging import run_job
    run_job('sales_snapshot', main)  # or 'customer_last_price', 'supplier_last_purchase', 'cheque_bond_maturity', 'stock_details'
```

The other two registry jobs — `orders_in_transit` and `warehouse_stock` —
keep a thin `try`/`except` around `run_job()` so they can still fire an
**immediate failure** Telegram alert (they have no success ping):

```python
if __name__ == '__main__':
    from job_logging import run_job
    try:
        run_job('orders_in_transit', main)
    except Exception as e:
        try:
            send_telegram(f"[orders_in_transit] FAILED: {type(e).__name__}: {e}")
        except Exception:
            print("Telegram alert also failed to send.")
        sys.exit(1)
```

`main()` itself is unchanged except that it now `return`s the total row
count instead of nothing. No other sync logic was touched.

**Telegram note, corrected:** as of 2026-08-19 (when this section was
written), none of the then-four sync scripts sent a per-job Telegram message
themselves — `run_job()` was the only place any of them touched logging, and
nothing touched Telegram. That's no longer true instance-wide: every job
added since (`vault_status`, `vault_movements_hourly`,
`vault_movements_daily`, `cek_senet_portfoy`) wraps its `run_job()` call in
its own `try`/`except` and calls `send_telegram()` directly on failure — see
each job's own section further down this file for the exact message format
and the "Per-job Telegram notification removed" section below for why the
original four still don't.

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

Cron: `0,30 9-19 * * *` — every 30 minutes, 09:00 through 19:00 daily (widened
from hourly on 2026-08-26, per Arslan's request). Confirmed no overlap with
the 02:30/02:45/03:00/03:15/03:30 jobs — entirely different time window.
Checked the script's actual runtime before widening — a consistent ~0.10s
per run across its entire hourly history — so halving the interval to 30
minutes leaves an enormous margin, nowhere close to risking overlap.
Checked the full crontab (not just `vault_movements_hourly`) for any other
job at `:30` inside 09:00–19:00: none — `vault_movements_daily` and
`bump_filter_defaults.py` both run at `:30` but outside this window
(03:30), so no new collision was introduced.

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

**Re-verified (2026-08-26, cadence widened to every 30 minutes):** manual
run loaded 3 rows matching the ERP view exactly, logged `status=ok` in
`job_runs.csv` (`0.10s`, consistent with its entire prior history), fresh
values confirmed directly against `reporting-db` afterward. Cron changed
from `0 9-19 * * *` to `0,30 9-19 * * *`.

## `refresh_vault_movements_hourly.py` / `refresh_vault_movements_daily.py` — vault movement detail sync (2026-08-21)

Two separate scripts, both on athena in `~/reporting-scripts/` (not in this
repo), syncing `reporting.vault_movements` (see `docs/tables.md`) from the ERP
view `aa_vault_movements`. Deliberately two scripts rather than one
parameterized script, matching this codebase's existing pattern of one script
per schedule/scope rather than a shared script with a mode flag.

- **`refresh_vault_movements_hourly.py`** — `SELECT * FROM aa_vault_movements
  WHERE IslemTarihi >= CURDATE() - INTERVAL 4 DAY` from the ERP (widened from
  3 to 4 days on 2026-08-24, per Arslan's request), then a scoped
  `DELETE FROM vault_movements WHERE IslemTarihi >= %s` (same 4-day cutoff,
  computed once in Python and reused for both the source fetch and the
  destination delete so the two windows can't drift apart) followed by chunked
  `INSERT` — never touches rows older than 4 days. Cron: `5 9-19 * * *` —
  hourly, 09:00-19:00, offset 5 minutes past `refresh_vault_status.py`'s `0
  9-19 * * *` deliberately, so the two hourly jobs' ERP connections don't fire
  in the same instant even though they hit unrelated tables (zero-cost
  mitigation, not a fix for a confirmed collision).
- **`refresh_vault_movements_daily.py`** — full `SELECT * FROM
  aa_vault_movements` (no date filter), full `DELETE FROM vault_movements`
  (unconditional) then chunked `INSERT` of every row. **Full replace via
  `DELETE`, not `TRUNCATE`**, for the same reason as `supplier_last_purchase`/
  `cheque_bond_maturity` above: `reporting_writer` has `SELECT, INSERT, UPDATE,
  DELETE` on this table but no `DROP`, and MySQL's `TRUNCATE` requires `DROP`.
  Confirmed live before writing the script (`TRUNCATE TABLE vault_movements` →
  `ProgrammingError 1142 (42000): DROP command denied`), not assumed from the
  other tables' history. Cron: `30 3 * * *` — checked against
  `bump_filter_defaults.py`'s `30 3 1 * *`: same minute, but that job only
  fires on the 1st of the month, so on 28 of ~29 days there's no overlap at
  all, and on the 1st both jobs touch entirely separate systems (this one is
  MySQL-only, `bump_filter_defaults.py` is Metabase-API-only) — no real
  collision either way.

**One-time manual backfill:** `refresh_vault_movements_daily.py` was run by
hand once before the cron entries took over, per this table's spec — 1,306
rows fetched from `aa_vault_movements`, 0 existing rows deleted (fresh table),
1,306 rows inserted, matching the ERP view's row count exactly.

**Deliberately not in `monitored_jobs.yml`:** same underlying reason as
`refresh_vault_status.py` above — the hourly job's first daily run (09:05)
postdates the 07:00 digest, so registering it would falsely cry `DID NOT RUN`
every morning. The daily job (03:30) technically could fit the registry's
daily-freshness check, but was kept out for consistency with its sibling
hourly script and because both share the same failure-only alerting pattern
below rather than the digest's ok/fail-every-day pattern.

**Telegram alerting — failure only:** both scripts send a Telegram message
only on failure/exception, prefixed `[vault_movements_hourly]` and
`[vault_movements_daily]` respectively — same bot, same `.env`, same reasoning
as `refresh_vault_status.py` (hourly cadence would flood the channel with
success pings otherwise). Both call `job_logging.run_job(...)` (job names
`vault_movements_hourly` / `vault_movements_daily`) so every run lands in
`job_runs.csv` for local history, independent of the registry/digest.

**Verified (2026-08-21):** both scripts run manually first (backfill above,
plus a real hourly run afterward) with correct row counts and no Telegram
message on success. Forced-failure test on each (`ERP_DB_PASSWORD` env
override, same method as `refresh_vault_status.py`'s test) crashed with the
expected `ProgrammingError` 1045, logged `status=fail` in `job_runs.csv`,
exited non-zero, sent the `[vault_movements_hourly]`/`[vault_movements_daily]`
Telegram alert, and left `vault_movements` unaffected (crashed before any
write).

**Window widened 3→4 days (2026-08-24):** `refresh_vault_movements_hourly.py`'s
cutoff changed from `timedelta(days=3)` to `timedelta(days=4)` per Arslan's
request. `refresh_vault_movements_daily.py` is unaffected — it already does a
full unconditional replace every night regardless of window, so no backfill
was needed; the wider window only changes how far back the *hourly* job
re-syncs during the day. Verified live: a manual run on 2026-08-24 fetched 20
rows (cutoff `2026-08-20`) vs. 9 rows on the prior cron run under the old
3-day window (cutoff would have been `2026-08-21`), logged `status=ok` in
`job_runs.csv`.

## `refresh_cek_senet_portfoy.py` — portfolio çek/senet sync (2026-08-23)

Lives on athena in `~/reporting-scripts/refresh_cek_senet_portfoy.py` (not in
this repo), same `.env`/connection-handling convention as the other sync
scripts. Syncs `reporting.cek_senet_portfoy` (10 rows — see `docs/tables.md`)
from the ERP view `aa_cek_senet_portfoy`.

Cron: `0 9-19/2 * * *` — every 2 hours on the hour, firing at 09:00, 11:00,
13:00, 15:00, 17:00, 19:00 daily. These are the same top-of-hour slots as
`refresh_vault_status.py`'s `0 9-19 * * *`, so the two jobs' ERP connections
do fire simultaneously on those six slots — not offset like
`vault_movements_hourly` was deliberately offset from `vault_status`. Not
treated as a real collision: the two jobs hit entirely different source
views and destination tables, so there's nothing to contend over beyond two
concurrent read connections to the same ERP box, which it comfortably
handles (same as the existing 3-way overlap at 03:00-ish already tolerated
among the nightly jobs).

**Mandatory pre-flight duplicate check, unique to this job:** before
touching the destination table, the script fetches all source rows and
does a Python-side `groupby (Firma, CekSiraNo)` (the table's primary key),
checking for any group with `count > 1`. If found, it raises before any
`DELETE`/`INSERT` — the exception message states the duplicate-group count
and that the job aborted, which flows through the normal failure path (see
below) rather than a bespoke early-exit, so it still lands in `job_runs.csv`
via `job_logging.run_job`. Rationale (from the task spec): `(Firma,
CekSiraNo)` being the real primary key is an assumption about the source
view's shape, and if that assumption is ever wrong, a silent
`DELETE`+`INSERT` would quietly collapse duplicate rows — this check makes
that failure loud instead. **Not yet verified against real duplicate data**
— fabricating a `(Firma, CekSiraNo)` collision would require writing to the
read-only ERP source, which is out of scope here; the check has been
verified by code review only (the 10-row live source currently has none).

**`DELETE`, not `TRUNCATE`:** same discovery process as
`vault_movements_daily` — the spec called for `TRUNCATE`, but the first live
run failed with `ProgrammingError 1142 (42000): DROP command denied to user
'reporting_writer'`, confirming `reporting_writer` has no `DROP` grant on
this table either. Switched to an unconditional `DELETE FROM
cek_senet_portfoy` (same end state for this small table) before chunked
`INSERT`, consistent with `supplier_last_purchase`/`cheque_bond_maturity`/
`vault_movements_daily`. No grants were touched to work around this, per the
task's constraints.

**Deliberately not in `monitored_jobs.yml`:** same reasoning as
`vault_status`/`vault_movements_hourly` — the first daily run (09:00)
postdates the 07:00 digest, so registering it would falsely cry `DID NOT
RUN` every morning.

**Telegram alerting — failure only (changed 2026-08-24):** originally sent a
message on both success and failure, per the original task spec's explicit
instruction — a deliberate deviation from the failure-only pattern used by
`vault_status`/`vault_movements_hourly`/`vault_movements_daily`. Arslan asked
for that changed on 2026-08-24 after the every-2-hours success pings became
noise, so the success-path `send_telegram(...)` call was removed — it now
matches its siblings exactly: silent on success, alerts only on
failure/exception, prefixed `[cek_senet_portfoy]`. Same bot
(`@cansun_reporting_bot`, `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` from the
same `.env`). Still calls `job_logging.run_job('cek_senet_portfoy', main)`
so every run (success or failure) lands in `job_runs.csv`, independent of
the registry/digest.

**Verified (2026-08-23):** manual run loaded 10 rows matching the ERP view
exactly, logged `status=ok` in `job_runs.csv`, and (under the original
both-success-and-failure design) sent the `[cek_senet_portfoy] OK: 10 rows
loaded.` Telegram message. Forced-failure test (`ERP_DB_PASSWORD` env
override, same method as the other jobs' tests) crashed with the expected
`ProgrammingError` 1045, logged `status=fail` with the error detail in
`job_runs.csv`, exited non-zero, and sent the `[cek_senet_portfoy] FAILED:`
Telegram alert. `cek_senet_portfoy` table contents were unaffected by the
forced failure (crashed before any write) — reconfirmed at 10 rows
afterward.

**Re-verified (2026-08-24, after switching to failure-only):** manual run
loaded 10 rows, logged `status=ok` in `job_runs.csv`, sent **no** Telegram
message (success is now silent, matching `vault_status`). Forced-failure
test (same `ERP_DB_PASSWORD` override method) still crashed with the
expected `ProgrammingError` 1045, logged `status=fail`, exited non-zero, and
still sent the `[cek_senet_portfoy] FAILED:` Telegram alert — failure
alerting confirmed intact after the change. Table contents unaffected
(reconfirmed at 10 rows).

## `refresh_stock_details.py` — nightly stock/product catalog sync (2026-08-25)

Lives on athena in `~/reporting-scripts/refresh_stock_details.py` (not in
this repo), same `.env`/connection-handling convention as every other sync
script. Syncs `reporting.stock_details` (40,104 rows — see `docs/tables.md`)
from the ERP view `cansun.aa_rapor_stok_listesi`.

Cron: `45 3 * * *` — nightly, 30 minutes after `cheque_bond_maturity` and 15
minutes after `refresh_vault_movements_daily.py` (the job in the neighboring
03:30 slot), which consistently finishes in ~0.25s per `job_runs.csv` —
confirmed before finalizing the time, not assumed safe.

**`TRUNCATE`, not `DELETE`+`INSERT`:** unlike every table added since
`cheque_bond_maturity`, `reporting_writer` genuinely has `DROP` on
`stock_details` — confirmed live via `SHOW GRANTS FOR CURRENT_USER()`
before writing the script, not assumed from the newer tables' pattern. So
this job uses the simpler `TRUNCATE` + chunked `INSERT` shape, matching
`customer_last_price`'s pattern instead of the DROP-less tables' `DELETE`.

**No pre-flight duplicate check needed:** `StkoKodu` (the primary key) was
confirmed unique on the live source view before writing the script
(`COUNT(*) = COUNT(DISTINCT StkoKodu)`, both 40,104) — unlike
`cek_senet_portfoy`, there's no fragile multi-column key assumption here to
guard against.

**Registered in the daily digest, no standalone Telegram:** this is a
nightly job on the same cadence as the original four, so it was added to
`monitored_jobs.yml` (see above) rather than given its own
`send_telegram()` call — the 07:00 digest covers it like
`sales_snapshot`/`customer_last_price`/`supplier_last_purchase`/
`cheque_bond_maturity`.

**Verified (2026-08-25):** manual run fetched and loaded 40,104 rows,
matching the ERP view's row count exactly, in 6.66s; logged `status=ok` in
`job_runs.csv`. Column list and placeholder counts (31 each) checked to
match exactly between the `SELECT` and `INSERT` statements before the run,
per this project's explicit-column-list convention. Destination row count
and a 3-row spot check confirmed directly against `reporting-db` after the
run.

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

## `refresh_sales_snapshot.py` — prune-on-sync added (2026-08-25)

**Root cause found:** the stage-then-`REPLACE INTO` merge this script has always used only ever adds/updates rows whose `(Firma, ID)` key appears in the current 4-month ERP fetch — it has no mechanism to remove a row that simply stops appearing (e.g. an invoice corrected/reissued under a new ID in the ERP). Every such correction left a permanent stale duplicate in `sales_snapshot`, invisibly inflating revenue totals. Found via a customer-reported total mismatch (HesapKodu `S 35831`, August 2026), then confirmed table-wide: see `docs/tables.md` for the full one-time cleanup numbers (90 stale rows, 2,490,380.13 TL, across 3 HesapKodu, deleted only after a dry-run `SELECT` was shown to and confirmed by Arslan).

**Fix:** added a `DELETE ... WHERE Tarih >= <cutoff> AND NOT EXISTS (matching key in sales_staging)` step immediately after the merge, using a cutoff computed **once** (`SELECT DATE_SUB(CURDATE(), INTERVAL 4 MONTH)` against the ERP connection) and passed as a bound parameter to both the fetch and the prune — deliberately not two independently-evaluated `CURDATE()` calls against two different MySQL servers (ERP vs. `reporting-db`), which could in principle drift apart.

**Logging decision, documented rather than silently made:** `main()` still returns only `total_inserted` (merged-row count), so `job_runs.csv`'s `rows` field keeps its existing meaning — comparable across this job's entire history, and unchanged for every other job that reads/writes that same shared CSV via `job_logging.py`. The prune count is printed to stdout (`Pruned N stale rows...`, captured in `refresh.log` exactly like every other diagnostic line this script already prints) rather than added as a new column to `job_runs.csv`'s schema. Extending that schema would touch all eleven jobs' log format and `report_job_status.py`'s parsing for the sake of one job's second number — judged out of proportion to what was asked, so this is a deliberate deviation from a literal "prune count shows up in job_runs.csv" reading of the request. Flagging it here in case that tradeoff should be revisited (e.g. if prune-count anomalies need to be visible without reading `refresh.log`).

**Verified (2026-08-25):**
- Manual run after the fix: `68,007 rows merged, 0 stale rows pruned` (correctly zero — the one-time cleanup had already removed the only stale rows that existed at the time), zero errors, logged `status=ok` in `job_runs.csv`.
- Table-wide re-check: rows with `Tarih >= cutoff` in `sales_snapshot` (68,007) exactly equals the fresh ERP fetch's row count — zero orphans anywhere in the table, not just for `S 35831`.
- Rows older than the rolling window structurally cannot be touched (both the merge and the prune are scoped to `Tarih >= cutoff`) — confirmed live: 618,498 pre-window rows, spot-checked a few directly (e.g. ID 495, 2023-09-06, unchanged).
- Spot-checked known-good current IDs (152769, 152775, 152780 — the real, still-current `S 35831` Almer invoices) survived the prune untouched.

## `refresh_orders_in_transit.py` — nightly in-transit ("Yolda") orders sync (2026-08-27)

Lives on athena in `~/reporting-scripts/refresh_orders_in_transit.py` (not in
this repo), same `.env`/connection-handling convention as every other sync
script. Syncs `reporting.orders_in_transit` (1,852 rows on first sync — see
`docs/tables.md`) from the ERP view `cansun.aa_rapor_yolda_hepsi`. This is the
**12th** distinct `job_name` in the shared `job_runs.csv` schema.

Cron: `5 3 * * *` — nightly at 03:05, in the gap between `customer_last_price`
(02:45) and `stock_details` (03:45). Collision check before finalizing, from
`job_runs.csv`: the nearest neighbour is `supplier_last_purchase` at 03:00,
which starts ~03:00:14 and finishes in ~12s (done well before 03:01);
`cheque_bond_maturity` at 03:15 is untouched. This job's own first live run
took **1.00s** for 1,852 rows, so 03:05 has a wide margin on both sides —
worth reconfirming after a few real cron cycles rather than treating it as
permanently final, per usual practice here.

**athena timezone — checked, and it's fixed:** `timedatectl` on athena now
reports `Europe/Istanbul (+03)` with the clock synchronized, so `5 3 * * *`
fires at 03:05 Istanbul wall-clock as intended. The old project note about
athena still running UTC (which would have fired this 3 hours late) no longer
applies — verified live on 2026-08-27, not assumed.

**Mandatory pre-flight duplicate check:** same pattern as
`cek_senet_portfoy` — before touching the destination the script fetches all
source rows and does a Python-side `Counter` over `(Firma, ID)` (the table's
primary key), raising `RuntimeError` with the offending group count if any
key repeats. The exception flows through the normal failure path
(`job_logging.run_job` → `status=fail` in `job_runs.csv` → `[orders_in_transit]
FAILED:` Telegram → non-zero exit), writing **nothing** to the destination.
Rationale: `(Firma, ID)` uniqueness is an assumption about the source view,
and a chunked `REPLACE INTO` on top of a collision would silently collapse
real rows. Verified collision-free on the live 1,852-row source (1,852
distinct pairs); the raise path itself is code-review-only (fabricating a
collision needs a write to the read-only ERP source).

**`REPLACE INTO`, not `TRUNCATE`:** `reporting_writer` has
`SELECT, INSERT, UPDATE, DELETE` but no `DROP` on `orders_in_transit`
(confirmed live via `SHOW GRANTS`), same as `cek_senet_portfoy` /
`supplier_last_purchase`. Chunked `REPLACE INTO` on `(Firma, ID)`,
`CHUNK_SIZE = 5000` matching `customer_last_price`. No staging table.

**Prune step, built in from day one:** after the upsert the script reads back
every `(Firma, ID)` in `orders_in_transit`, subtracts the set of keys this
run pulled — held in **one** shared in-memory variable (`pulled_keys`), never
re-queried from a second source call that could drift from the fetch, the
same once-and-reuse intent as `sales_snapshot`'s shared cutoff — and
`DELETE`s the difference in chunks. This matters here specifically: an order
leaves "Yolda" status (its `SiparisDurumu`/`Kalan` changes on receipt) and
drops out of the source view without ever reappearing under the same `ID`, so
`REPLACE INTO` alone would leave every received order in the table forever.

**Logging split, same convention as `sales_snapshot`:** `main()` returns only
the final table row count, so `job_runs.csv`'s `rows` field keeps its
cross-job meaning. The prune count goes to stdout (`Pruned N stale row(s)...`)
and is captured in `~/reporting-scripts/refresh_orders_in_transit.log` (this
job's own cron log), **not** added as a column to `job_runs.csv`.

**Telegram — failure only; in the 07:00 digest (changed 2026-08-28):**
originally this job sent a Telegram message on *both* success and failure
(per its task spec) and was kept out of `monitored_jobs.yml` for that reason.
On 2026-08-28 that was reversed — a nightly per-job success ping is the exact
noise the 2026-08-19 consolidation removed for the original jobs, and since
`orders_in_transit` runs at 03:05 (before the 07:00 digest) there was never a
false-"DID NOT RUN" obstacle to registering it. The success-path
`send_telegram()` call was removed; the script now keeps only a thin
`try`/`except` around `run_job()` that fires `[orders_in_transit] FAILED:
<ExceptionType>: <msg>` **immediately on any crash** — including the
pre-flight duplicate-check abort (a `RuntimeError` raised inside `main()`,
re-raised by `run_job`). Same bot (`@cansun_reporting_bot`,
`TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` from the same `.env`). Added to
`monitored_jobs.yml` with `expected_time: "03:05"`, so the 07:00 digest now
reports its row count / status like every other nightly job.

**Metabase:** backs the native question `Yoldaki Ürün Listesi` (card 104) in
the `3. İthalat/İhracat` collection (id 10), querying `orders_in_transit`
directly. No dashboard attachment. No group-level permission override — it
inherits folder 3's existing cascade (`Director`, `Manager`,
`Sales/Purchase`, `Satış` — see `docs/metabase-permissions.md`).

**Verified (2026-08-27):**
- First manual run: fetched 1,852 rows from the ERP view, upserted all 1,852,
  pruned 0, logged `status=ok` (`rows=1852`, `1.00s`) in `job_runs.csv`. (Under
  the original both-success-and-failure design it also sent the
  `[orders_in_transit] OK: 1852 rows in transit.` Telegram — removed 2026-08-28.)
- Prune path: inserted a synthetic stale row (`Firma='Cansun Yolda'`,
  `ID=-999`) directly, re-ran — script reported `Pruned 1 stale row(s)`, table
  back to 1,852.
- Idempotency: a third clean run on unchanged data upserted 1,852, pruned 0,
  left the table at exactly 1,852.
- Almer is not in the source view yet — when it is added, the `(Firma, ID)`
  PK must be re-verified collision-free across all three companies (noted in
  `docs/tables.md`); the pre-flight check will abort rather than corrupt if it
  isn't.

**Re-verified (2026-08-28, after removing the success ping + registry add):**
manual daytime run loaded 1,852 rows, logged `status=ok` (`rows=1852`,
`0.83s`) in `job_runs.csv`, sent **no** Telegram message. `report_job_status.py`
dry run (registry + today's log → `build_message`, no send) now lists
`✅ orders_in_transit: ok — 1852 rows in 0.83s` with no "DID NOT RUN" flag.
Failure path unchanged (the outer `except` still calls
`send_telegram("[orders_in_transit] FAILED: …")`).

## `refresh_warehouse_stock.py` — nightly warehouse on-hand quantities sync (2026-08-28)

Lives on athena in `~/reporting-scripts/refresh_warehouse_stock.py` (not in
this repo), same `.env`/connection-handling convention as every other sync
script. Syncs `reporting.warehouse_stock` (17,029 rows on first sync — see
`docs/tables.md`) from the ERP table `cansun.eryaz_zeus_stok_slim`. This is the
**13th** distinct `job_name` in the shared `job_runs.csv` schema.

Cron: `10 3 * * *` — nightly at 03:10, in the 10-minute gap between
`orders_in_transit` (03:05) and `cheque_bond_maturity` (03:15). Checked
against real cron data, not assumed: the 2026-08-28 cron run of
`orders_in_transit` started 03:05:03 and finished in ~1s per `job_runs.csv`,
and `warehouse_stock`'s own first manual run took **0.62s** for 17,029 rows
(0.70s on a second run), so 03:10 clears `orders_in_transit` by minutes and
leaves ~5 minutes before `cheque_bond_maturity`. **Still worth reconfirming
once a few real 03:10 cron cycles exist** — per the spec, "approximately
03:10 is safe" is not something to take on faith long-term; compare the
actual `orders_in_transit` and `warehouse_stock` start/finish times in
`job_runs.csv` after it's been running alongside for a few nights.

**athena timezone — checked, and it's fixed:** `timedatectl` reports
`Europe/Istanbul (+03)`, clock synchronized, so `10 3 * * *` fires at 03:10
Istanbul wall-clock. Same finding as the `orders_in_transit` section above —
the old "athena still on UTC" project note no longer applies to the host
(verified live 2026-08-28). The reporting-db MySQL *container* is still UTC,
but this job writes no timestamp column, so that doesn't matter here.

**Full replace via `DELETE` + chunked `INSERT`, not `TRUNCATE`:** the task
spec said `TRUNCATE` (pointing at `customer_last_price`'s pattern), but
`customer_last_price` has a `DROP` grant and `warehouse_stock` does not —
`reporting_writer`'s grant here is `SELECT, INSERT, UPDATE, DELETE`
(confirmed via `SHOW GRANTS FOR CURRENT_USER()` before writing the script,
not discovered via a failed run this time), and MySQL's `TRUNCATE` needs
`DROP`. So an unconditional `DELETE FROM warehouse_stock` then chunked
`INSERT` (`CHUNK_SIZE = 5000`), same substitution already used by
`cek_senet_portfoy` / `vault_movements_daily`, same end state for a
full-replace table. Not a `REPLACE INTO` upsert: no `Firma` dimension, single
row per `StokKodu`, and the ERP source is itself a nightly full-rebuild, so
wipe-and-reload is the honest match. No staging table, no prune step.

**Column projection:** the `SELECT` pulls only 5 of the source's 7 columns —
`Code, QTY_200, QTY_210, QTY_500, QTY_510` — and renames them on write
(`Code→StokKodu`, `QTY_200→Catalca`, `QTY_210→CatalcaMalKabul`,
`QTY_500→Merkez`, `QTY_510→MerkezMalKabul`). `QTY_230` / `QTY_530` (the İade
buckets) are excluded by design per Arslan — see `docs/tables.md`.

**Telegram — failure only; in the 07:00 digest (changed 2026-08-28):** the
job originally sent its own `[warehouse_stock]`-prefixed Telegram on *both*
success and failure (per its task spec) and was left out of
`monitored_jobs.yml` for that reason. On 2026-08-28 — the same day it was
built — that was reversed alongside `orders_in_transit`: the per-run success
ping is the noise the 2026-08-19 consolidation removed, and the job runs at
03:10 (before the 07:00 digest), so nothing blocked registering it. The
success-path `send_telegram()` call was removed; the script keeps only a
thin `try`/`except` around `run_job()` firing `[warehouse_stock] FAILED:
<ExceptionType>: <msg>` **immediately on any crash**. Added to
`monitored_jobs.yml` with `expected_time: "03:10"`; still calls
`job_logging.run_job('warehouse_stock', main)` so every run lands in
`job_runs.csv` and the 07:00 digest. Same bot (`@cansun_reporting_bot`,
`TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` from the same `.env`).

**Metabase:** the table was made visible for future native-SQL question
building via a schema rescan (`POST /api/database/3/sync_schema` on
`reporting-db`, 2026-08-28) — it now shows as table id 20 with all 5 fields.
**No question or dashboard was built** — report shape is a follow-up from
Arslan, per the spec.

**Verified (2026-08-28, initial build):**
- First manual run: fetched 17,029 rows, inserted all 17,029, logged
  `status=ok` (`rows=17029`, `0.62s`) in `job_runs.csv`. (Under the original
  both-success-and-failure design it also sent the `[warehouse_stock] OK:
  17029 rows loaded.` Telegram — removed later the same day.)
- Column mapping spot-checked against the ERP source for 3 codes
  (`3RG_10109`, `AE_V91925`, `AE_V91926`) — all 5 mapped values matched
  exactly.
- Idempotency: a second run reloaded 17,029 → 17,029, `0.70s`, no drift.
- `DELETE` path confirmed working under the no-`DROP` grant (no `1142` error).

**Re-verified (2026-08-28, after removing the success ping + registry add):**
manual daytime run loaded 17,029 rows, logged `status=ok` (`rows=17029`,
`0.67s`), sent **no** Telegram message. `report_job_status.py` dry run now
lists `✅ warehouse_stock: ok — 17029 rows in 0.67s` with no "DID NOT RUN"
flag. All 7 registry jobs resolved to a logged run (`jobs with no run today:
[]`).