# Replication & Extension in Two Languages

**NLP Group Project — Option 3 · Group 5 · June 2026**

## Overview

This project replicates and extends the paper:

> **Vladika & Matthes (EACL 2024)** — *Comparing Knowledge Sources for Open-Domain Scientific Claim Verification*

We chose this paper because it has clean benchmark results on SCIFACT, public code that runs, and is already cited by follow-up work — making it a reliable replication target.

## What We Do

### Replication
We reproduce the main result: retrieval-augmented claim verification on the SCIFACT test set using the best-performing knowledge source from the paper. We report F1, document any deviations, and compare our numbers against the published figures.

### Extension
We build a new test set of ~60 social media posts and run the same pipeline on them.

- **30 English + 30 Spanish** posts collected from Twitter/X
- Labels: **True / False / Unverifiable**, with two annotators and Cohen's κ reported
- Spanish posts are machine-translated before retrieval — a controlled comparison
- Qualitative failure taxonomy alongside quantitative results

**Core question:** How much does performance drop when the pipeline — built for formal scientific claims — faces informal, noisy social media text? Does it differ between English and Spanish?

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

Spanish posts were machine-translated to English (see `MT Translation (EN)` column in `Spanish_Tweets_Annotated.xlsx`) before being passed to the pipeline.

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

## Team & Roles

| Member | Role |
|--------|------|
| **Lea** | Replication Lead |
| **Mateus** | Data & Annotation — English |
| **Claudia** | Data & Annotation — Spanish |
| **Andrea** | Extension Pipeline & Analysis |
| **Dalton** | Literature, Report & Presentation |

### Lea — Replication Lead
- Set up the shared repo, clone the paper's code, check GitHub issues
- Run the pipeline on the SCIFACT test set and reproduce the F1 score from the paper
- Document every deviation, assumption, or environment fix needed to get it running
- Write the Replication section of the report (our numbers vs. published numbers)

### Mateus — Data & Annotation (English)
- Collect 30 English posts from Twitter/X containing scientific claims
- Label each post: True / False / Unverifiable
- Cross-annotate a shared sample with Claudia to report Cohen's κ
- Write and maintain the annotation guidelines document

### Claudia — Data & Annotation (Spanish)
- Collect 30 Spanish posts from Twitter/X
- Label each post using the same guidelines as Mateus
- Apply machine translation (DeepL or Google Translate) to Spanish posts before running the pipeline — document the approach
- Cross-annotate a shared sample with Mateus for Cohen's κ

### Andrea — Extension Pipeline & Analysis
- Run the pipeline on the full 60-post dataset
- Report quantitative results (F1, accuracy) for English vs. Spanish vs. SCIFACT baseline
- Build the failure taxonomy: at least 5–10 documented failure cases with hypotheses for why they fail
- Write the Extension Results and Failure Analysis sections of the report

### Dalton — Literature, Report & Presentation
- Write the mini-survey: where the paper fits in the claim-verification landscape, what came before and after
- Write the Critical Assessment and Open Questions sections based on empirical findings
- Build the slides and lead the 20-minute in-class presentation; organise a rehearsal
- Write the one-page executive summary for a non-technical reader

---

## Repository Structure

```
NLP-Group-Project/
├── README.md
├── Group_Assignment.pdf
├── NLP Proposal - Group 5.html
│
├── replication/              # Lea — pipeline code, SCIFACT run, deviations log
│
├── data/
│   ├── english/              # Mateus — 30 English posts, annotation guidelines, spreadsheet
│   └── spanish/              # Claudia — 30 Spanish posts, annotation guidelines, spreadsheet
│       └── Annotation_Guidelines_Spanish_Tweets.md
│
├── pipeline/                 # Andrea — extension pipeline scripts, results, failure taxonomy
│
└── report/                   # Dalton — report drafts, slides, mini-survey, executive summary
```

---

## Tags

`EACL 2024` · `Fact-checking` · `Social media` · `English · Spanish` · `SCIFACT` · `Misinformation`
