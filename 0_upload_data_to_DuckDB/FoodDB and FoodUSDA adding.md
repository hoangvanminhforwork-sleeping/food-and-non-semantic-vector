# FoodDB and FoodUSDA Adding Process

A chronological, one-by-one account of how I added the FoodDB and FoodUSDA tables into DuckDB: the initial information I was given, every error/trouble I hit, and what I did next to inspect and resolve each one.

---

## 1. My advantage: the initial information you gave me

Before I started, you provided these pieces of context that guided everything:

| # | Information | How it helped |
|---|-------------|---------------|
| 1 | **Mission doc**: add tables into DuckDB, then write the process here | Told me the goal and the deliverable. |
| 2 | **Reference process**: `/workspace/0_upload_data_to_DuckDB/ZenodoDataset_handling_process.md` | Showed me the expected style/format and the known-good notebook workflow. |
| 3 | **Target notebook**: `/workspace/0_upload_data_to_DuckDB/upload_data.ipynb` | Told me exactly which file to extend. |
| 4 | **Constraint**: prioritize SQL, use Python only if SQL cannot work | Set the technical approach. |
| 5 | **Note**: do not run the Zenodo cells; verify by running the new table-adding cells | Told me how to validate without touching the Zenodo dataset. |
| 6 | **Schema naming rule**: `schema` = name of the database, e.g. schema `FooDB`, table `Flavor` → `FooDB.flavor` | Defined the exact schema/table naming convention. |
| 7 | **Confirmed schema names**: `FooDB`, `FoodUSDA_latest`, `FoodUSDA_legacy` in `/workspace/ICOCO.duckdb` | Removed ambiguity before I started. |

These seven pieces of context were the "advantage" — they told me where to look, what the goal was, and how to name things before I ran a single command.

---

## 2. The errors and troubles I encountered (in order)

### 2.1 `Conversion Error: Could not convert string "7/19/2023" to 'DATE'` (FoodUSDA_latest/food.csv)
The first notebook execution failed on the `FoodUSDA_latest.food` table. DuckDB's CSV auto-detector sampled the first 20,480 rows (all `2019-04-01` style) and typed `publication_date` as `DATE`, but line 55,226 contains `7/19/2023` (MM/DD/YYYY), which cannot be converted to `DATE`.

### 2.2 `TypeError: ExecutePreprocessor.preprocess_cell() takes 4 positional arguments but 5 were given`
My first attempt to run the notebook while skipping the Zenodo cells used a custom preprocessor with the wrong method signature for this nbconvert version.

---

## 3. What I did next to inspect and resolve each trouble

### 3.1 Inspected the environment first
- Confirmed `duckdb 1.1.3`, `jupyter`, and `nbconvert 7.17.1` are installed.
- Listed the data files: all 8 target CSVs exist under `/workspace/data/`.

### 3.2 Pre-tested every CSV with pure SQL `read_csv`
- Ran `SELECT COUNT(*) FROM read_csv('<file>', header=true)` for all 8 files in a throwaway connection.
- Result: **all 8 loaded fine** with default auto-detection (883 / 992 / 87,990 / 477 / 170,469 / 7,793 / 474 / 644,125 rows).
- This proved the data was clean enough for a pure-SQL approach — no Python preprocessing needed.

### 3.3 Extended the notebook with SQL cells
- Tagged the 3 Zenodo cells (`import csv` preprocessing, `CREATE OR REPLACE TABLE bronze.RecipeZenodo`, and its `COUNT(*)`) with the `zenodo` tag so they can be skipped during execution.
- Appended new cells:
  - `CREATE SCHEMA IF NOT EXISTS FooDB; FoodUSDA_latest; FoodUSDA_legacy;`
  - 8 × `CREATE OR REPLACE TABLE <schema>.<table> AS SELECT * FROM read_csv('<file>', header=true);`
  - 1 verification cell: `UNION ALL` of `COUNT(*)` for all 8 tables.

### 3.4 Executed the notebook, skipping Zenodo cells (trouble 2.2)
- Wrote a custom `SkipZenodoEP(ExecutePreprocessor)` that returns tagged cells untouched instead of executing them.
- First run failed with a `TypeError` because this nbconvert version's `preprocess_cell(self, cell, resources, index)` takes 3 args, not 4.
- **Resolution**: fixed the override signature to `preprocess_cell(self, cell, resources, index)`.

### 3.5 Hit the real data problem (trouble 2.1)
- The `FoodUSDA_latest.food` cell failed: `publication_date` was auto-detected as `DATE` from the sample, but later rows use `7/19/2023`.
- **Diagnosis**: DuckDB's default `sample_size = 20480` only scans the first ~20k rows for type detection; the mixed date format appears later in the file.
- **Resolution (pure SQL)**: added `sample_size=-1` to every `read_csv(...)` call so the whole file is scanned for type detection. Verified with a quick test that all 8 files then load cleanly:
  - `FoodUSDA_latest.food.publication_date` → `VARCHAR` (mixed formats preserved as text)
  - `FoodUSDA_legacy.food.publication_date` → `DATE` (uniform format)
- Re-ran the notebook end-to-end with the fixed preprocessor → **all cells pass**.

### 3.6 Verified the results
- Queried `information_schema.schemata` and `information_schema.tables` in `/workspace/ICOCO.duckdb` → all 3 schemas and 8 tables present.
- Ran `SELECT COUNT(*)` per table → matches the CSV row counts exactly.
- The notebook's final verification cell shows the same counts.

---

## 4. Final result

All 8 tables were added to `/workspace/ICOCO.duckdb` using **pure SQL** (no Python preprocessing needed):

| Schema | Table | Source CSV | Rows |
|--------|-------|------------|------|
| `FooDB` | `flavor` | `data/foodb_2020_04_07/Flavor.csv` | 883 |
| `FooDB` | `food` | `data/foodb_2020_04_07/Food.csv` | 992 |
| `FoodUSDA_latest` | `food` | `data/FoodUSDA_latest/food.csv` | 87,990 |
| `FoodUSDA_latest` | `nutrient` | `data/FoodUSDA_latest/nutrient.csv` | 477 |
| `FoodUSDA_latest` | `food_nutrient` | `data/FoodUSDA_latest/food_nutrient.csv` | 170,469 |
| `FoodUSDA_legacy` | `food` | `data/FoodUSDA_legacy/food.csv` | 7,793 |
| `FoodUSDA_legacy` | `nutrient` | `data/FoodUSDA_legacy/nutrient.csv` | 474 |
| `FoodUSDA_legacy` | `food_nutrient` | `data/FoodUSDA_legacy/food_nutrient.csv` | 644,125 |

The notebook at `0_upload_data_to_DuckDB/upload_data.ipynb` runs end-to-end without errors (Zenodo cells skipped as instructed).

---

## 5. Status summary

Adding the schema `FooDB`: `success`
Adding the table `FooDB.flavor`: `success`
Adding the table `FooDB.food`: `success`

Adding the schema `FoodUSDA_latest`: `success`
Adding the table `FoodUSDA_latest.food`: `success`
Adding the table `FoodUSDA_latest.nutrient`: `success`
Adding the table `FoodUSDA_latest.food_nutrient`: `success`

Adding the schema `FoodUSDA_legacy`: `success`
Adding the table `FoodUSDA_legacy.food`: `success`
Adding the table `FoodUSDA_legacy.nutrient`: `success`
Adding the table `FoodUSDA_legacy.food_nutrient`: `success`