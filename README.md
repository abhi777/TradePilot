# TradePilot AI

An AI-powered business-development and procurement agent for B2B distributors and
industrial suppliers. TradePilot discovers and qualifies prospects, researches
them with grounded AI, prepares approval-gated outreach, tracks a lightweight
sales pipeline, and manages supplier/RFQ procurement — all scoped to a secure,
multi-tenant SaaS architecture.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Features](#features)
- [Screenshots / Demo](#screenshots--demo)
- [Technology Stack](#technology-stack)
- [Architecture & How It Works](#architecture--how-it-works)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
- [Environment Variables & Configuration](#environment-variables--configuration)
- [Usage Guide](#usage-guide)
- [API & Integration Reference](#api--integration-reference)
- [Database & Data Flow](#database--data-flow)
- [Development & Testing](#development--testing)
- [Deployment](#deployment)
- [Security & Privacy](#security--privacy)
- [Troubleshooting & FAQ](#troubleshooting--faq)
- [Limitations & Roadmap](#limitations--roadmap)
- [Contributing](#contributing)
- [License & Acknowledgements](#license--acknowledgements)
- [Additional Resources](#additional-resources)

---

## Project Overview

TradePilot is a vertical SaaS platform for **B2B distributors and industrial
suppliers** (and the sales/business-development teams inside them). It automates
the manual, low-leverage parts of outbound business development:

- **Discovery** — turn an Ideal Customer Profile (ICP) into a ranked list of
  real prospects using Google Maps data.
- **Qualification** — deterministic, explainable lead scoring against the ICP.
- **Research** — AI-generated research cards grounded in the prospect's own
  website and listing data (no hallucinated facts).
- **Outreach prep** — approval-gated email, WhatsApp, and cold-call scripts the
  user must explicitly approve; nothing is sent automatically.
- **Pipeline & follow-ups** — a lightweight Kanban pipeline plus calendar-based
  follow-up scheduling.
- **Procurement** — supplier management, RFQs, and quote comparison.

The system is designed as a **multi-tenant SaaS**: every domain entity is scoped
to an `Organization`, authentication is delegated to Clerk, and usage is governed
by subscription plans with enforced quotas.

### Intended audience

- B2B distributors, industrial suppliers, and trading companies.
- Founders and sales/business-development teams that prospect manually today.
- (Architecturally) developers looking at a production-oriented, multi-tenant
  FastAPI + Next.js + ARQ reference implementation.

### Key differentiators

- **Grounded AI** — research and outreach are generated from fetched prospect
  website content and structured Maps data, not from generic knowledge.
- **Approval-gated outreach** — deterministic compliance rules (scoring, gates,
  approval state, send-rules) live in code, not in the LLM.
- **Tenant isolation by construction** — `organization_id` scoping on every
  domain query, org-scoped quotas, org-scoped rate limits, and org-scoped cache
  keys.
- **Pluggable discovery** — a `DiscoveryProvider` abstraction with both a
  Google Places API backend and a Playwright/Chromium scraper backend.

### Current status

**Active development / pre-release.** The application is functional for local
development (all services start, migrations apply, the seed script provisions a
full demo environment), but the repository has not yet been tagged for a public
release. Some SaaS capabilities are implemented end-to-end while others are
modeled but not yet fully wired (see
[Limitations & Roadmap](#limitations--roadmap)).

---

## Features

### Implemented

| Area | What it does |
| --- | --- |
| **Multi-tenancy** | `Organization` (tenant), `User`, and `Membership` (role: `owner`/`admin`/`member`/`viewer`). Personal organizations are auto-created for org-less users so every row stays org-scoped. Suspended orgs are blocked. |
| **Clerk authentication** | Hosted identity; API verifies RS256 session JWTs against the instance JWKS. |
| **Platform admin** | `is_platform_admin` flag (from Clerk `public_metadata`) gates an `/admin` API (organizations, subscriptions, usage, stats, coupon generation) and the `/admin` UI. |
| **Subscription billing** | Seeded plans (Free/Starter/Growth), Dodo Payments checkout, customer portal, webhook-synced subscriptions, coupon-code redemption. |
| **Quota enforcement** | `require_quota()` enforces per-plan limits for `discovery_runs`, `ai_tokens`, and `leads` (calendar-month usage), returning HTTP 402 with an upgrade CTA when exceeded. |
| **Distributed rate limiting** | Redis-backed, multi-layer (global → IP → org → endpoint), fails open for general traffic and closed for expensive endpoints. |
| **Lead discovery** | ICP-driven query planning, two backends (Playwright scraper by default; Google Places API optional), dedup by `(organization_id, maps_url)` upsert. |
| **Lead qualification** | Deterministic, weighted scoring (geography, industry, contactability, web presence, reputation, signals). |
| **AI research** | Grounded research cards (what they do, likely need, fit, talking points, confidence) with per-generation token usage recorded. |
| **Outreach drafts** | Email, WhatsApp, and cold-call scripts (`English`/`Hinglish`), approval-gated, with regenerate-by-instruction. Call scripts use a multi-step pipeline with deterministic QA + LLM sales QA + correction. |
| **Location intelligence** | LLM-recommended customer/supplier markets for a lead (recommendations only — never auto-triggers searches). |
| **Lead-level discovery** | Run customer/supplier discovery scoped to a single parent lead with explicit search locations. |
| **Sales pipeline** | Lead stages `new → qualified → contacted → interested → meeting → quote → won` (terminal: `lost`, `rejected`); a sent draft auto-advances a lead to `contacted`. |
| **Follow-ups & calendar** | Scheduled follow-ups grouped by day; approving a draft with a days-ahead hint auto-creates a follow-up. |
| **Procurement / RFQ** | Supplier CRUD, RFQ lifecycle (`draft → sent → quoted → closed`), quote comparison. |
| **Analytics** | Funnel summary, draft/status breakdown, token usage + cost estimate (Redis-cached 60s), and per-plan quota usage. |
| **Export** | Leads export to CSV and Excel (openpyxl). |
| **PWA** | Installable web manifest, icons, standalone display. |
| **Observability** | Structured logging, request-ID tracing, `/health`, `/ready`, and a Prometheus-style `/metrics` endpoint; optional Sentry. |

### Partially implemented / not yet wired

These exist as models, tables, or UI but are **not** complete end-to-end:

- **Job tracking** — the `Job` model and `jobs` table exist and the frontend has a
  `/jobs` page, but there is **no `/api/v1/jobs` route** yet, so the page cannot
  load data.
- **Role-based authorization** — roles are stored on `Membership` and returned by
  `/auth/me`, but per-role endpoint enforcement (`require_role`) is not active;
  only the platform-admin gate (`require_admin`) is enforced today.
- **Audit log** — the `AuditLog` model/table exist but are not yet populated by
  middleware.
- **Idempotency keys** — the `IdempotencyKey` model/table exist but are not yet
  wired into request handling.

---

## Screenshots / Demo

No screenshots, GIFs, or public demo URLs are present in the repository.

<!-- TODO: add screenshots/demo. Suggested locations:
     - web/public/screenshots/*.png
     - a demo GIF of a discovery → research → outreach flow
     Then reference them here with relative links. -->

---

## Technology Stack

### Languages & runtime

| Technology | Role |
| --- | --- |
| Python 3.12 | Backend API, worker, integrations, scripts |
| TypeScript 5.8 | Frontend |
| Node.js 22 | Frontend build/runtime (Docker image) |

### Frontend & UI

| Technology | Role |
| --- | --- |
| Next.js 15 (App Router, Turbopack) | React framework, routing, SSR/SSG |
| React 19 | UI component layer |
| Tailwind CSS v4 | Styling |
| SWR | Client data fetching and caching |

### Backend

| Technology | Role |
| --- | --- |
| FastAPI | Async API framework (OpenAPI docs at `/docs`) |
| Uvicorn | ASGI server |
| Pydantic v2 + pydantic-settings | Request/response schemas and configuration |
| SQLAlchemy 2 (async) | ORM / data access |
| Alembic | Database migrations |
| asyncpg | PostgreSQL driver |

### Databases, storage & caching

| Technology | Role |
| --- | --- |
| PostgreSQL 16 | Source of truth for all persistent business state |
| Redis 7 | ARQ job queue, analytics cache, distributed rate limiting (AOF persistence enabled) |

### AI / LLM

| Technology | Role |
| --- | --- |
| DeepSeek (default) | LLM for research, outreach, call scripts, location intelligence |
| OpenAI-compatible client | `httpx`-based; swap models via `LLM_BASE_URL` / `LLM_MODEL` |

### APIs & third-party integrations

| Integration | Role |
| --- | --- |
| Clerk | Authentication (session JWTs, organization claims) |
| Dodo Payments | Subscription billing: checkout, portal, webhooks |
| Google Places API | Optional discovery provider (Text Search + Place Details) |
| Sentry | Optional error tracking |

### Automation & background processing

| Technology | Role |
| --- | --- |
| ARQ | Redis-based async task queue and worker |
| Playwright + Chromium | Browser-based Google Maps discovery (worker container) |

### Testing & code quality

| Technology | Role |
| --- | --- |
| pytest + pytest-asyncio | Backend tests |
| Vitest + Testing Library | Frontend tests |
| Ruff | Python lint + formatting |
| mypy | Python type checking |
| ESLint | Frontend lint |
| `tsc --noEmit` | Frontend type checking |

### Build, containers & infrastructure

| Technology | Role |
| --- | --- |
| Docker + Docker Compose | Local and production orchestration |
| GitHub Actions | CI (lint, typecheck, test, migrations, builds) |

---

## Architecture & How It Works

TradePilot is a classic three-tier, event-driven SaaS. State lives in PostgreSQL;
Redis is used only for ephemeral coordination (queue, cache, rate-limit counters).

```
            ┌──────────────────┐
            │   Next.js Web    │  :3000  (Clerk-protected UI, PWA)
            └────────┬─────────┘
                     │ HTTPS + Bearer (Clerk session JWT)
            ┌────────▼─────────┐
            │   FastAPI API    │  :8000  (16 routers under /api/v1)
            └───┬──────────┬───┘
                │          │
        SQL (asyncpg)      │ Redis (enqueue jobs / cache / rate limits)
                │          │
      ┌─────────▼───┐  ┌───▼────────────────────┐
      │ PostgreSQL 16│  │       Redis 7          │
      │ (source of  │  └───┬────────────────────┘
      │  truth)     │      │ ARQ dequeue
      └─────────────┘  ┌───▼──────────────┐
                       │   ARQ Worker      │  (max_jobs=4, timeout=2400s, tries=3)
                       └───┬──────────┬────┘
                           │          │
                    Playwright/Chromium  │  httpx → DeepSeek (LLM)
                    (Maps scraping)      │
                                   ┌─────▼──────┐
                                   │ PostgreSQL  │  (worker writes results)
                                   └────────────┘
```

### Components & responsibilities

- **`web/`** — Next.js app. Clerk middleware protects all authenticated routes.
  Pages call the API with a Clerk session token injected by
  `lib/api.ts` (`makeApiClient(getToken)`).
- **`api/`** — FastAPI service exposing REST routes under `/api/v1`. Each domain
  endpoint depends on `get_principal()` (verify JWT) → `get_membership()` /
  `get_org()` (resolve tenant) and, for mutations, `require_quota()`.
- **`worker/`** — a single ARQ worker process consuming jobs from Redis:
  `run_discovery_task`, `research_lead_task`, `draft_outreach_task`, plus a
  `recover_stale_runs` cron. The worker imports shared code from `api/app`.
- **PostgreSQL** — authoritative store; accessed only through org-scoped queries.
- **Redis** — ARQ queue (`arq:*`), analytics cache (`analytics:{org_id}`),
  rate-limit counters (`rl:*`).

### Authentication & authorization flow

1. Next.js `clerkMiddleware` protects `/dashboard`, `/onboarding`, `/leads`,
   `/pipeline`, `/calendar`, `/suppliers`, `/rfqs`, `/products`, `/settings`,
   `/usage`, `/jobs`, `/admin`.
2. The browser calls the API with `Authorization: Bearer <clerk session token>`.
3. FastAPI verifies the RS256 token against the instance JWKS (URL derived from
   the publishable key or set explicitly via `CLERK_JWKS_URL`).
4. `GET /api/v1/auth/me` upserts the `User` and resolves/creates the
   `Organization` + `Membership`. Org-less users get a personal org keyed
   `personal:<clerk_user_id>`.

### Background job flow (discovery)

```mermaid
sequenceDiagram
    participant U as User (Web)
    participant W as Next.js
    participant A as FastAPI
    participant P as PostgreSQL
    participant R as Redis
    participant K as ARQ Worker
    participant D as Playwright / Google Places

    U->>W: Start discovery (ICP + target)
    W->>A: POST /api/v1/discovery/runs
    A->>A: verify JWT, resolve org, require_quota(discovery_runs)
    A->>P: INSERT discovery_runs (status=queued)
    A->>R: enqueue run_discovery_task(run_id)
    A-->>W: 201 { status: "queued" }
    R-->>K: dequeue job
    K->>P: load run, set status=running
    K->>D: provider.search(query plan)
    D-->>K: candidate leads
    K->>P: UPSERT leads (scored, org-scoped)
    K->>P: set status=completed (+ stats)
    W->>A: poll GET /api/v1/discovery/runs/{id}
```

Research and outreach follow the same shape: the API enqueues
`research_lead_task` / `draft_outreach_task`, the worker loads the org-scoped
resource, calls the LLM, records a `UsageRecord`, and persists the
`ResearchSnapshot` / `OutreachDraft`. The frontend polls until results appear.
Drafts stay `draft` until a user approves them; approving with
`schedule_followup_days` auto-creates a `FollowUp`.

---

## Project Structure

```
TradePilot/
├── api/                              # FastAPI backend
│   ├── app/
│   │   ├── api/
│   │   │   ├── deps.py               # principal / membership / org resolution
│   │   │   ├── deps_quota.py         # plan-quota enforcement dependency
│   │   │   ├── deps_admin.py         # platform-admin guard
│   │   │   ├── middleware/           # access log, distributed rate limiting
│   │   │   └── routes/               # 16 routers (health, auth, admin, billing,
│   │   │                             #   company, products, icps, discovery, leads,
│   │   │                             #   research/outreach, analytics, events,
│   │   │                             #   follow-ups, procurement, export)
│   │   ├── core/                     # config, db, auth, redis, rate limiter,
│   │   │                             #   errors, logging
│   │   ├── integrations/             # LLM client, Dodo Payments,
│   │   │                             #   discovery providers, Maps scraper
│   │   ├── models/                   # SQLAlchemy ORM models (23 tables)
│   │   ├── schemas/                  # Pydantic request/response models
│   │   ├── services/                 # business logic (planner, qualification,
│   │   │                             #   research, outreach, call_script,
│   │   │                             #   location_intelligence, webtext, geo,
│   │   │                             #   sales_pipeline, security)
│   │   ├── tasks/                    # ARQ tasks + connection pool
│   │   ├── prompts/                  # LLM prompt templates + output schemas
│   │   └── scripts/                  # seed_dev and maintenance scripts
│   ├── alembic/                      # migrations (24 revisions)
│   ├── tests/                        # pytest suite (unit + security)
│   ├── requirements.txt
│   ├── pytest.ini / ruff.toml / mypy.ini
│   ├── start.py                      # wait-for-db + migrate + seed helper
│   └── Dockerfile
├── web/                              # Next.js frontend
│   ├── src/app/                      # pages (dashboard, leads, pipeline,
│   │   │                             #   calendar, products, suppliers, rfqs,
│   │   │                             #   settings, pricing, usage, jobs, admin…)
│   ├── src/components/               # UI kit + layout + discovery panel
│   ├── src/lib/                      # API client, SWR hooks, contexts
│   ├── src/__tests__/                # Vitest tests
│   ├── public/                       # PWA manifest + icons
│   ├── package.json
│   └── Dockerfile
├── worker/                           # ARQ worker entrypoint
│   ├── main.py                       # WorkerSettings, cron, Redis DSN
│   └── Dockerfile                    # Playwright/Python image
├── docs/                             # architecture, security, CI, guides
├── scripts/ci.sh                     # local CI runner
├── docker-compose.yml                # development stack
├── docker-compose.prod.yml           # production-oriented override
├── .env.example                      # environment template (committed)
├── start.sh / stop.sh                # convenience scripts
└── AGENTS.md
```

---

## Prerequisites

- **Docker** with **Docker Compose v2** (easiest path — everything runs in
  containers). Python and Node are only needed for host-based development.
- **Clerk account** (free tier) — required for authentication. Obtain
  `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` from the
  [Clerk dashboard](https://dashboard.clerk.com).

Optional, for full functionality:

- **DeepSeek API key** — enables AI research, outreach, call scripts, and
  location intelligence.
- **Google Maps (Places) API key** — enables the `google_places` discovery
  provider (otherwise the Playwright scraper is used).
- **Dodo Payments** credentials — enables real billing/checkout (otherwise the
  `dev` billing provider is used).

For host-based development (outside Docker):

- Python 3.12+, Node.js 22+, and a running PostgreSQL 16 + Redis 7.

---

## Installation & Setup

### 1. Clone the repository

```bash
git clone <repo-url>
cd <repo-name>
```

### 2. Create the environment file

```bash
cp .env.example .env
```

Edit `.env` and set at least your Clerk keys:

```
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
```

### 3. Start the stack

```bash
docker compose up --build
```

This starts PostgreSQL, Redis, the API, the worker, and the web app. The API
exposes:

- Web UI — http://127.0.0.1:3000
- API docs (Swagger UI) — http://127.0.0.1:8000/docs
- Health check — http://127.0.0.1:8000/api/v1/health
- Readiness (DB + Redis probes) — http://127.0.0.1:8000/api/v1/ready

> The `web` service also has its own `.env.local` (gitignored) for Clerk keys and
> `NEXT_PUBLIC_API_URL`. The Compose web build injects `NEXT_PUBLIC_API_URL` as a
> build arg (default `http://localhost:8000`).

### 4. Run migrations and seed data

Migrations are not run automatically by `docker compose up`. Apply them and
provision a demo environment:

```bash
docker compose exec api alembic upgrade head
docker compose exec api python -m app.scripts.seed_dev
```

`seed_dev` creates the three plans, a development organization, an owner user, an
active subscription on the Growth plan, a company profile, products, and ICPs.
(Refuses to run when `ENVIRONMENT=production`.)

### 5. Alternative: convenience scripts

```bash
./start.sh   # starts backend containers + web dev server (background)
./stop.sh    # stops web dev server + containers
```

---

## Environment Variables & Configuration

Configuration flows through `pydantic-settings` (`api/app/core/config.py`).
Environment profiles (`development`, `test`, `production`) apply different
defaults and startup validation.

### Required

| Variable | Purpose | Example |
| --- | --- | --- |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk publishable key (browser-safe) | `pk_test_...` |
| `CLERK_SECRET_KEY` | Clerk server secret (JWT verification) | `sk_test_...` |

In production, `DATABASE_URL` and `CLERK_SECRET_KEY` are validated at startup and
the app fails fast if missing.

### Optional (feature keys)

| Variable | Purpose | Default |
| --- | --- | --- |
| `DEEPSEEK_API_KEY` | LLM access (research/outreach/call scripts/location) | empty → AI features unavailable |
| `GOOGLE_MAPS_API_KEY` | Google Places discovery provider | empty → uses scraper |
| `DODO_API_KEY` / `DODO_WEBHOOK_SECRET` | Dodo Payments billing + webhook HMAC | empty → `dev` billing |
| `SENTRY_DSN` | Error tracking | empty → disabled |
| `CLERK_JWKS_URL` | Explicit JWKS URL override | derived from publishable key |

### With safe defaults

| Variable | Default | Notes |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql+asyncpg://tradepilot:tradepilot_dev@postgres:5432/tradepilot` | |
| `REDIS_URL` | `redis://redis:6379/0` | |
| `DISCOVERY_PROVIDER` | `scraper` | `scraper` or `google_places` |
| `BILLING_PROVIDER` | `dev` | `dev` or `dodo` |
| `LLM_MODEL` / `LLM_BASE_URL` | `deepseek-flash` / `https://api.deepseek.com` | any OpenAI-compatible endpoint |
| `CORS_ORIGINS` | localhost:3000 origins | JSON list |
| `RATE_LIMIT_PER_MINUTE` | `60` | per-IP default |
| `DB_POOL_SIZE` / `DB_MAX_OVERFLOW` | `5` / `10` | |
| `LOG_LEVEL` | `DEBUG` (dev), `INFO` (prod) | |

Docker Compose also reads `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`,
`POSTGRES_PORT`, and `REDIS_PORT` (host port mappings). In production, `REDIS_PASSWORD`
is required by `docker-compose.prod.yml`.

See `.env.example` for the complete, commented template.

> Never commit `.env` or `web/.env.local` — both are gitignored. `.env.example`
> contains only placeholders and safe local defaults.

---

## Usage Guide

### Typical workflow

1. **Onboard** — complete the company profile (name, products, target
   geographies) on `/onboarding`.
2. **Define an ICP** — industries, business types, geographies, and qualification
   gates on `/products` → ICP settings.
3. **Run discovery** — on `/leads`, start a discovery run; the worker plans
   `{industry} in {city}` queries, discovers, scores, and stores leads.
4. **Research a lead** — trigger AI research; a grounded research card appears
   (website text + listing data).
5. **Generate outreach** — choose a channel (`email`, `whatsapp`, `call`) and
   language (`English` / `Hinglish`); review, edit, and approve the draft.
6. **Track** — move leads through `/pipeline`; schedule follow-ups on `/calendar`.
7. **Procure** — manage `/suppliers`, create `/rfqs`, compare quotes.
8. **Monitor** — `/dashboard` (analytics), `/usage` (quota), `/settings/billing`
   (plan), `/admin` (platform admins only).

### API examples (authenticated)

Most endpoints require `Authorization: Bearer <clerk session token>`. Public
endpoints include `/health`, `/ready`, `/metrics`, and `/billing/plans`.

```bash
# List plans (public)
curl http://127.0.0.1:8000/api/v1/billing/plans

# Current user + membership (authenticated)
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/v1/auth/me

# List leads (authenticated, org-scoped)
curl -H "Authorization: Bearer $TOKEN" "http://127.0.0.1:8000/api/v1/leads?status=qualified&limit=20"
```

Interactive documentation for every endpoint is available at
http://127.0.0.1:8000/docs and http://127.0.0.1:8000/redoc.

---

## API & Integration Reference

All routes are prefixed `/api/v1`. Every domain endpoint requires
`OrgDep + DBSession`; mutations additionally enforce `require_quota()`.

| Group | Prefix | Key endpoints |
| --- | --- | --- |
| System | `/` | `GET /health`, `GET /ready`, `GET /metrics` |
| Auth | `/auth` | `GET /auth/me` |
| Company | `/company` | `GET/PUT /company` |
| Products | `/products` | `GET/POST /products`, `PATCH/DELETE /products/{id}` |
| ICPs | `/icps` | `GET/POST /icps`, `PATCH/DELETE /icps/{id}` |
| Discovery | `/discovery` | `POST/GET /discovery/runs`, `GET /discovery/runs/{id}` |
| Leads | `/leads` | `GET/DELETE /leads`, `GET/PATCH /leads/{id}`, `POST /leads/{id}/location-recommendations`, `POST /leads/{id}/discover`, `GET /leads/{id}/discovered` |
| Research/Outreach | `/leads`, `/outreach` | `POST/GET /leads/{id}/research`, `POST/GET /leads/{id}/outreach`, `POST /leads/{id}/outreach/regenerate`, `PATCH /outreach/{draft_id}`, `PATCH /outreach/{draft_id}/body` |
| Analytics | `/analytics` | `GET /analytics/summary`, `GET /analytics/quota` |
| Events | `/events` | `POST /events` |
| Follow-ups | `/follow-ups` | `GET/POST /follow-ups`, `GET /follow-ups/calendar`, `PATCH/DELETE /follow-ups/{id}` |
| Procurement | `/procurement` | CRUD for `suppliers`, `rfqs`, `quotes`; `GET /procurement/quotes/compare` |
| Export | `/leads` | `GET /leads/export/csv`, `GET /leads/export/excel` |
| Billing | `/billing` | `GET /billing/plans`, `GET /billing/subscription`, `POST /billing/checkout`, `GET /billing/portal`, `POST /billing/redeem-coupon`, `POST /billing/webhooks/dodo` |
| Admin | `/admin` | `GET /admin/organizations`, `GET /admin/subscriptions`, `GET /admin/stats`, `GET /admin/usage`, `POST/GET /admin/coupons` |

### Authentication

- **User requests** — `Authorization: Bearer <clerk session token>` (RS256 JWT
  verified against the instance JWKS).
- **Billing webhooks** — Dodo Payments signs requests; the API verifies the
  HMAC-SHA256 signature (`x-webhook-signature` header).
- **Unauthenticated** — `/health`, `/ready`, `/metrics`, `/billing/plans`, and
  the Dodo webhook.

### Rate limiting & errors

- Redis-backed sliding-window rate limits (global, per-IP, per-org, per-endpoint).
  Rate-limit rejections return HTTP 429.
- Quota exhaustion returns HTTP 402 with `{quota_type, current, limit, upgrade_url}`.
- Unhandled errors are caught by middleware and returned as a generic
  `{"detail": "Internal server error"}` — internals are never leaked.

---

## Database & Data Flow

**Technology:** PostgreSQL 16 via SQLAlchemy 2 (async) + asyncpg. All primary
keys are UUIDv4; every domain table carries `organization_id` with `CASCADE`.

### Main entities (23 tables)

| Group | Tables |
| --- | --- |
| Tenancy | `organizations`, `users`, `memberships` |
| Profile (M1) | `company_profiles`, `products`, `icps`, `activation_events` |
| Discovery (M2) | `discovery_runs`, `leads` |
| AI (M3) | `research_snapshots`, `outreach_drafts`, `prompt_versions`, `usage_records` |
| Follow-ups (M5) | `follow_ups` |
| Procurement (M7) | `suppliers`, `rfqs`, `quotes` |
| Billing | `plans`, `subscriptions`, `coupon_codes` |
| SaaS foundation | `jobs`, `idempotency_keys`, `audit_logs` |

### Key relationships

- `Organization 1—N` everything domain-scoped.
- `Lead 1—N` `ResearchSnapshot`, `OutreachDraft`, `FollowUp`.
- `DiscoveryRun 1—N` `Lead` (nullable `run_id`, plus `parent_lead_id` for
  lead-level discovery).
- `RFQ 1—N` `Quote`, `Supplier 1—N` `Quote`.
- `Plan 1—N` `Subscription` (one active/trialing subscription per org).
- `User N—N Organization` via `Membership` (role + status).

### Data flow summary

- **Created:** leads are discovered/upserted by the worker; research/drafts by the
  worker; the rest via user CRUD routes.
- **Processed:** deterministic scoring/planning in `services/`; LLM generation in
  `integrations/llm`; browser scraping in `integrations/maps_scraper`.
- **Queried:** org-scoped reads only; analytics summary is Redis-cached for 60s
  per org.
- **Migrated:** Alembic (`api/alembic/`, 24 revisions). Apply with
  `alembic upgrade head`.
- **Deleted:** cascades from the organization; leads may be bulk-deleted per ICP
  or by ID list.

### Caching

Redis caches analytics summaries (`analytics:{org_id}`, 60s TTL) and backs the
rate limiter and ARQ queue. PostgreSQL remains the source of truth.

---

## Development & Testing

### Backend

```bash
cd api

ruff check .                # lint
ruff format --check .       # format check (ruff format . to apply)
mypy app/                   # type check (see mypy.ini)

python -m pytest tests/ -v --tb=short        # tests (requires Postgres + Redis)
docker compose exec api python -m pytest -q  # or run inside the container
```

The test suite (22 files) covers qualification/planning logic, tenant isolation,
authorization, SSRF protection, rate limiting, entitlements, exports, and the
sales pipeline. It requires a reachable PostgreSQL/Redis (see
`api/tests/conftest.py`).

### Frontend

```bash
cd web

npm run dev          # dev server (Turbopack)
npm run build        # production build
npm run lint         # ESLint
npm run typecheck    # tsc --noEmit
npm run test         # Vitest
```

### Run all checks locally

```bash
./scripts/ci.sh            # lint + typecheck + tests + docker builds
./scripts/ci.sh lint
./scripts/ci.sh typecheck
./scripts/ci.sh test       # requires Postgres + Redis
./scripts/ci.sh migration  # alembic upgrade/downgrade/upgrade cycle
./scripts/ci.sh docker
```

> **Current status note:** as of this writing, `ruff` reports pre-existing lint
> errors and unformatted files, `tsc --noEmit` reports two errors in
> `web/src/__tests__/api.test.ts`, and `npm run lint` fails on an ESLint config
> module-resolution issue. These are pre-existing and unrelated to this README;
> see [Limitations & Roadmap](#limitations--roadmap).

### Adding or modifying features

- **Model change** → edit `api/app/models/`, generate an Alembic revision, apply it.
- **API change** → add a router under `api/app/api/routes/`, wire it in
  `main.py`, and add tests under `api/tests/`.
- **Background job** → add a function in `api/app/tasks/`, register it in
  `worker/main.py` (`WorkerSettings.functions`), and enqueue it via `get_arq_pool()`.
- **Frontend change** → edit `web/src/`, keeping data fetching in `lib/` behind
  the shared API client.

---

## Deployment

Two Compose files are provided:

- **`docker-compose.yml`** — development (hot-reload API volume mount, published
  Postgres/Redis ports, dev billing default).
- **`docker-compose.prod.yml`** — production-oriented: multi-stage images, Uvicorn
  with `--workers 2`, Redis `requirepass`, resource limits, log rotation, isolated
  Docker networks (`frontend` / `backend` / `data`), and required secrets
  (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `REDIS_PASSWORD`, `NEXT_PUBLIC_API_URL`).

```bash
# Production-style (override the dev file)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

### Production considerations

- **Migrations** — run `alembic upgrade head` before/at deploy; automate it in a
  release step rather than at app startup.
- **Secrets** — set required env vars in a secure secret store; never bake them
  into images.
- **Persistence** — mount persistent volumes for `pgdata` and `redisdata`; back
  up PostgreSQL regularly.
- **Networking** — front the API and web with a reverse proxy that terminates TLS
  (production mode restricts CORS to `https://` origins and enables HSTS/CSP).
- **Health checks** — `/health` (liveness) and `/ready` (DB + Redis) are available
  for orchestrators; the worker logs a periodic `ping` and a
  `recover_stale_runs` cron marks stuck runs failed.
- **Horizontal scaling** — API and worker are stateless w.r.t. in-process state;
  PostgreSQL and Redis are the shared state. See
  `docs/architecture/target-saas.md` for the full scaling design.

---

## Security & Privacy

- **Secrets** — kept only in gitignored `.env` / `web/.env.local`. `.env.example`
  has placeholders. Never commit real credentials.
- **Authentication** — Clerk-hosted identity; the API verifies RS256 session
  tokens and never stores passwords.
- **Tenant isolation** — every domain query filters by `organization_id` derived
  from the authenticated principal; workers re-derive the org from the resource
  itself, not from request input.
- **Authorization** — platform-admin operations are gated by `require_admin`.
  (Per-role endpoint enforcement is not yet implemented — see Limitations.)
- **Input validation** — Pydantic schemas on all request bodies; parameterized
  SQL (no string-built queries); URL validation and content sanitization for
  LLM-bound web content (`app/services/security/`), including SSRF protections.
- **Error handling** — generic `500` responses; internal errors, credentials,
  tokens, and stack traces are not exposed.
- **Outbound safety** — Playwright runs in an isolated worker container with no
  access to internal services.

Known limitations for public users are listed in the next section.

---

## Troubleshooting & FAQ

**Connection refused / services down**

```bash
docker compose ps
docker compose exec postgres pg_isready
docker compose exec redis redis-cli ping
curl http://127.0.0.1:8000/api/v1/health
```

**API returns 500 on first request / tables missing**

```bash
docker compose exec api alembic upgrade head
docker compose logs api | grep -i migration
```

**Worker not processing jobs**

```bash
docker compose logs worker
docker compose exec redis redis-cli ping
```

**Web page shows "Loading…" forever**

```bash
docker compose logs web
docker compose exec web wget -qO- http://api:8000/api/v1/health
```

**Port already in use** — change `POSTGRES_PORT` / `REDIS_PORT` (and 3000/8000 if
needed) in `.env`.

**AI features silently do nothing** — ensure `DEEPSEEK_API_KEY` is set and the
worker is healthy; the API enqueues jobs but the worker performs generation.

**Reset the database**

```bash
docker compose down -v
docker compose up --build
docker compose exec api alembic upgrade head
docker compose exec api python -m app.scripts.seed_dev
```

---

## Limitations & Roadmap

### Verified limitations

- **No per-role authorization enforcement** — `owner`/`admin`/`member`/`viewer`
  roles are stored and returned by `/auth/me`, but endpoints only enforce the
  platform-admin gate. Any active member currently has full org access.
- **Job tracking incomplete** — the `jobs` table/model and the `/jobs` page exist,
  but no `/api/v1/jobs` API route is implemented yet.
- **Audit log & idempotency not wired** — models/tables exist but are not yet used
  by middleware/routes.
- **Single worker process** — discovery (long-running Playwright) and LLM jobs
  share one worker with `max_jobs=4`; splitting into discovery/fast workers is
  documented but not yet applied in Compose.
- **No distributed job locking** — safe for a single worker; would need locking
  before scaling workers horizontally.
- **Pre-existing tooling issues** — `ruff`, `tsc`, and `eslint` currently report
  errors (see [Development & Testing](#development--testing)).
- **Tests not verified in this environment** — the backend test suite requires
  Postgres/Redis and was not executed here; results are therefore not asserted.

### Potential future improvements (not commitments)

These are inferred from `docs/architecture/target-saas.md` and the current code,
not committed plans:

- Enforce `require_role()` and feature flags (`plan.features`) per endpoint.
- Wire idempotency keys and admin audit logging into request handling.
- Split workers (`worker-discovery` vs `worker-fast`) and add Redis-based job
  locks and per-org concurrency limits.
- Add a dedicated `/jobs` API and complete the job state machine.
- Introduce Prometheus/Grafana dashboards and distributed tracing.
- Add PgBouncer and read replicas for higher-scale deployments.

---

## Contributing

Basic workflow:

1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/your-feature`).
3. Make and test your changes (see [Development & Testing](#development--testing)).
4. Commit and push to your fork.
5. Open a pull request.

No project-specific contribution policy or code of conduct is present in the
repository; the standard fork → branch → PR flow is assumed.

---

## License & Acknowledgements

No `LICENSE` file was found in the repository, so the project does not currently
declare an open-source license. Assume "all rights reserved" until the author
adds one.

No third-party acknowledgements or attribution files were found.

---

## Additional Resources

- [`docs/architecture.md`](docs/architecture.md) — service and data-model overview
- [`docs/architecture/current-state.md`](docs/architecture/current-state.md) — codebase audit
- [`docs/architecture/target-saas.md`](docs/architecture/target-saas.md) — target SaaS design
- [`docs/local-development.md`](docs/local-development.md) — local development guide
- [`docs/configuration.md`](docs/configuration.md) — configuration reference
- [`docs/redis-usage.md`](docs/redis-usage.md) — Redis key/usage breakdown
- [`docs/ci.md`](docs/ci.md) — CI pipeline documentation
- [`docs/security/threat-model.md`](docs/security/threat-model.md) and
  [`docs/security/production-checklist.md`](docs/security/production-checklist.md) — security analysis

