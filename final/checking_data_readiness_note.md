# Task Note — data-readiness of the two ICOCO notebooks

<h1>1. Mission</h1>

<blockquote>
Turn the existing single Colab notebook into <b>two</b> notebooks and make them run without the
(blocked) upstream downloads:
<ul>
<li>Create <code>final/ICOCO_notebook.ipynb</code> — runs on the <b>local host</b> using the files
already on disk.</li>
<li>Fix/update <code>final/ICOCO_notebook_Colab.ipynb</code> — runs on <b>Google Colab</b>, loading
everything from a <b>GitHub Release</b>.</li>
</ul>
Both must keep structural parity. Constraints: reuse pre-computed artifacts (no regeneration where
avoidable), <b>do NOT run K-means</b>, <b>do NOT clone/run FoodBERT</b>. After the local notebook
creates the <code>.npy</code> vectors, report how much they differ from
<code>7_create_npy_files/</code>.
</blockquote>

<h1>2. What I did, troubles, how I resolved them</h1>

<h2>2.1 Inspected the environment first <font color="blue">(terminal)</font></h2>
- Confirmed the raw data on disk: <code>data/RecipeZenodo_final.csv</code> (50,511 recipes after the
curation filter), <code>data/coutries.csv</code> (249 countries), <code>data/foodb_2020_04_07/</code>,
<code>data/FoodUSDA_latest|legacy/</code>, and <code>data/GBIF_query/</code>
(<code>dirty_ingredient_country_mapping.json</code> = 4,280 entries, <code>gbif_species_match.jsonl</code>).
- Confirmed the reference vectors in <code>7_create_npy_files/</code>:
  geography <code>(50511, 249)</code>, entity <code>(50511, 28)</code>, odor <code>(50511, 27)</code>,
  entropy <code>(50511,)</code> — all <code>float32</code>.
- Confirmed the pre-computed FoodBERT embedding
  <code>old_code/15_KMean_battle_field/all_vector/foodbert_Zenodo.npy</code> <code>(50513, 768)</code>.

<h2>2.2 Built both notebooks from one source (keeps parity)</h2>
- Wrote a throw-away builder (<code>/tmp/build_nb.py</code>, deleted) that reads the original Colab
notebook, applies a small set of string substitutions and inserts a few cells, and emits
<b>both</b> notebooks. Cell counts: local <b>122</b>, Colab <b>123</b>.
- <b>Local</b> changes: paths under <code>/workspace</code>; reuse the curated recipe CSV; reuse the
on-disk GBIF caches (no live API calls); <code>REGENERATE_VECTORS=True</code> (rebuild the 3 vectors
and write them to <code>final/data/</code>); FoodBERT loaded from the pre-computed file; K-means
guarded off.
- <b>Colab</b> changes: <code>FooDB</code>/<code>FoodUSDA</code> are <b>not</b> fetched from their
blocked upstream servers — they come from a <code>GH_RELEASE</code>; a new
<code>fetch()</code> cell downloads all 15 artifacts (merged LLM files, mapping jsons, GBIF caches,
the 4 <code>.npy</code>, and the FoodBERT embedding); vectors are <b>reused</b>
(<code>REGENERATE_VECTORS=False</code>); FoodBERT loaded from the release; K-means guarded off.
- Added a <code>3.5 Verify</code> cell (both notebooks) that compares the vectors against the
reference <code>7_create_npy_files/</code> and prints <code>max|diff|</code>.

<h2>2.3 Troubles I hit and how I resolved them</h2>
- <font color="red">I ran the builder <b>twice</b>; because it read its own output
(<code>final/ICOCO_notebook_Colab.ipynb</code>) as the source, the second run re-transformed an
already-transformed notebook (123 cells → 127). The original was not in git (<code>final/</code>
is untracked).</font>
  <font color="green">Resolved by reconstructing the pristine 118-cell original from the full dump
I had saved to <code>/tmp/colab_dump.txt</code>, then pointing the builder at a stable source
(<code>/tmp/ICOCO_colab_ORIGINAL.ipynb</code>) so re-runs are idempotent.</font>
- <font color="red">A few <code>/content/…</code> paths survived in the local notebook (entity/odor
mapping jsons, <code>reaction_cooking.py</code>, the guarded vector loads).</font>
  <font color="green">Resolved with a final path sweep; verified <code>grep '/content'
final/ICOCO_notebook.ipynb</code> → none, and the reverse for Colab.</font>
- <font color="red"><b>Ordering bug:</b> the "verify the uploaded data" cell ran <i>before</i> the
download/fetch steps, so it asserted on files that did not exist yet (this is exactly the failure
seen on Colab).</font>
  <font color="green">Resolved by moving the verify section to <code>## 0.4 Verify the data</code>,
<i>after</i> <code>0.2b</code> (the fetch step). Re-validated end-to-end afterwards.</font>
- <font color="red">RAM is very tight (~0.6 GB free at times).</font>
  <font color="green">Ran the notebooks in the background with a DuckDB cap
(<code>SET memory_limit='1.5GB'</code>, <code>threads=4</code>) and monitored
<code>/proc/meminfo</code> between polls; the peak stayed within budget.</font>

<h1>3. Can I prove the task is done?</h1>

<table>
<tr><th>Check</th><th>Result</th></tr>
<tr><td>Local notebook, all 122 cells executed top-to-bottom</td><td>✅ <code>=== EXECUTION OK (all cells) ===</code></td></tr>
<tr><td>Colab notebook, all 123 cells executed against a <b>mock GitHub Release</b> (local HTTP server)</td><td>✅ <code>=== COLAB EXECUTION OK (all cells) ===</code></td></tr>
<tr><td>Python syntax of every code cell (magics stripped)</td><td>✅ 0 errors (35 local / 36 Colab)</td></tr>
<tr><td>Leftover foreign paths</td><td>✅ none (<code>/content</code> in local, <code>/workspace</code> in Colab)</td></tr>
<tr><td>Battle split matches the paper</td><td>✅ 30,621 <code>has_geo_label</code> + 19,890 <code>no_geo_label</code> = 50,511</td></tr>
<tr><td>K-means / FoodBERT executed?</td><td>✅ no — all five K-means cells print <code>SKIPPED</code>; FoodBERT is loaded from the artifact</td></tr>
</table>

<h2>3.1 The requested comparison — generated <code>.npy</code> vs <code>7_create_npy_files/</code></h2>
<table>
<tr><th>File</th><th>Shape</th><th>max|diff|</th><th>identical (&le;1e-6)</th></tr>
<tr><td>recipe_geography_vectors.npy</td><td>(50511, 249)</td><td>0.000e+00</td><td>✅ True</td></tr>
<tr><td>entity_vectors.npy</td><td>(50511, 28)</td><td>0.000e+00</td><td>✅ True</td></tr>
<tr><td>recipe_odor_vectors.npy</td><td>(50511, 27)</td><td>0.000e+00</td><td>✅ True</td></tr>
<tr><td>recipe_odor_entropy.npy</td><td>(50511,)</td><td>0.000e+00</td><td>✅ True</td></tr>
</table>
<b>The local notebook rebuilds the three vectors + entropy from the raw data and they are bit-for-bit
identical to the reference artifacts</b> (all differences are exactly 0). The new files are written to
<code>final/data/</code>.

<h1>4. Things you should review</h1>

<h2>4.1 <code>GH_RELEASE</code> is a placeholder</h2>
Both the Colab notebook and the artifact fetch use
<code>GH_RELEASE = 'https://github.com/&lt;owner&gt;/&lt;repo&gt;/releases/download/&lt;tag&gt;'</code>.
Set it to your real release before running on Colab. The release must contain the 15 files listed in
<code>ARTIFACTS_TO_FETCH</code>, <b>plus</b> the two raw archives under these exact names:
<code>foodb_2020_4_7_csv.tar.gz</code> and <code>FoodData_Central_sr_legacy_food_csv_2018-04.zip</code>
(note: the local copy of the legacy zip is named <code>…2018-04 (1).zip</code> — drop the
<code> (1)</code> when uploading).

<h2>4.2 Reuse vs regenerate — a flag decides</h2>
Your brief asked to <i>reuse</i> the pre-computed <code>.npy</code>, and separately to report the
difference after <i>creating</i> them. I resolved this with <code>REGENERATE_VECTORS</code>:
<b>local = True</b> (rebuild + compare — this is how the table in §3.1 was produced),
<b>Colab = False</b> (pure reuse). Flip the flag if you prefer the other behaviour.

<h2>4.3 FoodBERT row count</h2>
The reused <code>foodbert_Zenodo.npy</code> has <b>50,513</b> rows while the vectors have
<b>50,511</b>. This 2-row offset is pre-existing in the legacy artifact (documented in earlier notes);
the notebook simply reuses it. FoodBERT is never re-run here.

<h2>4.4 K-means is present but disabled</h2>
Parts 5–6 are kept for structural parity and reproducibility but wrapped in
<code>if not RUN_KMEANS: … else: …</code>. Set <code>RUN_KMEANS=True</code> to execute them (this is
the expensive path and was intentionally left off).

<h2>4.5 Release payload completed</h2>
To make the release self-consistent, I added the missing files to <code>final/data/</code>:
<code>coutries.csv</code>, <code>gbif_species_match.jsonl</code>,
<code>dirty_ingredient_country_mapping.json</code>, the 4 regenerated <code>.npy</code>, and
<code>foodbert_Zenodo.npy</code>.
<code>/proc/meminfo</code> between polls; the peak stayed within budget.</font>