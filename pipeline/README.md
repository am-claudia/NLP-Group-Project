# Tweet Verification Pipeline

This folder contains the pipeline for verifying claims extracted from social media posts (English and Spanish tweets).

## How it works

The pipeline is reproduced directly from the API pipeline notebook at [`reproduction/api-pipeline/open_domain_claim_verification_api.ipynb`](../reproduction/api-pipeline/open_domain_claim_verification_api.ipynb). That notebook runs the full three-step pipeline (retrieval → evidence selection → verdict prediction) against public APIs. No local data dumps are needed.

## Which knowledge source to use

Based on the reproduction results in [`reproduction/overview/overview.md`](../reproduction/overview/overview.md), two sources are available:

| Source | Strength | Recommended for |
|--------|----------|----------------|
| **Wikipedia API** | Most robust across all datasets; best on CoVERT (tweet-format data) | General-purpose tweet claims |
| **PubMed API** | Strong on health/medical topics | Health-specific claims |

You can run one or both sources. Wikipedia API is the default recommendation. If the tweet dataset is health-focused, running PubMed API as well gives a useful comparison.

## Setting up on Google Colab

1. Upload the repository to your Google Drive at `My Drive/NLP-Group-Project/`
2. Open the api-pipeline notebook in Colab
3. Run the Drive mount cell at the top — it will connect automatically
4. Set up your API credentials (see below)
5. Run all cells

The notebook auto-detects Colab and uses Drive paths. Resume points are built in — if the session disconnects mid-run, re-run the resume cell to reload saved results rather than re-running retrieval from scratch.

## Adapting the claims source

The api-pipeline notebook loads claims from the four benchmark datasets (SciFact, PubMedQA, HealthFC, CoVERT). To run on tweets instead:

1. Load your tweet claims into a Python list, e.g.:
   ```python
   tweet_claims = ["Claim text 1", "Claim text 2", ...]
   tweet_labels = [1, 0, ...]  # 1=True, 0=False (for evaluation)
   ```
2. Replace the dataset loading cells (Step 0) with your list
3. Update `all_datasets` to include your tweet dataset:
   ```python
   all_datasets = {
       'tweets_en': (tweet_claims, tweet_labels),
   }
   ```
4. Update `DATASETS_TO_RUN` accordingly:
   ```python
   DATASETS_TO_RUN = ['tweets_en']
   ```

Steps 1–3 (retrieval, evidence selection, verdict prediction) run unchanged.

## API keys

Two API credentials are required. Copy `.env.example` from the project root to `.env` and fill in your values:

```
cp .env.example .env
```

On Colab, upload `.env` to your Drive folder (`My Drive/NLP-Group-Project/.env`) — the notebook loads it automatically from there.

| Key | Required for | How to get |
|-----|-------------|------------|
| `NCBI_EMAIL` | PubMed retrieval | Any valid email — required by NCBI terms of use |
| `NCBI_API_KEY` | PubMed retrieval (optional) | Free key at ncbi.nlm.nih.gov/account — raises rate limit from 3 to 10 req/sec |

See `.env.example` at the project root for the full list of variables and setup instructions.
