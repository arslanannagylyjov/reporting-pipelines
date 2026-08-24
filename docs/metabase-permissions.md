# Metabase Permission Groups — Director, Sales/Purchase, Other

Added 2026-08-11 on the Metabase instance at `10.20.52.43:3000` (internal
hostname `rapor.wenderparts.int:3000` — does not resolve from outside the
office network; use the IP directly when working remotely). Started as two
groups beyond Metabase's built-in `Administrators` and `All Users`; on
2026-08-13 this was redesigned into a proper collection structure — see
"Collections & permissions redesign (2026-08-13)" below for the current
model. `User` was renamed to `Other`, and a new `Sales/Purchase` group was
added.

## Groups

| Group | Metabase group ID | Intended scope |
|---|---|---|
| **Director** | 5 | Full read access to the `1. Yöneticiler`, `2. Satış/Satın Alma`, and `3. Diğer` collections (renamed with numeric prefixes 2026-08-13 to force sidebar order — see below), plus View-only access to `_Hesaplama Kaynağı` (formerly `Boss Dashboard`, renamed and nested under `1. Yöneticiler` 2026-08-13 — see "Boss Dashboard lockdown regression fix" below). View-only on data — cannot build new questions (see Data permissions). Not a superuser/admin group; has zero implicit access to anything not explicitly granted. |
| **Sales/Purchase** | 7 | Added 2026-08-13. Read access to `2. Satış/Satın Alma` only. Ten real members as of 2026-08-24 — Sales/Purchase #1 through #10 (see "Sales/Purchase provisioning" below). |
| **Other** | 6 | Renamed from `User` on 2026-08-13. Read access to `3. Diğer` only, which is currently empty. No real members yet. |

## Why this exists

**Terminology note (corrected 2026-08-12):** "Boss Dashboard" has been used
loosely throughout this doc to refer to two *different* Metabase objects,
which live in separate ID namespaces and matter differently for permissions:

- The **`Boss Dashboard` collection** (collection ID `6`) — a folder. This is
  what the collection-permissions table below governs.
- The actual **dashboard**, named **"Cansun Satış Genel Bakış"** (dashboard
  ID `3`), which *lives inside* that collection alongside its 14 questions.
  This is the object with the `Genel Bakış` / `Ürünler & Müşteriler` /
  `Coğrafya & Kanal` tabs and the `Tarih`/`Firma` filters — the thing a
  Director-group user is actually meant to land on and look at.

Confusing the two caused a real bug — see "Homepage fix" below: setting a
user's landing page to the *collection* lands them on a folder listing, not
the dashboard. Verified both IDs directly via Metabase's API rather than
trusting the numbers already in this doc, since the mixup showed the
existing IDs weren't self-evidently reliable. Dashboard ID 3 turned out to
be correct all along; it was the homepage *setting* pointing at the wrong
kind of object, not a wrong ID.

## Homepage fix (2026-08-12)

**Symptom:** Director #1 (first real `Director`-group user) logged in and
landed on the `Boss Dashboard` *collection* — a folder listing — instead of
the `Cansun Satış Genel Bakış` dashboard itself.

**Root cause:** `Admin > Settings > General > Homepage` was set to "Default
Metabase home." That default doesn't point at any specific dashboard — it
falls back to the one collection a user can actually see. Since `Director`
only has **View** on the `Boss Dashboard` collection (not on the root `Our
analytics` collection), that fallback landed Director #1 on the collection
folder, not the dashboard inside it. This was never a wrong-ID problem: confirmed
directly via `/api/dashboard/3` that dashboard ID 3 genuinely is `Cansun
Satış Genel Bakış` (tabs `Genel Bakış` / `Ürünler & Müşteriler` / `Coğrafya &
Kanal`, parameters `Tarih`/`Firma`), correctly living inside collection ID 6.

**Fix:** Set `Admin > Settings > General > Homepage` to "Dashboard" →
`Cansun Satış Genel Bakış`. Verified persisted via `/api/setting`:
`custom-homepage: true`, `custom-homepage-dashboard: 3`.

**Verification:** From an Administrator session, navigating to
`http://10.20.52.43:3000/` now redirects to `/dashboard/3?...`, showing the
real dashboard with all three tabs and both filters.

**Confirmed (2026-08-12):** Director #1 refreshed/re-navigated and confirmed
they now land directly on the `Cansun Satış Genel Bakış` dashboard, not the
collection folder. Fix verified end-to-end from both the admin side and a
real `Director`-group user's session.

The dashboard and its questions are built on `sales_snapshot` and include
two cost columns — `FabrikaFiyati` and `FabrikaTutarUsd` — that must never
be visible to non-boss users. `Director` is the group that's allowed to see
them; `User` is the template for everyone else until they're individually
granted access to specific non-cost reports.

## Data permissions (Admin > Permissions > Data > `metabase_reporting_db`)

| Table | Director | User |
|---|---|---|
| `sales_snapshot` | Query builder and native | **No** |
| `customer_last_price` | Query builder and native | **No** |

`sales_staging` doesn't appear in this list for either group — the
Metabase-facing `metabase_reporting_ro` MySQL account has no grant on it, by design
(see `docs/credentials.md`/`docs/tables.md`), so it's invisible to Metabase
entirely regardless of group permissions.

**Note on defaults:** Metabase gives every newly created group full
("Query builder and native") access to all tables by default. `Director`
already matched what we wanted and needed no change. `User` had to be
explicitly set to "No" on both tables — the default would otherwise have
been full access.

## Collection permissions (Admin > Permissions > Collections > Boss Dashboard)

| Group | Access |
|---|---|
| Administrators | Curate (unchanged, built-in) |
| All Users | **No access** (changed from Curate — see below) |
| Director | **View** |
| User | **No access** |

**Important finding, fixed as part of this work:** `All Users` — the group
every Metabase user is automatically a member of, with no way to opt out —
had **Curate** (full edit) access to the `Boss Dashboard` collection by
default. Since Metabase permissions are most-permissive-wins across all of a
user's groups, this would have silently overridden the new `User` group's
restrictions the moment a real, non-director person was added: they'd be in
both `All Users` and `User`, and `All Users`'s Curate access would win. This
had zero live impact today (the only current user, Arslan, is an
Administrator and bypasses group permissions anyway), but it would have
defeated the entire point of the `User` group as soon as real people were
added. Set to "No access" so `Boss Dashboard` is now genuinely gated on
`Director` membership.

`Our analytics` (root collection) and Arslan's personal collection were left
untouched — see the exposure check below for why that's safe.

## Cost-column exposure check (2026-08-11, deepened 2026-08-12)

Checked whether `FabrikaFiyati`/`FabrikaTutarUsd` are reachable from
anywhere in the instance other than the now-locked `Boss Dashboard`
collection:

- Enumerated every collection in the instance: `Our analytics` (root)
  contains only the `Boss Dashboard` folder and question 56 (Müşteri Son
  Fiyat Sorgusu — no cost columns, intentionally left open to All Users per
  Arslan's instruction); `Boss Dashboard` contains the questions + dashboard
  described in the task; Arslan's personal collection is empty.
- Initial pass: ran a Metabase search for "Fabrika" with "search the
  contents of native queries" and "search items in trash" both enabled —
  zero results anywhere in the instance, including trash. This only covers
  native-SQL text and card titles/descriptions, not GUI query-builder column
  selections.
- **Follow-up (2026-08-12), prompted by a direct question about whether GUI
  questions were actually checked:** pulled every card's real query
  definition via Metabase's own `/api/card/<id>` endpoint (returns
  `result_metadata` — the literal output columns — regardless of native or
  GUI query type). This caught a card the manual collection browse had
  missed: card 55 ("2026 Satis Ay Bazinda"), visible on dashboard 3 but not
  rendered in the collection listing at the time it was checked. Traced its
  GUI aggregation down to the actual summed field (`TutarTL`, revenue — not
  a cost column). All 14 cards in the Boss Dashboard collection (13 original
  + card 55) confirmed clean via this method — none select or aggregate
  `FabrikaFiyati`/`FabrikaTutarUsd`. Note: none of the current 14 questions
  expose the cost columns at all today; the raw table's data-permission
  block (above) is what actually stands between them and anyone.
- Also re-verified the `User` group's data-permission block via
  `/api/permissions/graph` (the ground truth Metabase's backend reads),
  not just the admin UI: `User`'s entry for database 3 has no
  `create-queries` key at all, while every group with real access (including
  `Director`) has one explicitly. Cross-checked against the save-time
  confirmation dialog Metabase itself showed when the setting was applied
  ("User will not be able to use the query builder or write native queries
  for metabase_reporting_db") — consistent, not contradictory.
- Conclusion: the two cost columns have no path to any user outside
  `Director` (or an Administrator). `User`'s "No access"/"No" settings above
  are the only thing standing between it and the data, and there's no
  second dashboard, question, or archived item that bypasses that.

## Provisioned accounts

| Account | Groups | User ID | Provisioned | Status |
|---|---|---|---|---|
| Director #1 | Director (+ All Users, automatic) | 2 | 2026-08-12 | Logged in, changed their temporary password — first live login, see below |
| Director #2 | Director (+ All Users, automatic) | 3 | 2026-08-13 | Account created, invited via generated temporary password (no SMTP) — not yet logged in |

**First real (non-admin) account in either group.** Confirmed via
`/api/user` (ground truth, not the admin UI label) that `group_ids` is
exactly `[1, 5]` — `1` is `All Users` (automatic, no opt-out), `5` is
`Director`. No `User` group, no `Administrators`.

**Invite flow:** this instance has no SMTP configured (`Admin > Settings >
Email` shows "Configure," not "Edit"; confirmed via `/api/setting` that
`email-smtp-host` is `null`). Metabase therefore could not send an email
invite and instead generated a one-time temporary password on account
creation, shown once in the admin UI, with the explicit message that it
couldn't send an email invitation and to relay the login email and
generated password directly. Arslan needed to relay the temporary password
to Director #1 through some other channel (it is not recorded here).
Director #1 was prompted to set their own password on first login.

**Why this account matters beyond onboarding:** it's the first live,
real-world test of the permission work above. Director #1's login confirmed
`Director` has full working access to the dashboard and its underlying data
(not just correct on paper), and their post-fix refresh confirmed they land
directly on `Cansun Satış Genel Bakış` rather than the collection folder or
the generic Metabase home — see "Homepage fix" above.

**Second Director account (2026-08-13):** created the same way as Director
#1's, via `Admin > People > Invite someone`, group set to `Director` only.
Confirmed via `/api/user/3` (ground truth, not the admin UI label) that
`user_group_memberships` is exactly `[{"id":1},{"id":5}]` — `1` is `All
Users` (automatic, no opt-out), `5` is `Director`. No `User` group (`6`), no
`Administrators`.

**Invite flow:** same as Director #1's — this instance still has no SMTP
configured (re-confirmed via `/api/setting`: `email-smtp-host` is still
`null`). Metabase generated a one-time temporary password on account
creation, shown once in the admin UI with the same "we couldn't send them an
email invitation" message. Arslan needed to relay the temporary password to
Director #2 directly through some other channel (it is not recorded here).
Director #2 was prompted to set their own password on first login.

**Homepage redirect:** no separate configuration needed for this account.
The Homepage setting fixed above (see "Homepage fix") is instance-wide, not
per-user — it points at dashboard ID 3 regardless of which `Director`-group
user logs in. Since Director #2 is in the same `Director` group as Director
#1 with the same **View**-only access to the `Boss Dashboard` collection,
the same fallback logic that was broken (and is now fixed) applies
identically. Expected to work on first login without any additional change;
not yet confirmed live at the time this was written.

## Collections & permissions redesign (2026-08-13)

Full restructure from the single `Boss Dashboard` collection model to three
flat, purpose-built collections under root, plus a second group
(`Sales/Purchase`) and a hard lockdown of both the root collection and
database-level query-building for everyone except the actual admin
account. Done via Playwright MCP against the live instance (per current
`~/.claude/CLAUDE.md` policy — see the gstack-vs-Playwright cleanup from
2026-08-13), not gstack's `/browse`.

### Groups

- Renamed `User` (group ID 6) → `Other`. Kept its existing zero-access
  config as-is; only the name changed.
- Created `Sales/Purchase` (group ID 7), no members yet — real people are
  explicitly out of scope for this phase per the task.
- `Director` unchanged as a group, but is treated as a plain non-admin
  group throughout this redesign: it has zero implicit access to anything
  new, same as `Other` and `Sales/Purchase`. Every collection below needed
  Director explicitly granted, same as the other two groups.

### Collections (flat, directly under root — `Our analytics`)

| Collection | ID | Director | Sales/Purchase | Other | All Users |
|---|---|---|---|---|---|
| **Yöneticiler** | 9 | View | No access | No access | No access |
| **Satış/Satın Alma** | 10 | View | View | No access | No access |
| **Diğer** | 11 | View | No access | View | No access |

**Same gotcha as the original `Boss Dashboard` setup, hit again:** every
newly created collection defaults `All Users` to **Curate**. Since
permissions are most-permissive-wins across a user's groups, and every
real person is automatically in `All Users`, leaving that default would
have silently overridden every one of the grants above the moment a real
person was added to `Director`/`Sales/Purchase`/`Other`. Set `All Users` to
**No access** on all three new collections, matching the fix already
applied to `Boss Dashboard` back on 2026-08-11.

Confirmed via `/api/collection/graph` (ground truth, not the admin UI
label):

```
"5" (Director):       {"6":"read","9":"read","10":"read","11":"read"}
"6" (Other):           {"11":"read"}
"7" (Sales/Purchase):  {"10":"read"}
```

No entry for a group on a given collection ID means no access — exactly
the intended shape. `Boss Dashboard` (collection 6) itself was left as-is;
see "What was deliberately left alone" below for why.

### Content moved

- Dashboard 3 (`Cansun Satış Genel Bakış`) → moved into `Yöneticiler`.
  Confirmed via `/api/dashboard/3`: `collection_id: 9`.
- Question 56 (`Müşteri Son Fiyat Sorgusu`) → moved into `Satış/Satın
  Alma`. Confirmed via `/api/card/56`: `collection_id: 10`.

**What was deliberately left alone:** the task only named these two
objects to move. The old `Boss Dashboard` collection (ID 6) and its other
13 questions — the actual chart definitions that dashboard 3's tabs
render — were **not** moved or touched. Metabase dashboards reference
cards by ID regardless of which collection the dashboard itself lives in,
so this doesn't break anything: `Director` already had View on `Boss
Dashboard` from the 2026-08-11 setup, and still does, so the dashboard's
cards keep rendering for Director. But it does mean `Boss Dashboard` is
now a slightly orphaned fourth collection sitting alongside the three new
ones, holding content that conceptually belongs in `Yöneticiler`. Flagging
this for Arslan — a follow-up cleanup pass (moving those 13 questions into
`Yöneticiler` and retiring the `Boss Dashboard` collection) would tidy this
up but wasn't requested and wasn't done here.

### Root collection lockdown

`All Users`' access to root (`Our analytics`) — previously **Curate** — is
now **No access**. Nothing can be saved loose at root going forward;
everything must land in `Yöneticiler`, `Satış/Satın Alma`, or `Diğer`.
Confirmed via `/api/collection/graph`: group `1` (`All Users`) has no
`root` key at all now (previously `"root": "write"`).

### Data permissions (Admin > Permissions > Data > `metabase_reporting_db`)

`metabase_reporting_db` is the only connected database on this instance —
confirmed live via `Admin > Permissions > Data`, no `Sample Database` or
anything else present.

| Group | Create queries |
|---|---|
| Administrators | Query builder and native (unchanged) |
| All Users | **No** (changed from Query builder and native) |
| Director | **No** (changed from Query builder and native) |
| Sales/Purchase | **No** (new group, defaulted to full access — set to No) |
| Other | **No** (already set 2026-08-11, unchanged) |

**Same override gotcha as the collections, at the database-permission
layer this time:** changing Director's and Sales/Purchase's data
permission to "No" triggered Metabase's own warning — *"Revoke access even
though 'All Users' has greater access?"* — because `All Users` still had
its default **Query builder and native** access to `metabase_reporting_db`
from setup. Since every real person is automatically in `All Users`,
leaving it untouched would have made the Director/Sales-Purchase/Other
restrictions meaningless. Set `All Users` to **No** as well — this is
deliberate, not a placeholder: **only the actual Metabase admin account(s)
can build new questions going forward.** This also removes the
Databases/Models/Metrics browse entries from the sidebar for everyone
non-admin (Models/Metrics still show as menu items but empty; Databases is
gone entirely) — confirmed live, not assumed.

Confirmed via `/api/permissions/graph`, ground truth: groups `1`, `5`, `6`,
`7` each have `view-data: unrestricted` and `download: full` on database 3
(`metabase_reporting_db`) but **no `create-queries` key at all**, while
group `2` (Administrators) has `create-queries: query-builder-and-native`
plus `data-model`/`details`/`transforms` access. Groups `3` and `4` also
appear in the graph — these are Metabase-internal magic groups (`All
tenant users`, `Data Analysts`), zero members, not visible in `Admin >
People > Groups`, and unrelated to anything managed here; safe to ignore.

### Verification (live, not assumed)

Screenshots taken during verification are in the repo's
`.playwright-mcp/` working directory from this session
(`verify-director-homepage.png`, `verify-root-no-access.png`).

- **No real Director/Sales-Purchase user's credentials were available** to
  log in as (Director #1's and Director #2's temporary passwords were
  relayed to them directly and were never recorded per this doc's own
  policy). Rather than
  skip the live-login check, created a throwaway test account
  (`Test DirectorVerify`, Director group only, temp password shown once in
  the admin UI, not recorded), logged in as it, screenshotted, then
  deactivated it afterward. Confirmed:
  - Sidebar **Data** section shows only `Models` and `Metrics` — no
    `Databases` entry.
  - Sidebar **Collections** shows `Yöneticiler`, `Satış/Satın Alma`,
    `Diğer`, and `Boss Dashboard` (plus the account's own personal
    collection) — all visible.
  - `Cansun Satış Genel Bakış` renders as the account's homepage with all
    three tabs, both filters, and real data. Metabase itself surfaced a
    toast confirming this: *"Your admin has set this dashboard as your
    homepage."*
  - Navigating to `/collection/root` returns **"You don't have permissions
    to do that."**
- **Sales/Purchase** has no real members yet (out of scope per the task),
  so this was a permissions-matrix dry-run rather than a live login — see
  the `/api/collection/graph` and `/api/permissions/graph` output above:
  group 7 has `read` on collection 10 only, and no `create-queries` key on
  the database. Confirms it would see `Satış/Satın Alma` only, not
  `Yöneticiler`, if a real user existed.
- **Cost columns:** re-checked `/api/card/56`'s `result_metadata` directly
  (API-level query-definition inspection, not UI browsing) after the move.
  Columns: `HesapKodu`, `HesapAciklamasi`, `Firma`, `StokKodu`,
  `StokAciklamasi`, `DvzBirimFiyat`, `DovizKodu`, `BelgeTarihi`. No
  `FabrikaFiyati`/`FabrikaTutarUsd` — consistent with the 2026-08-12
  exposure check.

### Incident during verification: admin session lost, regained

While setting up the throwaway test-account login above, the live admin
session was lost — Metabase invalidates sessions server-side on logout, so
a pre-saved session cookie could not restore access. Regained access using
Metabase's own reset-password CLI tool on the server. No other accounts or
data were affected; this was purely a login-recovery step on the admin
account, and verification proceeded normally afterward.

## Sidebar ordering, Boss Dashboard lockdown, Sales/Purchase provisioning (2026-08-13, second pass)

Same day as the redesign above, three follow-up changes. Done via
Playwright MCP, same as before.

### Sidebar ordering

Renamed the three collections to force alphabetical sort into the intended
order:

| Collection | ID | Old name | New name |
|---|---|---|---|
| 9 | — | `Yöneticiler` | `1. Yöneticiler` |
| 10 | — | `Satış/Satın Alma` | `2. Satış/Satın Alma` |
| 11 | — | `Diğer` | `3. Diğer` |

Confirmed live (not assumed): navigated to root and read the actual
sidebar tree order after all three renames —
`1. Yöneticiler` → `2. Satış/Satın Alma` → `3. Diğer` → `Boss Dashboard`.
Matches intent exactly.

### Boss Dashboard lockdown

Set `Director`'s access on the `Boss Dashboard` collection (ID 6) from
**View** to **No access**. Administrators unchanged (Curate, and admins
bypass collection permissions entirely regardless). Confirmed via
`/api/collection/graph`: group `5`'s entry is now `{"9":"read",
"10":"read","11":"read"}` — no `"6"` key at all.

Verified live via a throwaway test account (`Test BossDashVerify`,
Director group, deactivated after): `Boss Dashboard` is gone from both the
sidebar collection tree and from search results (`Didn't find anything`
for a search of "Boss Dashboard").

**Side effect this caused, fixed same day** — see "Boss Dashboard lockdown
regression fix" below: revoking Director's access outright broke 5 of
`Cansun Satış Genel Bakış`'s ~8 visible cards, since those cards' questions
still physically live in this collection. Resolved by restoring Director
to **View** (not Curate) rather than reversing the lockdown or moving the
questions.

### Sales/Purchase provisioning

Created two real accounts, `Sales/Purchase` group only (not `Director`,
not `Other`):

| Account | User ID |
|---|---|
| Sales/Purchase #1 | 6 |
| Sales/Purchase #2 | 7 |

Confirmed via `/api/user/6` and `/api/user/7` (ground truth, not the admin
UI label): both have `user_group_memberships` exactly `[{"id":1},{"id":7}]`
— `1` is `All Users` (automatic), `7` is `Sales/Purchase`. No `Director`,
no `Other`.

**Second batch (2026-08-24):** seven more real accounts added, same group
rule (`Sales/Purchase` only):

| Account | User ID |
|---|---|
| Sales/Purchase #3 | 20 |
| Sales/Purchase #4 | 21 |
| Sales/Purchase #5 | 22 |
| Sales/Purchase #6 | 23 |
| Sales/Purchase #7 | 24 |
| Sales/Purchase #8 | 25 |
| Sales/Purchase #9 | 26 |

Confirmed via `/api/user/20` through `/api/user/26`: all seven have
`user_group_memberships` exactly `[{"id":1},{"id":7}]`, matching #1/#2
above — no `Director`, no `Other`. Live-verified in the admin UI too
(Admin > People, screenshot), all seven tagged `Sales/Purchase`.

An eighth candidate account (originally requested for the `Other` group)
was **deliberately not created** at the same time as the batch above —
`Other` is a zero-access placeholder per standing project rules, and the
requester's actual intended access (`Sales/Purchase` vs. genuinely no
access) was unconfirmed at request time. Arslan confirmed shortly after
(same day) that this account belongs in `Sales/Purchase`:

| Account | User ID |
|---|---|
| Sales/Purchase #10 | 27 |

Confirmed via `/api/user/27`: `user_group_memberships` exactly
`[{"id":1},{"id":7}]`, same as every other account in this table — no
`Director`, no `Other`.

**Invite flow:** same as every prior account on this instance — no SMTP
configured, so no invite email sent. Metabase generated one-time temporary
passwords, shown once in the admin UI. Per this task's explicit
instruction, these were **not** pasted into chat or committed anywhere —
sent to Arslan as a standalone file (to relay directly and then delete),
not recorded in this doc.

**Second batch (2026-08-24) password handling — deliberately different:**
Metabase's "Create user" dialog unavoidably auto-generates and displays a
temporary password immediately on creation (no way to suppress it via the
stock admin UI); each was immediately superseded via the admin "Reset
password" action (not "Get reset link") before being used, so the
creation-time value never mattered. Per Arslan's explicit instruction for
this batch, the final reset-password values **were** relayed directly in
chat as a table (not the prior batch's standalone-file method) — still
never written to this doc, a log file, or committed anywhere.

**Live verification** (throwaway test account `Test SalesPurchaseVerify`,
`Sales/Purchase` group, deactivated after):
- Sidebar **Collections** shows only `2. Satış/Satın Alma` — no
  `1. Yöneticiler`, no `3. Diğer`, no `Boss Dashboard`.
- Sidebar **Data** section shows only `Models`/`Metrics` — no `Databases`,
  consistent with the existing `Sales/Purchase` Create Queries = No setting
  from the first 2026-08-13 pass.
- **Default landing view is not dashboard 3** — unlike `Director`,
  `Sales/Purchase` has no access to the collection dashboard 3 lives in
  (`1. Yöneticiler`), so the instance-wide homepage setting can't apply to
  them. Confirmed live: they land on the generic Metabase home screen
  ("Hey there, [name]"), not an error page and not a folder listing — a
  clean fallback, not a repeat of the original homepage bug. Worth knowing
  going in, not something to fix.

## Boss Dashboard lockdown regression fix (2026-08-13, third pass)

Corrected the regression from the "Boss Dashboard lockdown" section above,
same day. Goal: keep the collection locked down (no edit access, not
publicly discoverable) while restoring Director's ability to actually see
its contents, so the dashboard cards render again — without retiring the
collection or moving its 13 questions, both explicitly out of scope for
this pass.

### 1. Director access restored to View (not Curate)

Set `Director`'s permission on the collection back to **View** — not
**Curate**, and not the original problem (**No access**). Confirmed via
`/api/collection/graph`: group `5` → `{"6":"read","9":"read","10":"read",
"11":"read"}`. `Sales/Purchase` and `Other` untouched — confirmed via the
same API call: group `6` still only has `{"11":"read"}`, group `7` still
only has `{"10":"read"}`, neither has a `"6"` key.

### 2. Renamed to `_Hesaplama Kaynağı`

Collection ID 6 renamed from `Boss Dashboard` → `_Hesaplama Kaynağı`
("calculation source" in Turkish — a leading underscore plus a generic
technical name, nothing implying importance or seniority). URL slug is
now `/collection/6-hesaplama-kaynagi`.

### 3. Nested under `1. Yöneticiler`

Moved the collection so it's a sub-collection of `1. Yöneticiler` (ID 9)
instead of a top-level sidebar item. Confirmed via `/api/collection/6`:
`location: "/9/"`. Confirmed visually: root collection's sidebar tree and
main listing show only three top-level items now (`1. Yöneticiler`,
`2. Satış/Satın Alma`, `3. Diğer`) — `_Hesaplama Kaynağı` only appears
when `1. Yöneticiler`'s new expand arrow is clicked.

### Verification (live, throwaway test account `Test FixVerify`, Director group, deactivated after)

- **Dashboard cards:** `Cansun Satış Genel Bakış` now renders all 8 cards
  correctly — Bu Ay Ciro, Bu Yıl Ciro, Toplam Satış Adedi, Ortalama Fatura
  Değeri, Yıllık Karşılaştırma, 2026 Satis Ay Bazinda, İhracat/Yurtiçi
  Dağılımı, Firma Karşılaştırması. Zero *"Sorry, you don't have permission
  to see this card"* placeholders. Screenshot sent to Arslan.
- **Sidebar nesting for a real Director account:** confirmed
  `_Hesaplama Kaynağı` appears nested under `1. Yöneticiler`'s expand
  arrow, not as a separate top-level row — same shape as the admin view.
- **Edit/move/delete blocked:** opened a question inside the collection
  (`Bölgesel Satış`) — Metabase itself displays an explicit **"View-only"**
  badge with a lock icon next to the title. The collection's per-item
  "Actions" menu offers only "Bookmark" (no Move/Duplicate/Archive); the
  question's own "Move, trash, and more…" menu offers only "Add to
  dashboard" and "Create an alert" (no Move/Edit/Trash). Screenshot sent
  to Arslan.
- **Sales/Purchase and Other unaffected:** confirmed via
  `/api/collection/graph` above — neither group gained any access to this
  collection.

## Adding a real report to `Other` later (future phase, not done here)

Per Arslan's instruction, per-department/per-report mapping for `Other`
is still deferred — `Diğer` stays empty for now. `Sales/Purchase` now has
its first two real members (see above) but no reports of their own have
been assigned beyond the existing `Müşteri Son Fiyat Sorgusu` question
that already lived in `Satış/Satın Alma`. When the `Other` phase starts,
the collection and group grants already exist; the remaining work is just
adding real people and populating `Diğer` with their actual reports. Data
access stays view-only for everyone non-admin — deliberate standing
policy, not a placeholder to revisit.

## `Tedarikçi Son Alış Fiyatı` added to `2. Satış/Satın Alma` (2026-08-14)

New native SQL question (card ID 57) against the new `reporting.supplier_last_purchase` table, added to the existing `2. Satış/Satın Alma` collection (ID 10) alongside `Müşteri Son Fiyat Sorgusu` (card 56) — no new collection created, matching Arslan's instruction. Two Field Filter search widgets, same pattern as card 56: `tedarikci_kodu` → `HesapKodu` (Field Filter, "String contains" widget, label "Tedarikçi Kodu") and `tedarikci_adi` → `HesapAciklamasi` (Field Filter, "String contains" widget, label "Tedarikçi"), both "A single value."

**No group permission changes were made** — this addition relies entirely on the existing `2. Satış/Satın Alma` grants from the 2026-08-13 redesign (`Director`: read, `Sales/Purchase`: read, `Other`: no access), which were already correct for this use case. Confirmed via `/api/collection/graph` before adding anything: group `5` (Director) and group `7` (Sales/Purchase) both had `"10":"read"`; group `6` (Other) had no `"10"` key. Nothing needed to change.

No cost-column exposure check was needed — `supplier_last_purchase` has no `FabrikaFiyati`/`FabrikaTutarUsd` equivalent column at all (see `docs/tables.md`).

### Verification (live, two throwaway test accounts, both deactivated after)

Real `Sales/Purchase` members now exist (`Sales/Purchase #1`, `Sales/Purchase #2` — see "Sales/Purchase provisioning" above), but their credentials were never recorded, same policy as every account in this doc. Rather than log into the shared admin browser session as a different user (risks server-side session invalidation — hit this exact problem during the 2026-08-13 verification pass above), verification used the Metabase API directly: created a throwaway `Test SalesPurchaseVerify` (Sales/Purchase group) and a throwaway `Test OtherVerify` (Other group), authenticated each via `POST /api/session` to get an independent session token, and queried with that token — the admin browser session was never touched and stayed logged in throughout.

- **Sales/Purchase test account:** `GET /api/card/57` → **200**, full card definition returned, `collection_id: 10`. `GET /api/collection/10/items` → both `Müşteri Son Fiyat Sorgusu` and `Tedarikçi Son Alış Fiyatı` listed. `GET /api/collection/9/items` (`1. Yöneticiler`, no grant for this group) → **403**, correctly denied.
- **Other test account:** `GET /api/card/57` → **403**, denied. `GET /api/collection/10/items` → **403**, denied. `GET /api/collection/11/items` (`3. Diğer`, this group's own collection) → **200**, empty list — matches "Diğer stays empty for now" above.
- Both throwaway accounts deactivated immediately after (`Admin > People > Deactivate user`) — same cleanup discipline as every other test account in this doc.

## `Firmamızın Senet-Çek Durumu` split into tabs, `Takip` monthly calendar added (2026-08-19)

**Superseded 2026-08-20 — see "`Takip` rebuilt with a date-driven range and moved to `Cansun Satış Genel Bakış`" below.** The fixed 12-month `Takip` tab described in this section no longer exists: it was deleted (both the tab and its 12 cards) and rebuilt elsewhere. Kept here for history only.

Dashboard 4 (in `1. Yöneticiler`, card IDs 58/59 unchanged) now has two tabs: **`Genel Bakış`** (tab ID 7, the original `Senet-Çek Detay Tablosu` + `Vade Tarihine Göre Senet-Çek Yoğunluğu` cards, untouched) and **`Takip`** (tab ID 8, new), each stacked full-width table cards, no dashboard filter widget. Cards were placed via `PUT /api/dashboard/4/cards` — note this endpoint deletes any `dashboard_tab`s not included in the same request body, so `tabs` must always be sent alongside `cards` on every call, not just the one that creates them, or existing tabs get dropped and the write 500s on the now-missing `dashboard_tab_id` foreign key.

`Takip` holds 12 new native SQL questions (cards 60–71, `Ocak Takip` … `Aralık Takip`, one per calendar month), each a `WITH RECURSIVE` date series LEFT JOINed to `reporting.cheque_bond_maturity` on `VadeTarihi`, year taken from `YEAR(CURDATE())` — no hardcoded year, rolls forward automatically. `WEEKDAY(...)+1` numbering (Mon=1…Sun=7) verified against a real calendar before building all 12 (2026-08-22/23/24 → Sat=6/Sun=7/Mon=1, matching both Python's `date.weekday()` and MySQL's own `DAYNAME()`), so `TATİL` correctly flags Saturday/Sunday only — no holiday-name data, out of scope. Same collection (`1. Yöneticiler`, ID 9) as the rest of this dashboard's cards — no permission changes needed.

**Verified live:** `Genel Bakış` still returns 222 rows on card 58, unchanged. Row counts on 3 spot-checked monthly cards matched days-in-month exactly (Ocak 31, Şubat 28, Ağustos 31), and non-zero `ToplamTutar` values cross-checked against a direct `SUM(TutarYerel) WHERE VadeTarihi = ...` on `cheque_bond_maturity` matched exactly (e.g. 2026-02-25 → 1,800,000; 2026-08-17/18/20 → 300,000/300,000/550,000). Director/Sales-Purchase access re-verified with two fresh throwaway accounts (`Test TabsVerifyDirector`, `Test TabsVerifySalesPurchase`, both deactivated after) — this time authenticated in a genuinely separate Playwright browser context (`browser.newContext()`) rather than the admin's own tab, after the 2026-08-13 and 2026-08-19 incidents where logging a test account into the same tab silently overwrote the admin session cookie. Director: `200` on the dashboard, both tabs listed, `200` on card 58 and card 67. Sales/Purchase: `403` on all three — admin session in the original tab confirmed untouched throughout (`/api/user/current` before and after).

## `Takip` rebuilt with a date-driven range and moved to `Cansun Satış Genel Bakış` (2026-08-20)

The fixed Ocak–Aralık `Takip` tab from 2026-08-19 assumed a full calendar year and didn't roll across a year boundary. Replaced with a range driven by the actual data: `SELECT MAX(VadeTarihi) FROM cheque_bond_maturity` (currently `2027-02-04`) through the current month, inclusive — currently **7 months**, Ağustos 2026 through Şubat 2027. The old tab (dashboard 4, tab ID 8) and its 12 cards (60–71) were deleted entirely, not just hidden; dashboard 4 (`Firmamızın Senet-Çek Durumu`) is back to a single `Genel Bakış` tab with only the original 2 cards.

New cards (72–78, `Ağustos 2026 Takip` … `Şubat 2027 Takip` — month **and year** in the title now, since the range can cross a year boundary) live in the same collection (`1. Yöneticiler`, ID 9) as before, same query shape (`WITH RECURSIVE` date series LEFT JOINed to `cheque_bond_maturity` on `VadeTarihi`), just parameterized by explicit `(year, month)` pairs instead of `YEAR(CURDATE())`.

**Conditional formatting confirmed to exist in v0.63.2.7** (checked live before assuming, per Arslan's instruction not to fake it): each question's visualization settings has a `Conditional Formatting` tab → `Formatting style: Single color` or `Color range`. Applied `Color range` to `ToplamTutar` on all 7 cards via `PUT /api/card/:id` (`visualization_settings.table.column_formatting`) — Metabase's default 3-color diverging scale (red at the column's own min, transparent at the midpoint, green at the max), auto-scaled per card (`min_type`/`max_type: null`). Not a literal "darker = higher" single-hue ramp — Arslan can swap the 3 colors in the UI in one click if a different look is wanted; not changed further here since the default already satisfies "higher totals shade more intense."

Added one more card (79, `Aylara Göre Toplam Tutar`, bar chart) at the top of `Takip`, X-axis = month, Y-axis = `SUM(TutarYerel)`, same date-driven range logic as the monthly tables via a second `WITH RECURSIVE` (this one steps by month instead of by day) — no hardcoded months here either.

**`Takip` moved from `Firmamızın Senet-Çek Durumu` to `Cansun Satış Genel Bakış`** (dashboard 3), as a new tab (ID 9) positioned right after the existing `Genel Bakış` tab (position 1, pushing `Ürünler & Müşteriler` and `Coğrafya & Kanal` to positions 2/3) — dashboard 3's tab structure was checked live first ( `Genel Bakış` / `Ürünler & Müşteriler` / `Coğrafya & Kanal`, 13 cards across all three, all with `Tarih`/`Firma` filter parameter mappings) rather than assumed; all 13 existing cards' exact `row`/`col`/`size_x`/`size_y`/`parameter_mappings` were preserved unchanged in the same `PUT /api/dashboard/3/cards` call that added the 8 new `Takip` cards (bar chart + 7 monthly tables, stacked top to bottom). Same gotcha as 2026-08-19 applies here too — `tabs` had to include all 4 tabs (3 existing + new), not just the new one, or the existing tabs get dropped.

**Verified live:** `Firmamızın Senet-Çek Durumu` confirmed back to single `Genel Bakış` tab, 2 cards. `Cansun Satış Genel Bakış` confirmed to have exactly 4 tabs in the right order, `Takip` holding exactly 8 cards (bar chart + the 7 correct months, no leftover Ocak–Temmuz or Mart–Temmuz cards from the old fixed-year build) — screenshotted and visually confirmed the bar chart shape and color-formatted table render correctly. Director/Sales-Purchase access re-verified on dashboard 3 with two more fresh throwaway accounts (`Test TakipMoveVerifyDirector`, `Test TakipMoveVerifySalesPurchase`, both deactivated after), same isolated-`browser.newContext()` method as 2026-08-19: Director `200` on the dashboard (all 4 tabs listed) and both a monthly card and the bar chart card; Sales/Purchase `403` on all three; admin session confirmed untouched via `/api/user/current` before and after.

Arslan renamed the tab from `Takip` to `Çek-Senet Takip` shortly after this — noted here since later sections reference it by that name.

### `Vade Tarihi` column rename + narrower date format — standard template updated (2026-08-20, same day)

**Column alias:** `ds.calendar_date AS Tarih` → `` ds.calendar_date AS `Vade Tarihi` `` (backticked, since the alias now has a space) in all 7 existing monthly cards (72–78) and in the base query pattern going forward — accurate naming, since the column is `VadeTarihi`, not an arbitrary date. Applied via `PUT /api/card/:id` with the modified `dataset_query.native.query` string (fetched each card's exact current SQL first and did a targeted string replace, rather than reconstructing from scratch, to avoid drift). Row counts re-verified unchanged after the rename (31/30/31/30/31/31/28 — days in each of the 7 months) to confirm nothing else broke.

**Column width:** checked live which levers actually exist before picking one, per Arslan's instruction —
- **Manual column-width drag: does not exist.** Empirically tested by dragging the column-header border in the table viz; it **reorders columns** instead of resizing them (confirmed via the resulting `visualization_settings.table.columns` order changing on an unsaved ad-hoc exploration — the saved card itself was unaffected since it was never explicitly saved).
- **Date display format: exists**, via each column's format gear → `Date style`. Available options: `January 31, 2018`, `31 January, 2018`, `Wednesday, January 31, 2018`, `1/31/2018`, `31/1/2018`, `2018/1/31`. **No literal dot-separated `DD.MM.YYYY` option exists** — closest to Arslan's Excel-style day/month/year ordering is `31/1/2018` (slash-separated, unpadded), stored as `date_style: "D/M/YYYY"` under `visualization_settings.column_settings["[\"name\",\"Vade Tarihi\"]"]`. Applied to all 7 cards.

**Verified live:** all 7 cards' `Vade Tarihi` header confirmed (via `/api/card/:id/query` column names and a dashboard screenshot) — no card still shows `Tarih`. Dates render as e.g. `1/8/2026` instead of `August 1, 2026`. Checked the actual dashboard rendering (not just the standalone question view, since that's where the original width complaint was): for all 7 monthly table `dashcard`s on `Çek-Senet Takip`, the table grid's `scrollWidth` exactly equals its `clientWidth` (1031px = 1031px) — **no horizontal scrollbar is needed or present**; the narrower format fully resolved it, not just reduced it. `Genel Bakış` (8 cards), `Ürünler & Müşteriler` (3), and `Coğrafya & Kanal` (2) card counts confirmed unchanged — this task only edited cards 72–78's own `dataset_query`/`visualization_settings`, never touched `/api/dashboard/3/cards`.

**Standard template going forward:** any future rebuild of these monthly cards (e.g. when the range needs regenerating) should use `` ds.calendar_date AS `Vade Tarihi` `` from the start and apply both `table.column_formatting` (color range on `ToplamTutar`) and `column_settings` (`date_style: "D/M/YYYY"` on `Vade Tarihi`) at creation time, in the same collection (`1. Yöneticiler`) at full dashboard width (`size_x: 24`) — no manual fixup pass needed afterward.
