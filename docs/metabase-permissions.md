# Metabase Permission Groups — Director and User

Added 2026-08-11 on the Metabase instance at `10.20.52.43:3000`. Two groups
now exist beyond Metabase's built-in `Administrators` and `All Users`. No
real people have been added to either yet — group scaffolding only, per
Arslan's instruction; he'll supply the real user list once department
mapping is ready.

## Groups

| Group | Metabase group ID | Intended scope |
|---|---|---|
| **Director** | 5 | Full data access. Can see everything a boss should see. |
| **User** | 6 | Minimal by default — sees effectively nothing until specific reports are granted to it later, item by item. That per-report mapping is a **future phase**, not done here. |

## Why this exists

`Boss Dashboard` (dashboard ID 3, in the "Boss Dashboard" collection) and its
12 questions are built on `sales_snapshot` and include two cost columns —
`FabrikaFiyati` and `FabrikaTutarUsd` — that must never be visible to
non-boss users. `Director` is the group that's allowed to see them; `User`
is the template for everyone else until they're individually granted access
to specific non-cost reports.

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

## Cost-column exposure check (2026-08-11)

Checked whether `FabrikaFiyati`/`FabrikaTutarUsd` are reachable from
anywhere in the instance other than the now-locked `Boss Dashboard`
collection:

- Enumerated every collection in the instance: `Our analytics` (root)
  contains only the `Boss Dashboard` folder and question 56 (Müşteri Son
  Fiyat Sorgusu — no cost columns, intentionally left open to All Users per
  Arslan's instruction); `Boss Dashboard` contains exactly the 12 questions
  + 1 dashboard described in the task; Arslan's personal collection is
  empty.
- Ran a Metabase search for "Fabrika" with "search the contents of native
  queries" and "search items in trash" both enabled — zero results anywhere
  in the instance, including trash.
- Conclusion: the two cost columns have no path to any user outside
  `Director` (or an Administrator). `User`'s "No access"/"No" settings above
  are the only thing standing between it and the data, and there's no
  second dashboard, question, or archived item that bypasses that.

## Adding a real report to `User` later (future phase, not done here)

Per Arslan's instruction, per-department/per-report mapping is explicitly
deferred. When that phase starts, the pattern will be: grant `User` (or a
new department-specific group, if that's the direction chosen) **View**
collection access to the specific collection holding that report, and
**"Query builder and native" or "Query builder only"** data access to
exactly the tables that report needs — never blanket database access, and
never anything touching the `Boss Dashboard` collection or its two cost
columns.
