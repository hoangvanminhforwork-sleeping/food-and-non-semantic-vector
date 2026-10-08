Task Refinement & Clarification for Agent:

I need you to work on two Jupyter Notebooks:
1. Fix and update: `final/ICOCO_notebook_Colab.ipynb` (Targeted for Google Colab environment to ensure paper reproducibility: `/workspace/ICOCO IEEE eXPress.pdf`).
2. Create: `final/ICOCO_notebook.ipynb` (Targeted for running locally on my host machine).

---
### BACKGROUND & CORE ISSUE:
- The existing code in `final/ICOCO_notebook_Colab.ipynb` fails when trying to download raw data (FooDB and FooDUSDA) programmatically because the remote server blocks requests (even with custom User-Agent headers).

### CORE REQUIREMENTS & UPDATED CONSTRAINTS:

1. **Data & Pre-processed Artifacts Handling:**
   - **Reuse Existing Local Artifacts:** Any artifacts/files already computed or available locally on my machine (e.g., pre-calculated `.npy` files or embeddings) should be reused directly instead of re-generating them.
   - **Colab/GitHub Release Fallback:** Assume all necessary pre-processed data files and raw data from the local `final/data/` directory will be hosted in a **GitHub Release**. 
   - Update the data fetching logic so that when executed on Google Colab, it loads these `.npy` and data files directly (either from working directory or downloaded from GitHub Release endpoints).

2. **Execution Exclusions:**
   - **Do NOT run KMeans:** Skip any KMeans clustering execution.
   - **Do NOT clone or run FoodBERT:** Avoid executing FoodBERT scripts/models.
   - You may use HTTP libraries (e.g., `requests`, `httpx`) solely to verify data readiness/availability if needed.

3. **Notebook Deliverables:**
   - `final/ICOCO_notebook.ipynb`: Ensure it works seamlessly on local host using local existing files (`.npy`, etc.).
   - `final/ICOCO_notebook_Colab.ipynb`: Keep structural parity with the local notebook, but adapt paths/downloads for Google Colab compatibility.

Please inspect the local files (including `.npy` files and current notebook structure) and begin updating the local notebook first, then port the changes to the Colab version.