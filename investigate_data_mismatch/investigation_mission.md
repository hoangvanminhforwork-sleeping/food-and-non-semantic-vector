Your mission: Investigate data mismatch, number mismatch

# 1. About the research

This project is for cleaning the old code (My old code was not clean after all, I cannot understand the flow, and there are some mismatchs in data, number...)

Please do not update the folder old_code into the "agents.md", in project structure part. The old_code will not be removed soon

I used LLM to clean data

Mostly, I have system prompt and user prompt

System prompt is the same in every request

There are file txt 

And one prompt is sent 3 times, in fixed temperature (I do not remember, maybe from 0.2 to 0.5)

To check whether LLM is consistent with the output or not

The progress:
- From messry data: Example "100g of unpeeled apple and salt"

- We have standardized data: ["Apple", "Salt"] through LLM

- Then:

+) "Apple": ["Domestica...."] // scientific name

+) Salt: [] // no scientific name

- Return to standardized name, we used those name to query FoodUSDA to get the information about nutrients

- We just take the name, we do not care about how much for each nutrient

# 2. Problem: Data mismatch

- If the difference is small, that is okay, maybe DuckDB, Python version...etc can lead to different numbers

- After all, the csv is malformed

- Here is the detailed situation:

## 2.1 Recipe Zenodo

- Address:

+) 2_data_exploratory\explore_zenodo_recipe_note.md

+) 2_data_exploratory\explore_zenodo_recipe.ipynb

+) 4_AI_cleaned_data_exploratory

+) Folder in old code should investigate first:
old_code\03_ingredient_extraction
old_code\09_gold_data_create

- There are:
+) 50,511 dishes in new code, maybe the first line...the description of this dataset said there are only 50,510 dishes only

+) Number of messry ingredients mismatch: 3586 (contradicts to my expectation: 3500), and the number 4286. What is the difference between these three numbers?

## 2.2 FooDB and FoodUSDA

- How many categorization of Food in USDA? How many odors in FooDB? Do they contradict with new code?

+) Old code: old_code\09_gold_data_create, old_code\13_FoodUSDA_queryable

+) New code: 2_data_exploratory. Or you can check DuckDB database by yourself: ICOCO.duckdb

## 2.3 Performance of LLM

There are several metrics: Self-consistency, Input coverage, Adjusted Input coverage, Queryability

- Old code: old_code\utils, old_code\08_LLM_data_validation, old_code\09_gold_data_create

Investigate whether the numbers are different with:
2_data_exploratory

4_AI_cleaned_data_exploratory

# 3. Output and Constraint
Output: An md file for reporting, in the folder: investigate_data_mismatch

Constraint: You can run the code in the folder old code, but try not to do that. Only do it when needed

Only write new code if necessary, this is basically the task for reading and analyze only. I do not think you need to write code

If something confused, you can contact me any time. You can read other files, but I think for the address I provided...they are mostly enough

Recommended approach:
- First stage: Confirm files exist
- Second stage: Investigate the RecipeZenodo
- Third stage: Investigate FooDB and FoodUSDA
- Fourth stage: Write report