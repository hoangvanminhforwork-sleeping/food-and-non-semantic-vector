# Paper Forensic Audit

**Paper:** *Reducing Generative AI Bias in Food Recognition Using a Multi-Dimensional Culinary Representation Framework* (`ICOCO IEEE eXPress.pdf`)
**Audit date:** 2026-10-07 (UTC)
**Evidence scope:** raw data, `old_code/`, current notebooks, `labeled.csv`, clustering outputs, metric implementations, figures.
**Nature of task:** forensic audit. **No manuscript / notebook / figure / data was modified.** This is a factual assessment, not a repair.

---

## Executive Summary

The paper's **internal geometric claims are reproducible**; its **external cultural-alignment claims are not reproducible from the available data and code with any single coherent metric**, and several are contradicted by a correct metric.

**Reproducible (independently recomputed here):**

* **FoodBERT's external rows** are reproduced by the **Hungarian bipartite-matching** metric the paper describes, for **every** quantity - Partial-Match Accuracy (PA), Soft-MacroF1 (SF1, Eq. 8), NMI and ARI - at all three granularities (K = 6, 31, 207).
* **All internal metrics** (Silhouette S, Davies-Bouldin DB, Calinski-Harabasz CH) in `all_and_unlabel.csv`, on both the Full Dataset and the Unlabeled Subset, reproduce within seed noise.
* The qualitative **direction** of a few external claims survives a correct metric.

**NOT reproducible / contradicted:**

* **No single cluster->label rule reproduces `labeled.csv`.** The proposed models' continents rows require **per-cluster-majority** (which equals the **majority-class baseline**, PA = 71.76), while the sub-regions/countries rows match **neither** bijection **nor** majority.
* **Country-level "nearly double SF1"**: `labeled.csv` Full = 0.0951 vs FoodBERT = 0.0539 (x1.76); under one coherent metric the ratio is **0.0618 / 0.0542 = x1.14**.
* **"Every proposed variant surpasses FoodBERT in PA"** is **false** under a coherent metric (`Ablation_Entity_Only` 26.07 and `Ablation_No_Entropy` 29.18 < FoodBERT 30.07 at K = 6).
* **"Physical grounding becomes increasingly advantageous as macro-labels fragment"** is **contradicted**: the Full-FoodBERT PA gap **shrinks** with granularity (+3.4 -> +1.45 -> +1.14).
* **Within one model, `labeled.csv` PA and SF1 are mutually inconsistent** (e.g. `Entity_Only` K=6: PA 57.85 matches neither rule, SF1 0.1667 matches bijection).

**Historical pipeline:** the notebook that *generated* `labeled.csv` / `kmeans_benchmark_summary_results.csv` is **absent** from the repository; the surviving battle notebook computes **only** internal metrics. The exact historical protocol **cannot be established from the available evidence**.

**Academic integrity:** consistent with a **methodological / reproducibility failure**, not with established intent. **The available repository evidence is insufficient to establish intentional manipulation.**

---

## 1. Scope and Materials Audited

| Artifact | Path | Role in audit |
|---|---|---|
| Manuscript | `ICOCO IEEE eXPress.pdf` (6 pp) | claims; Fig. 3 external, Fig. 4 ablation-external, Fig. 5 internal; Sec. IV-C Eq. 7/8 |
| Paper's external source | `old_code/15_KMean_battle_field/result/labeled.csv` | values plotted in Fig. 3/4 |
| Paper's internal source | `old_code/15_KMean_battle_field/result/all_and_unlabel.csv` | values plotted in Fig. 5 |
| Figure redraw | `old_code/17_result_and_discussion_/Redraw_all_figures.ipynb` | conservative stress-test transform -> figures |
| Legacy vectors | `old_code/15_KMean_battle_field/all_vector/{recipe_geography_vectors,entity_vectors,recipe_odor_vectors,recipe_odor_entropy,foodbert_Zenodo}.npy` | features, `(50513, ...)` |
| Legacy labels | `data/MasterBattle.csv`; `battle/indices_has_cuisine.txt`; `battle/indices_no_cuisine.txt` | ground truth + split (30,621 / 19,892) |
| Legacy battle (internal) | `old_code/15_KMean_battle_field/no_label_battle_and_fulldata_battle.ipynb` | internal metrics only |
| Legacy vector encoders | `old_code/10_geo_vector/...`, `11_entity_vector/...`, `12_odor_vector/...` | vector construction |
| Current notebooks | `final/ICOCO_notebook_Colab.ipynb`, `final/ICOCO_notebook.ipynb` | current external metric (cell 5.3) |
| New vectors | `7_create_npy_files/*.npy` | `(50511, ...)` |
| Figure assets | `old_code/17_result_and_discussion_/*.jpg`; `old_code/Figure assets/` | rendered figures |
| **Absent** (searched, not found) | producer of `labeled.csv` / `kmeans_benchmark_summary_results.csv`; `raw_results_*.csv`; `final_leaderboard_summary.csv` | **missing** |

**Reproduction protocol used in this audit.** All external numbers below were recomputed here from the **legacy vectors + legacy labels** with `sklearn.KMeans(n_init=3)`, single seed **42**, on the 30,621 labelled rows, and separately on the full 50,513 rows. `labeled.csv` is a mean over 5 seeds, so single-seed values differ by seed noise; the discrepancies reported are **orders of magnitude larger** than seed noise.

**Minor data-count observation.** The manuscript states 50,510 recipes (30,621 labelled + 19,889 unlabelled); `MasterBattle.csv` actually holds **50,513** rows with a **30,621 / 19,892** split, and the current `7_create_npy_files/*.npy` have **50,511** rows. The small discrepancy is immaterial to the metrics but should be reconciled in any reproducibility package.

---
## 2. External Metric Reproduction

### 2.1 Protocols tested

1. **Hungarian bipartite matching** (`scipy.optimize.linear_sum_assignment(-M)`) — the procedure the paper *describes* (Sec. IV-C / Eq. 7).
2. **Per-cluster majority / argmax** — assign each cluster the most frequent ground-truth tag among its members.
3. **`EXT_ASSIGN='legacy'`** — the behaviour currently implemented in the notebook's cell 5.3 (bijection for FoodBERT, per-cluster majority for the proposed models).
4. **Cluster labelled subset only** vs **cluster the full 50,513 and evaluate the labelled subset** (the paper says labelled+unlabelled are "clustered jointly").
5. K in {6 (continents), 31 (sub-regions), 207 (countries)}.

### 2.2 The paper's source values (`labeled.csv`) — PA (%) / SF1

| model | PA K=6 | PA K=31 | PA K=207 | SF1 K=6 | SF1 K=31 | SF1 K=207 |
|---|---|---|---|---|---|---|
| FoodBERT_Baseline | 30.01 | 9.42 | 5.94 | 0.1731 | 0.0479 | 0.0539 |
| Our_Full_Fusion | 71.74 | 43.94 | 12.53 | 0.1393 | 0.0639 | 0.0951 |
| Ablation_No_Entropy | 71.74 | 37.28 | 12.30 | 0.1393 | 0.0712 | 0.0905 |
| Ablation_Geo_Only | 71.68 | 61.27 | 28.80 | 0.1398 | 0.0724 | 0.1043 |
| Ablation_Entity_Only | 57.85 | 22.50 | 8.13 | 0.1667 | 0.0619 | 0.0695 |
| Ablation_Odor_Only | 55.61 | 30.35 | 20.82 | 0.1786 | 0.0528 | 0.0748 |
| Ablation_Odor+Entropy | 54.93 | 29.90 | 20.82 | 0.1784 | 0.0512 | 0.0762 |

`labeled.csv` also reports NMI/ARI; the majority-class baseline's NMI is ~0 for the constant models.

### 2.3 Independent reproduction (legacy vectors/labels; seed 42; `n_init=3`)

**PA (%) - cluster labelled subset only:**

| model | bijection K=6/31/207 | per-cluster-majority K=6/31/207 |
|---|---|---|
| FoodBERT_Baseline | **30.07 / 8.76 / 5.33** | 71.76 / 63.65 / 61.78 |
| Our_Full_Fusion | 33.47 / 10.21 / 6.47 | 71.76 / 63.65 / 62.05 |
| Ablation_No_Entropy | 29.18 / 9.62 / 6.62 | 71.76 / 63.93 / 62.11 |
| Ablation_Geo_Only | 39.15 / 26.57 / 26.95 | 71.76 / 63.91 / 62.63 |
| Ablation_Entity_Only | 26.07 / 10.75 / 6.44 | 71.76 / 63.65 / 61.91 |
| Ablation_Odor_Only | 35.36 / 24.83 / 20.15 | 71.76 / 63.65 / 61.86 |
| Ablation_Odor+Entropy | 40.10 / 20.45 / 20.22 | 71.76 / 63.69 / 61.86 |

**SF1 (Eq. 8) - cluster labelled subset only:**

| model | bijection K=6/31/207 | per-cluster-majority K=6/31/207 |
|---|---|---|
| FoodBERT_Baseline | **0.1747 / 0.0478 / 0.0542** | 0.1393 / 0.0311 / 0.0112 |
| Our_Full_Fusion | 0.1781 / 0.0471 / 0.0618 | 0.1393 / 0.0311 / 0.0211 |
| Ablation_No_Entropy | 0.1638 / 0.0533 / 0.0653 | 0.1393 / 0.0424 / 0.0234 |
| Ablation_Geo_Only | 0.1914 / 0.0632 / 0.0885 | 0.1393 / 0.0493 / 0.0494 |
| Ablation_Entity_Only | 0.1623 / 0.0580 / 0.0601 | 0.1393 / 0.0311 / 0.0188 |
| Ablation_Odor_Only | 0.1794 / 0.0560 / 0.0760 | 0.1393 / 0.0311 / 0.0139 |
| Ablation_Odor+Entropy | 0.1818 / 0.0546 / 0.0720 | 0.1393 / 0.0320 / 0.0140 |

**Majority-class baseline:** K=6 (North America) PA **71.76**, SF1 0.1393; K=31 (Northern America) PA **63.65**, SF1 0.0251; K=207 (United States of America) PA **61.78**, SF1 0.0037.

**Joint (full-dataset) clustering** returned the *same* labelled-row scores as labelled-subset clustering (Full: 34.24/10.36/6.36 bijection, 71.76/63.65/61.98 majority; Geo: 39.48/26.74/27.01, 71.76/63.94/62.63) -> the "clustered jointly" hypothesis does **not** explain the differences.

### 2.4 Which rows reproduce, and which do not

* **Reproduce (foodbert):** FoodBERT matches **bijection** for PA (30.07/8.76/5.33 vs 30.01/9.42/5.94) **and** SF1 (0.1747/0.0478/0.0542 vs 0.1731/0.0479/0.0539) **and** NMI (0.0130/0.0354/0.0789 vs 0.0134/0.0377/0.0801).
* **Reproduce (proposed, continents only):** Full/NoEnt/Geo PA at K=6 match the **majority baseline** (71.76 vs 71.74/71.74/71.68) and Full/NoEnt/Geo SF1 at K=6 match **per-cluster-majority** (0.1393).
* **Do NOT reproduce:** every proposed-model cell at **K=31 and K=207** (PA and SF1) matches neither rule; and `Entity_Only`/`Odor_Only`/`Odor+Entropy` at **K=6** match neither rule.
* `Geo_Only` at K=207 matches bijection PA (26.95 vs 28.80) and `Odor_Only`/`Odor+Entropy` at K=207 match bijection PA (20.15/20.22 vs 20.82).
* **Internal inconsistency:** `labeled.csv` `Entity_Only` K=6 has PA that matches no rule while its SF1 matches bijection -> PA and SF1 cannot come from one assignment.

---

## 3. Claim-by-Claim Audit

Status legend: 🟢 SUPPORTED · 🟡 SUPPORTED BUT OVERSTATED · 🟠 PARTIALLY SUPPORTED · 🔴 UNSUPPORTED / CONTRADICTED · ⚫ NOT AUDITABLE.

| ID | Manuscript claim | Evidence / figure / metric | Reproducible result | Status | Reason |
|----|------------------|----------------------------|---------------------|--------|--------|
| C1 | "our non-semantic representation consistently outperforms FoodBERT in PA across all evaluated granularities" (§V-B-1) | Fig. 3; `labeled.csv` Full 71.74/43.94/12.53 vs FB 30.01/9.42/5.94 | Under Hungarian bijection: Full 33.47/10.21/6.47 vs FB 30.07/8.76/5.33 (Full > FB at all K, marginally) | 🟠 PARTIALLY SUPPORTED | Direction holds under a correct metric, but the **magnitude/evidence in Fig. 3 does not reproduce**; the K=6 value (71.74) is the **majority-class baseline**, not a learned result |
| C2 | "relies on biological presence and chemical profiles that are **immune to textual noise**" (§V-B-1) | none - interpretive | No experiment isolates or measures "immunity to textual noise" | 🟡 SUPPORTED BUT OVERSTATED | "Immune" is a mechanistic absolute never tested; the comparison only shows non-semantic vectors sometimes cluster better |
| C3 | "maintain superior alignment at regional scales" (§V-B-1) | Fig. 3; `labeled.csv` | Full > FB at K=31 (10.21 vs 8.76) and K=207 (6.47 vs 5.33) under bijection | 🟠 PARTIALLY SUPPORTED | Direction holds, but the reported values (37.23 / 12.08) are **not reproducible** by any rule |
| C4 | "ultimately achieving **nearly double FoodBERT's SF1** at the country level (K=207)" (§V-B-1) | Fig. 3; `labeled.csv` 0.0951 vs 0.0539 | Under coherent bijection (Eq. 8): 0.0618 vs 0.0542 = **x1.14** | 🔴 UNSUPPORTED / CONTRADICTED | "Nearly double" holds **only** under the inconsistent historical values; a coherent metric gives ~1.14x |
| C5 | "**proving** that physical grounding becomes **increasingly advantageous** as macro-labels fragment" (§V-B-1) | Full-FB PA gap | Gap under bijection: +3.40 -> +1.45 -> +1.14 (**shrinks**); even in `labeled.csv` PA gap shrinks 41.7 -> 34.5 -> 6.6 | 🔴 CONTRADICTED | The measured advantage **decreases** with granularity; "proving" is a causal absolute |
| C6 | "**every** proposed variant consistently surpasses FoodBERT in PA across all granularities" (§V-B-2) | Fig. 4; `labeled.csv` | Under bijection: Entity_Only 26.07 and No_Entropy 29.18 are **below** FoodBERT 30.07 at K=6 | 🔴 CONTRADICTED | Fails for two variants under a coherent metric |
| C7 | "they [Entity/Odor] consistently maintain higher scores than the baseline at K=31 and K=207" (§V-B-2) | Fig. 4 | Entity 10.75/6.44 > FB 8.76/5.33; Odor 24.83/20.15 > FB - all under bijection | 🟢 SUPPORTED | Direction reproduces under a coherent metric (magnitudes differ from `labeled.csv`) |
| C8 | "trends **confirm** that ingredient categorizations and thermal aroma profiles **naturally encode** cultural signals" (§V-B-2) | Fig. 4 | Entity/Odor > FB at deep K (correlational) | 🟡 SUPPORTED BUT OVERSTATED | Result is a clustering correlation; "confirm"/"naturally encode" imply a mechanism not established |
| C9 | "our non-semantic framework demonstrates **clear superiority in geometric cohesion**" (§V-B-3) | Fig. 5; `all_and_unlabel.csv` | Silhouette/DB/CH reproduced (see §6); proposed > FB on Silhouette & DB, FB wins some CH | 🟢 SUPPORTED | Internal metrics reproduce; FoodBERT wins isolated CH cells (paper acknowledges this) |
| C10 | "consistently outperforms FoodBERT across **both internal geometric and external alignment** metrics at deeper levels" (§V-C-a) | Fig. 5 + Fig. 3/4 | Internal: yes; External deeper K: Full>FB marginal (>FB) under bijection | 🟠 PARTIALLY SUPPORTED | Internal part solid; external part is marginal and rests on non-reproducible values |
| C11 | "These findings **prove** that regional culinary traditions **naturally align** with ecological and chemical constraints" (Conclusion) | Fig. 3/4/5 | Correlational clustering; no causal test | 🟠 PARTIALLY SUPPORTED | A correlation is shown, but "prove"/"naturally align" overstate a clustering correlation computed with a disputed metric |
| C12a-i | Abstract: "nearly doubles FoodBERT's Soft-MacroF1" (country level) | Fig. 3 | ratio x1.14 (coherent bijection), not ~x2 | 🔴 UNSUPPORTED / CONTRADICTED | Same failure as C4 |
| C12a-ii | Abstract: "maintaining higher Partial-Match Accuracy across cultural granularities" | Fig. 3 | PA higher at all K (marginal) | 🟠 PARTIALLY SUPPORTED | Same as C1 |
| C12b | Contribution: "Unsupervised Verification ... validate cultural boundaries via internal and external metrics" | Fig. 3-5 | Internal valid; external not reproducible | 🟠 PARTIALLY SUPPORTED | The external "validation" depends on the unreproducible metric |
| C12c | Limitations: "Geo-Only **occasionally** yields higher alignment than Full Fusion" | `labeled.csv` / repro | Under bijection **Geo > Full at all three K** (39.15>33.47, 26.57>10.21, 26.95>6.47) | 🟠 PARTIALLY SUPPORTED | The limitation is **understated**: under a coherent metric it is systematic, not occasional |
| C12d | Internal headline "Silhouette up to 0.908" | `all_and_unlabel.csv` | max S = 0.9080 (Geo, No-Label, K=248); reproduced | 🟢 SUPPORTED | Exact |
| C12e | Practical-implications / cold-start prior | design claim | Independent of the disputed metric; representation exists | 🟢 SUPPORTED | Not a quantitative claim; not affected by the external-metric problem |
| C12f | LLM validation numbers (SC/IC/AC; USDA 71.20%, GBIF 93.29%) | `6_confirm_LLM_performance_number/` | Not re-run in this audit (out of scope) | ⚫ NOT AUDITABLE | Outside the external-metric dispute; documented elsewhere in the repo |

**Additional material claims depending on the disputed metric:**

* "demonstrates strong cluster cohesion and **superior fine-grained cultural alignment over FoodBERT**" (Abstract) - the alignment clause inherits C1/C3/C4.
* "confirming that culinary traditions share natural constraints" (Abstract) - inherits C11.
* "fine cultural boundaries emerge from natural ecological and chemical constraints" (§V-C-a) - inherits C11.

---

## 4. External PA Audit

### 4.1 What procedure produced the proposed-model PA values?

The proposed models' **continents** PA in `labeled.csv` (Full 71.74, No_Entropy 71.74, Geo 71.68) exactly equals the **global majority-class baseline**: over the 30,621 labelled recipes, the majority continent tag is **North America** (present in 71.76% of recipes). "Always predict North America" yields PA = 71.76, which reproduces Full/NoEnt/Geo at K=6 to two decimals. This is the definitional signature of a **per-cluster-majority (argmax)** assignment on clusters that are **not** aligned with continents - i.e. *every* cluster takes the same modal label.

At **K=31 / K=207** the proposed-model PA values in `labeled.csv` (e.g. Full 43.94 / 12.53) match **neither** per-cluster majority (63.65 / 61.78) **nor** Hungarian bijection (10.21 / 6.47). They are therefore not explained by either rule.

### 4.2 Does "consistently outperforms FoodBERT in PA" hold?

Under one coherent metric (Hungarian bijection, PA in %):

| model | K=6 | K=31 | K=207 |
|---|---|---|---|
| FoodBERT_Baseline | 30.07 | 8.76 | 5.33 |
| Our_Full_Fusion | 33.47 (+3.40) | 10.21 (+1.45) | 6.47 (+1.14) |

* Full **does** exceed FoodBERT at all three K - but by a **shrinking** margin (+3.40 -> +1.45 -> +1.14), the **opposite** of claim C5.
* The paper's figure shows a much larger, **non-reproducible** margin (41.7 at K=6), because 71.74 is the majority baseline.

### 4.3 Does "every proposed variant surpasses FoodBERT in PA across all granularities" hold?

Under Hungarian bijection at **K=6**: FoodBERT 30.07; Full 33.47; **No_Entropy 29.18 (< FoodBERT)**; Geo 39.15; **Entity_Only 26.07 (< FoodBERT)**; Odor 35.36; Odor+Ent 40.10. -> **two of six variants fall below FoodBERT** at K=6. The claim is **contradicted** for a coherent metric; it holds only under the inconsistent `labeled.csv`.

### 4.4 Quantified difference

| quantity | `labeled.csv` | Hungarian bijection | per-cluster majority |
|---|---|---|---|
| Full PA K=6 | 71.74 | 33.47 | 71.76 |
| Full PA K=31 | 43.94 | 10.21 | 63.65 |
| Full PA K=207 | 12.53 | 6.47 | 62.05 |

At continents, the paper's number is the **majority-class prediction**; at sub-regions/countries it is **between** the two rules and matches neither.

---

## 5. External SF1 Audit

### 5.1 Does the reported SF1 use one coherent protocol?

The paper's Eq. 8 (single `(y_true, y_pred)` pair per recipe: hit -> `y_true = y_pred = y'_q`; miss -> `y_true` = first-alphabetical true tag; macro-F1 over classes) was implemented here. FoodBERT's SF1 reproduces it under **Hungarian bijection** at **all three K** (0.1747 / 0.0478 / 0.0542 vs `labeled.csv` 0.1731 / 0.0479 / 0.0539). So FoodBERT's external metric is a **coherent bijection**.

### 5.2 Are the proposed-model SF1 values reproducible under the same protocol?

| model | K | `labeled.csv` SF1 | bijection SF1 | majority SF1 | reproducible? |
|---|---|---|---|---|---|
| Full | 6 | 0.1393 | 0.1781 | 0.1393 | by **majority** only |
| Full | 31 | 0.0639 | 0.0471 | 0.0311 | **neither** |
| Full | 207 | 0.0951 | 0.0618 | 0.0211 | **neither** |
| Geo_Only | 6 | 0.1398 | 0.1914 | 0.1393 | by **majority** |
| Entity_Only | 6 | 0.1667 | 0.1623 | 0.1393 | by **bijection** (but its PA at K=6 matches neither rule) |

### 5.3 Is the country-level "nearly double SF1" claim supported?

* `labeled.csv`: Full 0.0951 / FB 0.0539 = **x1.76** (paper plot conservative values: 0.0941 / 0.0546 = x1.72).
* Coherent metric (Hungarian bijection, Eq. 8): 0.0618 / 0.0542 = **x1.14**.
* Per-cluster majority: 0.0211 / 0.0112 = x1.88 (but this is a degenerate/majority prediction, not a cultural-alignment result).

**Verdict:** the statement is **true only under the historical `labeled.csv` values**; it is **not supported by a consistent metric** (x1.14 is not "nearly double").

### 5.4 Internal inconsistency between PA and SF1

For `Entity_Only` at K=6, `labeled.csv` PA = 57.85 matches **no** rule (bijection 26.07 / majority 71.76) while its SF1 = 0.1667 matches **bijection** (0.1623). Because PA and SF1 are functions of the **same** assignment, a single coherent protocol cannot yield both. This is direct evidence that the historical table is **not internally consistent**.

---

## 6. Internal Metric Audit

The internal metrics are **independent of the external-metric problem** and were audited separately. They reproduce well.

| Group | Model | K | legacy `all_and_unlabel.csv` (S / DB / CH) | reproduced here (S / DB / CH) | match |
|---|---|---|---|---|---|
| Full_Dataset | FoodBERT | 6 | 0.1010 / 2.0276 / 7469.5 | 0.1002 / 2.0275 / 7469.5 | ✅ |
| Full_Dataset | FoodBERT | 31 | 0.0457 / 2.7126 / 2087.2 | 0.0458 / 2.7337 / 2086.7 | ✅ |
| Full_Dataset | Full_Fusion | 6 | 0.2005 / 1.8951 / 114457.8 | 0.1762 / 1.8280 / 116477.8 | ~ (legacy SD 0.073) |
| Full_Dataset | Full_Fusion | 31 | 0.1158 / 2.0744 / 30604.0 | 0.1151 / 2.1206 / 30454.2 | ✅ |
| Full_Dataset | Geo_Only | 6 | 0.5638 / 0.9625 / 24894.9 | 0.5610 / 0.9795 / 24982.2 | ✅ |
| Full_Dataset | Geo_Only | 31 | 0.7623 / 0.9782 / 22895.6 | 0.7640 / 1.0522 / 22851.5 | ✅ |
| No_Label_Subset | FoodBERT | 6 | 0.1011 / 2.0490 / 3038.2 | 0.1004 / 2.0528 / 3038.2 | ✅ |
| No_Label_Subset | FoodBERT | 31 | 0.0443 / 2.7566 / 837.7 | 0.0455 / 2.7336 / 837.5 | ✅ |
| No_Label_Subset | Full_Fusion | 6 | 0.2166 / 1.8079 / 47482.9 | 0.1585 / 2.1459 / 47106.3 | ~ (legacy SD 0.077) |
| No_Label_Subset | Full_Fusion | 31 | 0.1171 / 2.0780 / 12772.1 | 0.1206 / 1.9848 / 12793.3 | ✅ |

(Full legacy values in `all_and_unlabel.csv` - full table at K in {6,31,248} for all 7 models, both groups.)

**Findings:**

* Every checked internal value reproduces to <~5% (most <1%); the CH column is essentially exact (legacy CH SD was frequently 0.0, i.e. all seeds agreed).
* The two `~` rows are `Full_Fusion` at K=6, where the legacy **per-seed SD is large** (Silhouette SD 0.073-0.077); a single seed lands inside that band. No contradiction.
* The legacy internal battle uses **K in {6, 31, 248}** (matching the paper's Fig. 5), whereas the external table uses **K in {6, 31, 207}** (matching Fig. 3). This K mismatch between the internal and external evaluations is by design (207 gold countries vs 248 country codes) and is a legitimate choice, not an error - but it should be stated explicitly.
* **The internal claims (C9, C12d) are supported and do NOT depend on the disputed external metric.** High external scores being "not driven by sparse or fragmented clusters" (§V-B-3) is consistent with these reproducible internal numbers.

---

## 7. Interpretation and Causal-Claim Audit

The manuscript uses strong epistemic verbs ("proving", "confirming", "demonstrates", "validating") and mechanistic absolutes ("immune", "naturally", "inherently", "increasingly advantageous"). The audit flags these where the experiments do not justify the strength.

| Passage | Word(s) | Does the experiment justify it? | Flag |
|---|---|---|---|
| "**proving** that physical grounding becomes **increasingly advantageous** as macro-labels fragment" | proving / increasingly | No - the advantage **shrinks** (+3.40 -> +1.14 PA) under a coherent metric; "proving" implies a controlled causal test that was not run | 🔴 CONTRADICTED (also §4) |
| "chemical profiles that are **immune to textual noise**" | immune | No - "immunity to textual noise" is never measured; only a clustering comparison is shown | 🟡 OVERSTATED |
| "trends **confirm** that ingredient categorizations and thermal aroma profiles **naturally encode** subtle cultural signals" | confirm / naturally encode | Partly - a correlational clustering gap is shown, not an encoding mechanism; and the inputs are **curated database** fields (USDA categories, FooDB nutrients), so "naturally" is imprecise | 🟡 OVERSTATED |
| "These findings **prove** that regional culinary traditions **naturally align** with ecological and chemical constraints" | prove / naturally align | No - a correlation under a (disputed) metric cannot establish a natural/causal alignment; no ecological variable is manipulated | 🟠 PARTIALLY SUPPORTED |
| "Our empirical findings directly **validate** our core research questions" | validate | The external half of the validation rests on non-reproducible values | 🟠 PARTIALLY SUPPORTED |
| "physical grounding **naturally constructs** dense, well-separated cluster spaces" | naturally | The internal geometry is reproducible, but "naturally" adds a mechanism the experiment does not isolate | 🟡 OVERSTATED |

**Two structural confounds that weaken the mechanistic reading even before the metric issue:**

1. **Non-isolation of the mechanism.** The comparison is *non-semantic vector vs a text baseline*. It does not isolate "biological presence" from "chemical profiles" from "geographical footprint"; the ablation tracks are correlated (Geo especially) and `Geo_Only` frequently **beats** `Full_Fusion`, so the claim that the *specific* mechanisms "encode" culture is not established.
2. **Correlation vs causation.** All evidence is unsupervised clustering scored against tags derived (via the LLM pipeline) from the **same recipe metadata**, so a degree of circularity exists between representation and evaluation target; none of this supports the causal verbs used.

---

## 8. Historical Pipeline Reconstruction

| # | Question | Answer from the repository evidence |
|---|---|---|
| 1 | Which code generated `labeled.csv`? | **Not found.** No notebook in `old_code/` writes it; the files that mention "Partial Match"/"Soft Macro" (`16_figure_export/unsupervised_with_geographic_label.ipynb`, `17_result_and_discussion_/ablation_labeled_analysis.ipynb`, `.../labeled_sample_basic_analysis.ipynb`, `.../Redraw_all_figures.ipynb`) only **read** `labeled.csv`. The producer of `labeled.csv` / `kmeans_benchmark_summary_results.csv` is missing. |
| 2 | Which code generated the figures? | `old_code/17_result_and_discussion_/Redraw_all_figures.ipynb` - it reads `labeled.csv` + `all_and_unlabel.csv`, applies the **conservative stress-test** transform (Proposed = mean-SD, FoodBERT = mean+SD), and plots. This step is fully reconstructable and correct. |
| 3 | Which assignment rule was actually used? | **Cannot be established.** FoodBERT's rows are consistent with Hungarian bijection; the proposed-model rows are consistent with **neither** bijection **nor** majority at K=31/207, and PA/SF1 are mutually inconsistent for `Entity_Only`. |
| 4 | Same rule for FoodBERT and proposed models? | **No single rule fits both.** The evidence is inconsistent with one coherent rule applied uniformly. |
| 5 | Evidence the metric changed between experiments? | Circumstantial: the FoodBERT rows fit bijection, the proposed continents rows fit majority, and finer rows fit neither - consistent with a **rule change or a bug**, but not provable from the surviving files. |
| 6 | Is the original producer notebook/code available? | **No.** Explicitly missing (searched `old_code/` and the repository). |
| 7 | Can the exact historical protocol be reconstructed? | **No.** It cannot be established from the available evidence. |
| 8 | Unexplained transformations between raw results and plotted values? | The **raw-cluster -> `labeled.csv`** step is unexplained (producer missing). The **`labeled.csv` -> figure** step is fully explained (conservative transform in `Redraw_all_figures.ipynb`). |

> **Statement of missing evidence (per instruction):** *The notebook that produced `labeled.csv` is missing from the repository. Accordingly, the exact historical cluster->label assignment protocol cannot be established from the available evidence; this audit does not infer what that code "must have" done.*

---

## 9. Academic-Integrity Assessment

**Question:** *If the author did not know that the assignment procedure was inconsistent at the time the paper was written, does the available evidence support characterizing this as a methodological / reproducibility failure, or is there evidence of intentional manipulation?*

**Assessment (evidence-based, intent not inferred):**

The evidence is consistent with **methodological error / reproducibility failure**, and **there is no evidence in the repository of intentional manipulation**. Concretely, mapping the observed phenomena onto the requested categories:

| Category | Present in the evidence? | Basis |
|---|---|---|
| **Implementation bug** | **Plausible** | PA and SF1 for `Entity_Only` cannot both arise from one assignment; finer-K rows fit no rule. Both are signatures of a coding error (e.g. an assignment applied to the wrong array, a mixed/overwritten mapping, or a `classes[...]` indexing mistake). |
| **Methodological error** | **Yes** | Using per-cluster-majority for the proposed models while the paper *describes* Hungarian bipartite matching (Sec. IV-C). The stated method and the apparent method disagree. |
| **Undocumented metric inconsistency** | **Yes** | The same table mixes a FoodBERT-consistent rule with a differently-assigned proposed-model rule; the inconsistency is not documented anywhere in the repo. |
| **Incorrect interpretation** | **Yes** | The degenerate majority-baseline value (71.76 at K=6) is interpreted as "superior cultural alignment" and as "increasingly advantageous" (C4/C5), which a correct metric contradicts. |
| **Irreproducible historical pipeline** | **Yes** | The producer of `labeled.csv` is missing; the protocol cannot be reconstructed. |
| **Selective reporting** | **Not established** | No alternative run, seed sweep or negative result is present to compare against; a single summary CSV survives. Selective reporting cannot be established from what remains. |
| **Intentional manipulation** | **Not established** | See below. |

**Why intent is not established (and must not be inferred):**

* An inconsistent or incorrect result is, by itself, **not** evidence of intent. Bugs and mis-specified metrics produce exactly the pattern seen here.
* The proposed models are **not uniformly favored**: the degenerate rule *inflates* their continents PA, but at finer granularity their `labeled.csv` values are **lower** than a per-cluster-majority rule would give (e.g. Full K=31: 43.94 vs majority 63.65). A deliberate inflation would more plausibly use the higher (majority) value everywhere.
* FoodBERT's row is the **honest, correct** bijection - an author manipulating results to favor their own method would not normally leave the baseline on a correct metric while placing their own on an inconsistent one.
* No hidden script, override, or hard-coded result table was found; the missing producer is missing, not concealed.

> **Required statement:** *The available repository evidence is insufficient to establish intentional manipulation.*

**What additional evidence would be needed before a stronger accusation could even be considered:**

1. The **original producer notebook/script** for `labeled.csv` / `kmeans_benchmark_summary_results.csv` (to see the actual assignment code).
2. The **raw per-seed clustering outputs** (`raw_results_*.csv`), to test whether the reported means are faithful summaries.
3. Any **version history / backups** of the metric code showing whether and when the assignment rule changed.
4. Any record showing the authors **knew** the rule differed from the stated Hungarian matching while still reporting it as such.

Absent these, the conservative, defensible characterization is a **methodological and reproducibility failure**, not fabrication.

---

## 10. Severity Ranking

### 🔴 Must be corrected or withdrawn

* **C4 / C12a** — "nearly double FoodBERT's SF1 at the country level". A coherent metric gives **x1.14**, not ~x2. The headline (also in the Abstract and Conclusion) is **not supported**.
* **C5** — "proving that physical grounding becomes increasingly advantageous as macro-labels fragment". The measured advantage **decreases** with granularity; also a causal absolute ("proving").
* **C6** — "every proposed variant consistently surpasses FoodBERT in PA across all granularities". **False** under a coherent metric (`Entity_Only`, `No_Entropy` fall below FoodBERT at K=6).
* **Fig. 3 / Fig. 4 plotted values for the proposed models at K=31 and K=207** — match **no** coherent assignment rule; they are not reproducible from the available data and code.

### 🟠 Must be qualified

* **C1** — "consistently outperforms FoodBERT in PA across all evaluated granularities": direction survives, but the magnitude/evidence is an artifact (K=6 = majority baseline).
* **C3** — "maintain superior alignment at regional scales": direction survives; the reported magnitudes do not.
* **C10** — "consistently outperforms ... across both internal geometric and external alignment metrics at deeper levels": internal solid, external marginal and resting on non-reproducible values.
* **C11** — "These findings prove that regional culinary traditions naturally align with ecological and chemical constraints": causal/mechanistic overreach plus disputed metric.
* **C12b** — Contribution claim that cultural boundaries are "validated" via external metrics.
* **C12c** — Limitation "Geo-Only *occasionally*" higher: under a coherent metric Geo-Only beats Full-Fusion at **all three** K (understated).

### 🟡 Wording should be weakened

* **C2** — "immune to textual noise": no experiment measures textual robustness.
* **C8** — "trends *confirm* ... *naturally encode*": a correlation is shown; the encoding mechanism is not isolated.
* **§V-B-3** — "physical grounding *naturally constructs* dense, well-separated cluster spaces": internal result is sound, but "naturally" implies an untested mechanism.
* "Our empirical findings directly *validate* our core research questions": external half rests on non-reproducible values.

### 🟢 Supported

* **C7** — Entity/Odor tracks maintain higher PA than FoodBERT at K=31 and K=207 (under a coherent metric).
* **C9 / C12d** — internal geometric superiority; Silhouette up to **0.908** (exact).
* **All internal metrics** (Fig. 5): Silhouette / Davies-Bouldin / Calinski-Harabasz reproduce within seed noise.
* **FoodBERT's external rows** (PA, SF1, NMI, ARI) reproduce under the paper's stated Hungarian bijection.
* **Fig-construction pipeline** (`labeled.csv` -> conservative transform -> figure) is correct and reproducible.
* **C12e** — practical/cold-start design claim (independent of the disputed metric).
* **LLM validation numbers** — documented separately in the repo (audited only superficially here).

---

## 11. Recommended Next Step

This is a factual recommendation; **the manuscript is not rewritten in this audit.**

1. **Recover or formally retire the producer.** Locate the notebook/script that generated `labeled.csv` / `kmeans_benchmark_summary_results.csv`. If it cannot be found, **explicitly declare the historical protocol unrecoverable** rather than reconstructing a metric that happens to match.
2. **Recompute the external metric once, with one documented rule.** Use the procedure the paper *describes* - Hungarian bipartite matching (Eq. 7) with the SF1 of Eq. 8 - for **all** models, and regenerate Fig. 3 / Fig. 4 from that single, consistent run (with the majority-class baseline shown alongside for reference).
3. **Re-derive the conclusions under the corrected metric.** Specifically re-evaluate the "nearly double country-level SF1" and "increasingly advantageous with granularity" statements; if they do not survive, **revise the claims**, not the metric.
4. **State the evaluation constants explicitly**: the two different K sets (external K = 207 gold countries; internal K = 248 country codes), the labelled/unlabelled split (30,621 / 19,892), and the number of seeds.
5. **Align the stated method with the implementation.** Ensure the shipped code implements Hungarian bijection for every model (or document, with justification, any model-specific assignment).
6. **Rewrite causal/mechanistic language** where the experiments only support correlation ("prove" -> "are consistent with"; "immune"/"naturally" -> describe the measured quantity).
7. **Verify the SF1 implementation** against Eq. 8 (single-pair formulation) in the released code, since the naive multi-hot macro-F1 diverges at deeper K (a trap that can itself reintroduce inconsistency).

**Bottom line:** the internal geometry results and FoodBERT's external results are sound and reproducible; the paper's central *external* claim - that the proposed non-semantic representation substantially out-aligns FoodBERT (especially at the country level) - is **not** supported by a coherent metric and must be re-established or revised.




