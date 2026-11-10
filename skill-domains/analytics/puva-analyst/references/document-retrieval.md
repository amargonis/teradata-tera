# Document retrieval (`FIN_PUVA_RAG_KB`)

How to find and read the Bank Indonesia press releases behind `rag-only` and `sql-rag-parallel` answers.

## What the collection holds

| Item | Value |
|---|---|
| Collection | `FIN_PUVA_RAG_KB` (Enterprise Vector Store, AI Studio) |
| Content | Bank Indonesia press releases (siaran pers), Q1 2026: RDG decisions, Rupiah and FX market conditions, reserves, policy instruments |
| Language | Bahasa Indonesia |
| Embedding | Amazon Titan Text Embeddings V2, chunk size 1500, cosine similarity |
| Chunk table | In database `demo-user@demo-workshop.com`, name starting with `FIN_PUVA_RAG_KB` and containing `chunks` |
| Chunk columns | `TD_FILENAME` (file name), `page_number`, `TD_FILESPLITS` (chunk text) |
| File names | Start with the release number, e.g. `BI_SP_28-55-DKom_...pdf` |

The collection is small (a handful of releases), so reading all chunks of a few candidate documents is acceptable when targeted search is not possible.

## Option 1: similarity-search tool

If a vector-search or RAG tool for `FIN_PUVA_RAG_KB` is available, use it first because it ranks chunks by meaning rather than keywords.

1. Query in Bahasa Indonesia with the key terms of the question, e.g. "cadangan devisa akhir Maret 2026", "keputusan RDG Maret 2026 BI-Rate".
2. Retrieve enough chunks (top 5 to 8) to cover releases for neighbouring months, because the same indicator often appears in several releases.
3. Continue with **Reading the text** below.

## Option 2: SQL on the chunk table

Use this when no similarity-search tool is available. All statements are read-only.

### Step 1: find the chunk table
```sql
SELECT TableName
FROM DBC.TablesV
WHERE DatabaseName = 'demo-user@demo-workshop.com'
  AND TableName LIKE 'FIN_PUVA_RAG_KB%chunks%';
```
If more than one table matches, use the most recent one (the name carries a timestamp) and mention it under **Asumsi / catatan**.

### Step 2: list the documents
```sql
SELECT TD_FILENAME, COUNT(*) AS jumlah_chunk
FROM "demo-user@demo-workshop.com".<chunk_table>
GROUP BY 1
ORDER BY 1;
```

### Step 3: search by keyword
Search with the key terms and their Bahasa synonyms. Teradata `LIKE` is case-sensitive on some column types, so compare in upper case.
```sql
SELECT TD_FILENAME, page_number, TD_FILESPLITS
FROM "demo-user@demo-workshop.com".<chunk_table>
WHERE UPPER(TD_FILESPLITS) LIKE '%CADANGAN DEVISA%'
ORDER BY TD_FILENAME, page_number;
```

Useful synonym sets:

| Topic | Search terms |
|---|---|
| Reserves | `CADANGAN DEVISA`, `MILIAR DOLAR AS` |
| Policy rate | `BI-RATE`, `RAPAT DEWAN GUBERNUR`, `RDG`, `DEPOSIT FACILITY`, `LENDING FACILITY` |
| Rupiah | `NILAI TUKAR RUPIAH`, `RUPIAH`, `MELEMAH`, `MENGUAT`, `DEPRESIASI` |
| Geopolitics | `GEOPOLITIK`, `TIMUR TENGAH`, `PERANG`, `KETIDAKPASTIAN GLOBAL` |
| Ratings | `FITCH`, `OUTLOOK`, `PERINGKAT` |
| SRBI and money market | `SRBI`, `PUAB`, `INDONIA`, `LIKUIDITAS` |
| FX market measures | `DNDF`, `FORWARD`, `SWAP`, `THRESHOLD`, `UNDERLYING` |
| Trade | `NERACA PERDAGANGAN`, `SURPLUS`, `EKSPOR`, `IMPOR` |

### Step 4: read whole documents when needed
If the keyword search returns nothing useful, or the question needs the full argument of one release (e.g. "jelaskan keputusan RDG"), read all chunks of the candidate document in page order:
```sql
SELECT page_number, TD_FILESPLITS
FROM "demo-user@demo-workshop.com".<chunk_table>
WHERE TD_FILENAME = '<file name>'
ORDER BY page_number;
```

### Step 5: use `peristiwa_pasar` as an index, not as evidence
`FIN_PUVA.peristiwa_pasar.sumber_dokumen` names the release behind each event. Use it to find the right document quickly, then quote the document itself, because the table holds only a one-line summary.

## Reading the text

1. **Match the period asked, not the release date.** A March release can report February reserves; an April release reports March. Find the sentence that names the month in the question.
2. **Quote numbers exactly as written**, with their unit (e.g. "USD1,2 miliar", "5,25%").
3. **Record the citation**: file name, release number (from the file name, e.g. `28/55/DKom`) and page.
4. **Report what each document says, not what it implies.** When a release lists several policy changes, list each one with its direction (raised, lowered, unchanged) rather than summarising them with one word.
5. **Say what is missing.** If no chunk answers the question, write "tidak disebutkan dalam dokumen" and name the documents that were searched.

## Failure handling

| Situation | What to do |
|---|---|
| Chunk table not found | Say the document collection is not reachable here; answer only the data part, if any |
| Permission denied on the chunk table | Same as above, and name the database that could not be read |
| Keyword search returns too many chunks | Narrow with a second term (month name, instrument), or restrict to the release closest to the asked period |
| Two documents give different numbers | Report both with their dates and explain which one matches the period asked |
