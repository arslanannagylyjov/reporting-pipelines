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
```

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
| `timestamp` | Run start time, ISO 8601, second precision |
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

Both sync scripts now end with:

```python
if __name__ == '__main__':
    from job_logging import run_job
    run_job('sales_snapshot', main)  # or 'customer_last_price'
```

`main()` itself is unchanged except that it now `return`s the total row
count instead of nothing. No other sync logic was touched.

## Report — `report_job_status.py`

Cron'd daily at **07:00** on athena
(`~/reporting-scripts/report_job_status.log` captures its own output). For
every job in the registry, using only today's log entries (most recent one if
a job ran more than once):

- No entry for today → **`❌ <name>: DID NOT RUN today (no log entry)`**
  — checked and reported first; this is the priority signal, since a crash at
  least leaves a trace but a job that silently stopped running leaves nothing.
- Latest entry is `status=fail` → **`❌ <name>: FAILED — <error_message>`**
- Latest entry is `status=ok` → **`✅ <name>: ok — <rows> rows in
  <duration_seconds>s`**

Failures and missing jobs are listed first, successes after. The message is
sent via one HTTP POST to the Telegram Bot API
(`api.telegram.org/bot<token>/sendMessage`), using `TELEGRAM_BOT_TOKEN` and
`TELEGRAM_CHAT_ID` from `~/reporting-scripts/.env` — same file, same
stdlib-only loader (`_load_env`) already used by the sync scripts. No new
`.env` file, no secrets added to this repo.

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
