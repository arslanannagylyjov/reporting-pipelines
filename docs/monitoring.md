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