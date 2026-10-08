# Task Note — enabling FoodBERT encoding + GPU K-means in `ICOCO_notebook_Colab.ipynb`

<h1>1. Mission</h1>

<blockquote>
The Colab notebook was a <b>data-readiness</b> run: it <i>reused</i> the pre-computed
<code>foodbert_Zenodo.npy</code> and never executed the K-means "battle"
(<code>RUN_FOODBERT=False</code>, <code>RUN_KMEANS=False</code>).<br><br>
Write the missing code so that:
<ul>
<li>the notebook <b>encodes the dataset with FoodBERT</b> itself and produces
<code>foodbert_Zenodo.npy</code> (no longer downloads it);</li>
<li>the K-means battle runs <b>on the GPU</b>;</li>
<li>both the <b>full dataset</b> and the <b>labeled / unlabeled subsets</b> are supported
(reference: <code>old_code/17_result_and_discussion_</code> +
<code>old_code/15_KMean_battle_field/no_label_battle_and_fulldata_battle.ipynb</code>).</li>
</ul>
Constraint: <b>do not run</b> the cells here — the paths are Colab paths and this box has no
torch/GPU.
</blockquote>

<h1>2. What I did</h1>

<h2>2.1 Flipped the scope flags (config cell <code>83337070</code>)</h2>
- <code>RUN_FOODBERT = True</code> — clone + run FoodBERT (section 3.4).
- <code>RUN_KMEANS = True</code> — run the K-means battle (section 5).
- Added <code>KMEANS_DEVICE = 'cuda'</code> — the K-means device selector (<code>'cuda'</code>
  with automatic fallback to CPU when CUDA is unavailable).

<h2>2.2 Stopped treating <code>foodbert_Zenodo.npy</code> as an input</h2>
- Removed it from <code>ARTIFACTS_TO_FETCH</code> (cell <code>d893902b</code>) and from the
  required-files check (cell <code>2af622c9</code>), because it is now <b>produced</b> by the
  notebook.

<h2>2.3 Rewrote section 3.4 (3 cells) into a real FoodBERT encoder</h2>
- <code>b66d41b2</code> — install <code>transformers</code> and clone
  <code>github.com/chambliss/foodbert</code> (<code>--depth 1</code>) for fidelity with the
  reference notebook.
- <code>0de40ad0</code> — build the text <b>via DuckDB</b>: it reads the raw
  <code>Ingredients_Raw</code> + <code>Instructions</code> columns straight from
  <code>/content/zenodo_final.csv</code> (produced in section 1.3) with
  <code>read_csv(...)</code>, and forms
  <code>"Ingredients: {raw}. Instructions: {instr}. "</code> — identical to
  <code>old_code/FoodBERT_encoding_npy.ipynb</code>.
- <code>92309470</code> — tokenise (max_length 512) → <code>AutoModel</code> →
  <b>mean pooling</b> over <code>last_hidden_state</code> → <code>np.save(FOODBERT_NPY, X)</code>,
  768-D <code>float32</code>.

<h2>2.4 Added a self-contained GPU K-means (cell 5.1, <code>33e88249</code>)</h2>
- Introduced <code>gpu_kmeans(X, n_clusters, seed, n_init=3)</code> built on <b>torch only</b>
  (no extra installs — Colab already ships torch):
  <ul>
  <li><b>k-means++</b> seeding (sequential <code>torch.multinomial</code> on the squared-distance
  to the nearest chosen centre);</li>
  <li><b>Lloyd</b> iterations (assign via <code>_sqdist</code>, update with
  <code>index_add_</code> / <code>bincount</code>, early stop on centre shift &lt; <code>tol</code>);</li>
  <li><code>n_init=3</code> restarts, keep the lowest inertia — mirrors
  <code>sklearn.KMeans(..., n_init=3)</code>;</li>
  <li>returns an <code>int32</code> numpy label array (drop-in for
  <code>KMeans(...).fit_predict(X)</code>);</li>
  <li>runs on CUDA if available, otherwise CPU (<code>KMEANS_DEVICE</code>).</li>
  </ul>

<h2>2.5 Swapped sklearn K-means for the GPU one in the battle</h2>
- 5.2 internal metrics (<code>c7552e56</code>): <code>lab = gpu_kmeans(M, K, seed)</code> for the
  <b>Full_Dataset</b> and the <b>No_Label_Subset</b> (via <code>no_idx</code>).
- 5.3 external alignment (<code>787f6e49</code>): <code>lab = gpu_kmeans(X, K, seed)</code> over the
  labeled subset (<code>lab_rows</code>).
- Kept the <b>sklearn</b> internal metrics (Silhouette / Davies-Bouldin / Calinski-Harabasz) on
  CPU — only the clustering itself was moved to the GPU.

<h2>2.6 Text-build via DuckDB</h2>
- As you preferred, the FoodBERT inputs are read with <b>DuckDB</b>
  (<code>read_csv</code> over <code>zenodo_final.csv</code>), not with pandas/<code>pd.read_csv</code>.

<h1>3. Troubles I hit and how I resolved them</h1>

<h2>3.1 Editing a notebook JSON by hand is brittle</h2>
- <font color="red">A notebook cell's <code>source</code> is a list of newline-terminated JSON
strings; replacing them with the editor is error-prone.</font>
- <font color="green">Resolve: I wrote a throw-away patch script
(<code>/workspace/_tmp_patch_notebook.py</code>) that loads the notebook, replaces the source of
cells <b>by id</b>, and dumps it back (<code>json.dump(..., indent=1, ensure_ascii=False)</code>).
Then I deleted the script.</font>

<h2>3.2 The editor rejects writes &gt; ~6 KB</h2>
- <font color="red">My first patch-script write (≈19 KB) was rejected.</font>
- <font color="green">Resolve: I built the script in small appended chunks using a sentinel line,
exactly as in the earlier builder tasks.</font>

<h2>3.3 Row alignment: 50,511 vs the legacy 50,513</h2>
- The legacy <code>foodbert_Zenodo.npy</code> has <b>50,513</b> rows (it encoded the
<i>unfiltered</i> CSV), while the geometry/entity/odor vectors have <b>50,511</b>.
- <font color="green">I aligned FoodBERT to <b>50,511</b> by reading
<code>zenodo_final.csv</code> (the same file that feeds <code>zenodo.recipes</code> and the
vectors). This removes the pre-existing 2-row offset, so every "player" in the battle has the same
row count and the subset indices line up. (Flip to the raw <code>dataset.csv</code> if you want the
legacy 50,513 instead.)</font>

<h2>3.4 torch is not installed on this box</h2>
- <font color="red">I could not execute the torch K-means here (<code>No module named 'torch'</code>),
so I could not run the real GPU path.</font>
- <font color="green">Resolve: I validated the <b>algorithm</b> by mirroring the same control flow
in NumPy — on 3 synthetic blobs it reproduces sklearn exactly (identical inertia, ARI = 1.0) and is
deterministic across runs. The torch tensor ops used (<code>bincount</code>,
<code>index_add_</code>, <code>multinomial</code>, <code>randint</code>, CUDA
<code>Generator</code>) are all standard and GPU-supported.</font>

<h2>3.5 Line magics inside an <code>if/else</code></h2>
- <font color="red">Line magics (<code>%pip</code>) cannot appear inside an indented block.</font>
- <font color="green">Resolve: the install/clone cell (<code>b66d41b2</code>) keeps
<code>%pip</code>/<code>!git</code> at top level; only the pure-Python encode cell is wrapped in
<code>if RUN_FOODBERT:</code>.</font>

<h1>4. Can I prove the task is done?</h1>

| Check | Result |
|---|---|
| <code>nbformat.validate</code> | ✅ valid (123 cells) |
| Python syntax of the 8 modified cells (magics stripped) | ✅ 8/8 compile |
| GPU K-means algorithm vs sklearn (NumPy mirror) | ✅ identical inertia, ARI = 1.0, deterministic |
| <code>foodbert_Zenodo.npy</code> removed from fetch + required lists | ✅ |
| <code>RUN_FOODBERT</code> / <code>RUN_KMEANS</code> set to <code>True</code>; <code>KMEANS_DEVICE</code> added | ✅ |

- Temp files removed: <code>/workspace/_tmp_patch_notebook.py</code> and the
<code>/tmp</code> notebook backup.

<h1>5. Things you should review</h1>

<h2>5.1 The K-means "full dataset + subset" logic is unchanged</h2>
Only the clustering backend changed (sklearn → torch GPU). Subsets are still
<code>no_idx</code> (unlabeled) and <code>lab_rows</code> (labeled) computed from
<code>battle.recipe_status</code> — same as before.

<h2>5.2 If you prefer cuML instead of torch</h2>
Swap <code>gpu_kmeans</code> for <code>cuml.cluster.KMeans</code> (RAPIDS). The torch version was
chosen because it needs <b>no extra install</b> on Colab.

<h2>5.3 Release payload still missing the vector <code>.npy</code></h2>
- <code>final/data/</code> currently holds the merged LLM files + mappings + GBIF cache, but <b>no
<code>.npy</code></b> (neither the 4 vectors nor FoodBERT).
- Since FoodBERT is now generated in-notebook this is fine for FoodBERT, but the
  <b>4 vector <code>.npy</code></b> are still fetched from <code>GH_RELEASE</code> (section 0.2b).
  Either upload them to the release, or set <code>REGENERATE_VECTORS=True</code> to rebuild them in
  the notebook.

<h2>5.4 Colab cost note</h2>
With the flags ON, section 3.4 (FoodBERT, GPU) and section 5 (K = 207 K-means, GPU) are the
expensive cells. If you only want readiness, set both flags back to <code>False</code>.

