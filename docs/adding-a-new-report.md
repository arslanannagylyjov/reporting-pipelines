# Playbook: Adding a New Report

Step-by-step process for taking a new report from "there's an ERP view for
this" to "it's a monitored, notified, permission-scoped Metabase question."
Written generically so it's the reference for the *next* new table, not a
description of any one report — `supplier_last_purchase` (added 2026-08-14)
is used throughout as the worked example, but every step applies regardless
of which table you're adding.

## 0. Before you start

Read the most recently added sibling script in `~/reporting-scripts/` on
athena (not in this repo — scripts live only on athena, see `README.md`).
The new script should look like a sibling of the existing ones: same
connection pattern, same logging style, same `.env` loader. Don't invent a
new pattern unless the existing one genuinely doesn't fit.

## 1. ERP-side view

Confirm the source view exists on the ERP MySQL server and is genuinely
ready to sync from — not a table you're about to build a view on top of.
Check whether it already covers every company you need (some views UNION
all companies behind a `Firma`/company column; query it once, don't query
per-schema/per-company if the view already handles that).

## 2. Grant 1 of 3 — ERP-side `metabase_ro` → `SELECT` on the new view

The read-only ERP account the sync script uses to pull data
(`metabase_ro@<erp-host>`, distinct from the reporting-db account of the
same name — see Grant 3). Confirm live, don't assume: connect as this
account and query the view directly. If it fails, this grant is missing or
scoped wrong and needs fixing before anything else.

## 3. athena-side destination table

Create the table on `reporting-db` (MySQL container on athena) with an
appropriate primary key — usually matching whatever the source view already
deduplicates on. Decide the replace strategy *before* writing the sync
script, and base it on the actual grant you're about to apply in step 4, not
on copying the most recent sibling script's strategy blind:

- If the write account will have `DROP` on this table → `TRUNCATE` + chunked
  `INSERT` is simplest (see `customer_last_price`'s pattern).
- If it won't have `DROP` (e.g. grant is `SELECT, INSERT, UPDATE, DELETE`
  only) → `TRUNCATE` will fail. Use upsert-then-prune instead: capture a
  `run_start` timestamp from the destination DB's own `NOW()` (not the
  sync script's host clock — see the timezone note below), upsert every
  source row via `INSERT ... ON DUPLICATE KEY UPDATE` setting a `SyncedAt`
  column to that captured `run_start`, then `DELETE FROM <table> WHERE
  SyncedAt < run_start` to prune rows no longer in the source. This is what
  `supplier_last_purchase` does.

**Timezone trap:** if the `reporting-db` container doesn't have a `TZ` env
var set (check `docker exec <container> date` and compare to the host's
`date` — see `docs/server-architecture.md`), its MySQL clock runs on the
*container's* system time, which may be UTC even though the athena host
itself is Europe/Istanbul. If your replace strategy compares a timestamp
written by MySQL (`SyncedAt`) against a timestamp computed in Python, and
they come from two different clocks, the comparison will be systematically
wrong — this bit `supplier_last_purchase` on its first real run (every row
got pruned immediately because the Python-side timestamp had sub-second
precision the DB column didn't, making every stored value look "older" than
the cutoff even though they were written in the same statement). Fix: pull
the cutoff timestamp from the destination DB itself (`SELECT NOW()`, not
`NOW(6)` unless the column's fractional-second precision actually matches),
and never mix a host-clock timestamp with a container-clock timestamp in
the same comparison.

## 4. Grant 2 of 3 — athena `reporting_writer` → write access on the new table

The account the sync script uses to write to `reporting-db`. Grant exactly
what the chosen replace strategy needs (see step 3) — `SELECT, INSERT,
DELETE` if using stage-and-merge, `SELECT, INSERT, DROP` if using
`TRUNCATE`, or `SELECT, INSERT, UPDATE, DELETE` if using upsert-then-prune.
**Don't assume a stated grant is correct — verify it live** (`SHOW GRANTS
FOR 'reporting_writer'@'<host>'`, or if you lack the privilege to run that
directly, query `information_schema.table_privileges` as that account, or
infer it from what actually succeeds/fails when the script runs) before
writing code that depends on it.

## 5. Grant 3 of 3 — athena reporting-db's *own* `metabase_reporting_ro` → `SELECT` on the new table

**This is the grant that gets missed.** It's a *third*, separate MySQL
account — not the ERP-side `metabase_ro` from step 2, and not the
`reporting_writer` from step 4. It's the account Metabase itself uses for
its *own* connection to `reporting-db` (`Admin > Databases >
metabase_reporting_db > Edit connection details`), and it needs its own
`SELECT` grant on every table you want Metabase to be able to see — the
write account having full access does not help Metabase at all, because
Metabase never connects as the write account.

Symptom if this grant is missing: the table exists, is populated, and the
sync job succeeds — but a manual re-scan (`Admin > Databases > Sync
database schema`) silently does not pick up the new table. No error is
shown in the UI. To confirm this is actually the cause (not a sync timing
issue), check the sync task log via the API: `GET
/api/task?limit=50&sort_column=started_at&sort_direction=desc`, find the
most recent `sync-tables` task for the right `db_id`, and read its `logs`.
Each table Metabase *can* see logs `"<table>": SELECT privileges
confirmed`; a table it can't see is simply absent from that list entirely
— it's not attempted, not denied, just missing. That absence is the
confirmation this grant is missing.

Root access is required to apply this grant (same as steps 2 and 4) — if
you don't have it, get someone who does to run it, then verify live by
triggering a re-scan and re-checking the `sync-tables` task log for the
table name before proceeding to step 8.

## 6. Sync script

Write `~/reporting-scripts/refresh_<table>.py` on athena directly (never
committed to this repo — see `README.md`). Match the sibling script read in
step 0: same `_load_env` pattern, same `SOURCE`/`DEST` connection dict
shape, same chunked-batch approach for large row counts. End the script
with:

```python
if __name__ == '__main__':
    from job_logging import run_job
    run_job('<table>', main)
```

`main()` should `return` the final row count. Log start time, end time, row
count, and success/failure — confirm the host's system timezone is actually
Europe/Istanbul (`date` on athena) rather than assuming it, since
`datetime.now()` just follows whatever the host clock says.

## 7. Cron entry

Before adding the cron line, check the actual recent run durations of
whatever jobs run near your intended time slot — pull real numbers from
`job_runs.csv`, don't guess. If the new job's start time is tight against
another job's typical finish time, say so and get a decision before adding
it rather than risking an overlap. Add the entry to `crontab -e` following
the existing lines' format, and add a matching entry to
`~/reporting-scripts/monitored_jobs.yml` (see `docs/monitoring.md` — this
is what makes the 07:00 daily digest aware of the new job).

## 8. Telegram notification

Two distinct things exist here — don't conflate them:

- The **daily 07:00 digest** (`report_job_status.py`) automatically covers
  any job listed in `monitored_jobs.yml` (step 7) — nothing extra needed
  for this part.
- A **per-job, immediate notification** (sent right after every run,
  success or failure) is a separate thing the daily digest does *not*
  give you for free — it has to be added explicitly inside the new
  script itself. Reuse the existing bot token, chat ID, and
  `send_telegram()` HTTP-POST pattern (same `.env` file, same
  `api.telegram.org/bot<token>/sendMessage` call) — copy the small helper
  into the new script rather than creating a second bot or duplicating
  config; this codebase's existing convention is small self-contained
  per-script helpers (see `_load_env`, duplicated the same way across
  every script), not a shared importable module.

Message content should include: a job name clearly distinguishable from
every other job in the chat history, success/failure status, row count, and
a human-readable completion timestamp in Europe/Istanbul time using the
**athena host's** local clock — not a database container's clock, which may
be on a different timezone (see the timezone trap in step 3).

**Verify by triggering a manual run and confirming a message actually
arrives for this specific job** — don't treat "the bot already works for
other jobs" as proof this job is wired up; the notification call has to be
added and tested per script.

## 9. Metabase: schema sync

`Admin > Databases > <database> > Sync database schema`. The new table
won't appear until this runs (or the next scheduled auto-sync). If it
doesn't appear after a manual sync, this almost always means grant 3 of 3
(step 5) is missing — check the `sync-tables` task log as described there
before assuming something else is wrong.

## 10. Metabase: question creation

Build a native SQL question (not the GUI query builder, if the existing
convention in the target collection uses native SQL) against the new
table. If there's an existing question with a similar purpose, open it
first and copy its Field Filter pattern exactly: variable type "Field
Filter" mapped to the relevant column, matching filter widget type (e.g.
"String contains"), and matching "single value" vs. "multiple values"
setting — consistency here matters more than any individual choice.

Name it to match the existing naming convention in the target collection
(language, capitalization, level of formality).

## 11. Collection placement

Place the new question in whichever existing collection already matches
its audience — don't create a new collection unless the report genuinely
doesn't fit any existing one. Don't touch group permissions on that
collection unless live verification (step 12) actually shows something
wrong; an existing collection's access was very likely already deliberately
configured for exactly this purpose.

**Only finished, Director-facing dashboards go directly in
`1. Yöneticiler`.** Any native SQL question built purely as a component
for a dashboard card is created directly inside `_Hesaplama Kaynağı` from
the start — never placed at the top level and moved later. This keeps the
top level of `1. Yöneticiler` as a short, stable list of the actual
deliverables (the dashboards themselves), not cluttered with every
question that happens to feed one of their cards.

If the table has any columns equivalent to known "must stay restricted"
cost/sensitive columns elsewhere in this database, decide and document
whether they're exposed by the new question before moving on — don't
silently skip this decision even when the answer is "not applicable."

## 12. Verification — live, not assumed

Every claim below needs to be checked against the live system, not
inferred from "the config looks right":

- **Sync ran and populated the table:** query row count and
  `MAX(<timestamp column>)` directly against `reporting-db` — don't just
  check that cron didn't log an error.
- **Notification arrived:** trigger a manual run if the next scheduled
  time hasn't hit yet; confirm the message content, not just that the bot
  call didn't throw.
- **Question is visible to the intended group and not to others:** if you
  don't have real user credentials for the groups involved (and per this
  project's credential policy, you generally won't — temporary passwords
  are relayed directly and never recorded), don't skip this check or fall
  back to just reading the permissions config. Two options, in order of
  preference:
  - Create a throwaway test account per group, authenticate it via
    `POST /api/session` to get an independent session token, and query
    `/api/card/<id>` and `/api/collection/<id>/items` with that token.
    This avoids ever touching the admin browser's own session.
  - If a live login through the browser is unavoidable, be aware that
    logging out of an admin session can invalidate it server-side even if
    you intend to log back in — this has happened before on this project
    and required a password-reset CLI recovery on the server. Prefer the
    API-token approach above.
  - Deactivate every throwaway test account immediately after — don't
    leave them active.
- Report verification results with the actual output (row counts,
  timestamps, HTTP status codes, message IDs) — not just "confirmed" or
  "done."

## 13. Documentation

Update `docs/tables.md` (schema, source, sync strategy, row-count order of
magnitude), `docs/monitoring.md` (new cron entry, expected row-count range
for future anomaly spotting, and the per-job notification if one was
added), and `docs/metabase-permissions.md` (new question, which collection,
and explicitly note if no permission changes were made). Redact: no real
names or emails in any committed doc — use the existing `Director #1/#2`,
`Sales/Purchase #1/#2` convention if a person needs referencing.
