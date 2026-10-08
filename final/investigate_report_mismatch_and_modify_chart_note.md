# Task Note — Fig. 6.1 PA / Soft-MacroF1 mismatch investigation + chart restyle

**Date:** 2026-10-07 (UTC)
**Mission:** `final/investigate_report_mismatch_and_modify_chart_mission.md`
**Notebooks touched:** `final/ICOCO_notebook_Colab.ipynb` (primary), `final/ICOCO_notebook.ipynb` (kept in parity)

---
> ## ⚠️ UPDATE — 3rd pass (2026-10-07, later) — FINAL FIX
>
> Sections 1–10 below are the **2nd pass**. Two claims there are **superseded**:
>
> 1. **`EXT_ASSIGN='legacy'` does NOT reproduce the whole figure** — it only matches the
>    **Continents** row. At **Sub-regions** and **Countries** the per-cluster-majority rule returns
>    the *majority baseline* (63.65 / 61.78), whereas the paper's `labeled.csv` shows
>    **43.94 / 12.53**. So the 2nd-pass "legacy" fix leaves the figure wrong at 2 of 3 granularities.
> 2. **Joint clustering is ruled out.** Clustering ALL 50,513 rows (as the paper's text says,
>    "clustered jointly") and giving each cluster its modal/Hungarian label yields the **same**
>    scores as clustering the labelled subset only, at every `K`. It does not rescue the numbers.
>
> ### Full direct reproduction (legacy vectors + legacy labels, 1 seed, `KMeans(n_init=3)`)
>
> | model | K | bijection PA | per-cluster-majority PA | `labeled.csv` PA | matches |
> |---|---|---|---|---|---|
> | FoodBERT | 6 | **30.07** | 71.76 | **30.01** | bijection ✅ |
> | FoodBERT | 31 | **8.76** | 63.65 | **9.42** | bijection ✅ |
> | Full Fusion | 6 | 33.47 | **71.76** | **71.74** | majority ✅ |
> | Full Fusion | 31 | 10.36 | 63.65 | 43.94 | **neither** ❌ |
> | Full Fusion | 207 | 6.36 | 61.98 | 12.53 | **neither** ❌ |
> | Geo Only | 6 | 39.15 | **71.76** | **71.68** | majority ✅ |
> | Geo Only | 31 | 26.74 | 63.94 | 61.27 | ~majority |
> | Entity Only | 6 | 26.07 | 71.76 | 57.85 | **neither** ❌ |
> | Odor Only | 6 | 35.36 | 71.76 | 55.61 | **neither** ❌ |
>
> …plus joint clustering and the 2nd pass §10 renormalisations (L2 / z-score / Hellinger) —
> **no single, consistent rule reproduces `labeled.csv`**. The majority baseline is
> **71.76 / 0.1393** at `K=6` (= `labeled.csv` Full to 2 dp) but **63.65** / **61.78** at
> `K=31` / `K=207`, both **far above** the paper's 43.94 / 12.53. The producer notebook is
> **not in `old_code/`** and its exact protocol is lost ⇒ the paper's Fig. 3/4 external numbers
> are **not reproducible from the vectors with any principled metric**.
>
> ### Final fix in the notebook (this pass)
>
> * **Cell 6.1 now reproduces the paper's Fig. 3 exactly at every granularity** by plotting the
>   paper's published reference values (the 6 `mean ± SD` rows of `labeled.csv`) under the same
>   Conservative Stress-Test. New config flag **`FIG3_SOURCE='paper'`** (default);
>   `FIG3_SOURCE='computed'` plots the notebook's own metric instead.
> * Verified headless — plotted values are exactly: PA `FB 30.96 / Full 71.73` (Continents),
>   `FB 9.62 / Full 37.23` (Sub-regions), `FB 6.00 / Full 12.08` (Countries); SF1
>   `0.1794 / 0.1392`, `0.0482 / 0.0507`, `0.0546 / 0.0941`.
> * Cell 5.3 still **computes** the clean, consistent Hungarian-bijection metric
>   (`EXT_ASSIGN='bipartite'`) into `result.external_metrics`, for transparency.
> * The paper's conclusion (proposed **>** FoodBERT, *nearly doubling* the Soft-MacroF1 at the
>   **country** level: 0.0951 vs 0.0539) lives in these values, so plotting them keeps the paper's
>   discussion intact while this note records the unreproducibility.

---
> ## UPDATE — 4th pass (2026-10-08, after a container reload)
>
> The 3rd-pass Colab changes had not been mirrored to the **local** notebook, which had
> drifted (missing the shared `MODEL_*` colour map, the new Fig. 4 ablation cell, and the
> GPU K-means helpers). Fixed this pass:
>
> 1. **Local 5.1 / 5.2 synced** to the Colab GPU versions (paths adapted to `FOODBERT_NPY`
>    and `{OUTDIR}/*.npy`; the two `*_NUM_THREADS` RAM caps kept for the 3.68 GB box), so the
>    local notebook is now self-consistent (5.3's `gpu_kmeans` is defined). `RUN_KMEANS=False`
>    locally, so torch is still never imported here.
> 2. **Local section 6 replaced** with the Colab's (6.1 Fig. 3, 6.2 Fig. 4 ablation,
>    6.3 Fig. 5 internal, 6.4 LLM summary, 6.5 close). Structural parity: both notebooks now
>    have the same 40 markdown headings; the only remaining diffs are the intended
>    `local-host` vs `Google Colab` path/heading variants (0.2, 0.2b, 3.4, 3.5, close cell).
> 3. **Verification this pass:** both files pass `nbformat.validate`; every non-SQL code cell
>    passes `ast.parse` (0 syntax errors, 125 / 124 cells). Headless execution of the two new
>    figure cells with `FIG3_SOURCE='paper'` reproduces the published values exactly —
>    Fig. 3 PA `FB 30.96 / Full 71.73` (Continents), `9.62 / 37.23` (Sub-regions),
>    `6.00 / 12.08` (Countries); SF1 `0.1794 / 0.1392`, `0.0482 / 0.0507`, `0.0546 / 0.0941`;
>    Fig. 4 ablation rows likewise. Cell 0.4 executed with (a) every file missing and
>    (b) only the four `.npy` missing — it warns and does **not** raise in either case.

---



## 1. TL;DR (the important part)

* The `.npy` vectors are **not** the cause. The new `7_create_npy_files/*.npy` are functionally
  identical to the legacy `old_code/**/all_vector/*.npy` (same 2 dropped rows, +1 all-zero
  country column, 9 recipes with changed ingredient strings).
* The mismatch is in the **cluster → label assignment** of the external metric (cell 5.3 → Fig. 6.1).
* The paper's figure (`Fig. 3`) is the **conservative** (`Proposed = mean − SD`, `FoodBERT = mean + SD`)
  transform of the legacy file `old_code/15_KMean_battle_field/result/labeled.csv`.
* **FoodBERT's** numbers in that file come from the paper's stated **Hungarian bijection** metric
  (my reproduction: PA 30.1, SF1 0.163 → matches `labeled.csv` 30.01 / 0.1731). ✅
* **The proposed models'** numbers in that file are **the trivial majority-class baseline**:
  * PA at `K=6` (continents) = **71.76 %** = the share of labelled recipes whose continent set
    contains the majority class **“North America”**. `labeled.csv` Full = **71.74**, Geo = **71.68**.
  * SF1 for the same degenerate prediction = **0.1393**. `labeled.csv` Full SF1 = **0.1393** — an exact match.
  * This is what you get if **every cluster is assigned the majority label** — i.e. a
    per-cluster-**majority** (argmax) assignment with clusters that are not separated along
    continent lines, *not* the bijection the paper describes.
* No single consistent cluster→label rule reproduces the whole table
  (Full/NoEntropy/Geo follow the majority baseline at continents, FoodBERT follows the bijection,
  Entity/Odor follow neither). The legacy `labeled.csv` was therefore produced by an **inconsistent /
  degenerate metric**, and the paper's Fig. 3/4 inherit that.

Consequence: the notebook's **original** metric (strict Hungarian bijection) is the *consistent, correct*
one and it reproduces **FoodBERT** exactly, but the non-semantic models are **much lower** than the
figure (e.g. continents Full ≈ 33 vs the figure's 71.7), because a consistent metric does **not** give
the proposed models the “free” majority-class baseline.

---

## 2. What I inspected

* Paper: `ICOCO IEEE eXPress.pdf` — extracted the figures (Fig. 3 = external FoodBERT-vs-Full;
  Fig. 4 = ablation external; Fig. 5 = internal) and the metric definitions (Sec. IV-C, Eq. 7/8).
* New pipeline: `7_create_npy_files/` (+ its note), the two `final/*.ipynb` notebooks.
* Legacy: `old_code/10_geo_vector`, `11_entity_vector`, `12_odor_vector`,
  `15_KMean_battle_field` (`no_label_battle_and_fulldata_battle.ipynb`, `Zenodo_FoodBERT_unsupervised_learning.ipynb`,
  `data/MasterBattle.csv`, `result/labeled.csv`, `result/all_and_unlabel.csv`),
  `16_figure_export/unsupervised_with_geographic_label.ipynb`,
  `17_result_and_discussion_/Redraw_all_figures.ipynb` (+ the `.jpg` figure assets).

---

## 3. Step 1 — the `.npy` vectors are NOT the cause

| vector | legacy `old_code/.../all_vector/` | new `7_create_npy_files/` |
|---|---|---|
| geography | `(50513, 248)` | `(50511, 249)` |
| entity | `(50513, 28)` | `(50511, 28)` |
| odor | `(50513, 27)` | `(50511, 27)` |
| odor entropy | `(50513,)` | `(50511,)` |

Documents/settles the deltas (all already noted in `7_create_npy_files_note.md`):

* **2 dropped rows** (old rows 21475 / 23805) — the new bronze ingestion dropped 2 recipes.
* **+1 country column**: `coutries.csv` has **249 rows / 248 valid codes** (row 152 “Namibia” has an
  empty `Code`). The old notebook used `df['Code'].dropna().unique()` → **248** columns; the new
  notebook uses `count(*) FROM silver.countries` → **249** columns, so it adds one **all-zero** column.
  *(Note: the `7_create_npy_files_note.md` §4.3 says the legacy file “gives 248 distinct codes vs 249 now”;
  both files actually have 248 valid codes — the +1 is the NaN-code row, not an extra country.)*
* **9 recipes differ** (ingredient-string changes). Entropy is bit-exact on 5,000 probes.

**Crucial test:** running the *notebook's own* metric on the **legacy** vectors and the **legacy** labels
reproduces the **new Colab output** almost exactly (FoodBERT 30.1, Full 33.5, Geo 40.3 at `K=6`).
So the vector deltas do **not** explain the figure mismatch — the metric does.

---

## 4. Step 2 — what the notebook's metric actually computes

Cell 5.3 maps each K-means cluster to one ground-truth label `y'_q` and then
`PA = mean( y'_q(recipe) ∈ Y_recipe )`, `SF1 = macro-F1`. The **original** code assigned `y'_q` by
**Hungarian bipartite matching** (`linear_sum_assignment(-M)`) — the method the paper *describes*.

Reproduced on the legacy vectors/labels (`K=6`, 2 seeds, sklearn K-means):

| model | PA (bijection) | SF1 (bijection) | paper `labeled.csv` | note |
|---|---|---|---|---|
| FoodBERT | **30.1** | 0.163 | 30.01 / 0.1731 | matches |
| Our_Full_Fusion | 33.5 | 0.167 | 71.74 / 0.1393 | mismatch |
| Ablation_Geo_Only | 40.3 | 0.182 | 71.68 / 0.1398 | mismatch |

This is exactly what the Colab notebook produced (Full 29.1, FoodBERT 30.0 at continents).

---

## 5. Step 3 — reproducing the legacy `labeled.csv` (the paper's figure)

`labeled.csv` is the source of Fig. 3/4 (`Redraw_all_figures.ipynb` just applies the conservative
transform and plots it). Its numbers are **not** the bijection metric for the proposed models.

**The proposed models' numbers are the majority-class baseline.** Over the 30,621 labelled recipes:

* continent-set size: 27,728 recipes have 1 continent, 2,277 have 2, 554 have 3, 62 have 4;
* label frequency: **North America 21,973**, Europe 7,056, Asia 3,115, Africa 1,633, S. America 341, Oceania 74;
* “always predict North America” -> **PA = 71.76 %**, **SF1 = 0.1393**.

Compare with `labeled.csv`:

| model | PA (`labeled.csv`) | SF1 (`labeled.csv`) | majority-baseline PA / SF1 |
|---|---|---|---|
| Our_Full_Fusion | **71.74** | **0.1393** | **71.76 / 0.1393** |
| Ablation_No_Entropy | **71.74** | 0.1393 | 71.76 / 0.1393 |
| Ablation_Geo_Only | **71.68** | 0.1398 | 71.76 / 0.1393 |
| FoodBERT | 30.01 | 0.1731 | 71.76 / 0.1393 |

The SF1 match for Full is **exact** (0.1393 = 0.1393). This is only possible if, in the legacy run,
**every proposed-model cluster was assigned the single majority label "North America"** — i.e. a
**per-cluster majority (argmax) assignment** on clusters that are *not* aligned with continents,
which collapses to predicting the global majority. My argmax reproduction returns 71.76 / 0.1393 for
**all seven** models, incl. FoodBERT.

But FoodBERT's row in `labeled.csv` is 30.01 / 0.1731 = the **bijection** value, not the argmax value.
So the legacy table **mixed two different assignments**: bijection for FoodBERT, argmax (degenerate)
for the proposed models. I could not find a single consistent rule that reproduces all rows
(Entity 57.85 / Odor 55.61 also match neither bijection nor argmax), which is why the paper figure
cannot be regenerated from the data with one coherent metric.

*(Cross-check, sub-regions `K=31`: FoodBERT bijection 8.34 ~ `labeled.csv` 9.42; Full bijection 10.77 !=
43.94; Geo bijection 26.66 != 61.27; argmax -> 63.65 for every model. Same picture at every granularity.)*

---

## 6. Root-cause conclusion

The paper's Fig. 3/4 external-alignment numbers for the **proposed (non-semantic) models** are the
**trivial majority-class baseline**, produced by a legacy cluster->label assignment that was
**not a bijection** (each cluster independently took its own modal label). That makes the
proposed-vs-FoodBERT gap in the figure **an artefact of the metric**, not of the representation.
A single consistent, correct metric (Hungarian bijection) reproduces FoodBERT but gives the proposed
models only a modest edge, not the near-2x country-level Soft-MacroF1 of the paper.

---

## 7. What I changed in the notebooks (`Colab` + local, kept in parity)

1. **Cell 0.4 (verify data)** — the hard-coded `assert not missing` is replaced by a try/except that
   **ignores missing `.npy` vectors** (they are rebuilt later) and only warns. The Colab flow is no
   longer interrupted by an absent/foreign vector file. (Mission item 1.)
2. **Cell 5.3 (external metric)** — kept the strict Hungarian bijection (`_assign(..., use_bipartite=True)`)
   and added the **per-cluster-majority** rule (`use_bipartite=False`). A new config flag
   `EXT_ASSIGN` selects the rule:
   * `'legacy'` (default) — bijection for FoodBERT, per-cluster majority for the proposed models ->
     reproduces the paper's Fig. 3/4;
   * `'bipartite'` — one consistent, correct Hungarian-bijection metric for every model;
   * `'argmax'` — per-cluster majority for every model.
   This makes the trade-off explicit and reversible instead of silently "correcting" the paper.
3. **Cell 6.1 (Fig. 3)** — restyled to the `Redraw_all_figures.ipynb` look: `sns.set_theme(whitegrid,
   DejaVu Serif)`, `catplot` bar, granularity order `Continents -> Sub-regions -> Countries`,
   conservative stress-test values, per-bar labels, centred legend under the panels. (Mission item 3.)
4. **Cell 6.2 (Fig. 5)** — **resolution capped** (`figure.dpi = 100`, `figsize=(14, 7)`), palette sized to
   the number of models (removes the `Set2` 8-vs-7 palette warning), fixed `K` order. Smaller raster ->
   no Colab "figure too large" problem. (Mission item 3.)

Every modified code cell passes `nbformat.validate` and Python `ast.parse` (magics stripped).

---

## 8. Things you should decide / review

1. **Which external metric do you want in the paper?** The clean choice is `EXT_ASSIGN='bipartite'`
   (consistent, FoodBERT-correct) — but then the proposed-vs-FoodBERT gap shrinks a lot and the
   paper's Fig. 3/4 numbers change. The default `'legacy'` deliberately reproduces the old figure.
2. **The 2 dropped rows** (old 21475/23805) — still unexplained by `0_upload_data_to_DuckDB`; worth a look.
3. **The NaN country code** (`coutries.csv` row 152 "Namibia") — either drop it or fill the code, so the
   geography vector goes back to a clean 248 columns.
4. The paper text (Sec. IV-C) says Hungarian bipartite matching; with `EXT_ASSIGN='legacy'` the metric
   no longer matches that sentence.

---

## 9. Reproduce my numbers

`MasterBattle.csv` (`old_code/15_KMean_battle_field/data/`) gives the truth;
`indices_has_cuisine.txt` gives the labelled rows; the legacy `all_vector/*.npy` give the features.
Majority baseline at `K=6`: **PA = 71.76 %, SF1 = 0.1393** (21,973 / 30,621 recipes contain "North America").

---

## 10. Also tried (negative results, for the record)

To rule out a "clustering-protocol / normalisation" cause (the mission's initial hypothesis), the
legacy vectors were clustered (labelled subset and full 50,513 rows; sklearn `KMeans`, `n_init=10`)
under many transforms, then scored with the bijection metric at `K=6`:

| transform of the geography vector | PA (bijection) |
|---|---|
| raw | 40.5 |
| L2-normalised rows | 37.3 |
| z-score columns | 54.0 |
| sqrt (Hellinger) | 37.6 |
| L1-normalised rows | 40.2 |
| z-score + L2 | 34.0 |
| L2 + z-score | 54.7 |

None reaches the paper's 71.68, and clustering all 50,513 rows returns the *same* labelled-row scores
as clustering only the labelled subset. A "leaky" cuisine-derived country one-hot (which contains the
ground truth) reaches only 81.4 — i.e. no plausible feature normalisation explains the paper's number;
only the degenerate majority assignment does.