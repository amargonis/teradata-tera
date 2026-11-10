# Analysis rules: combining PUVA data and BI documents

Rules for `sql-rag-parallel` answers and for any answer that explains a number. They come from testing the PUVA assistant: in each case the numbers were right but the explanation was not.

## 1. Limit language

The regulatory maximum for rasio PDN is 20%. Describe a value by comparing it with the limit, not by its size alone.

| Value | Wording |
|---|---|
| Above 20% | "melampaui batas maksimum 20%" (a breach). Name the date(s) and the excess in percentage points |
| Exactly 20% | "tepat di batas maksimum" |
| 18% to below 20% | "mendekati batas maksimum" |
| Below 18% | "di bawah batas maksimum" |

Report how many days the bank was above the limit and when it came back under. A breach is the headline of the answer, not a detail.

## 2. Look before the window

When a question asks whether events affected a period ("selama 2-17 Maret 2026, apakah ada pengaruh isu geopolitik?"):

1. Search events and releases from about **two weeks before the start** of the window up to its end, because a trigger usually precedes its effect.
2. List each event with its date and whether it falls before or inside the window.
3. Use `peristiwa_pasar` to find the events quickly, then confirm each one in the release named in `sumber_dokumen`.

## 3. Find the driver in the data first

Before explaining a movement with external events, find which numbers moved it. External context then explains *why* those numbers moved.

| Movement | Check in the data |
|---|---|
| Rasio PDN up or down | `PDN` vs `MODAL` over the period (which one changed?); the bank's FX flows in `TRANSAKSI_HARIAN_BANK`: `DERIVATIF_SPOT_*`, `DERIVATIF_FORWARD_*`, `DERIVATIF_SWAP_*` BELI vs JUAL by counterparty (INDIVIDU, INTERBANK, KORPORASI, NONRESIDEN), summed per week; USD/IDR from `kurs_harian` (revaluation effect) |
| Rupiah weakening | Total `DERIVATIF_SPOT_NONRESIDEN_JUAL` and `_BELI` across banks per day; `kurs_harian` for USD; events and releases |
| Liquidity tightening | Total `TABUNGAN`, `GIRO`, `DEPOSITO` per date; `OM_RUPIAH_LF` use by bank; `PUAB_IDR_*` volumes; `indonia` in `suku_bunga_harian`; `FLAG_RAMADAN` and holidays in `DIM_KALENDER` |
| SRBI holdings change | `OM_RUPIAH_SRBI` positions; `SRBI_BELI` / `SRBI_JUAL` flows; SRBI auction information in the releases |

Show the driver in **Detail** (a small table per week is usually enough) and include its SQL.

Revaluation check for PDN: if the FX position in USD is roughly constant, a PDN change in Rupiah of about the same percentage as the USD/IDR move is revaluation. A much larger change means the bank changed its position through transactions.

## 4. Coincidence vs cause

| Evidence | Wording |
|---|---|
| A document states the link (e.g. "outflows due to geopolitical tension") | "menurut siaran pers BI ..., disebabkan oleh ..." with the citation |
| The data shows the mechanism (e.g. the bank bought USD heavily in the same weeks) | "kenaikan sejalan dengan pembelian valas sebesar ..." |
| Only the dates line up | "bertepatan dengan ..." |
| No support at all | Leave it out, or put it under **Asumsi / catatan** as "dugaan" |

Avoid wording that implies Bank Indonesia acted because of one bank ("BI merespons kenaikan PDN bank X") unless a document says so. An RDG on the same date is a coincidence of timing.

Avoid speculative mechanisms ("ekspansi SRBI dapat meningkatkan aktivitas valas", "bank mengambil posisi sebelum threshold baru") unless the data or a document supports them.

## 5. Policy changes

When a release changes several parameters at once (thresholds, rates, instruments):

1. List each parameter with its old value, new value and direction (dinaikkan, diturunkan, tetap).
2. Do not summarise mixed changes with one word such as "pengetatan" or "pelonggaran"; say which parts tightened and which loosened.
3. Give the effective date if the document states it.

## 6. Periods and dates

- "Akhir Maret" for a position means the last business day of March (`FLAG_AKHIR_BULAN_KERJA = 'Y'`); for a macro figure it means the March row of `makro_bulanan` or the release that reports March.
- "Selama periode" for a position means the series of daily values (trend, minimum, maximum), not their sum.
- Weeks: group by `DIM_KALENDER.MINGGU_KE` and label each week by its first business day.
- Holidays: a missing date inside a period is normally a holiday or collective leave; say so instead of reporting missing data.

## 7. Sources and labels

- Each number in the answer should trace to a table (with the SQL shown) or a document (with release number and page).
- General background (how PDN regulation works elsewhere, textbook economics) is allowed only when labelled "latar belakang umum", and never as a substitute for a missing source.
- If one source fails (e.g. the chunk table is not readable), say which, and answer from the sources that work.

## 8. Self-check before sending

- [ ] Positions taken on a date, not summed across dates.
- [ ] Units stated; USD and Rupiah not mixed.
- [ ] Limit wording matches the value (rule 1).
- [ ] Events searched from about two weeks before the window (rule 2).
- [ ] The driver in the data is shown before any external explanation (rule 3).
- [ ] Causal words supported by data or a cited document (rule 4).
- [ ] Policy changes listed per parameter (rule 5).
- [ ] SQL and sources included.
