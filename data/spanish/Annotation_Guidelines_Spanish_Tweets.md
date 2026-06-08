# Annotation Guidelines — Spanish Tweet Dataset

**NLP Group Project · Option 3 · Scientific Claim Verification**
**Group 5 · June 2026**

---

## 1. Purpose

These guidelines govern the annotation of 30 Spanish-language tweets for a scientific claim verification extension study. The same pipeline tested on the SCIFACT benchmark (Vladika & Matthes, EACL 2024) will be applied to this dataset. Two annotators must label each tweet independently before comparing and resolving disagreements. Inter-annotator agreement will be reported as Cohen's κ.

---

## 2. What Counts as an Annotatable Tweet

A tweet is suitable for this dataset only if it meets **ALL** of the following:

- It makes a specific, concrete factual claim about science, health, medicine, climate, nutrition, or similar empirical topics.
- The claim is in Spanish (mixed Spanish/English is acceptable if the main claim is in Spanish).
- The claim is, in principle, checkable against scientific literature — even if the answer is "unverifiable".
- It is not pure opinion, satire, or a question.

**Discard a tweet if:**

- It contains no factual claim (e.g. purely emotional or political).
- The claim is about a person, event, or social matter rather than a scientific one.
- The tweet is clearly satirical or a joke.

---

## 3. Label Definitions

Assign exactly one of the three labels below to each tweet.

| Label | Definition | Key test | When in doubt |
|-------|-----------|----------|---------------|
| **TRUE** | The claim is supported by current scientific consensus or can be confirmed by a peer-reviewed source. | Would a scientist in this field agree with the claim as stated? | Prefer TRUE only if evidence is clear and unambiguous. |
| **FALSE** | The claim contradicts scientific consensus or is demonstrably wrong according to peer-reviewed evidence. | Is there clear scientific evidence that directly contradicts this claim? | Prefer FALSE only if the contradiction is documented, not just implausible. |
| **UNVER.** | The claim cannot be verified or refuted because it is too vague, anecdotal, predictive, or lacks a scientific basis to check. | Is there a specific, testable scientific claim here at all? | **Default to UNVERIFIABLE when you are genuinely unsure.** |

---

## 4. Edge Cases and Decision Rules

### 4.1 Negation and sarcasm

Read the tweet carefully for negation ("no es cierto que…") or sarcasm. If you suspect sarcasm but cannot be sure, mark it UNVERIFIABLE and note it in the Notes column.

### 4.2 Partially true claims

If a tweet contains one true part and one false part, label based on the dominant claim. Note the partial truth in the Notes column. Do not split labels.

### 4.3 Outdated information

If a claim was once scientific consensus but is now superseded, label it FALSE and note "outdated". Use current consensus as the reference point.

### 4.4 Anecdotal claims

Personal experience framed as general fact (e.g. "A mí me curó…" generalised to everyone) → UNVERIFIABLE.

### 4.5 Claims with imprecise numbers

If a numerical claim is roughly correct within the margin of scientific uncertainty, label TRUE. If substantially wrong, label FALSE.

### 4.6 Misspellings and abbreviations

Twitter Spanish frequently omits accents and uses abbreviations. Interpret charitably based on most likely meaning. Note any ambiguity.

---

## 5. Tweet Content: What-If Cases

Tweets often contain elements beyond plain text. The rules below specify exactly how to handle each element in the spreadsheet.

### 5.1 URLs and links to external articles

URLs in tweets (e.g. a link to a news article or study) should be treated as follows:

- **Tweet Text (Spanish) column:** paste the full tweet text as it appears, including the raw URL.
- **URL column:** if the tweet itself contains a link, copy that link here instead of (or in addition to) the tweet's source URL.
- **Do not visit the linked article** to help decide the label. Your annotation must be based on the tweet text alone. If the tweet text is insufficient to determine a claim without reading the linked page, label it UNVERIFIABLE and add `[LINK-DEPENDENT]` in the Notes column.
- If the link is broken or the page is unavailable, note `[DEAD-LINK]` in Notes and still annotate based on the tweet text only.

> **Example:** Tweet: *"Nuevo estudio confirma que el azúcar causa TDAH en niños https://t.co/xxx"* → Paste full tweet text including URL in column B, copy URL to column C, annotate the claim in the tweet text (likely FALSE based on the claim as stated). Do NOT click the link.

### 5.2 Emojis

Emojis are common in Spanish tweets and should be handled as follows:

- **Tweet Text (Spanish) column:** keep all emojis exactly as they appear in the original tweet. Copy-paste the full text including emoji characters.
- Emojis do **not** count as part of the factual claim. Ignore them when labelling.
- **Exception:** if an emoji clearly contradicts or ironises the text (e.g. 🤣 after a dubious claim, or 🙄), treat the tweet as potentially sarcastic and apply the rules in section 4.1.
- **MT Translation (EN) column:** DeepL/Google Translate will usually omit or transliterate emojis. This is fine — the translation column is pipeline input only. You are labelling the original Spanish text.

> **Example:** Tweet: *"El café cura el cáncer ☕😍 definitivamente demostrado"* → Keep emojis in column B. Label FALSE based on the scientific claim. Emojis do not change the label.

### 5.3 Hashtags

Hashtags are metadata and should be treated as follows:

- **Tweet Text (Spanish) column:** include all hashtags exactly as they appear.
- Hashtags do **not** form part of the verifiable scientific claim. Disregard them when deciding the label.
- **Exception:** if a hashtag is integral to understanding the meaning (e.g. `#Ironía`, `#Broma`, `#Satira`), treat it as a sarcasm signal per section 4.1 and mark accordingly.
- Hashtag-only tweets (no propositional content) → DISCARD as they contain no factual claim.

### 5.4 Mentions (@usernames)

Twitter @mentions appear frequently but carry no scientific content:

- **Tweet Text (Spanish) column:** keep @mentions as they appear.
- Do **not** look up the mentioned account. The identity of the person mentioned is irrelevant to the claim label.
- If the tweet is directed at someone but contains no claim (e.g. "@doctor mira esto!"), DISCARD as non-annotatable.

### 5.5 Retweets (RT) and quote tweets

When a tweet is a retweet or quote-tweet of another:

- Paste the full visible text into Tweet Text (Spanish), including the "RT @username:" prefix if present.
- The scientific claim is whatever is stated in the tweet text visible to you. Do not follow links to find the original tweet.
- If the retweet prefix or quote frame introduces irony or disagreement (e.g. the user is quoting to mock or refute), annotate the framing, not the quoted claim. Note `[QUOTE-FRAME]` in Notes.

### 5.6 Images, GIFs, and videos attached to tweets

You will only have access to tweet text, not attached media. Handle as follows:

- If the tweet text alone contains a clear scientific claim, annotate normally.
- If the tweet text says something like "Leer imagen", "Ver hilo", or similar and the claim depends on unseen media, label UNVERIFIABLE and note `[MEDIA-DEPENDENT]` in Notes.
- Do **NOT** attempt to retrieve or view the attached media.

### 5.7 Threads and truncated tweets

Some tweets are part of a longer thread or cut off with "…":

- Annotate only the text available to you. Do not follow thread links.
- If the text is truncated mid-claim and the claim cannot be understood from what is visible, label UNVERIFIABLE and note `[TRUNCATED]` in Notes.
- If the truncated tweet still contains a self-contained, verifiable claim, annotate it normally.

### 5.8 Mixed-language tweets

Tweets may blend Spanish with English, Portuguese, or other languages:

- Acceptable if the main scientific claim is in Spanish. Annotate normally.
- If the claim is entirely in English (tweet just happens to be in a Spanish-language dataset), label DISCARD and note `[WRONG-LANG]`.
- If the claim is ambiguous due to code-switching, annotate based on the most likely reading and note `[CODE-SWITCH]` in Notes.

### 5.9 Numbers, statistics, and percentages in tweets

Tweets often cite figures that may or may not be accurate:

- If a statistic is presented as a scientific fact, apply the rules in section 4.5.
- If no source is given for a statistic, label UNVERIFIABLE unless the figure matches well-known consensus data.
- Copy numbers exactly as they appear in the tweet text. Do not round or convert in the spreadsheet.

### 5.10 Dates and timestamps

Tweets may reference dates (publication date, study date, etc.):

- Record the tweet's own publication date in the Notes column if it is known, using the tag `[DATE: YYYY-MM-DD]`.
- Outdated claims should be labelled per section 4.3.
- Do not modify the Tweet Text column to reflect the current date.

---

### Summary: What to log in the spreadsheet

| Content type | Tweet Text column (B) | Notes column flag | Affects label? |
|---|---|---|---|
| URL / link | Keep in text | `[LINK-DEPENDENT]` or `[DEAD-LINK]` | No — annotate text only |
| Emojis | Keep exactly | None (unless ironic) | Only if sarcasm cue |
| Hashtags | Keep exactly | `[SATIRE]` if hashtag signals joke | Only if signals satire |
| @mentions | Keep exactly | None | No |
| RT / quote tweet | Keep full text incl. RT prefix | `[QUOTE-FRAME]` if ironic framing | Frame may affect label |
| Attached media | Text only | `[MEDIA-DEPENDENT]` | No — use UNVER. if needed |
| Truncated tweet | Paste what is available | `[TRUNCATED]` | UNVER. if claim unclear |
| Mixed language | Full text as-is | `[CODE-SWITCH]` if ambiguous | No if main claim in ES |
| Statistics / numbers | Copy exactly | None unless imprecision noted | Yes — see §4.5 |
| Tweet date | Do not modify | `[DATE: YYYY-MM-DD]` | Possibly — see §4.3 |

---

## 6. Annotation Process

- Each annotator reads the tweet independently, without discussing with the other annotator.
- The annotator searches briefly for supporting or contradicting scientific evidence (1–2 minutes max per tweet).
- The annotator records their label, a confidence score (1–3), and a brief rationale in the spreadsheet.
- After both annotators finish, compare labels. Agreements → directly enter as Gold Label. Disagreements → discuss until consensus; if none, escalate to a third group member.
- Calculate Cohen's κ across all 30 tweets and record the score in the report.

---

## 7. Confidence Score

| Score | Meaning |
|-------|---------|
| **1** | Low confidence — I could see this going either way; required a lot of interpretation. |
| **2** | Medium confidence — I found relevant evidence but it is not fully conclusive. |
| **3** | High confidence — The label is clear and well-supported. |

---

## 8. Worked Examples

### TRUE example

| | |
|---|---|
| **Tweet** | *"Las vacunas contra el sarampión son efectivas en más del 97% de los casos, según la OMS."* |
| **Label: TRUE** | WHO data confirms ~97% efficacy for the measles vaccine. The claim is accurate and cites a real source. |

### FALSE example

| | |
|---|---|
| **Tweet** | *"Los teléfonos móviles causan cáncer cerebral, está científicamente demostrado."* |
| **Label: FALSE** | Major reviews (WHO, IARC) find no confirmed causal link between mobile phone use and brain cancer. The claim overstates the evidence. |

### UNVERIFIABLE example

| | |
|---|---|
| **Tweet** | *"A mí el magnesio me curó completamente la ansiedad, algo que los médicos no quieren que sepas."* |
| **Label: UNVER.** | Personal anecdote generalised to universal claim. No testable scientific claim; also relies on a conspiracy framing. Cannot be verified or falsified. |

### URL example *(new)*

| | |
|---|---|
| **Tweet** | *"Demostrado: el azúcar causa hiperactividad en niños https://t.co/abc123"* |
| **Label: FALSE** | The claim in the tweet text is false (no robust evidence sugar causes hyperactivity). Copy URL to column C. Do NOT visit the link to decide label. |

### Emoji + sarcasm example *(new)*

| | |
|---|---|
| **Tweet** | *"Claro que las vacunas causan autismo 🙄 si lo dice internet debe ser verdad"* |
| **Label: UNVER.** | The 🙄 emoji and phrase "si lo dice internet" signal sarcasm. Cannot confirm ironic intent with certainty. Label UNVERIFIABLE, note [IRONY] in Disagree Code if applicable. Keep emoji in column B. |

---

## 9. Machine Translation Note

Per the project design, Spanish tweets will be machine-translated into English (using DeepL or Google Translate) before being passed to the RAG verification pipeline. Annotators label the original Spanish tweet. The translation is only for pipeline input — do not let the translation influence your label.

If you notice that the translation significantly changes the meaning of a claim, flag it in the Notes column using the tag `[TRANS-ISSUE]`.

> **Note on emojis in MT:** Machine translation tools typically drop or ignore emojis. This is expected behaviour. The MT Translation (EN) column is for pipeline use only; the original tweet text with emojis must remain in column B.

---

## 10. Disagreement Taxonomy

When resolving disagreements, classify the source of disagreement using one of the following codes and record it in the Disagreement column of the spreadsheet:

| Code | Meaning |
|------|---------|
| `SCOPE` | Disagreement about whether the tweet contains a scientific claim at all. |
| `EVIDENCE` | Both agreed it is a claim, but found conflicting or differently-weighted evidence. |
| `IRONY` | One annotator read the tweet as sarcastic, the other did not. |
| `PRECISION` | Disagreement over whether a numerically imprecise claim is TRUE or FALSE. |
| `VAGUE` | Disagreement over whether the claim is specific enough to verify. |
| `OTHER` | Any other source — briefly describe in Notes. |

---

*NLP Group Project · Option 3 · Annotation Guidelines · Group 5 · June 2026*
