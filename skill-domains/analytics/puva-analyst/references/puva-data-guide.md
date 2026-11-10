# PUVA data guide (`FIN_PUVA`)

How the PUVA tables are organised and how to query them correctly. The table and column comments in `FIN_PUVA` are the authoritative definitions; this guide summarises them so the agent knows what to look for. If a Context Engine data product for PUVA is available, its descriptions take precedence over both.

## Tables

| Table | Grain | Kind | Use for |
|---|---|---|---|
| `DIM_BANK` | One row per bank | Reference | Bank name, group, business type, KBMI, FX-bank flag |
| `DIM_KALENDER` | One row per calendar day, Q1 2026 | Reference | Business days, holidays, month-end business day, Ramadan |
| `POSISI_HARIAN_BANK` | Bank × business day | **Position** (end-of-day snapshot) | DPK, kredit, MODAL, PDN, rasio PDN, SBN, OM Rupiah, OM Valas, PUAB outstanding |
| `TRANSAKSI_HARIAN_BANK` | Bank × business day | **Flow** (daily volume) | PUAB transactions, SRBI trading, valas derivatif, repo |
| `MASTER_OMPMPUVA` | Bank × business day | View | BI's original wide layout (positions + transactions + NAMA_BANK, KELOMPOK_BANK) |
| `kurs_harian` | Currency × business day | Market reference | JISDOR-style kurs beli, tengah, jual |
| `suku_bunga_harian` | Business day | Market reference | BI-Rate, DF rate, LF rate, INDONIA |
| `makro_bulanan` | Month | Market reference | Cadangan devisa, months of imports, trade balance, inflation |
| `peristiwa_pasar` | Event | Market reference | Short summaries of market and policy events, each with its source document |

Keys: `SANDI_BANK` joins to `DIM_BANK`; `CALENDAR_SK` joins to `DIM_KALENDER.CALENDAR_SK` and to `tanggal` in the market tables.

Prefer the narrow tables (`POSISI_HARIAN_BANK`, `TRANSAKSI_HARIAN_BANK`) for analysis because the position/flow distinction is explicit. Use `MASTER_OMPMPUVA` when the user refers to BI's original column layout.

## `DIM_BANK`

| Column | Meaning |
|---|---|
| `SANDI_BANK` | Three-character bank code (text, not a number: quote it, e.g. `'999'`) |
| `NAMA_BANK` | Official name |
| `KELOMPOK_BANK` | `PERSERO` (state-owned), `BUSN` (private national, including sharia and digital subsidiaries), `BPD` (regional development banks), `KCLN` (branches of foreign banks) |
| `JENIS_USAHA` | `KONVENSIONAL` or `SYARIAH` |
| `KBMI` | Core-capital group, KBMI 1 to KBMI 4 |
| `FLAG_PRIMARY_DEALER` | `Y` / `N` |
| `FLAG_BANK_DEVISA` | `Y` if the bank may trade FX. Only FX banks have PDN, OM Valas and FX/derivative data |
| `SANDI_BANK_INDUK` | Parent bank code for subsidiaries, empty otherwise |

Bank-name matching: use `UPPER(NAMA_BANK) LIKE '%GARUDA%'` and add `AND UPPER(NAMA_BANK) NOT LIKE '%SYARIAH%' AND UPPER(NAMA_BANK) NOT LIKE '%DIGITAL%' AND UPPER(NAMA_BANK) NOT LIKE '%BANTEN%'` unless the user names one of those variants. If more than one bank still matches, list the candidates and state which one was used under **Asumsi**.

`KELOMPOK_BANK = 'BUSN'` corresponds to the `BSN` group in the `PUAB_OS_BSN_*` columns.

## `DIM_KALENDER`

| Column | Meaning |
|---|---|
| `CALENDAR_SK` | Date |
| `FLAG_HARI_KERJA` | `Y` on business days. Positions and transactions exist only on business days |
| `KETERANGAN_LIBUR` | Holiday or collective leave name |
| `FLAG_AKHIR_BULAN_KERJA` | `Y` on the last business day of the month: use it for month-end positions |
| `FLAG_RAMADAN` | `Y` during Ramadan (approximate); withdrawals usually rise before Idul Fitri |
| `MINGGU_KE`, `BULAN`, `NAMA_BULAN`, `KUARTAL`, `TAHUN` | Calendar attributes for grouping |

A date with no data is usually a holiday, not missing data. Check `KETERANGAN_LIBUR` before reporting a gap.

## `POSISI_HARIAN_BANK` (positions)

End-of-day snapshots. **Never sum across dates.** For "total X in March", take the value on the last business day of March. Summing across banks on the same date is correct.

| Column family | Unit | Notes |
|---|---|---|
| `GIRO`, `TABUNGAN`, `DEPOSITO` | Rupiah | Third-party funds (DPK) |
| `KREDIT_BANK`, `KREDIT_NON_BANK` | Rupiah | Loans to banks / to non-banks |
| `MODAL` | Rupiah | Capital, the denominator of rasio PDN |
| `PDN` | Rupiah | Net open FX position. Positive = long FX, negative = short. FX banks only |
| `RASIO_PDN` | **Percent** | `PDN / MODAL × 100`. Regulatory maximum 20%. FX banks only |
| `NOMINAL_SBN` | Rupiah | Government securities held |
| `OM_RUPIAH_*` (DF, DFS, LF, RP, RR, SRBI, SUKBI, SWAP_RP) | Rupiah | Rupiah monetary-operation positions. LF use signals a liquidity need. DFS, SUKBI: sharia banks only. SRBI: conventional banks only. RP and SWAP_RP are stored as positive values |
| `OM_VALAS_*` (DHESDA, SUVBI, SVBI, SWAP, TD_SYARIAH, TD_VALAS) | **USD** | FX monetary-operation positions, not Rupiah |
| `PUAB_OS_<group>_BERI` / `_PINJAM` | Rupiah | Interbank outstanding lent to / borrowed from BPD, PERSERO, BSN, KCLN |

## `TRANSAKSI_HARIAN_BANK` (flows)

Daily volumes. Summing across dates is correct.

| Column family | Unit | Notes |
|---|---|---|
| `PUAB_IDR_BERI`, `_PINJAM`, `_NETT` | Rupiah | Interbank money market. NETT = beri − pinjam |
| `SRBI_BELI`, `_JUAL`, `_NETT` | Rupiah | SRBI trading. NETT = beli − jual |
| `DERIVATIF_<instrument>_<counterparty>_<BELI, JUAL or NETT>`, instrument = SPOT, FORWARD, SWAP; counterparty = INDIVIDU, INTERBANK, KORPORASI, NONRESIDEN | **USD** | FX transactions by instrument and counterparty. NETT = beli − jual (bank's perspective) |
| `REPO_<counterparty>_GIVER`, `_TAKER`, `_NETT`, counterparty = BI, NONBI, UNKNOWN | Rupiah | Repo by counterparty. **NETT = taker − giver**, the opposite direction to the other NETT columns |

## Empty vs zero

PDN, RASIO_PDN, OM Valas and all `DERIVATIF_*` columns are empty (NULL) for non-FX banks (`FLAG_BANK_DEVISA = 'N'`). Exclude them from rankings and averages of those columns, and say so; do not treat NULL as 0. Use `COALESCE(col, 0)` only when summing a column where NULL genuinely means "no activity", such as `OM_RUPIAH_LF`.

## Market reference tables

| Table | Columns | Notes |
|---|---|---|
| `kurs_harian` | `tanggal`, `mata_uang`, `satuan`, `kurs_beli`, `kurs_tengah`, `kurs_jual` | Rupiah per unit of currency. JPY is quoted per 100 JPY (`satuan = 100`). Convert USD to Rupiah with `kurs_tengah` for `mata_uang = 'USD'` on the same date |
| `suku_bunga_harian` | `tanggal`, `bi_rate`, `deposit_facility_rate`, `lending_facility_rate`, `indonia` | Percent |
| `makro_bulanan` | `bulan`, `cadangan_devisa_miliar_usd`, `bulan_impor`, `bulan_impor_dan_uln`, `neraca_perdagangan_miliar_usd`, `inflasi_ihk_yoy`, `sumber` | `bulan` is the first day of the month the figure refers to |
| `peristiwa_pasar` | `tanggal`, `kategori`, `judul`, `deskripsi`, `presisi_tanggal`, `sumber_dokumen` | A short summary only. For explanations, open the document in `sumber_dokumen` through the Document route |

## Query patterns

Month-end position for one bank:
```sql
SELECT b.NAMA_BANK, p.SANDI_BANK, p.CALENDAR_SK, p.PDN, p.RASIO_PDN
FROM FIN_PUVA.POSISI_HARIAN_BANK p
JOIN FIN_PUVA.DIM_BANK b ON b.SANDI_BANK = p.SANDI_BANK
JOIN FIN_PUVA.DIM_KALENDER k ON k.CALENDAR_SK = p.CALENDAR_SK
WHERE k.FLAG_AKHIR_BULAN_KERJA = 'Y' AND k.BULAN = 3 AND k.TAHUN = 2026
  AND p.SANDI_BANK = '999';
```

Ranking on one date (FX banks only):
```sql
SELECT TOP 5 b.NAMA_BANK, p.SANDI_BANK, p.RASIO_PDN
FROM FIN_PUVA.POSISI_HARIAN_BANK p
JOIN FIN_PUVA.DIM_BANK b ON b.SANDI_BANK = p.SANDI_BANK
WHERE p.CALENDAR_SK = DATE '2026-03-05' AND p.RASIO_PDN IS NOT NULL
ORDER BY p.RASIO_PDN DESC;
```

Flow summed over a period, by week:
```sql
SELECT k.MINGGU_KE,
       SUM(t.DERIVATIF_SPOT_NONRESIDEN_JUAL) / 1e6 AS jual_spot_nonresiden_juta_usd
FROM FIN_PUVA.TRANSAKSI_HARIAN_BANK t
JOIN FIN_PUVA.DIM_KALENDER k ON k.CALENDAR_SK = t.CALENDAR_SK
WHERE t.CALENDAR_SK BETWEEN DATE '2026-02-02' AND DATE '2026-03-31'
GROUP BY k.MINGGU_KE
ORDER BY k.MINGGU_KE;
```

USD flow converted to Rupiah on the same date:
```sql
SELECT t.CALENDAR_SK,
       SUM(t.DERIVATIF_SPOT_KORPORASI_BELI) AS beli_usd,
       SUM(t.DERIVATIF_SPOT_KORPORASI_BELI * k.kurs_tengah) / 1e12 AS beli_triliun_rupiah
FROM FIN_PUVA.TRANSAKSI_HARIAN_BANK t
JOIN FIN_PUVA.kurs_harian k ON k.tanggal = t.CALENDAR_SK AND k.mata_uang = 'USD'
WHERE t.CALENDAR_SK BETWEEN DATE '2026-03-02' AND DATE '2026-03-06'
GROUP BY t.CALENDAR_SK
ORDER BY t.CALENDAR_SK;
```

Teradata notes: use `TOP n` or `QUALIFY ROW_NUMBER() OVER (...)` rather than `LIMIT`; write dates as `DATE 'YYYY-MM-DD'`; quote `SANDI_BANK` values as text.