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
