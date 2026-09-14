# AI Support Agent — Amazon @AmazonHelp

An AI-powered customer support agent for Amazon tweets. Classifies intent, retrieves similar historical resolutions via RAG (ChromaDB), drafts empathetic replies, and decides escalation — all running in Docker.

---

## Prerequisites

- Docker + Docker Compose (v2+)
- A Google Gemini API key → [Get one free](https://aistudio.google.com/app/apikey)

---

## Quickstart (< 5 minutes)

### 1 — Clone & enter the repo

```bash
git clone <repo-url>
cd AI_support_agent
```

### 2 — Create your `.env` file

```bash
cp .env.example .env
```

Open `.env` and fill in **one required value**:

```env
GOOGLE_API_KEY=your_key_here
```

All other values have sensible defaults and don't need to be changed.

### 3 — Start the stack

```bash
docker compose up --build
```

The API server is ready when you see:

```
api-1  | 🚀 Starting Amazon Support Agent API on port 8088...
api-1  | INFO:     Application startup complete.
```

### 4 — Test it

```bash
# Health check
curl http://localhost:8088/health

# Full agent pipeline (classify + RAG reply + escalation)
curl -s -X POST http://localhost:8088/agent \
  -H "Content-Type: application/json" \
  -d '{"message": "My order still hasn'\''t arrived and it'\''s been 2 weeks!", "conversation_id": "test-001"}' \
  | python3 -m json.tool

# Interactive API docs
open http://localhost:8088/docs
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | System + model status |
| `POST` | `/agent` | Full pipeline: classify → RAG → reply → escalate |
| `POST` | `/classify` | Intent classification only |
| `POST` | `/reply` | RAG reply generation only |
| `GET` | `/logs` | Conversation audit trail |

### Example: `/agent` request

```json
{
  "message": "I was charged twice for my order #112-3456789",
  "conversation_id": "optional-session-id"
}
```

### Example: `/agent` response

```json
{
  "intent": "billing_payment",
  "confidence": 0.91,
  "reply_draft": "We're really sorry to hear you were double-charged — that's the last thing we want for you...",
  "twitter_reply": "Really sorry about the double charge! Please DM us your order # so we can sort this right away. 🙏",
  "should_escalate": true,
  "escalation_reason": "billing_payment intent requires human review",
  "retrieved_examples": 3
}
```

---

## Configuration (`.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `GOOGLE_API_KEY` | **required** | Gemini API key |
| `MODEL_NAME` | `gemini-3.1-flash-lite` | LLM for classification + reply |
| `EMBEDDING_MODEL` | `gemini-embedding-2` | Embedding model for ChromaDB |
| `FASTAPI_PORT` | `8088` | Port the API listens on |
| `LOG_LEVEL` | `info` | Uvicorn log level (lowercase) |
| `CHROMA_PERSIST_DIR` | `./data/chroma` | ChromaDB storage directory |
| `MAX_RETRIEVED_EXAMPLES` | `3` | RAG k — similar examples fetched |
| `SQLITE_DB_PATH` | `./data/support_agent.db` | Conversation log database |
| `GOLDEN_SET_SIZE` | `200` | Target golden set size |
| `GOLDEN_SET_PATH` | `./data/golden_set/golden_set.jsonl` | Golden set file |

---

## Data & ChromaDB Index

The `./data/` directory is mounted into the container. It contains:

```
data/
├── chroma/            ← ChromaDB vector index (950 Amazon docs)
├── golden_set/
│   └── golden_set.jsonl   ← 200 labelled evaluation examples
└── support_agent.db   ← SQLite conversation log (auto-created)
```

> **Note:** If `data/chroma/` is missing, run the indexing script first (see below).

---

## Running Scripts Manually (without Docker)

These scripts are one-off tools run on the host, not inside Docker.

```bash
# Create a virtual environment
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 1. Prepare raw data (requires data/raw/twcs.csv from Kaggle)
python scripts/prepare_data.py

# 2. Build ChromaDB vector index (~950 Amazon conversation pairs)
python scripts/build_index.py

# 3. Build golden evaluation set (200 LLM-pre-labelled examples)
python scripts/build_golden_set.py
```

---

## Evaluation

After manually reviewing `data/golden_set/golden_set.jsonl` (setting `human_verified: true`):

```bash
# Quick sanity check (trivial baseline, no LLM judge)
python eval/evaluate.py --system trivial --skip-judge

# TF-IDF baseline
python eval/evaluate.py --system tfidf --skip-judge

# Full agent evaluation
python eval/evaluate.py --system agent

# Run all 3 systems with a 10-example smoke test
python eval/evaluate.py --limit 10
```

Results are saved to `eval/results/<timestamp>.json`.

---

## Project Structure

```
AI_support_agent/
├── app/                    # FastAPI application (runs in Docker)
│   ├── main.py             # App entry point + route registration
│   ├── agent.py            # LangChain classify + reply + escalate chains
│   ├── retrieval.py        # ChromaDB vector store retrieval
│   ├── database.py         # SQLite + SQLAlchemy async ORM
│   ├── models.py           # Pydantic request/response schemas
│   └── config.py           # pydantic-settings (reads .env)
├── data/                   # Data directory (gitignored, Docker-mounted)
├── eval/                   # Evaluation harness
│   ├── evaluate.py         # 3-way evaluation runner
│   ├── judge.py            # LLM-as-judge (Gemini)
│   ├── metrics.py          # Accuracy, F1, Kappa calculations
│   └── results/            # JSON evaluation outputs
├── scripts/                # One-off data preparation scripts
│   ├── prepare_data.py     # Filter Amazon tweets from raw CSV
│   ├── build_index.py      # Embed + index into ChromaDB
│   └── build_golden_set.py # Build 200-example golden set
├── Dockerfile
├── docker-compose.yml
├── docker-entrypoint.sh
├── requirements.txt
├── .env.example
└── AGENTS.md               # Project rules and conventions
```

---

## Stopping

```bash
docker compose down
```

Data persists in `./data/` across restarts.