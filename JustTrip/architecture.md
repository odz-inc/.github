JustTrip - Architecture Overview

**MVP travel application "One-Flow Planner"**

## Project Structure

```
travelai/
├── api/                # FastAPI backend
│   └── src/
│       ├── main.py
│       ├── routers/        # plans.py, users.py
│       └── services/
│       └── firebase_auth.py
│
├── engine/             # AI Engine (Vertex AI)
│   └── src/
│       ├── worker.py           # Arq worker (async queue)
│       ├── embeddings.py       # Vertex AI embeddings (768-dim)
│       ├── planner.py          # Gemini trip planning + retry
│       ├── planner_smart.py    # Smart planning with availability
│       ├── planner_cached.py   # Cached plan generation
│       ├── scoring.py          # ML recommendation scoring
│       ├── tuning.py           # User preference learning
│       ├── vector_search.py    # pgvector semantic search
│       ├── weather.py          # Open-Meteo weather fallback
│       ├── cache.py            # Redis plan + embedding cache
│       ├── smart_cache.py      # Similar-user plan reuse
│       ├── mcp_client.py       # MCP server interaction
│       └── trends.py           # Trend tracking
│
├── mcp_server/         # MCP tools server (port 5001)
│   └── src/
│       ├── server.py
│       └── tools/
│           ├── weather.py      # Weather forecast (Open-Meteo)
│           ├── packing.py      # Packing suggestions (15+ countries)
│           ├── transport.py    # Transport tools
│           └── visa_rules.json # Visa/document rules (20+ pairs)
│
├── db/                 # Database configuration
│   ├── database.py             # Async engine + session
│   ├── init/                   # DB init scripts (extensions)
│   └── migrations/             # Alembic
│
├── shared/             # Shared modules
│   ├── models/
│   │   └── models.py           # User, Place, TripPlan, etc.
│   ├── schemas.py              # Pydantic validation
│   └── config.py               # Settings
│
├── tests/              # pytest suite (scoring, tuning, planner, API)
├── deploy/             # GCP deployment scripts + guides
├── docker_compose.yaml # Full local stack (DB, Redis, API, Engine, MCP)
├── cloudbuild.yaml     # GCP Cloud Build pipeline
├── alembic.ini         # Migrations config
└── Makefile            # Dev commands
```

## Separation of Concerns

| Module | Role | Contains | Vertex AI? |
|--------|------|----------|------------|
| **db/** | Database config | Engine, session, migrations | No |
| **shared/models/** | Data models | User, Place, TripPlan, UserFeedback | No |
| **engine/** | AI Engine | Vertex AI, embeddings, planning, scoring, caching, ML | Yes |
| **api/** | REST API | FastAPI endpoints, auth | No |
| **mcp_server/** | MCP Tools | Weather, packing, transport, visa rules | No |
| **shared/** | Common | Config, schemas, utils | No |

## Tech Stack

### Backend
- **Framework**: FastAPI (async)
- **ORM**: SQLModel (SQLAlchemy + Pydantic)
- **Database**: PostgreSQL 15+ + pgvector
- **Queue**: Redis + Arq (async worker)
- **MCP Server**: Weather, packing, transport tools (port 5001)

### AI
- **Provider**: Google Vertex AI
- **Embeddings**: text-embedding-004 (768-dim)
- **LLM**: Gemini 2.5 Flash (cost-optimized)

### Infrastructure
- **Container**: Docker + docker-compose
- **Deploy**: Google Cloud Run + Cloud Build
- **CI/CD**: GitHub Actions (lint, test, deploy)
- **Migrations**: Alembic

## Data Flow

### 1. Trip Generation Request
```
User → API → Redis Queue → Engine Worker
                              ↓
                      Vertex AI (embeddings)
                              ↓
                      pgvector (search) + scoring
                              ↓
                      Weather check (MCP / Open-Meteo)
                              ↓
                      Smart cache lookup
                              ↓
                      Gemini 2.5 Flash (planning)
                              ↓
                      Packing list (MCP)
                              ↓
                      PostgreSQL (save)
                              ↓
User ← API ← PostgreSQL
```

### 2. User Feedback (Tuning)
```
User edits plan → API → UserFeedback table
                          ↓
                    Engine (background)
                          ↓
                  Analyze patterns
                          ↓
                  Update user.preference_embedding
                          ↓
                  Better recommendations next time!
```

## 🗄️ Database Schema

### Core Tables

**users**
- `id`, `firebase_uid`, `email`
- **`preference_embedding`** (vector 768) ← Learned from feedback
- `preferences_metadata` (JSONB)

**places**
- `id`, `name`, `place_type`, `city`, `country`
- **`embedding`** (vector 768) ← For semantic search
- `tags`, `metadata` (JSONB)
- `popularity_score`, `rating`

**trip_plans**
- `id`, `user_id`, `destination`, `days`
- `status` (draft/generated/edited/confirmed)
- `tuning_params` (JSONB)

**trip_items**
- `id`, `trip_plan_id`, `place_id`
- `day_number`, `order_in_day`
- `start_time`, `end_time`, `ai_notes`

**user_feedbacks** ⭐ **KEY TABLE**
- `id`, `user_id`, `trip_plan_id`
- `action` (removed/added/replaced/reordered)
- **`place_id_old`, `place_id_new`** ← What changed
- `context` (JSONB) ← Why/when/what

## Key Features

### 1. Semantic Search (pgvector)
```python
from engine.src import find_places_by_text_query

places = await find_places_by_text_query(
    session, "romantic dinner with city views", "Paris"
)
```

### 2. Personalization + ML Scoring
```python
from engine.src import find_personalized_places

places = await find_personalized_places(session, user_id, "Tokyo")
# Uses user.preference_embedding + collaborative filtering
```

### 3. Smart AI Planning
```python
from engine.src import generate_trip_plan

plan = await generate_trip_plan(
    destination="Paris", days=3,
    pre_selected_places=places,
    tuning=TripTuning(preferred_style="luxury", excluded_ids=[])
)
```

### 4. Smart Caching
```python
from engine.src.smart_cache import find_similar_cached_plan

# Reuse plans from similar users to reduce AI calls
cached = await find_similar_cached_plan(session, user_id, destination)
```

### 5. Weather-Aware Planning
```python
from engine.src.weather import get_weather_forecast

# Boosts indoor/outdoor places based on forecast
forecast = await get_weather_forecast(destination, dates)
```

### 6. MCP Tools
```python
# Packing lists, weather, transport, visa rules
# Called via mcp_server (port 5001) or engine/src/mcp_client.py
```

### 7. ML Tuning (Learning)
```python
from engine.src import update_user_preferences

await update_user_preferences(session, user_id)
# Analyzes feedback → updates preference_embedding
```

### 8. Alternatives (Plan B)
```python
from engine.src import generate_alternative_plan

alt_plan = await generate_alternative_plan(
    original_plan_id=plan.id, destination="Paris", days=3,
    tuning=same_tuning, places_to_exclude=[...]
)
```

## Security & Authorization

### Auth Architecture

```
Flutter App
    │
    ├─ Firebase SDK (Google Sign-In / email+password)
    │       ↓
    │  Firebase Auth servers (Google-managed)
    │       ↓ issues short-lived ID token (JWT, ~1h)
    │
    ├─ POST /api/v1/auth/firebase  { id_token }
    │       ↓
    │  Firebase Admin SDK  →  auth.verify_id_token()
    │       ↓ decoded payload: uid, email, name
    │  get_or_create_user()  →  PostgreSQL users table
    │       ↓ returns user_id
    │
    └─ All subsequent requests: Authorization: Bearer <firebase_id_token>
            ↓
       get_current_user() dependency — verifies token on every request
```

### 1. Firebase Identity Verification (`api/src/firebase_auth.py`)

| What | Detail |
|------|--------|
| Token type | Firebase ID token (RS256 JWT signed by Google) |
| Verification | `firebase_admin.auth.verify_id_token()` server-side — no secret shared with client |
| Clock skew | `clock_skew_seconds=10` — tolerates minor server clock drift |
| Error handling | Distinct 401 for: expired / invalid / revoked tokens |
| User mapping | `firebase_uid` → internal `users.id`; auto-creates row on first login |
| Profile sync | Email + display name updated from token on each login (handles Google account link) |

**Local dev mode** (`IS_LOCAL=true`): Firebase bypassed, static token `123456789` required — endpoint `POST /auth/login` available only in local mode.

### 2. Firebase App Check (`api/src/app_check.py`)

Validates `X-Firebase-AppCheck` header to prove request comes from genuine app binary.

| Platform | Attestation provider |
|----------|---------------------|
| Android | Play Integrity API — hardware-attested, APK signature verified against Play |
| iOS | Apple App Attest — Secure Enclave hardware attestation |
| Debug | Static debug token registered manually in Firebase Console |

**Current enforcement: SOFT** (`_ENFORCE = False`) — invalid/missing tokens are logged, not blocked. Switch `_ENFORCE = True` once all clients are updated.

Token is device-bound and generated dynamically by hardware — extracting it from the APK is useless.

### 3. Rate Limiting (`api/src/rate_limiter.py`)

All limits backed by Redis with automatic key expiry.

#### Login Rate Limit

| Parameter | Value |
|-----------|-------|
| Key | `rl:login:{X-Device-ID}` or `rl:login:{client-ip}` |
| Limit | 10 attempts per 15-minute window |
| Response | `429 Too Many Requests` + `Retry-After` header |
| Identifier | X-Device-ID header preferred over IP (handles NAT/proxies) |

Applied to: `POST /auth/firebase`

#### Plan Creation Rate Limit (Tier-based)

| Tier | Plans/day | Max trip days |
|------|-----------|---------------|
| `free` | 1 | 3 |
| `premium` | 3 | 7 |
| `ultimate` | 10 | 14 |

- Key: `rl:plans:{user_id}:{YYYY-MM-DD}` (UTC day window, expires at midnight UTC)
- Days exceeded → `402 Payment Required` with `TIER_DAYS_EXCEEDED` code
- Count exceeded → `429 Too Many Requests` with `PLAN_DAILY_LIMIT_EXCEEDED` code + `Retry-After`

### 4. Security Flow per Request

```
Request
  │
  ├─ App Check (soft)  →  verify X-Firebase-AppCheck header
  │
  ├─ Rate Limit        →  check Redis counter (login / plan endpoints)
  │
  ├─ Auth              →  Depends(get_current_user)
  │                         verify Firebase ID token
  │                         decode uid + email
  │
  ├─ DB User           →  get_or_create_user()
  │                         map firebase_uid → User row
  │
  └─ Business Logic    →  user.id + user.tier passed to engine
```

### 5. What Firebase Handles (vs. us)

| Responsibility | Firebase | Our API |
|----------------|----------|---------|
| Password hashing | ✓ | — |
| OAuth (Google, Apple) | ✓ | — |
| Token signing + rotation | ✓ | — |
| Email verification | ✓ | — |
| Account lockout (brute force) | ✓ | — |
| Login rate limiting | — | ✓ Redis |
| Plan rate limiting | — | ✓ Redis + tier |
| App integrity (device) | ✓ App Check | soft verify |
| User identity in DB | — | ✓ firebase_uid FK |

## 🔍 Monitoring

### Costs
- **Embeddings**: $0.025 per 1K requests
- **Gemini**: $0.075 per 1M input tokens
- **Cloud Run**: Pay per request
- **Cloud SQL**: ~$10-50/month

### Performance
- Embedding generation: ~200ms
- Trip planning: ~2-5s
- Vector search: <100ms (with index)