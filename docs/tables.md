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

Live snapshot of Cansun's three cash vaults ("kasa"), sourced from the pre-built ERP view `aa_vault_status` on Natra. Full-replace via a single unchunked `REPLACE INTO` — no staging table (that pattern is exclusive to `sales_snapshot`) and no chunking (only 3 rows). Refreshed hourly, 09:00–19:00 daily, by `refresh_vault_status.py`. 3 rows as of the 2026-08-21 verification run, matching the ERP view's row count exactly.

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

**Cost-column note:** `FabrikaFiyatiUsd` is a cost column, same category as `sales_snapshot`'s `FabrikaFiyati`/`FabrikaTutarUsd`. The Metabase question built on this table (`Stok Listesi`, card 103) exposes it as part of a deliberate full-detail placeholder — see `docs/metabase-permissions.md` for the collection/permission scoping that keeps it restricted to `Director` and `Manager` only, and the standing note that which columns to actually show is a future decision, not made here.

Refreshed nightly at 03:45 by `refresh_stock_details.py`, registered in `monitored_jobs.yml` and covered by the 07:00 daily digest — see `docs/monitoring.md`.
