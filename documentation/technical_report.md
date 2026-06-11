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

---

## 6. Extension Results

### Setup

The replicated 3-step pipeline (document retrieval → SPICED evidence selection → DeBERTa-v3-large NLI) was applied **unchanged** to a new dataset of **60 bilingual social media posts about scientific claims** (30 English, 30 Spanish), annotated by the team (see `data/Annotation_Guidelines_Bilingual.md`, Cohen's κ = 0.54 for English, κ = 0.898 for Spanish).

- **English** posts use the `Tweet Text (English)` column directly.
- **Spanish** posts use the `MT Translation (EN)` column (machine-translated to English by Claudia), since the retrieval models and DeBERTa NLI model are English-only.
- **Knowledge sources:** both **Wikipedia API** and **PubMed API** were run, since the dataset mixes general and health-related scientific claims (cf. `reproduction/overview/overview.md`, §4: *"Wikipedia API is the most robust choice... PubMed API is a good complement for health-related claims"*).
- **Wikipedia retrieval fix:** an initial run returned almost no Wikipedia evidence for *any* claim, in either language. The cause was the `wikipedia` PyPI package's default `User-Agent` header (`wikipedia (https://github.com/goldsmith/Wikipedia/)`), which Wikimedia now rate-limits/blocks (HTTP 429 "You are making too many requests", or HTTP 403 with no User-Agent at all — see Wikimedia's [User-Agent policy](https://meta.wikimedia.org/wiki/User-Agent_policy)). Setting a unique, descriptive `User-Agent` (`pipeline/open_domain_claim_verification_tweets.ipynb`, Setup cell) fixed the API-blocking issue. The results below were produced with this fix applied — but as the "Retrieval coverage" section shows, fixing the *blocking* issue exposed *further*, dataset-specific retrieval problems that the User-Agent fix alone does not solve.

### Evaluation methodology

Each of the 60 posts was annotated **TRUE**, **FALSE**, or **UNVERIFIABLE**. Precision, recall, F1, and accuracy are computed on the **TRUE/FALSE subset only** — UNVERIFIABLE posts have no SUPPORTED/REFUTED ground truth, so they cannot contribute to a binary classification score. This is the same convention used for NEI-labelled claims in SciFact/PubMedQA/HealthFC/CoVERT (§5 and `documentation/step3_notebook_notes.md`).

| Dataset | TRUE | FALSE | UNVERIFIABLE | Scoreable (TRUE+FALSE) |
|---|---|---|---|---|
| English tweets | 16 | 3 | 11 | 19 / 30 |
| Spanish tweets | 13 | 9 | 8 | 22 / 30 |

The pipeline still produces a SUPPORTED/REFUTED prediction for **every** post, including UNVERIFIABLE ones — these are retained in `*_predictions.csv` and reviewed in the Failure Analysis (§7) below, since a forced verdict on a claim that cannot be verified is itself an informative failure mode.

### Results

| Knowledge source | Dataset | Precision | Recall | F1 | Accuracy | Scored |
|---|---|---|---|---|---|---|
| Wikipedia API | SciFact (baseline) | 73.5 | 83.3 | 78.1 | — | — |
| Wikipedia API | CoVERT (baseline) | 79.2 | 59.6 | 68.0 | — | — |
| Wikipedia API | tweets_en | 100.0 | 68.8 | 81.5 | 73.7 | 19/30 |
| Wikipedia API | tweets_es | 62.5 | 76.9 | 69.0 | 59.1 | 22/30 |
| PubMed API | SciFact (baseline) | 75.3 | 77.6 | 76.5 | — | — |
| PubMed API | CoVERT (baseline) | 79.1 | 63.1 | 70.2 | — | — |
| PubMed API | tweets_en | 100.0 | 87.5 | 93.3 | 89.5 | 19/30 |
| PubMed API | tweets_es | 62.5 | 76.9 | 69.0 | 59.1 | 22/30 |

*(Source: `pipeline/results/comparison/extension_vs_baseline.csv`. Baseline rows are copied from `reproduction/api-pipeline/results/{Wikipedia API,PubMed API}/metrics.csv`, which do not report accuracy or scored counts.)*

At face value, `tweets_en` (PubMed API) beats both baselines on F1, and `tweets_en` (Wikipedia API) beats CoVERT. Both `tweets_es` rows trail CoVERT and SciFact by ~7-9 F1 points. The "Retrieval coverage" and "Comparison against a trivial baseline" subsections below show why these headline numbers should **not** be read at face value.

### Retrieval coverage: how much evidence did the pipeline actually find?

For each claim, Step 1 (document retrieval) either returns ≥1 article (whose sentences feed Step 2/3) or returns nothing — in which case the DeBERTa input degenerates to `"<claim> [SEP] "`, i.e. the verdict is based on the **claim text alone**, with zero external evidence.

| Knowledge source | Dataset | Claims with non-empty retrieved evidence |
|---|---|---|
| Wikipedia API | tweets_en | 13 / 30 |
| Wikipedia API | tweets_es | 0 / 30 |
| PubMed API | tweets_en | 8 / 30 |
| PubMed API | tweets_es | 1 / 30 |

Two distinct, dataset-specific problems are visible here:

**1. English Wikipedia retrieval now *works*, but the queries are too noisy to find the right article.** After fixing the User-Agent, `wikipedia.search()` returns results for 13/30 English claims — but inspecting `tweets_en_wiki_api_titles.txt` shows most of these are off-topic. For example, EN-02 ("RECENT: Study finds just one minute of anger weakens the immune system for 5 hours") returns *A Treatise of Human Nature*, *List of The Expanse episodes*, *Iran hostage crisis*, *2026 Iran war*, *Nikita Khrushchev*, and similar — none related to anger, immunology, or the underlying study. The pipeline sends the **entire tweet** as a literal `wikipedia.search()` query; MediaWiki's search tokenizes this into ~15-30 words and returns whichever articles share the most individual word matches, which for informal multi-clause tweet text is rarely the article actually being referenced.

**2. Spanish (MT-translated) claims return zero Wikipedia results, even with the fix.** Across all 30 `tweets_es` claims, `wikipedia.search()` returned no fetchable results — 0/30 non-empty, unchanged by the User-Agent fix. The most likely cause is **claim length**: the Spanish dataset's `MT Translation (EN)` column is on average more than twice as long as the English tweets (mean 392.9 vs. 188.0 characters; *every* Spanish claim is ≥199 characters — already at the English median). Within the English dataset alone, claims under 200 characters returned non-empty Wikipedia evidence 69% of the time (11/16), vs. only 14% (2/14) for claims ≥200 characters — and the **entire** Spanish dataset falls in that high-failure length range (median 283.5, max 971 characters). Machine translation tends to preserve or expand the original tweet's hashtags, emoji-derived text, and multi-sentence structure, compounding the problem for Spanish specifically.

**3. PubMed retrieval has its own, largely independent near-total failure**, consistent with `reproduction/api-pipeline`: `Entrez.esearch()` runs without error but returns `Count: 0` for almost all literal-tweet-text queries — only 8/30 English and 1/30 Spanish claims returned any abstracts.

Net effect: **48/60 (80%) of all claim × knowledge-source combinations had zero retrieved evidence**, and DeBERTa's verdict was based on the bare claim text.

### Comparison against a trivial "always predict TRUE" baseline

Given gold-label distributions of 16 TRUE / 3 FALSE (English, 19 scoreable) and 13 TRUE / 9 FALSE (Spanish, 22 scoreable), a trivial classifier that always predicts SUPPORTED scores:

| Dataset | Precision | Recall | F1 | Accuracy |
|---|---|---|---|---|
| tweets_en (trivial "always TRUE") | 84.2 | 100.0 | 91.4 | 84.2 |
| tweets_es (trivial "always TRUE") | 59.1 | 100.0 | 74.3 | 59.1 |

- **`tweets_en` / Wikipedia API (F1 81.5, accuracy 73.7) is *worse* than the trivial baseline on both metrics.** The 13/30 claims that now retrieve real-but-often-irrelevant Wikipedia evidence push several previously-"default SUPPORTED" TRUE claims toward REFUTED — 5 false negatives in total (EN-06, EN-08, EN-09, EN-19, EN-30; see §7 Cases 1-3).
- **`tweets_en` / PubMed API (F1 93.3, accuracy 89.5) is the only extension result that beats the trivial baseline**, by +1.9 F1 / +5.3 accuracy — but with only 8/30 claims retrieving any PubMed evidence, most of this margin still reflects the same "mostly-TRUE dataset + default-to-SUPPORTED" effect as the trivial baseline, not genuine evidence-based verification.
- **`tweets_es` (both sources, identical 69.0 F1 / 59.1 accuracy)**: accuracy exactly matches the trivial baseline (13/22 correct either way), but F1 is *lower* (69.0 vs. 74.3) — the no-evidence model correctly flags 3/9 FALSE claims as REFUTED (which the trivial baseline never does), but at the cost of incorrectly flagging 3/13 TRUE claims as REFUTED, netting out to a worse F1 under this class imbalance.

### Discussion

- **The "PubMed API beats CoVERT/SciFact on `tweets_en`" headline is not evidence the pipeline works well on tweets** — it is close to a restatement of the English dataset's 16:3 TRUE:FALSE imbalance, achieved with real PubMed evidence for only 8/30 claims.
- **Fixing the Wikipedia User-Agent bug *worsened* the English Wikipedia score** relative to the trivial baseline (81.5 F1 / 73.7 accuracy vs. 91.4 / 84.2): the newly-retrieved evidence is frequently topically irrelevant (§7 Case 2) or selects the wrong sentences from a relevant article (§7 Case 3), and DeBERTa treats irrelevant or non-supporting evidence as grounds for REFUTED even when the claim is TRUE. Fixing the *infrastructure* bug (User-Agent) was necessary but not sufficient — it surfaced a deeper *query-formulation* bug.
- **English vs. Spanish is confounded by claim length, not (only) language or translation quality.** The Spanish dataset's MT-translated claims are systematically ~2x longer than the English tweets, which is the most parsimonious explanation for why Wikipedia retrieval returns 0/30 for Spanish — the same length effect already degrades English retrieval for claims over ~200 characters.
- **Across both languages, 80% of claim × source pairs (48/60) produced zero retrieved evidence**, meaning the "3-step pipeline" effectively collapses to "Step 3 only: DeBERTa NLI on the bare claim text" for the large majority of this dataset.
- **For a misinformation-detection use case, the most actionable failure is the FALSE-claims pattern in §7 Case 5:** with no evidence, the model defaults toward SUPPORTED for most claims — the worst possible default for flagging health misinformation.

---

## 7. Failure Analysis

`pipeline/results/comparison/failure_candidates.csv` lists every TRUE/FALSE post the pipeline scored incorrectly, plus every UNVERIFIABLE post (for which the pipeline always forces a SUPPORTED/REFUTED verdict). The 8 cases below were selected to cover each taxonomy category and both knowledge sources/languages.

### Taxonomy categories

| # | Category | Description |
|---|---|---|
| 1 | Irrelevant retrieval | Retrieved Wikipedia/PubMed article doesn't match the claim's topic |
| 2 | Evidence-selection mismatch | Right article retrieved, but SPICED picked unrelated sentences |
| 3 | Informal-language NLI failure | Evidence is relevant, but DeBERTa fails on informal/sarcastic/ambiguous tweet phrasing |
| 4 | Forced verdict on UNVERIFIABLE | Pipeline confidently outputs SUPPORTED/REFUTED for a claim annotators couldn't verify |
| 5 | MT-translation-induced retrieval failure (Spanish only) | The MT translation process produces claim text too long/verbose for the retrieval APIs to return any results |
| 6 | Numeric/statistical precision | Claim hinges on a specific number/statistic the evidence doesn't address |
| 7 | Empty-evidence default bias | With zero retrieved evidence, the verdict is driven by claim phrasing alone, with an inconsistent — and often FALSE-claim-friendly — bias toward SUPPORTED |

### Documented cases

**Case 1 — EN-09 — Category 3 (Informal-language NLI failure)**
- Claim: *"Revolution's pancreatic cancer drug doubles survival, boosts quality of life"*
- Gold: TRUE (SUPPORTED) — Predicted: REFUTED
- Evidence (Wikipedia API, excerpt): *"31 May – a phase 3 trial published in the New England Journal of Medicine reports that daraxonrasib, an investigational oral RAS(ON) inhibitor, nearly doubles median overall survival in patients with previously treated metastatic pancreatic cancer, from 6.6 months with standard chemotherapy to 13.2 [months]..."*
- Hypothesis: this is the most striking failure in the dataset — retrieval and evidence selection both worked (the evidence is topically correct *and* numerically confirms "doubles survival": 13.2 / 6.6 ≈ 2.0×), yet DeBERTa predicted REFUTED. The claim refers to the drug informally as *"Revolution's [drug]"* (a company/brand reference), while the evidence names it by its generic compound name *"daraxonrasib"* and class *"RAS(ON) inhibitor"*. DeBERTa appears unable to bridge this entity-naming gap, treating the passage as *about a different drug* and therefore non-supporting. This rules out Category 6 (the number itself isn't the problem) and points to claim-evidence entity linking as the failure point.

**Case 2 — EN-19 — Category 1 (Irrelevant retrieval)**
- Claim: *"A new study in Geophysical Research Letters provides robust confirmation that northern hemisphere atmospheric circulation on the largest scale is responding to greenhouse gas emissions."*
- Gold: TRUE (SUPPORTED) — Predicted: REFUTED
- Evidence (Wikipedia API, excerpt): *"14 May — astronomers publish supporting evidence of water plume activity on Europa, moon of the planet Jupiter... On 20 June, NASA reported that the dust storm had grown to c[over...]"*
- Hypothesis: a textbook irrelevant-retrieval failure. The claim's vocabulary ("study", "robust confirmation", "largest scale", "responding to") is generic science-news phrasing that, fed verbatim into `wikipedia.search()`, surfaces an unrelated "year in science" timeline article (planetary science / Jupiter), not anything about atmospheric circulation or climate. DeBERTa correctly judges this Europa/Jupiter passage as unrelated to — and therefore not supporting — the claim, producing a confident but topically baseless REFUTED.

**Case 3 — EN-30 — Category 2 (Evidence-selection mismatch)**
- Claim: *"Bumblebees know how to problem solve their way to a reward."*
- Gold: TRUE (SUPPORTED) — Predicted: REFUTED
- Evidence (Wikipedia API, excerpt): *"Bumblebees gather nectar to add to the stores in the nest, and pollen to feed their young. === Communication and social learning === Bumblebees do not have ears, and it is not known whether or how well they can hear. Bumblebees have also been observed to engage in social learning..."*
- Hypothesis: unlike Cases 1-2, retrieval got the **right article** ("Bumblebee" — topically correct). But SPICED's top-k sentence selection picked passages about foraging behaviour and social learning/communication, not the article's content on cognition or problem-solving (the Wikipedia "Bumblebee" article does cover cognition elsewhere). DeBERTa then sees evidence that is bumblebee-related but doesn't address "problem solving for a reward", and predicts REFUTED. This is a pure Step-2 (SPICED) failure with correct Step-1 retrieval.

**Case 4 — EN-02 — Category 1 (Irrelevant retrieval, "right for the wrong reason")**
- Claim: *"RECENT: Study finds just one minute of anger weakens the immune system for 5 hours"*
- Gold: FALSE (REFUTED) — Predicted: REFUTED ✓ (counted as correct)
- Evidence (Wikipedia API, excerpt): *"...passions arising from 'the mixture of love and hatred with other emotions'. Unsurprisingly, the violence of a passion makes it stronger... Hume focuses on the factors which increase the violence of passions..."* (retrieved from David Hume's *A Treatise of Human Nature*)
- Hypothesis: this prediction is scored "correct", but for the wrong reason. The retrieved evidence is about 18th-century philosophy of emotion, not immunology — `wikipedia.search()` matched on shared words like "anger"/"passions" and "minute"/"violence" rather than topic. With completely irrelevant evidence, DeBERTa appears to default toward REFUTED (the inverse of the no-evidence default-to-SUPPORTED bias in Case 5), which happens to align with the FALSE gold label here. This case is a reminder that the headline accuracy/F1 numbers contain "accidentally correct" predictions that don't reflect genuine fact-checking.

**Case 5 — ES-08, ES-09, ES-14, ES-17, ES-19 (grouped) — Category 7 (Empty-evidence default bias)**
- Claims: 5 Spanish health-misinformation tweets — claiming COVID vaccines deplete white blood cells in "calculated waves" (ES-08), chlorine dioxide cures stage-4 prostate cancer (ES-09), skin warts are caused by insulin/cortisol dysregulation (ES-14), glyphosate causes the rise in celiac disease (ES-17), and aspartame permanently damages the brain/CNS (ES-19).
- Gold: all 5 are FALSE (REFUTED) — Predicted: all 5 SUPPORTED
- Evidence: empty for all 5, both knowledge sources (Wikipedia API: 0/30 for `tweets_es`; PubMed API: empty for these 5 specifically).
- Hypothesis: this is the single most policy-relevant failure in the dataset. With zero retrieved evidence, DeBERTa's verdict is based purely on the claim text — and confidently-worded, declarative misinformation ("X causes Y", "this doctor explains exactly...") reads to the model as plausible/supportable, regardless of its actual truth value. **5/5 of the FALSE-labelled Spanish misinformation claims were predicted SUPPORTED.** A claim-verification pipeline that cannot distinguish confidently-stated misinformation from confidently-stated facts — when it has no evidence — is, for this exact use case, actively counterproductive.

**Case 6 — ES-18, ES-20, ES-24 (grouped) — Category 7 (Empty-evidence default bias, inverse direction)**
- Claims: ES-18 (zero screen time recommended for under-6s; "biggest brain hack in history"), ES-20 (ADHD involves prefrontal-cortex dopamine deficiency, linked to screen affinity), ES-24 (turmeric as effective as ibuprofen for knee osteoarthritis).
- Gold: all 3 are TRUE (SUPPORTED) — Predicted: all 3 REFUTED
- Evidence: empty for all 3 (both sources).
- Hypothesis: shows the Case-5 "default to SUPPORTED" bias is **not** a uniform rule — for these 3 TRUE claims (also evidence-free), the model defaulted to REFUTED instead. All three are framed as precise causal/mechanistic or comparative-efficacy statements ("X is as effective as Y", "the alteration is Z, that's why..."), which may read to DeBERTa as over-specific/strong claims more typical of the REFUTED class in its NLI training data. Net effect: with no evidence, the no-evidence verdict is essentially a coin-flip driven by surface phrasing, not claim veracity — 10/13 evidence-free TRUE claims and only 3/9 evidence-free FALSE claims were predicted SUPPORTED.

**Case 7 — Wikipedia API, `tweets_es`, all 30 claims — Category 5 (MT-translation-induced retrieval failure)**
- Claims: the entire Spanish dataset (`MT Translation (EN)` column).
- Gold: mixed — Predicted: 0/30 claims have any retrieved Wikipedia evidence.
- Hypothesis: see "Retrieval coverage" in §6. The Spanish dataset's MT translations average 392.9 characters (vs. 188.0 for English), and every single one is ≥199 characters — already at the English median, and well past the ~200-character point where English retrieval success drops from 69% to 14%. This is a **retrieval-formulation** failure specific to how MT-translated claims are fed into `wikipedia.search()`, not a translation-*accuracy* failure (the `looks_untranslated()` check confirms all 30 are genuinely translated to English).

**Case 8 — EN-13 — Category 1 + 4 (Irrelevant retrieval + Forced verdict on UNVERIFIABLE)**
- Claim: *"'Alien' DNA found inside humans — it was inserted into our genes, mind-blowing study published in Oct, 2025 says."*
- Gold: UNVERIFIABLE — Predicted: REFUTED
- Evidence (Wikipedia API, excerpt): *"The purported interstellar meteorite, technically known as CNEOS 2014-01-08, impacted Earth in 2014... 27 April — a lineage of H3N8 bird flu is found to infect humans for the f[irst time...]"* (from a "Year in science" timeline article)
- Hypothesis: `wikipedia.search()` matched on "alien"/"inserted"/"genes"/"study" and surfaced a science-timeline article containing an unrelated item about an interstellar meteorite and an unrelated item about bird flu crossing into humans — both superficially "foreign thing entering/infecting" stories, but neither about horizontal gene transfer or "alien DNA". DeBERTa confidently predicts REFUTED against this irrelevant evidence. Because the claim is UNVERIFIABLE, this "REFUTED" is neither right nor wrong against gold — but it illustrates that the pipeline produces a *confident, evidence-cited-looking* verdict for a claim the human annotators explicitly could not adjudicate.

### Summary observations

- **Category 7 (empty-evidence default bias) is the most consequential failure mode**, affecting 48/60 (80%) of claim × source combinations — for these, the entire 3-step pipeline reduces to "DeBERTa NLI on the bare claim text".
- **The bias is asymmetric and dangerous for misinformation detection**: across both languages, 19 UNVERIFIABLE posts received a forced verdict, with 14/19 (74%) predicted SUPPORTED — and, more importantly, **5/5 FALSE-labelled Spanish misinformation claims with no evidence were predicted SUPPORTED** (Case 5). A tool meant to flag health misinformation is, in its current form, biased toward validating it when retrieval fails — which is most of the time.
- **Fixing the User-Agent bug was necessary but revealed a second-order bug**: 13/30 English claims now retrieve *real* Wikipedia evidence, but it is frequently irrelevant (Cases 2, 4, 8) or mis-selected by SPICED (Case 3) — and this newly-introduced "noise" evidence *reduced* the English Wikipedia F1/accuracy below the trivial baseline (Case 1, and §6).
- **Spanish retrieval (Wikipedia: 0/30) is best explained by claim length, not translation quality** — MT translations are ~2x longer than the original-language English tweets (Case 7).
- **Recommended next steps, in priority order:**
  1. **Add a "NOT ENOUGH EVIDENCE" output class** for the 80% of cases with zero retrieved evidence, instead of forcing SUPPORTED/REFUTED — directly addresses Categories 4 and 7, the largest and most dangerous failure modes.
  2. **Replace literal-tweet-as-query with key-phrase/entity extraction** before calling `wikipedia.search()` / `Entrez.esearch()` — would likely fix the irrelevant-retrieval cases (2, 4, 8) and may also shorten Spanish queries enough to return Wikipedia results (Case 7).
  3. **For Spanish specifically**, consider truncating/summarizing the `MT Translation (EN)` text before retrieval, or querying `es.wikipedia.org` directly with the original Spanish text and translating only the retrieved evidence.
  4. **Improve SPICED evidence selection** (e.g., re-rank by entity overlap with the claim, not just sentence-level cosine similarity) to address cases like EN-30 (Case 3) where the right article is retrieved but the wrong sentences are selected.
