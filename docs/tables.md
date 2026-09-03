# Tables — `reporting` database (reporting-db, athena)

## `sales_snapshot`

Durable table Metabase dashboards read from. 675,810 rows as of 2026-08-10.

| Column | Type | Null | Key |
|---|---|---|---|
| ID | bigint | NO | PRI |
| Firma | varchar(50) | NO | PRI |
| EvrakNo | varchar(50) | YES | |
| FaturaNo | varchar(50) | YES | |
| StokKodu | varchar(50) | YES | MUL |
| KompleAdi | varchar(500) | YES | |
| Marka | varchar(100) | YES | |
| Model | varchar(200) | YES | |
| AnaGrup | varchar(100) | YES | |
| AltGrup | varchar(100) | YES | |
| Kategori | varchar(100) | YES | |
| UrunAd | varchar(200) | YES | |
| Tedarikci | varchar(100) | YES | |
| SatisAdet | decimal(18,4) | YES | |
| HesapKodu | varchar(50) | YES | |
| CariAdi | varchar(300) | YES | |
| Plasiyer | varchar(100) | YES | |
| Il | varchar(100) | YES | |
| Ulke | varchar(100) | YES | |
| Bolge | varchar(100) | YES | |
| SatisFiyatTL | decimal(18,4) | YES | |
| KDVDahil | varchar(10) | YES | |
| TutarTL | decimal(18,2) | YES | |
| DovizKuru | decimal(18,6) | YES | |
| DovizFiyat | decimal(18,4) | YES | |
| DovizTutar | decimal(18,2) | YES | |
| DovizKodu | varchar(10) | YES | |
| FabrikaFiyati | decimal(18,4) | YES | |
| FabrikaTutarUsd | decimal(18,2) | YES | |
| Tarih | date | YES | MUL |
| SatisTipi | varchar(20) | YES | |

Indexes beyond the primary key: `StokKodu`, `Tarih` (both `MUL` — non-unique, used for lookups/filtering).

**Prune-on-sync added (2026-08-25):** `refresh_sales_snapshot.py`'s stage-then-`REPLACE INTO` merge only ever adds/updates rows whose `(Firma, ID)` key is present in the current 4-month ERP fetch — it never removes a row on its own. When an ERP invoice is corrected or reissued under a new `ID` (the old one dropping out of the source view entirely), the old row used to linger in `sales_snapshot` forever, silently double- (or triple-) counting revenue for that document. The script now adds an explicit `DELETE FROM sales_snapshot WHERE Tarih >= <cutoff> AND NOT EXISTS (... matching key in sales_staging ...)` step immediately after the merge, using the exact same cutoff value computed once for the fetch — not a second, independently-evaluated `CURDATE()` call that could drift between the ERP server and `reporting-db`. Rows older than the rolling window are structurally untouched by this (both the merge and the prune are scoped to `Tarih >= cutoff`), confirmed live: 618,498 pre-window rows unchanged, 68,007 in-window rows after a clean run with zero orphans. See `docs/monitoring.md` for the logging behavior and `session-notes.md` for the full incident writeup.

**One-time cleanup (2026-08-25), before the fix was deployed:** a live audit (triggered by a reported total mismatch for HesapKodu `S 35831`, August 2026) found **90 stale rows** across **3 distinct HesapKodu** that had accumulated this way, totaling **2,490,380.13 TL** of phantom revenue:

| Firma | HesapKodu | CariAdi | stale rows | sum TutarTL |
|---|---|---|---|---|
| Cansun | 34-947 | MEHMET ERDEM | 1 | 2,759.42 |
| Almer | S 35831 | MK KAMMAZ OTO YEDEK PARCA MAKINA SAN. VE TIC.LTD.STI. | 24 | 1,881,466.15 |
| Cansun | Y 01823 | MIKAIL MOLDOVA | 65 | 606,154.56 |

Confirmed via a dry-run `SELECT` first (shown to and approved by Arslan before any deletion), then deleted with the identical `WHERE` clause — exactly 90 rows removed, matching the dry run precisely. Post-cleanup, `S 35831`/August matched the live ERP view exactly (12 Almer rows / 1,161,510.03 TL, 4 Cansun rows / 63,449.23 TL) before the permanent fix was even deployed.

## `sales_staging`

Identical schema to `sales_snapshot`. Transient landing table — each cron run stages a batch of ERP rows here, merges them into `sales_snapshot`, then clears the table. 0 rows is the expected resting state; non-zero rows outside a run window would indicate a failed/interrupted sync.

## `customer_last_price`

Each customer's last transaction price per product, sourced from the pre-built ERP view `aa_customer_last_price` (one row per `Firma`/`HesapKodu`/`StokKodu`, already deduplicated on the ERP side — no ranking logic in the sync script). Full-replace table, not incremental: every run truncates and reloads all rows. 65,143 rows as of the 2026-08-10 verification run (matches the ERP view's row count exactly).

| Column | Type | Null | Key |
|---|---|---|---|
| Firma | varchar(20) | NO | PRI |
| HesapKodu | varchar(50) | NO | PRI |
| StokKodu | varchar(50) | NO | PRI |
| HesapAciklamasi | varchar(255) | YES | MUL |
| StokAciklamasi | varchar(500) | YES | |
| DvzBirimFiyat | decimal(18,4) | YES | |
| DovizKodu | varchar(10) | YES | |
| DovizKuru | decimal(18,6) | YES | |
| BirimFiyat | decimal(18,4) | YES | |
| BelgeTarihi | date | YES | |
| BelgeNo | varchar(50) | YES | |
| FaturaD_ID | bigint | YES | |
| synced_at | timestamp | YES | DEFAULT CURRENT_TIMESTAMP |

`FaturaD_ID` is `int` on the ERP view side — widened to `bigint` here, which is a safe, intentional mismatch (not a bug). PK columns are non-nullable in the source too (verified 0 NULLs across all three columns at time of sync), so no insert-failure risk there.

Indexes beyond the primary key: `HesapKodu` (`MUL`).

**Data quality note (`DovizKodu`, observed 2026-08-11):** values are inconsistent for the same currency — e.g. some rows have `"EUR"`, at least one has `"EURO"` instead. This is a source data issue on the ERP side (the view/underlying ERP tables), not a bug in the sync pipeline — the sync copies `DovizKodu` verbatim and does no normalization by design. Not fixed here. Anyone building a report that groups, filters, or joins on currency code should normalize/canonicalize `DovizKodu` values (e.g. map `"EURO"` → `"EUR"`) at the report/query layer, or confirm the full set of variant spellings first — don't assume `EUR` is the only spelling in use.

## `supplier_last_purchase`

Each supplier's last purchase per product, sourced from the pre-built ERP view `aa_supplier_last_purchase` (despite the `cansun` schema prefix, this view already UNIONs all three companies — Cansun/Karacan/Almer — distinguished by the `Firma` column; queried once, not once per company). Full-replace table, not incremental — but unlike `customer_last_price`, replace is implemented as upsert-then-prune rather than `TRUNCATE`, because `reporting_writer`'s grant on this table is `SELECT, INSERT, UPDATE, DELETE` (no `DROP`, which `TRUNCATE` requires in MySQL). ~29,300 rows as of the 2026-08-14 verification run (29,351 exactly, matching the ERP view's row count).

| Column | Type | Null | Key |
|---|---|---|---|
| Firma | varchar(20) | NO | PRI |
| StokKodu | varchar(50) | NO | PRI |
| StokAciklamasi | varchar(255) | YES | |
| HesapKodu | varchar(50) | YES | |
| HesapAciklamasi | varchar(255) | YES | |
| BelgeNo | varchar(50) | YES | |
| GirenMiktar | decimal(18,4) | YES | |
| Fiyat | decimal(18,2) | YES | |
| Tarih | datetime | YES | |
| IslemTipi | varchar(50) | YES | |
| Seri | varchar(10) | YES | |
| DovizKuruEvrak | decimal(18,4) | YES | |
| SyncedAt | datetime | NO | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

Composite primary key on `(Firma, StokKodu)` — one row per supplier product per company, already deduplicated on the ERP side.

**`SyncedAt` timezone note:** this column is written from the `reporting-db` MySQL container's own `NOW()`, which is UTC (the container does not inherit the host's Europe/Istanbul timezone — see `docs/server-architecture.md`). The sync script captures `run_start` from the same `NOW()` call rather than the athena host's local clock specifically to avoid a timezone mismatch between the value written to `SyncedAt` and the value used to identify stale rows for pruning — mixing the two caused every row to be pruned on the first real run (see `docs/monitoring.md`). Don't read `SyncedAt` as Istanbul time.

No cost-column equivalent (`FabrikaFiyati`/`FabrikaTutarUsd`) exists on this table — no exposure decision was needed when adding the Metabase question.

## `cheque_bond_maturity`

Cheque/bond maturity schedule for Cansun and Almer (`Firma` column), sourced from the pre-built ERP view `aa_rapor_cek_borc_durum` — a UNION of Cansun's and almer23's own `cek_d`/`cari`/`cek_h` tables, filtered to `EvrakNo LIKE 'BCC%' AND cekdurumu = 13` on the ERP side (not reimplemented in the sync script). Full-replace-by-key table via chunked `REPLACE INTO` (same reason as `supplier_last_purchase` — `reporting_writer` has `SELECT, INSERT, UPDATE, DELETE` but no `DROP`, so `TRUNCATE` isn't available), no staging table. Karacan is intentionally excluded (not present in the source view's UNION). 222 rows as of the 2026-08-19 verification run, matching the ERP view's row count exactly.

| Column | Type | Null | Key |
|---|---|---|---|
| Firma | varchar(20) | NO | PRI |
| EvrakNo | varchar(50) | NO | |
| BelgeNo | varchar(50) | NO | PRI |
| Tarih | datetime | YES | |
| VadeTarihi | date | YES | |
| BelgeBankaKodu | varchar(20) | YES | |
| BelgeBankaAdi | varchar(100) | YES | |
| HesapKodu | varchar(20) | NO | |
| HesapAciklamasi | varchar(255) | YES | |
| Tutar | decimal(15,2) | YES | |
| TutarYerel | decimal(15,2) | YES | |

**Composite key note (2026-08-19):** the table's primary key was originally going to be `(Firma, EvrakNo)`, matching this table's original task spec — but that pair turned out **not unique** in the source view: one `EvrakNo` (a single cheque/bond document) can carry several installments, each with its own `BelgeNo`, `VadeTarihi`, and `Tutar`. Verified live before writing any sync code: 222 source rows collapsed to only 70 distinct `(Firma, EvrakNo)` pairs, while `(Firma, BelgeNo)` is fully unique (222/222). Arslan changed the primary key to `(Firma, BelgeNo)` directly on `reporting-db` before the sync script was built, so every installment gets its own row — `REPLACE INTO` on the original key would have silently collapsed each cheque's installments down to one row per sync, discarding real maturity-date data every night. `EvrakNo` remains a plain (non-unique) column, since it's still useful for grouping a cheque's installments together.

**Stale-row note:** `REPLACE INTO` only touches keys present in the current run's source rows — a cheque that drops out of the source view (paid, cancelled, or `cekdurumu` changes) is not pruned from this table and will linger until manually cleaned up. Unlike `supplier_last_purchase`, this table has no prune step (not requested — data delivery only, no business logic).

## `vault_status`

Live snapshot of Cansun's three cash vaults ("kasa"), sourced from the pre-built ERP view `aa_vault_status` on Natra. Full-replace via a single unchunked `REPLACE INTO` — no staging table (that pattern is exclusive to `sales_snapshot`) and no chunking (only 3 rows). Refreshed every 30 minutes, 09:00–19:00 daily (widened from hourly on 2026-08-26), by `refresh_vault_status.py`. 3 rows as of the 2026-08-21 verification run, matching the ERP view's row count exactly.

| Column | Type | Null | Key |
|---|---|---|---|
| KasaKodu | varchar(20) | NO | PRI |
| KasaAciklamasi | varchar(150) | YES | |
| para_birimi | varchar(10) | YES | |
| bakiye | double(20,6) | YES | |

Not in `monitored_jobs.yml` / the 07:00 daily digest — see `docs/monitoring.md` for why, and for the failure-only Telegram alerting this job uses instead.

## `vault_movements`

Detailed transaction history behind the three `vault_status` balances, sourced from the pre-built ERP view `aa_vault_movements` on Natra — one row per cash-vault transaction across all three vaults (TL/EUR/USD, distinguished by `KasaKodu`). 1,306 rows as of the 2026-08-21 initial backfill, matching the ERP view's row count exactly; grows daily as new transactions post (no pruning beyond what the hourly/daily syncs themselves do).

| Column | Type | Null | Key |
|---|---|---|---|
| ID | bigint | NO | PRI |
| EvrakNo | varchar(50) | YES | |
| KasaKodu | varchar(20) | NO | MUL |
| KasaAciklamasi | varchar(150) | YES | |
| HesapKodu | varchar(50) | YES | |
| HesapAciklamasi | varchar(255) | YES | |
| IslemTarihi | datetime | YES | MUL |
| Aciklama | varchar(500) | YES | |
| tutar_giren | decimal(18,2) | YES | |
| tutar_cikan | decimal(18,2) | YES | |
| HesapTipi | varchar(50) | YES | |
| DovizKodu | varchar(10) | YES | |
| DovizKuru | decimal(18,6) | YES | |

Two separate sync jobs, not one, matching this table's two different freshness needs:

- **`refresh_vault_movements_hourly.py`** — hourly, 09:00-19:00 (`5 9-19 * * *`), scoped to the last 4 days (widened from 3 on 2026-08-24): `DELETE FROM vault_movements WHERE IslemTarihi >= (today - 4 days)` then re-inserts that window from the ERP. Never touches older rows.
- **`refresh_vault_movements_daily.py`** — once daily at 03:30, full replace of all rows via `DELETE FROM vault_movements` (unconditional) then re-insert. **`DELETE`, not `TRUNCATE`** — same reason as `supplier_last_purchase`/`cheque_bond_maturity` above: `reporting_writer` has `SELECT, INSERT, UPDATE, DELETE` on this table but no `DROP`, and `TRUNCATE` requires `DROP` in MySQL. Confirmed live (`1142 DROP command denied`) before writing the script.

Both are failure-only on Telegram (no success ping, given the hourly cadence) — see `docs/monitoring.md` for the full schedule reasoning, collision checks against `vault_status`/`bump_filter_defaults.py`, and the one-time manual backfill that seeded this table before cron took over.

## `cek_senet_portfoy`

Cansun's outstanding cheque/promissory-note ("çek/senet") portfolio, sourced from the pre-built ERP view `aa_cek_senet_portfoy` on Natra. Full-replace-by-key table via a mandatory pre-flight duplicate check (Python-side `groupby (Firma, CekSiraNo)`, abort + Telegram alert + non-zero exit if any group has `count > 1`) followed by chunked `DELETE FROM` + `INSERT` — **`DELETE`, not `TRUNCATE`**, same reason as `supplier_last_purchase`/`cheque_bond_maturity`/`vault_movements_daily`: `reporting_writer` has no `DROP` grant, confirmed live (`1142 DROP command denied`) before switching. 10 rows as of the 2026-08-23 verification run, matching the ERP view's row count exactly.

| Column | Type | Null | Key |
|---|---|---|---|
| Firma | varchar(10) | NO | PRI |
| Tur | varchar(10) | NO | |
| CekSiraNo | int | NO | PRI |
| EvrakNo | varchar(10) | NO | |
| BelgeNo | int | YES | |
| AsilCiro | varchar(10) | NO | |
| HesapKodu | varchar(20) | NO | |
| HesapAciklamasi | varchar(150) | YES | |
| VadeTarihi | date | YES | |
| TutarYerel | double(20,6) | YES, default 0.000000 | |
| Seri | char(3) | YES | |

Refreshed every 2 hours, 09:00–19:00 daily, by `refresh_cek_senet_portfoy.py`. Telegram alerting is failure-only (no success ping), same convention as `vault_status`/`vault_movements_*` — see `docs/monitoring.md` for the history (it briefly sent a success message too, per the original task spec, until Arslan asked for failure-only on 2026-08-24), plus the pre-flight duplicate-check design and cron collision check against `vault_status`.

Backs three Metabase questions in `_Hesaplama Kaynağı` ("Portföy Çek Toplamı (TL)", "Portföy Senet Toplamı (TL)", "Portföy Çek/Senet Listesi"), placed on dashboard 3's "Kasa Durumu" tab between the vault scalar cards and the `vault_movements` tables.

## `stock_details`

Full stock/product catalog (pricing, supplier, physical dimensions, cost columns), sourced from the pre-built ERP view `cansun.aa_rapor_stok_listesi` on Natra via `metabase_ro`. Full-replace via `TRUNCATE` + chunked `INSERT` — unlike every table added since `cheque_bond_maturity`, `reporting_writer` genuinely has `DROP` on this one (confirmed live via `SHOW GRANTS`, not assumed), so `TRUNCATE` is valid here. `StkoKodu` is the primary key, confirmed unique on both the source view and destination table (40,104/40,104 distinct, live-checked before writing the sync script — no pre-flight duplicate check needed, unlike `cek_senet_portfoy`). 40,104 rows as of the 2026-08-25 verification run, matching the ERP view's row count exactly.

| Column | Type | Null | Key |
|---|---|---|---|
| Sira | varchar(50) | YES | |
| StkoKodu | varchar(30) | NO | PRI |
| KompleAdi | varchar(200) | YES | |
| FullName | varchar(200) | YES | |
| RU | varchar(250) | YES | |
| Birim | varchar(10) | YES | |
| ReklamKodu | varchar(50) | YES | |
| UrunAdi | varchar(50) | YES | |
| DovizKodu | varchar(20) | YES | |
| Cns_Eur | double(20,6) | YES | |
| Cns_Usd | double(20,6) | YES | |
| Trk_Usd | double(20,6) | YES | |
| FabrikaFiyatiUsd | double(20,6) | YES | |
| Barkod | varchar(60) | YES | |
| AnaGrup | varchar(50) | YES | |
| AltGrup | varchar(50) | YES | |
| Kategori | varchar(50) | YES | |
| UreticiKodu | varchar(100) | YES | |
| Arac | varchar(50) | YES | |
| Marka | varchar(50) | YES | |
| AracTipi | varchar(50) | YES | |
| OncekiKod | varchar(100) | YES | |
| TedarikciKodu | varchar(50) | YES | |
| TedarikciAdi | varchar(150) | YES | |
| TedarikciGrup | varchar(14) | NO | |
| StoklanmaTuru | varchar(50) | YES | |
| Agirlik | double(14,6) | YES | |
| Hacim | double(14,6) | YES | |
| En | double(14,2) | YES | |
| Boy | double(14,2) | YES | |
| Yukseklik | double(14,2) | YES | |

**Cost-column note:** `FabrikaFiyatiUsd` (plus `Cns_Usd`/`Trk_Usd`) is a cost column, same category as `sales_snapshot`'s `FabrikaFiyati`/`FabrikaTutarUsd`. The Metabase question built on this table (`01_Genel_StokListesi` — card 103, renamed from `Stok Listesi` on 2026-08-28) exposes it as part of a deliberate full-detail placeholder. As of 2026-08-27 card 103 lives in the `3. İthalat/İhracat` collection and is therefore visible to `Director`, `Manager`, and `Sales/Purchase` (the cascade — see `docs/metabase-permissions.md`); Arslan explicitly accepted this cost-column exposure when it was moved out of the former Director+Manager-only `2. Manager` folder. Which columns to actually show is still a future decision, not made here.

Refreshed nightly at 03:45 by `refresh_stock_details.py`, registered in `monitored_jobs.yml` and covered by the 07:00 daily digest — see `docs/monitoring.md`.

## `orders_in_transit`

Orders placed but not yet received — the "Yolda" (in-transit) report for Cansun and Karacan, sourced from the pre-built ERP view `cansun.aa_rapor_yolda_hepsi` on Natra via `metabase_ro`. Despite the `cansun` schema prefix the view is meant to UNION Cansun + Karacan (distinguished by the `Firma` column); as of the 2026-08-27 initial sync it returns **1,852 rows, all `Firma = 'Cansun Yolda'`** — no Karacan rows are currently in "Yolda" status, not a pipeline bug. **Almer is deferred** — not in the view's UNION yet. When Almer is added, re-verify the `(Firma, ID)` primary key is still collision-free across all three companies before trusting the first sync (the ERP `ID` is only guaranteed unique within one company's document series; the pre-flight duplicate check in the sync script will abort rather than corrupt data if it isn't, but confirm deliberately).

Full-replace-by-key table via chunked `REPLACE INTO` on `(Firma, ID)` — **no staging table** (that pattern is exclusive to `sales_snapshot`), and **`REPLACE INTO`, not `TRUNCATE` + `INSERT`**, because `reporting_writer`'s grant here is `SELECT, INSERT, UPDATE, DELETE` with no `DROP` (`TRUNCATE` requires it), same grant shape as `cek_senet_portfoy` / `supplier_last_purchase`. Column list is identical (name + order) on the source view and the destination table, so the sync selects an explicit column list, not `SELECT *`.

| Column | Type | Null | Key |
|---|---|---|---|
| Firma | varchar(13) | NO | PRI |
| ID | int | NO | PRI |
| EvrakNo | varchar(20) | YES | |
| HesapKodu | varchar(20) | YES | |
| HesapAciklamasi | text | YES | |
| StokKodu | varchar(30) | YES | |
| StokAciklamasi | text | YES | |
| Miktar | double(20,6) | YES | |
| DovizKodu | varchar(20) | YES | |
| DovizKuru | double(14,6) | YES | |
| DovizFiyat | double(20,6) | YES | |
| DovizTutar | double(20,6) | YES | |
| BelgeTarihi | date | YES | |
| TerminTarihi | date | YES | |
| Kargo | varchar(30) | YES | |
| Aciklama1 | varchar(100) | YES | |
| Aciklama2 | varchar(100) | YES | |
| Aciklama3 | varchar(100) | YES | |
| Aciklama4 | varchar(100) | YES | |
| EvrakTutari | double(20,6) | YES | |

Composite primary key on `(Firma, ID)` — one row per order line per company. Verified collision-free live before the first sync: 1,852 source rows / 1,852 distinct `(Firma, ID)` pairs.

**Prune-on-sync (built in from day one, not retrofitted):** `REPLACE INTO` only ever adds or updates keys present in the current ERP fetch — it never removes a row on its own. An order can drop out of "Yolda" status (its `SiparisDurumu` / `Kalan` changes once it's received) and disappear from the source view **without ever reappearing under the same `ID`** — the exact voided/reissued-document staleness pattern that bit `sales_snapshot`. So `refresh_orders_in_transit.py` runs an explicit prune immediately after the upsert: it reads back every `(Firma, ID)` currently in `orders_in_transit`, subtracts the set of keys this run pulled (held in one shared in-memory variable — never re-queried from a second source call that could drift from the fetch), and `DELETE`s the difference in chunks. Confirmed live: a synthetic stale row (`ID = -999`) was pruned on the next run, and a clean re-run of unchanged data pruned 0 and left all 1,852 rows intact.

Refreshed nightly at **03:05** by `refresh_orders_in_transit.py` — see `docs/monitoring.md` for the schedule reasoning, the mandatory pre-flight duplicate check, and the success/failure Telegram alerting. Backs the Metabase question **`03_Ithalat_YoldakiÜrünListesi`** (card 104, renamed from `Yoldaki Ürün Listesi` on 2026-08-28) in the `3. İthalat/İhracat` collection (id 10), which therefore inherits that folder's cascade (`Director`, `Manager`, `Sales/Purchase`, `Satış` — see `docs/metabase-permissions.md`); no group-level override was created. This table carries no cost column (`FabrikaFiyati`-style), so no boss-only exposure decision was needed.

## `warehouse_stock`

Per-product on-hand quantities across Cansun's two physical warehouses (Çatalca and Merkez), each split into a settled bucket and a goods-receiving ("Mal Kabul") bucket. Sourced from the ERP table `cansun.eryaz_zeus_stok_slim` on Natra via `metabase_ro` — itself a nightly full-truncate-and-reload table on the ERP side, so this pipeline mirrors that: no incremental logic, no history kept. **One row per `StokKodu`, no `Firma` dimension** — unlike almost every other table here, warehouse stock isn't a multi-company concept. 17,029 rows as of the 2026-08-28 initial sync (`COUNT(*) = COUNT(DISTINCT Code)` on the source, both 17,029 — PK-safe, no pre-flight duplicate check needed).

Only 5 of the source's 7 columns are pulled. The two `İade` (return-goods) buckets — `QTY_230` (Çatalca İade) and `QTY_530` (Merkez İade) — are **excluded by design**, per Arslan; they're not needed for the intended reporting and adding them later just means widening both the `SELECT` and the destination table.

**Column mapping** (source → destination):

| ERP source (`eryaz_zeus_stok_slim`) | `warehouse_stock` | Meaning |
|---|---|---|
| `Code` | `StokKodu` | product code (PK) |
| `QTY_200` | `Catalca` | Çatalca warehouse, settled stock |
| `QTY_210` | `CatalcaMalKabul` | Çatalca, in goods-receiving |
| `QTY_500` | `Merkez` | Merkez warehouse, settled stock |
| `QTY_510` | `MerkezMalKabul` | Merkez, in goods-receiving |

| Column | Type | Null | Key |
|---|---|---|---|
| StokKodu | varchar(255) | NO | PRI |
| Catalca | int | YES | |
| CatalcaMalKabul | int | YES | |
| Merkez | int | YES | |
| MerkezMalKabul | int | YES | |

**Full-replace pattern:** each run does an unconditional `DELETE FROM warehouse_stock` then a chunked `INSERT` (`CHUNK_SIZE = 5000`, matching `customer_last_price`). Not a composite-key `REPLACE INTO` — there's no `Firma` dimension and the ERP source is itself a nightly full-rebuild, so a plain wipe-and-reload is the honest match. **`DELETE`, not `TRUNCATE`:** the task spec called for `TRUNCATE`, but `reporting_writer`'s grant on this table is `SELECT, INSERT, UPDATE, DELETE` with no `DROP` (confirmed via `SHOW GRANTS`), and `TRUNCATE` requires `DROP` in MySQL — same substitution already made for `cek_senet_portfoy` / `vault_movements_daily`. Same end state for a full-replace table. No staging table, no prune step (a full wipe every run makes both moot).

Refreshed nightly at **03:10** by `refresh_warehouse_stock.py` — see `docs/monitoring.md` for the schedule reasoning and the success/failure Telegram alerting. Visible to Metabase as table id 20 in `reporting-db` (schema rescan triggered 2026-08-28); **no question built yet** — report shape is a follow-up from Arslan.

## `supplier_orders_pending`

Open supplier quotes / purchase requests that have **not yet been turned into a firm order** — the "Siparişte" (order-pending) counterpart to `orders_in_transit`. Sourced from the pre-built ERP view `cansun.aa_rapor_sipariste_hepsi` on Natra via `metabase_ro`. The view filters on `EvrakNoSiparis IS NULL`: a line stays in it only while it's still a pending request, and drops out the instant it's converted into an actual order (at which point it belongs in `orders_in_transit` instead). Despite the `cansun` schema prefix the view is a Cansun/Karacan UNION keyed on `Firma`; as of the 2026-09-01 initial sync it returns **2,194 rows, all `Firma = 'Cansun Sipariste'`** — no Karacan lines pending, not a pipeline bug. **Almer is deferred** (not in the view's UNION yet) — same note as `orders_in_transit`: when Almer joins, re-verify the `(Firma, ID)` primary key is still collision-free across all three companies before trusting the first sync.

**Full replace via `DELETE FROM` + chunked `INSERT` (`CHUNK_SIZE = 5000`), not an upsert.** Because a converted request vanishes from the source view without ever reappearing under the same `ID`, a `REPLACE INTO` / upsert would strand every converted line in the table forever (showing as still-pending) — the same voided/reissued-document staleness pattern behind `sales_snapshot`'s prune step. Wiping and reloading the whole table each run sidesteps it entirely, so there is **no prune step and no staging table**. **`DELETE`, not `TRUNCATE`:** `reporting_writer`'s grant here is `SELECT, INSERT, UPDATE, DELETE` with no `DROP` (`SHOW GRANTS` checked up front), and MySQL's `TRUNCATE` needs `DROP` — same substitution as `warehouse_stock` / `cek_senet_portfoy` / `vault_movements_daily`. No pre-flight duplicate check: the live source is 2,194 rows / 2,194 distinct `(Firma, ID)` pairs, and a full wipe-and-reload can't collapse rows the way an upsert on a bad key would — same reasoning as `warehouse_stock` / `customer_last_price`.

Three source columns are aliased with a space in the name in the ERP view — `` `Doviz Fiyat` ``, `` `Doviz Tutar` ``, `` `Evrak Tutari` `` — and map to `DovizFiyat` / `DovizTutar` / `EvrakTutari` on `reporting-db` (verified against `SHOW COLUMNS` on both sides). The sync uses an explicit source-expression → destination-column map rather than `SELECT *`.

| Column | Type | Null | Key |
|---|---|---|---|
| Firma | varchar(17) | NO | PRI |
| ID | int | NO | PRI |
| EvrakNo | varchar(20) | YES | |
| HesapKodu | varchar(20) | YES | |
| HesapAciklamasi | varchar(150) | YES | |
| StokKodu | varchar(30) | YES | |
| StokAciklamasi | varchar(150) | YES | |
| Miktar | double(20,6) | YES | |
| DovizKodu | varchar(20) | YES | |
| DovizKuru | double(14,6) | YES | |
| DovizFiyat | double(20,6) | YES | |
| DovizTutar | double(20,6) | YES | |
| BelgeTarihi | date | YES | |
| TeslimTarihi | date | YES | |
| TeklifNotlari | text | YES | |
| EvrakTutari | double(20,6) | YES | |

Composite primary key on `(Firma, ID)` — one row per request line per company.

Refreshed nightly at **03:20** by `refresh_supplier_orders_pending.py` — see `docs/monitoring.md` for the schedule reasoning and the failure-only Telegram alerting. Backs the Metabase question **`03_Ithalat_SiparişteÜrünListesi`** (card 105) in the `3. İthalat/İhracat` collection (id 10), a native `SELECT * FROM supplier_orders_pending`, which therefore inherits that folder's cascade (`Director`, `Manager`, `Sales/Purchase`, `Satış` — see `docs/metabase-permissions.md`); no group-level override was created. This table carries no cost column (`FabrikaFiyati`-style), so no boss-only exposure decision was needed.

## `product_min_max_stock`

Per-product min / max / total monthly demand over a rolling 6-month window, alongside current on-hand quantity — the input to a slow-mover / min-max stocking report. **Aggregate table, one row per product** (`ProductId`), not a transaction log — 1,078 rows as of the 2026-09-02 initial sync. Sourced from the pre-built ERP view `cansun.aa_product_min_max_stock` on Natra via `metabase_ro`; the view itself aggregates over an upstream monthly-per-product view, so the `SELECT` against it takes ~11–12s (all of this job's runtime — the local write is sub-second).

| Column | Type | Null | Key | Meaning |
|---|---|---|---|---|
| ProductId | varchar(30) | NO | PRI | product code |
| ProductName | varchar(200) | YES | | |
| MinMonthlyQty | decimal(23,6) | YES | | lowest monthly demand in the window |
| MaxMonthlyQty | decimal(23,6) | YES | | highest monthly demand in the window |
| Total6mQty | decimal(23,6) | YES | | total demand over the 6-month window |
| AvgPerMonthQty | bigint unsigned | YES | | average monthly demand (pre-rounded on the ERP side) |
| QuantityInStorage | int | YES | | current on-hand quantity |
| StokDurumu | varchar(20) | YES | | stocking-status flag computed on the ERP view — `Eksik` (stock insufficient) / `Yeterli` (stock sufficient); only these two values occur |

`QuantityInStorage` is `bigint` on the ERP view side, narrowed to `int` here — a safe, intentional mismatch (on-hand counts, small values), same category as `customer_last_price`'s `FaturaD_ID`. `StokDurumu` is `varchar(7)` `NOT NULL` on the ERP view, widened to a nullable `varchar(20)` here (same widen-and-relax pattern as other text columns); it was added to the source view on 2026-09-03 and picked up the same day.

**Load pattern — `DELETE` + chunked `INSERT` in a single transaction, one commit at the end**, not incremental. This is an aggregate snapshot with no rolling-window staleness to prune, so a full wipe-and-reload is the honest match (same as `customer_last_price`). Two deviations from the spec's literal wording, both deliberate:

- **`DELETE FROM`, not `TRUNCATE`.** `reporting_writer` has `SELECT, INSERT, UPDATE, DELETE` but no `DROP` on this table (`SHOW GRANTS`), and MySQL `TRUNCATE` requires `DROP` — same substitution as `cek_senet_portfoy` / `warehouse_stock`. Separately, `TRUNCATE` is DDL: it forces an implicit commit and **cannot be inside a transaction**, so the spec's own requirement ("single transaction so a mid-run failure never leaves the table empty for readers") is only achievable with `DELETE`.
- **Single transaction, single commit.** The `DELETE` and every `INSERT` batch run in one uncommitted transaction; a mid-run failure rolls back and Metabase keeps seeing the previous 1,078 rows until the new set commits atomically. (Contrast `customer_last_price`, which commits per batch and can briefly expose a partial table.)

**Pre-write sanity check (before the `DELETE`):** the job aborts — leaving the table untouched — if the ERP pull returns 0 rows, or if the row count is under 50% of the previous successful run's count recorded in `job_runs.csv`. A reload pattern has no prune-step equivalent to catch an upstream view silently breaking (e.g. the monthly-per-product view erroring to empty), so this guard is the only thing standing between "view broke" and "report table wiped." Same protective intent as `cek_senet_portfoy`'s pre-flight duplicate check, adapted from an upsert to a reload. On the first run there is no previous count, so only the empty check applies.

Refreshed nightly at **03:25** by `refresh_product_min_max_stock.py` — see `docs/monitoring.md` for the schedule reasoning and the failure-only Telegram alerting. Backs the Metabase question **`01_Üretim_MinMaxStok`** (card 106) in the `5. Envanter Yönetimi` collection (id 11), a native query with an explicit column list, each aliased to a Turkish display name: `ProductId` → `Ürün Kodu`, `ProductName` → `Ürün Adı`, `MinMonthlyQty` → `Aylık Min. Satış`, `MaxMonthlyQty` → `Aylık Maks. Satış`, `Total6mQty` → `6 Aylık Toplam Satış`, `AvgPerMonthQty` → `Aylık Ortalama Satış`, `QuantityInStorage` → `Depo Stok Miktarı`, `StokDurumu` → `Stok Durumu`. The aliases live only in the card's SQL (Metabase's display layer) — the underlying `product_min_max_stock` table and column names in `reporting-db` stay English (see the column table above). Card 106 also carries **conditional formatting** on the `Stok Durumu` column, configured in the Metabase UI (not derivable from the SQL): cell value `Eksik` → red (`#EF8C8C`), `Yeterli` → green (`#88BF4D`). Only those two label states exist in the data; if a third ever appears it will render uncolored until a rule is added. The card's SQL is **not a flat `SELECT`** — it ends with a `WHERE 1=1` plus three `[[ ]]` optional-block filters backed by text template tags: `{{urun_kodu}}` (`Ürün Kodu`, `ProductId LIKE %…%` contains-match, input box), `{{urun_adi}}` (`Ürün Adı`, `ProductName LIKE %…%` contains-match, input box), and `{{stok_durumu}}` (`Stok Durumu`, `StokDurumu = …` exact-match, **dropdown** with a static two-value list `Eksik` / `Yeterli` configured in the UI). Filters target the raw English columns, not the Turkish display aliases; with no filter set the optional blocks drop out and all 1,078 rows return. Folder 5 is the bottom of the permission cascade, so this question is visible to every group (`Director`, `Manager`, `Sales/Purchase`, `Satış`, `Other` — see `docs/metabase-permissions.md`); no group-level override was created. No cost column, so no boss-only exposure decision was needed.
