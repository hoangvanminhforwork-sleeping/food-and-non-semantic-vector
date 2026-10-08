# FINAL MISSION — One huge Colab notebook to reproduce the ICOCO paper

**Target runtime:** Google Colab · **Code style:** "dry code" (authored here, executed later by the user) · **Companion doc:** `recommended_huge_notebook_structure.md`

## 1. Objective

Build **a single, extremely large Jupyter notebook** that reproduces the **complete scientific result** of the ICOCO paper — *"Reducing Generative AI Bias in Food Recognition Using a Multi-Dimensional Culinary Representation Framework"* (`ICOCO IEEE eXPress.pdf`) — from the raw datasets all the way to the paper's figures.

The notebook must rebuild the paper's **304-dimensional non-semantic recipe representation**
(`248 geography + 28 entity + 28 odor`) and everything that feeds or evaluates it:

- LLM canonicalization + database grounding (ingredient standardization, taxonomic extraction)
  and its metrics (Self-Consistency, Input Coverage, Adjusted Coverage, Queryability);
- GBIF species matching;
- the three vectors **plus** the FoodBERT baseline embeddings;
- the K-means "battle" experiment (external cultural alignment + internal geometry);
- the paper's figures.

## 2. How it will be run (read carefully)

- **Do NOT execute the notebook on this machine.**
- Write **"dry code"**: it is authored here but only ever run by the user, who will **paste it into
  Google Colab** and then **fix any bugs by hand there**.
- Design for *"runs top-to-bottom in a fresh Colab runtime"*; a few cell-level fixes later are
  expected and acceptable.
- **Success criterion: "execute all without error."** Every cell should be written so it completes.

## 3. Hard constraints

1. **One notebook** covering the whole pipeline (it is deliberately "huge").
2. **Minimize markdown.** Keep markdown cells as short as possible (no cognitive overload):
   one section title, at most a one-line note.
3. **SQL-first.** Prefer `%sql` / `%%sql` (DuckDB via jupysql) over pandas; use Python only where
   SQL cannot do the job (e.g. `ast.literal_eval`, the `.npy` scatter, model inference, plotting).
4. **No bronze/silver layering.** Load the source data and go **straight to the final tables**.
5. **Idempotent SQL.** Always `CREATE OR REPLACE` / `CREATE SCHEMA IF NOT EXISTS` /
   `INSERT OR REPLACE`, so the notebook can be re-run without corrupting or duplicating state.
6. **Install only what Colab lacks.** `numpy, pandas, scipy, matplotlib, seaborn, plotly,
   scikit-learn, torch, transformers, tqdm` are preinstalled → do **not** reinstall them.
   Install the rest (e.g. `duckdb`, `duckdb-engine`, `jupysql`, `requests`).
7. **Delete all temporary files** created during this working session before finishing.

## 4. Deliverables

- The single huge notebook (`.ipynb`).
- A short note describing what was done and any issues, following `agentTemplates.md`.

## 5. Reference material

| Reference | Path |
|---|---|
| Paper — workflow, formulas and target numbers | `ICOCO IEEE eXPress.pdf` |
| Raw structure idea (origin of this mission) | `merge_notebook_template.md` |
| **Recommended notebook structure — read first** | `recommended_huge_notebook_structure.md` |
| New clean pipeline (reference, split by folder) | `0_upload_data_to_DuckDB` … `8_create_battle` |
| Legacy pipeline (read-only reference for late stages) | `old_code/` |
| Working conventions | `agents.md`, `agentTemplates.md` |

## Appendix — original notes (verbatim)

> tạo một notebook cực lớn
> giảm số dòng trong markdown cell đến mức tối đa, ko gây cognitive quá lớn
> reproduce thành quả khoa học
> ko chạy code trên máy này, chỉ cần cố gắng thử tạo ra một notebook siêu lớn, sau đó là tui sẽ paste file này lên google colab, rồi mới chạy, lúc đó tui sẽ tự tay fix bug. Nói chung là...viết code chay đó
> markdown này sẽ là chạy trên google colab
> Ở khoản upload data, thì ko cần chia schema bronze, silver gì hết nhé. thẳng vào data cuối cùng luôn
> ở file setup cấu hình, thì tải những library nào cần thiết thôi, những thư viện tiêu biểu đã có trên google colab như numpy, pandas...thì thôi, còn những cái khác thì phải tải
> Ưu tiên SQL format
> Idempotency: For SQL, always use "if not exist", "create or replace"...etc, so I can run the notebook multiple times without changing the state of the data so much
> Xóa các file temp trong toàn bộ phiên làm việc này
> Execute all ko lỗi là dc...yên
> file pdf: ICOCO IEEE eXPress.pdf
