# Confirm the numbers in the ICOCO IEEE paper — Note

**Date**: 2026-09-27 (UTC)
**Notebook produced**: [`confirm_numbers.ipynb`](confirm_numbers.ipynb)
**Database**: `/workspace/ICOCO.duckdb` (read-only)
**Mission**: `5_Confirm_number/confirm_mission.md`

---

## 1. Mission

Read `ICOCO IEEE eXPress.pdf` and build a **reproducible SQL notebook** that confirms every
number in the paper's dataset-preprocessing sentence, using **only** the data in DuckDB, and
flag any mismatch in a report.

The sentence (Methodology §IV-A) to confirm:

> *"We utilize a culinary dataset from Zenodo [26] comprising **50,510 recipes** (**30,621
> geographically labeled** across **116 raw tags**, and **19,889 unlabeled**). Preprocessing
> standardized **17,010 raw ingredient strings** and **22 cooking methods** into **3,500
> canonical ingredients** alongside **207 countries**, **31 sub-regions**, and **6**
> continents."*

I produced the SQL notebook `confirm_numbers.ipynb` (42 cells, 15 read-only `%%sql` cells +
setup cells, all executed cleanly with **0 errors**). This note documents what I did, the
troubles I hit, the proof of completion, and the things worth the user reviewing.

---

## 2. What I did, what troubles I encountered, how I resolved them

### 2.1 Reading the PDF (no PDF libraries were installed)
- <font color="red">Consequence</font>: The container had no `pdftotext`, `pdfplumber`, or `PyPDF2`; I could not read the mission PDF at first.
- <font color="green">Resolve</font>: I installed `pypdf` (PIP) into the devcontainer and extracted the 6-page text to a temp file, then read it to isolate the exact methodology sentence and the 304-D framework description.

### 2.2 jupysql notebooks: `source` must be a single string (not a list)
- <font color="red">Consequence</font>: My first generated notebook stored each cell as a list of lines *without trailing newlines*. nbformat joins such a list with `''`, so `%load_ext sql` and `%config ...` were concatenated into one bogus module name (`sql%config SqlMagic`) and execution failed.
- <font color="green">Resolve</font>: Rebuilt the cell writer so each cell's `source` is a **single string with embedded `\n`** (fully valid nbformat), and added per-cell `id` fields to silence the nbformat `MissingIDFieldWarning`. The notebook then executed top-to-bottom with 0 errors.

### 2.3 DuckDB `STRUCT` coercion error on empty-string array elements
- <font color="red">Consequence</font>: `UNNEST(continent_list) ... WHERE co <> ''` threw `Conversion Error: ... can't be cast to STRUCT`. The `continent_list` column is a LIST OF STRUCTs, and one row contains a bare **empty string** element instead of a proper value; the `<>` filter coerced that element to a STRUCT and failed.
- <font color="green">Resolve</font>: I counted continents through an inner sub-query that casts the element to `VARCHAR` *before* filtering (`SELECT DISTINCT co::VARCHAR FROM (SELECT UNNEST(...) AS co ...)`), which sidesteps the coercion quirk. That empty-string element turned out to be exactly why a naive count says "7 continents" instead of the true **6**.

### 2.4 The terminal "hangs" on heredocs (`cat << EOF` / `python3 << EOF`)
- <font color="red">Consequence</font>: Left-running heredocs caused dead-looking terminal states and stale output while I prototyped the verification queries.
- <font color="green">Resolve</font>: For quick questions I switched to the **editor tool** on a temp `.py` file and to single-purpose `python3 -c` commands instead of multi-line heredocs, which removed the flakiness.

### 2.5 The labelled/unlabelled split does **not** reconcile (main finding)
- <font color="red">Consequence</font>: The paper's **30 621 / 19 889** does **not** match the current DB under any natural definition. I also initially mis-counted dishes using `COUNT(*)` over dish–tag pairs (**34 497**) instead of `COUNT(DISTINCT dish)` (**28 657**).
---

## 3. Can I prove I completed the tasks? (before / after)

The database was **not modified** (the notebook is read-only). "Before" = the raw tables; "after" =
the same tables, but every one of the **15 SQL cells** now runs and returns a verifiable number.
Proof points:

| # | Paper claims | Value from `confirm_numbers.ipynb` (current DB) | Verdict |
|---|---|---|---|
| A | 50,510 recipes total | **50,511** | ⚠ +1 (malformed-CSV artifact) |
| B | 30,621 geo-labeled / 19,889 unlabeled | (i) 31,585 / 18,926 tag-based; (ii) 28,657 / 21,854 country-resolved | ❌ largest mismatch |
| C | 116 raw tags | **116** | ✅ |
| D | 17,010 raw ingredient strings | 17,033 distinct (17,014 lower/trim); 17,017 feeds | ⚠ near-miss ≤0.1% |
| E | 22 cooking methods | **22** | ✅ |
| F | 3,500 canonical ingredients | 3,586 per-replica; 4,283 union (3,500 = majority-vote gold, not stored) | ⚠ different definition |
| G | 207 / 31 / 6 countries·sub-regions·continents | 207 / 32 raw(→~31) / 6 | ✅ / ⚠ / ✅ |
| H | 304-D = 248+28+28 | Entity=28 ✅ · local countries=249 (GBIF \|Y\|=248) · odour=50 free-text | ⚠ context |

**Proof it ran**: `jupyter nbconvert --execute` completed with **0 errors**; `nbformat` 4.5,
valid JSON, all cells carry ids, final cell closes the connection.

---

## 4. Things for you to review

### 4.1 §B labelled/unlabelled split is the real outlier (30 621 / 19 889)
The paper's split does **not** reproduce:
- tag-based (has a non-empty `Cuisine`) → 31 585 / 18 926 (total 50 511)
- country-resolution (`COUNT(DISTINCT dish)`, any tag → non-empty country) → 28 657 / 21 854

Neither equals 30 621. The paper's total (50 510) is also 1 less than current (50 511). I suspect
the paper used an **earlier, pre-curation snapshot** of the Zenodo data or a **different labelling
rule** than is encoded in `silver.RecipeZenodo`. This is the **only large mismatch**; everything
else is exact or ≤3% — please re-check it against your published pipeline/gold files.

### 4.2 "3,500 canonical ingredients" is a *consensus* number, not a raw count
The stored `ingredient_standardize.output{1,2,3}` give **3,586 / 3,594 / 3,585** per-replica and
**4,283** for the 3-replica union — none equal 3,500. The paper's 3,500 is the **majority-vote
(gold)** vocabulary, but that gold table is **not materialized** in this DuckDB (it lives in the
`old_code` gold-generation outputs). To *prove* 3,500 from DB-only sources, that table (or an
equivalent majority-vote view) would need to be added.

### 4.3 Sub-region "31" vs raw "32"; continents "6" vs raw "6 + empty-string"
- Sub-regions: the raw `cuisine_normalize.output1` emits **32** distinct strings (4 spellings of
  South-East Asia, `North Africa` vs `Northern Africa`). The paper's 31 is the **normalized**
  taxonomy; a raw DB count gives 32. Within tolerance, but be aware the raw mapping is noisy.
- Continents: one malformed row injects an **empty-string** continent, so a naive count = 7.
  Filtering it gives the correct 6. That empty-string element is the same STRUCT/coercion quirk
  from §2.3.

### 4.4 Odour track: paper says 28-D, the LLM produced 50 descriptors
The prompt fixed a **27-term** odour lexicon, the model returned **50** distinct free-text
descriptors (≈23 off-vocabulary). The paper's "28-D `V_odor`" is a *designed canonical* set, not
the raw LLM vocabulary, so they are not directly comparable. If a downstream consumer assumes
only the 27 approved terms, these 23 extras must be mapped or dropped.

### 4.5 Entity/geography dimension sizes
- Entity = **28** FoodUSDA groups: exact ✅.
- Geography: the paper's 248-D is the **GBIF** country set (`|Y|=248`); the local `countries`
  table has **249** rows. Different provenance — not an error, but worth noting if you compare
  the local ISO list to the paper's 248.

---

## 5. Files
- `confirm_numbers.ipynb` — the reproducible SQL notebook (this mission's output).
- `confirm_note.md` — this report.
- The notebook source builder and temp verification scripts were kept in `/tmp` and removed on
  completion, per the project's temp-file convention.
- <font color="green">Resolve</font>: I re-ran both queries side-by-side, confirmed `COUNT(DISTINCT "Name")` = **28 657 distinct dishes** whose *any* tag resolves to a country, corrected the note and summary, and flagged §B as the **largest mismatch** to be reviewed (see §4).