# Mission & Specification: Redraw Figures & Update ICOCO Notebook

## Overview
- **Reference Style Notebook:** `old_code/17_result_and_discussion_/Redraw_all_figures.ipynb`
- **Target Notebook:** `final/ICOCO_notebook_Colab.ipynb`
- **Paper File:** `ICOCO IEEE eXPress.pdf`
- **LLM Country Matching Data:** `6_confirm_LLM_performance_number`

---

## 1. Objectives & Visual Style Updates

### A. Internal Metrics Ablation Figure Style
1. **Remove Error Bars (Candlesticks):**
   - The current Colab notebook plots error bars (black vertical lines/candlesticks) due to multiple random seed runs.
   - **Remove all error bars** (`errorbar=None` or `ci=None` in Seaborn/Matplotlib) to make the bar plot clean and flat, matching the paper's style.
2. **Color Palette & Baselines:**
   - Change the **FoodBERT Baseline** bar color to **dark charcoal / black (`#2D2D2D` or `#000000`)**.
   - Keep color consistency across all model variants (Entity Only, Geo Only, No Entropy, Odor+Entropy, Odor Only, FoodBERT Baseline, Our Full Fusion).
3. **Cluster Granularities ($K$):**
   - Set $K \in \{6, 31, 248\}$ (assuming all countries from the Zenodo dataset).

---

### B. External Metrics Ablation Figure Style
1. **Add External Ablation Figure:**
   - Add the external metrics plot comparing **Partial Match Acc (%)** and **Soft Macro F1 Score**.
2. **Color Consistency:**
   - Set the **FoodBERT Baseline** bar color to **black / dark charcoal**, identical to the internal metrics plot.
3. **Cluster Granularities ($K$):**
   - Set $K \in \{6, 31, 207\}$ (207 corresponds to the matching countries extracted from the LLM outputs in `6_confirm_LLM_performance_number`).

---

### C. General Figure Formatting
- **Typography & Grid:** Use `serif` font (Times New Roman / LaTeX style) with clean horizontal gridlines.
- **Axis Labels:** Ensure X-axis is labeled as `Number of Clusters (K)`.
- **Legend Location:** Position the combined legend neatly at the bottom of the multi-panel figure.

---

## 2. Reproducibility Assessment

### Question:
*If `labeled.csv` is added into GitHub Releases, can the paper's results be fully reproduced?*

### Answer & Analysis template:
**Yes, but with additional requirements.**

1. **What `labeled.csv` Solves:**
   - It freezes the LLM outputs and labels, avoiding discrepancy caused by non-deterministic LLM generations across different runs.
   - It allows downstream clustering and evaluation scripts to reliably compute metrics for $K \in \{6, 31, 207/248\}$.

2. **Required Additional Steps for 100% Reproducibility:**
   - **Fixed Seeds:** Hardcode random seeds (`random_state` / `np.random.seed`) for all clustering algorithms (e.g., K-Means, UMAP, t-SNE).
   - **Dependency Management:** Provide a clear `requirements.txt` or `environment.yml` specifying exact library versions (e.g., `scikit-learn`, `pandas`, `numpy`).
   - **Relative File Paths:** Ensure the notebook uses relative paths or downloads `labeled.csv` automatically from GitHub Release assets.

---

## 3. Required Deliverables
1. Modified `final/ICOCO_notebook_Colab.ipynb` with all figures redrawn according to the specs above.
2. Summary execution log verifying that all $K$ values and bar colors match the target paper style in one md file in the folder `final` + answer my question in that file