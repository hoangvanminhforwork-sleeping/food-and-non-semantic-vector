# Confirm the numbers in the ICOCO IEEE paper — Mission 02 note (Geographic mismatch resolved)

**Date**: 2026-09-27 (UTC)
**Notebook updated**: [`confirm_numbers.ipynb`](confirm_numbers.ipynb) — new **§B3** section (8 cells)
**Mission**: `5_Confirm_number/confirm_mission02.md`
**Database**: `/workspace/ICOCO.duckdb` (read-only)
**Companion**: `confirm_note.md` (mission 01 — §B was flagged there as the *largest mismatch*; this mission resolves it)

---

## 1. Mission

Investigate the geographic number mismatch (labels vs. no-labels) of the Zenodo dataset
and update `confirm_numbers.ipynb` with a detailed analysis section that shows, per the
mission rules: (1) a labels-vs-no-labels breakdown **before and after** the LLM
voting/standardization with **"world" ignored**; (2) a mismatch-investigation summary;
(3) reproducible verification queries.

## 2. Root cause — how the paper's 30,621 / 19,889 is really computed

The paper says *"30,621 geographically labeled across 116 raw tags, and 19,889 unlabeled"*.
Two naive definitions both miss:

| Definition | Labeled | Unlabeled |
|---|---|---|
| §B1 — raw non-empty `Cuisine` (no standardization) | 31,585 | 18,926 |
| §B2 — single replica `output1` → country, **distinct dish names** | 28,657 | 21,854 |
| **Paper — after majority-vote standardization, `world` ignored, records** | **30,621** ✅ | **19,889** ✅ |

The paper's count reflects the data **after** the LLM geography-standardization + voting
step (`cuisine_normalize.output{1,2,3}`), not the raw tags:

> a recipe row is *geographically labeled* iff **≥ 1 of its `Cuisine` tags** resolves, after
> the **majority vote (element seen in ≥ 2 of the 3 replicas)**, to **≥ 1 country** — with the
> raw tag **"world" ignored** — counting **records**, not distinct dish names.

Rebuilt in SQL this gives **exactly 30,621** (see §3 below).

## 3. Numbers verified in the database

| Quantity | Value |
|---|---|
| Raw distinct cuisine tags (incl. `world`) | **116** |
| Rows with ≥ 1 raw tag / no tag | 31,585 / 18,926 (total 50,511) |
| Rows mentioning `world` / whose only tag is `world` | 213 / 152 |
| Tags that survive the vote (≥ 1 geo attribute) | **109 / 116** |
| Tags dropped entirely (no geography): `Authentic`, `Copycat`, `Creole`, `Fusion`, `Inspired`, `Kosher`, **`world`** | **7** |
| Tags mapping (after vote) to ≥ 1 country | **107** |
| **Labeled rows after standardization (paper rule)** | **30,621** ✅ |
| Unlabeled rows (current DB) | 19,890 (= 50,511 − 30,621) |
| Unlabeled decomposition | 18,926 (no tag) + 152 (only `world`) + 812 (non-`world` tags but none → country) |
| Paper's unlabeled **19,889** = its total **50,510** − 30,621 (Δ 1 = the known §A CSV artifact) |
| Post-vote vocabulary | **207** countries / **31** sub-regions / **6** continents ✅ |

Root causes of the original mismatch:
1. **Raw vs. standardized.** §B1 counted every row that carries *any* raw tag, including
   non-geographic noise (`Kosher`, `Creole`, `Fusion`, `Copycat`, `Inspired`, `Authentic`,
   `Asian Inspired`, `Southwestern`, `world` …) → +964 rows over the paper.
2. **Records vs. distinct names, and single vs. voted replica.** The previous §B2 used
   `COUNT(DISTINCT "Name")` on a single replica; the table contains **2,167 duplicate-name
   rows** (48,344 distinct names across 50,511 rows), and single-replica mappings are noisier,
   so distinct-name counts land ~2,000 below the paper.
3. **"world" is noise.** `world` fails parsing in all 3 replicas (`parsed = FALSE`; the LLM
   returned an oversized country dump visible in `content_raw`), so it contributes **zero**
   geography and never labels a recipe — i.e. it is naturally ignored by the vote.
## 4. Changes made to `confirm_numbers.ipynb`

1. **Replaced the §B verdict** (the §B2 verdict cell) — it now summarises the two naive
   definitions and points forward to §B3.
2. **Added the §B3 section** ("The mismatch, resolved — labels are counted after
   standardization", 8 cells: 4 markdown + 4 `%%sql`):
   - §B3.1 — rebuild the majority-vote gold mapping in SQL and count labeled **rows** → **30,621**;
   - §B3.2 — *BEFORE* table (raw tags incl./excl. `world`);
   - §B3.3 — *AFTER* mapping/vocabulary (109 survive / 107 to country / 207 / 31 / 6);
   - §B3.4 — *AFTER* labeled/unlabeled partition with the unlabeled decomposition;
   - closing markdown with the full breakdown table and the mismatch summary.
3. **Updated the §G verdict** — sub-regions = **exactly 31** under the vote (raw `output1`
   has 32; the single-occurrence `North Africa` spelling is dropped by the vote), continents = 6 before and after.
4. **Updated the §I summary** — rows B and G are now **✅ exact**, bottom line re-written.
5. Re-executed the **whole** notebook (`jupyter nbconvert --execute`): **0 errors**, 50 cells.

## 5. Troubles encountered

- **Records vs. distinct names (biggest confusion).** The same "≥ 1 tag → country" query
  returns 30,621 counted by *rows* and 28,569 counted by `COUNT(DISTINCT "Name")` because
  `silver.RecipeZenodo` holds duplicate recipe names. The pandas prototype and the SQL agreed
  once this was recognised; the paper's number matches the **record** count.
- **`UNNEST` + `parsed`.** "world" rows are `parsed = FALSE` in all three replicas, so their
  `country_list` is `NULL`. Filtering `WHERE parsed` correctly excludes them from the vote;
  a naive `UNNEST` over the raw lists contributes nothing for them anyway.
- **Notebook assembly.** Initially built the cells via a single script with inline strings;
  an off-by-one insertion corrupted the SQL strings. Rewrote the assembler to read one
  content file per cell from `/tmp/nb_cells/` (temp files deleted afterwards, see §7).

## 6. Things for the user to review

1. **Duplicate names.** `silver.RecipeZenodo` has 50,511 rows but only 48,344 distinct
   "Name" values (2,167 duplicate rows). The paper's counts behave as **record-based**;
   every downstream consumer should be aware which of the two is meant.
2. **"world" is unparseable, not just ignored.** The mission says "ignore world"; the data
   confirms it is *automatically* ignored because the LLM reply for `world` failed parsing in
   all 3 replicas (`parsed = FALSE`). If the pipeline is re-run with a stricter prompt,
   `world` could start producing geography and 30,621 could shift — deliberately excluding it
   in downstream code is still the safe choice.
3. **§G sub-regions.** The previous note flagged 32 raw vs 31 as a near-miss; under the vote
   it is **exactly 31**. The raw 32 is still visible in `output1` (spelling variants).
4. The 19,889 vs 19,890 unlabeled difference is the known **+1 row** CSV artifact (§A); it is
   not a labelling problem.

## 7. Temp files

All exploration/assembly scripts and per-cell content files were kept in `/tmp` and removed
on completion (see project convention in `agents.md`).