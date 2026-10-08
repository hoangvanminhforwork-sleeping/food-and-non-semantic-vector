# Battle Generation — Task Note

<h1>1. Mission</h1>

<blockquote>
Create the <b>battle</b> for the ICOCO (<i>IEEE eXPress</i>) K-means experiment, following the
structure of <code>8_create_battle/BattleGeneration.ipynb</code> and the legacy code
<code>old_code/15_KMean_battle_field</code>. <b>Do not run any unsupervised learning yet</b> — only
prepare the ground-truth geographic tags (labels) for the post-hoc evaluation.

Deliverable: a polished, idempotent, SQL-first notebook that reproduces the legacy battle artifacts
inside DuckDB:

| Legacy file | New DuckDB table |
|---|---|
| <code>09_gold_data_create/data/gold_geography.csv</code> | <code>battle.cuisine_gold</code> |
| <code>15_KMean_battle_field/data/MasterBattle.csv</code> | <code>battle.master_battle</code> |
| <code>15_KMean_battle_field/battle/indices_has_cuisine.txt</code> + <code>indices_no_cuisine.txt</code> | <code>battle.recipe_status</code> |
</blockquote>

<h1>2. What I did, what troubles I encountered, how I resolved them</h1>

<h2>2.1 Recon — paper, legacy code, database</h2>
- <font color="blue">I read the paper (<code>ICOCO IEEE eXPress.pdf</code>) and extracted the key
numbers: 50,510 recipes, 30,621 geographically labeled / 19,889 unlabeled, 116 raw cuisine tags,
207 countries / 31 sub-regions / 6 continents, post-hoc evaluation at K = 6 / 31 / 207 via
Hungarian bipartite matching.</font>
- <font color="blue">I read the legacy pipeline (<code>MasterBattleGeneration.ipynb</code>,
<code>smallerBattleGeneration.ipynb</code>, <code>gold_geography.ipynb</code>) and inspected the new
DuckDB: <code>cuisine_normalize.output1/2/3</code> (116 rows each) exist, <b>no gold table exists</b>,
and the vector <code>.npy</code> files from <code>7_create_npy_files</code> are
<code>(50511, …)</code>, keyed by <code>silver.RecipeZenodo</code> <code>rowid</code>.</font>

<h2>2.2 The database was locked by a live kernel</h2>
- <font color="red">Consequence: PID 3778 (an <code>ipykernel_launcher</code> of the user's VS Code
session) held the only DuckDB connection — even a read-only connect failed with
<code>Conflicting lock is held</code>.</font>
- <font color="green">Resolve: I inspected everything through a byte-copy
(<code>cp ICOCO.duckdb /tmp/ICOCO_inspect.duckdb</code>). I never killed the user's kernel. When the
lock was later released (verified by a probe), I executed the real notebook against the real
<code>/workspace/ICOCO.duckdb</code> to create the <code>battle</code> tables.</font>

<h2>2.3 Reproducing the gold — the ≥ 2-of-3 majority vote</h2>
- <font color="blue">The mission points to the investigation report:
"old record-level match ≥2/3 ranged 96–98% per field"; the same majority-vote rule already exists in
the silver gold views (<code>silver.v_ingredient_gold</code>, <code>silver.v_scientific_gold</code>).</font>
- <font color="green">Resolve: gold = items appearing in ≥ 2 of the 3 replicas, per field
(country / sub-region / continent), computed in pure SQL. Validated entry-by-entry against the legacy
<code>gold_geography.jsonl</code>: 109 tags in both, <b>0 rows differing</b> (compared as sorted sets),
207 / 31 / 6 terms — the exact paper numbers.</font>
- <font color="orange">Note: list <i>ordering</i> differs from the legacy in 25 of 327 fields — the
legacy kept first-appearance order (a pandas <code>Counter</code> artifact) while we sort
alphabetically. The <i>sets</i> are identical, so every set-based metric is unaffected.</font>

<h2>2.4 Edge cases — "World" and two country-less gold tags</h2>
- The tag <b>World</b> got no votes (all 3 LLM replies were truncated / unparsable) and is naturally
excluded — satisfying "ignore the edge case World".
- <b>Southwestern</b> and <b>Asian Inspired</b> are kept in the gold with empty
<code>final_country_list</code> but non-empty sub-regions/continents.
- 7 raw tags (Authentic, Copycat, Creole, Fusion, Inspired, Kosher, World) receive no gold at all;
dishes whose only tags are these resolve to <code>no_geo_label</code>.

<h2>2.5 Empty-cuisine dishes were silently dropped by a naive UNNEST</h2>
- <font color="red">Consequence: <code>FROM base, UNNEST(Cuisine)</code> drops every dish with an
empty cuisine array — the master table would lose 18,926 rows.</font>
- <font color="green">Resolve: <code>LEFT JOIN UNNEST(b.Cuisine) AS t(tag) ON true</code> keeps all
<h2>2.6 The user's draft template was not executable as-is</h2>
- <font color="red">Consequence: the draft used pure-SQL cells with <code>vscode.languageId=sql</code>
but no <code>%%sql</code> magic — under the project's Python kernel those cells fail with
<code>SyntaxError</code> (proved by an nbconvert run).</font>
- <font color="green">Resolve: followed the accepted project convention
(<code>silver_curation.ipynb</code> / <code>upload_data.ipynb</code>): every SQL cell gets both
<code>%%sql</code> <i>and</i> <code>vscode.languageId=sql</code>, so it runs under ipykernel and still
renders as SQL in VS Code.</font>

<h2>2.7 jupysql only displays the last statement of a cell</h2>
- <font color="green">Resolve: every multi-query cell was split so each intermediate result
(gold tags, countries/regions/continents, tagged/labeled counts, …) is visible and independently
verifiable.</font>

<h2>2.8 Notebook-building hiccups</h2>
- <font color="red">Consequence: the editor tool truncated large writes mid-file (unbalanced triple
quotes, orphaned lines; <code>SyntaxError</code> on compile).</font>
- <font color="green">Resolve: rebuilt the generator in small chunks, <code>py_compile</code> +
<code>nbformat.validate</code>, and executed the notebook end-to-end via
<code>jupyter nbconvert --execute</code> (first on a DB copy, later on the real DB).</font>

<h1>3. Can I prove the task is done?</h1>

- The notebook executed end-to-end with <b>0 error cells</b> (checked the executed outputs beyond the
last statement of each cell).
- The <code>battle</code> schema now exists in <code>/workspace/ICOCO.duckdb</code>:

| Table | Rows | Key facts |
|---|---|---|
| <code>battle.cuisine_gold</code> | 109 | 207 countries / 31 sub-regions / 6 continents; identical to legacy gold (0 diffs) |
| <code>battle.master_battle</code> | 50,511 | 30,621 labeled / 19,890 unlabeled; spot-rows match legacy <code>MasterBattle.csv</code> |
| <code>battle.recipe_status</code> | 50,511 | <code>(id, status)</code>; ids exactly 1..50,511; 30,621 <code>has_geo_label</code> / 19,890 <code>no_geo_label</code> |

- Numbers reproduce the paper: 30,621 labeled; 207/31/6 vocabulary. The 19,890 vs the paper's 19,889
(and legacy's 19,892) is the documented CSV-parsing artifact (§50,511 vs 50,510 vs 50,513).
- Idempotent: every table is <code>CREATE OR REPLACE</code>; re-running the notebook rebuilds the same
tables without touching other data.

<h1>4. Things you may want to review</h1>

<h2>4.1 Two gold tags carry sub-region/continent but no country</h2>
<code>Southwestern</code> and <code>Asian Inspired</code> resolve to <code>no_geo_label</code> yet
still contribute sub-region/continent info via <code>master_battle.final_sub_regions/continents</code>.
They are relevant for the K = 31 / 6 partial-match evaluation.

<h2>4.2 Country labels are not ISO-normalized</h2>
Confirmed by <code>4_AI_cleaned_data_exploratory</code>: only 166 of the 207 gold country strings
exist in <code>silver.countries</code>. If external alignment ever needs to compare gold country labels
with the geo-vector <b>columns</b> (which follow <code>silver.countries</code>), a string-mapping pass
is required.

<h2>4.3 Row alignment with the vector files</h2>
<code>battle.id = silver.RecipeZenodo.rowid + 1</code>, and the four <code>.npy</code> matrices in
<code>7_create_npy_files</code> are in <code>rowid</code> order, so <code>npy_row = id - 1</code>.
If the vectors are ever regenerated with a different ordering, the battle ids must be re-derived.

<h2>4.4 Deliberate list-ordering change</h2>
<code>final_country_list</code> etc. are alphabetically sorted (deterministic) instead of legacy
first-appearance order. Value sets are identical; only ordering differs (25/327 fields).
50,511 dishes (tag is NULL for empty arrays). Verified: 50,511 rows, ids 1..50,511 consecutive.</font>