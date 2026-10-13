# Video Script — Reproducing the ICOCO paper (one huge Colab notebook)

> Narration script for a walkthrough video of `ICOCO_notebook_Colab.ipynb`.
> One numbered section per **code cell**, in notebook order (85 code cells).
> The paper: *"Reducing Generative AI Bias in Food Recognition Using a
> Multi-Dimensional Culinary Representation Framework"*.

# 1. Global configuration of the Colab run (GLOBAL CONFIG)

- This is the control cell: it defines every constant and switch the rest of the notebook reads.
- Paths: `DATA`, `ARTIFACTS`, `OUTDIR` all point to `/content`, and `DB` is the single DuckDB file `/content/ICOCO.duckdb`; `FOODBERT_NPY` is the FoodBERT embedding file.
- `GH_RELEASE` is the GitHub Release URL that hosts all the heavy / blocked inputs (raw archives, merged LLM files, mapping JSONs, GBIF caches and the pre-computed `.npy` vectors), because the original servers block programmatic downloads.
- Scope flags decide how much we (re)compute: `REGENERATE_VECTORS` (rebuild the 304-D vectors instead of reusing the shipped ones), `RUN_FOODBERT` (recompute the FoodBERT baseline on the GPU), `RUN_KMEANS` (run the K-means "battle" on the GPU), and `KMEANS_DEVICE='cuda'`.
- `EXT_ASSIGN` selects the external cluster→label rule used later in section 5.3: `'bipartite'` = the strict Hungarian bijection the paper describes (the default, and the only rule that keeps the ablation variants distinguishable); `'argmax'` and `'legacy'` are documented as degenerate fallbacks.
- `LLM_TASKS` lists the four LLM tasks — `ingredient_standardize`, `scientific_name`, `cuisine_normalize`, `odor` — one merged file each.
- `FIG3_SOURCE` chooses whether the external figures (6.1/6.2) use the paper's published reference values or the values the notebook itself computes.
- The cell ends by printing the key settings so the run log records which mode we are in.

# 2. Install the libraries Colab does not ship

- A single `pip` line installs only what's missing: `duckdb`, `duckdb-engine`, `jupysql` and `requests`.
- Everything else the notebook needs — numpy, pandas, scipy, matplotlib, seaborn, plotly, scikit-learn, torch, transformers — is already preinstalled on Colab, so reinstalling would only slow the run down and risk version clashes.

# 3. Sanity-check the DuckDB engine

- A one-liner that opens an `ICOCO.duckdb` file, then immediately closes it.
- Purpose: fail fast. If DuckDB installed correctly, this cell passes; if not, we find out before spending time on downloads.

# 4. Download the Zenodo recipe corpus

- Downloads the raw recipe dataset (`dataset.csv`) from Zenodo record `15169428` using a streaming `requests.get`, with a `tqdm` progress bar.
- It is idempotent: if the file already exists it prints "already downloaded" and skips, so re-running the notebook never re-downloads.
- This CSV is the source of the 50,511 recipe rows that everything downstream is aligned to.

# 5. Download and extract FooDB and both FoodUSDA releases

- Defines two helpers, `_download` and `_extract`, that are idempotent: they skip a file if it already exists and non-empty, and skip extraction if the target folder already exists.
- `_download` streams the file with a progress bar and deletes a partially downloaded file if the transfer is interrupted, so a broken download never gets treated as valid.
- `_extract` unpacks `.zip` or `.tar.gz` into a temp folder, then moves the real inner folder into place; if extraction fails it deletes the corrupt archive so the next run retries cleanly.
- It then pulls the three reference datasets from the GitHub Release: FooDB (`foodb_2020_4_7_csv.tar.gz`), FoodUSDA foundation/latest and FoodUSDA SR legacy.
- It finishes by printing the first few entries of each extracted folder to confirm the layout is correct.

# 6. Define the artifact downloader (`fetch`) — the GitHub Release helper

- A small `fetch(name, dest)` function that downloads one file from the GitHub Release into `/content`.
- Idempotent by design: if the file is already present it says "present" and returns immediately; otherwise it streams with a progress bar and raises on HTTP errors.
- Every pre-computed input later in the notebook is pulled through this single helper, which keeps the download logic in one place.

# 7. Fetch all pre-computed artifacts from the GitHub Release

- Declares `ARTIFACTS_TO_FETCH`: the four merged LLM files, the ingredient→category and ingredient→nutrient mapping JSONs, the cooking-reaction config, the country reference, the GBIF caches, and the four pre-computed `.npy` vectors (geography, entity, odor, odor-entropy).
- Loops over that list calling `fetch()`, wrapped in a `try/except` so an optional missing artifact is reported but does not abort the notebook — it can simply be (re)built later.
- Note: `foodbert_Zenodo.npy` is deliberately **not** fetched; that baseline is rebuilt here in section 3.4.

# 8. Verify the data: hard vs soft requirements

- Builds the list of required inputs: the four merged LLM files, both mapping JSONs, the cooking config, the country reference, plus the four `.npy` vectors.
- Prints a present/missing report (`OK` / `MISS`) for each, then separates them into "hard" requirements (the non-`.npy` inputs) and "soft" ones (the `.npy` vectors).
- Design rationale: if a pre-computed vector is missing it is simply ignored here and rebuilt in the vector sections, so a missing or foreign `.npy` never breaks the top-to-bottom Colab flow. Only a genuinely missing non-vector input raises a warning.

# 9. Open the DuckDB database (jupysql `%sql`)

- Loads the `sql` extension (`%load_ext sql`), sets the result display limit to 50 rows, and connects the notebook to `duckdb:////content/ICOCO.duckdb`.
- From this point on, `%sql` / `%%sql` cells and the Python side share this one connection.

# 10. Close the connection to flush the write-ahead log

- Closes the DuckDB connection so the WAL (write-ahead log) is flushed to the database file.
- This is a deliberate discipline step: closing and reopening guarantees a clean, fully-persisted database before we start writing tables.

# 11. Reconnect to the fresh database

- Reopens the same `ICOCO.duckdb` connection right after the flush.
- One clean `%sql` connection is now active for the rest of the run.

# 12. Reuse the single `%sql` connection in Python

- DuckDB permits only ONE connection to a file at a time, so instead of calling `duckdb.connect()` again it grabs the connection jupysql already opened, via `ConnectionManager().current...connection`.
- It then tunes the engine: `SET memory_limit = '8GB'` and `SET threads = 12`, giving the heavy SQL work the memory and parallelism it needs.
- The `raw` handle created here is what all later Python cells use for `.execute(...)`.

# 13. Create the database schemas (section 1.1)

- Creates the twelve schemas that organise the whole pipeline: `zenodo`, `foodb`, `foodusda_latest`, `foodusda_legacy`, and one per LLM task (`ingredient_standardize`, `scientific_name`, `cuisine_normalize`, `odor`), plus `gbif`, `metrics`, `battle` and `result`.
- Uses `CREATE SCHEMA IF NOT EXISTS`, so the cell is idempotent and safe to re-run.

# 14. Load the reference datasets straight into final tables (section 1.2)

- Loads the country-code reference (`coutries.csv`) into `zenodo.countries` with a fixed column order, because the geography vector later indexes countries by their CSV row order.
- Loads the FooDB flavour table (only the columns we need: `id, name, flavor_group, category`).
- Loads the two FoodUSDA releases — foundation/latest and SR legacy — each into `food_category`, `food`, `nutrient` and `food_nutrient` tables, keeping only the emitted columns.
- Following the project's "no bronze/silver layering" rule, the CSVs go directly into their final tables with `CREATE OR REPLACE TABLE`.

# 15. Preprocess the raw Zenodo CSV so DuckDB can read it (section 1.3)

- The raw file is cp1252-encoded, contains control characters, and has unescaped double quotes, all of which break DuckDB's strict CSV parser (and this DuckDB version has no `encoding` option).
- Step 1 converts the file to UTF-8 and strips control characters (e.g. NUL) with a regex.
- Step 2 re-emits a properly quoted semicolon-delimited CSV with Python's `csv` module, and **drops malformed rows** (any row that doesn't have exactly 18 fields).
- Result: a clean `zenodo_final.csv` that DuckDB can load reliably. This is the step responsible for the 50,511-vs-paper-50,510 row-count nuance.

# 16. Load `zenodo.recipes` (section 1.3)

- Creates the main recipes table with `read_csv` on the cleaned file using `delim=';'`, header, quote `"`, `ignore_errors=true`, `null_padding=true` and `parallel=false` (single-threaded to keep parsing deterministic).
- This is the canonical row order — `rowid` here defines the index that every `.npy` vector row will match.

# 17. Parse the list-valued columns of the recipes (section 1.3)

- Reads the raw columns back into pandas and turns the text-stored Python literals into real lists using `ast.literal_eval` — for `Category`, `Cuisine` and `Cooking Methods`.
- `Ingredients` is special: it's a list of dicts, so a custom parser keeps only the `ingredient` name of each entry.
- Writes the curated frame back over `zenodo.recipes` (`CREATE OR REPLACE`), so after this cell all list columns are proper DuckDB lists, ready to `UNNEST`.

# 18. Load the first two LLM output tables — ingredient + scientific name (section 1.4)

- Reads the two merged LLM files (`ingredient_standardize_merged.jsonl`, `scientific_name_merged.jsonl`) with `read_json_auto` (newline-delimited).
- Each task keeps the "input + 3 replicas" shape: `custom_id, input, output1, output2, output3`.
- Prints row counts per table so we can confirm both loaded.

# 19. Load the other two LLM output tables — cuisine + odor (section 1.4)

- `cuisine_normalize` answers are JSON objects like `{country_list, sub_region_list, continent_list}`, and `odor` answers are JSON lists like `["balsamic","nutty",...]`, so they can't go in as raw text — they are parsed in Python.
- For each of the three replicas (`output1/2/3`) it parses the JSON, tolerates fenced code blocks and malformed answers (falling back to empty lists), and writes a typed table with list columns.
- `od` variants go into `cuisine_normalize.output1/2/3`; odor variants into `odor.output1/2/3`.

# 20. Verify the LLM input volumes (section 1.5)

- A quick SQL report comparing the size of each input set: the recipe ingredient vocabulary, the ingredient LLM inputs, the scientific-name LLM inputs, the cuisine tags, and the odor inputs.
- Purpose: confirm the LLM work covers the expected inputs before we start computing quality metrics.

# 21. Parsing layer for Task A — `metrics.v_ingredient_runs` (section 1.6)

- Builds a view that unpivots `ingredient_standardize.runs` into one row per (input, replica) by `UNION ALL`-ing `output1/2/3`.
- For each row it trims whitespace and classifies the answer as `parsed_kind = 'list'` when it is a valid JSON array, otherwise `'string'`.
- If it's a list it produces a cleaned, lower-cased `value_list`; if it's a plain string it produces a lower-cased `norm_str`.
- This view is the common, normalized substrate that the Self-Consistency / Input-Coverage metrics and the gold aggregation read from.

# 22. Parsing layer for Task B — `metrics.v_scientific_runs` (section 1.6)

- Exactly the same normalization applied to the `scientific_name` task: unpivot the three replicas, classify `list` vs `string`, and emit `value_list` / `norm_str`.
- Keeping the two tasks as parallel views is what lets the exact same metric SQL be reused for both.

# 23. Report `parsed_kind` distribution per task (section 1.6)

- A quick count of how many LLM answers parsed as JSON lists vs fell back to plain strings, for Task A and Task B.
- This is a diagnostic: a high share of "string" answers means the model kept returning unstructured text, which affects the metrics.

# 24. Self-Consistency / Input-Coverage / Adjusted-Coverage for Task A (section 1.7)

- Canonicalizes each replica per input, then computes three LLM-quality metrics over the inputs.
- **Self-Consistency (SC)**: share of inputs where at least two of the three replicas agree (a canonical answer seen ≥ 2 times). Paper target ≈ 98.41%.
- **Input-Coverage (IC)**: share of inputs where at least one ingredient element appears in ≥ 2 of the 3 replicas — i.e. the answer isn't just empty noise. Target ≈ 97.17%.
- **Adjusted-Coverage (AC)**: coverage that also credits inputs where all replicas were empty, so "consistently empty" still counts. Target ≈ 99.30%.
- All three are reported as counts and percentages in one row.

# 25. SC / IC / AC for Task B (taxonomic extraction) (section 1.7)

- The same metric computation as the previous cell, but run over `metrics.v_scientific_runs`.
- Targets from the paper: SC ≈ 82.34%, IC ≈ 61.36%, AC ≈ 98.74% — the taxonomic task is inherently harder than ingredient standardization.

# 26. Gold aggregation for Task A — `metrics.v_ingredient_gold` (section 1.8)

- Builds the "gold" answer for each input by **majority vote**: an ingredient is kept only if it appears in at least 2 of the 3 replicas (`cnt >= 2`).
- Aggregates the surviving elements into a single `final_list` per `custom_id`.
- This ≥ 2/3 rule mirrors the legacy gold-file construction.

# 27. Gold aggregation for Task B — `metrics.v_scientific_gold` (section 1.8)

- The identical majority-vote (≥ 2/3) rule applied to the scientific-name task, producing `final_list` of accepted taxon names per input.

# 28. Compare LLM inputs vs inputs with a gold output (section 1.8)

- A diagnostic that compares the total number of inputs in the `runs` table against how many inputs produced a non-empty gold output.
- Deliberately counts inputs from the *runs* table, not from the gold views — the note explains that counting the gold views could never expose a gap, because gold only keeps rows that reached consensus, so it always printed `n_inputs == with_output`.

# 29. Materialize the final LLM tables — `*.final` (section 1.9)

- Joins each task's `runs` table with its gold view to produce the final per-input record: `custom_id, input, output1, output2, output3, final_output`.
- Creates `ingredient_standardize.final` and `scientific_name.final` — the "one row per LLM input, with the three replicas and the voted answer" tables the paper describes.
- Ends with a `LIMIT 5` preview so we can eyeball the shape.

# 30. Reference universe #1 — `metrics.v_ref_food_foundation` (section 1.9)

- Builds the strict USDA reference set from `foodusda_latest.food`, keeping only rows with `data_type = 'foundation_food'` (469 rows).
- Casts every field to `VARCHAR` and adds a lower-cased `lower_desc` column, because queryability is a case-insensitive substring test against the description.

# 31. Reference universe #2 — `metrics.v_ref_food_full` (section 1.9)

- The full reference universe actually used by the paper: foundation-food (latest) **UNION ALL** the SR legacy foods = 8,262 rows.
- Same columns and lower-cased description, so both universes can be plugged into the same queryability query.

# 32. Report the size of the two reference universes (section 1.9)

- Prints the row count of `v_ref_food_full` and `v_ref_food_foundation` so the 8,262 vs 469 distinction is explicit in the log.

# 33. Queryability for Task A against the full paper universe (section 1.9)

- For every input with a non-empty gold list, checks whether **at least one** voted ingredient occurs as a substring of some USDA description (`lower_desc LIKE '%'||elem||'%'`).
- A recipe counts as queryable if any of its ingredients is grounded; the cell reports `with_output`, `queryable` and the percentage.
- Expected result: **71.20% = 11,765 / 16,524**, matching the paper.

# 34. Queryability for Task A against the foundation-only universe (section 1.9)

- The identical queryability rule, but only against the mission-strict `foundation_food`-only view.
- Same rule, smaller reference set, so it gives a lower number: **46.02% = 7,604 / 16,524** — showing how strongly queryability depends on the reference universe.

# 35. Reuse the cached GBIF species matches (section 2.1)

- Prints that GBIF species matches are **reused from the release cache**, so there are no live API calls and therefore no rate-limit risk.
- Counts the number of cached species in `gbif_species_match.jsonl`.

# 36. Load `gbif.species_match` into DuckDB (section 2.1)

- Materializes the GBIF cache into a table with `scientific_name` and a boolean `is_queryable` flag.
- Reports how many cached names there are and how many actually matched in GBIF.

# 37. Queryability for Task B (GBIF grounding) (section 2.2)

- The taxonomic analogue of the Task A queryability test: an input is queryable if at least one voted scientific name exists in `gbif.species_match` with `is_queryable = TRUE`.
- Expected result: **93.29% (2,253 / 2,415)** — the taxonomic names ground very well in GBIF.

# 38. Reuse the cached ingredient→country map (section 2.2)

- Prints that the ingredient → candidate-countries mapping is also reused from the release cache.
- This mapping is what turns an ingredient's GBIF species occurrences into a country signal for the geography vector.

# 39. Load `gbif.dirty_ingredient_country_mapping` (section 2.2)

- Reads the cached JSON (`{input, country_list}` records) into a pandas frame and materializes it as `gbif.dirty_ingredient_country_mapping` in DuckDB.
- Prints the row count. This table is the backbone of the geography vector two cells later.

# 40. Sanity-check the inputs to the geography vector (section 3.1)

- Prints row counts for `zenodo.recipes`, `zenodo.countries` and `gbif.dirty_ingredient_country_mapping`.
- A quick "are the inputs to the geo vector the sizes we expect" check.

# 41. Build the long-format geography signal — `geo_long` (section 3.1)

- Explodes each recipe into its ingredient occurrences (`UNNEST(Ingredients)`), then joins each ingredient to its candidate countries via the GBIF mapping.
- Counts country occurrences per recipe and computes the row-normalized `share = country count / total country occurrences` of the recipe.
- It is deliberately faithful to the legacy Python logic: duplicates are kept, and the denominator counts every country occurrence returned by the mapping.
- Countries keep their CSV `rowid` as the column index, so the geography vector columns are stable.
- Output columns: `recipe_idx, country_idx, country_code, cnt, total, share`.

# 42. Report the geography signal shape (section 3.1)

- Prints the number of long rows, how many recipes actually have a geography signal, how many distinct country columns are used, and the total country occurrences.
- Confirms the geography signal is non-trivial before we build the dense matrix.

# 43. Build `recipe_geography_vectors.npy` (section 3.1)

- Reads `geo_long` into pandas and builds a dense `float32` matrix of shape `(n_recipes, n_countries)` — the 248-dimension geography part of the representation.
- Scatters each `share` into `geo[recipe_idx, country_idx]`, so every recipe row is a normalized country-distribution vector.
- If `REGENERATE_VECTORS` is False it simply loads and re-saves the shipped `.npy`.
- Prints the saved shape, dtype and file size (MB).

# 44. Verify the geography vector (section 3.1)

- Internal checks: how many rows have signal (row sum > 0) and that each row sums to at most 1 (tolerance 1e-5), confirming the shares are normalized.
- A spot check on `rowid = 0`: prints the recipe name, its ingredients, and the non-zero country shares, so we can sanity-check one concrete recipe.

# 45. Sanity-check the entity-vector inputs (section 3.2)

- Prints row counts for `zenodo.recipes` and `foodusda_latest.food_category`, the two inputs to the entity (food-category) vector.

# 46. Build the long-format entity signal — `ent_long` (section 3.2)

- Explodes recipe ingredients, maps each ingredient to its USDA category via `dirty_ingredient_category_mapping.json`, and joins to `food_category` to get the category index (`id - 1`, matching the legacy index building).
- Only categories that exist in `food_category` are counted, and unknown categories are filtered **before** the tally — so they leave both the numerator and the denominator, exactly like the legacy `if cat and cat in cat_to_idx`.
- Output: `recipe_idx, cat_idx, share` where `share = category count / total valid category occurrences`.

# 47. Preview the entity signal (section 3.2)

- A `SELECT * FROM ent_long LIMIT 8` to eyeball the long-format rows before building the dense matrix.

# 48. Report the entity signal shape (section 3.2)

- Prints the number of long rows, the number of recipes with an entity signal, and the number of distinct categories actually used.

# 49. Build `entity_vectors.npy` (section 3.2)

- Reads `ent_long` and builds the dense `float32` matrix `(n_recipes, n_categories)` — the 28-dimension entity/food-category part of the representation.
- Scatters each `share` into `ent[recipe_idx, cat_idx]`.
- Same reuse-or-rebuild switch (`REGENERATE_VECTORS`) as the geography vector.
- Prints shape, dtype and file size.

# 50. Verify the entity vector (section 3.2)

- Row-signal count and row-sum normalization check, plus a spot check on `rowid = 0` printing the non-zero category shares.

# 51. Pre-build the odor lookups: cooking reactions and the 27-odor axis (section 3.3)

- Executes `reaction_cooking.py` to get the canonical `cooking_reactions` config, then flattens it into `(method, reaction)` pairs (both lower-cased).
- Defines the fixed 27-odor axis (vegetable, herbaceous, fatty, … animal) in the exact order used by the legacy notebook, and asserts it is 27 long.
- Pushes both into DuckDB **TEMP** tables (`cooking_reactions`, `odor_axis`) that the odor SQL later joins against.

# 52. Build the odor gold map — `odor_gold_map` (section 3.3)

- Stacks the three odor replicas and, per input, keeps the odors that appear in ≥ 2 of the 3 replicas (majority vote), the same normalization as the legacy gold-data creator.
- The input text is `"<Nutrient> AND <Reaction>"`; a regex extracts the nutrient (part 1) and reaction (part 2).
- Output: `(custom_id, nutrient, reaction, odor)` — the dictionary that maps a nutrient × cooking reaction to the odors it produces.

# 53. Report the odor gold map shape (section 3.3)

- Prints the number of gold rows and the distinct counts of nutrients, reactions and odors, so we can see how rich the odor knowledge map is.

# 54. Preview the odor gold map (section 3.3)

- A `LIMIT 8` preview of `odor_gold_map` to read a few `(nutrient, reaction) → odor` entries directly.

# 55. Build the long-format odor signal — `odor_long` (section 3.3)

- Computes, per recipe, the **set** of nutrients from its ingredients and the **set** of reactions from its cooking methods (via the `cooking_reactions` map).
- Joins those nutrient × reaction pairs against `odor_gold_map` to collect the resulting odors (multiplicities kept).
- Row-normalizes into `share` and maps each odor to its axis index via the `odor_axis` table; odors outside the 27-axis get no column but still count in the denominator (legacy-faithful).
- Output: `recipe_idx, odor_idx, share`.

# 56. Report the odor signal shape (section 3.3)

- Prints the number of long odor rows and how many recipes have an odor signal.

# 57. Build `recipe_odor_vectors.npy` (section 3.3)

- Builds the dense `float32` matrix `(n_recipes, 27)` — the odor part of the representation.
- Scatters each `share` into `odo[recipe_idx, odor_idx]`, reusing-or-rebuilding per `REGENERATE_VECTORS`, then prints shape/dtype/size.

# 58. Verify the odor vector (section 3.3)

- Row-signal and normalization checks, plus a spot check on `rowid = 0` that also prints the recipe's cooking methods and the non-zero odor shares.

# 59. Build `recipe_odor_entropy.npy` (section 3.3)

- Defines the Shannon-entropy function verbatim from the legacy notebook: drop zero probabilities (to avoid `log2(0)`) and return `-Σ p·log2(p)`; returns 0 for an all-zero row.
- Applies it row-by-row over the odor matrix, producing one entropy value per recipe, cast to `float32`.
- Odor entropy is the 28th odor-related dimension — it measures how "spread out" a recipe's odor profile is.
- Saves to `recipe_odor_entropy.npy`, reuse-or-rebuild again, and prints shape/dtype/size.

# 60. Verify the generated vectors against the reference (section 3.5)

- For each of the four `.npy` files it compares the freshly generated array with the reference copy from the release.
- Reports shape mismatch, or the max and mean absolute difference, and an `identical (<=1e-6)` boolean.
- This is the reproducibility gate: it proves the notebook's vector reconstruction matches the shipped artifacts.

# 61. Prepare FoodBERT — install transformers and clone the repo (section 3.4)

- Installs `transformers>=4.40` and shallow-clones the FoodBERT repository (`chambliss/foodbert`) into `/content`.
- The repo is cloned only for fidelity with the reference implementation; the actual embeddings are produced with HuggingFace `AutoModel` (mean pooling), exactly as in the reference notebook.
- Idempotent: it skips the clone if the folder already exists.

# 62. Build the FoodBERT input text per recipe (section 3.4)

- Reads the raw `Ingredients_Raw` and `Instructions` columns straight from `zenodo_final.csv` (the same row order as `zenodo.recipes` and the `.npy` vectors).
- Concatenates them into one text per recipe: `"Ingredients: {...}. Instructions: {...}. "`.
- Prints how many recipes are to be encoded and shows an example prefix. Gated by `RUN_FOODBERT`.

# 63. Encode the FoodBERT baseline → `foodbert_Zenodo.npy` (section 3.4)

- Loads `chambliss/distilbert-for-food-extraction` (tokenizer + `AutoModel`) onto the GPU (falls back to CPU).
- Runs the recipes through the model in batches of 64 with `max_length=512`, and mean-pools the last hidden state over the attention mask to get a 768-dim vector per recipe.
- Saves the `(n_recipes, 768)` `float32` matrix to `foodbert_Zenodo.npy` — this is the semantic baseline that our non-semantic 304-D representation is compared against.

# 64. Inspect the cuisine tables before building gold (section 4.1)

- Checks metadata: (a) which cuisine-related tables exist (the three replicas in the `cuisine_normalize` schema), and (b) whether a gold cuisine table already exists.
- Expected: no gold table yet, so the next cell creates `battle.cuisine_gold`.

# 65. Build `battle.cuisine_gold` by majority vote (section 4.1)

- Stacks the three cuisine replicas and, per raw cuisine tag, keeps each country / sub-region / continent that appears in **at least 2 of the 3** replicas (the same ≥ 2/3 rule as the legacy gold geography file).
- FULL-OUTER-JOINs the three voted lists into one row per raw tag, then drops fully-degenerate tags whose assessment is empty (e.g. the truncated "World" tag).
- Output columns: `custom_id, input_text, final_country_list, final_sub_regions, final_continents`.
- Idempotent (`CREATE SCHEMA IF NOT EXISTS` + `CREATE OR REPLACE TABLE`).

# 66. Report the gold geography vocabulary (section 4.2)

- Prints the number of gold tags and how many of them have a non-empty country list.
- Expected: **109 gold tags**, the same 109 as the legacy gold.

# 67. Report the gold geography term counts (section 4.2)

- Counts the distinct terms the gold uses at each granularity.
- Expected — and matching the paper's Table I: **207 countries / 31 sub-regions / 6 continents**.

# 68. Report how many dishes carry a cuisine tag (section 4.2)

- Counts total recipes, recipes with at least one cuisine tag, and recipes with none.
- Expected: ~31,585 dishes with a tag, ~18,926 without.

# 69. Report the raw cuisine vocabulary (section 4.2)

- Counts the distinct raw cuisine tags at the element level across all recipes.
- Expected: **~116 raw tags** — exactly the input vocabulary that the `cuisine_normalize` LLM task received.

# 70. Report the labeled / unlabeled dish split (section 4.2)

- A dish is "geographically labeled" iff at least one of its cuisine tags resolves to ≥ 1 country in the gold (≥ 2/3 majority).
- Expected: 50,511 total, **30,621 labeled**, **19,890 unlabeled** (the paper reports 30,621 / 19,889 on 50,510; the ±1 difference is the CSV-parsing artifact).

# 71. Build the master battle table — `battle.master_battle` (section 4.3)

- Rebuilds the legacy `MasterBattle.csv` as a DuckDB table: it assigns a 1-based `id` to each recipe in `zenodo.recipes` rowid order (so `npy_row_index = id - 1`), then attaches the gold country/sub-region/continent lists resolved from the recipe's cuisine tags.
- Aggregates the per-dish values with `list_distinct(flatten(...))` so a dish's labels are the union over all its tags.
- Adds `country_count`, `sub_region_count`, `continent_count` helper columns and orders by `id`.

# 72. Verify the master battle split (section 4.3)

- Re-checks the labeled/unlabeled split from `battle.master_battle` — the same 30,621 / 19,890 as the paper.
- Confirms the master table is consistent with the earlier status check.

# 73. Preview the master battle table (section 4.3)

- `SELECT * ORDER BY id LIMIT 5` to confirm the first rows read like the legacy `MasterBattle.csv`.

# 74. Build `battle.recipe_status` (section 4.4)

- Collapses the master table into a single `(id, status)` table, where `status` is `'has_geo_label'` (≥ 1 country, usable for post-hoc evaluation) or `'no_geo_label'`.
- This is the split that the K-means battle uses to define the labeled set and the unlabeled subset.

# 75. Verify the recipe-status counts (section 4.4)

- Groups `battle.recipe_status` by status; expected 30,621 `has_geo_label` + 19,890 `no_geo_label` = 50,511 rows.

# 76. Verify the id alignment (section 4.4)

- Confirms the ids run exactly 1 … 50,511 with no gaps and no duplicates — the guarantee that vector row `i` matches `id = i + 1`.

# 77. Load the vectors and define the GPU K-means (section 5.1)

- Defines a self-contained GPU K-means (Lloyd's algorithm + k-means++ init) in PyTorch, with numerical-stability guards (`torch.cdist`, explicit nan/inf handling) and a `gpu_kmeans(X, k, seed)` entry point that does 3 random inits and keeps the lowest inertia.
- Loads the five feature matrices: `foodbert`, `geo`, `entity`, `odor`, and `entropy` (reshaped to a column).
- **Row-alignment guard**: the legacy FoodBERT artifact has 50,513 rows, two more than the curated 50,511, because it predates dropping recipes #21,476 and #23,806; the cell deletes those two legacy rows and then asserts every matrix has exactly 50,511 rows.
- Defines the seven "players" (models to compare): FoodBERT baseline, and the full fusion (`geo+entity+odor+entropy`) plus five ablations (no-entropy, geo-only, entity-only, odor-only, odor+entropy).
- Sets the shared display order and colour map so every figure uses the same bars/colours, then reads `battle.recipe_status` to build the labeled/unlabeled row index arrays.

# 78. Internal metrics: Silhouette / Davies-Bouldin / Calinski-Harabasz (section 5.2)

- Evaluates clustering **geometry** on two datasets: the full dataset and the unlabeled subset (`no_geo_label`).
- Loops over the models, over `K ∈ {6, 31, 248}`, and over 5 seeds, running `gpu_kmeans` each time.
- Computes silhouette (sub-sampled to 10k for speed), Davies-Bouldin and Calinski-Harabasz, and stores one row per (group, model, K, seed).
- Writes everything to `result.internal_metrics`. These are the "does the representation form clean clusters" numbers.

# 79. External cultural alignment: Partial-Match Accuracy + Soft-MacroF1 (section 5.3)

- Tests whether the clusters **align with real geography**: it runs the K-means on the labeled subset and then maps each cluster to a representative ground-truth label.
- `_assign` supports two rules: strict Hungarian bipartite matching (the paper's Eq. 7, used for every model when `EXT_ASSIGN='bipartite'`) or per-cluster majority.
- For each granularity (`K=207` countries / `K=31` sub-regions / `K=6` continents) and each seed it computes Partial-Match Accuracy (a prediction counts if it is in the ground-truth label set) and Soft-Macro-F1, writing `result.external_metrics`.
- It also computes the majority-class baseline and prints a **degeneracy guard**: if the variants differ by < 0.5 pp at some K, it warns that the cluster→label rule has collapsed the ablation and suggests `EXT_ASSIGN='bipartite'`.

# 80. Figure 3 — external alignment: FoodBERT vs the full model (section 6.1)

- Plots the conservative stress test for the two headline models: FoodBERT shown at (mean + SD) and the full model at (mean − SD), across the three granularities.
- `FIG3_SOURCE='paper'` plots the published Fig. 3 reference values (default here); `'computed'` plots the notebook's own `result.external_metrics`.
- Two panels: Partial-Match Accuracy and Soft-MacroF1, with serif fonts, horizontal gridlines and value labels — styled to match the legacy `Redraw_all_figures.ipynb`.

# 81. Figure 4 — external alignment ablation across all variants (section 6.2)

- The ablation version of the external figure: all seven variants plus FoodBERT, with x-axis = number of clusters `K ∈ {6, 31, 207}` and two panels (PA and SF1).
- Uses the same conservative stress rule (proposed variants at mean − SD, FoodBERT at mean + SD) and the same shared colour map, so the bars stay consistent with the internal figure.
- Also switchable between the published reference values (`PAPER_EXT_ABLATION`) and the computed results.
- This is the figure showing that the full model beats FoodBERT in accuracy at every granularity, while FoodBERT keeps an F1 edge at K=6 that reverses by K=31/207.

# 82. Figure 5 — internal geometry, full dataset and unlabeled subset (section 6.3)

- A 2×3 grid: rows = {Full Dataset, No Label Subset}, columns = {Silhouette ↑, Davies-Bouldin ↓, Calinski-Harabasz ↑}; x-axis = K ∈ {6, 31, 248}.
- Averages the seed runs into one flat bar per (group, model, K) so there are no error bars, with the Calinski-Harabasz panel on a log scale.
- Same shared palette and serif/horizontal-grid styling as the other figures, with a combined legend underneath.
- Shows the internal "quality of the geometry" result — including the interesting finding that a single property (e.g. geo-only) can beat the naive full concatenation on some internal metrics (dimensional interference).

# 83. LLM validation summary — Task A (section 6.4)

- Re-runs the Task A SC / IC / AC SQL as the final consolidated validation row for ingredient standardization.
- This is the notebook's self-contained reproduction of the LLM-quality numbers reported for Task A (SC 98.41% / IC 97.17% / AC 99.30%).

# 84. LLM validation summary — Task B (section 6.4)

- The same consolidated SC / IC / AC row for the taxonomic-extraction task (SC 82.34% / IC ≈ 61.36% / AC 98.74%).
- Together with the previous cell, this is the "LLM validation summary table" of the paper.

# 85. Close the database connection (section 6.5)

- The final cell: `%sql --close duckdb:////content/ICOCO.duckdb` cleanly closes the single DuckDB connection and flushes the write-ahead log.
- Ending with an explicit close keeps the database file in a consistent state, so a re-run starts from a clean, fully-persisted database.

