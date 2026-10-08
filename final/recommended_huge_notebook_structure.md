# Recommended structure — the single "huge" Colab notebook

**Companion to** `final_mission.md` · **Derived from** `merge_notebook_template.md` · **Paper:** `ICOCO IEEE eXPress.pdf`

> This is a *proposal* (nothing here is written yet). It turns the raw idea in
> `merge_notebook_template.md` into a concrete, top-to-bottom cell layout, the naming
> conventions, the execution conventions, and the decisions to settle before any code is written.

---

## 1. Scope

One notebook, run top-to-bottom on Google Colab, that reproduces the paper **end to end**:
raw datasets → LLM canonicalization + grounding metrics → GBIF → the 304-D non-semantic
representation (`248 geo + 28 entity + 28 odor`) + FoodBERT baseline → K-means "battle"
(external alignment + internal geometry) → figures.

## 2. Design principles

1. **Single file, strictly top-to-bottom.** Everything a cell needs was produced by an earlier cell.
2. **SQL-first** (DuckDB via jupysql); Python only for what SQL cannot do
   (`ast.literal_eval`, the dense `.npy` scatter, model inference, plotting).
3. **Colab paths** (`/content/...`), **not** this container's `/workspace/...`.
4. **Idempotent SQL** everywhere.
5. **Terse markdown** — section titles only (one line at most).
6. Treat expensive/reproducible-at-a-cost products as **inputs from the GitHub release**,
   not as cells that spend money: the 12 LLM output files, the GBIF cache, the FoodBERT `.npy`.
7. Three clearly separated data zones:
   - **Downloaded by code:** Zenodo, FooDB, FoodUSDA.
   - **Uploaded (precomputed):** 12 LLM json files, GBIF cache, FoodBERT `.npy`.
   - **Derived in-notebook (cheap):** final tables, metrics, our 3 vectors, battle, clustering, figures.

## 3. Global conventions

### 3.1 Libraries (install only what Colab lacks)

```python
# Already on Colab -> DO NOT reinstall:
#   numpy, pandas, scipy, matplotlib, seaborn, plotly, scikit-learn, torch, transformers, tqdm
!pip -q install duckdb duckdb-engine jupysql requests
# FoodBERT (Part 3.4 only):
!git clone -q https://github.com/chambliss/foodbert.git
%cd foodbert && !pip -q install -e . && %cd /content
```

### 3.2 DuckDB connection discipline

DuckDB allows only **one** connection. Keep the project pattern (top of the notebook, once):

```python
%load_ext sql
%config SqlMagic.displaylimit = 50
%sql duckdb:////content/ICOCO.duckdb
%sql --close duckdb:////content/ICOCO.duckdb     # flush WAL
%sql duckdb:////content/ICOCO.duckdb             # fresh reconnect
```

Python-side fetch reuses the **same** connection (never a second `duckdb.connect`):

```python
from sql.connection import ConnectionManager
raw = ConnectionManager().current.connection_sqlalchemy.connection
raw.execute("SET memory_limit='4GB'")
raw.execute("SET threads=4")
```

End the notebook with `%sql --close duckdb:////content/ICOCO.duckdb`.

### 3.3 Idempotency

`CREATE SCHEMA IF NOT EXISTS`, `CREATE OR REPLACE TABLE/VIEW`, `INSERT OR REPLACE`.
Never `DROP` a shared object; re-running must not corrupt or duplicate state.

## 4. Data & naming conventions

### 4.1 The uploaded LLM files — resolving the open naming question
`merge_notebook_template.md` §0b proposed names like `ingredient_extraction_output1.json`,
but the new codebase names each LLM task after its **DuckDB schema**. **Recommendation:** name
each file after the schema it populates, so `file = {schema}_output{n}` maps 1:1 to a table:

| Legacy folder (LLM task) | Schema / table prefix | File in `/content/` |
|---|---|---|
| `old_code/03_ingredient_extraction` | `ingredient_standardize` | `ingredient_standardize_output1.json` … `_output3.json` |
| `old_code/04_geography_extraction` | `cuisine_normalize` | `cuisine_normalize_output1.json` … `_output3.json` |
| `old_code/05_scientific_extraction` | `scientific_name` | `scientific_name_output1.json` … `_output3.json` |
| `old_code/06_reactions` | `odor` | `odor_output1.json` … `_output3.json` |

Rule: **`{schema}_output{n}.json`, `n ∈ {1,2,3}`** = the three fixed-temperature LLM replicas.
Pick one extension for the release (`.json` or `.jsonl`) and use it everywhere — the source
files under `data/` are `.jsonl`.

### 4.2 Downloaded datasets
| Dataset | Source | Extracted into `/content/` |
|---|---|---|
| Zenodo recipes | record `15169428`, file `dataset.csv` | `dataset.csv` → preprocessed `zenodo_final.csv` |
| FooDB | `foodb.ca/public/system/downloads/foodb_2020_4_7_csv.tar.gz` | `foodb_2020_04_07/` |
| FoodUSDA latest | `FoodData_Central_foundation_food_csv_2026-04-30.zip` | `foodusda_latest/` |
| FoodUSDA legacy | `FoodData_Central_sr_legacy_food_csv_2018-04.zip` | `foodusda_legacy/` |

### 4.3 DuckDB schemas / tables
- One schema per source domain: `zenodo`, `foodb`, `foodusda_latest`, `foodusda_legacy`.
- One schema per LLM task: `ingredient_standardize`, `cuisine_normalize`, `scientific_name`,
  `odor` — each holding `output1 / output2 / output3` (+ a `gold`/`final` view).
- Derived schemas: `gbif`, `battle`, `result`.

## 5. Pipeline overview

```
download (Zenodo / FooDB / FoodUSDA) ─┐
uploaded LLM json (12 files) ─────────┤
                                      ▼
   PART 0  environment + data + verification
                                      ▼
   PART 1  final tables (zenodo/foodb/foodusda) + LLM outputs
           + input reconstruction + LLM metrics (SC / IC / AC + USDA grounding)
                                      ▼
   PART 2  GBIF query + Task-B queryability + ingredient->country mapping
                                      ▼
   PART 3  vectors: geo(249) / entity(28) / odor(28)+entropy  +  FoodBERT(768)
                                      ▼
   PART 4  battle (labels): gold geography + master_battle + recipe_status
                                      ▼
   PART 5  K-means battle: external (PA / SF1) + internal (S / DB / CH) + stress test
                                      ▼
   PART 6  figures (Fig 3 external, Fig 4 ablation, Fig 5 internal)
```

---

## 6. Section-by-section structure

Terse titles + code cells. "Cell type" is `md` (markdown) or `code`.

### PART 0 — Environment, data & verification
| # | Type | Purpose |
|---|---|---|
| 0.0 | md | `# 0. Environment & data setup` |
| 0.1 | code | **Install** the missing libraries (§3.1). |
| 0.2 | md | `## Verify uploaded LLM data` |
| 0.3 | code | Check the 12 `{schema}_output{n}.json` files exist in `/content`; raise a clear error listing what is missing. (They come from the GitHub release; without them the run cannot continue and regenerating them costs money.) |
| 0.4 | md | `## Download datasets` |
| 0.5 | code | Zenodo: stream-download `dataset.csv`. |
| 0.6 | code | FooDB: download + extract the `tar.gz`. |
| 0.7 | code | FoodUSDA: download + extract both `zip`s (latest, legacy). |
| 0.8 | md | `## Open DuckDB` |
| 0.9 | code | `%load_ext sql` + displaylimit + `%sql` connect. |
| 0.10 | code | `%sql --close` then reconnect (flush WAL). |
| 0.11 | code | `ConnectionManager` helper + `SET memory_limit/threads`. |

### PART 1 — Final tables + LLM outputs + performance
| # | Type | Purpose |
|---|---|---|
| 1.0 | md | `# 1. Load final data & LLM outputs` |
| 1.1 | code | **Preprocess Zenodo** (cp1252 → UTF-8, strip control chars, re-quote) → `zenodo_final.csv`. |
| 1.2 | code | `CREATE OR REPLACE TABLE zenodo AS SELECT * FROM read_csv('.../zenodo_final.csv', delim=';', header=true, ignore_errors=true, null_padding=true, parallel=false)`. |
| 1.3 | code | Parse the list-literal columns (`Category / Cuisine / Ingredients / Cooking Methods`) with `ast.literal_eval` into real `VARCHAR[]`; keep only `Name, Cuisine, Ingredients, Cooking Methods`. |
| 1.4 | code | FooDB → `foodb.flavor` (final columns only). |
| 1.5 | code | FoodUSDA latest & legacy → `foodusda_latest/legacy.food / _nutrient / _food_nutrient / _food_category` (final columns only). |
| 1.6 | md | `## Upload LLM outputs` |
| 1.7 | code | For each task: `CREATE OR REPLACE TABLE {schema}.output{n} AS SELECT ... FROM read_json_auto('{schema}_output{n}.json')` (`, n=1,2,3`). |
| 1.8 | md | `## Verify the LLM input` |
| 1.9 | code | Reproduce the **unique element lists** (ingredients / cuisines / common names / nutrients) from `zenodo` and compare with the union of the outputs' `input_text`. |
| 1.10 | md | `## LLM performance` |
| 1.11 | code | Build the per-task shape `input, output1, output2, output3, final_output` (`final_output` = majority vote ≥ 2/3, matching `old_code/utils/gold_data_creator.py`). |
| 1.12 | code | Self-Consistency / Input Coverage / Adjusted Coverage per task (+ % per criterion). |
| 1.13 | code | Queryability — **Task A** against `foodusda` (foundation food only). |

### PART 2 — GBIF
| # | Type | Purpose |
|---|---|---|
| 2.0 | md | `# 2. GBIF` |
| 2.1 | code | Build the unique scientific-name list from the scientific gold (majority vote ≥ 2/3). |
| 2.2 | code | Query the GBIF Species Matching API (stream to `gbif_match.jsonl`), reusing a cache when present. |
| 2.3 | code | Queryability — **Task B** (GBIF) → target `93.29%`. |
| 2.4 | code | Build `dirty_ingredient_country_mapping` (scientific name → country list) for the geo vector. |

### PART 3 — Vectors
| # | Type | Purpose |
|---|---|---|
| 3.0 | md | `# 3. Vectors` |
| 3.1 | code | **Geography vector** — long-format SQL → dense `(N, 249)` → `recipe_geography_vectors.npy`. |
| 3.2 | code | **Entity vector** — ingredient → 28 USDA categories → `entity_vectors.npy`. |
| 3.3 | code | **Odor vector** — nutrient × reaction → 28 odor dims + Shannon entropy → `recipe_odor_vectors.npy`, `recipe_odor_entropy.npy`. |
| 3.4 | code | **FoodBERT** — 768-D embeddings (mean pooling) → `foodbert_Zenodo.npy` (GPU cell). |

### PART 4 — Battle (labels)
| # | Type | Purpose |
|---|---|---|
| 4.0 | md | `# 4. Battle (labels)` |
| 4.1 | code | `battle.cuisine_gold` = geography gold per raw cuisine tag (majority vote ≥ 2/3). |
| 4.2 | code | `battle.master_battle` = per-dish countries / sub-regions / continents. |
| 4.3 | code | `battle.recipe_status` = per-dish `has_geo_label` / `no_geo_label`. |

### PART 5 — K-means battle
| # | Type | Purpose |
|---|---|---|
| 5.0 | md | `# 5. K-means battle` |
| 5.1 | code | Load the vectors; define the labelled / unlabelled sets from `battle.recipe_status`. |
| 5.2 | code | Loop model variants × K ∈ {6, 31, 207} × 5 seeds → internal metrics (Silhouette, Davies-Bouldin, Calinski-Harabasz). |
| 5.3 | code | Post-hoc Hungarian matching → Partial-Match Accuracy (PA) + Soft-Macro-F1 (SF1). |
| 5.4 | code | **Conservative stress test:** Proposed (mean − sd) vs FoodBERT (mean + sd); write `result` tables. |

### PART 6 — Figures
| # | Type | Purpose |
|---|---|---|
| 6.0 | md | `# 6. Figures` |
| 6.1 | code | **Fig 3** — external alignment (PA, SF1): FoodBERT vs Full model across K. |
| 6.2 | code | **Fig 4** — ablation across K. |
| 6.3 | code | **Fig 5** — internal metrics (full dataset + unlabelled subset). |
| 6.4 | code | LLM validation summary table (SC / IC / AC per task). |

---

## 7. Ordering & dependency notes

- **Part 2 before Part 3:** the geography vector needs the GBIF ingredient→country mapping.
- **Part 2 before the Task-B number:** the final LLM-validation summary (cell 6.4) only *reads*
  results, so it can live at the end even though Task B is computed in Part 2.
- **Part 4 before Part 5:** the battle labels feed the post-hoc evaluation.
- **Row alignment:** every `.npy` vector is in `zenodo` row order; `battle.id = row_index + 1`.
- If the no-label experiment is included, Part 5 runs **two** battles (full dataset + unlabelled subset).

## 8. Open decisions to confirm before coding

1. **File extension** for the release: `.json` vs `.jsonl`.
2. **File naming:** `{schema}_output{n}` (recommended) vs legacy `{task_extraction}_output{n}`.
3. **FoodBERT:** recompute in Colab (GPU) or ship `foodbert_Zenodo.npy` as a release artifact?
4. **GBIF:** live-query in Colab (rate limits) or ship the cache?
5. **`50,510` vs `50,511`:** the paper says 50,510; the new pipeline yields 50,511 (documented CSV artifact). Keep the new count and note it.
6. **Scope of Parts 5–6:** main figures only, or also the ablation + no-label battles + heatmaps?

## 9. Appendix — old ↔ new pipeline map

| Stage | Legacy (`old_code/`) | New code (folders) | New schema / table |
|---|---|---|---|
| Zenodo load | `01_preprocess_Zenodo` | `0_upload_data_to_DuckDB` | `zenodo` |
| Curation | (inline) | `1_silver_create` | (folded into Part 1) |
| LLM outputs | `03/04/05/06_*` | `3_add_AI_standardized_data` | `{task}.output{1,2,3}` |
| LLM metrics | `08_LLM_data_validation`, `09_gold_data_create`, `utils/` | `5_`, `6_confirm_LLM_performance_number` | (views) |
| GBIF | `10_geo_vector` | — | `gbif.*` |
| Vectors | `10/11/12_*_vector` | `7_create_npy_files` | `.npy` |
| Battle | `15_KMean_battle_field` | `8_create_battle` | `battle.*` |
| K-means | `15_KMean_battle_field` | (new) | `result.*` |
| Figures | `16_figure_export`, `17_result_and_discussion_` | (new) | — |

