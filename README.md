# Enterprise AI Assistant with RAG & Analytics

A production-shaped backend that lets employees ask natural-language questions
over company documents (PDF/Word) and get grounded, source-cited answers —
plus a Power BI dashboard tracking usage.

**Stack:** Python, FastAPI, LangChain, OpenAI API, MySQL, AWS S3, Power BI

## Architecture

```
                 ┌─────────────┐
   client  ───▶  │  FastAPI    │
                 │  REST API   │
                 └──────┬──────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                 ▼
 ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
 │   AWS S3    │  │    MySQL     │  │  FAISS index  │
 │ file storage│  │ users, docs, │  │ (embeddings,  │
 │             │  │ query_logs   │  │  local disk)  │
 └─────────────┘  └──────┬───────┘  └──────┬───────┘
                          │                  │
                          ▼                  ▼
                   ┌─────────────┐   ┌───────────────┐
                   │  Power BI   │   │ LangChain RAG │
                   │  dashboard  │   │ chain + GPT   │
                   └─────────────┘   └───────────────┘
```

**Request flow for a question:**
1. User's question hits `POST /query`.
2. FAISS retrieves the top-k most relevant chunks (embedding similarity).
3. Chunks are inserted into a prompt that instructs the LLM to answer **only**
   from that context — this is what keeps answers grounded instead of
   hallucinated.
4. The answer, its sources, and latency are logged to MySQL (`query_logs`) for
   both the user's history and the analytics dashboard.

## Project layout

```
app/
  main.py            FastAPI app, router registration, startup hook
  config.py          Settings (env-driven)
  database.py        SQLAlchemy engine/session (MySQL)
  models.py          User, Document, QueryLog tables
  schemas.py         Pydantic request/response models
  auth.py            Password hashing + JWT
  storage.py         S3 upload/download/delete wrapper
  rag/
    ingestion.py      Load PDF/Word -> chunk -> embed -> FAISS index
    pipeline.py        Retrieval + grounded-answer generation chain
  routers/
    auth.py           /auth/register, /auth/login
    documents.py       /documents (upload, list, delete)
    query.py           /query (ask), /query/history
    analytics.py        /analytics/* (usage metrics for Power BI / dashboards)
scripts/init_db.py    Create tables + optional admin user
sql/analytics_views.sql  MySQL views for Power BI's native connector
docker-compose.yml     Local MySQL for development
```

## Setup

### 1. Install dependencies
```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

### 2. Configure environment
```bash
cp .env.example .env
# fill in OPENAI_API_KEY, MySQL creds, AWS creds, SECRET_KEY
```

### 3. Start MySQL (or point at an existing instance)
```bash
docker compose up -d
```

### 4. Create an S3 bucket
```bash
aws s3 mb s3://enterprise-rag-documents --region us-east-1
```

### 5. Initialize the database
```bash
python scripts/init_db.py --admin-email admin@company.com --admin-password changeme
```

### 6. Run the API
```bash
uvicorn app.main:app --reload
```
Interactive docs: `http://localhost:8000/docs`

## Using the API

```bash
# Log in (form-encoded, OAuth2 password flow)
curl -X POST http://localhost:8000/auth/login \
  -d "username=admin@company.com&password=changeme"
# -> {"access_token": "...", "token_type": "bearer"}

# Upload a document
curl -X POST http://localhost:8000/documents/upload \
  -H "Authorization: Bearer <token>" \
  -F "file=@handbook.pdf"

# Ask a question
curl -X POST http://localhost:8000/query \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"question": "What is the company'\''s remote work policy?"}'
```

## Connecting Power BI

Two options, in order of preference:

**A. Direct MySQL connection (recommended, scales better)**
1. Run `sql/analytics_views.sql` against your database to create the views.
2. In Power BI Desktop: `Get Data` → `MySQL database` → host/port/database →
   select `vw_daily_query_volume`, `vw_top_questions`, `vw_document_activity`,
   `vw_user_activity`.
3. Build visuals: line chart of daily query volume, bar chart of top
   questions, table of document status, card visuals for headline KPIs.
4. Set a scheduled refresh in Power BI Service for a live dashboard.

**B. REST API (quick/local testing)**
`Get Data` → `Web` → point at `http://localhost:8000/analytics/daily-query-volume`
(and the other `/analytics/*` endpoints), adding an `Authorization: Bearer <token>`
header under advanced options. Fine for prototyping; the MySQL route is better
for production since it avoids re-authenticating on every refresh.

## Notes on scaling / hardening this further

- **Vector store:** FAISS-on-disk is simple and fast to stand up locally. For
  multi-instance deployments, swap it for a managed/hosted vector DB
  (pgvector, Pinecone, or Chroma with a server) so all API replicas share one
  index.
- **Background jobs:** document ingestion currently runs as a FastAPI
  `BackgroundTask`. For heavy upload volume, move it to a real task queue
  (Celery/RQ + Redis) so ingestion survives API restarts.
- **Migrations:** `Base.metadata.create_all` is fine for first run; use
  Alembic for schema changes after that.
- **Auth:** JWT is set up here as a self-contained example. For an actual
  enterprise deployment, consider SSO (SAML/OIDC) via your identity provider.
