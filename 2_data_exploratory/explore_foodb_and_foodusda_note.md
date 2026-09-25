# FooDB & FoodUSDA — Data Exploratory Note

**Mission**: Data exploratory on the FooDB and FoodUSDA datasets (schema `silver` only) stored in `/workspace/ICOCO.duckdb`. No specific goal — just to understand the data and flag anything worth investigating.

**Notebooks** (both in this folder):
- [`explore_foodb.ipynb`](explore_foodb.ipynb) — FooDB flavor exploration
- [`explore_foodusda.ipynb`](explore_foodusda.ipynb) — FoodUSDA exploration, **latest vs legacy comparison**

**Source notes**: [`../1_silver_create/silver_data_create_note.md`](../1_silver_create/silver_data_create_note.md) / [`../1_silver_create/silver_curation.ipynb`](../1_silver_create/silver_curation.ipynb)

---

## 1. What I did

Two read-only notebooks (SQL-first, `%%sql` magic, no tables created/modified). The table `silver.Zenodo` was **not touched** (per constraint).

| Notebook | Tables explored | What I answered |
|---|---|---|
| `explore_foodb.ipynb` | `silver.FooDB_flavor` | schema, rows, nulls, `category`, `flavor_group` distributions |
| `explore_foodusda.ipynb` | `silver.FoodUSDA_latest_*` + `silver.FoodUSDA_legacy_*` | counts, `data_type`, food categories, nutrient coverage, **side-by-side comparison** |

Both follow the same connection pattern as `silver_curation.ipynb` (close → reconnect to flush WAL) and the same jupysql `%%sql` setup.

---

## 2. Key findings

### 2.1 FooDB — `silver.FooDB_flavor` (883 rows)

| Question | Answer |
|---|---|
| Total descriptors | **883** |
| `name` / `category` completeness | 100% |
| `flavor_group` completeness | only ~13% (770 / 883 are `NULL`) |
| Distinct `category` | **1 — all rows are `odor`** |
| Distinct `flavor_group` | **27** (plus `NULL`) |
| Largest groups | `fruity` (24), `floral` (14), `balsamic` (10), `vegetable` (8), `fatty` (7) |

**Flags**
- ~87% of descriptors have **no `flavor_group`** — any join on that column silently drops most rows.
- Only the `odor` category survives in silver. The bronze `FooDB.flavor` may still hold `taste`/`taste_and_odor` — if so, the silver extract is **lossy** and that should be confirmed before trusting it as "the FooDB flavor table".

### 2.2 FoodUSDA — latest vs legacy

| Dimension | Latest | Legacy |
|---|---|---|
| Foods | **87,990** | 7,793 |
| `data_type` | 5 (`sub_sample_food` 75k, `market_acquisition` 7.6k, `sample_food` 4.1k, `agricultural_acquisition` 810, `foundation_food` 469) | 1 (`sr_legacy_food`) |
| Foods without category | 3,811 (≈4%) | 0 |
| Nutrient definitions | 477 | 474 |
| Nutrients actually used | 235 | 149 |
| food↔nutrient rows | 170,469 | 644,125 |
| **Avg nutrient rows / food** | **~2.2** | **~82.7** |

**Flags**
- **The two versions are complementary, not interchangeable.** `legacy` (SR Legacy) = small, dense, curated reference with essentially complete profiles (~100% of foods have nutrient rows). `latest` = large catalog, mostly branded, with **sparse** nutrients.
- **In `latest`, nutrition coverage is very uneven by `data_type`**: `foundation_food` ~45.8 rows/food, `agricultural_acquisition` ~18.0, but `sub_sample_food` (the 75k majority) only ~1.8, and `market_acquisition`/`sample_food` have **0** rows here.
- **`silver.*_food_nutrient` dropped the `amount`/`unit` values** — it only records incidence (`fdc_id ↔ nutrient_id`). You can tell *which* nutrients a food lists, but **not** *how much*. Any intake/quantity analysis needs the silver schema extended.
- Only **134** nutrient ids are shared between the two versions' nutrition incidence (latest uses 235 total, legacy 149).

---

## 3. Troubles encountered & how I resolved them

- **Forgetting the `%%sql` cell magic.** My first build created the SQL code cells as plain Python cells, so `nbconvert` raised `SyntaxError` on the SQL text. **Resolution**: every SQL cell now starts with the jupysql `%%sql` cell magic (kept via nbformat). Setup cells (line magics) are left as-is.
- **Multiline strings in the build script.** Replacing `code("""…""")` (triple-quoted) with `code("%%sql\n…")` (single-quote style) produced *unterminated string literal* because real newlines landed inside a non-triple-quoted Python string. **Resolution**: always use triple-quoted Python strings for multi-line SQL cells.
- **`COUNT(*)` vs `COUNT(DISTINCT)` after a join.** In the legacy nutrient-coverage cell, `COUNT(*) AS n_food` over a `LEFT JOIN` counted joined rows (644,125) instead of distinct foods (7,793), because `food_nutrient` has many rows per food. **Resolution**: use `COUNT(DISTINCT f.fdc_id)` for food counts.

No DuckDB connection conflicts occurred — the notebooks only **read** and hold a single `%sql` connection.

---

## 4. Reproducibility notes

- **Read-only**: neither notebook creates tables/views/schemas, so re-running does not change the database state.
- Same connection pattern as `silver_curation.ipynb` (close → reconnect to flush WAL), though not strictly required for read-only work.
- Re-run with:
  ```bash
  jupyter nbconvert --to notebook --execute --inplace explore_foodb.ipynb
  jupyter nbconvert --to notebook --execute --inplace explore_foodusda.ipynb
  ```
