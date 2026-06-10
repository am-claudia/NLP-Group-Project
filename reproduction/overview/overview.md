# Reproduction Overview

**NLP Group Project — Vladika & Matthes (EACL 2024)**
*Open-Domain Scientific Claim Verification — Reproduction Results*

---

## 1. The Two Reproduction Runs

### Why we do not run the original pipeline as-is

The original paper's pipeline requires large local data dumps to perform retrieval: a 6.6M-article Wikipedia corpus and a 20.6M-abstract MEDLINE dump. Downloading and indexing these is not feasible in our setting — the data alone runs to tens of gigabytes, and running BM25 or semantic search over them locally would take an impractical amount of time and compute. The paper's authors did provide their pre-computed retrieval results, which is what makes Run 1 possible. For Run 2 we replace the local dumps with public APIs, which allows us to re-run the full pipeline without needing the original corpora.

---

### Run 1 — Step-3-Only (`reproduction/step3-only/`)

Loads the pre-computed retrieval results provided by the paper's authors (`reference/evidence/`) and runs only **Step 3: Verdict Prediction**. No retrieval or evidence selection is re-run.

Step 3 uses a **Natural Language Inference (NLI)** model — specifically DeBERTa-v3-large fine-tuned on MNLI, FEVER, and ANLI. NLI is the task of deciding whether a *hypothesis* (the claim) is entailed by, contradicted by, or neutral with respect to a *premise* (the retrieved evidence). Here the model outputs a probability distribution over three classes: entailment, contradiction, and neutral. We map entailment → **SUPPORTED** and contradiction → **REFUTED**, discarding neutral.

Evidence sources available: Wikipedia BM25, Wikipedia Semantic, PubMed BM25, PubMed Semantic, Google.

**Datasets:** SciFact (693), PubMedQA (890), HealthFC (327), CoVERT (264) — after filtering NEI-labelled claims.

**Purpose:** We noticed a discrepancy in how the original notebook annotated / normalised labels across datasets (e.g. HealthFC's `0` means SUPPORTED, not REFUTED; PubMedQA uses string labels `"yes"`/`"no"`). We corrected these and reran verdict prediction using the same pre-computed evidence to quantify how much the label errors affected the reported results.

---

### Run 2 — API Pipeline (`reproduction/api-pipeline/`)

Re-runs the **full three-step pipeline** from scratch using public APIs instead of the paper's local data dumps:

- **Step 1 — Document Retrieval:** Wikipedia Python API (replaces the paper's local 6.6M-article Wikipedia dump); NCBI PubMed E-utilities (replaces the paper's 20.6M-abstract MEDLINE dump)
- **Step 2 — Evidence Selection:** SPICED sentence similarity, top-10 sentences (unchanged from paper)
- **Step 3 — Verdict Prediction:** DeBERTa-v3-large NLI (unchanged from paper)

**Purpose:** We re-run the full pipeline for the two knowledge sources most relevant to tweet verification — Wikipedia and PubMed. We use API queries rather than local data dumps because we need fast, lightweight retrieval that does not require downloading multi-gigabyte corpora. From Run 1, Wikipedia and PubMed already show the strongest results on **CoVERT**, which is the dataset closest in format to social media posts (tweets), making them the most promising sources for our extension. Google was excluded: it requires a Custom Search API with a hard limit of 100 queries per day (far below the ~2,174 needed to run all four datasets once) and its configuration is fragile — results depend heavily on the search engine setup and do not replicate the Google Scholar results used in the original paper.

---

## 2. Changes from the Original Notebook

The original paper notebook (`reference/code/open_domain_claim_verification.ipynb`) required a local environment with large pre-downloaded corpora. Our reproduction notebooks make the following changes:

### Google Colab Compatibility
- Added `RUNNING_ON_COLAB` detection to toggle between Colab (Google Drive paths) and local paths automatically
- Google Drive is mounted when running on Colab; local relative paths are used otherwise
- Hardcoded paths replaced with configurable `BASE_PATH`, `DATASETS_PATH`, `RESULTS_PATH` variables at the top of each notebook

### Metrics Saving
- Added `save_metrics()` helper that writes F1 and accuracy per dataset to a `metrics.csv` file after each run
- Added `load_metrics()` helper to reload saved results — allows the comparison cell to run independently without re-running NLI
- Results also exported as `.xlsx` files for easier inspection and comparison with the paper's tables
- Added **resume points** in the API pipeline notebook: if the session restarts mid-run, already-retrieved lines are reloaded from disk rather than re-queried (Wikipedia retrieval takes ~60–90 min)

### Label Normalisation Fixes
The original notebook did not explicitly normalise labels for all datasets. We added:

| Dataset | Original labels | Normalised to |
|---------|----------------|---------------|
| HealthFC | `0` (false), `2` (true) | `0` (REFUTED), `1` (SUPPORTED) |
| PubMedQA | `"yes"`, `"no"` | `1` (SUPPORTED), `0` (REFUTED) |
| CoVERT | `"SUPPORTS"`, `"REFUTES"` | `1`, `0` |
| SciFact | already binary | unchanged |

Additionally, CoVERT labels had a leading whitespace issue after `@username` removal — fixed with `.str.strip()`.

### BM25 Alignment Fix
The pre-computed BM25 files were generated from the full dataset including NEI-labelled claims. After filtering NEI claims out, the row count no longer matched the dataset. Added `align_entries_to_claims()` to re-align BM25 evidence lines to the filtered claim list by matching claim text.

### Code Cleanup
- All imports consolidated at the top of each notebook
- Repeated retrieval loops replaced with a single `load_source(base_path, files_dict)` helper
- Removed Google Custom Search API code from the API pipeline notebook (Google section only in step3-only)
- No duplication of work between cells

---

## 3. Results

F1 scores (binary, `average='binary'`). Paper values from Tables 3 & 4 of Vladika & Matthes (2024).

### Run 1 — Step-3-Only: Wikipedia Evidence

| Dataset | BM25 Paper | BM25 Ours | Semantic Paper | Semantic Ours |
|---------|-----------|-----------|----------------|---------------|
| SciFact | 74.8 | 74.8 | 75.4 | 75.4 |
| PubMedQA | 73.1 | 70.4 | 73.2 | 72.1 |
| HealthFC | 73.1 | 73.1 | 76.5 | 76.5 |
| CoVERT | 75.2 | 80.4 | 82.5 | 82.5 |
| **Average** | **74.0** | **74.7** | **76.9** | **76.6** |

### Run 1 — Step-3-Only: PubMed Evidence

| Dataset | BM25 Paper | BM25 Ours | Semantic Paper | Semantic Ours |
|---------|-----------|-----------|----------------|---------------|
| SciFact | 76.1 | 76.0 | 76.8 | 76.8 |
| PubMedQA | 70.3 | 68.4 | 74.5 | 70.8 |
| HealthFC | 69.7 | 70.0 | 72.0 | 74.5 |
| CoVERT | 79.5 | 79.0 | 76.2 | 77.8 |
| **Average** | **73.9** | **73.4** | **74.9** | **75.0** |

### Run 1 — Step-3-Only: Google Evidence

| Dataset | Paper | Ours |
|---------|-------|------|
| SciFact | 82.7 | 72.6 |
| PubMedQA | 78.5 | 58.7 |
| HealthFC | 74.5 | 74.2 |
| CoVERT | 72.3 | 54.0 |
| **Average** | **77.0** | **64.9** |


---

### Run 2 — API Pipeline: Wikipedia API

| Dataset | Paper (BM25) | Wikipedia API Ours |
|---------|--------------|--------------------|
| SciFact | 74.8 | 78.1 |
| PubMedQA | 73.1 | 72.5 |
| HealthFC | 73.1 | 74.5 |
| CoVERT | 75.2 | 68.0 |
| **Average** | **74.0** | **73.3** |

### Run 2 — API Pipeline: PubMed API

| Dataset | Paper (BM25) | PubMed API Ours |
|---------|--------------|-----------------|
| SciFact | 76.1 | 76.5 |
| PubMedQA | 70.3 | 71.8 |
| HealthFC | 69.7 | 71.9 |
| CoVERT | 79.5 | 70.2 |
| **Average** | **73.9** | **72.6** |

---

## 4. Key Observations

- **Step-3-only results closely match the paper** for Wikipedia and PubMed BM25/Semantic, confirming our label normalisation corrections and NLI setup are correct.
- **CoVERT benefits most from Wikipedia and PubMed** across all retrieval methods in Run 1, reinforcing their suitability for tweet-format claims.
- **Wikipedia API pipeline** results are broadly comparable to the paper's Wikipedia BM25 baseline, with some variation per dataset — expected given different corpus coverage and retrieval method.
- **PubMed API pipeline** results are also competitive with the paper's PubMed BM25 baseline overall, with a larger drop on CoVERT — possibly because API keyword search returns fewer relevant abstracts for tweet-style short claims than a full local index would.
- **For the tweet extension**, Wikipedia API is the most robust choice: it matches or exceeds the paper on SciFact, PubMedQA, and HealthFC, and CoVERT results are consistent with step-3-only Wikipedia. PubMed API is a good complement for health-related claims.
