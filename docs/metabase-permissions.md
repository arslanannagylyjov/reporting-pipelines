# Metabase Permission Groups — Director and User

Added 2026-08-11 on the Metabase instance at `10.20.52.43:3000`. Two groups
now exist beyond Metabase's built-in `Administrators` and `All Users`. Group
scaffolding was done with no real people added, per Arslan's instruction; the
first real account (`Director`) was provisioned 2026-08-12 — see "Provisioned
accounts" below. `User` still has no real members; Arslan will supply the
rest of the real user list once department mapping is ready.

## Groups

| Group | Metabase group ID | Intended scope |
|---|---|---|
| **Director** | 5 | Full data access. Can see everything a boss should see. |
| **User** | 6 | Minimal by default — sees effectively nothing until specific reports are granted to it later, item by item. That per-report mapping is a **future phase**, not done here. |

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

**Symptom:** Engin (first real `Director`-group user) logged in and landed
on the `Boss Dashboard` *collection* — a folder listing — instead of the
`Cansun Satış Genel Bakış` dashboard itself.

**Root cause:** `Admin > Settings > General > Homepage` was set to "Default
Metabase home." That default doesn't point at any specific dashboard — it
falls back to the one collection a user can actually see. Since `Director`
only has **View** on the `Boss Dashboard` collection (not on the root `Our
analytics` collection), that fallback landed Engin on the collection folder,
not the dashboard inside it. This was never a wrong-ID problem: confirmed
directly via `/api/dashboard/3` that dashboard ID 3 genuinely is `Cansun
Satış Genel Bakış` (tabs `Genel Bakış` / `Ürünler & Müşteriler` / `Coğrafya &
Kanal`, parameters `Tarih`/`Firma`), correctly living inside collection ID 6.

**Fix:** Set `Admin > Settings > General > Homepage` to "Dashboard" →
`Cansun Satış Genel Bakış`. Verified persisted via `/api/setting`:
`custom-homepage: true`, `custom-homepage-dashboard: 3`.

**Verification:** From an Administrator session, navigating to
`http://10.20.52.43:3000/` now redirects to `/dashboard/3?...`, showing the
real dashboard with all three tabs and both filters.

**Confirmed (2026-08-12):** Engin refreshed/re-navigated and confirmed he now
lands directly on the `Cansun Satış Genel Bakış` dashboard, not the
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
Metabase-facing `metabase_ro` MySQL account has no grant on it, by design
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

| Name | Email | Groups | User ID | Provisioned | Status |
|---|---|---|---|---|---|
| Engin Karacan | `enginkaracan@wenderparts.com` | Director (+ All Users, automatic) | 2 | 2026-08-12 | Logged in, changed his temporary password — first live login, see below |

**First real (non-admin) account in either group.** Confirmed via
`/api/user` (ground truth, not the admin UI label) that `group_ids` is
exactly `[1, 5]` — `1` is `All Users` (automatic, no opt-out), `5` is
`Director`. No `User` group, no `Administrators`.

**Invite flow:** this instance has no SMTP configured (`Admin > Settings >
Email` shows "Configure," not "Edit"; confirmed via `/api/setting` that
`email-smtp-host` is `null`). Metabase therefore could not send an email
invite and instead generated a one-time temporary password on account
creation, shown once in the admin UI, with the explicit message: *"We
couldn't send them an email invitation, so make sure to tell them to log in
using enginkaracan@wenderparts.com and this password we've generated for
them."* Arslan needs to relay the temporary password to Engin directly
through some other channel (it is not recorded here). Engin will be
prompted to set his own password on first login.

**Why this account matters beyond onboarding:** it's the first live,
real-world test of the permission work above. Engin's login confirmed
`Director` has full working access to the dashboard and its underlying data
(not just correct on paper), and his post-fix refresh confirmed he lands
directly on `Cansun Satış Genel Bakış` rather than the collection folder or
the generic Metabase home — see "Homepage fix" above.

## Adding a real report to `User` later (future phase, not done here)

Per Arslan's instruction, per-department/per-report mapping is explicitly
deferred. When that phase starts, the pattern will be: grant `User` (or a
new department-specific group, if that's the direction chosen) **View**
collection access to the specific collection holding that report, and
**"Query builder and native" or "Query builder only"** data access to
exactly the tables that report needs — never blanket database access, and
never anything touching the `Boss Dashboard` collection or its two cost
columns.
