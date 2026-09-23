# Silver Data Create Note

**Mission**: Data exploratory with DuckDB and SQL — build the `silver` schema in `/workspace/ICOCO.duckdb` by curating the bronze/raw tables.

**Notebook**: [`silver_curation.ipynb`](silver_curation.ipynb:1)

---

## 1. What I did (the pipeline)

All work follows the recommended approach: **copy bronze → silver first, then process in silver**. Bronze is never modified, so it stays as the recovery layer.

| Step | Action | Silver table(s) |
|------|--------|-----------------|
| 0 | `CREATE SCHEMA IF NOT EXISTS silver` | — |
| 1 | `coutries.csv` → bronze → silver | `silver.countries` (249 rows) |
| 2 | `food_category.csv` (latest & legacy) → bronze → silver | `silver.FoodUSDA_latest_food_category`, `silver.FoodUSDA_legacy_food_category` (28 rows each) |
| 3 | `bronze.RecipeZenodo` → copy → keep 5 columns → parse list columns → extract ingredient names | `silver.RecipeZenodo` (50,511 rows) |
| 4 | `FooDB.flavor` → keep `id, name, flavor_group, category` | `silver.FooDB_flavor` (883 rows) |
| 5 | `FoodUSDA_latest` & `FoodUSDA_legacy` → keep only the needed columns | `silver.FoodUSDA_*_food` / `_nutrient` / `_food_nutrient` |

### 1.1 RecipeZenodo — the only step that needed Python

The columns `Category`, `Cuisine`, `Ingredients`, `Cooking Methods` are stored as **VARCHAR** but actually contain **Python list literals**:

```
Category:  ['Dinner']
Cuisine:   ['American']
Ingredients: [{'ingredient': 'pineapple', 'quantity': '1', 'unit': '', 'misc': '...'}, ...]
Cooking Methods: ['microwave']
```

SQL string functions would be fragile here, so I used Python `ast.literal_eval` (as you suggested):

1. **Category / Cuisine / Cooking Methods** → parsed into real lists → stored as DuckDB `VARCHAR[]`.
2. **Ingredients** → it is a list of dicts; I kept **only the ingredient name** (`d['ingredient']`) and dropped quantity/unit/misc → stored as `VARCHAR[]`.

Resulting schema of `silver.RecipeZenodo`:

| Column | Type |
|--------|------|
| Name | VARCHAR |
| Category | VARCHAR[] |
| Cuisine | VARCHAR[] |
| Ingredients | VARCHAR[] |
| Cooking Methods | VARCHAR[] |

### 1.2 FoodUSDA (both latest & legacy)

| Source table | Kept columns | Silver table |
|---|---|---|
| `food` | `fdc_id, data_type, description, food_category_id` | `silver.FoodUSDA_<version>_food` |
| `nutrient` | `id, name` | `silver.FoodUSDA_<version>_nutrient` |
| `food_nutrient` | `id, fdc_id, nutrient_id` | `silver.FoodUSDA_<version>_food_nutrient` |

---

## 2. Troubles I encountered & how I inspected/resolved them

### 2.1 `Connection Error: Can't open a connection to same database file with a different configuration than existing connections`

**Symptom**: The RecipeZenodo Python cell opened its own `duckdb.connect('/workspace/ICOCO.duckdb')` while the `%sql` magic connection was already open. DuckDB refused.

**Inspection**: I tested opening a second connection in a throwaway script — it failed with the same error. I then inspected jupysql's internals (`sql.connection.ConnectionManager`) to find how to reuse the existing connection.

**Resolution**: DuckDB allows **only one connection** to the database at a time. The Python cell now reuses the `%sql` connection via:

```python
from sql.connection import ConnectionManager
raw = ConnectionManager().current.connection_sqlalchemy.connection
```

No second `duckdb.connect()` is ever made.

### 2.2 `%sql --persist` stringifies list columns back to VARCHAR

**Symptom**: My first attempt to write the processed DataFrame back used `%sql --persist df_zenodo`. It "worked" but the `DESCRIBE` showed `Category: VARCHAR` — the lists had been stringified by `to_sql`.

**Inspection**: Ran `DESCRIBE` on the persisted table and compared with the expected `VARCHAR[]`.

**Resolution**: Instead of `--persist`, I use the raw connection's `register()` + `CREATE OR REPLACE TABLE`:

```python
raw.register('df_zenodo', df)                       # creates a temporary view
raw.execute('CREATE OR REPLACE TABLE silver.RecipeZenodo AS SELECT * FROM df_zenodo')
raw.execute('DROP VIEW IF EXISTS df_zenodo')         # clean up the temp view
```

This preserves the real `VARCHAR[]` list types.

### 2.3 jupysql truncates query results to 10 rows

**Symptom**: The final verification query returned 11 tables but only 10 were displayed (`Truncated to displaylimit of 10`), hiding `silver.countries`.

**Inspection**: Checked the notebook output — the data was all there, only the display was cut.

**Resolution**: Added `%config SqlMagic.displaylimit = 100` in the setup cell so all rows are shown.

### 2.4 `DROP TABLE` vs `DROP VIEW` on the temporary object

**Symptom**: Cleaning up the temporary object created by `register()` with `DROP TABLE` failed with `Existing object ... is of type View, trying to drop type Table`.

**Inspection**: `register()` creates a **view**, not a table.

**Resolution**: Use `DROP VIEW IF EXISTS df_zenodo` for the temporary object.

---

## 3. Final result

All 11 silver tables created and verified (0 errors in the notebook):

| Table | Rows |
|-------|------|
| silver.countries | 249 |
| silver.FoodUSDA_latest_food_category | 28 |
| silver.FoodUSDA_legacy_food_category | 28 |
| silver.RecipeZenodo | 50,511 |
| silver.FooDB_flavor | 883 |
| silver.FoodUSDA_latest_food | 87,990 |
| silver.FoodUSDA_latest_nutrient | 477 |
| silver.FoodUSDA_latest_food_nutrient | 170,469 |
| silver.FoodUSDA_legacy_food | 7,793 |
| silver.FoodUSDA_legacy_nutrient | 474 |
| silver.FoodUSDA_legacy_food_nutrient | 644,125 |

Bronze tables are untouched (e.g. `bronze.RecipeZenodo` still has all 18 original columns).

---

## 4. Reproducibility notes

- Every `CREATE` uses `IF NOT EXISTS` / `CREATE OR REPLACE`, so the notebook can be re-run safely.
- The notebook starts by closing and reconnecting the `%sql` connection to flush the WAL file (same pattern as the upload notebook).
- Only one DuckDB connection is used at any time (DuckDB limitation).