# Task Note — building `ICOCO_notebook.ipynb`

<h1>1. Mission</h1>

<blockquote>
Implement the single "huge" notebook proposed in <code>recommended_huge_notebook_structure.md</code>
at <code>/workspace/ICOCO_notebook.ipynb</code>: one Colab notebook that reproduces the ICOCO paper
end-to-end (raw data → LLM canonicalization + metrics → GBIF → 304-D vectors + FoodBERT →
K-means battle → figures). It is <b>dry code</b> — authored here, executed later by the user on Colab.
</blockquote>

<h1>2. What I did</h1>

<h2>2.1 Read the whole pipeline</h2>
- <font color="blue">I read the paper, the new folders <code>0…8</code>, and the legacy late stages
(<code>10_geo_vector</code>, <code>11_entity_vector</code>, <code>12_odor_vector</code>,
<code>15_KMean_battle_field</code>, <code>17_result_and_discussion_</code>, <code>FoodBERT_encoding_npy</code>).</font>
- I extracted the exact source of every relevant cell (metrics, vectors, battle) so the notebook reuses
**validated** SQL instead of re-typing it.

<h2>2.2 Built the notebook programmatically</h2>
- Wrote a temporary builder (<code>/tmp/build/build.py</code>, since deleted) that reads the existing
notebooks, selects the validated cells, applies a small set of string substitutions
(<code>/workspace/ICOCO.duckdb → /content/ICOCO.duckdb</code>; <code>silver.RecipeZenodo → zenodo.recipes</code>;
<code>bronze.llm_ingredient → ingredient_standardize.runs</code>; …), and assembles the merged notebook.
- Added new Colab glue cells: config, library install, data-verification, dataset downloads,
  LLM-output ingestion, GBIF fetch, ingredient→country map, and the FoodBERT / K-means / figure code.

<h2>2.3 Result — 117 cells (80 code, 37 markdown)</h2>
Parts: 0 setup · 1 final tables + LLM metrics · 2 GBIF · 3 vectors + FoodBERT · 4 battle ·
5 K-means · 6 figures. All SQL is idempotent; markdown is terse.

<h1>3. Troubles I hit and how I resolved them</h1>

<h2>3.1 The editor caps writes at ~6 KB</h2>
- <font color="red">Consequence: a single chunk of the builder was rejected.</font>
- <font color="green">Resolve: split the builder into small appended chunks via a sentinel comment.</font>

<h2>3.2 The read tool truncated long cell dumps in the middle</h2>
- <font color="red">Consequence: I could not see the full SQL of some cells.</font>
- <font color="green">Resolve: let the builder read the source notebooks directly with <code>nbformat</code>
(exact bytes — no manual copy/escaping), and re-read the few cells I needed with <code>sed</code> ranges.</font>

<h2>3.3 The container disconnected (RAM) during my SQL smoke test</h2>
- <font color="red">Consequence: the session restarted mid-validation.</font>
- <font color="green">Resolve: I stopped all heavy operations (no more loading the 116 MB database),
finished the lightweight validation below, and cleaned up. The notebook file was already written and intact.</font>

<h2>3.4 Smoke-test false alarms: views have no <code>rowid</code></h2>
- <font color="red">Consequence: aliasing the new table names to <b>views</b> made <code>geo_long</code> / battle
cells fail with “does not have a column named <code>rowid</code>”.</font>
- <font color="green">Resolve: re-ran with <b>materialised table</b> aliases (as the notebook really creates),
after which all those cells executed.</font>

<h1>4. Can I prove the task is done?</h1>

| Check | Result |
|---|---|
| <code>nbformat.validate</code> | ✅ valid |
| Python compilation of every code cell (magics stripped) | ✅ **0 syntax errors** |
| SQL smoke test on a DB **copy** (39 file-independent SQL cells) | ✅ all executed; the only failures were the cells that read uploaded files I had not provided |
| Leftover <code>/workspace</code> or <code>/content/</code> paths | ✅ none (all paths are Colab <code>/content/…</code>) |

- Temp files removed: <code>/tmp/build</code>, <code>/tmp/nbdump</code>, <code>/tmp/smoke.duckdb</code>,
and the temporary <code>/content</code> symlink dir.

<h1>5. Things you should review</h1>

<h2>5.1 External metrics are a re-implementation</h2>
The notebook that computed <i>Partial-Match Accuracy</i> / <i>Soft-MacroF1</i> is <b>not</b> in
<code>old_code/</code> (only its result-readers are). Part 5.3 is a faithful-in-spirit Hungarian
re-implementation; the <b>internal</b> metrics (Silhouette / DB / CH) are exact.

<h2>5.2 FoodBERT text source</h2>
FoodBERT is encoded from the curated recipes (Name + Ingredients + Cuisine) so it stays row-aligned
with the vectors; the legacy notebook used the raw CSV's <code>Ingredients_Raw + Instructions</code>.

<h2>5.3 Two mapping files must be uploaded</h2>
Entity and odor vectors need <code>dirty_ingredient_category_mapping.json</code> and
<code>dirty_ingredient_nutrient_mapping.json</code> (not in the DB) — Part 0.1 checks for them.

<h2>5.4 Cost watch-outs</h2>
GBIF fetches, FoodBERT (GPU) and K = 207 K-means are the expensive cells; all are clearly separated.
