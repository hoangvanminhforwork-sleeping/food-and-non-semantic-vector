# Task Note — Documenting the "one huge Colab notebook" mission

<h1>1. Mission</h1>

<blockquote>
This was a <b>documentation / hand-off task</b> (no pipeline code was run). Two deliverables:
<ul>
<li>Rewrite <code>final_mission.md</code> — the author's raw Vietnamese brief for the
"one huge Colab notebook" job — into a clean, enhanced mission brief.</li>
<li>Write the notebook-structure proposal into <code>recommended_huge_notebook_structure.md</code>,
derived from the author's raw idea in <code>merge_notebook_template.md</code>.</li>
</ul>
Context: <i>agent hand-off</i>. The project is finished; the author no longer remembers how it
worked, so the paper (<code>ICOCO IEEE eXPress.pdf</code>) is the source of truth for the workflow.
</blockquote>

<h1>2. What I did, troubles, resolutions</h1>

<h2>2.1 Read the paper first</h2>
- <font color="blue">I extracted all 6 pages with <code>pypdf</code> (24,945 chars) and read the
methodology + results.</font> Key facts captured:
  - <b>304-D</b> representation = <b>248</b> geography (GBIF) + <b>28</b> entity (USDA) + <b>28</b> odor
  (USDA+FooDB); <b>50,510</b> recipes, <b>30,621</b> geographically labeled; 116 raw cuisine tags,
  207 countries / 31 sub-regions / 6 continents.
  - LLM canonicalization at <b>T = 0.5, 3 runs</b>; metrics <b>SC / IC / AC / Queryability</b>.
  - K-means at <b>K ∈ {6, 31, 207}</b>, <b>5 seeds</b>, <b>Hungarian</b> post-hoc matching;
    external metrics <b>PA</b> + <b>Soft-MacroF1</b>; internal metrics <b>Silhouette / Davies-Bouldin /
    Calinski-Harabasz</b>; <b>conservative stress test</b> (Proposed mean−sd vs FoodBERT mean+sd).

<h2>2.2 Mapped the whole pipeline (old ↔ new)</h2>
- <font color="blue">I read <code>merge_notebook_template.md</code>, the new folders
<code>0_upload_data_to_DuckDB</code> … <code>8_create_battle</code>, the relevant notes, and the
legacy late stages (<code>old_code/10_geo_vector</code>, <code>11_entity_vector</code>,
<code>12_odor_vector</code>, <code>13..17</code>, <code>FoodBERT_encoding_npy.ipynb</code>).</font>
- I inspected the actual LLM data folders (<code>data/ingredient_standardize</code>,
<code>data/odor</code>, <code>data/cuisine_normalize</code>, <code>data/scientific_name</code>):
each holds <code>output1/2/3.jsonl</code> — confirming the "3 replicas" rule.
- From this I built the old↔new mapping table that now closes the proposal (the appendix, §9).

<h2>2.3 Wrote <code>final_mission.md</code> (enhanced brief)</h2>
- Turned the raw bullet list into a structured English brief: Objective, "how it will be run"
  (dry code on Colab), 7 hard constraints, deliverables, and a reference table.
- Kept the <b>original Vietnamese notes verbatim</b> in an appendix so nothing is lost.

<h2>2.4 Wrote <code>recommended_huge_notebook_structure.md</code> (proposal)</h2>
- Sections: scope → design principles → global conventions (libraries, DuckDB connection,
  idempotency) → data & naming conventions → pipeline diagram → <b>cell-by-cell structure
  Parts 0–6</b> → ordering/dependency notes → open decisions → old↔new appendix.
- I promoted the author's raw idea into a linear, executable order and flagged the dependencies
  (e.g. GBIF must precede the geo vector).

<h2>2.5 Troubles I hit and how I resolved them</h2>
- <font color="red">The editor refused to create <code>recommended_huge_notebook_structure.md</code>
because the file already existed (0 bytes) and no <code>old_text</code> was given.</font>
<font color="green">Resolved by inserting at line 1 and appending the rest through a
<code>&lt;!-- CONTINUE --&gt;</code> sentinel, in small chunks — this also sidesteps the
known "editor truncates large writes" problem.</font>
- <font color="red">The original <code>final_mission.md</code> had trailing spaces, so a full-file
<code>old_text</code> match would be brittle.</font>
<font color="green">Resolved by removing the file and recreating it cleanly with the editor.</font>

<h1>3. Can I prove the task is done?</h1>

| File | State | Check |
|---|---|---|
| <code>final_mission.md</code> | rewritten, 72 lines | starts with <code># FINAL MISSION …</code>; Vietnamese original preserved in the appendix |
| <code>recommended_huge_notebook_structure.md</code> | new, 245 lines | headings <code>## 1</code> … <code>## 9</code>; Parts 0–6 present; no leftover <code>CONTINUE</code> sentinel (verified with <code>grep</code>) |

- Both files verified through the terminal (line counts + heading grep), so the VS Code cache can
  be reloaded to view them.

<h1>4. Things you may want to review</h1>

<h2>4.1 The naming decision needs your sign-off</h2>
Your template proposed <code>ingredient_extraction_output1.json</code>, but the new codebase names
LLM tasks after their <b>schemas</b> (<code>ingredient_standardize</code>,
<code>cuisine_normalize</code>, …). I recommended <code>{schema}_output{n}.json</code> so
file → table is 1:1. Kept as an explicit "open decision" (§8) rather than silently choosing.

<h2>4.2 Two ways the run could spend money / hit limits</h2>
FoodBERT in Colab (GPU) and live GBIF queries. I recommended shipping both as release artifacts
(<code>foodbert_Zenodo.npy</code>, GBIF cache) so re-runs are free and deterministic — please confirm.

<h2>4.3 50,510 vs 50,511</h2>
The paper says 50,510; the new pipeline yields 50,511 (documented malformed-CSV artifact). The
proposal says to keep the new count and note the difference.

<h2>4.4 Scope of the last parts</h2>
Parts 5–6 could be "main figures only" or include the ablation + no-label battles + heatmaps.
Flagged in §8; the legacy code supports the full set.
