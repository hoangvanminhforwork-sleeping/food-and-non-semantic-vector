# Note — Data exploratory on the AI-cleaned (`output1/2/3`) tables

## 1. Mission

Do a data exploratory on every table named `output{number}` in `/workspace/ICOCO.duckdb`
(the LLM-standardized / "AI-cleaned" layer). Focus on the **content** exchanged between the LLM
**input** and its **output** (not on tokens / cost). Deliverable: an idempotent, SQL-first notebook
in `/workspace/4_AI_cleaned_data_exploratory/`.

## 2. What I did, what troubles I encountered, how I resolved them

### 2.1 First, inventory the 12 tables
<font color="blue">I inspected `ICOCO.duckdb` through duckdb and found 12 tables named
`output1/2/3` = 4 schemas (`scientific_name`, `odor`, `cuisine_normalize`,
`ingredient_standardize`) × 3 replicas of the same model run.</font>
All 12 share one 12-column layout (`custom_id`, `input_text`, `status_code`, `model`, `parsed`,
`value_list`, `country_list`, `sub_region_list`, `continent_list`, `content_raw`,
`prompt_tokens`, `completion_tokens`). Only the **task** decides which columns are filled:
list-tasks use `value_list`; the object-task (`cuisine_normalize`) uses the three `*_list` columns.

### 2.2 I explained the "relationship between the tables"
- Each task holds the **same** `input_text` across the 3 versions (verified 100% identical), so the
  versions are **LLM replicas** — ideal for measuring output reproducibility.
- `scientific_name` and `ingredient_standardize` both draw from recipe ingredients
  (`silver.RecipeZenodo.Ingredients`); `cuisine_normalize` from `silver.RecipeZenodo.Cuisine`;
  `odor` is a **nutrient × reaction** chemistry cross-product, unrelated to recipes.
- Cross-checked provenance with `LIST_CONTAINS` against `silver.RecipeZenodo`.

### 2.3 `UNNEST` returned STRUCT-wrapped strings in duckdb's driver
- <font color="red">Consequence</font>: `SELECT DISTINCT u FROM tbl, UNNEST(value_list) u`
  rendered values as `{'unnest': 'vegetable'}` and `v NOT IN (list)` failed with a
  "STRUCT vs VARCHAR" Binder error.
- <font color="green">Resolve</font>: wrap every `UNNEST` in an inner subquery
  (`SELECT UNNEST(value_list) AS extra_term FROM ...`), then reference that **named** VARCHAR column
  in the outer query. This both renders cleanly and makes `IN / NOT IN` work.

### 2.4 A `status_code = 200` is not reliability
- <font color="red">Consequence</font>: the overview counted "parsed vs unparsed" only from the
  payload JSON — `scientific_name` keeps ~212 rows where the model appended stray bytes
  (e.g. `crab` → `...\"]\n"`), `odor` 4 rows where the model returned **two** candidate answers
  (`[]\n["fatty","nutty"]`), `ingredient` 9 with truncated strings, and `cuisine` 1 over-long list.
- <font color="green">Resolve</font>: I present both `rows` and `parsed`/`unparsed`, and flag that
  the model-formatting artifacts are the real failure mode (not the HTTP status).

### 2.5 Built the notebook programmatically (avoid JSON-escaping pain)
- <font color="red">Consequence</font>: embedding SQL/markdown with many `"` and `\` directly into a
  `.ipynb` JSON is error-prone.
- <font color="green">Resolve</font>: each cell was written to a small file under
  `/tmp/explore_build/cells/`, then assembled with `nbformat` and validated + executed with
  `jupyter nbconvert --execute` (0 error cells).

### 2.6 SQL-idiom fixes discovered during execution
- <font color="red">Consequence</font>: an inventory query that joined the catalog tables
  (`information_schema`) referenced data columns (`parsed`,`model`), which do not exist there.
- <font color="green">Resolve</font>: rewrote it as a straightforward `UNION ALL` over the 12 tables.
- A `... ORDER BY ... LIMIT .. UNION ALL ...` failed ("syntax error at or near UNION") and a
  leading-parenthesis `(SELECT ..) UNION` rendered an **empty** result through jupysql.
- <font color="green">Resolve</font>: replaced the provenance with `COUNT(DISTINCT ...)` (the naive
  join inflated counts, e.g. cuisine 116 → 35 804), and samples with a `ROW_NUMBER() OVER (PARTITION
  BY grp ORDER BY custom_id)` top-N-per-group query that starts with a plain `SELECT`.

## 3. Can I prove the task is done?

- Created **`AI_generated_data_exploratory.ipynb`** (31 cells, SQL-first, idempotent, follows the
  project connect → close → reconnect pattern).
- Executed end-to-end with `jupyter nbconvert --execute` → **0 error cells**. Every query output
  was verified against direct duckdb reads.
- Confirmed numbers, e.g.: 4 280 / 3 792 / 116 / 17 017 rows; provenance 74% / – / 100% / 99.9%;
  odour vocabulary **27 allowed vs 50 used (23 off-vocab)**; cross-replica repro **67% / 36% / 83% /
  86%**; cuisine 207 countries yet only 166 match `silver.countries`.
- No `ICOCO.duckdb.wal`, lock, or temporary files left in the workspace; the DB was only read.

## 4. Things you may want to review

- **Odour vocabulary drift**: the model was told a 27-term lexicon but emitted **50** (23 off-vocab,
  led by `caramel`, `sweet`, `buttery`, `roasted`). Decide keep-vs-map before a strict consumer.
- **Cuisine countries are colloquial, not ISO**: only 166 of 207 country strings match
  `silver.countries` (e.g. "United States", "Ivory Coast", "Russia", "Latin America").
- **`odor` is the least reproducible** (~36% identical across 3 replicas); majority-voting across
  replicas may be worth considering.
- **`n_continents = 7`** in the notebook includes the empty-string `''` as a distinct value — the 6
  real continents are Europe/Asia/North America/Africa/South America/Oceania.
- Parse failures are formatting artifacts, not "no answer" — the raw payload is preserved in
  `content_raw` so nothing is lost.