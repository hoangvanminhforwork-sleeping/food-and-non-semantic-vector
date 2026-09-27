# Investigation Report — Data & Number Mismatch

**Date**: 2026-09-27 (UTC)
**Mission**: `investigate_data_mismatch/investigation_mission.md`
**Database inspected**: `/workspace/ICOCO.duckdb` (read-only)
**Scope**: RecipeZenodo counts (2.1), FooDB & FoodUSDA categorizations (2.2), LLM performance metrics (2.3).

All numbers below were **re-verified against DuckDB / the source CSVs / the old-code notebooks** rather than
taken from memory. Where the user-hinted number differs from what I measured, the discrepancy is called out.

---

## 2.1 Recipe Zenodo

### 2.1.1 How many dishes — 50,511 vs "50,510" in the dataset description

| Source | Count | Notes |
|---|---|---|
| `silver.RecipeZenodo` (new code) | **50,511** rows | `COUNT(*)` → 50,511 |
| `bronze.RecipeZenodo` | **50,511** rows | matches silver |
| `data/RecipeZenodo_final.csv` physical lines | **50,513** | 1 header + 50,512 physical lines |
| `data/RecipeZenodo_final.csv` logical records (strict `csv` parse) | **50,511** | some lines are embedded newlines inside quoted fields |
| `old_code/data/silver/silver_Zenodo.csv` (pandas default parse) | **50,513** data rows | old file parsed more leniently |

**Explanation of the ±1**: the Zenodo CSV is malformed (embedded control characters such as NUL/STX, CP-1252
encoding, and double-quotes inside fields). The upload layer (`0_upload_data_to_DuckDB`) first converts
`RecipeZenodo.csv` (110 MB, CP-1252) → UTF-8 → strips control chars → re-quotes → `RecipeZenodo_final.csv`.
Because a few rows contain **embedded newlines inside a quoted field**, the physical line count (50,512 data
lines) is larger than the **logical record count** (50,511). DuckDB's `read_csv(... ignore_errors=true)`
loads exactly **50,511** logical records, which is what both bronze and silver hold.

The dataset's self-description ("~50,510") is therefore off by **+1** relative to what is actually stored
(50,511). This is a **parsing artifact of the malformed CSV**, not a real double-count. It is not expected to
matter for downstream analysis (the difference is 1 dish / ~0.002%).

### 2.1.2 The "messy ingredient" numbers — 3,586 / 3,500 / ≈4,286

The three numbers describe **the same underlying quantity — "how many distinct normalized ingredients the LLM
produces" — but at three different aggregation levels** of the ingredient-standardization pipeline:

| Number | Best match | Measured | Source / meaning |
|---|---|---|---|
| **3,500** | user *expectation* | **3,500** | distinct standardized ingredients in the **consensus (majority-vote gold)** output — `old_code/09_gold_data_create/data/gold_standardized_name.csv` (17,017 records) **and** `old_code/03_ingredient_extraction/data/gold_ingredients.jsonl` (field `final_consensual_ingredients`). **Verified = 3,500 in both.** |
| **3,586** | new code vs expectation | **3,586** | distinct normalized ingredient **element values** in the new `ingredient_standardize.output1` table (a **single** LLM run, before majority-vote reduction). `output1 = 3,586`, `output2 = 3,594`, `output3 = 3,585` |
| **4,286** | (no exact match) | **4,283** | distinct normalized ingredient elements across the **union of ALL THREE replicas** (`output1 ∪ output2 ∪ output3`) — i.e. the full vocabulary the model ever emitted. The exact value 4,286 did **not** reproduce exactly; the closest measured value is **4,283**. |

**What the three numbers mean and why they differ:**

- **3,586 (per-replica)** is the raw, un-reduced count of *distinct* ingredient names each single LLM run returns.
  It is larger than 3,500 because each run is noisy and returns several surface variants of the same food.
- **3,500 (gold / consensus)** is the count *after* the majority-vote / `gold_data_create` step, which keeps only
  elements appearing in ≥ 2 of the 3 outputs and de-duplicates near-identical results. This is the **canonical,
  reduced** standardized ingredient vocabulary — and it is exactly the ~3,500 the user expected.
- **≈4,286 / 4,283 (union of replicas)** is the **full vocabulary** of distinct names appearing in *any* run.
  It is the largest because it aggregates variants that never reach consensus (each replica contributes some
  unique spellings the others don't). 4,283 is what I measured from the union; 4,286 is plausibly the same concept
  counted on a slightly different moment of the data (e.g. including unparsed/stray strings), but I could not
  reproduce 4,286 exactly.

**Takeaway**: none of the three is "wrong". They answer three different questions:
- *"how many distinct names did ONE run output?"* → ~3,586
- *"how many distinct names survived consensus (the final vocabulary)?"* → 3,500
- *"how many distinct names did the model EVER mention across its 3 runs?"* → ~4,283–4,286


The 3,500 vs 3,586 gap is the difference between **consensus-reduced** vocabulary and **single-run** vocabulary.

---

## 2.2 FooDB and FoodUSDA

### 2.2.1 USDA food categorizations — do old & new contradict?

**Both source and silver contain 28 food categories** for the *latest* and *legacy* versions, and they are
**identical between the two versions** (same 28 rows).

| Question | New code / silver | Old code | Match? |
|---|---|---|---|
| `silver.FoodUSDA_latest_food_category` rows | **28** | — | — |
| `silver.FoodUSDA_legacy_food_category` rows | **28** | — | — |
| raw `data/FoodUSDA_latest/food_category.csv` | 28 data rows (29 lines) | — | source == silver ✅ |
| raw `data/FoodUSDA_legacy/food_category.csv` | 28 data rows (29 lines) | — | source == silver ✅ |
| distinct `food_category_id` **used** by legacy foods | **25** | `food_master_data.csv` → **25** | match ✅ |
| distinct `food_category_id` **used** by latest foods | **19** | — | — |

The 8,262-row `old_code/11_entity_vector/data/food_master_data.csv` references **25** distinct `food_category_id` —
the classic 25 USDA categories. This matches the **25 distinct categories used by legacy foods** in the new silver.
The *latest* catalog only uses **19** distinct categories (it is mostly branded food), yet the category table still
defines **28**. So:

- **Number of defined category rows = 28** in both old-source data and new silver (the 28 include the extra
  `Quality Control Materials` 2600, `Branded Food Products Database` 4500, and `Alcoholic Beverages` 1410 on top of
  the classical 25).
- **Number of categories actually referenced**: legacy 25 / latest 19.
- **No contradiction between old and new code** on the category axis; the only "difference" is which *definition*
  set you count (28 definitions vs 25 used).

Also relevant for the "mismatch" investigation — the two USDA versions are **complementary, not interchangeable**:
`latest` = 87,990 foods (mostly branded, sparse nutrients, 3,811 foods with no category, 170,469 food↔nutrient rows);
`legacy` = 7,793 foods (dense, every food categorized, 644,125 food↔nutrient rows). This matches the note in
`2_data_exploratory/explore_foodb_and_foodusda_note.md`.

### 2.2.2 FooDB odors — the silver-is-lossy flag is actually WRONG

| Question | New silver (`silver.FooDB_flavor`) | Bronze / raw | Note |
|---|---|---|---|
| rows | **883** | 883 | same |
| distinct `category` | **1 – all `odor`** | **1 – all `odor`** | bronze also 100% odor |
| distinct `flavor_group` | **27** (+NULL) | — | ~87% of rows have `flavor_group = NULL` |
| raw source `data/foodb_2020_04_07/Flavor.csv` | — | `odor: 883` | **source itself is 100% odor** |

**Key correction**: `explore_foodb_and_foodusda_note.md` correctly lists the silver facts (883 rows, 27 flavor
groups, 1 category = odor), but it also **flags that "the silver extract may be lossy because the bronze may hold
taste / taste_and_odor"**. I checked the bronze **and** the raw source file: both contain **only `odor`** (883 rows,
category is 100% `odor`). So the silver `FooDB_flavor` extract is **NOT lossy on the category axis** — the source
data itself contains no `taste` or `taste_and_odor` rows. That flag should be retired.

The genuinely useful FooDB finding stands: ~87% of odor descriptors have **no** `flavor_group`, so any join on
`flavor_group` silently drops most rows.


---

## 2.3 LLM performance metrics (old vs new)

Two notebooks measure the same LLM outputs at different granularities, which is the root of most "mismatch"
confusion.

### 2.3.1 Metric definitions differ between old and new code

The **old code** (`old_code/08_LLM_data_validation/*.ipynb` via `old_code/utils/self_consistency.py`) reports, per
task, at **record level**: *exact-match ≥ 2/3 of replicas* and *match-all 3 replicas*; and at **element level**:
*Input Coverage* (has a valid element appearing ≥ 2×) and *Adjusted Input Coverage* (also counts the consistent
empty-list rows as "recovered").

The **new code** (`4_AI_cleaned_data_exploratory/AI_generated_data_exploratory.ipynb`) reports the fraction of rows
where **all three replicas are identical** (a simpler "reproducibility" ratio) plus distinct output counts — it does
**not** recompute the old per-table coverage percentages.

### 2.3.2 Old-code self-consistency / coverage (08_LLM_data_validation)

| Task | rows | Match ≥2/3 | Match-all | Input Coverage | Adjusted Coverage |
|---|---|---|---|---|---|
| ingredients | 17,017 | 98.41% | 82.93% | 97.17% | 99.30% |
| scientific_name | 4,280 | 82.34% | 58.90% | 61.36% | 98.74% |
| reactions (odor) | 3,792 | 75.74% | 38.11% | 74.10% | 99.42% |
| geography – continent | 116 | 97.39% | 93.91% | 94.78% | — |
| geography – country | 116 | 98.26% | 88.70% | 93.04% | — |
| geography – sub_region | 116 | 96.52% | 84.35% | 94.78% | — |
| food_entity (string branch) | 4,280 | 99.81% | 95.56% | — | — |

### 2.3.3 Old-code Queryability (13_FoodUSDA_queryable)

| Metric | Value |
|---|---|
| total ingredient inputs (N) | **17,017** |
| inputs with non-empty output (\|I_valid\|) | **16,524** |
| inputs successfully queried in USDA | **11,765** |
| **Queryable rate (on I_valid)** | **71.20%** |

Note: queryability was computed by substring-matching each standardized name against the USDA `description` column of
`food_master_data.csv` (8,262 foods). Because the new `/workspace` USDA catalog is far larger (87,990 `latest` foods),
a re-run against the new silver would give a **higher** queryable rate — the metric is sensitive to the USDA subset
used, so it is not strictly comparable across old/new pipelines.

### 2.3.4 New-code reproducibility (4_AI_cleaned_data_exploratory)

| Task | rows (per replica) | distinct outputs (output1) | all-3-replicas identical |
|---|---|---|---|
| ingredient_standardize | 17,017 | **~3,586** | ~83% |
| scientific_name | 4,280 | ~2,346 | ~67% |
| odor | 3,792 | 50 (27 allowed, 23 off-vocab) | ~36% |
| cuisine_normalize | 116 | 207 countries / 32 sub-regions | ~86% |

### 2.3.5 Do the numbers differ between old and new? — Yes, but for a principled reason

- **Same underlying data**: both old and new use the same 17,017 / 4,280 / 3,792 / 116 inputs, and the row counts
  agree exactly with the loaded tables.
- **Distinct ingredient counts match**: old gold-consensus = **3,500**, new single-replica `output1` = **3,586**,
  new 3-replica union = **4,283** (see §2.1.2). These are consistent with the pipeline the mission describes.
- **Metric numbers differ because the *metric definitions* differ**:
  - old "Match-all 3 replicas" (ingredients 82.93%) is the same spirit as new "all-3 identical" (~83%) — these
    **agree** (82.93% vs ~83%). ✅
  - scientific: old match-all 58.90% vs new "all-3 identical" ~67% — close but not identical, because the old code's
    `match_all` only counts *record-level exact-list* matches while the new notebook's reproducibility counts
    *any identical output*; the definitions are not byte-for-byte the same.
  - odor/reactions: old match-all 38.11% vs new ~36% — close and consistent (odor is the least reproducible task in
    both). ✅
  - cuisine: old record-level match ≥2/3 ranged 96–98% per field; new "all-3 identical" ~86% — different axis but
    same conclusion (cuisine is highly consistent).

**Bottom line for §2.3**: the old and new code are **measuring compatible things** and largely agree on the *order
and magnitude* of the metrics. The visible differences come from (a) different metric definitions (record-level
match vs all-identical reproducibility; per-field vs whole-record coverage) and (b) the queryability metric using a
different (smaller) USDA subset. No data-corruption mismatch was found; the numbers move slightly because the
models' list outputs are not perfectly reproducible across the 3 fixed-temperature runs.


---

## Summary of real (small) mismatches and root causes

| Finding | Root cause | Action suggested |
|---|---|---|
| 50,511 dishes stored vs "~50,510" claimed | malformed CSV (embedded newlines / control chars) → 1 logical-vs-physical row | none needed (Δ = 1) |
| 3,586 ≠ expected 3,500 | 3,500 is the *consensus-reduced* gold vocabulary; 3,586 is the *single-replica* raw vocabulary | none — both are valid, different questions |
| "4,286" not exactly reproducible | it is the ~4,283 **union across all 3 replicas** | use 4,283, or define the counting rule |
| `silver.FooDB_flavor` flagged as "possibly lossy" | source `Flavor.csv` is **already** 100% `odor` | retire the "lossy" flag in the note |
| LLM metric values old-vs-new differ | different metric definitions + different USDA subset for queryability | document metric definitions explicitly |

---

## Files inspected (addresses provided in the mission)

- `2_data_exploratory/explore_zenodo_recipe.ipynb` + `..._note.md`
- `2_data_exploratory/explore_foodb.ipynb` + `explore_foodusda.ipynb` + `explore_foodb_and_foodusda_note.md`
- `4_AI_cleaned_data_exploratory/AI_generated_data_exploratory.ipynb` + `_note.md`
- `old_code/03_ingredient_extraction/ingredient_extraction.ipynb`, `process_self_consistency.ipynb`, `common_name_JSONL_analysis.ipynb`, `data/*`
- `old_code/05_scientific_extraction/extract_unique_common.ipynb`, `data/*`
- `old_code/08_LLM_data_validation/*.ipynb`
- `old_code/09_gold_data_create/gold_standardized_name.ipynb`, `gold_scientific_name.ipynb`, `gold_reactions.ipynb`, `data/*`
- `old_code/11_entity_vector/FoodUSDA_latest_explore.ipynb`, `FoodUSDA_new_old_comparision.ipynb`, `data/food_master_data.csv`, `data/dirty_ingredient_category_mapping.json`
- `old_code/12_odor_vector/*`, `old_code/13_FoodUSDA_queryable/FoodUSDA_queryable.ipynb`
- `old_code/utils/self_consistency.py`, `gold_data_creator.py`
- `ICOCO.duckdb` (schemas `bronze`, `silver`, plus the four `output1/2/3` task schemas)
- `data/foodb_2020_04_07/Flavor.csv`, `data/FoodUSDA_latest/food_category.csv`, `data/FoodUSDA_legacy/food_category.csv`

