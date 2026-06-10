# Replication & Extension in Two Languages

**NLP Group Project — Option 3 · Group 5 · June 2026**

## Overview

This project replicates and extends the paper:

> **Vladika & Matthes (EACL 2024)** — *Comparing Knowledge Sources for Open-Domain Scientific Claim Verification*

We chose this paper because it has clean benchmark results on SCIFACT, public code that runs, and is already cited by follow-up work — making it a reliable replication target.

## What We Do

### Replication

We reproduce the main results across two complementary approaches:

**Approach 1 — Step-3-only (`reproduction/step3-only/`):**
The paper's authors provided pre-computed retrieval results (BM25 and Semantic Search over Wikipedia and PubMed). We load these directly and run only Step 3 (DeBERTa-v3-large NLI verdict prediction) with corrected label normalisation. This isolates the NLI component and validates our reproduction against the paper's reported F1 scores.

**Approach 2 — Full API pipeline (`reproduction/api-pipeline/`):**
We re-run the complete three-step pipeline (retrieval → evidence selection → verdict prediction) from scratch, replacing the paper's local 6.6M-article Wikipedia dump and 20.6M-abstract MEDLINE dump with the **Wikipedia Python API** and **NCBI PubMed E-utilities API**. This produces independently retrieved evidence and tests the pipeline end-to-end.

Both approaches are compared against the paper's published F1 scores (Tables 3 & 4 of Vladika & Matthes 2024).

### Extension
We build a new test set of ~60 social media posts and run the same pipeline on them.

- **30 English** posts from Twitter/X and Reddit (r/science); **30 Spanish** posts from Twitter/X only
- Labels: **True / False / Unverifiable**, with two annotators and Cohen's κ reported
- Spanish posts are machine-translated before retrieval — a controlled comparison
- Qualitative failure taxonomy alongside quantitative results

**Core question:** How much does performance drop when the pipeline — built for formal scientific claims — faces informal, noisy social media text? Does it differ between English and Spanish?

## English Dataset — Annotation Summary

30 posts collected from Twitter/X (22) and Reddit r/science (8), covering a range of science and health topics.

**Topic areas:** Astronomy, Animal Behavior, Climate, Fertility, Genetics, Health, Marine Biology, Mental Health, Neurology, Neuroscience, Oncology, Pharmaceuticals, Pregnancy, Psychiatry, Quantum Physics, Vaccines, Vaping, Wildlife

**Gold label distribution (30 posts):**

| Label | Count |
|-------|-------|
| TRUE | 16 |
| FALSE | 3 |
| UNVERIFIABLE | 11 |

**Inter-annotator agreement:**

| Metric | Value |
|--------|-------|
| Agreements | 24 / 30 (80%) |
| Disagreements | 6 |
| Cohen's κ | 0.5395 |
| κ interpretation | Moderate (0.4–0.6) |

All 6 disagreements used the `EVIDENCE` code (annotators agreed a scientific claim was present but weighted evidence differently). Pipeline predictions are pending.

---

## Spanish Dataset — Annotation Summary

30 tweets collected from Twitter/X, covering a range of health and science topics.

**Topic areas:** Vaccines, Cancer, Cardiology, Dermatology, Food Science, Infectious Disease, Mental Health, Medicine, Neuroscience, Nutrition, Sleep, Hormonal Health, Climate/Environment

**Gold label distribution (30 tweets):**

| Label | Count |
|-------|-------|
| TRUE | 13 |
| FALSE | 9 |
| UNVERIFIABLE | 8 |

**Inter-annotator agreement:**

| Metric | Value |
|--------|-------|
| Agreements | 28 / 30 (93.3%) |
| Disagreements | 2 |
| Cohen's κ | 0.8983 |
| κ interpretation | Excellent (> 0.8) |

Disagreements were resolved by discussion and assigned a final Gold Label. Disagreement codes used: `EVIDENCE` (conflicting or differently-weighted evidence).

Spanish posts were machine-translated to English (see `MT Translation (EN)` column in `data/spanish/Spanish_Tweets_Annotated.xlsx`) before being passed to the pipeline.

Pipeline predictions are pending — the `Pipeline Prediction` column will be filled after Andrea runs the extension pipeline.

## Why It Matters

Science misinformation spreads mainly through social media, yet existing benchmarks use curated formal claims. We measure that gap. The Spanish angle has not been tested with this pipeline — genuinely new ground.

## Deliverables

- Technical report: replication results, extension results, mini-survey, and critical assessment
- 60 annotated social media posts + annotation guidelines
- Code repository: replication and extension scripts
- Slides for 20-minute in-class presentation + individual reflections
- One-page executive summary for a non-technical reader

---

## Repository Structure

```
NLP-Group-Project/
│
├── data/                          # All datasets — shared across notebooks
│   ├── scifact_no-nei_dataset.csv
│   ├── pubmedqa.json
│   ├── healthFC_annotated.csv
│   ├── CoVERT_FC_annotations.jsonl
│   ├── Annotation_Guidelines_Bilingual.md
│   └── spanish/
│       └── Spanish_Tweets_Annotated.xlsx
│
├── reference/                     # Original paper code and evidence — read-only, not modified
│   ├── code/
│   │   ├── open_domain_claim_verification.ipynb        # paper's full pipeline
│   │   └── open_domain_claim_verification_small_sample.ipynb
│   └── evidence/                  # Pre-computed retrieval results provided by the paper's authors
│       ├── Google/
│       ├── PubMed/
│       │   ├── BM25 Search/
│       │   └── Semantic Search/
│       └── Wikipedia/
│           ├── BM25 Search/
│           └── Semantic Search/
│
├── reproduction/                  # Our reproduction experiments
│   │
│   ├── step3-only/                # Replication approach 1:
│   │   │                          # Loads the pre-computed evidence from reference/evidence/,
│   │   │                          # applies corrected label normalisation, and runs only
│   │   │                          # Step 3 (DeBERTa NLI). Fast to run — no retrieval needed.
│   │   ├── open_domain_claim_verification_step3_only.ipynb
│   │   └── results/               # metrics.csv + Excel comparison tables per source
│   │
│   └── api-pipeline/              # Replication approach 2:
│       │                          # Full end-to-end pipeline (Steps 1–3) using public APIs
│       │                          # instead of the paper's local dumps:
│       │                          #   Step 1 — Wikipedia API / NCBI PubMed E-utilities
│       │                          #   Step 2 — SPICED sentence similarity (unchanged)
│       │                          #   Step 3 — DeBERTa NLI (unchanged)
│       ├── open_domain_claim_verification_api.ipynb
│       ├── comparison/            # Cross-notebook comparison outputs
│       └── results/
│           ├── Wikipedia API/     # metrics.csv + retrieved evidence lines
│           └── PubMed API/        # metrics.csv + retrieved evidence lines
│
├── pipeline/                      # Tweet verification pipeline (to be built — Andrea)
│                                  # Will run the best-performing configuration from
│                                  # reproduction/ on the 60 English/Spanish social media posts
│
└── documentation/
    ├── 2024.eacl-long.128.pdf     # Reference paper
    ├── step3_notebook_notes.md    # Data issues, label normalisation, fixes log
    ├── technical_report.md
    ├── Group_Assignment.pdf
    └── NLP Proposal - Group 5.html
```

### How the pieces fit together

| Folder | Purpose | Status |
|--------|---------|--------|
| `reference/` | Paper's original code and pre-computed evidence. Not modified. | Done |
| `reproduction/step3-only/` | Replication approach 1: NLI only on paper's evidence, corrected labels. | Done |
| `reproduction/api-pipeline/` | Replication approach 2: full pipeline re-run via public APIs. | Done |
| `pipeline/` | Extension: run on 60 tweet dataset. | To do |

---

## API Keys

The API pipeline notebook (`reproduction/api-pipeline/open_domain_claim_verification_api.ipynb`) uses two external APIs. Credentials are loaded from a `.env` file that **must never be committed** (it is in `.gitignore`).

### Setup

1. Copy the example file and fill in your values:
   ```
   cp .env.example .env
   ```
   On **Google Colab**: upload `.env` to the Colab file browser (`/content/.env`) at the start of each session — the notebook picks it up automatically.

2. Fill in the values below.

---

### Google Custom Search API (required for Google section)

Free tier: **100 queries/day**. Running all 4 datasets requires ~2,174 queries.

| Step | Instructions |
|------|-------------|
| 1. Create a project | [console.developers.google.com](https://console.developers.google.com) → New Project |
| 2. Enable the API | Search for **Custom Search API** → Enable |
| 3. Get an API key | Credentials → Create Credentials → API Key → copy into `GOOGLE_API_KEY` |
| 4. Create a search engine | [programmablesearchengine.google.com](https://programmablesearchengine.google.com) → Add → set "Search the entire web" → copy the **Search engine ID** into `GOOGLE_CSE_ID` |

---

### NCBI E-utilities (required for PubMed API section)

Free, no billing required.

| Field | Instructions |
|-------|-------------|
| `NCBI_EMAIL` | Any valid email address — required by NCBI's terms of use |
| `NCBI_API_KEY` | Optional. Register at [ncbi.nlm.nih.gov/account](https://www.ncbi.nlm.nih.gov/account/) to get a free key. Raises the rate limit from 3 to 10 requests/second (~3× faster retrieval). |

---

## Team & Roles

| Member | Role |
|--------|------|
| **Lea** | Replication Lead |
| **Mateus** | Data & Annotation — English |
| **Claudia** | Data & Annotation — Spanish |
| **Andrea** | Extension Pipeline & Analysis |
| **Dalton** | Literature, Report & Presentation |

### Lea — Replication Lead
- Set up the shared repo, replicate the paper's pipeline, document deviations
- Run both reproduction notebooks and compare results against published figures
- Write the Replication section of the report (our numbers vs. published numbers)

### Mateus — Data & Annotation (English)
- Collect 30 English posts from Twitter/X and Reddit (r/science) containing scientific claims
- Label each post: True / False / Unverifiable
- Cross-annotate a shared sample with Claudia to report Cohen's κ
- Write and maintain the annotation guidelines document

### Claudia — Data & Annotation (Spanish)
- Collect 30 Spanish posts from Twitter/X
- Label each post using the same guidelines as Mateus
- Apply machine translation (DeepL or Google Translate) to Spanish posts before running the pipeline — document the approach
- Cross-annotate a shared sample with Mateus for Cohen's κ

### Andrea — Extension Pipeline & Analysis
- Run the pipeline on the full 60-post dataset using `pipeline/`
- Report quantitative results (F1, accuracy) for English vs. Spanish vs. benchmark baseline
- Build the failure taxonomy: at least 5–10 documented failure cases with hypotheses for why they fail
- Write the Extension Results and Failure Analysis sections of the report

### Dalton — Literature, Report & Presentation
- Write the mini-survey: where the paper fits in the claim-verification landscape, what came before and after
- Write the Critical Assessment and Open Questions sections based on empirical findings
- Build the slides and lead the 20-minute in-class presentation; organise a rehearsal
- Write the one-page executive summary for a non-technical reader

---

## Tags

`EACL 2024` · `Fact-checking` · `Social media` · `English · Spanish` · `SCIFACT` · `Misinformation`
