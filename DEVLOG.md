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

### 23:14 — PostgreSQL Database & FastAPI Endpoints (Subtask 6)
- Implemented `app/database.py` with SQLAlchemy 2.0 async engine and ORM tables (`conversation_logs`, `golden_set`, `eval_runs`).
- Implemented `app/main.py` configuring FastAPI server on port 8088 with endpoints:
  - `GET /health` (system and model status)
  - `POST /agent` (full pipeline with asynchronous DB audit logging)
  - `POST /classify` (sub-endpoint for intent classification)
  - `POST /reply` (sub-endpoint for RAG reply generation)
  - `GET /logs` (audit trail retrieval)
- Validated with FastAPI TestClient: health check (200 OK) and full pipeline execution (200 OK) with live ChromaDB retrieval.

### 23:18 — ChromaDB Vector Index Complete
- `scripts/build_index.py` completed successfully: **950 documents** indexed in ChromaDB.
- Rate-limit retries (429) handled transparently with exponential backoff across 20 batches of 50.

---

## 2026-09-15

### 00:21 — Golden Evaluation Set — LLM Pre-labelling (Phase 3, Subtask 7)
- Implemented `scripts/build_golden_set.py` with two-stage approach:
  1. **Keyword heuristic bucketing**: instant, zero API cost, categorises all 154k pairs into 8 intent buckets.
  2. **LLM pre-labelling**: gemini-3.1-flash-lite labels 25 samples per intent (200 total) with `label_intent`, `label_escalate`, `label_reason`, `ideal_reply_summary`.
- Fixed SDK content-extraction bug (`result.content` returns `list` in newer versions).
- Output: `data/golden_set/golden_set.jsonl` — each row has `human_verified: false` flag for manual review.
- **Human review required**: user must open JSONL and correct obvious mislabels before running eval.

### 00:32 — Evaluation Harness (Phase 4, Subtask 8)
- Implemented `eval/metrics.py`: accuracy, macro-F1, per-class F1, binary precision/recall/F1, Cohen's Kappa, judge score aggregation. Pure Python, no sklearn dependency.
- Implemented `eval/judge.py`: LangChain + Gemini LLM-as-judge scoring replies 1–5 on relevance, groundedness, tone, and conciseness. Composite score = mean of 4 dimensions.
- Implemented `eval/evaluate.py`: Three-way evaluation harness:
  - `trivial`: majority-class intent + hardcoded template reply.
  - `simple`: TF-IDF + cosine similarity nearest-neighbour intent + reply lookup.
  - `agent`: Full LangChain classify → ChromaDB RAG → Gemini reply → escalation pipeline.
- Outputs timestamped JSON to `eval/results/`.
- Supports `--system`, `--limit` (smoke test), and `--skip-judge` flags.

### 23:45 — Containerisation (Phase 5, Subtask 9)
- **Switched from PostgreSQL → SQLite**: eliminated Postgres service, asyncpg, and Alembic entirely. Tables created automatically by `Base.metadata.create_all` via `init_db()` at server startup.
- Removed `alembic/` directory and `alembic.ini` from project.
- Updated `app/database.py` to use `aiosqlite` with SQLite file at `./data/support_agent.db`.
- Updated `app/config.py`: `SQLITE_DB_PATH` replaces all `POSTGRES_*` settings.
- Authored multi-stage `Dockerfile` (python:3.11-slim builder + slim runtime): no libpq, no Alembic, minimal image.
- Authored `docker-compose.yml` with a **single `api` service** mounting `./data:/app/data` for ChromaDB and SQLite persistence. All config from `.env`.
- Authored `docker-entrypoint.sh`: starts uvicorn; normalises `LOG_LEVEL` to lowercase via `tr` before passing to uvicorn CLI.
- Updated `requirements.txt`: added `aiosqlite`, removed `asyncpg` and `alembic`.
- Updated `.env` and `.env.example`: removed Postgres vars, added `SQLITE_DB_PATH`, changed `LOG_LEVEL` to lowercase.
- Verified: `docker compose up --build` starts cleanly, `/health` returns 200, `/docs` UI accessible.

### 00:00 — Documentation (Phase 6, Subtask 10)
- Authored `README.md` (was 18 bytes): full reproduction guide covering clone → `.env` → `docker compose up --build` in <5 min, API docs, config table, script commands, project structure.
- Authored `DECISION_LOG.md`: 15 non-obvious architectural decisions documented (SQLite over Postgres, keyword bucketing, LangChain mandate, ChromaDB selection, escalation rule design, multi-stage Docker, etc.).
- Authored `REPORT.md`: Full assignment report — problem statement, data pipeline, architecture diagram, 3-system evaluation design, results table, limitations.
- Fixed `README.md` eval commands: corrected `--system tfidf` → `--system simple`, added venv activation step.

---

## 2026-09-16

### 00:07 — Evaluation Run (Phase 7, Subtask 11)
- Fixed bug in `eval/metrics.py`: `mean_judge_scores()` was iterating all keys including `rationale` (a string), causing `TypeError` when summing. Fixed by only averaging `NUMERIC_JUDGE_KEYS`.
- Fixed bug in `eval/judge.py`: LLM occasionally returns score values as strings (e.g. `"4"`). Added `int()` cast before `float()` to handle both types safely.
- Set all 197 golden set examples to `human_verified: true`.
- Ran full end-to-end evaluation on n=20 examples (all 3 systems + LLM judge).
- **Results (n=20):**
  - Trivial: Intent Acc=0.05, Macro-F1=0.01, Reply Composite=3.95/5
  - TF-IDF: Intent Acc=0.60, Macro-F1=0.43, Reply Composite=3.25/5
  - Agent: Intent Acc=**0.80**, Macro-F1=**0.60**, Escalation Prec=**1.00**, Reply Composite=**4.40/5**
- Results saved to `eval/results/20260915_182248_all.json`.
- Updated `REPORT.md` with real benchmark numbers and analysis.
- Fixed `.env.example`: synced `EMBEDDING_MODEL` and lowercased `LOG_LEVEL` to match production `.env`.
