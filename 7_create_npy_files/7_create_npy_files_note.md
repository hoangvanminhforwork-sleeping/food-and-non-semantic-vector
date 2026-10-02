# Create NPY Vector Files — Task Note

<h1>1. Mission</h1>

<blockquote>
Reproduce the research of <i>ICOCO (IEEE eXpress)</i>: create the <b>three recipe vector files</b>
(geography, entity, odor) exactly like the legacy folders whose names contain “vector”
(<code>old_code/10_geo_vector</code>, <code>old_code/11_entity_vector</code>, <code>old_code/12_odor_vector</code>),
but rebuilt from the <b>new cleaned pipeline</b> (<code>1_silver_create</code>, <code>2_data_exploratory</code>,
<code>4_AI_cleaned_data_exploratory</code> → <code>ICOCO.duckdb</code>).

Deliverables, all in <code>/workspace/7_create_npy_files</code>:
<ol>
<li><b>3 <code>.npy</code> vector files</b> — <code>recipe_geography_vectors.npy</code>, <code>entity_vectors.npy</code>, <code>recipe_odor_vectors.npy</code></li>
<li><b>+ 1 <code>.npy</code> follow-up</b> — <code>recipe_odor_entropy.npy</code> (added on request: per-recipe entropy)</li>
<li><b>3 notebooks</b> that reproduce those files (SQL-first, safe & idempotent)</li>
<li>this note, including <i>“how much did my initial information help you”</i></li>
</ol>
Constraints: try not to create new tables in DuckDB; no API calls for the geo vector;
watch RAM (container limit ≈ 3.68 GB).
</blockquote>

<h1>2. What I did, troubles, resolutions</h1>

<h2>2.1 Reverse-engineered the legacy pipeline (read-only inspection)</h2>

- <font color="blue">I read the three legacy export notebooks plus their upstream dependencies
(<code>09_gold_data_create/gold_reactions.ipynb</code>, <code>utils/gold_data_creator.py</code>,
<code>data/odor/reaction_cooking.py</code>) and decoded every legacy artifact size from its npy header:
<code>(50513, 248)</code> geo, <code>(50513, 28)</code> entity, <code>(50513, 27)</code> odor.</font>
- <font color="blue">I inspected the new DuckDB (<code>ICOCO.duckdb</code>): 9 schemas, 45 tables/views —
found <code>silver.RecipeZenodo</code> (50,511 rows, <code>Ingredients</code>/<code>Cooking Methods</code> as real
<code>VARCHAR[]</code>), <code>silver.countries</code> (249 codes), <code>bronze.dirty_ingredient_country_mapping</code>,
<code>silver.FoodUSDA_latest_food_category</code> (28 categories), and the LLM replica tables
<code>odor.output1/2/3</code> (3,792 rows each).</font>
- <font color="red">Consequence: the legacy notebooks read files that no longer exist in the new
workspace (<code>silver_Zenodo.csv</code>, <code>gold_reactions.csv</code>, <code>bronze/FoodUSDA_latest/food_category.csv</code>),
and the ingredient→category / ingredient→nutrient LLM mappings are <b>not</b> in the new DuckDB.</font>
- <font color="green">Resolve: map every legacy input to a new equivalent — recipes/countries/
country-mapping/categories come from DuckDB tables; the two LLM mapping JSONs are read
directly from <code>old_code/…/data/*.json</code> via DuckDB <code>read_json()</code>; the odor gold map is
<b>re-derived inside SQL</b> from <code>odor.output1/2/3</code> with the exact majority-vote logic of
<code>old_code/utils/gold_data_creator.py</code> (≥ 2 of 3 replicas, strip+lower). No new persistent
tables — only <code>CREATE OR REPLACE TEMP VIEW/TABLE</code>, gone when the session closes.</font>

<h2>2.2 Fidelity details I had to replicate exactly</h2>

- <font color="red">Consequence: the three legacy notebooks use <b>different denominator semantics</b>;
one wrong choice silently changes thousands of rows:</font>
  - <b>geo</b>: denominator counts <i>every</i> country occurrence returned by the mapping, even
    codes missing from the country table (they get no column);
  - <b>entity</b>: categories are filtered to the 28 valid ones <b>before</b> the tally
    (<code>if cat and cat in cat_to_idx</code>), so unknown categories leave both numerator and denominator;
  - <b>odor</b>: denominator counts <i>every</i> odor returned, even names outside the 27-odor axis.
- <font color="green">Resolve: implemented each variant verbatim in SQL (the <code>tally</code> CTE
sits before/after the join depending on the notebook). Also replicated: occurrence-based
counting for geo/entity vs <i>set</i>-based nutrients for odor; <code>rsplit(' AND ', 1)</code> ≙ greedy
regex <code>^(.*) AND (.*)$</code>; row order = physical <code>rowid</code> order (verified contiguous 0..50510);
the 22-method → 107-pair <code>cooking_reactions</code> config loaded from
<code>/workspace/data/odor/reaction_cooking.py</code> instead of being re-typed.</font>

<h2>2.3 Troubles I hit, and how I resolved them</h2>

<h3>2.3.1 Wrong entity denominator (my first SQL draft)</h3>
- <font color="red">Consequence: new entity rows summed to 0.75 instead of 1.0 (row 2:
0.125/0.25/0.25/0.125) — unknown categories stayed in the denominator; only 4/50,511 rows
matched the legacy matrix.</font>
- <font color="blue">I compared row-by-row against the legacy npy: identical support but
smaller shares.</font>
- <font color="green">Resolve: join <code>silver.FoodUSDA_latest_food_category</code> <b>inside</b> the
<code>matched</code> CTE (filter before tally). Result: 50,504/50,511 rows bit-identical.</font>

<h3>2.3.2 Legacy row alignment (50,513 vs 50,511 rows)</h3>
- <font color="red">Consequence: naive 1:1 comparison matched only 24,259/50,511 odor rows —
the legacy silver CSV has 2 rows that the new bronze ingestion dropped.</font>
- <font color="blue">I ran a two-pointer alignment plus a candidate search over drop sets
({21475, 23805} vs 1:1-prefix vs tail-drops); the winner was old rows <b>21475</b> and
<b>23805</b> (50,509 identical odor rows vs 24,259 for the alternatives).</font>
- <font color="green">Resolve: the notebooks build the legacy-comparison index with a
while-loop that skips exactly those two old rows. Offsets are used <b>only</b> for
verification, never for building the vectors.</font>

<h3>2.3.3 <code>%%sql</code> magic missing on SQL cells</h3>
- <font color="red">Consequence: <code>nbconvert</code> failed with <code>SyntaxError: invalid decimal literal</code>
— cells starting with <code>-- comments</code> were executed as Python.</font>
- <font color="green">Resolve: prepend <code>%%sql</code> to every SQL-only cell; notebooks
regenerated.</font>

<h3>2.3.4 DuckDB <code>.df()</code> returns LIST columns as <code>numpy.ndarray</code></h3>
- <font color="red">Consequence: the gold cross-check printed 987/3,792 “identical” because
non-empty rows failed <code>isinstance(v, list)</code> — the gold sets themselves were correct.</font>
- <font color="green">Resolve: accept <code>(list, np.ndarray)</code> in the comparison; also grouped
the cross-check by <code>custom_id</code> only (<code>odor_gold_map</code> does not carry <code>input_text</code>).</font>

<h3>2.3.5 Environment/tooling friction</h3>
- <font color="blue">Long shell heredocs got mangled by terminal echo; <code>jupysql</code> imports
only as module <code>sql</code> (so <code>import jupysql</code> fails while <code>%load_ext sql</code> works);
<code>free</code>/<code>ps</code> don't exist in the container.</font>
- <font color="green">Resolve: wrote prototype/generator scripts to files and read their logs;
used <code>/proc/meminfo</code> for RAM (3.77 GB total, ≈ 1.16 GB available at start); verified the
sqlalchemy connection-fairy proxies <code>execute</code>/<code>executemany</code>/<code>.df()</code> before relying on
it; removed an accidental <code>nohup.out</code> from the mission folder.</font>

<h2>2.4 RAM / safety engineering</h2>

- <font color="blue">I designed everything around the 3.68 GB limit: all heavy lifting
(grouping, unnest, pair expansion ≈ millions of intermediate rows) happens <b>inside DuckDB</b>
with <code>SET memory_limit='1.5GB'</code> and <code>SET threads=4</code>; Python only fetches the small
<b>long-format</b> result (281k geo / 260k entity / 984k odor rows → ≈ 10–50 MB as pandas)
and scatters it into the dense matrix with NumPy fancy indexing.</font>
- <font color="green">Peak RAM observed well under 1 GB per notebook; legacy npys are opened
with <code>mmap_mode='r'</code>. Each notebook follows the house pattern: connect → close (flush WAL)
→ reconnect → single shared connection for both <code>%sql</code> and Python; final cell disconnects.
Only <b>TEMP</b> objects are created, so re-running never changes the database state.</font>

<h1>3. Can you prove that you have completed the tasks?</h1>

<h2>3.1 Before → After</h2>

- <b>Before</b>: <code>/workspace/7_create_npy_files/</code> contained only <code>missions.md</code>.
- <b>After</b> (all verified on disk, <code>ls -la</code>):

| File | Size | Content |
|---|---|---|
| <code>recipe_geography_vectors.npy</code> | 50,309,084 B | float32 matrix <b>(50511, 249)</b> |
| <code>entity_vectors.npy</code> | 5,657,360 B | float32 matrix <b>(50511, 28)</b> |
| <code>recipe_odor_vectors.npy</code> | 5,455,316 B | float32 matrix <b>(50511, 27)</b> |
| <code>recipe_odor_entropy.npy</code> | 202,172 B | float32 vector <b>(50511,)</b> — per-recipe Shannon entropy of the odor matrix |
| <code>geo_vector_npy_export.ipynb</code> | executed, 0 errors | builds + verifies the geo vector |
| <code>entity_vector_npy_export.ipynb</code> | executed, 0 errors | builds + verifies the entity vector |
| <code>odor_vector_npy_export.ipynb</code> | executed, 0 errors | builds + verifies the odor vector <b>and its entropy</b> |

Rows = <code>silver.RecipeZenodo</code> in rowid order (0..50510). Columns: countries in
<code>silver.countries</code> rowid order; categories by <code>id</code> (<code>idx = id-1</code>, legacy
convention); the fixed 27-odor axis in the legacy order.

<h2>3.2 Reproduction quality vs the legacy artifacts</h2>

| Check | Result |
|---|---|
| Odor gold map (SQL majority-vote) vs legacy <code>gold_reactions.csv</code> | <b>3,792 / 3,792 sets identical</b> (2,805 non-empty + 987 empty) |
| Geo “has signal” pattern vs legacy (aligned rows) | <b>100.0 %</b> agreement — 34,408 recipes with signal in both |
| Entity rows bit-identical (|Δ| ≤ 1e-6) | <b>50,504 / 50,511 (99.986 %)</b> |
| Odor rows bit-identical (|Δ| ≤ 1e-6) | <b>50,509 / 50,511 (99.996 %)</b> |
| Entropy values bit-identical (|Δ| ≤ 1e-6) | <b>50,509 / 50,511 (99.996 %)</b> — differs exactly on the same 2 recipes |
| Entropy internal consistency | 13,044 zero-entropy rows == 13,044 no-signal rows; range [0, 3.8634] ≤ log2(27) ≈ 4.7549 |
| Geo pre-shift country columns 0..150 | 223 rows differ &gt; 1e-6 (max |Δ| 0.089) — data-level, see §4.2 |
| Internal sanity (all 3) | max row sum = 1.0; spot-checks of named recipes correct |

The remaining differences are <b>not logic errors</b>: they come from the input data
itself (2 dropped rows; ingredient strings that changed between the legacy silver CSV and
the new bronze ingestion). Details in §4.

<h2>3.3 How to re-run</h2>

<code>jupyter nbconvert --to notebook --execute --inplace &lt;notebook&gt;.ipynb</code> — each notebook
is self-contained, idempotent (TEMP views only), and re-writes its <code>.npy</code>. No cell
mutates <code>ICOCO.duckdb</code>.

<h1>4. I need you review something</h1>

<h2>4.1 The 2 dropped rows (50,513 → 50,511)</h2>
The legacy silver CSV had 50,513 recipes; the new <code>bronze.RecipeZenodo</code> has 50,511.
Old rows <b>21475</b> and <b>23805</b> have no counterpart in the new table. I did not
investigate <i>why</i> the new ingestion dropped them (that belongs to
<code>0_upload_data_to_DuckDB</code>) — worth a look if you expect 50,513.

<h2>4.2 Nine recipes whose vectors legitimately differ</h2>
Entity differs on 7 rows <code>[5688, 28200, 36436, 36710, 42430, 45302, 49156]</code>, odor on 2
<code>[5688, 42430]</code>. Root cause (verified on <code>rowid 5688 “Grilled Peach Bourbon Smash”</code>):
ingredient <b>strings changed</b> between old and new data — e.g. <code>'peach  halved'</code>
(double space) no longer matches the LLM mapping key <code>'peach halved'</code>, and
<code>"Tito's Peppermint Hot Chocolate"</code> now has only 2 ingredients in the new data
(legacy vector implies more). The legacy code also did <b>not</b> collapse internal
whitespace, so my pipeline reproduces the same behaviour. If you want those 9 recipes to
match again, the fix belongs in bronze/silver data cleaning (collapse internal spaces),
not in the vector notebooks.

<h2>4.3 Country axis changed width (248 → 249)</h2>
The legacy <code>coutries.csv</code> yielded 248 distinct codes; today's file gives 249. Column
order matches <code>silver.countries</code> rowid; compared to the legacy matrix there is a +1
column shift after legacy index ≈ 151. Also note 104 recipes contain double-space
ingredients and 8,175 contain duplicate ingredients — the occurrence-based geo/entity
vectors count them, faithfully to the legacy code.

<h2>4.4 Legacy quirks I preserved (do not “fix” lightly)</h2>
- The old odor notebook's cell 4 defines <code>odors</code> as a list <b>without commas</b> (one
concatenated string — a Python trap). It is harmless because cell 6 redefines the axis
properly; my notebooks use the correct 27-item axis.
- The LLM category mapping contains odd labels (e.g. <code>'ice' → 'sausages and luncheon
meats'</code>, <code>'mint' → 'sweets'</code>). I reproduced them as-is; they influence the vectors.
- <code>recipe_odor_entropy.npy</code> is produced by <code>odor_vector_npy_export.ipynb</code> §6 with the
<b>verbatim</b> legacy function (row-wise Shannon entropy base 2, empty rows → 0, float32),
verified bit-exact against <code>old_code/12_odor_vector/odor_entropy_get.ipynb</code> output
(max |Δ| = 0.0 on 5,000 probe rows).

<h2>4.5 Single points of failure</h2>
The ingredient→category and ingredient→nutrient JSONs exist <b>only</b> under
<code>/workspace/old_code/…/data/</code>. If that folder is ever removed, the entity/odor notebooks
break. Consider copying them into <code>/workspace/data/</code> (e.g. <code>data/ingredient_standardize/</code>,
<code>data/odor/</code>) someday.

<h1>5. How much did your initial information help me?</h1>

<p><b>Straight answer: a lot — I would estimate it saved me 60–70 % of the total effort.</b>
Here is the honest scorecard:</p>

<h2>5.1 What helped enormously</h2>
<table border="1">
<tr><th>Your hint</th><th>Effect</th></tr>
<tr><td>“Read old_code 09 / 10 / 11 / 12 + the three new folders”</td>
<td>The legacy export notebooks are fully self-contained (paths, dicts, semantics).
I did not have to guess <b>any</b> part of the vector math. The 4 folders were exactly the right set.</td></tr>
<tr><td>“3 npy files as each folder has 'vector'”</td>
<td>Removed all ambiguity about which of the many npy/csv/json files are the deliverables
(e.g. <code>recipe_odor_entropy.npy</code> stays out of scope).</td></tr>
<tr><td>“Try not to create a new table in DuckDB”</td>
<td>Shaped the whole design: TEMP views/tables + <code>read_json</code> for the missing mappings.
It also forced the nice discovery that the odor gold can be re-derived in pure SQL.</td></tr>
<tr><td>“I do not think you need to run the code calling API for geo vector”</td>
<td>Correct — <code>bronze.dirty_ingredient_country_mapping</code> is already in the DB.
Zero network calls, fully reproducible.</td></tr>
<tr><td>“SQL notebooks, but do the best and safe operations (I guess Python notebook)”</td>
<td>Gave me the freedom to use the hybrid that fits DuckDB best: SQL for all aggregation,
thin Python only for the <code>.npy</code> scatter — respecting your single-connection rule via the
<code>ConnectionManager</code> trick from <code>silver_curation.ipynb</code>.</td></tr>
<tr><td>“Watch RAM/CPU — you failed many times before”</td>
<td>I measured first (<code>/proc/meminfo</code>: 3.77 GB total), capped DuckDB at 1.5 GB / 4 threads,
kept pandas fetches in long format, and executed notebooks one at a time in the background.
No OOM, no container death this time.</td></tr>
<tr><td>“agentTemplates.md” + fixed phases</td>
<td>Note structure, proof section and cleanup were pre-defined — no guessing about the
deliverable format.</td></tr>
</table>

<h2>5.2 What I still had to discover myself</h2>
<ul>
<li>Legacy artifact shapes and the 2 dropped rows (50,513 vs 50,511) — only recoverable from
the npy headers + alignment analysis.</li>
<li>The gold majority-vote logic lives in <code>old_code/utils/gold_data_creator.py</code>, not in the
mission-listed folders.</li>
<li>The ingredient→category / ingredient→nutrient JSONs are not in DuckDB — they must be read
from <code>old_code/…/data/</code>.</li>
<li>The three notebooks' subtly different denominator semantics (§2.2) — invisible until you
diff against the legacy matrices.</li>
<li><code>/workspace/data/odor/reaction_cooking.py</code> already exists as the canonical config.</li>
</ul>

<h2>5.3 Suggestions for next time</h2>
<ul>
<li>If a mission depends on legacy <b>data</b> files (not just code), mentioning that they live
under <code>old_code/*/data/</code> would save one exploration round.</li>
<li>Mentioning the expected npy shapes (or that the paper's n = 50,513) would have made the
row-count difference a non-surprise. (I verified 50,513 from the legacy npy headers —
your “hints” being slightly stale is exactly why I always re-check.)</li>
</ul>

<hr>
<p><b>Summary</b>: your information was accurate (nothing contradicted what I found in the
old code), well-scoped, and the constraints (“no new tables”, “no API calls”, “watch RAM”)
were all load-bearing design decisions rather than decorations. The only stale bit was the
implicit assumption that the new data would have the same 50,513 recipes / 248 countries as
the paper — everything else was spot-on.</p>



