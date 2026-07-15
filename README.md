# JobScripts

**A multi-source job aggregation and search engine — built to demonstrate real backend architecture, not just CRUD.**

JobScripts pulls live job listings from multiple public job APIs, stores the data across three purpose-built databases, indexes it for fast full-text and faceted search, and exposes it through a documented REST API with JWT authentication. It does not process applications — it's a discovery layer, the same way Google Jobs or Indeed's aggregation layer works: search, filter, bookmark, then hand off to the original posting to apply.

---

## Why This Project Exists

Most portfolio projects are a single database behind a CRUD API. JobScripts intentionally isn't — it's built around a question every backend engineer eventually has to answer: **what happens when your data comes from external sources you don't control, at a scale where a single database engine can't do everything well?**

The answer here is three data stores, each doing exactly one job:

| Store | Role | Why not just use Postgres for everything? |
|---|---|---|
| **MongoDB** | Stores the raw, untouched JSON response from every API call, before any parsing | If a source API changes its schema, or a parsing bug ships, the original data is never lost — it can be reprocessed from scratch |
| **PostgreSQL** | Stores the normalized, canonical `Job` record — the source of truth for the app | Relational integrity for users, bookmarks, and joins (`user → saved jobs`) that a document store handles awkwardly |
| **OpenSearch** | A search index built from the Postgres data | Full-text search and multi-field faceted filtering (skill, location, salary range) at speed a `WHERE title ILIKE '%...%'` query cannot sustain as data grows |

Every write flows through all three in a fixed order — **raw → canonical → indexed** — so each layer has a clear, single responsibility.

---

## How It Works — Step by Step

This is the exact path a job listing takes, and the exact path a search request takes, end to end.

### 1. Ingestion (scheduled, automatic)
- An APScheduler job runs on a fixed interval (default: every 4 hours).
- It calls the RemoteOK public API and the Adzuna API (`app_id`/`app_key` from a free developer account).
- Each raw JSON response is written to MongoDB immediately, **before** any parsing happens — this is the audit trail. Even a job with malformed fields survives here.

### 2. Normalization & Storage
- Each source has its own adapter that maps the source's fields onto one shared `Job` schema (`title`, `company`, `location`, `skills`, `salary_min`, `salary_max`, `source`, `source_id`, `url`, `posted_date`).
- The record is upserted into PostgreSQL using a unique constraint on `(source, source_id)`. Re-running the sync never creates duplicates — this is what makes the pipeline idempotent and safe to retry after a partial failure.

### 3. Indexing
- Immediately after a successful Postgres write, the same record is pushed into an OpenSearch index (`jobs`), with `skills`/`location` mapped as keyword facets and `salary_min`/`salary_max` mapped as a numeric range field.
- Postgres and OpenSearch are kept in lockstep on every write — there's no separate "reindex later" batch job in the normal path (a manual `/admin/reindex` endpoint exists as a safety net if the mapping ever changes).

### 4. Search
- `GET /jobs/search?q=&skill=&location=&min_salary=&max_salary=` never touches Postgres directly — it queries OpenSearch.
- The query is a `bool` query: a `multi_match` on `q` across title/company, `term` filters for skill/location, and a `range` filter for salary.
- The same call also runs a terms aggregation on `skills` and `location`, returning live facet counts alongside the results — this is what powers the filter chips in the UI ("Python (14)", "Remote (9)", etc.), computed from whatever the current query already matched, not the whole dataset.

### 5. Authentication
- `POST /auth/register` hashes the password (bcrypt) and stores the user in Postgres — plaintext passwords are never stored or returned.
- `POST /auth/login` verifies the password and returns a signed JWT.
- Every protected route (bookmarks, admin status) validates that JWT via a shared FastAPI dependency before touching the database.

### 6. Bookmarking
- `POST /jobs/{job_id}/bookmark` writes a row to a `bookmarks` join table (`user_id`, `job_id`), unique per pair — bookmarking twice is a no-op, not a duplicate.
- `GET /users/me/bookmarks` joins that table back to `jobs` to return the user's saved list.

### 7. Applying
- JobScripts does **not** submit applications. Every `Job` record carries the original `url` from its source. Clicking a job in the UI opens that URL directly — the same handoff pattern used by every major job aggregator for third-party-sourced listings.

### 8. Resilience
- If RemoteOK or Adzuna is down, times out, or rate-limits during a scheduled run, that source's failure is caught, logged to a `SyncRun` record with a status and error message, and the other source's sync continues unaffected. One dead API never takes down the whole pipeline.
- `GET /admin/sync-status` exposes the last run time, status, and inserted/skipped counts per source — basic observability into a background system, not just a black box that either "works" or silently doesn't.

---

## Architecture

```
                    ┌──────────────────┐
   APScheduler ───▶ │ RemoteOK / Adzuna │
   (every 4h)       │   Public APIs     │
                    └─────────┬─────────┘
                              │ raw JSON
                              ▼
                    ┌──────────────────┐
                    │     MongoDB       │  raw payloads, audit trail
                    └─────────┬─────────┘
                              │ normalized (idempotent upsert)
                              ▼
                    ┌──────────────────┐
                    │    PostgreSQL     │  canonical jobs, users, bookmarks
                    └─────────┬─────────┘
                              │ indexed on write
                              ▼
                    ┌──────────────────┐
                    │    OpenSearch     │  full-text + faceted search
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   FastAPI (REST)  │  JWT auth · search · bookmarks
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  React Frontend   │  search · filters · bookmarks
                    └──────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| API framework | FastAPI (Python) |
| Relational database | PostgreSQL + SQLAlchemy |
| Document store | MongoDB (Motor async driver) |
| Search engine | OpenSearch |
| Scheduler | APScheduler |
| Auth | JWT (`python-jose` / `passlib`) |
| Frontend | React + Vite + Tailwind CSS |
| Containerization | Docker Compose |
| Testing | pytest |
| API documentation | Swagger / OpenAPI (auto-generated by FastAPI at `/docs`) |
| Manual API testing | Postman (collection included, see `/backend/postman`) |
| External data sources | RemoteOK API (no key required), Adzuna API (free tier) |

---

## API Reference

Full interactive docs are auto-generated at `/docs` (Swagger UI) once the server is running. Summary:

| Method | Endpoint | Auth required | Description |
|---|---|---|---|
| `POST` | `/auth/register` | No | Create a new user account |
| `POST` | `/auth/login` | No | Authenticate and receive a JWT |
| `GET` | `/jobs/search` | No | Full-text + faceted search (`q`, `skill`, `location`, `min_salary`, `max_salary`) |
| `GET` | `/jobs/{job_id}` | No | Fetch a single job by ID |
| `POST` | `/jobs/{job_id}/bookmark` | Yes | Bookmark a job |
| `GET` | `/users/me/bookmarks` | Yes | List the current user's bookmarked jobs |
| `GET` | `/admin/sync-status` | Yes | Last sync run time and job counts per source |

A ready-to-import Postman collection covering every endpoint above (including auth token handling) lives at `backend/postman/JobScripts.postman_collection.json`.

---

## Project Structure

```
job-scripts/
├── backend/
│   ├── app/
│   │   ├── models/          # SQLAlchemy models (User, Job, Bookmark, SyncRun)
│   │   ├── schemas/         # Pydantic request/response schemas
│   │   ├── routers/         # auth, jobs, search, bookmarks, admin
│   │   ├── ingestion/       # RemoteOK + Adzuna fetchers and normalizers
│   │   ├── search/          # OpenSearch client, index mapping, indexer
│   │   ├── db/               # Postgres + MongoDB connection setup
│   │   ├── auth/             # password hashing, JWT creation/validation
│   │   └── scheduler.py      # APScheduler job registration
│   ├── tests/                # pytest suite
│   ├── postman/               # exported Postman collection
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── components/       # SearchBar, JobCard, FilterPanel, BookmarkButton
│   │   ├── pages/             # Home, Saved
│   │   └── api/                # API client
│   └── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## Getting Started

### Prerequisites
- Docker Desktop (or Docker Engine + Compose plugin)
- An Adzuna developer account (free) — [developer.adzuna.com](https://developer.adzuna.com/)

### Run locally

```bash
git clone (https://github.com/shariftasrik/JobScripts.git)
cd job-scripts
cp backend/.env.example backend/.env   # add your Adzuna app_id / app_key
docker compose up --build
```

Once running:
- API: `http://localhost:8000`
- Swagger docs: `http://localhost:8000/docs`
- Frontend: `http://localhost:5173`

### Run tests

```bash
cd backend
pytest -v
```

---

## Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `MONGO_URL` | MongoDB connection string |
| `OPENSEARCH_URL` | OpenSearch host |
| `JWT_SECRET` | Secret key used to sign access tokens |
| `ADZUNA_APP_ID` / `ADZUNA_APP_KEY` | Credentials from developer.adzuna.com |
| `SYNC_INTERVAL_HOURS` | How often the ingestion pipeline runs |

---

## Roadmap

- [ ] Click-through tracking (`POST /jobs/{id}/click`) to surface most-clicked listings on the admin dashboard
- [ ] Additional job sources (e.g. USAJobs, GitHub Jobs-style feeds)
- [ ] Saved search alerts (email/notification when a new job matches a saved filter)
- [ ] Horizontal scaling notes: separating the scheduler into its own worker process, sharding the OpenSearch index by region

---

## License

MIT

---

**Built by Nimur Rahman Sharif (Tasrik) - The Logits ** — this project was built specifically to explore multi-database architecture, third-party API integration, and search infrastructure in a single, coherent system.
