# Task Note — Fig. 4 "all ablations = Full Fusion" data/query bug

**Date:** 2026-10-08 (UTC)
**Mission:** `final/query_data_bug.md`
**Notebooks touched:** `final/ICOCO_notebook_Colab.ipynb` (5 cells) and `final/ICOCO_notebook.ipynb` (6 cells, kept in parity)
**Database (read-only checks):** `/workspace/ICOCO.duckdb`
**Legacy reference:** `old_code/{10_geo_vector,11_entity_vector,12_odor_vector,13_FoodUSDA_queryable,15_KMean_battle_field}`

---

## 1. Symptom (from the mission)

> Cell **6.2 "External cultural alignment — ablation across all variants (paper Fig. 4)"**: *all ablations
> appear in the same number = Full Fusion*. Two hypotheses were offered:
> **(H1)** the vectors are generated incorrectly (`7_create_npy_files` vs `old_code/10-12`);
> **(H2)** querying/matching is incorrect — the cells in parts **1.8** (gold counts) and **1.9** (USDA
> queryability); *"the output is not right"* (`old_code/13_FoodUSDA_queryable`, `5_confirm_methodology_number`,
> `investigate_data_mismatch`).

## 2. Verdict (short)

| Hypothesis | Verdict | Evidence |
|---|---|---|
| **H1 — vectors wrong** | ❌ **not the cause** | The new vectors are a faithful reproduction of the legacy ones (identical row-for-row after the 2 dropped recipes) — §5 |
| **H2 — 1.8/1.9 matching wrong** | ❌ **not the cause of the figure**, but **1 real defect** was found and fixed (a tautological count cell) — §6 |
| **Real cause** | ✅ **the cluster→label assignment rule in cell 5.3 is *degenerate*** | §3, §4 — `EXT_ASSIGN='legacy'` makes every proposed variant score the **majority-class base rate** |

The figure was not "wrong data" — it was **a wrong metric**: with the per-cluster-majority rule, *every*
variant predicts the same label for every cluster, so all 6 proposed bars printed the identical number that
`labeled.csv` shows for `Our_Full_Fusion` (PA 71.74 ≈ 71.76, SF1 0.1393).

---

## 3. Root cause — the per-cluster-majority assignment is degenerate

Cell 5.3 maps each cluster to a ground-truth label with `_assign(..., use_bipartite)`. Two rules exist:

* `use_bipartite=True` — Hungarian bipartite matching (what the paper *describes*, Sec. IV-C / Eq. 7);
* `use_bipartite=False` — **per-cluster majority** (`M[cluster].argmax()`): the cluster's modal label.

`EXT_ASSIGN='legacy'` (the shipped default) used **majority for the 6 proposed models** and bijection only
for FoodBERT. The majority rule is degenerate because the ground-truth labels are extremely imbalanced:

| granularity | modal ground-truth label | share of the 30,621 labelled recipes |
|---|---|---|
| continents (K=6) | North America | **71.76 %** |
| sub-regions (K=31) | Northern America | **63.65 %** |
| countries (K=207) | United States of America | **61.78 %** |

At K=6 the modal continent of **every** cluster is North America, so the majority rule predicts "North
America" for *every* recipe *whatever the clustering is* — the score is then exactly the base rate
(PA 71.76, SF1 0.1393) for all six variants. The same collapse happens (slightly less completely) at
K=31 / K=207, where all variants land within ~0.5–0.8 pp of each other. FoodBERT looked different only
because it alone was scored with the bijection.

**This reproduces the symptom exactly** — full local reproduction (sklearn `KMeans(n_init=3, seed=42)`,
new vectors + DB labels, `PA % / Soft-MacroF1`):

| model | K=6 majority | K=6 bipartite | K=31 majority | K=31 bipartite | K=207 majority | K=207 bipartite |
|---|---|---|---|---|---|---|
| FoodBERT_Baseline | 71.76 / 0.1393 | 30.07 / 0.1629 | 63.65 / 0.0251 | 8.76 / 0.0412 | 61.78 / 0.0037 | 5.33 / 0.0176 |
| **Our_Full_Fusion** | **71.76 / 0.1393** | 33.47 / 0.1671 | 63.65 / 0.0251 | 10.42 / 0.0465 | 62.13 / 0.0058 | 6.42 / 0.0194 |
| Ablation_No_Entropy | **71.76 / 0.1393** | 29.19 / 0.1539 | 63.94 / 0.0313 | 10.07 / 0.0448 | 62.01 / 0.0052 | 6.39 / 0.0195 |
| Ablation_Geo_Only | **71.76 / 0.1393** | 40.27 / 0.1777 | 64.09 / 0.0384 | 26.63 / 0.0539 | 62.59 / 0.0080 | 26.94 / 0.0148 |
| Ablation_Entity_Only | **71.76 / 0.1393** | 26.11 / 0.1526 | 63.65 / 0.0251 | 10.72 / 0.0482 | 61.81 / 0.0040 | 6.50 / 0.0193 |
| Ablation_Odor_Only | **71.76 / 0.1393** | 32.84 / 0.1636 | 63.65 / 0.0251 | 20.56 / 0.0451 | 61.88 / 0.0042 | 20.23 / 0.0155 |
| Ablation_Odor+Entropy | **71.76 / 0.1393** | 38.68 / 0.1655 | 63.67 / 0.0257 | 22.19 / 0.0465 | 61.90 / 0.0041 | 20.25 / 0.0155 |

Spread of the 7 variants at each K: **majority = 0.00 / 0.44 / 0.81 pp** (degenerate) vs
**bipartite = 14.16 / 19.22 / 22.33 pp** (informative). The FoodBERT row uses the **row-aligned**
legacy embedding — see §7.

---

* Reproduce: the complete recipe (SQL + Python) is in §11.

## 4. Why the *paper's* Fig. 4 has the same collapse (context)

The un-reproducibility was already documented in
`final/investigate_report_mismatch_and_modify_chart_note.md` and `final/paper_audit.md`: the legacy
`labeled.csv` itself shows PA **71.74 / 71.74 / 71.68** for `Our_Full_Fusion` / `Ablation_No_Entropy` /
`Ablation_Geo_Only` at K=6 (all three ≈ the majority-class base rate 71.76), while the other three variants
sit at 54.93–57.85. So the previous agent kept the degenerate rule *on purpose*
(`EXT_ASSIGN='legacy'`) to reproduce the published figure, and `FIG3_SOURCE` was later flipped to
`'computed'` — the combination the user then read as "all ablations = Full Fusion".

---

## 5. H1 — the vectors are **not** wrong (checked)

The three curated vectors were produced twice: legacy (`old_code/10_geo_vector`,
`old_code/11_entity_vector`, `old_code/12_odor_vector` → collected in
`old_code/15_KMean_battle_field/all_vector/`) and new (`7_create_npy_files/`).
Comparison with `np.load` (exact equality + max abs diff):

| pair | legacy shape | new shape | rows identical | note |
|---|---|---|---|---|
| `entity_vectors.npy` | (50513, 28) | (50511, 28) | **50,504 / 50,511** | 7 rows differ by float noise |
| `recipe_odor_vectors.npy` | (50513, 27) | (50511, 27) | **50,509 / 50,511** | 2 rows differ |
| `recipe_odor_entropy.npy` | (50513,) | (50511,) | **50,509 / 50,511** | 2 rows differ |
| `recipe_geography_vectors.npy` | (50513, 248) | (50511, 249) | **50,288 / 50,511** | after removing the extra country column; mean abs diff 3.9e-6 |
| `foodbert_Zenodo.npy` | (50513, 768) | (50513, 768) | **identical** (both copies) | reused artifact |

Two structural differences, both understood:

1. **2 recipes dropped by the curation** — the legacy artifacts have 50,513 rows, the curated data 50,511.
   The dropped legacy rows are **21475 and 23805** (0-based). Verified independently of the vectors: after
   applying that mapping, the `final_continents` labels and the `has_geo_label` / `no_geo_label` split
   (`battle.recipe_status`) match `old_code/.../data/MasterBattle.csv` and
   `battle/indices_has_cuisine.txt` **row for row (0 mismatches)**.
2. **one extra geography column** — the new vector has 249 columns vs the legacy 248: `coutries.csv` gained
   the "Namibia" row (the known NaN-code row, index 152). Removing column 152 realigns the matrices; the
   remaining differences are tiny float noise in the share denominators.

`checking_data_readiness_note.md` additionally shows the notebook's *regenerated* vectors equal
`7_create_npy_files/` **bit-for-bit** (max|diff| = 0). ⇒ the vector stage is faithful; H1 is ruled out.

---

## 6. H2 — the 1.8 / 1.9 cells (checked, one real defect fixed)

### 6.1 The 1.9 queryability queries are correct — and reproduce the paper

The matching rule in the notebook is the legacy one (`old_code/13_FoodUSDA_queryable`): an ingredient is
queryable iff its lower-cased, trimmed text occurs as a **substring** of some lower-cased USDA
`description`; empty strings are skipped; a recipe is queryable iff ≥ 1 of its voted ingredients is.
Both notebook cells re-run against the database:

| reference view | rows | queryable | with_output | % |
|---|---|---|---|---|
| `v_ref_food_full` (foundation_food latest 469 + legacy 7,793) | 8,262 | **11,765** | 16,524 | **71.20 %** = the paper |
| `v_ref_food_foundation` (foundation_food only) | 469 | 7,604 | 16,524 | 46.02 % = "mission-strict" variant |

Neither output is therefore "wrong": the paper's 71.20 % (`6_confirm_LLM_performance_number`) uses the
**full** reference universe, not the foundation-only one (`missions.md` asked to "restrict to
`foundation_food`" — that instruction and the published number disagree; already flagged in
`6_confirm_LLM_performance_number/confirm_note.md` §5). The problem was *readability*: the two cells look
contradictory unless one knows which universe the paper used — the 1.9 markdown now states both numbers.

### 6.2 Real defect found: the 1.8 count cell was a tautology

`metrics.v_ingredient_gold` keeps only rows that reached a ≥2/3 consensus (`WHERE cnt >= 2`), so
**every row in it has a non-empty `final_list`**. The old count cell

```sql
SELECT 'Task A' AS task, count(*) AS n_inputs,
       sum(CASE WHEN len(final_list) > 0 THEN 1 ELSE 0 END) AS with_output
FROM metrics.v_ingredient_gold
```

necessarily printed `n_inputs == with_output` (16,524 / 16,524), and its `n_inputs` was **not** the number of
LLM inputs. That is exactly the "suspicious" cell the mission pointed at. It now reads the true
denominators from the `*_runs` tables:

| task | n_inputs (runs) | with_output (gold) |
|---|---|---|
| Task A (ingredients) | **17,017** | **16,524** |
| Task B (scientific) | **4,280** | **2,415** |

(16,524 / 2,415 are the paper's grounding denominators; the 11-row gap to IC = 16,535 comes from the
`parsed_kind='list'` parse gate, documented in `6_confirm_LLM_performance_number/confirm_note.md`.)

### 6.3 These sections cannot influence cell 6.2

The ablation metric takes its labels from `battle.recipe_status` / `battle.master_battle` and its features
from the `.npy` vectors — never from the USDA reference views. The queryability sections can therefore
print any number without changing Fig. 4, which is why H2 cannot be the cause of the symptom.

---

## 7. Second real bug found (not on the user's list): FoodBERT row alignment

While checking H1 I found that the **legacy FoodBERT artifact is not row-aligned** with the curated data:

* legacy FoodBERT / legacy vectors / legacy labels: **50,513** rows (order before the curation);
* curated vectors + `battle.*` labels: **50,511** rows.

Cell 5.1 did `players['FoodBERT_Baseline'] = foodbert` and then indexed it with
`lab_rows = (lab_ids - 1)` — no check. With a 50,513-row FoodBERT, every row ≥ 21475 is therefore off by 1
and every row ≥ 23804 by 2, i.e. ~95 % of the 30,621 labelled rows were scored against the wrong recipe.
The Colab path rebuilds FoodBERT from `zenodo_final.csv` (50,511 rows) and is fine; the **local** notebook
reuses the legacy file (`old_code/15_KMean_battle_field/all_vector/foodbert_Zenodo.npy`), so it was
affected. Measured effect (Hungarian bijection, all three granularities):

| K / granularity | `fb[:50511]` (legacy order, **misaligned**) | `np.delete(fb, [21475, 23805])` (**aligned**) | paper / audit reference |
|---|---|---|---|
| 6 / continents | 28.73 / 0.1516 | **30.07 / 0.1629** | 30.01 / 0.1731 (audit bijection: 30.07) |
| 31 / sub-regions | 7.41 / 0.0346 | **8.76 / 0.0412** | 9.42 / 0.0479 (audit bijection: 8.76) |
| 207 / countries | 4.61 / 0.0163 | **5.33 / 0.0176** | 5.94 / 0.0539 |

The aligned values reproduce the audit's legacy-bijection numbers exactly (30.07 and 8.76), which confirms the
mapping; the misaligned slice is 0.7–1.4 pp lower at every K because ~95 % of the labelled rows were paired
with the wrong recipe. Both notebooks now realign defensively (only when `rows == n_recipes + 2`) and assert
that all five feature matrices share the same row count.

---

## 8. Exact changes

### 8.1 `final/ICOCO_notebook_Colab.ipynb` (5 cells)

| cell | section | change |
|---|---|---|
| **2** | config | `EXT_ASSIGN = 'legacy'` → **`'bipartite'`**, with a comment explaining that `argmax`/`legacy` are degenerate (all six proposed variants collapse onto the majority-class base rate) |
| **42** | 1.8 | count cell now reads `n_inputs` from `ingredient_standardize.runs` / `scientific_name.runs` (17,017 / 4,280) instead of from the gold views (tautology) |
| **44** | 1.9 (markdown) | states the matching rule and **both** reference numbers (full = 71.20 % = paper; foundation-only = 46.02 %) |
| **108** | 5.1 | **row-alignment guard**: realigns a 50,513-row legacy FoodBERT (drops rows 21475/23805), asserts all matrices have `n_recipes` rows |
| **112** | 5.3 | prints the **majority-class baseline** per granularity and **warns** when the selected rule is degenerate (variant spread < 0.5 pp) |

### 8.2 `final/ICOCO_notebook.ipynb` (6 cells, kept in parity)

| cell | section | change |
|---|---|---|
| **2** | config | `EXT_ASSIGN` → `'bipartite'` (same rationale; `RUN_KMEANS=False` here, so the stored code is for parity) |
| **41** | 1.8 | same count-cell fix |
| **43** | 1.9 (markdown) | same reference numbers |
| **84** | 3.4 | the FoodBERT artifact copied into `final/data/` (the release payload) is now **row-aligned** (legacy rows 21475/23805 dropped) before `np.save` |
| **107** | 5.1 | same row-alignment guard |
| **111** | 5.3 | same majority-class baseline + degeneracy warning |

Nothing else was touched: cell count, `metadata`, cell ids, outputs and every other source are byte-identical
to before (verified cell-by-cell against the backups).

---

## 9. Validation (before → after)

| check | result |
|---|---|
| JSON parse + `nbformat.validate` (both notebooks) | ✅ OK |
| Cell count / cell ids / outputs / metadata | ✅ unchanged (125 Colab, 124 local) |
| Cell-by-cell diff vs the backups | ✅ **only** cells `{2,42,44,108,112}` (Colab) and `{2,41,43,84,107,111}` (local) changed |
| Python syntax of every touched code cell (`ast.parse`, magics stripped) | ✅ 0 errors |
| New 1.8 count SQL executed on the real DB | ✅ Task A 17,017 → 16,524 · Task B 4,280 → 2,415 |
| New 5.3 block executed on the real labels + reproduced `ext` table | ✅ baseline = North America **71.76** / Northern America **63.65** / United States of America **61.78** |
| Degeneracy warning fires only for the degenerate rule | ✅ majority: spread 0.00 / 0.44 / 0.81 pp → warns at K=6, 31; bipartite: 14.16 / 19.22 / 22.33 pp → no warning |
| Row-alignment guard | ✅ label/split identity 0 mismatches; FoodBERT PA 28.73 → **30.07** (K=6), 7.41 → **8.76** (K=31), 4.61 → **5.33** (K=207) |

*(One false positive in my own syntax checker: Colab cell 85 — the `if not os.path.isdir(...): !git clone …`
line — is unchanged and flags only because the checker strips `!` lines and leaves an empty `if` body.)*

---

## 10. What the figure shows now, and what you should decide

1. **Default is now honest**: `FIG3_SOURCE='computed'` + `EXT_ASSIGN='bipartite'` ⇒ Fig. 6.2 (paper Fig. 4)
   shows seven **different** bars per K, and cell 5.3 prints the majority-class baseline next to them.
2. **The published figure is still reachable** by setting `FIG3_SOURCE='paper'` (plots the 7 `mean ± SD` rows
   of `labeled.csv` under the conservative stress test). That path is unchanged — but note that the paper's
   own K=6 rows *are* the degenerate pattern (71.74 / 71.74 / 71.68), so the two settings cannot agree.
3. **Which metric goes into the paper?** This is a scientific decision, not a code one: the run in §3 shows
   that under the metric the paper *describes* (Hungarian bijection, Eq. 7) `Ablation_Geo_Only` is the best
   external variant (40.27 at K=6, 26.63 at K=31, 26.94 at K=207) and `Our_Full_Fusion` is mid-field — i.e.
   the "fusion wins" claim is not supported; the paper's Fig. 4 shape is only obtained with the degenerate
   majority rule. (Same conclusion as `final/paper_audit.md` §10–11.)
4. **FoodBERT rebuild (Colab).** Cell 3.4 builds FoodBERT from `Ingredients_Raw + Instructions`
   (50,511 rows, aligned). The legacy artifact used the same two raw columns but on the 50,513-row table, so
   the rebuilt embeddings should be compared once against the aligned legacy ones (max abs diff per row) if
   you want the Colab numbers to sit on the same scale as `labeled.csv`.
5. **Release payload.** `final/data/` currently holds **no `.npy`** files (only the archives, merged LLM
   jsonl files, mappings and the GBIF cache), while cell 0.2b still tries to fetch the four vectors and
   `REGENERATE_VECTORS=True` overwrites them anyway. Either upload the vectors to the release or drop them
   from `ARTIFACTS_TO_FETCH` — right now the fetch failures are silently tolerated.
6. **Local notebook parity.** `RUN_KMEANS=False` locally, so the fixed 5.1/5.3 code is stored but not
   exercised there; the FoodBERT realignment in cell 3.4 (which writes the release payload) *is* exercised.



---

## 11. Reproduce my numbers

Everything below is read-only against `/workspace/ICOCO.duckdb` and the on-disk artifacts.
(The same queries live in the notebooks: the notebook's `metrics.*` views are the `silver.*` views of this
DB — they are rebuilt identically by notebook cell 1.6/1.8.)

**(a) the "all ablations = Full Fusion" reproduction** — 3 steps:

1. labels + `has_geo_label` split:

```sql
SELECT s.id, s.status, m.final_country_list, m.final_sub_regions, m.final_continents
FROM battle.recipe_status s JOIN battle.master_battle m USING (id) ORDER BY s.id;   -- 50,511 rows, 30,621 has_geo_label
```

2. features: `geo = np.load('7_create_npy_files/recipe_geography_vectors.npy')` (50,511 × 249),
   `entity` (× 28), `odor` (× 27), `entropy` (× 1), `foodbert = np.delete(np.load(
   'old_code/15_KMean_battle_field/all_vector/foodbert_Zenodo.npy'), [21475, 23805], axis=0)` (× 768).
   `players = {'Our_Full_Fusion': hstack([geo, entity, odor, entropy]), 'Ablation_No_Entropy':
   hstack([geo, entity, odor]), 'Ablation_Geo_Only': geo, 'Ablation_Entity_Only': entity,
   'Ablation_Odor_Only': odor, 'Ablation_Odor+Entropy': hstack([odor, entropy]), 'FoodBERT_Baseline':
   foodbert}`.

3. for each `(K, granularity)` in `(6, continents) / (31, sub-regions) / (207, countries)`: cluster
   `X[lab_rows]` (`lab_rows = status[id == 'has_geo_label'].id - 1`) with `KMeans(n_clusters=K, n_init=3,
   random_state=42)`, then apply `_assign(labels, truth, use_bipartite)` from cell 5.3 and score PA / macro-F1
   exactly as the cell does. Majority rule ⇒ 71.76 / 0.1393 for every model at K=6; bijection ⇒ the values in §3.

**(b) the 1.9 queryability numbers** (uses the notebook's own view names):

```sql
WITH valid AS (
    SELECT custom_id, final_list FROM metrics.v_ingredient_gold WHERE len(final_list) > 0
), flagged AS (
    SELECT v.custom_id,
           max(CASE WHEN e.elem <> ''
                     AND EXISTS (SELECT 1 FROM metrics.v_ref_food_full r
                                 WHERE r.lower_desc LIKE '%' || e.elem || '%')
                    THEN 1 ELSE 0 END) AS is_queryable
    FROM valid v CROSS JOIN UNNEST(v.final_list) AS e(elem)
    GROUP BY v.custom_id
)
SELECT count(*) AS with_output, sum(is_queryable) AS queryable,
       round(100.0 * sum(is_queryable) / count(*), 2) AS queryable_pct
FROM flagged;   -- 16,524 / 11,765 / 71.20 ; swap v_ref_food_full -> v_ref_food_foundation for 46.02
```

**(c) the label/row-alignment identity check** that fixes the dropped rows:

```python
legacy = pd.read_csv('old_code/15_KMean_battle_field/data/MasterBattle.csv')     # 50,513 rows
off = np.where(np.arange(50511) < 21475, 0, np.where(np.arange(50511) < 23804, 1, 2))
# legacy row (i + off[i]) must equal curated row i, for the continents column and for the
# has_geo_label split (old_code/15_KMean_battle_field/battle/indices_has_cuisine.txt): 0 mismatches
```

---

## 12. Temp files

All scratch work (`/tmp/qdb/exp_ext.py`, `exp_align.py`, `exp_fb.py`, `patch_nb.py`, `verify_nb.py`,
`test_snippets.py`, the notebook backups and the CSV/log outputs) lived in `/tmp` and was removed on
completion — nothing temporary was left inside the workspace.

