# Technical Report: Re-running the Reference Code

This document records observations, costs, and practical options for re-running the pipeline from Vladika & Matthes (EACL 2024) — *"Comparing Knowledge Sources for Open-Domain Scientific Claim Verification"*.

---

## 1. Cost Warning: Full Re-run Is Prohibitively Expensive

Re-running the complete pipeline from scratch is not realistic on consumer hardware or a free cloud tier. The three main bottlenecks are:

| Step | Operation | Original cost (paper) | Why it is slow |
|---|---|---|---|
| 1 | Encode 20.6M PubMed abstracts with BioSimCSE | Ran overnight on a single Nvidia V100 (16 GB VRAM) | Each abstract must pass through a transformer encoder; 20.6M × 768-dim vectors |
| 1 | Build BM25 inverted index over PubMed | ~1.6 hours CPU | Tokenizing and indexing 20.6M documents |
| 2 | Evidence sentence selection (SPICED model) | ~1 hour per dataset on GPU | Pairwise sentence scoring across 10 retrieved docs per claim |
| 3 | Verdict prediction (DeBERTa-v3-large NLI) | ~1 hour per dataset on GPU | Large model inference over hundreds of claims |

**Storage alone is a blocker:** the PubMed MEDLINE 2022 dump is tens of gigabytes. The resulting embedding matrix (20.6M × 768 float32 values) requires approximately **60 GB of RAM** to hold in memory — more than most laptops and all free cloud tiers.

---

## 2. Options to Speed Things Up

### Option A — Use the Pre-computed Results (Fastest, No GPU Needed)

The `reference-code/Original Results/` folder already contains the output of steps 1 and 2 for every dataset, knowledge source, and retrieval method. Each file is in `claim [SEP] evidence1 evidence2 ...` format, one line per claim.

You only need to run **step 3** (verdict prediction). This takes ~10–20 minutes on a mid-range GPU, and even runs on CPU in under an hour for one dataset.

**What you can still vary without re-running retrieval:**
- Swap the NLI model (e.g., use a lighter BERT-based model vs. DeBERTa-v3-large)
- Change the prediction threshold (entailment vs. contradiction probability cutoff)
- Experiment with different label mappings

**Limitation:** you cannot change the retrieval parameters (k, embedding model, corpus) without re-running steps 1–2.

---

### Option B — Google Colab

**Short answer:** Colab can run step 3 (verdict prediction) and step 2 (evidence selection) comfortably. It cannot realistically run step 1 for PubMed at full scale.

| Task | Free Colab (T4, 16 GB VRAM, ~13 GB RAM) | Colab Pro+ (A100, 40 GB VRAM, ~83 GB RAM) |
|---|---|---|
| Step 3: Verdict prediction (DeBERTa-v3-large) | Yes — fits on T4 | Yes |
| Step 2: Evidence selection (SPICED) | Yes — fits on T4 | Yes |
| Step 1: BioSimCSE encoding of full 20.6M PubMed corpus | No — embedding matrix (~60 GB) exceeds RAM; sessions also time out before finishing | Borderline — A100 has enough VRAM but storage and session limits are still a problem |
| Step 1: BM25 index over full PubMed | No — 1.6-hour index build plus persistent disk storage needed (~40 GB+) | Possible with Google Drive mount, but slow and fragile |

**Practical Colab workflow:**
1. Mount Google Drive and upload the `Original Results/` and `Datasets/` folders there.
2. Run only the verdict prediction cell from the notebook — this is the only step that needs a GPU and is fast enough for Colab.
3. For BM25 or semantic search experiments at a smaller scale (e.g., on the 693-claim SciFact dataset against a 500k-abstract subset), Colab is viable.

---

### Option C — Use a Smaller Corpus Subset

Rather than 20.6M PubMed abstracts, run retrieval over a curated subset:

- **SciFact-Open** (Wadden et al., 2022) provides ~500k biomedical abstracts pre-selected for scientific claim verification — this is a manageable size.
- Encoding 500k abstracts with BioSimCSE takes ~30–60 minutes on a T4 and fits in Colab RAM.
- The embedding matrix (500k × 768 × 4 bytes) is ~1.5 GB — easy to save to Google Drive and reload.

This means you can re-run the full pipeline end-to-end for SciFact on a free Colab session in a few hours, rather than days.

---

### Option D — Use FAISS for Efficient Vector Search

The notebook loads all embeddings into a single NumPy array and computes cosine similarity in one shot. This requires all ~60 GB in RAM at once. Switching to [FAISS](https://github.com/facebookresearch/faiss) (Facebook AI Similarity Search) allows index-based approximate nearest-neighbour search on disk:

```python
import faiss
import numpy as np

# Build a flat (exact) or IVF (approximate, faster) index
d = 768  # embedding dimension
index = faiss.IndexFlatIP(d)  # inner product = cosine sim on normalized vectors
index.add(all_embeddings)  # add in batches to avoid OOM

# Search
query_embedding = model.encode([claim])
faiss.normalize_L2(query_embedding)
distances, indices = index.search(query_embedding, k=10)
```

FAISS can keep the index on disk (using `IndexIVFFlat` + `faiss.write_index`) and search without loading everything into RAM. This makes the full PubMed index feasible on machines with ~16 GB RAM.

---

### Option E — Use the Hugging Face Inference API (No Local GPU)

For step 3 only, the DeBERTa-v3-large NLI model is hosted on the Hugging Face Hub. You can call it via API without downloading model weights:

```python
import requests

API_URL = "https://api-inference.huggingface.co/models/MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli"
headers = {"Authorization": "Bearer YOUR_HF_TOKEN"}

def query(payload):
    response = requests.post(API_URL, headers=headers, json=payload)
    return response.json()

output = query({"inputs": "claim text [SEP] evidence text"})
```

Free tier has rate limits (~1 request/second), so this is slow for hundreds of claims but requires zero local compute.

---

### Option F — Use a Lighter NLI Model for Step 3

DeBERTa-v3-**large** is a 400M-parameter model. Swapping to a smaller variant reduces inference time significantly with modest F1 drop:

| Model | Size | Approx. speed vs. DeBERTa-v3-large |
|---|---|---|
| `MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli` (original) | ~400M params | 1× (baseline) |
| `MoritzLaurer/deberta-v3-base-mnli-fever-anli` | ~180M params | ~2–3× faster |
| `cross-encoder/nli-deberta-v3-small` | ~44M params | ~8–10× faster |
| `typeform/distilbert-base-uncased-mnli` | ~67M params | ~6× faster |

---

## 3. Modified Notebook — Wikipedia API Version

A revised copy of the notebook was created at `reference-code/Code/open_domain_claim_verification_wikipedia_api.ipynb`. The changes relative to the original are documented below.

### Step 1 — Document retrieval (replaced)

The original code downloads the full Wikipedia dump (6.6M articles), encodes every article with BioSimCSE, and searches with either cosine similarity or a BM25 inverted index. This was replaced with the `wikipedia` Python library:

```python
def retrieve_wikipedia_articles(claim, n_results=10):
    titles = wikipedia.search(claim, results=n_results)
    for title in titles:
        page = wikipedia.page(title, auto_suggest=False)
        articles.append(page.content)
        time.sleep(0.3)  # respect rate limits
    return articles
```

This requires no download and no GPU for retrieval. The Wikipedia API's search is keyword-based, making results comparable to the **Wikipedia BM25** rows in Table 3 of the paper — not the semantic search rows.

### Steps 2 & 3 — Unchanged

Evidence selection (SPICED) and verdict prediction (DeBERTa-v3-large) are identical to the original.

### Bug fixes applied to the original code

The original notebook contained several bugs that would produce incorrect evaluation metrics. These were identified by inspecting the actual label values in each dataset CSV and cross-checking with the DeBERTa prediction format (1 = SUPPORTED, 0 = REFUTED):

| Dataset | Bug in original | Fix applied |
|---|---|---|
| **HealthFC** | Labels in CSV are `{0=SUPPORTED, 1=NEI, 2=REFUTED}` but were passed directly to sklearn, causing a mismatch with predictions `{0, 1}` | Remapped: `0→1`, `2→0` |
| **CoVERT** | `covert_labels = [mapper[l] for l in labels]` — used undefined variable `labels` instead of `covert_labels` | Fixed variable name |
| **PubMedQA** | `'maybe'` (NEI) exclusion was implicit and could silently include unmapped labels | Made explicit with `.isin(['yes', 'no'])` filter |

### Saving results

The original notebook had save code scattered inline through each dataset block, writing to hardcoded paths on the authors' machine (`/mnt/mydrive/...`). These files were then manually committed to the GitHub repo as the `Original Results/` folder — they were not saved automatically.

The modified notebook adds three dedicated save cells that write results in the same formats as the originals, to `reference-code/Results/Wikipedia API/`:

| File saved | Format | Equivalent in Original Results |
|---|---|---|
| `{dataset}_wiki_api_titles.txt` | 3 lines per claim: claim / [title1, title2, ...] / blank | `*_pmids.txt` (PMIDs replaced by Wikipedia titles) |
| `{dataset}_wiki_api_lines.txt` | 1 line per claim: `claim [SEP] evidence...` | `*_lines.txt` |
| `metrics.csv` | CSV with precision, recall, F1 per dataset | Not in original (metrics only printed to screen) |

The `RESULTS_PATH` variable at the top of the notebook controls where files are saved. On Colab, set it to the corresponding Google Drive path.

### Running on Google Colab

The notebook detects its environment automatically — no manual changes are needed when switching between local and Colab.

**Environment auto-detection:**
```python
try:
    import google.colab
    RUNNING_ON_COLAB = True
except ImportError:
    RUNNING_ON_COLAB = False
```
When on Colab, all paths point to Google Drive. When local, relative paths are used.

**Drive mount auto-detection:** The Drive mount cell also handles itself — it mounts on Colab and silently skips the step locally:
```python
try:
    from google.colab import drive
    drive.mount('/content/drive')
except ImportError:
    pass  # running locally, nothing to do
```

**pip install:** Only `wikipedia`, `sentence-transformers`, and `transformers` are installed via pip. `torch`, `numpy`, `pandas`, `scikit-learn`, and `nltk` are excluded because they come pre-installed on Colab with GPU support — reinstalling them could overwrite Colab's GPU-enabled PyTorch with a CPU-only version. When running locally, install all packages manually:
```bash
pip install wikipedia sentence-transformers transformers torch nltk scikit-learn pandas numpy
```

The notebook can be run via the VSCode Colab extension without opening the browser interface: connect to Colab from the kernel selector, then run cells normally from VSCode.

**Required Drive folder structure before running on Colab:**
```
My Drive/
└── NLP-Group-Project/
    └── reference-code/
        ├── Code/
        │   └── open_domain_claim_verification_wikipedia_api.ipynb
        └── Datasets/
            ├── scifact_no-nei_dataset.csv
            ├── pubmedqa.json
            ├── healthFC_annotated.csv
            └── CoVERT_FC_annotations.jsonl
```

### What the modified notebook covers

- All 4 original datasets (SciFact, PubMedQA, HealthFC, CoVERT)
- A `DATASETS_TO_RUN` list at the top lets you skip datasets to save time
- A `RUNNING_ON_COLAB` toggle switches all paths automatically
- Evaluation prints a summary table for direct comparison with Table 3 of the paper
- All intermediate and final results are saved automatically to `Results/Wikipedia API/`

### PubMed API section

A PubMed section was added after the Google section to replicate the **PubMed BM25** rows in Table 3. The original notebook loads a local 20.6M-abstract MEDLINE dump — replaced here with the **NCBI E-utilities API** via Biopython, which queries the same PubMed database.

The structure is identical to the Wikipedia API section:

| Cell | Purpose |
|---|---|
| Setup cell | Sets `Entrez.email` (required by NCBI), optional `Entrez.api_key`, and `PUBMED_RESULTS_PATH` |
| `retrieve_pubmed_articles()` | Calls `Entrez.esearch()` to get PMIDs, then `Entrez.efetch()` for XML, parses title + abstract. Mirrors `retrieve_wikipedia_articles()` |
| Retrieval loop | Iterates over all claims, saves `{dataset}_pubmed_api_pmids.txt` (same 3-line format as `*_wiki_api_titles.txt`) |
| SPICED evidence selection | Reuses `evidence_model` and `select_top_sentences()` already loaded; saves `{dataset}_pubmed_api_lines.txt` |
| Verdict prediction + metrics save | Reuses `predict_verdicts()`, `nli_model`, `nli_tokenizer`; saves `Results/PubMed API/metrics.csv` |
| Full comparison table | Prints F1 for all three knowledge sources side-by-side with paper reference values from Tables 3 and 4 |

**Rate limit:** 3 requests/second (free); 10 req/second with a free NCBI API key. Running all 4 datasets takes ~25 minutes at the free tier.

**pip install:** `biopython` was added to the install cell alongside the existing packages.

---

### Google Custom Search section

A Google section was added after the Wikipedia API section to replicate **Table 4** of the paper. The changes were kept as close as possible to the original notebook.

**What was added (no changes to the Wikipedia cells):**

| Cell | Purpose |
|---|---|
| Credentials cell | Sets `GOOGLE_API_KEY`, `GOOGLE_CSE_ID`, and `GOOGLE_RESULTS_PATH` |
| `search_google()` function | Mirrors the original paper's `search()` function (returns raw API response; uses `HttpError` for error handling) |
| Retrieval loop | Iterates over all claims, calls `search_google()`, saves raw results to `{dataset}_google_lines.txt` |
| Evidence builder | Concatenates snippets directly — no SPICED step (per paper: *"top 10 returned Google snippets concatenated as evidence"*) |
| Verdict prediction | Reuses the same `predict_verdicts()` function and `nli_model` / `nli_tokenizer` already loaded |
| Metrics save | Writes `metrics.csv` to `Results/Google/` |
| Comparison table | Prints F1 for Wikipedia API and Google side-by-side with the paper's Table 3 and Table 4 reference values |

**Format of saved Google results (`Results/Google/{dataset}_google_lines.txt`):**
```
CLAIM
<claim text>
EVIDENCE
<title>
<url>
<snippet>
...
[blank line]
```
If no results were returned: `No results found!` (with exclamation mark, matching the original).

**Rate limit:** The free tier allows 100 queries/day. Running all 4 datasets requires ~2,174 queries.

---

## 4. Summary Recommendation

| Goal | Recommended approach |
|---|---|
| Reproduce the paper's exact numbers | Use pre-computed `Original Results/` + run only step 3 locally or on Colab |
| Re-run retrieval at scale | Option C (500k subset) + Colab Pro, or Option D (FAISS) on a machine with ≥32 GB RAM |
| Experiment with the NLI step quickly | Option F (lighter model) + Option A (pre-computed evidence) |
| No GPU at all | Option E (HF Inference API) for step 3 only |

---

## 5. Project Scope Decision: BM25 Only, 100 Queries Per Dataset

For the replication and extension runs in this project, we use **BM25 retrieval only** and cap each dataset at **100 queries**. This section documents the rationale.

### Why BM25 only (no semantic search)

The reference notebook implements both retrieval methods. We drop semantic search for the following reasons:

**Performance parity.** The reference notebook reports near-identical F1 on SciFact for both methods (BM25: 0.761, semantic: 0.768 — a gap of <0.01). BM25 actually achieves *higher precision* (0.80 vs. 0.74), which is particularly relevant for fact-checking where retrieving irrelevant documents as "evidence" directly harms verdict quality.

**Infrastructure cost.** Semantic search requires:
- Encoding 20.6M abstracts overnight on a V100 GPU
- Storing the resulting embedding matrix (~60 GB)
- Loading it fully into RAM at query time

BM25 requires only the inverted index, which is built once (~1.6 hours CPU) and then queries 1,000 claims in milliseconds — no GPU, no large RAM requirement.

**Debuggability.** Keyword-based retrieval is transparent: you can inspect exactly which terms matched which documents. This makes failure analysis (a required deliverable) tractable. Cosine similarity over 768-dim embeddings is harder to interpret qualitatively.

**Extension dataset fit.** Our social media posts are informal, noisy text. Both methods are expected to struggle with this distribution shift; there is no strong a priori reason to prefer the added complexity of semantic search here.

### Why 100 queries per dataset

**Proportionality with the extension set.** Our extension dataset is 60 claims (30 English + 30 Spanish). Running 100 claims from SciFact gives a directly comparable baseline — it is already larger than the extension set and avoids over-weighting the replication relative to our novel contribution.

**Statistical sufficiency.** 100 labelled examples is enough to compute F1, precision, recall, and bootstrap confidence intervals at a resolution appropriate for a course project report.

**Consistency with the Google API free-tier limit.** The Google Custom Search API allows 100 queries/day on the free tier (see section 3). Although we are not using Google Search for the primary runs, 100 queries is the natural ceiling that the overall project was scoped around.

**Reproducibility.** A fixed 100-claim slice (e.g., the first 100 claims of the SciFact test set, in CSV order) is easy to state precisely, easy to re-run, and easy for a reader to verify against the public dataset.

### Reporting note

When reporting replication numbers, state explicitly: *"evaluated on the first 100 claims of the SciFact test set"*. This distinguishes our figures from the paper's full-test-set numbers (which used all ~300 SciFact test claims) and lets readers interpret any gap correctly.
