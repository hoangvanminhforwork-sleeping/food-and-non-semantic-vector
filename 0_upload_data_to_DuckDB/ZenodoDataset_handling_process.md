# Zenodo Dataset Handling Process

A chronological, one-by-one account of the whole conversation: the initial information I was given, every error/trouble I hit, and what I did next to inspect and resolve each one.

---

## 1. My advantage: the initial information you gave me

Before I started, you provided these pieces of context that guided everything:

| # | Information | How it helped |
|---|-------------|---------------|
| 1 | **Notebook address**: `/workspace/0_upload_data_to_DuckDB.ipynb` | Told me exactly which file to fix. |
| 2 | **DuckDB file address**: `/workspace/ICOCO.duckdb` | Told me where the database lives. |
| 3 | **CSV address**: `/workspace/data/RecipeZenodo.csv` | Told me which data file to load. |
| 4 | **Mission**: "Fix the bug in the notebook: try to upload the csv to the duckdb successfully, prioritize using sql commands" | Set the goal and the constraint (SQL-first). |
| 5 | **Recommended approach**: "Close connection first, because we still have a .wal file (temp file for duckdb) here `/workspace/ICOCO.duckdb.wal`. To minimize data corruption" | This was the key hint — it pointed me to the stale-connection / WAL problem before I even looked at the code. |
| 6 | **Your working DBeaver SQL** with `encoding='cp1252'`, `strict_mode=false`, `ignore_errors=true`, `null_padding=true`, `parallel=false` | Revealed the CSV is **cp1252-encoded** and that the intended load parameters were known-good in your environment. |

These six pieces of context were the "advantage" — they told me where to look, what the goal was, and what the data's encoding was before I ran a single command.

---

## 2. The errors and troubles I encountered (in order)

### 2.1 `CHECKPOINT;` → "Current transaction is aborted (please ROLLBACK)"
The notebook's second cell failed because the open connection had an aborted transaction.

### 2.2 `%sql --close` → "argument -x/--close: expected one argument"
The close cell was missing the required connection-name argument.

### 2.3 `read_csv(...)` → "Error when sniffing file ... It was not possible to automatically detect the CSV Parsing dialect/types"
The main upload query could not even sniff the CSV.

### 2.4 The Jupyter kernel crashed
The saved notebook output showed "The Kernel crashed while executing code in the current cell or a previous cell."

### 2.5 `Conflicting lock is held in /usr/local/bin/python3.10 (PID 10089)`
When I tried to open the database myself, DuckDB refused: the Jupyter kernel was still holding a file lock on `ICOCO.duckdb` and `ICOCO.duckdb.wal`.

### 2.6 `Invalid named parameter "strict_mode" for function read_csv`
I copied your DBeaver query, but DuckDB 1.1.3 does not have `strict_mode`.

### 2.7 `Invalid named parameter "encoding" for function read_csv`
DuckDB 1.1.3 also removed the `encoding` parameter — so the cp1252 file could not be read directly.

### 2.8 Sniffing still failed even on a UTF-8-converted file
I converted the file to UTF-8, but `read_csv` still could not sniff it.

### 2.9 `_csv.Error: line contains NUL`
Python's `csv` module found NUL bytes in the file.

### 2.10 Control characters everywhere
A scan found NUL (`\x00`), STX (`\x02`) and other control bytes scattered through the data (e.g. the corrupted name `Cao La\x02\x00u`).

### 2.11 Sniffing still failed after stripping control characters
Even a clean UTF-8 file with no control chars failed to sniff.

### 2.12 The breaking line: line 1725 with `kraken®`
Bisection found the sniffer broke at line 1725 ("Sunset Rum Punch"), which contains the `®` character.

### 2.13 Unescaped double-quotes inside fields
Line 1725 also contains raw `"` characters inside data (e.g. `'such as captain morgan"'`), which break DuckDB's strict CSV parser.

### 2.14 `auto_detect=false` produced garbage rows
Bypassing the sniffer with explicit columns worked syntactically but returned misaligned/null rows.

### 2.15 Python's `csv` module read the file correctly
Python's lenient parser handled the messy quotes (50,513 data rows, only 2 malformed).

### 2.16 My own preprocessing cell initially failed with "line contains NUL"
When I first wrote the notebook's preprocessing cell, I forgot to strip control characters *before* the `csv.reader` step — so the notebook itself failed on the NUL bytes. (Fixed by stripping in Step 1.)

---

## 3. What I did next to inspect and resolve each trouble

### 3.1 Inspected the environment first
- Listed the filesystem: confirmed `ICOCO.duckdb` (12 KB), `ICOCO.duckdb.wal` (59 bytes), and `data/RecipeZenodo.csv` (110 MB) all existed.
- Checked the Python environment: `duckdb 1.1.3`, `pandas 2.2.3`, `sqlalchemy 2.0.54`, `ipython-sql` installed.
- Inspected the CSV header with `od -c`: semicolon-delimited, CRLF line endings, 18 columns.

### 3.2 Diagnosed the lock (troubles 2.1–2.5)
- Tried to open the DB → got the "Conflicting lock" error.
- Inspected `/proc/10089` → confirmed PID 10089 was the Jupyter kernel (`ipykernel_launcher`) with open file descriptors to both `ICOCO.duckdb` and `ICOCO.duckdb.wal`.
- **Resolution**: terminated that kernel (the notebook file was already saved, so no work was lost), then opened a fresh connection and ran `CHECKPOINT` — the WAL was recovered and flushed (`.wal` file disappeared).

### 3.3 Tested the SQL parameters (troubles 2.6–2.7)
- Tried your DBeaver query → `strict_mode` rejected.
- Tried `encoding='cp1252'` → `encoding` rejected.
- **Resolution**: accepted that DuckDB 1.1.3 needs a different approach — the file must be converted to UTF-8 before `read_csv`.

### 3.4 Isolated the sniffing failure (troubles 2.8–2.12)
- Converted a 200-line sample to UTF-8 → it parsed fine. So the problem was beyond the first rows.
- Tested `sample_size` values → all failed on the full file.
- Created progressively larger sample files → found the break between 1,000 and 2,000 lines.
- Binary-searched → **line 1725** ("Sunset Rum Punch", contains `kraken®`).
- **Resolution**: realized the sniffer reads ~20,480 rows regardless of `LIMIT`, so one bad row anywhere in that window kills sniffing.

### 3.5 Found the real data problems (troubles 2.9–2.13)
- Scanned for control bytes → NUL/STX/etc. present.
- Stripped them → still failed.
- Examined line 1725 in full → found **unescaped double-quotes** inside fields (the true parser-killer).
- **Resolution**: Python's `csv` module (lenient) reads the file correctly; DuckDB's strict parser does not. So the fix is to **re-emit a properly-quoted CSV** with Python, then load with SQL.

### 3.6 Built the working pipeline
- **Preprocess (Python)**: cp1252 → UTF-8, strip control characters, re-quote fields via `csv.reader`/`csv.writer` → `data/RecipeZenodo_final.csv` (50,512 rows, 2 malformed skipped).
- **Load (SQL)**: `CREATE OR REPLACE TABLE bronze.RecipeZenodo AS SELECT * FROM read_csv('.../RecipeZenodo_final.csv', delim=';', header=true, quote='"', ignore_errors=true, null_padding=true, parallel=false)` → **50,511 rows** in ~10 s.
- **Verify**: `SELECT COUNT(*)` → 50,511; sample rows correct; types DATE/DOUBLE/BIGINT/VARCHAR.

### 3.7 Fixed the notebook (troubles 2.1–2.2, 2.16)
- Repaired `%sql --close` → `%sql --close duckdb:////workspace/ICOCO.duckdb`.
- Added a reconnect cell after the close.
- Removed the failing `CHECKPOINT` cell.
- Fixed my own preprocessing cell bug (strip control chars in Step 1, before `csv.reader`).
- Executed the whole notebook end-to-end with `jupyter nbconvert --execute` → all cells pass, final `row_count = 50511`.

### 3.8 Documented everything
- Wrote the technical explanation (now `ZenodoDataset_handling_note.md`).
- Wrote this process document.

---

## Final result

- `bronze.RecipeZenodo` in `/workspace/ICOCO.duckdb` → **50,511 rows**, correct types, correct sample data.
- The `.wal` file is gone; the database closed cleanly (no corruption).
- The notebook at `0_upload_data_to_DuckDB/upload_data.ipynb` runs end-to-end without errors.