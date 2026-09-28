# Confirm the LLM performance numbers in the ICOCO IEEE paper — Mission note

**Date**: 2026-09-27 (UTC)
**Folder**: `/workspace/6_confirm_LLM_performance_number/`
**Mission**: [`missions.md`](missions.md)
**Database**: `/workspace/ICOCO.duckdb`
**Deliverables**:
- [`01_bronze_ingestion.ipynb`](01_bronze_ingestion.ipynb) — bronze layer ingestion (raw inputs)
- [`02_LLM_performance_verification.ipynb`](02_LLM_performance_verification.ipynb) — reproducible SQL verification of all 8 paper numbers
- `data/GBIF_query/gbif_match_cache.json` / `gbif_species_match.jsonl` — cached GBIF species-match results (offline re-runs)

---

## 1. Mission

1. Ingest the raw JSON input `data/GBIF_query/dirty_ingredient_country_mapping.json` into DuckDB (`bronze` schema).
2. Build reproducible SQL notebook(s) that verify the LLM performance numbers reported in the paper:

| Task | SC | IC | AC | Grounding |
|---|---|---|---|---|
| A. Ingredient Standardization | 98.41% | 97.17% | 99.30% | 71.20% (11,765 / 16,524) |
| B. Taxonomic Extraction | 82.34% | 61.36% | 98.74% | 93.29% (2,253 / 2,415) |

---

## 2. What I did (blue = inspection/exploration)

- **Inspected the environment** through terminal commands: confirmed the raw inputs
  (`merged_results.jsonl`, `output{1,2,3}.jsonl`, `dirty_ingredient_country_mapping.json`),
  the existing DuckDB schemas, and the legacy code under `old_code/` (read-only).
- **Studied the legacy pipeline** (`old_code/08_LLM_data_validation/*`, `utils/self_consistency.py`,
  `utils/gold_data_creator.py`, `13_FoodUSDA_queryable/FoodUSDA_queryable.ipynb`,
  `10_geo_vector/GBIF_queryable_rate.ipynb`, `09_gold_data_create/*`) and read the paper PDF
  (`ICOCO IEEE eXPress.pdf`) to pin down the **exact** definitions and denominators.
- **Prototyped the whole verification in pure DuckDB SQL** against the raw files, iterated until
  it reproduced every number, then packaged that logic into the two notebooks.
- **Executed both notebooks** (`jupyter nbconvert --execute`, 0 errors) and confirmed each number
  matches the paper (see §4).

## 3. Troubles encountered & how I resolved them

### 3.1 SC isn't "all 3 identical" — it's "≥ 2/3 identical"
- <font color="red">**Consequence**</font>: implementing the mission's literal SC definition
  ("all 3 lists non-empty and identical after sorting") gave 81.19% / 26.17%, nowhere near
  98.41% / 82.34%.
- <font color="blue">**Action**</font>: read the paper's own sentence — *"Self-Consistency (SC, ≥2/3
  identical outputs [31])"* — and the legacy notebook output, which prints "Tỷ lē khớp Record >=
  2/3 lần: 98.41%".
- <font color="green">**Resolve**</font>: SC = *at least 2 of the 3 runs produce the same canonical
  sorted list* (empty lists count as identical to each other). This reproduces 98.407% → **98.41%**
  and 82.336% → **82.34%** exactly. I followed the paper's numbers over the paraphrase.

### 3.2 DuckDB `trim()` does NOT strip `\n` like Python's `str.strip()`
- <font color="red">**Consequence**</font>: Task B rows whose LLM cell ended with a newline
  (`["..."]\n`) were classified as *unparseable strings*, so SC was 4 low and the gold
  "with-output" count was 5 low.
- <font color="blue">**Action**</font>: row-level diff between the SQL result and the legacy Python
  semantics highlighted exactly those rows (`item-01639`, `item-02015`, …).
- <font color="green">**Resolve**</font>: use `regexp_replace(raw, '^\s+|\s+$', '')` (a real
  whitespace trim) as the parsing gate — that single change fixed Task B to 3,524 / 82.34% and
  2,415 valid.

### 3.3 Empty-string canonicalisation
- <font color="red">**Consequence**</font>: DuckDB `string_to_array('', '')` returns `['']`, while
  Python `sorted('')` returns `[]` → the canonical form of an empty string cell differed
  (`'[""]'` vs `'[]'`), breaking 1 SC row (e.g. `item-00238`).
- <font color="green">**Resolve**</font>: special-case `WHEN len(norm_str)=0 THEN '[]'` (empty
  strings also contribute *no* characters to IC), matching Python exactly.

### 3.4 One extra "queryable" row for Task A
- <font color="red">**Consequence**</font>: grounding query returned 11,766 instead of 11,765,
  because an empty-string candidate matched *every* description (`LIKE '%%'`).
- <font color="green">**Resolve**</font>: skip empty candidates (`e.elem <> ''`), mirroring the
  legacy `if not item: continue`. → 11,765 / 16,524 = **71.20%**.

### 3.5 GBIF API is stateful — keep a cache
- <font color="red">**Consequence**</font>: Task B grounding requires online GBIF species matching
  (~1,581 unique names), which is neither fast nor deterministic on re-runs.
- <font color="green">**Resolve**</font>: the notebook caches results in
  `data/GBIF_query/gbif_match_cache.json` + `gbif_species_match.jsonl`; it only calls the API for
  names *not already cached*, and otherwise runs fully offline (the executed notebook shows
  "cache complete — no API calls made"). Verified: **2,253 / 2,415 = 93.29%**.

---

## 4. Proof of completion (before → after)

**Before** (database): no `bronze.dirty_ingredient_country_mapping`, no `bronze.llm_*`, no
verification views.

**After** (database `/workspace/ICOCO.duckdb`):
- `bronze.dirty_ingredient_country_mapping` — **4,280** rows (from `dirty_ingredient_country_mapping.json`)
- `bronze.llm_ingredient` — **17,017** rows (3-replica ingredient LLM outputs)
- `bronze.llm_scientific` — **4,280** rows (3-replica scientific-name LLM outputs)
- `silver.v_ingredient_runs` / `silver.v_scientific_runs` — parsed replica views
- `silver.v_ingredient_gold` / `silver.v_scientific_gold` — majority-vote (≥2/3) gold views
- `silver.v_ref_food_full` (8,262 = foundation_food latest 469 ∪ legacy 7,793),
  `silver.v_ref_food_foundation` (469)
- `silver.gbif_species_match` — 1,581 cached GBIF matches (1,294 matched)

**Computed vs paper (all ✅):**

| # | Task | Metric | Paper | Computed |
|---|---|---|---|---|
| A1 | Ingredient | SC | 98.41% | 16,746 / 17,017 = **98.41%** |
| A2 | Ingredient | IC | 97.17% | 16,535 / 17,017 = **97.17%** |
| A3 | Ingredient | AC | 99.30% | 16,898 / 17,017 = **99.30%** |
| A4 | Ingredient | Grounding (USDA) | 71.20% | 11,765 / 16,524 = **71.20%** |
| B1 | Taxonomic | SC | 82.34% | 3,524 / 4,280 = **82.34%** |
| B2 | Taxonomic | IC | 61.36% | 2,626 / 4,280 = **61.36%** |
| B3 | Taxonomic | AC | 98.74% | 4,226 / 4,280 = **98.74%** |
| B4 | Taxonomic | Grounding (GBIF) | 93.29% | 2,253 / 2,415 = **93.29%** |

---

## 5. Things for you to review (discrepancies vs the mission text)

1. **The `16,254` in `missions.md` is a typo.** The true denominator is **16,524** (the 16,524
   Task-A inputs whose majority-vote output is non-empty). `11,765 / 16,524 = 71.198 % → 71.20 %`
   — an exact match. `11,765 / 16,254` would print 72.38 % which no computation produces.
2. **"Restrict USDA to `foundation_food`" contradicts 71.20%.** The 11,765 queryable inputs are
   matched against **foundation_food (latest, 469) ∪ FoodUSDA_legacy (7,793) = 8,262 records**
   (the legacy `food_master_data.csv`, rebuilt by `silver.v_ref_food_full`). Restricted to
   `foundation_food` only (469 records) the same inputs give **7,604 (46.02%)**, not 71.20%.
   The notebook computes both so you can see it.
3. **SC definition in the mission doc ≠ paper.** Paper & legacy = "≥ 2/3 identical outputs";
   mission text says "all 3 non-empty identical". Only the former reproduces 98.41%/82.34%.
4. **AC uses "≥ 2/3 of runs empty".** 98.74% is only reached when *2* empty runs + 1 non-empty
   also recover coverage (`empty_cnt >= 2`); "all 3 empty" would give ~94.1% for Task B.
5. **Truncated LLM cells.** 21 (Task A) / 661 (Task B) of the 3-run cells are not valid JSON
   (trailing newlines, dangling quotes, `[""]…` artifacts). The legacy parser treats them as raw
   strings and `sorted()`s their characters inside SC/IC — quirky, but it is exactly what makes
   the printed numbers reproducible. The SQL mirrors it deliberately.

## 6. Temp files

All exploration/prototype/build scripts were kept in `/tmp` and removed on completion
(project convention in `agents.md`). The only persistent derived files are the two GBIF cache
files under `data/GBIF_query/`.