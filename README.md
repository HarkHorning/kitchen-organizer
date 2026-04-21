# Kitchen Inventory Tracker + Recipe Recommender

A cloud-based web app that tracks your kitchen inventory and suggests recipes based on what you have on hand.

---

## Problem Statement

Managing a large kitchen inventory manually is tedious. Users need a fast, low-friction way to add and maintain items so the app can surface relevant recipe suggestions — not just recipes they're missing half the ingredients for.

---

## MVP Scope

### Core Features

1. **Inventory Management**
   - Add, edit, and remove ingredients (name, quantity, unit, expiry date)
   - Mark items as "running low" or "out of stock"
   - Simple category grouping (produce, dairy, pantry, protein, etc.)

2. **Recipe Recommendations**
   - Suggest recipes based on available inventory
   - Rank by "% of ingredients you already have"
   - Filter: "I can make this now" vs. "missing 1–2 items"
   - Link out to full recipe (via Spoonacular API)

3. **User Accounts**
   - Sign up / login (email + password, Google OAuth)
   - Per-user inventory persisted in PostgreSQL
   - Accessible from any device

---

## Solving the Large Inventory Input Problem

The MVP offers **three input modes** to cover different user types:

### 1. Barcode Scanning (fastest for packaged goods)
- Device camera via a JS barcode library (e.g. `ZXing`)
- Flask endpoint looks up barcode against Open Food Facts API, returns product data
- User confirms item and sets quantity
- Best for: pantry staples, canned goods, packaged items

### 2. Search + Autocomplete (fastest for fresh/bulk items)
- Type-ahead hits a Flask `/ingredients/search` endpoint backed by the ingredients table
- Select item, set quantity and unit
- Best for: fruits, vegetables, spices, bulk dry goods

### 3. CSV Bulk Import (best for initial setup)
- POST to `/inventory/import`, Flask parses and bulk-inserts via SQLAlchemy
- Provide a downloadable CSV template
- Best for: first-time setup, migrating from a spreadsheet

> **Recommendation:** Lead with barcode + search. Add CSV import early — it kills the cold-start problem for large pantries.

---

## Recommended Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Frontend | **AI-generated** (v0.dev / Lovable / Bolt) | We have better things to do |
| Backend | **Flask** (Python) | Lightweight, fast to iterate, great ecosystem |
| ORM | **SQLAlchemy** + **Flask-Migrate** | Clean models, Alembic migrations |
| Database | **PostgreSQL** | Solid relational DB, runs great in Docker |
| Auth | **Flask-JWT-Extended** | Stateless JWT tokens, works well with any frontend |
| Recipe API | **Spoonacular API** | Ingredient-to-recipe matching, free tier available |
| Barcode Lookup | **Open Food Facts API** | Free, no key needed, large product database |
| Testing | **pytest** + **pytest-flask** + **factory_boy** | AI writes these — we just run them |
| Hosting | **Fly.io** or **Railway** | Docker-native, easier than AWS for a Flask + Postgres stack |
| Containerization | **Docker + Docker Compose** | Consistent environments across dev/prod |
| Build Automation | **Makefile** | Single entry point for all lifecycle commands |

---

## Containerization Strategy

All services run in Docker containers. Docker Compose orchestrates them per environment.

### Project Structure

```
kitchen/
├── Makefile
├── docker-compose.yml          # shared base
├── docker-compose.dev.yml      # dev overrides (debug, hot reload)
├── docker-compose.prod.yml     # prod overrides (gunicorn, no mounts)
├── .env.dev
├── .env.prod
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app/
│   │   ├── __init__.py
│   │   ├── models/
│   │   ├── routes/
│   │   └── services/
│   └── tests/
└── frontend/                   # AI-generated, drop it in here
```

### backend/Dockerfile (multi-stage)

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# --- dev stage ---
FROM base AS dev
RUN pip install watchdog pytest pytest-flask factory_boy
COPY . .
CMD ["flask", "--app", "app", "run", "--host=0.0.0.0", "--port=5000", "--debug"]

# --- prod stage ---
FROM base AS prod
COPY . .
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:create_app()"]
```

### docker-compose.yml (base)

```yaml
services:
  db:
    image: postgres:16-alpine
    env_file: .env.${ENV}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 5s
      retries: 5

  backend:
    build:
      context: ./backend
    env_file: .env.${ENV}
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres_data:
```

### docker-compose.dev.yml

```yaml
services:
  backend:
    build:
      target: dev
    volumes:
      - ./backend:/app     # hot reload
    environment:
      - FLASK_ENV=development
```

### docker-compose.prod.yml

```yaml
services:
  backend:
    build:
      target: prod
    restart: always
    environment:
      - FLASK_ENV=production
```

### .env.dev (example — never commit real secrets)

```
POSTGRES_USER=kitchen
POSTGRES_PASSWORD=kitchen
POSTGRES_DB=kitchen_dev
DATABASE_URL=postgresql://kitchen:kitchen@db:5432/kitchen_dev
JWT_SECRET_KEY=dev-secret-change-me
SPOONACULAR_API_KEY=your_key_here
FLASK_APP=app
```

---

## Makefile

```makefile
ENV ?= dev

.PHONY: up down build logs shell test migrate seed clean

up:
	ENV=$(ENV) docker compose -f docker-compose.yml -f docker-compose.$(ENV).yml up --build -d

down:
	ENV=$(ENV) docker compose -f docker-compose.yml -f docker-compose.$(ENV).yml down

build:
	ENV=$(ENV) docker compose -f docker-compose.yml -f docker-compose.$(ENV).yml build

logs:
	ENV=$(ENV) docker compose -f docker-compose.yml -f docker-compose.$(ENV).yml logs -f

shell:
	docker compose exec backend sh

## Run all tests inside the container
test:
	docker compose exec backend pytest tests/ -v

## Run DB migrations
migrate:
	docker compose exec backend flask db upgrade

## Seed the DB with sample data
seed:
	docker compose exec backend flask seed

clean:
	ENV=$(ENV) docker compose -f docker-compose.yml -f docker-compose.$(ENV).yml down -v --remove-orphans
```

### Usage

```bash
make up              # start dev stack
make up ENV=prod     # start prod stack
make test            # run all tests
make migrate         # apply DB migrations
make seed            # seed sample data
make logs            # tail logs
make shell           # shell into backend container
make clean           # full teardown + wipe volumes
```

---

## Database Schema (PostgreSQL)

```sql
-- Users
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT NOT NULL UNIQUE,
    password_hash TEXT,              -- null if OAuth only
    google_id   TEXT UNIQUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Canonical ingredient list (shared, seeded from Open Food Facts / Spoonacular)
CREATE TABLE ingredients (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    category    TEXT,                -- produce, dairy, pantry, protein, etc.
    barcode     TEXT UNIQUE
);

-- Per-user inventory
CREATE TABLE inventory_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    ingredient_id   INT NOT NULL REFERENCES ingredients(id),
    quantity        NUMERIC NOT NULL,
    unit            TEXT NOT NULL,   -- g, ml, count, etc.
    expiry_date     DATE,
    low_stock       BOOLEAN DEFAULT FALSE,
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Saved recipes
CREATE TABLE saved_recipes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    spoonacular_id  INT NOT NULL,
    title           TEXT NOT NULL,
    saved_at        TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (user_id, spoonacular_id)
);
```

---

## API Routes (Flask)

```
POST   /auth/register          → create account
POST   /auth/login             → return JWT
POST   /auth/google            → Google OAuth token exchange

GET    /inventory              → list user's inventory
POST   /inventory              → add item
PUT    /inventory/<id>         → update item
DELETE /inventory/<id>         → remove item
POST   /inventory/import       → CSV bulk import

GET    /ingredients/search?q=  → autocomplete search
GET    /ingredients/barcode/<code> → barcode lookup (proxies Open Food Facts)

GET    /recipes                → recommendations based on current inventory
GET    /recipes/<id>           → recipe detail (proxies Spoonacular)
POST   /recipes/<id>/save      → save recipe
DELETE /recipes/<id>/save      → unsave recipe
```

---

## Testing Strategy

> Tests are AI-generated. Run them, don't write them.

Use **Claude** or **Copilot** to generate test files. Prompt pattern:
```
Given this Flask route/model/service, write pytest tests covering:
happy path, auth failure, missing fields, DB constraint violations.
Use factory_boy for fixtures.
```

### Test layout

```
backend/tests/
├── conftest.py          # app fixture, test DB setup
├── factories.py         # factory_boy model factories
├── test_auth.py
├── test_inventory.py
├── test_ingredients.py
└── test_recipes.py
```

### Test rules
- Tests run against a real PostgreSQL test DB (spun up in Docker) — no mocking the DB
- Each test gets a clean DB state via transactions that roll back after each test
- `make test` must pass before any PR merges

---

## Recipe Recommendation Logic (MVP)

1. `GET /recipes` fetches the calling user's inventory ingredient names
2. Calls Spoonacular `/recipes/findByIngredients` with those names
3. Results cached in memory (or Redis post-MVP) to stay within free tier (150 req/day)
4. Returns two buckets:
   - **make_now** — `missedIngredientCount == 0`
   - **almost_there** — `missedIngredientCount <= 2`

---

## MVP Pages (Frontend — AI handles this)

```
/                  → Landing / sign in
/dashboard         → Inventory overview + quick-add
/inventory         → Full list, search, filter, bulk edit
/inventory/scan    → Barcode scanner
/recipes           → Recommendations
/recipes/[id]      → Recipe detail
/settings          → Account, units, export
```

Recommended AI frontend tools: **v0.dev** (Vercel), **Lovable**, or **Bolt.new**. Feed them the API route list and let them generate the UI. Drop the output into `/frontend`.

---

## Project Timeline

Update status as work progresses.

**Status key:** `[ ]` Not started · `[~]` In progress · `[x]` Done · `[-]` Blocked

### Phase 1 — Foundation (Week 1)

| # | Task | Status | Target | Notes |
|---|---|---|---|---|
| 1.1 | Repo setup + branch strategy | `[ ]` | Week 1 | |
| 1.2 | Docker + Makefile scaffolding | `[ ]` | Week 1 | Dev and prod targets |
| 1.3 | Flask app factory pattern + PostgreSQL connected | `[ ]` | Week 1 | Verify `make up` works cleanly |
| 1.4 | Flask-Migrate initialized, base schema applied | `[ ]` | Week 1 | |
| 1.5 | Environment variable strategy (.env.dev / .env.prod) | `[ ]` | Week 1 | Never commit secrets |
| 1.6 | CI pipeline (lint + test on PR) | `[ ]` | Week 1 | GitHub Actions |

### Phase 2 — Auth + Inventory Core (Week 2–3)

| # | Task | Status | Target | Notes |
|---|---|---|---|---|
| 2.1 | User registration + login (JWT) | `[ ]` | Week 2 | |
| 2.2 | Google OAuth token exchange endpoint | `[ ]` | Week 2 | |
| 2.3 | Inventory CRUD routes | `[ ]` | Week 2 | |
| 2.4 | Ingredient search endpoint (autocomplete) | `[ ]` | Week 3 | |
| 2.5 | Seed ingredient list from Spoonacular or static file | `[ ]` | Week 3 | |
| 2.6 | AI-generated tests for auth + inventory | `[ ]` | Week 3 | `make test` must pass |

### Phase 3 — Advanced Input (Week 4)

| # | Task | Status | Target | Notes |
|---|---|---|---|---|
| 3.1 | Barcode lookup endpoint (Open Food Facts proxy) | `[ ]` | Week 4 | |
| 3.2 | CSV import endpoint | `[ ]` | Week 4 | Validate + bulk insert |
| 3.3 | AI-generated tests for import + barcode | `[ ]` | Week 4 | |

### Phase 4 — Recipe Recommendations (Week 5)

| # | Task | Status | Target | Notes |
|---|---|---|---|---|
| 4.1 | Spoonacular integration + response caching | `[ ]` | Week 5 | Stay within free tier |
| 4.2 | Recipe recommendations endpoint | `[ ]` | Week 5 | make_now + almost_there buckets |
| 4.3 | Save / unsave recipe endpoints | `[ ]` | Week 5 | |
| 4.4 | AI-generated tests for recipe routes | `[ ]` | Week 5 | |

### Phase 5 — Frontend (Week 5–6)

| # | Task | Status | Target | Notes |
|---|---|---|---|---|
| 5.1 | Generate frontend with v0 / Lovable / Bolt | `[ ]` | Week 5 | Feed it the API route list |
| 5.2 | Wire frontend to Flask API | `[ ]` | Week 6 | Auth, inventory, recipes |
| 5.3 | Barcode scanner UI | `[ ]` | Week 6 | Mobile-first |
| 5.4 | CSV upload UI | `[ ]` | Week 6 | |

### Phase 6 — Polish + Launch (Week 7–8)

| # | Task | Status | Target | Notes |
|---|---|---|---|---|
| 6.1 | Low stock indicators + expiry warnings | `[ ]` | Week 7 | |
| 6.2 | Mobile responsiveness pass | `[ ]` | Week 7 | |
| 6.3 | Error states + loading states | `[ ]` | Week 7 | |
| 6.4 | End-to-end smoke test | `[ ]` | Week 8 | |
| 6.5 | Deploy to Fly.io / Railway | `[ ]` | Week 8 | |
| 6.6 | MVP launch | `[ ]` | Week 8 | |

---

## Sprint Plan

### Summary

| Sprint | Weeks | Goal | Deliverable |
|---|---|---|---|
| 1 | 1 | Skeleton works end-to-end | `make up` brings up Flask + Postgres; CI runs on every PR |
| 2 | 2–3 | Users can manage their inventory | Auth, inventory CRUD, ingredient search — all tested |
| 3 | 4 | Inventory input is fast even at scale | Barcode lookup + CSV import — all tested |
| 4 | 5 | Core value prop is live | Recipe recommendations returned from real API — all tested |
| 5 | 5–6 | Something a human can click on | AI-generated frontend wired to the Flask API |
| 6 | 7–8 | Shippable | Hardened, deployed, smoke-tested — MVP live |

---

### Sprint 1 — Skeleton (Week 1)

**Goal:** The project structure is in place and every developer can get a running stack with one command.

**Why this first:** Nothing else can happen until `make up` works and CI exists. Getting this right early prevents environment drift and "works on my machine" forever.

**Deliverables:**
- `make up` boots Flask + PostgreSQL with no manual steps
- `make test` runs (even if there's nothing to test yet — the harness works)
- Base DB schema applied via `flask db upgrade`
- GitHub Actions runs lint + tests on every PR
- `.env.dev` / `.env.prod` strategy documented, no secrets committed

**Definition of Done:** A new contributor can clone the repo, run `make up && make migrate`, and hit `GET /health → 200` with zero additional setup.

---

### Sprint 2 — Auth + Inventory Core (Weeks 2–3)

**Goal:** A user can create an account, log in, and fully manage their ingredient inventory via the API.

**Why this matters:** This is the load-bearing foundation. Every other feature (recipes, barcode, CSV) bolts onto an authenticated inventory. Getting auth and CRUD solid — and tested — before moving on avoids compounding bugs.

**Deliverables:**
- `POST /auth/register` and `POST /auth/login` return valid JWTs
- Google OAuth endpoint working
- Full inventory CRUD: `GET`, `POST`, `PUT`, `DELETE /inventory`
- Ingredient autocomplete: `GET /ingredients/search?q=`
- Ingredient seed data loaded via `make seed`
- AI-generated pytest suite covering: happy path, bad credentials, missing fields, ownership checks (user A can't touch user B's inventory)

**Definition of Done:** `make test` passes. All routes return correct status codes for both valid and invalid inputs. No route is reachable without a valid JWT.

---

### Sprint 3 — Advanced Input (Week 4)

**Goal:** Adding a large inventory isn't painful. Users can scan a barcode or drop a CSV and be done.

**Why this matters:** This is the key UX differentiator. Without it, the app is just another list. With it, a user with 200 pantry items can be onboarded in minutes, not an hour.

**Deliverables:**
- `GET /ingredients/barcode/<code>` proxies Open Food Facts and returns ingredient data
- `POST /inventory/import` accepts a CSV, validates rows, and bulk-inserts clean data
- Downloadable CSV template available (static file served by Flask)
- Graceful handling of: unknown barcodes, malformed CSVs, duplicate items
- AI-generated tests for barcode lookup (hit/miss), CSV import (valid, bad format, duplicates)

**Definition of Done:** `make test` passes. A 200-row CSV imports without error. An unknown barcode returns a clean 404, not a 500.

---

### Sprint 4 — Recipe Recommendations (Week 5)

**Goal:** The app tells you what you can cook right now based on what's in your inventory.

**Why this matters:** This is the entire value proposition. Everything before this was infrastructure — this is the feature users actually want.

**Deliverables:**
- `GET /recipes` calls Spoonacular with the user's inventory items and returns `make_now` and `almost_there` buckets
- Response caching in-process (avoid burning Spoonacular's 150 req/day free tier)
- `POST /recipes/<id>/save` and `DELETE /recipes/<id>/save` working
- `GET /recipes/<id>` returns full recipe detail (proxied from Spoonacular)
- AI-generated tests for: recommendation bucketing logic, saving/unsaving, cache hits, Spoonacular error handling

**Definition of Done:** `make test` passes. A user with 10 inventory items gets a non-empty recipe list. Hitting `GET /recipes` twice doesn't make two Spoonacular calls.

---

### Sprint 5 — Frontend (Weeks 5–6)

**Goal:** There is a UI a human can use.

**Why AI handles this:** The API contract is fully defined and tested. Any competent AI tool (v0.dev, Lovable, Bolt.new) can generate a working React UI from the route list with less effort than writing it by hand. We review and wire it — we don't write components.

**Deliverables:**
- AI-generated frontend scaffolded and dropped into `/frontend`
- Auth flow working end-to-end (register, login, logout)
- Inventory list, add, edit, delete working against the live API
- Barcode scanner UI (camera access, confirm flow)
- CSV upload UI
- Recipe recommendations page with both tabs

**How to generate the frontend:**
1. Open v0.dev / Lovable / Bolt
2. Paste the API Routes section from this README
3. Describe each page from the MVP Pages section
4. Download the output, drop in `/frontend`, adjust API base URL
5. Wire JWT storage (localStorage or httpOnly cookie — decide before generating)

**Definition of Done:** All six pages render. Auth, inventory, and recipe flows work against the real Flask API running locally. No hardcoded data.

---

### Sprint 6 — Polish + Launch (Weeks 7–8)

**Goal:** The app is hardened, deployed, and something we'd let a stranger use.

**Why a dedicated polish sprint:** Rough edges compound. A missing loading state looks like a bug. An unhandled 401 looks like data loss. This sprint pays that debt before anyone outside the team touches it.

**Deliverables:**
- Low stock indicators and expiry warnings surfaced in the UI
- All API error states handled gracefully in the frontend (no blank screens)
- Loading skeletons on async routes
- Mobile responsiveness verified (inventory and scanner especially)
- Full end-to-end smoke test (new user → add items → get recipes)
- App deployed to Fly.io or Railway via `make up ENV=prod`
- Production environment variables set in hosting dashboard (not in repo)

**Definition of Done:** A fresh user account can complete the full flow on a mobile browser without hitting an unhandled error. `make test` passes against the prod DB schema.

---

## Out of Scope for MVP

- Meal planning calendar
- Shopping list generation (post-MVP quick win)
- Nutritional tracking
- Multi-user households / shared pantries
- Offline mode
- Native mobile app
- Redis caching (use in-process cache for MVP)

---

## Open Questions

- Fly.io vs Railway for hosting? (Both are Docker-native and cheap for MVP scale)
- Do we want quantity-aware recipe matching? (e.g. "you have 50g flour, recipe needs 200g")
- Staging environment between dev and prod?
- Spoonacular at scale — do we cache results in the DB to avoid hitting rate limits?
