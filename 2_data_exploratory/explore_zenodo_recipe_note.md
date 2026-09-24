# Zenodo Recipe — Data Exploratory Note

**Mission**: Data exploratory on the Zenodo recipe dataset (table `silver.RecipeZenodo` in `/workspace/ICOCO.duckdb`) — answer 5 counts, SQL-first.

**Notebook**: [`explore_zenodo_recipe.ipynb`](explore_zenodo_recipe.ipynb:1)

**Source notes**: [`../1_silver_create/silver_data_create_note.md`](../1_silver_create/silver_data_create_note.md) / [`../1_silver_create/silver_curation.ipynb`](../1_silver_create/silver_curation.ipynb)

---

## 1. What I did

The table `silver.RecipeZenodo` has 5 columns:

| Column | Type |
|--------|------|
| Name | VARCHAR |
| Category | VARCHAR[] |
| Cuisine | VARCHAR[] |
| Ingredients | VARCHAR[] |
| Cooking Methods | VARCHAR[] |

Since the list-like columns are real DuckDB arrays (`VARCHAR[]`), all counts were answered with pure SQL, using `UNNEST(...) AS t(e)` for element-level aggregations. **No Python data processing** was needed.

**Key interpretation decision** — I checked the data first instead of assuming: the mission note says *“the labels of a dish is a list such as `["America", "Mexico"]`”*. Those values are countries/cuisines and map to the **`Cuisine`** column (which contains values like `American`, `Mexican`, `U.S.`, `Latin American`, …). So **“labels” = the `Cuisine` array**.

Cell flow in the notebook:

| Cell | SQL / action | Answer |
|------|--------------|--------|
| Setup | connect (+ close/reconnect to flush WAL) | — |
| Overview | `DESCRIBE` + 3 sample rows | schema confirmed |
| 1 | `SELECT COUNT(*)` | total dishes |
| — | sample `Cuisine` rows | label column confirmed |
| 2 | `CASE WHEN len(Cuisine)>0` … group | with/without labels |
| 3 | `COUNT(DISTINCT e)` over `UNNEST(Cuisine)` | unique labels |
| 4 | `COUNT(DISTINCT e)` over `UNNEST(Ingredients)` | unique ingredients |
| 5 | `COUNT(DISTINCT e)` over `UNNEST("Cooking Methods")` | unique cooking methods |
| Summary | markdown table + top-8 labels | — |

---

## 2. Answers to the 5 questions

| # | Question | Answer |
|---|----------|--------|
| 1 | How many dishes? | **50,511** (rows in `silver.RecipeZenodo`) |
| 2 | Dishes with label / without label (`Cuisine` non-empty / empty) | **31,585** with / **18,926** without |
| 3 | Unique labels (Cuisine, element level) | **116** |
| 4 | Unique ingredients (element level) | **17,033** |
| 5 | Unique cooking methods (element level) | **22** |

All query outputs were verified in the executed notebook (0 errors).

---

## 3. Nuances / robustness check

- **“Dish with label”** = `len(Cuisine) > 0`. Every row is non-NULL (`cuisine_null = 0`), so `Cuisine IS NULL` is not a factor.
- The discrete counts above compare **exact** element strings. If we normalise whitespace + case (`lower(trim(e))`):
  - Cuisine: **116** (unchanged)
  - Ingredients: **17,033 → 17,014** (19 variants differ only by spacing/case, e.g. `tomato` vs `tomato `)
  - Cooking methods: **22** (unchanged)
- Top labels for context: `American` (17,664), `U.S.` (1,789), `Mexican Inspired` (1,499), `Italian` (1,475), …

---

## 4. Troubles encountered & how I resolved them

- **Pipeline build-script escaping**: my first attempt constructed the notebook from a shell heredoc; the heavy backslash escaping for the double-quoted column name `"Cooking Methods"` produced a Python `SyntaxError`. **Resolution**: build the notebook via `nbformat` from a plain Python script (edited with an editor, not a heredoc), using single-quoted Python strings wherever the SQL contains double quotes. No escaping trouble after that.

No DuckDB connection conflicts occurred (this notebook only **reads**, and holds a single `%sql` connection).

---

## 5. Reproducibility notes

- **Read-only**: the notebook creates no tables, views, or schemas — only `SELECT` statements — so it can be re-run as many times as desired without changing the database state.
- Same connection pattern as `silver_curation.ipynb` (close → reconnect to flush WAL), though not strictly required for read-only work.
- Uses jupysql `%%sql` magic; verify with `jupyter nbconvert --to notebook --execute --inplace explore_zenodo_recipe.ipynb`.
