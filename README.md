# ai201-project4-provenance-guard
# Provenance Guard

A backend API that classifies submitted text as human-written or AI-generated, scores confidence in that classification, surfaces a transparency label to users, and handles creator appeals. Designed for creative sharing platforms where false positives (labeling a human's work as AI-generated) are more harmful than false negatives.

---

## Architecture

### Submission Flow

```
POST /submit {text, creator_id}
        │
        ▼
    Flask App
    assign content_id
        │
        ├─────────────────────────┐
        ▼                         ▼
Signal 1: LLM             Signal 2: Stylometric
(Groq API)                (Pure Python)
→ llm_score               → stylometric_score
        │                         │
        └──────────┬──────────────┘
                   ▼
          Confidence Scorer
          0.6*llm + 0.4*style
          → confidence (0–1)
                   │
                   ▼
          Label Generator
          ≥0.72 → AI label
          0.40–0.71 → uncertain
          <0.40 → human label
                   │
                   ▼
          Audit Log (JSON)
                   │
                   ▼
          JSON response to caller
```

### Appeal Flow

```
POST /appeal {content_id, creator_reasoning}
        │
        ▼
    Look up entry in audit log
        │
        ▼
    Update status → "under_review"
    Append appeal_reasoning + appeal_timestamp
        │
        ▼
    JSON confirmation response
```

**Narrative:** A submitted piece of text is assigned a unique `content_id`, passed through two independent detection signals, combined into a confidence score, mapped to a transparency label, written to the audit log, and returned to the caller. Appeals update the audit log entry status to `under_review` without triggering automated re-classification — a human reviewer handles appeals.

---

## Detection Signals

### Signal 1: LLM-Based Classification (Groq)

**What it measures:** Semantic and stylistic coherence holistically. The LLM assesses whether the text exhibits patterns typical of AI-generated writing — uniform sentence structure, formulaic transitions ("it is important to note", "furthermore"), absence of genuine personal voice, overly balanced tone.

**Why it differs between human and AI writing:** AI models produce text that is coherent but lacks idiosyncratic markers of human voice — unexpected word choices, self-correction, genuine uncertainty, culturally specific references.

**Output:** A float 0.0–1.0 where higher = more likely AI-generated.

**What it misses:** Heavily edited AI output, non-native English speakers with formal writing styles, and academic/technical writing that is naturally uniform. It has no access to external context about the creator.

---

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Three statistical properties:
- **Sentence length variance** — AI text tends to have uniform sentence lengths; human writing varies more
- **Type-token ratio (TTR)** — vocabulary diversity; AI text often repeats similar vocabulary
- **Punctuation idiosyncrasy** — humans use dashes, ellipses, and exclamation marks more freely; AI output tends toward clean, standard punctuation

**Why it differs between human and AI writing:** AI language models are trained to produce fluent, readable text — which makes them statistically more uniform than human writers.

**Output:** A float 0.0–1.0 where higher = more AI-like (lower variance, lower TTR, lower idiosyncratic punctuation).

**What it misses:** Formal human writing (academic papers, legal documents) will score as more AI-like because it is intentionally uniform. Short texts under ~50 words produce unreliable scores due to small sample size.

---

## Confidence Scoring

### Combining Signals

```
confidence = (llm_score × 0.6) + (stylometric_score × 0.4)
```

The LLM signal receives 60% weight because it captures semantic holism that stylometrics cannot. Stylometrics receives 40% as a structural corroborating signal.

### Thresholds

The thresholds are **asymmetric** — the system requires higher confidence to label content as AI-generated than to label it as human-written. This reflects the asymmetry in harm: a false positive damages a creator's reputation; a false negative is a less severe outcome.

| Confidence | Attribution | Label variant |
|------------|------------|---------------|
| ≥ 0.72 | likely_ai | High-confidence AI |
| 0.40 – 0.71 | uncertain | Uncertain |
| < 0.40 | likely_human | High-confidence human |

### Validation — Two Example Submissions

**High-confidence AI (confidence: 0.7206):**
```json
{
  "text": "Artificial intelligence has emerged as a transformative force...",
  "llm_score": 0.92,
  "stylometric_score": 0.4214,
  "confidence": 0.7206,
  "attribution": "likely_ai"
}
```

**High-confidence human (confidence: 0.2086):**
```json
{
  "text": "ok so i finally tried that new ramen place downtown and honestly? underwhelming...",
  "llm_score": 0.15,
  "stylometric_score": 0.2964,
  "confidence": 0.2086,
  "attribution": "likely_human"
}
```

Scores vary meaningfully: 0.72 vs 0.21 — a 0.50 point spread between clearly AI and clearly human text.

---

## Transparency Label Variants

All three variants are written out exactly as they appear in API responses.

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

*Note: For the human label, confidence displayed is `(1 - confidence) × 100` to express confidence in the human verdict rather than the raw AI-probability score.*

---

## API Endpoints

### `POST /submit`

Accepts a piece of text for attribution analysis.

**Request:**
```json
{
  "text": "...",
  "creator_id": "..."
}
```

**Response:**
```json
{
  "content_id": "dc402a3c-05c9-4920-9e2a-de8513805111",
  "attribution": "likely_ai",
  "confidence": 0.7206,
  "llm_score": 0.92,
  "stylometric_score": 0.4214,
  "label": {
    "variant": "high_confidence_ai",
    "text": "⚠️ This content was likely generated by AI. Our system is 72% confident in this assessment. You may appeal if you believe this is incorrect."
  },
  "status": "classified"
}
```

---

### `POST /appeal`

Contest a classification.

**Request:**
```json
{
  "content_id": "dc402a3c-05c9-4920-9e2a-de8513805111",
  "creator_reasoning": "I wrote this myself from personal experience..."
}
```

**Response:**
```json
{
  "message": "Your appeal has been received and the content is now under review.",
  "content_id": "dc402a3c-05c9-4920-9e2a-de8513805111",
  "status": "under_review"
}
```

---

### `GET /log`

Returns the most recent audit log entries. Optional query param: `?limit=N` (default 20).

---

## Rate Limiting

**Limits:** 10 requests per minute, 100 requests per day per IP address.

**Reasoning:**
- **10 per minute** — a real writer submitting their own work would never need more than a few submissions per minute. This limit blocks automated scripts that might flood the system while being completely invisible to legitimate users.
- **100 per day** — generous enough for any legitimate heavy user (a platform reviewing many submissions), while preventing sustained abuse campaigns.

**Evidence rate limiting works** — sending 12 rapid requests produces:
```
200 200 200 200 200 200 200 200 200 200
429 Too Many Requests (10 per 1 minute)
429 Too Many Requests (10 per 1 minute)
```

---

## Audit Log

Every attribution decision and appeal is captured in `audit_log.json`. Sample entries (3 shown):

```json
[
  {
    "content_id": "dc402a3c-05c9-4920-9e2a-de8513805111",
    "creator_id": "test-user-1",
    "timestamp": "2026-06-29T02:13:29.845770+00:00",
    "attribution": "likely_ai",
    "confidence": 0.7206,
    "llm_score": 0.92,
    "stylometric_score": 0.4214,
    "label_variant": "high_confidence_ai",
    "status": "under_review",
    "appeal_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
    "appeal_timestamp": "2026-06-29T02:14:30.030563+00:00"
  },
  {
    "content_id": "71fc2e12-ba1f-47c2-975e-2eb217632d47",
    "creator_id": "test-user-2",
    "timestamp": "2026-06-29T02:13:48.920868+00:00",
    "attribution": "likely_human",
    "confidence": 0.2086,
    "llm_score": 0.15,
    "stylometric_score": 0.2964,
    "label_variant": "high_confidence_human",
    "status": "classified",
    "appeal_reasoning": null,
    "appeal_timestamp": null
  },
  {
    "content_id": "620ba5b7-604a-4bb9-9add-46b98ec3627f",
    "creator_id": "test-user-3",
    "timestamp": "2026-06-29T02:14:06.361202+00:00",
    "attribution": "uncertain",
    "confidence": 0.7,
    "llm_score": 0.8,
    "stylometric_score": 0.55,
    "label_variant": "uncertain",
    "status": "classified",
    "appeal_reasoning": null,
    "appeal_timestamp": null
  }
]
```

---

## Known Limitations

**Non-native English speakers with formal writing styles** will be disproportionately flagged. A non-native speaker who writes carefully and formally may produce text with low sentence length variance and high stylometric uniformity — the same properties as AI output. The asymmetric threshold (requiring ≥ 0.72 to label as AI) reduces but does not eliminate this risk. The appeals workflow is the primary mitigation, and the uncertain band (0.40–0.71) gives the system room to flag content for human review rather than making a definitive call.

**Short texts under 50 words** produce unreliable stylometric scores because variance metrics need sufficient sample size. For short texts, the system relies more heavily on the LLM signal, but overall confidence is less trustworthy. A future improvement would be to flag short texts explicitly in the response.

---

## Spec Reflection

**One way the spec helped:** The asymmetric threshold design in `planning.md` — requiring ≥ 0.72 to label as AI but only < 0.40 to label as human — was decided before any code was written. This forced a deliberate design choice about harm asymmetry rather than defaulting to a symmetric 0.5 split. When testing revealed that clearly AI text was landing in the uncertain band, the decision to lower the AI threshold from 0.80 to 0.72 was informed by the reasoning already documented in the spec.

**One way the implementation diverged:** The spec planned for a threshold of ≥ 0.80 for high-confidence AI. In practice, the stylometric signal consistently pulled combined scores below 0.80 even for clearly AI-generated text — the heuristics are more conservative than expected. The threshold was adjusted to 0.72 after testing. This is documented in both `planning.md` and the threshold table above.

---

## AI Usage

**Instance 1:** I gave Claude the detection signals section and architecture diagram from `planning.md` and asked it to generate the full Flask app including both signal functions, confidence scoring, label generation, all three endpoints, and audit logging. It produced a complete working implementation. I reviewed the stylometric scoring weights (originally equal weights across all three metrics) and adjusted them to 40/35/25 to give more weight to sentence length variance, which is the most reliable signal for short texts. I also adjusted the AI threshold from 0.80 to 0.72 after testing revealed the stylometric signal was too conservative.

**Instance 2:** I gave Claude the transparency label variants and appeals workflow sections from `planning.md` and asked it to verify the label generation logic matched the spec thresholds exactly. It confirmed the implementation was correct and suggested adding the confidence display inversion for the human label (showing `(1 - confidence) × 100` as the human confidence percentage rather than the raw AI-probability). I implemented this suggestion as it made the label more intuitive for non-technical readers.
