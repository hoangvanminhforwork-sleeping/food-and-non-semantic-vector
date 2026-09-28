# Agent Missions & Guidelines

## 1. Core Missions

Your primary objective is to execute two main tasks:

1. **Bronze Layer Ingestion:** Ingest raw JSON data from `data/GBIF_query/dirty_ingredient_country_mapping.json` into **DuckDB** under the `bronze` schema.
2. **Reproducible Verification:** Build clear, reproducible Jupyter notebook(s) to compute and verify the exact LLM performance numbers reported in the research paper.

---

## 2. Evaluation Metrics Definition

To evaluate the stability and validation accuracy of the LLM-based canonicalization process (mapping an input raw string $t$ to a candidate list $L$), compute the following four metrics:

* **Self-Consistency (SC):** Exact identity agreement across all runs. 
  * $t$ satisfies SC $\iff$ all 3 generated lists ($L_1, L_2, L_3$) are non-empty and identical after sorting ($L_1 = L_2 = L_3 \neq \emptyset$).
* **Input Coverage (IC):** Candidate frequency across runs. 
  * $t$ satisfies IC $\iff$ the union of its output lists contains at least one candidate entity appearing in $\ge 2$ out of 3 runs (frequency $\ge 2/3$).
* **Adjusted Coverage (AC):** IC extended to account for valid empty outputs. 
  * $t$ satisfies AC $\iff$ it meets **IC criterion** **OR** all 3 runs return empty lists ($L_1 = L_2 = L_3 = \emptyset$).
  * *Note:* Partial empty outputs (e.g., 2 empty runs and 1 non-empty run) fail the AC criterion.
* **Queryability:** Database groundability for AC-passing entries. 
  * An AC-passing candidate list is **Queryable** $\iff$ at least one candidate entity within its final aggregated list successfully matches a valid record in the reference database (USDA or GBIF).

> **Important Data Scope Constraint:** 
> For **USDA FoodData Central**, filter and restrict the reference database exclusively to `foundation_food`.

---

## 3. Target Numbers to Verify

Verify whether your script/notebook output matches the exact figures reported in the paper:

### Task A: Ingredient Standardization
* **Self-Consistency (SC):** `98.41%`
* **Input Coverage (IC):** `97.17%`
* **Adjusted Coverage (AC):** `99.30%`
* **Grounding (USDA - Foundation Food):** `71.20%` (Target: $11,765 / 16,254$)

### Task B: Taxonomic Extraction
* **Self-Consistency (SC):** `82.34%`
* **Input Coverage (IC):** `61.36%`
* **Adjusted Coverage (AC):** `98.74%`
* **Grounding (GBIF Species Matching):** `93.29%` (Target: $2,253 / 2,415$)

---

## 4. Reference Directories & Context

Analyze the legacy (and new) codebase to understand data schema, historical logic, and pipeline setup:

* `old_code/13_FoodUSDA_queryable`
* `old_code/10_geo_vector/GBIF_queryable_rate.ipynb`
* `old_code/09_gold_data_create`
* `4_AI_cleaned_data_exploratory`

> ⚠️ **Execution Rule:** 
> **DO NOT** execute scripts inside `old_code/`. Treat `old_code/` strictly as read-only reference material to understand the logic. Write clean, refactored, and reproducible code in your new workflow/notebooks.