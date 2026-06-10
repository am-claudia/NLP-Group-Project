# Step 3 Notebook — Data Notes

Notes on how the `open_domain_claim_verification_step3_only.ipynb` notebook uses pre-computed BM25 and Google results, normalises labels, and the data issues that were fixed.

---

## How the BM25 Result Files Are Used

Each BM25 `.txt` file (`bm25wiki_lines_*.txt`, `bm25pubmed_lines_*.txt`) stores one entry per claim in the format:

```
claim text [SEP] retrieved evidence text
```

Evidence may span multiple physical lines in the file. The `read_bm25_file` function reconstructs multi-line entries by grouping consecutive lines: a new entry begins whenever a line contains `[SEP]`, and subsequent lines without `[SEP]` are appended to the current entry.

The notebook skips Steps 1–2 (retrieval) entirely and feeds these pre-computed `claim [SEP] evidence` strings directly into the DeBERTa-v3-large NLI model (Step 3) to predict SUPPORTED / REFUTED verdicts.

---

## Label Normalisation

All datasets are normalised to **1 = SUPPORTED, 0 = REFUTED**. NEI (Not Enough Information) claims are excluded before evaluation.

| Dataset | Original labels | Transformation |
|---|---|---|
| SciFact | Already binary | No change |
| PubMedQA | `yes`, `no`, `maybe` | `yes`→1, `no`→0; `maybe` (NEI) excluded via `.isin(['yes','no'])` |
| HealthFC | `0=SUPPORTED, 1=NEI, 2=REFUTED` | `0`→1, `2`→0; `label == 1` (NEI) excluded |
| CoVERT | `SUPPORTS`, `REFUTES`, `NOT ENOUGH INFO` | `SUPPORTS`→1, `REFUTES`→0; `NOT ENOUGH INFO` excluded |

Post-filtering sizes: SciFact 693, PubMedQA 890, HealthFC 327, CoVERT 264.

---

## Functions Added to the Notebook

`run_and_save(joint_lists, label_source, save_subdir, label)` — added to `open_domain_claim_verification_step3_only.ipynb` (not present in the original `open_domain_claim_verification.ipynb`). Loops over all four datasets, calls `predict_verdicts`, computes precision/recall/F1 against gold labels, prints a results table, and writes a `metrics.csv` to `Results-Step3/<save_subdir>/`.

`align_entries_to_claims(entries, claims)` — also new, added to handle the two data issues described below.

---

## Two Data Issues Fixed

### Issue 1 — BM25 files indexed the full dataset (including NEI)

The Wikipedia and PubMed BM25 files for SciFact and HealthFC were generated from their **full** datasets before NEI filtering:

| File | BM25 entries | Claims after NEI filter | Difference |
|---|---|---|---|
| `bm25wiki_lines_scifact.txt` | 1109 | 693 | 416 NEI entries present |
| `bm25pubmed_lines_scifact.txt` | 1109 | 693 | 416 NEI entries present |
| `bm25wiki_lines_healthfc.txt` | 740 | 327 | 413 NEI entries present |
| `bm25pubmed_lines_healthfc.txt` | 740 | 327 | 413 NEI entries present |

**Fix:** Added `align_entries_to_claims(entries, claims)` in the helper cell. When the count mismatches, it extracts the claim text from each BM25 entry (the part before ` [SEP] `), builds a lookup dict, and returns only the entries whose claim text appears in the non-NEI claims list, in the correct order.

### Issue 2 — 7 HealthFC claims were never retrieved

After filtering to non-NEI claims, 7 HealthFC claims had no entry in the BM25 files at all — they were absent from the original retrieval run (no near-matches found either, so it is not a whitespace/encoding difference):

1. Can oral vein products containing herbal active ingredients such as rutosides, hidrosmin, diosmin, extracts of horse chestnut...
2. Do milk or dairy products promote bladder cancer?
3. Do milk or dairy products promote prostate cancer?
4. Do green tea products provide effective and safe support for weight loss?
5. Does brisk walking lower mortality? Is some exercise better than none at all?
6. Can arthroscopy reduce pain or improve mobility?
7. Do health benefits increase with duration and intensity of exercise?

**Fix:** `align_entries_to_claims` uses `.get(c, f'{c} [SEP] ')` as a fallback for claims with no BM25 entry, producing an empty-evidence entry (`claim [SEP] `). The NLI model still predicts a verdict from the claim text alone for these 7 claims, and a warning is printed listing them. This keeps the evaluation set size intact rather than silently dropping claims.

---

## Google Result File Issues

The same `align_entries_to_claims` fix was applied to the Google cell. Three of the four Google files had count mismatches:

| File | Google blocks | Claims after NEI filter | Root cause |
|---|---|---|---|
| `scifact_google_lines.txt` | 693 | 693 | OK — no issue |
| `pubmedqa_google_lines.txt` | 1000 | 890 | Generated from full PubMedQA (including `maybe`/NEI); claim text matches exactly after filtering |
| `covert_google_lines.txt` | 300 | 264 | Generated from full CoVERT (including NEI); claim text matches after fixing whitespace (see below) |
| `healthfc_google_lines.txt` | 742 | 327 | Dataset version mismatch — see Issue 4 |

### Issue 3 — CoVERT claim whitespace after @username removal

The CoVERT `claim` column was cleaned with `.str.replace('@username', '')`, but many tweets start with `@username`, leaving a leading space (e.g. `"@username Many vaccines..."` → `" Many vaccines..."`). The Google file stores the stripped version, so the claims didn't match.

**Fix:** Added `.str.strip()` to the CoVERT claim cleaning line in `cell-load-datasets`:
```python
covert_df['claim'] = covert_df.claim.str.replace('@username', '', regex=False).str.replace('\n', ' ', regex=False).str.strip()
```
This resolved all 157 apparent mismatches — CoVERT now has 0 missing claims.

### Issue 4 — HealthFC Google file uses an older version of the claims

The HealthFC `en_claim` column was updated (claims were reworded/extended) after the Google results were collected. 289 of the 327 non-NEI claims in the current dataset do not match any entry in `healthfc_google_lines.txt`. For example:

- **Google file has:** `"Does a nasal spray with carrageenan from red algae prevent colds"`
- **Current dataset has:** `"Does a nasal spray with carrageenan from red algae relieve discomfort from a cold ('chill')?"`

There is no way to recover the original evidence for these claims without re-running the Google retrieval. The `align_entries_to_claims` fallback applies empty evidence (`claim [SEP] `) for the 289 unmatched claims and prints a warning. The Google evaluation for HealthFC will therefore be unreliable and should not be compared to the paper's reported figures.
