# Add AI-labeled (standardized) data to DuckDB — Note

## 1. Mission

Write a reproducible notebook that ingests the LLM-standardization batch outputs (JSONL) from
`/workspace/data/{scientific_name, odor, cuisine_normalize, ingredient_standardize}` into the DuckDB
file `/workspace/ICOCO.duckdb`, one table per `{task folder}.{output version}`.

Target tables (schema name == source folder name):

| Table | Rows | Content |
|---|---|---|
| `scientific_name.output1/2/3` | 4 280 each | scientific names (list) |
| `odor.output1/2/3` | 3 792 each | perceived odors (list) |
| `cuisine_normalize.output1/2/3` | 116 each | country / sub-region / continent (object) |
| `ingredient_standardize.output1/2/3` | 17 017 each | normalized ingredients (list) |

Deliverables in this folder:
- `add_AI_labeled_data_to_bronze.ipynb` — the notebook (idempotent).
- `add_AI_labeled_data_to_bronze_note.md` — this note.

## 2. What I did, what troubles I encountered, how I resolved them

### 2.1 Inspect the raw data structure before writing any code
<font color="blue">I inspected the environment and data folders through terminal commands</font>
(`tree`, `ls`, `head`). Each `outputN.jsonl` is a raw OpenAI-style `/v1/chat/completions` response per
line; the real answer is buried at `response.body.choices[0].message.content`, and it is itself a JSON
**string**. `input_text` is **not** in the response, so it must be joined from `merged_results.jsonl`
via `custom_id`.

### 2.2 The content is not always valid JSON
- <font color="red">Consequence</font>: A few responses (e.g. `scientific_name`) end with a trailing markdown
  code fence, e.g. `["Vanilla planifolia"]``` ` . Trying to `json.loads` the raw string would fail and
  mark the row as `parsed = FALSE` even though the answer is valid.
- <font color="green">Resolve</font>: Strip leading/trailing ``` code fences before `json.loads`. This was
  confirmed by a delta-check: without stripping, `scientific_name.output1` had 11 mismatching rows vs the
  reference; after stripping it dropped to 0.

### 2.3 Empty results vs "not applicable" is semantically meaningful
- <font color="red">Consequence</font>: If I stored every empty result as `NULL`, I would lose the distinction
  between "the model returned an empty list `[]`" (valid answer) and "this field is not applicable" / "parse failed".
- <font color="green">Resolve</font>: Keep **empty results as real empty arrays `[]`**, and leave fields as
  `NULL` only when not applicable (e.g. `country_list` on a list-type row) or when `parsed = FALSE`. I verified
  this matches the reference table exactly (e.g. `scientific_name.output1` keeps 212 `value_list = NULL` rows,
  which are precisely the 212 rows where `parsed = FALSE`).

### 2.4 DuckDB allows only one connection — reuse the `%sql` one
- <font color="red">Consequence</font>: Opening a second `duckdb.connect()` while the `%sql` connection is open
  can corrupt/lock the DB (project rule #5a). My first attempt used the wrong attribute path
  `engine.raw_connection()`, raising `AttributeError: 'Connection' object has no attribute 'raw_connection'`.
- <font color="green">Resolve</font>: Follow the established connection/disconnection cells (connect → close → reconnect)
  and, inside the data cell, reach the **same** underlying DuckDB handle via
  `ConnectionManager().current.connection_sqlalchemy.connection.driver_connection`
  (a `duckdb_engine.ConnectionWrapper` exposing `register`/`unregister`/`execute`).

### 2.5 Writing DuckDB `VARCHAR[]` lists back
- <font color="red">Consequence</font>: `pandas.to_sql` does not work cleanly with a raw DuckDB cursor and does
  not infer the list/array type reliably.
- <font color="green">Resolve</font>: Use DuckDB's native `register(df)` + `CREATE OR REPLACE TABLE … AS SELECT * FROM _stage`,
  which infers `VARCHAR[]` for the list columns and is idempotent (safe to re-run).

### 2.6 Token counts must stay integer
- <font color="red">Consequence</font>: `prompt_tokens`/`completion_tokens` are integers, but pandas upcasts a
  numeric column to `float` as soon as one row is missing, and DuckDB then stores `DOUBLE` (e.g. `1` → `1.0`).
- <font color="green">Resolve</font>: Cast both token columns to pandas **nullable `Int64`** before `register`,
  so DuckDB stores them as `BIGINT` with proper `NULL`s.

### 2.7 Notebook-building escaped me twice
- <font color="red">Consequence</font>: When generating the `.ipynb` JSON programmatically, I hit python
  `SyntaxError` from inner `"""` docstrings inside an outer `"""` string, then stray artifact text.
- <font color="green">Resolve</font>: Build every cell source with `'''` triple-quoted strings (no docstring inside)
## 3. Can I prove the task is done?

- The notebook **executed end-to-end** with `jupyter nbconvert --execute` (0 error cells).
- Row counts per table: `scientific_name` 4 280, `odor` 3 792, `cuisine_normalize` 116, `ingredient_standardize` 17 017
  (×3 versions), each equal to the number of lines in its source `outputN.jsonl`.
- A per-row differential check of all **12 tables** (custom_id → input_text, status_code, model, parsed,
  value_list lists, content_raw, and both token counts) against the freshly-parsed source gives **0 mismatches**.
- Idempotent: running the notebook again `CREATE OR REPLACE`s the tables with the same values.
- No `ICOCO.duckdb.wal`/lock/temporary files remain.

## 4. Things you may want to review

### 4.1 Pre-existing tables were already populated
Before I started, `scientific_name.output1/2/3`, `odor.output1/2/3`, `cuisine_normalize.output1/2/3`,
`ingredient_standardize.output1/2/3` already existed in `ICOCO.duckdb` (the "gold" result of your research).
I made my notebook reproduce them from source. Running the notebook rebuilds them with `CREATE OR REPLACE`.

### 4.2 One schema detail changed vs the gold
The pre-existing gold tables stored `prompt_tokens` / `completion_tokens` with DuckDB type **`JSON`**
(odd, for plain integers). My notebook stores them as **`BIGINT`** (with `NULL` when absent) — same values,
a more analysis-friendly type. Running the notebook therefore rewrites those two columns' type. If you prefer
to keep the original `JSON` type, say the word and I'll adjust the cast.

### 4.3 Source files for `input_text`
`input_text` is taken from `merged_results.jsonl`. It exists in each folder and always matches the `outputN`
row count. (Note: `odor/reaction_base.jsonl` has only 3 618 lines while `output*` has 3 792 — the base file is
out of sync, which is why I used `merged_results.jsonl` instead of the base files.)

### 4.4 `content_raw` stores the raw (un-stripped) string
I intentionally keep `content_raw` exactly as returned by the model (including any trailing backticks) so the
raw payload is never lost; the parsed arrays live in `value_list` / `country_list` etc.
  and generate the JSON via `nbformat`. Re-compiled (`py_compile`) and re-validated (`nbformat.validate`).