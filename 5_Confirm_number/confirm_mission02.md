# TASK: Investigate Geographic Number Mismatch & Update `confirm_numbers.ipynb`

## 1. Context & Objective
We are auditing and confirming data numbers for the dataset (Zenodo). Previous agents have completed parts of this pipeline, but there is a mismatch regarding the "geographic" numbers (labels vs. no-labels).

Your mission is to investigate this geographic number mismatch and update the Jupyter notebook `confirm_numbers.ipynb` with a new, detailed analysis section.

## 2. Key Rules & Constraints
1. **Target Column**: Focus on the `Cuisine` column in the Zenodo dataset (Data type: `list`).
2. **Standardized Data**: The data was standardized using LLM/voting mechanisms. Refer to directory `3_add_AI_standardized_data/`.
3. **Filtering Rule**: **IGNORE** the term `"world"` present in the raw dataset.
4. **Paper Calculation Logic**: Note that the final number of labels reported in the paper reflects the state **"AFTER STANDARDIZATION"** (i.e., after the voting process).

## 3. Investigation Steps & Data Sources
- **Mission File**: Read `5_Confirm_number/confirm_mission.md` and review all context files in `5_Confirm_number/`.
- **Reference Code / Logic**: Review previous pipeline logic in:
  - `old_code/04_geography_extraction`
  - `old_code/09_gold_data_create`
- **Database Analysis**: Query and analyze tables `cuisine_normalize.output1`, `cuisine_normalize.output2`, and `cuisine_normalize.output3` inside the DuckDB database `ICOCO.duckdb`.

## 4. Required Output / Deliverables
Update `confirm_numbers.ipynb` by adding a dedicated section that includes:
1. **Breakdown Table / Statistics**: Clear count of columns/items with **labels** vs. **no labels** before and after voting/standardization (excluding `"world"`).
2. **Mismatch Investigation Summary**: Concise explanation of why the geographic mismatch occurred between raw data and the paper's reported numbers.
3. **Verification Steps**: Show reproducible queries/code blocks used to arrive at these final numbers.