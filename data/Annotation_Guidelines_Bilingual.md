# Annotation Guidelines — Bilingual Tweet Dataset (English & Spanish)

**NLP Group Project · Option 3 · Scientific Claim Verification**
**Group 5 · June 2026**

---

## 1. Purpose

These guidelines govern the annotation of 60 social media posts — 30 in Spanish and 30 in English — for a scientific claim verification extension study. The same pipeline tested on the SCIFACT benchmark (Vladika & Matthes, EACL 2024) will be applied to this dataset. Two annotators must label each tweet independently before comparing and resolving disagreements. Inter-annotator agreement will be reported as Cohen's κ, calculated separately for each language and across the full dataset.

**Division of labour:** One annotator handles the Spanish dataset; the other handles the English dataset. Both annotators apply the same label definitions, edge-case rules, and spreadsheet conventions described in this document.

---

## 2. What Counts as an Annotatable Post

A post is suitable for this dataset only if it meets **ALL** of the following:

- It makes a specific, concrete factual claim about science, health, medicine, climate, nutrition, or similar empirical topics.
- The claim is, in principle, checkable against scientific literature — even if the answer is "unverifiable".
- It is not pure opinion, satire, or a question.

**Additional requirement for Spanish posts:** The main claim must be in Spanish (mixed Spanish/English is acceptable if the core scientific claim is in Spanish).

**Additional requirement for English posts:** The main claim must be in English. Posts from Reddit or Twitter/X are both eligible.

**Discard a post if:**

- It contains no factual claim (e.g. purely emotional or political).
- The claim is about a person, event, or social matter rather than a scientific one.
- The post is clearly satirical or a joke.
- For Spanish: the claim is entirely in English → DISCARD, note `[WRONG-LANG]`.
- For English: the claim is entirely in another language → DISCARD, note `[WRONG-LANG]`.

---

## 3. Label Definitions

Assign exactly one of the three labels below to each post. These definitions apply equally to both the Spanish and English datasets.

| Label | Definition | Key test | When in doubt |
|-------|-----------|----------|---------------|
| **TRUE** | The claim is supported by current scientific consensus or can be confirmed by a peer-reviewed source. | Would a scientist in this field agree with the claim as stated? | Prefer TRUE only if evidence is clear and unambiguous. |
| **FALSE** | The claim contradicts scientific consensus or is demonstrably wrong according to peer-reviewed evidence. | Is there clear scientific evidence that directly contradicts this claim? | Prefer FALSE only if the contradiction is documented, not just implausible. |
| **UNVER.** | The claim cannot be verified or refuted because it is too vague, anecdotal, predictive, or lacks a scientific basis to check. | Is there a specific, testable scientific claim here at all? | **Default to UNVERIFIABLE when you are genuinely unsure.** |

---

## 4. Edge Cases and Decision Rules

These rules apply to **both languages**. Language-specific notes are added where relevant.

### 4.1 Negation and sarcasm

Read the post carefully for negation or sarcasm.

- **Spanish signals:** "no es cierto que…", "como si…", "claro que sí 🙄", ironic hashtags like `#Broma` or `#Ironía`.
- **English signals:** "Oh sure, X totally causes Y", "scientists don't want you to know", quote-retweeting to mock, "/s" (Reddit sarcasm tag).

If you suspect sarcasm but cannot be sure, mark it UNVERIFIABLE and note `[IRONY]` in the Notes column.

### 4.2 Partially true claims

If a post contains one true part and one false part, label based on the dominant claim. Note the partial truth in the Notes column. Do not split labels.

### 4.3 Outdated information

If a claim was once scientific consensus but is now superseded, label it FALSE and note "outdated". Use current consensus as the reference point regardless of when the post was published.

### 4.4 Anecdotal claims

Personal experience framed as general fact → UNVERIFIABLE.

- **Spanish example:** "A mí me curó…" generalised to everyone.
- **English example:** "I stopped eating gluten and my arthritis disappeared — it works for everyone."

### 4.5 Claims with imprecise numbers

If a numerical claim is roughly correct within the margin of scientific uncertainty, label TRUE. If substantially wrong (e.g. off by an order of magnitude or in the wrong direction), label FALSE.

### 4.6 Informal language, abbreviations, and misspellings

- **Spanish:** Twitter Spanish frequently omits accents and uses abbreviations (e.g. "xq" for "porque", "tb" for "también"). Interpret charitably based on most likely meaning. Note any ambiguity.
- **English:** Reddit and Twitter English use heavy abbreviations (e.g. "imo", "afaik", "lol", "fr"), all-caps for emphasis, and internet slang. Interpret based on most likely intended meaning. Note ambiguity in the Notes column.

### 4.7 Hedged or qualified claims (English-specific)

English posts on Reddit especially tend to hedge claims ("studies suggest", "some research indicates", "I read that…"). Apply the following:

- If the hedge is meaningful (e.g. "one small study found X" — and X is otherwise unverified), label UNVERIFIABLE.
- If the hedge is rhetorical but the underlying claim is a clear scientific assertion (e.g. "studies show vaccines cause autism, just saying"), evaluate the underlying claim.
- Note the hedge in the Notes column using `[HEDGED]`.

### 4.8 Conspiracy framing

Posts that frame a scientific claim within a conspiracy ("what the government doesn't want you to know", "big pharma suppresses this") do not make the underlying claim more or less true. Evaluate the factual claim on its scientific merits. Note `[CONSPIRACY-FRAME]` in Notes.

---

## 5. Post Content: What-If Cases

Posts often contain elements beyond plain text. The rules below specify exactly how to handle each element in the spreadsheet. **Rules apply to both Spanish and English posts unless otherwise noted.**

### 5.1 URLs and links to external articles

- **Tweet Text column:** paste the full post text as it appears, including the raw URL.
- **URL column:** if the post itself contains a link, copy that link here.
- **Do not visit the linked article** to help decide the label. Your annotation must be based on the post text alone. If the text is insufficient to determine a claim without reading the linked page, label it UNVERIFIABLE and add `[LINK-DEPENDENT]` in the Notes column.
- If the link is broken or the page is unavailable, note `[DEAD-LINK]` in Notes and still annotate based on the text only.

> **Spanish example:** *"Nuevo estudio confirma que el azúcar causa TDAH en niños https://t.co/xxx"* → Paste full text including URL; copy URL to column C; label the claim in the text (likely FALSE). Do NOT click the link.

> **English example:** *"New Harvard study proves intermittent fasting reverses diabetes https://t.co/xxx"* → Same procedure. Label based on the claim in the tweet text only.

### 5.2 Emojis

- **Tweet Text column:** keep all emojis exactly as they appear.
- Emojis do **not** count as part of the factual claim. Ignore them when labelling.
- **Exception:** if an emoji clearly ironises the text (e.g. 🙄, 🤣 after a dubious claim), treat the post as potentially sarcastic and apply section 4.1.

> **Spanish example:** *"El café cura el cáncer ☕😍 definitivamente demostrado"* → Keep emojis in text column. Label FALSE. Emojis do not change the label.

> **English example:** *"Turns out the moon landing was faked after all 🤣 wake up sheeple"* → The 🤣 signals sarcasm/mockery. Apply section 4.1; label UNVERIFIABLE and note `[IRONY]`.

### 5.3 Hashtags

- **Tweet Text column:** include all hashtags exactly as they appear.
- Hashtags do **not** form part of the verifiable scientific claim. Disregard them when deciding the label.
- **Exception:** if a hashtag signals irony or joke status (Spanish: `#Ironía`, `#Broma`, `#Satira`; English: `#sarcasm`, `#jokes`, `#NotADoctor`), treat as a sarcasm signal per section 4.1.
- Hashtag-only posts (no propositional content) → DISCARD.

### 5.4 Mentions (@usernames)

- **Tweet Text column:** keep @mentions as they appear.
- Do **not** look up the mentioned account. The identity of the person mentioned is irrelevant.
- If the post is directed at someone but contains no claim (e.g. "@doctor check this out!"), DISCARD as non-annotatable.

### 5.5 Retweets (RT) and quote tweets

- Paste the full visible text into the Tweet Text column, including the "RT @username:" prefix if present.
- The scientific claim is whatever is stated in the visible text. Do not follow links to find the original tweet.
- If the retweet or quote frame introduces irony or disagreement (e.g. the user is quoting to mock), annotate the framing, not the quoted claim. Note `[QUOTE-FRAME]` in Notes.

### 5.6 Reddit-specific formatting (English)

Reddit posts may include markdown that does not render in plain text:

- **Bold/italic markers** (`**text**`, `*text*`): treat the marked text as emphasis only; it does not change the factual claim.
- **Reddit "edit" notes** (e.g. "EDIT: someone pointed out I was wrong"): if the edit directly changes or retracts the scientific claim, annotate the post as UNVERIFIABLE and note `[EDITED-RETRACTION]`. If the edit is irrelevant, annotate the original claim.
- **Thread context:** if a post is a reply to another and the claim only makes sense in context, try to infer the full meaning from the visible text. If the meaning is truly unclear, label UNVERIFIABLE and note `[REPLY-CONTEXT]`.
- **"TL;DR" sections:** if the post has a TL;DR that summarises a scientific claim differently from the body, annotate based on the **body** claim and note any discrepancy.

### 5.7 Images, GIFs, and videos attached to posts

You will only have access to post text, not attached media.

- If the text alone contains a clear scientific claim, annotate normally.
- If the text says something like "see image", "watch this", "pic related", or similar and the claim depends on unseen media, label UNVERIFIABLE and note `[MEDIA-DEPENDENT]` in Notes.
- Do **NOT** attempt to retrieve or view attached media.

### 5.8 Threads and truncated posts

- Annotate only the text available to you. Do not follow thread links.
- If the text is truncated mid-claim and the claim cannot be understood from what is visible, label UNVERIFIABLE and note `[TRUNCATED]` in Notes.
- If the truncated text still contains a self-contained, verifiable claim, annotate it normally.

### 5.9 Mixed-language posts

- **Spanish posts:** acceptable if the main claim is in Spanish. If the claim is entirely in English, DISCARD and note `[WRONG-LANG]`. If the claim is ambiguous due to code-switching, annotate based on the most likely reading and note `[CODE-SWITCH]`.
- **English posts:** acceptable if the main claim is in English. If the claim is in another language, DISCARD and note `[WRONG-LANG]`. If a post mixes English with another language but the core scientific claim is clearly in English, annotate normally and note `[CODE-SWITCH]` if ambiguous.

### 5.10 Numbers, statistics, and percentages

- If a statistic is presented as a scientific fact, apply the rules in section 4.5.
- If no source is given for a statistic, label UNVERIFIABLE unless the figure matches well-known consensus data.
- Copy numbers exactly as they appear in the post text. Do not round or convert.

### 5.11 Dates and timestamps

- Record the post's publication date in the Notes column if known, using the tag `[DATE: YYYY-MM-DD]`.
- Outdated claims should be labelled per section 4.3.
- Do not modify the Tweet Text column to reflect the current date.

### 5.12 All-caps and heavy emphasis

Posts (especially English ones) often use all-caps or excessive punctuation for emphasis (e.g. "PROVEN!!!", "100% FACT", "DOCTORS HATE HIM"). This does not make a claim more or less true. Annotate the underlying factual claim on its scientific merits. Note `[EMPHASIS]` in Notes only if the stylistic framing is relevant to sarcasm detection.

---

### Summary: What to log in the spreadsheet

| Content type | Tweet Text column | Notes column flag | Affects label? |
|---|---|---|---|
| URL / link | Keep in text | `[LINK-DEPENDENT]` or `[DEAD-LINK]` | No — annotate text only |
| Emojis | Keep exactly | None (unless ironic) | Only if sarcasm cue |
| Hashtags | Keep exactly | `[SATIRE]` if hashtag signals joke | Only if signals satire |
| @mentions | Keep exactly | None | No |
| RT / quote tweet | Keep full text incl. RT prefix | `[QUOTE-FRAME]` if ironic framing | Frame may affect label |
| Reddit markdown | Keep as-is | `[EDITED-RETRACTION]` or `[REPLY-CONTEXT]` | Only if edit retracts claim |
| Attached media | Text only | `[MEDIA-DEPENDENT]` | No — use UNVER. if needed |
| Truncated post | Paste what is available | `[TRUNCATED]` | UNVER. if claim unclear |
| Mixed language | Full text as-is | `[CODE-SWITCH]` if ambiguous | No if main claim in target lang |
| Statistics / numbers | Copy exactly | None unless imprecision noted | Yes — see §4.5 |
| Conspiracy framing | Keep in text | `[CONSPIRACY-FRAME]` | No — evaluate claim on merits |
| Hedged claim (EN) | Keep in text | `[HEDGED]` | Yes — see §4.7 |
| Post date | Do not modify | `[DATE: YYYY-MM-DD]` | Possibly — see §4.3 |
| All-caps / heavy emphasis | Keep exactly | `[EMPHASIS]` only if relevant | No |

---

## 6. Annotation Process

- Each annotator works independently on their assigned language dataset, without discussing labels with the other annotator until both have finished.
- The annotator searches briefly for supporting or contradicting scientific evidence (1–2 minutes max per post).
- The annotator records their label, a confidence score (1–3), and a brief rationale in the spreadsheet.
- After both annotators finish their respective datasets, compare labels for any posts where cross-checking is needed. Agreements → directly enter as Gold Label. Disagreements → discuss until consensus; if none, escalate to a third group member.
- Calculate Cohen's κ across all 30 posts per language and record both scores in the report.

---

## 7. Confidence Score

| Score | Meaning |
|-------|---------|
| **1** | Low confidence — I could see this going either way; required a lot of interpretation. |
| **2** | Medium confidence — I found relevant evidence but it is not fully conclusive. |
| **3** | High confidence — The label is clear and well-supported. |

---

## 8. Worked Examples

### Spanish — TRUE example

| | |
|---|---|
| **Post** | *"Las vacunas contra el sarampión son efectivas en más del 97% de los casos, según la OMS."* |
| **Label: TRUE** | WHO data confirms ~97% efficacy for the measles vaccine. The claim is accurate and cites a real source. |

### Spanish — FALSE example

| | |
|---|---|
| **Post** | *"Los teléfonos móviles causan cáncer cerebral, está científicamente demostrado."* |
| **Label: FALSE** | Major reviews (WHO, IARC) find no confirmed causal link between mobile phone use and brain cancer. The claim overstates the evidence. |

### Spanish — UNVERIFIABLE example

| | |
|---|---|
| **Post** | *"A mí el magnesio me curó completamente la ansiedad, algo que los médicos no quieren que sepas."* |
| **Label: UNVER.** | Personal anecdote generalised to universal claim. Conspiracy framing. No testable scientific claim. Note `[CONSPIRACY-FRAME]`. |

### Spanish — URL example

| | |
|---|---|
| **Post** | *"Demostrado: el azúcar causa hiperactividad en niños https://t.co/abc123"* |
| **Label: FALSE** | No robust evidence links sugar to hyperactivity. Copy URL to URL column. Do NOT visit the link to decide label. |

### Spanish — Emoji/sarcasm example

| | |
|---|---|
| **Post** | *"Claro que las vacunas causan autismo 🙄 si lo dice internet debe ser verdad"* |
| **Label: UNVER.** | The 🙄 emoji and "si lo dice internet" signal sarcasm. Cannot confirm ironic intent with certainty. Note `[IRONY]`. |

---

### English — TRUE example

| | |
|---|---|
| **Post** | *"Regular aerobic exercise reduces the risk of cardiovascular disease — this has been established by decades of research."* |
| **Label: TRUE** | Well-established finding supported by extensive peer-reviewed literature (e.g. meta-analyses in JAMA, Lancet). Confidence: 3. |

### English — FALSE example

| | |
|---|---|
| **Post** | *"Scientists have confirmed: humans only use 10% of their brains. Imagine if we could unlock the rest!"* |
| **Label: FALSE** | The "10% of the brain" claim is a well-documented myth; neuroimaging studies show virtually all brain regions have documented functions. |

### English — UNVERIFIABLE example

| | |
|---|---|
| **Post** | *"I cut out seed oils completely and my chronic inflammation disappeared in two weeks — doctors don't want people knowing this."* |
| **Label: UNVER.** | Personal anecdote generalised as universal fact. Conspiracy framing (`[CONSPIRACY-FRAME]`). The relationship between seed oils and inflammation is an active research area with no clear consensus to definitively confirm or refute this specific claim. Confidence: 2. |

### English — Hedged claim example

| | |
|---|---|
| **Post** | *"Some research suggests that eating breakfast boosts your metabolism and helps with weight loss, afaik."* |
| **Label: UNVER.** | The hedge "some research suggests" combined with "afaik" signals genuine uncertainty. The claim is contested in the scientific literature (some RCTs show no metabolic effect). Note `[HEDGED]`. Confidence: 2. |

### English — Reddit /s sarcasm example

| | |
|---|---|
| **Post** | *"Oh sure, 5G towers definitely caused COVID-19 /s"* |
| **Label: UNVER.** | The "/s" tag is Reddit's explicit sarcasm marker. The poster does not believe this claim; they are mocking it. Cannot annotate the literal text as a genuine scientific claim. Note `[IRONY]`. Confidence: 3. |

### English — Conspiracy framing example

| | |
|---|---|
| **Post** | *"Big Pharma has been suppressing the cure for cancer for decades. Natural compounds like apricot kernels (B17) have been proven to destroy cancer cells."* |
| **Label: FALSE** | The claim about apricot kernels (amygdalin/laetrile) as a cancer cure is contradicted by clinical evidence; it is also potentially toxic. Conspiracy framing does not change the evaluation. Note `[CONSPIRACY-FRAME]`. Confidence: 3. |

---

## 9. Machine Translation Note (Spanish Dataset)

Per the project design, Spanish posts will be machine-translated into English (using DeepL or Google Translate) before being passed to the RAG verification pipeline. Annotators label the **original Spanish post**. The translation is only for pipeline input — do not let the translation influence your label.

If you notice that the translation significantly changes the meaning of a claim, flag it in the Notes column using the tag `[TRANS-ISSUE]`.

> **Note on emojis in MT:** Machine translation tools typically drop or ignore emojis. This is expected behaviour. The MT Translation (EN) column is for pipeline use only; the original tweet text with emojis must remain in the Tweet Text (Spanish) column.

The English posts do **not** require machine translation and are passed directly to the pipeline.

---

## 10. Disagreement Taxonomy

When resolving disagreements, classify the source of disagreement using one of the following codes and record it in the Disagreement column of the spreadsheet. Codes apply to both language datasets.

| Code | Meaning |
|------|---------|
| `SCOPE` | Disagreement about whether the post contains a scientific claim at all. |
| `EVIDENCE` | Both agreed it is a claim, but found conflicting or differently-weighted evidence. |
| `IRONY` | One annotator read the post as sarcastic, the other did not. |
| `PRECISION` | Disagreement over whether a numerically imprecise claim is TRUE or FALSE. |
| `VAGUE` | Disagreement over whether the claim is specific enough to verify. |
| `HEDGE` | Disagreement over whether hedging language (English) renders a claim UNVERIFIABLE. |
| `OTHER` | Any other source — briefly describe in Notes. |

---

*NLP Group Project · Option 3 · Annotation Guidelines · Group 5 · June 2026*
