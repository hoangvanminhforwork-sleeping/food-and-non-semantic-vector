This is the general guideline to help you understand about the context and environment

I am talking with you through Zoo code in VS code

Updated at: ...

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
├── agents.md
└── data
    ├── FoodUSDA_latest
    ├── FoodUSDA_legacy
    ├── RecipeZenodo.csv
    ├── RecipeZenodo_final.csv
    ├── RecipeZenodo_utf8.csv
    ├── coutries.csv
    └── foodb_2020_04_07

# 3. Preference

I have completed the research already. So yes, mostly I provide you some..."hints" about what I...suffered technically. Such as delimeter of csv, version...however, you still has to run commands to confirm. If you checked then find out something contradicts to my "hints", please tell me, so I will look at the old code

This project is just to make the code cleaner and reproducible, therefore, prioritize SQL notebooks

And please, I usually "mix" between "objectives" and "implementation guide". When I said something about implementation approach, that is just "I have run this, in old folder, and it worked", like a recommended approach. No need to strictly follow my "hint"

**Reproducibility**: For SQL, always use "if not exist", "create or replace"...etc, so I can run the notebook multiple times without changing the state of the data so much

# 4. Fixed jobs

Mostly, whenever I ask you to do something to me, document is always needed

You can think of...there are always two phases

Phase 01: You do the tasks

Phase 02: Document the tasks

There are two kinds of documents:

- Note what you did for the task

- Update agents.md for syncing the project: Update the time and project structure only

You can read this folder to understand my preference style in documenting and coding: /workspace/1_silver_create

# 5. Other notes

## a DuckDB
DuckDB database can have only one connection at the same time only (It is not like MySQL)

DuckDB is optimized for OLAP (Analysis), not OLPT (Reading and Writing in applications)

## b. VS code cache
Me (as a user) can look at the VS code GUI only

Sometimes, VS code shows me the old version of some files (the problem related to cache)

Please confirm through the terminal that the file has been modified and tell me (So I can reload the window to see the modified file)

## c. Tree command

Even though Dockerfile, requirements or Devcontainer does not have tree

But I have installed tree command into the container

I hope this can help you understand my project better

## .kilo folder
Just ignore it