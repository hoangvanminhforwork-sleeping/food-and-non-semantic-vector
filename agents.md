This is the general guideline to help you understand about the context and environment

I am talking with you through Cline in VS code

Updated at: 2026-09-27 16:26 UTC

Please run terminal command to check the time

# 1. Understand the environment

Here is the address of .devcontainer folder, you may need this one to explore about the library: /workspace/.devcontainer

You can run terminal commands to ensure all libraries are available

# 2. Explore the structure of the project

As I mentioned, I am talking to you through VS code agents platform

You may not see the entire project

Here is the current structure of the project: Just to look at structure only. Detailed missions will be provided later, mostly I specify the address of folder for each mission

.
├── .devcontainer
│   ├── Dockerfile
│   ├── devcontainer.json
│   └── requirements.txt
├── .gitignore
├── .kilo
│   ├── .gitignore
│   └── worktrees
├── 0_upload_data_to_DuckDB
│   ├── FoodDB and FoodUSDA adding.md
│   ├── VSCode_cache_and_notebook.md
│   ├── ZenodoDataset_handling_note.md
│   ├── ZenodoDataset_handling_process.md
│   └── upload_data.ipynb
├── 1_silver_create
│   ├── silver_curation.ipynb
│   └── silver_data_create_note.md
├── 2_data_exploratory
│   ├── explore_foodb.ipynb
│   ├── explore_foodb_and_foodusda_note.md
│   ├── explore_foodusda.ipynb
│   ├── explore_zenodo_mission.md
│   ├── explore_zenodo_recipe.ipynb
│   ├── explore_zenodo_recipe_note.md
│   └── general_FooDB_and_FoodUSDA_exploratory_mission.md
├── 3_add_AI_standardized_data
│   ├── add_AI_labeled_data_to_bronze.ipynb
│   ├── add_AI_labeled_data_to_bronze.md
│   └── add_AI_labeled_data_to_bronze_note.md
├── 4_AI_cleaned_data_exploratory
│   ├── AI_generated_data_exploratory.md
│   ├── AI_generated_data_exploratory_note.md
│   └── AI_generated_data_exploratory.ipynb
├── 5_confirm_methodology_number
│   ├── confirm_mission.md
│   ├── confirm_mission02.md
│   ├── confirm_mission02_note.md
│   ├── confirm_numbers.ipynb
│   └── confirm_note.md
├── 6_confirm_LLM_performance_number
│   ├── missions.md
│   ├── 01_bronze_ingestion.ipynb
│   ├── 02_LLM_performance_verification.ipynb
│   └── confirm_note.md
├── ICOCO.duckdb
├── agents.md
├── investigate_data_mismatch
│   ├── investigation_mission.md
│   └── investigation_report.md
└── data
    ├── FoodUSDA_latest
    ├── FoodUSDA_legacy
    ├── RecipeZenodo.csv
    ├── RecipeZenodo_final.csv
    ├── RecipeZenodo_utf8.csv
    ├── coutries.csv
    ├── cuisine_normalize
    ├── foodb_2020_04_07
    ├── FoodUSDA_query
    ├── GBIF_query
    │   ├── dirty_ingredient_country_mapping.json
    │   ├── gbif_match_cache.json
    │   └── gbif_species_match.jsonl
    ├── ingredient_standardize
    ├── odor
    └── scientific_name

# 3. Preference

I have completed the research already. So yes, mostly I provide you some..."hints" about what I...suffered technically. Such as delimeter of csv, version...however, you still has to run commands to confirm. If you checked then find out something contradicts to my "hints", please tell me, so I will look at the old code

This project is just to make the code cleaner and reproducible, therefore, prioritize SQL notebooks

And please, I usually "mix" between "objectives" and "implementation guide". When I said something about implementation approach, that is just "I have run this, in old folder, and it worked", like a recommended approach. No need to strictly follow my "hint"

**Idempotency**: For SQL, always use "if not exist", "create or replace"...etc, so I can run the notebook multiple times without changing the state of the data so much

# 4. Fixed jobs

Mostly, whenever I ask you to do something to me, document is always needed

You can think of...there are always fixed phases

Phase 01: You do the tasks

Phase 02: Document the tasks

There are two kinds of documents:

- Note what you did for the task

+) Basically: What you did? What troubles did you encounter? How do you resolve? Document in details!

+) And mostly, when I did not mention the address I want, the address is..the folder I told you in the mission. Write the file md in that

- Update agents.md for syncing the project: Update the time and project structure

Phase 03: Delete temp files
Clean up any temporary files and directories you created during this task before finishing

If you have any question, you can ask me any time

# 5. Other notes

## a DuckDB
DuckDB database can have only one connection at the same time only (It is not like MySQL)

DuckDB is optimized for OLAP (Analysis), not OLPT (Reading and Writing in applications)

Moreover, there should always two cells in top of every notebook used SQL: One for connection, and one for disconnection, to minimize the probability of data corruption

## b. VS code cache
Me (as a user) can look at the VS code GUI only

Sometimes, VS code shows me the old version of some files (the problem related to cache)

Please confirm through the terminal that the file has been modified and tell me (So I can reload the window to see the modified file)

## d. Tmp folder
Clean up any temporary files and directories you created during this task before finishing

Maybe there are md files from previous tasks, becareful with them, maybe like...checking the tmp/folder then do not name a tmp file with same name with one of existing files

## .kilo folder
Just ignore it