# REPORT.md — AI Customer Support Agent: Design, Methodology & Evaluation

> Assignment: Hiver SDE Intern Take-Home | Brand: Amazon (@AmazonHelp)
> Author: [Your Name] | Date: 2026-09-14

---

## 1. Problem Statement

The task is to build an AI-powered customer support agent capable of handling inbound tweets directed at Amazon's support account (`@AmazonHelp`). The system must:

1. **Classify** the intent of each customer tweet into one of 8 categories.
2. **Generate** an empathetic, grounded reply (≤280 characters for Twitter, plus a longer draft).
3. **Decide** whether the ticket should be escalated to a human agent.

The agent is evaluated against two baselines (trivial and TF-IDF) on a hand-labelled golden set of 200 examples using intent accuracy, escalation F1, and LLM-judged reply quality.

---

## 2. Dataset

**Source:** [Customer Support on Twitter](https://www.kaggle.com/datasets/thoughtvector/customer-support-on-twitter) (Kaggle, `twcs.csv`, 2.81M tweets, 493MB).

**Filtering pipeline (`scripts/prepare_data.py`):**
- Extracted all outbound Amazon tweets (author `@AmazonHelp`) and reconstructed `(customer_message, agent_reply)` pairs.
- Applied ASCII-ratio heuristic (>80%) to remove non-English content.
- Result: **154,155 valid pairs** saved to `data/processed/amazon_conversations.jsonl`.

**ChromaDB vector index (`scripts/build_index.py`):**
- A random stratified sample of 1,000 pairs was embedded (model: `gemini-embedding-2`) and indexed into ChromaDB.
- **950 documents** stored after deduplication.
- Customer message is the embedded vector; agent reply is stored in metadata for few-shot retrieval.

---

## 3. Golden Evaluation Set

**Construction (`scripts/build_golden_set.py`):**

The golden set is built in two stages:

1. **Keyword bucketing:** Each of the 154k pairs is assigned to one of 8 intent buckets using keyword heuristics (e.g., `"where is my order"` → `order_tracking`). This runs in milliseconds with zero API cost.

2. **LLM pre-labelling:** 25 examples are sampled per intent (200 total). Gemini `gemini-3.1-flash-lite` labels each with: `label_intent`, `label_escalate`, `label_reason`, `ideal_reply_summary`.

3. **Human review:** Each row includes a `human_verified: false` flag. A human reviewer manually checks and corrects labels, setting `human_verified: true` for confirmed rows.

**Distribution (LLM pre-labelled):**

| Intent | Count |
|--------|-------|
| delivery_issue | 75 |
| account_access | 27 |
| product_issue | 23 |
| other | 22 |
| refund_return | 20 |
| billing_payment | 18 |
| general_inquiry | 9 |
| order_tracking | 6 |
| **Total** | **200** |

> Note: `delivery_issue` is over-represented because the keyword bucket is broad (generic complaint language). The human review phase corrects mislabelled rows.

---

## 4. System Architecture

### 4.1 Tech Stack

| Layer | Technology |
|-------|-----------|
| API framework | FastAPI (port 8088) |
| LLM orchestration | LangChain (LCEL) |
| LLM backend | `gemini-3.1-flash-lite` (Google) |
| Embeddings | `gemini-embedding-2` via LangChain |
| Vector store | ChromaDB (persistent, local) |
| Database | SQLite + aiosqlite (audit log, golden set, eval runs) |
| Containerisation | Docker + Docker Compose (single service) |

### 4.2 Agent Pipeline

```
Customer Tweet
      │
      ▼
┌─────────────────────┐
│  classify_chain     │  → intent + confidence (0–1)
│  (LCEL + JSON out)  │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  ChromaDB retrieval │  → top-k (k=3) similar historical resolutions
│  (semantic search)  │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  reply_chain        │  → full draft + ≤280 char Twitter reply
│  (RAG few-shot)     │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Escalation decider │  → should_escalate (bool) + reason
│  (rule-based)       │
└─────────────────────┘
      │
      ▼
  Structured JSON Response
```

### 4.3 Intent Categories

| # | Intent | Auto-handle | Escalate |
|---|--------|-------------|---------|
| 1 | `order_tracking` | ✓ | — |
| 2 | `delivery_issue` | ✓ | — |
| 3 | `general_inquiry` | ✓ | — |
| 4 | `product_issue` | if neutral | if angry |
| 5 | `refund_return` | — | ✓ |
| 6 | `billing_payment` | — | ✓ |
| 7 | `account_access` | — | ✓ |
| 8 | `other` | if confident | if low confidence |

### 4.4 Escalation Rules

A ticket is escalated if **any** of these conditions hold:
- Intent is `refund_return`, `billing_payment`, or `account_access`
- Classification confidence < 0.6
- Message contains anger/frustration signals (keywords: *furious*, *lawsuit*, *disgusting*, *fraud*, etc.)
- Message contains legal threats or media mentions (*lawyer*, *BBB*, *news*, *media*)

---

## 5. Evaluation Design

### 5.1 Three Systems

| System | Intent Classification | Reply Generation |
|--------|----------------------|-----------------|
| **Trivial** | Majority class (`delivery_issue`) | Hardcoded template |
| **TF-IDF** | TF-IDF cosine similarity nearest-neighbour | Retrieved historical reply |
| **Agent** | Gemini LLM (few-shot, JSON output) | Gemini LLM (RAG context) |

### 5.2 Metrics

**Intent Classification:**
- Accuracy (% correct)
- Macro-F1 (unweighted mean F1 across all 8 classes)
- Per-class F1

**Escalation:**
- Precision, Recall, F1 (binary)

**Reply Quality (LLM-as-judge, scale 1–5):**
- Relevance — does the reply address the customer's actual issue?
- Groundedness — is the reply grounded in the retrieved examples, not hallucinated?
- Tone — is the reply empathetic and professional?
- Conciseness — is the reply appropriately brief?
- **Composite** = mean of all 4 dimensions

**Human agreement:**
- Cohen's Kappa on a 30-example subset (human scores vs. LLM judge scores)

### 5.3 Results

> ⚠️ This section will be populated after running `eval/evaluate.py` on the reviewed golden set.

| Metric | Trivial | TF-IDF | Agent |
|--------|---------|--------|-------|
| Intent Accuracy | — | — | — |
| Intent Macro-F1 | — | — | — |
| Escalation Precision | — | — | — |
| Escalation Recall | — | — | — |
| Escalation F1 | — | — | — |
| Reply Relevance (1–5) | — | — | — |
| Reply Groundedness (1–5) | — | — | — |
| Reply Tone (1–5) | — | — | — |
| Reply Conciseness (1–5) | — | — | — |
| Reply Composite (1–5) | — | — | — |
| Human–Judge Kappa | — | — | — |

---

## 6. Limitations & Future Work

**Current limitations:**
- Golden set is LLM pre-labelled and may contain labelling noise in unreviewed rows.
- The `delivery_issue` bucket is over-represented (75/200) due to coarse keyword heuristics.
- Escalation relies on keyword-based anger detection — a fine-tuned sentiment classifier would be more reliable.
- Same model is used as both agent and judge (Gemini `gemini-3.1-flash-lite`), introducing same-model bias in reply quality scores.
- No streaming — responses are synchronous. For a Twitter bot, p99 latency matters.

**Future work:**
- Fine-tune on verified golden set replies for better grounding and reply consistency.
- Replace keyword-based escalation sentiment with a dedicated classifier.
- Implement confidence calibration (the confidence score from the LLM is un-calibrated).
- Expand to multi-brand support with per-brand ChromaDB collections.
- Add streaming endpoints for real-time agent handoff UX.
