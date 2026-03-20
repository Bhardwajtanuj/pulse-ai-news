# Pulse — AI News Dashboard

> Real-time AI news aggregation, deduplication, and broadcasting.  
> Built for the Culinda Python/GenAI Intern assignment.

---

## Live Demo

| Service  | URL |
|----------|-----|
| Frontend | https://pulse-ai-news.vercel.app *(deploy link after hosting)* |
| API Docs | https://pulse-ai-news-api.onrender.com/docs |

---

## What it does

Pulse continuously ingests AI news from **22 sources**, deduplicates stories across outlets, enriches each article with Claude AI, and lets you save and broadcast the best ones — all in under 3 clicks.

```
[22 RSS Sources]
       ↓
[Async Ingestion Worker]  ←  runs every 30 min
       ↓
[Normalizer + Batch Dedup]
       ↓
[PostgreSQL]  →  [FastAPI]  →  [Next.js Dashboard]
       ↓                              ↓
[Claude AI Enrichment]        [Broadcast Layer]
  • summarize                    • Email (SendGrid)
  • tag extraction               • LinkedIn caption
  • impact scoring (1–10)        • WhatsApp deep link
  • embeddings for dedup         • Blog / Newsletter queue
```

---

## Architecture

### Ingestion Layer
- `feedparser` + `httpx` fetch all 22 RSS feeds concurrently (8-way semaphore)
- Each entry is normalized into a unified schema: title, URL, summary, author, date, image
- A **batch dedup pass** runs before DB insert — URL exact match + Jaccard title similarity catches same-story reposts within a fetch cycle

### AI Processing Layer (Claude claude-3-haiku-20240307)
- One batched prompt per article returns `summary`, `tags[]`, `impact_score`, and `impact_reason`
- A second pass generates **vector embeddings** (OpenAI `text-embedding-3-small` if key present, deterministic hash-vector fallback otherwise)
- Cosine similarity against recent items catches cross-source duplicates of the same story
- LinkedIn captions use `claude-3-5-sonnet` — the better creative model

### Storage Layer
- PostgreSQL via async SQLAlchemy 2.0
- `news_items.embedding` stored as JSON array (pgvector-ready for production upgrade)
- Redis for API response caching (3-min TTL on feed endpoints)

### API Layer (FastAPI)
- `GET /api/v1/news/` — paginated feed with search, sort, tag filter
- `GET /api/v1/news/stats` — dashboard stats
- `POST /api/v1/news/ingest` — manual trigger
- `GET/POST/DELETE /api/v1/favorites/{id}` — favorites CRUD
- `POST /api/v1/favorites/{id}/broadcast` — broadcast with AI caption generation

### Frontend (Next.js 14 + Tailwind)
- Linear-inspired design system with full dark/light/system theme support
- Infinite scroll feed with impact score rings, AI summary expand, tag chips
- Broadcast modal: platform picker → AI-generated caption → copy/open (≤3 clicks, per BRD)
- SWR for data fetching with optimistic UI updates

---

## Schema

```sql
sources        — id, name, url, rss_url, source_type, active, last_fetched_at
news_items     — id, source_id, title, url, original_summary, ai_summary,
                  author, image_url, tags (JSON), impact_score, embedding (JSON),
                  is_duplicate, duplicate_of_id, published_at, ai_processed_at
favorites      — id, news_item_id, created_at
broadcast_logs — id, favorite_id, platform, status, ai_caption, recipient, created_at
```

---

## Quick start (Docker)

```bash
# 1. Clone
git clone https://github.com/yourhandle/ai-news-dashboard
cd ai-news-dashboard

# 2. Set env vars
cp .env.example .env
# Edit .env — add ANTHROPIC_API_KEY at minimum

# 3. Start everything
docker compose up --build

# Frontend → http://localhost:3000
# API docs → http://localhost:8000/docs
```

The ingestion pipeline runs automatically on startup and every 30 minutes.  
To trigger it manually: click **Refresh feed** in the sidebar, or `POST /api/v1/news/ingest`.

---

## Local development (without Docker)

### Backend

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env   # add your ANTHROPIC_API_KEY

# Start Postgres + Redis (via Docker or locally)
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=password postgres:16-alpine
docker run -d -p 6379:6379 redis:7-alpine

uvicorn app.main:app --reload --port 8000
```

### Frontend

```bash
cd frontend
npm install
cp .env.example .env.local
# NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1

npm run dev   # → http://localhost:3000
```

---

## Deploying to production

### Backend → Render

1. Create a new **Web Service** on [render.com](https://render.com)
2. Root directory: `backend`
3. Build command: `pip install -r requirements.txt`
4. Start command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
5. Add environment variables from `.env.example`
6. Add a **PostgreSQL** database on Render (free tier) — copy the connection string to `DATABASE_URL`
7. Add a **Redis** instance (Render or Upstash free tier)

### Frontend → Vercel

```bash
cd frontend
npx vercel --prod
# Set NEXT_PUBLIC_API_URL to your Render backend URL
```

---

## Sources (22 feeds)

| # | Source | Type |
|---|--------|------|
| 1 | OpenAI Blog | RSS |
| 2 | Anthropic | RSS |
| 3 | Google AI Blog | RSS |
| 4 | DeepMind | RSS |
| 5 | Hugging Face | RSS |
| 6 | Meta AI | RSS |
| 7 | TechCrunch AI | RSS |
| 8 | VentureBeat AI | RSS |
| 9 | Wired AI | RSS |
| 10 | MIT Tech Review | RSS |
| 11 | The Verge | RSS |
| 12 | Ars Technica | RSS |
| 13 | arXiv cs.AI | RSS |
| 14 | arXiv cs.LG | RSS |
| 15 | arXiv cs.CL | RSS |
| 16 | Hacker News (AI filter) | RSS |
| 17 | Reddit r/MachineLearning | RSS |
| 18 | Microsoft AI Blog | RSS |
| 19 | Papers With Code | RSS |
| 20 | AWS ML Blog | RSS |
| 21 | Towards Data Science | RSS |
| 22 | NVIDIA AI Blog | RSS |

---

## GenAI integration points

| Feature | Model | Prompt strategy |
|---------|-------|-----------------|
| Article summarization | claude-3-haiku | Batched JSON: summary + tags + score in one call |
| Impact scoring (1–10) | claude-3-haiku | Same call as summarization |
| Tag extraction | claude-3-haiku | Fixed taxonomy of 16 tags |
| Dedup embeddings | text-embedding-3-small (or hash fallback) | Cosine similarity threshold: 0.85 |
| LinkedIn caption | claude-3-5-sonnet | Voice: "thoughtful practitioner, not a bot" |
| WhatsApp message | claude-3-haiku | Casual, <100 words, includes URL |
| Email newsletter | claude-3-haiku | Plain-text, formatted, up to 10 articles |

---

## Design decisions

**Why sync ingestion instead of Celery workers?**  
For an MVP with a 1.5-day deadline, background tasks via FastAPI's `asyncio` deliver the same result without the ops overhead of managing Celery beat + workers in Docker Compose. The service layer is fully abstracted — swapping in Celery is a one-file change when needed.

**Why JSON for embeddings instead of pgvector?**  
pgvector requires a Postgres extension that isn't available on all managed hosts (e.g. Render free tier). Storing as JSON array and computing cosine similarity in Python delivers the same dedup quality for datasets under ~50k articles. The schema is forward-compatible — adding a `VECTOR(1536)` column and migrating is straightforward once you upgrade the DB.

**Why one batched AI call per article?**  
Getting summary + tags + score in a single prompt costs one API call vs three. At 30 articles per cycle, that's a 3x cost reduction and 3x latency reduction during enrichment.

**Why claude-3-haiku for enrichment and claude-3-5-sonnet for captions?**  
Haiku is fast and cheap for structured extraction (JSON output). Sonnet is better for creative writing — LinkedIn posts need voice and personality, which haiku tends to flatten.

---

## Project structure

```
ai-news-dashboard/
├── backend/
│   ├── app/
│   │   ├── core/          # config, database, redis
│   │   ├── models/        # SQLAlchemy ORM models
│   │   ├── services/      # ai_service, ingestion, dedup, broadcast
│   │   ├── workers/       # ingestion pipeline orchestrator
│   │   └── api/           # FastAPI routers (news, favorites)
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── app/           # Next.js App Router pages
│   │   ├── components/    # Sidebar, NewsCard, BroadcastModal, FilterBar
│   │   ├── lib/           # api client, utils
│   │   └── types/         # TypeScript interfaces
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Author

Tanuj Bhardwaj — built for Culinda GenAI Intern assignment, March 2026.
# pulse-ai-news
