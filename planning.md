# Provenance Guard — planning.md

> Written before any implementation code. Updated before stretch features.

---

## System Overview

Provenance Guard is a backend API that classifies submitted text as human-written or AI-generated, scores confidence in that classification, surfaces a transparency label to users, and handles creator appeals. It is designed for creative sharing platforms where false positives (labeling a human's work as AI-generated) are worse than false negatives.

**Path a submission takes through the system:**

1. Creator submits text via `POST /submit` with their `creator_id`
2. The system assigns a unique `content_id` and passes the text to two detection signals in parallel
3. Signal 1 (LLM-based) sends the text to Groq and receives a probability score (0–1) that the text is AI-generated
4. Signal 2 (stylometric heuristics) computes measurable statistical properties of the text and returns a score (0–1)
5. The confidence scorer combines both signal scores into a single weighted confidence score
6. The confidence score is mapped to one of three transparency label variants
7. The result — content_id, attribution, confidence score, label text — is written to the audit log
8. The structured response is returned to the caller

**Appeal path:**

1. Creator submits `POST /appeal` with their `content_id` and `creator_reasoning`
2. The system updates the content status from `classified` to `under_review`
3. The appeal reasoning is appended to the audit log entry for that content_id
4. A confirmation response is returned

---

## Detection Signals

### Signal 1: LLM-Based Classification (Groq)

**What it measures:** Semantic and stylistic coherence holistically. The LLM reads the text and assesses whether it exhibits patterns typical of AI-generated writing — uniformity of sentence structure, formulaic transitions, absence of genuine personal voice, overly balanced hedging language.

**Output format:** A float between 0.0 and 1.0 representing the probability the text is AI-generated. Extracted from the LLM's structured JSON response.

**Why it differs between human and AI writing:** AI models tend to produce text that is coherent but lacks the idiosyncratic markers of human voice — unexpected word choices, sentence fragments, self-correction, genuine uncertainty, culturally specific references. The LLM is well-positioned to detect these holistic patterns.

**Blind spot:** The LLM can be fooled by heavily edited AI output, non-native English speakers whose writing is formally structured, and academic or technical writing that is naturally uniform. It also has no access to external context about the creator.

---

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Statistical properties of the text that differ between human and AI writing:

- **Sentence length variance** — AI text tends to have more uniform sentence lengths. Human writing varies more.
- **Type-token ratio (TTR)** — vocabulary diversity. AI text often repeats similar vocabulary; human writing is more lexically diverse in short samples.
- **Punctuation density** — humans use punctuation idiosyncratically (dashes, ellipses, exclamation marks). AI output tends toward clean, standard punctuation.
- **Average sentence complexity** — measured by average words per sentence. AI output tends toward medium complexity; human writing spans a wider range.

**Output format:** A float between 0.0 and 1.0. Each sub-metric is normalized and averaged into a single stylometric score where higher = more AI-like.

**Why it differs between human and AI writing:** AI language models are trained to produce fluent, readable text — which makes them statistically more uniform than human writers. Humans make stylistic choices that are inconsistent by nature.

**Blind spot:** Formal human writing (academic papers, legal documents, professional blog posts) will score as more AI-like because it is intentionally uniform. Short texts (under ~50 words) produce unreliable stylometric scores due to small sample size.

---

## Confidence Scoring and Uncertainty

### Combining Signals

Both signals produce a score between 0.0 and 1.0 where higher = more likely AI-generated.

**Weighting:**
- LLM score: 60% weight
- Stylometric score: 40% weight

The LLM signal gets higher weight because it captures semantic holism that stylometrics cannot. Stylometrics is a useful corroborating signal but can be fooled by writing style.

```
confidence = (llm_score * 0.6) + (stylometric_score * 0.4)
```

### Threshold Design

The thresholds are **asymmetric** — the system requires higher confidence to label content as AI-generated than to label it as human-written. This reflects the asymmetry in harm: a false positive (labeling a human's work as AI) damages a creator's reputation; a false negative (missing AI content) is a less severe outcome.

| Confidence range | Attribution | Label variant |
|-----------------|-------------|---------------|
| ≥ 0.72 | likely_ai | High-confidence AI |
| 0.40 – 0.71 | uncertain | Uncertain |
| < 0.40 | likely_human | High-confidence human |

**What 0.6 means:** A confidence of 0.6 means the system leans toward AI-generated but is not confident enough to assert it. The content lands in the uncertain band and is flagged for human review rather than labeled definitively.

**Calibration approach:** Tested manually on clearly AI-generated text (ChatGPT outputs) and clearly human text (personal anecdotes, informal posts). Scores should cluster above 0.80 for obvious AI text and below 0.40 for obvious human text. Borderline cases (formal human writing, lightly edited AI) should produce 0.40–0.79.

---

## Transparency Label Variants

All three variants are written out exactly as they will appear in API responses and user-facing displays.

**High-confidence AI (confidence ≥ 0.72):**
```
⚠️ This content was likely generated by AI. Our system is [X]% confident in this
assessment. You may appeal if you believe this is incorrect.
```

**Uncertain (confidence 0.40 – 0.71):**
```
❓ We could not confidently determine the origin of this content ([X]% confidence).
A human reviewer will assess this content.
```

**High-confidence human (confidence < 0.40):**
```
✅ This content appears to be human-written. Our system is [X]% confident in
this assessment.
```

*Note: [X]% is replaced with `round((1 - confidence) * 100)` for human label and `round(confidence * 100)` for AI label to express confidence in the stated verdict rather than the raw AI-probability score.*

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content (identified by `creator_id`) can appeal any classification associated with their `content_id`.

**What they provide:**
- `content_id` (str): the ID returned at submission
- `creator_reasoning` (str): free-text explanation of why they believe the classification is wrong

**What the system does:**
1. Looks up the audit log entry for the `content_id`
2. Updates its `status` from `classified` to `under_review`
3. Appends `appeal_reasoning` and `appeal_timestamp` to the entry
4. Returns a confirmation JSON response

**What a human reviewer sees when opening the appeal queue:**
- Original submission text
- Original attribution and confidence score
- Both signal scores (llm_score, stylometric_score)
- Creator's reasoning
- Timestamp of original classification and appeal

**Automated re-classification:** Not implemented. Appeals require human review.

---

## Anticipated Edge Cases

**Edge case 1: Non-native English speakers with formal writing style.**
A non-native English speaker who writes carefully and formally may produce text with low sentence length variance and high stylometric uniformity — the same properties as AI output. This will produce a falsely high confidence score. The asymmetric threshold (requiring ≥ 0.80 to label as AI) reduces but does not eliminate this risk. The appeals workflow is the primary mitigation.

**Edge case 2: Short texts under 50 words.**
Stylometric heuristics are unreliable on short texts because variance metrics need sufficient sample size. A 30-word poem may produce a near-random stylometric score. For short texts, the system will rely more heavily on the LLM signal, but the overall confidence is less trustworthy. The system should flag this in the response.

**Edge case 3: Lightly edited AI output.**
A creator who generates text with an AI tool and then edits it heavily may produce text that the LLM scores as uncertain. The stylometric signal may also be ambiguous. These cases will correctly land in the uncertain band, but the system cannot distinguish between human writing that looks AI-like and edited AI output.

---

## Architecture

```
                        POST /submit
                        {text, creator_id}
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Flask App         │
                    │   (app.py)          │
                    └────────┬────────────┘
                             │ assign content_id
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
   ┌──────────────────┐         ┌──────────────────────┐
   │  Signal 1        │         │  Signal 2             │
   │  LLM Classifier  │         │  Stylometric          │
   │  (Groq API)      │         │  Heuristics           │
   │  → llm_score     │         │  → stylometric_score  │
   └────────┬─────────┘         └──────────┬────────────┘
            │                              │
            └──────────────┬───────────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │  Confidence Scorer    │
               │  0.6*llm + 0.4*style  │
               │  → confidence (0–1)   │
               └──────────┬────────────┘
                          │
                          ▼
               ┌───────────────────────┐
               │  Label Generator      │
               │  ≥0.80 → AI label     │
               │  0.40–0.79 → uncertain│
               │  <0.40 → human label  │
               └──────────┬────────────┘
                          │
                          ▼
               ┌───────────────────────┐
               │  Audit Log            │
               │  (SQLite / JSON)      │
               │  writes entry         │
               └──────────┬────────────┘
                          │
                          ▼
               JSON response to caller
               {content_id, attribution,
                confidence, label, signals}


        ── Appeal Flow ──────────────────────

                    POST /appeal
                    {content_id, creator_reasoning}
                           │
                           ▼
                ┌─────────────────────┐
                │  Look up entry      │
                │  in audit log       │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Update status →    │
                │  "under_review"     │
                │  Append reasoning   │
                └──────────┬──────────┘
                           │
                           ▼
                JSON confirmation response
```

---

## API Surface

| Method | Endpoint | Accepts | Returns |
|--------|----------|---------|---------|
| POST | `/submit` | `{text, creator_id}` | `{content_id, attribution, confidence, label, llm_score, stylometric_score, status}` |
| POST | `/appeal` | `{content_id, creator_reasoning}` | `{message, content_id, status}` |
| GET | `/log` | — | `{entries: [...]}` |

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1

**Input to AI:** Detection signals section (Signal 1 only) + Architecture diagram + API surface table.

**Request:** Generate (1) Flask app skeleton with `POST /submit` route stub that accepts `{text, creator_id}` and returns a hardcoded JSON response, and (2) a `classify_with_llm(text)` function that calls the Groq API and returns a float 0–1.

**Verification:** Call `classify_with_llm()` directly on 3 test inputs (clear AI text, clear human text, borderline) before wiring into the endpoint. Check that scores vary meaningfully. Check that the Flask route returns valid JSON before adding any logic.

---

### Milestone 4 — Signal 2 + Confidence scoring

**Input to AI:** Detection signals section (Signal 2) + Uncertainty representation section + Architecture diagram.

**Request:** Generate (1) a `compute_stylometric_score(text)` function that computes sentence length variance, type-token ratio, and punctuation density and returns a float 0–1, and (2) a `compute_confidence(llm_score, stylometric_score)` function that applies the 60/40 weighting.

**Verification:** Run both signals on the 4 test inputs from Milestone 3. Print both scores separately — check where they agree and disagree. Verify that the combined confidence score maps to the correct label threshold for each test input.

---

### Milestone 5 — Production layer

**Input to AI:** Transparency label variants section + Appeals workflow section + Architecture diagram.

**Request:** Generate (1) a `generate_label(confidence)` function that maps scores to the three label texts, and (2) the `POST /appeal` endpoint that updates status and appends to the audit log.

**Verification:** Submit inputs that produce all three label variants and confirm the exact text matches the spec. Test the appeal endpoint with a known `content_id` and verify the audit log entry shows `status: under_review` and `appeal_reasoning` populated.
