# Tweet Verification Pipeline

This folder contains the extension pipeline: running the same three-step open-domain claim verification pipeline (retrieval → SPICED evidence selection → DeBERTa-v3-large NLI verdict) on the project's new 60-post bilingual tweet dataset (30 English, 30 Spanish), and comparing the results against the SciFact and CoVERT baselines from `reproduction/api-pipeline/`.

## Notebook

[`open_domain_claim_verification_tweets.ipynb`](open_domain_claim_verification_tweets.ipynb) — adapted from [`reproduction/api-pipeline/open_domain_claim_verification_api.ipynb`](../reproduction/api-pipeline/open_domain_claim_verification_api.ipynb). All retrieval, evidence selection, and NLI code is reused unchanged; only the dataset-loading, metrics, and comparison/failure-analysis cells are new.

Run order:

1. **Setup** — pip installs, Drive mount (Colab), imports, paths, model loading.
2. **Step 0** — loads the 60 tweets from `data/english/English_Tweets_Annotated.xlsx` and `data/spanish/Spanish_Tweets_Annotated.xlsx`. English uses the `Tweet Text (English)` column; Spanish uses the `MT Translation (EN)` column. Gold labels are normalised to `1` (TRUE), `0` (FALSE), `None` (UNVERIFIABLE).
3. **Steps 1–3, Wikipedia API** — document retrieval, SPICED evidence selection, DeBERTa NLI verdicts for `tweets_en` and `tweets_es`.
4. **Steps 1–3, PubMed API** — same, using NCBI E-utilities.
5. **Comparison** — builds `pipeline/results/comparison/extension_vs_baseline.csv`, comparing `tweets_en` / `tweets_es` against the `scifact` and `covert` rows from `reproduction/api-pipeline/results/`.
6. **Failure taxonomy** — builds `pipeline/results/comparison/failure_candidates.csv`: every TRUE/FALSE post the pipeline got wrong, plus every UNVERIFIABLE post (which the pipeline always forces a SUPPORTED/REFUTED verdict on). Use this to write up 5–10 cases in `documentation/technical_report.md`.

Resume points are included after the Wikipedia and PubMed retrieval+SPICED steps, same pattern as the api-pipeline notebook — if a Colab session disconnects, re-run from the resume cell to reload saved intermediate results instead of re-querying APIs.

## Evaluation

Precision/recall/F1/accuracy are computed on the **TRUE/FALSE subset only** (19/30 EN, 22/30 ES) — UNVERIFIABLE posts have no SUPPORTED/REFUTED ground truth, consistent with how NEI claims are excluded for SciFact/PubMedQA/HealthFC/CoVERT in `reproduction/`. The pipeline still produces (and saves) a prediction for every post, including UNVERIFIABLE ones, for the failure taxonomy.

## Which knowledge source to use

Based on the reproduction results in [`reproduction/overview/overview.md`](../reproduction/overview/overview.md):

| Source | Strength | Recommended for |
|--------|----------|----------------|
| **Wikipedia API** | Most robust across all datasets; best on CoVERT (tweet-format data) | General-purpose tweet claims |
| **PubMed API** | Strong on health/medical topics | Health-specific claims |

The notebook runs **both** sources, since the dataset covers a mix of general and health-related scientific claims.

## Setting up on Google Colab

1. Upload the repository to your Google Drive at `My Drive/NLP-Group-Project/` (must include `data/`, `reproduction/api-pipeline/results/`, and `pipeline/`)
2. Open `open_domain_claim_verification_tweets.ipynb` in Colab
3. Run the Drive mount cell at the top — it will connect automatically
4. Set up your API credentials (see below)
5. Run all cells

## API keys

PubMed retrieval requires NCBI credentials. Copy `.env.example` from the project root to `.env` and fill in your values:

```
cp .env.example .env
```

On Colab, upload `.env` to your Drive folder (`My Drive/NLP-Group-Project/.env`) — the notebook loads it automatically from there.

| Key | Required for | How to get |
|-----|-------------|------------|
| `NCBI_EMAIL` | PubMed retrieval | Any valid email — required by NCBI terms of use |
| `NCBI_API_KEY` | PubMed retrieval (optional) | Free key at ncbi.nlm.nih.gov/account — raises rate limit from 3 to 10 req/sec |

See `.env.example` at the project root for the full list of variables and setup instructions.

## Outputs

```
pipeline/results/
├── Wikipedia API/
│   ├── tweets_en_wiki_api_titles.txt
│   ├── tweets_en_wiki_api_lines.txt
│   ├── tweets_es_wiki_api_titles.txt
│   ├── tweets_es_wiki_api_lines.txt
│   ├── tweets_en_predictions.csv
│   ├── tweets_es_predictions.csv
│   └── metrics.csv
├── PubMed API/
│   └── (same structure, *_pubmed_api_*)
└── comparison/
    ├── extension_vs_baseline.csv
    └── failure_candidates.csv
```
