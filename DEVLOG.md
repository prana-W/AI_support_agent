# DEVLOG.md — Development Timeline & Activity Log

All significant architectural steps, subtasks, decisions, and updates are tracked chronologically here.

---

## 2026-09-13

### 22:16 — Project Initialization & Planning
- Received Hiver SDE Take-Home assignment specifications: AI Support Agent for Amazon (`@AmazonHelp`).
- Established core constraints: simplicity, proof of correctness over over-engineering, 4-5 hour budget.
- Created `AGENT.md` defining non-negotiable tech stack (FastAPI, LangChain, Google Gemini, ChromaDB, PostgreSQL, Docker) and coding guidelines.

### 22:34 — Kaggle Dataset Acquisition
- Configured Kaggle API credentials via `KAGGLE_TOKEN`.
- Automated download of `thoughtvector/customer-support-on-twitter` (`twcs.csv`, 493MB / 2.81M tweets) into `data/raw/`.

### 22:37 — Repository Scaffolding (Subtask 1)
- Generated `.gitignore` covering `.venv/`, `data/`, secrets, cache, and Docker volumes.
- Created `.env.example` as a safe configuration template.
- Implemented `app/config.py` using `pydantic-settings` to serve as single source of truth for configuration.
- Initial git commit (`fb42a10`).

### 22:45 — Data Cleaning & Filtering Pipeline (Subtask 2)
- Implemented `scripts/prepare_data.py` to extract conversations for `@AmazonHelp`.
- Filtered 169,840 Amazon outbound tweets and reconstructed `(customer_message, agent_reply)` pairs.
- Added fast ASCII-ratio heuristic (>80%) to remove non-English tweets.
- Extracted 154,155 valid pairs and saved initial dataset to `data/processed/amazon_conversations.jsonl`.
- Committed Subtask 2 (`7c61936`).

---

## 2026-09-14

### 22:50 — Environment & Dependency Management (Subtask 3)
- Created isolated Python virtual environment `.venv/`.
- Updated `requirements.txt` with latest compatible packages for Python 3.13 (FastAPI, LangChain ecosystem, ChromaDB, SQLAlchemy, asyncpg, scikit-learn).
- Configured `.vscode/settings.json` for IDE interpreter integration.
- Installed and verified all dependencies in `.venv/`.

### 23:02 — ChromaDB Vector Indexing Pipeline
- Authored `scripts/build_index.py` using LangChain's `Chroma` and `GoogleGenerativeAIEmbeddings`.
- Implemented document structure: customer query embedded as vector, agent reply stored in metadata for few-shot prompt context retrieval.
- Integrated rate-limit handling and automatic exponential retry for Google GenAI embedding API.
- Commenced batch indexing into `data/chroma/`.

### 23:07 — Data Models & Retrieval Engine (Subtask 4)
- Implemented `app/models.py` with Pydantic request/response schemas for `/agent`, `/classify`, and `/reply`.
- Implemented `app/retrieval.py` providing `SupportRetriever` wrapper for semantic similarity search over historical Amazon support resolutions.

### 23:10 — Core LangChain Support Agent (Subtask 5)
- Implemented `app/agent.py` containing three LangChain LCEL chains:
  1. `classify_chain`: Intent classification with structured JSON output and confidence score across 8 standard intents.
  2. `reply_chain`: Historical Twitter RAG context injection for empathy, policy compliance, and length restraint (full draft + <=280 character Twitter reply).
  3. `escalation_decider`: Comprehensive escalation heuristic evaluating confidence score (<0.6), high-risk intents (`refund_return`, `billing_payment`, `account_access`), and urgency/legal indicators.
- Verified end-to-end execution on live test cases (`order_tracking` auto-handled vs. `billing_payment` escalated).
