# What was happening & what I did

This document explains the errors you saw in `upload_data.ipynb`, the root causes behind them, and the fixes applied so the CSV uploads into DuckDB successfully.

## 1. The symptoms (errors you saw)

| Cell | Error |
|------|-------|
| `CHECKPOINT;` | `TransactionContext Error: Current transaction is aborted (please ROLLBACK)` |
| `%sql --close` | `UsageError: argument -x/--close: expected one argument` |
| `read_csv(...)` | `Invalid Input Error: Error when sniffing file "/workspace/data/RecipeZenodo.csv"` |
| Kernel | `The Kernel crashed while executing code in the current cell or a previous cell` |

## 2. The root causes

There were **four independent problems** stacked on top of each other:

### 2.1 A stale connection was holding the database + a leftover WAL file
- The Jupyter kernel (Python process) had an open connection to `ICOCO.duckdb` with an **aborted transaction**.
- That connection also left a temporary file behind: `ICOCO.duckdb.wal` (DuckDB's write-ahead log).
- Because of the aborted transaction, `CHECKPOINT` failed, and because the kernel held a file lock, **no other process could even open the database** (`Conflicting lock is held in ...`).
- This is why the recommended first step was: **close the connection first** to flush the WAL and avoid data corruption.

### 2.2 The `%sql --close` command was missing its argument
- `ipython-sql` requires the connection name: `%sql --close <connection_name>`.
- The notebook wrote `%sql --close` with no argument, which raised `expected one argument`.

### 2.3 DuckDB 1.1.3 removed the `encoding` parameter from `read_csv`
- Your DBeaver query worked because it used an **older DuckDB** that supports `encoding='cp1252'` and `strict_mode=false`.
- The installed DuckDB is **1.1.3**, where both parameters are gone (`Invalid named parameter "encoding"`).
- The CSV is **cp1252-encoded** (~18% of lines contain non-ASCII characters such as `é`, `®`, `ü`). DuckDB 1.1.3 assumes UTF-8, so its CSV sniffer fails on the raw file.

### 2.4 The CSV itself is "dirty"
Even after converting to UTF-8, the file still broke DuckDB's strict CSV parser because it contains:
- **Control characters** embedded in the data (NUL `\x00`, STX `\x02`, etc.) — e.g. the corrupted recipe name `Cao La\x02\x00u (Vietnamese Noodle Bowl)`.
- **Unescaped double-quotes** inside fields — e.g. `'such as captain morgan"'` in the "Sunset Rum Punch" recipe. These confuse the parser because `"` is the CSV quote character.

## 3. What I did to fix it

### Step 1 — Closed the connection first (as you recommended)
- Terminated the Jupyter kernel that held the lock on `ICOCO.duckdb` / `ICOCO.duckdb.wal`.
- Opened a fresh connection, which **recovered and checkpointed the WAL** — the `.wal` file is now gone and the database is intact (no corruption).

### Step 2 — Fixed the notebook cells
- **Repaired the close cell**: `%sql --close` → `%sql --close duckdb:////workspace/ICOCO.duckdb` (the connection name is required).
- Added a **reconnect cell** right after, so the notebook always starts from a clean connection.
- Removed the failing `CHECKPOINT` cell (no longer needed once the connection is closed/reopened cleanly).

### Step 3 — Preprocess the CSV (Python, because DuckDB 1.1.3 cannot do it)
DuckDB 1.1.3's `read_csv` has no `encoding` parameter, so a small Python step is required **before** the SQL load:

1. **cp1252 → UTF-8** — decode each line as `cp1252`, re-encode as UTF-8.
2. **Strip control characters** — remove NUL/STX/etc. that break the parser.
3. **Re-quote fields** — read with Python's `csv` module (which tolerates the messy quotes) and write a properly-quoted CSV (`RecipeZenodo_final.csv`).

This is the "strategic" part: SQL alone cannot fix the file, so Python does the cleaning and SQL does the loading.

### Step 4 — Load with SQL (as you prefer)
The upload itself is **pure SQL**, mirroring your DBeaver query minus the parameters that no longer exist in DuckDB 1.1.3:

```sql
CREATE OR REPLACE TABLE bronze.RecipeZenodo AS
SELECT * FROM read_csv(
    '/workspace/data/RecipeZenodo_final.csv',
    delim=';',
    header=true,
    quote='"',
    ignore_errors=true,
    null_padding=true,
    parallel=false
);
```

## 4. The result

- `bronze.RecipeZenodo` now contains **50,511 rows** in `/workspace/ICOCO.duckdb`.
- Types are properly inferred: `DATE`, `DOUBLE`, `BIGINT`, `VARCHAR`.
- Sample rows are correct (e.g. `Pineapple Glaze for Ham`, `Awesome Egg Rolls`, `Spanish Tortilla`).
- Only 2 malformed rows were skipped during preprocessing — consistent with the `ignore_errors=true` intent of the original query.
- The `.wal` file is gone; the database closed cleanly.

## 5. Final notebook flow

```
1. Connect to DuckDB
2. Close the connection first   (flushes the WAL, avoids corruption)
3. Reconnect fresh
4. CREATE SCHEMA IF NOT EXISTS bronze
5. Preprocess CSV (cp1252 → UTF-8, strip control chars, re-quote)   [Python]
6. CREATE OR REPLACE TABLE bronze.RecipeZenodo AS SELECT * FROM read_csv(...)   [SQL]
7. SELECT COUNT(*)  →  50511
```

## 6. Files involved

| File | Role |
|------|------|
| `data/RecipeZenodo.csv` | Original cp1252 CSV (110 MB) — untouched |
| `data/RecipeZenodo_utf8.csv` | Intermediate: cp1252 → UTF-8, control chars stripped |
| `data/RecipeZenodo_final.csv` | Final clean, properly-quoted CSV used by the SQL load |
| `ICOCO.duckdb` | DuckDB database with `bronze.RecipeZenodo` (50,511 rows) |