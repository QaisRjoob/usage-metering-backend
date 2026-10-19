<div align="center">

# Usage Metering & Quota Service — Backend

**REST API for two-level usage metering and quota enforcement**

![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178C6?logo=typescript&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-22-5FA04E?logo=nodedotjs&logoColor=white)
![Fastify](https://img.shields.io/badge/Fastify-5-000000?logo=fastify&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169E1?logo=postgresql&logoColor=white)
![JWT](https://img.shields.io/badge/Auth-JWT-000000?logo=jsonwebtokens&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)

[Frontend](https://github.com/QaisRjoob/usage-metering-frontend) ·
[Main project](https://github.com/QaisRjoob/usage-metering-service) ·
[API contract](docs/API_CONTRACTS.md) ·
[Database](docs/DATABASE.md) ·
[Security](docs/SECURITY.md)

</div>

---

## Live demo

| What | URL |
|---|---|
| Web app | https://usage-metering-frontend-w270.onrender.com |
| API | https://usage-metering-backend.onrender.com/api/v1 |
| API docs (Swagger UI) | https://usage-metering-backend.onrender.com/docs |
| Health check | https://usage-metering-backend.onrender.com/api/v1/ready |

The login page has a **Test accounts** box for one-click sign-in. Code:
[frontend](https://github.com/QaisRjoob/usage-metering-frontend) ·
[main project](https://github.com/QaisRjoob/usage-metering-service).

Render's free plan sleeps after about 15 minutes without traffic, so the first request can take up
to a minute.

## Overview

Other applications call this API before a user performs a metered action. Every request is checked
against two plans at once: the calling application's shared **account plan** and the acting
**user's plan**. The API records every decision, explains rejections (which rule, which level), keeps
history for any month, and stays correct under retries and concurrent requests.

It is one part of the **Usage Metering & Quota Service** project. The web app (operator console,
company portal, Client Demo) lives in the
[frontend repository](https://github.com/QaisRjoob/usage-metering-frontend).

## Features

- Two-level quota decision (user plan **and** account plan) in one PostgreSQL transaction
- Clear rejections: rule (`MONTHLY_ALLOWANCE`, `MAX_UNITS_PER_REQUEST`) and level (`user`, `account`)
- Idempotent retries through a client `requestId` (accepted and rejected decisions are replayed)
- Correct under concurrency: row locks in a fixed order, tested with up to 150 simultaneous requests
- Versioned plans: changes apply immediately, history keeps the exact versions that decided each request
- JWT access tokens with rotating refresh tokens (`HttpOnly` cookie, reuse detection) and role guards
- API keys for server-to-server clients, stored only as SHA-256 hashes
- Self-service sign-up with operator approval and one-time key collection
- Filtering, sorting and pagination on every list (zod query DTOs)
- AI Usage Advisor (mock, or OpenAI / any OpenAI-compatible API such as OpenRouter)
- OpenAPI document with Swagger UI, CLI example client, demo data generator

## Tech stack

| Area | Technologies |
|---|---|
| Runtime | Node.js 22, TypeScript 5.9 |
| Web framework | Fastify 5 (helmet, rate limit, cookie, CORS, swagger) |
| Database | PostgreSQL 17 with plain SQL (`pg`), SQL migrations |
| Validation | zod 4 (request and query DTOs shared with the frontend) |
| Auth | `jose` (JWT HS256), SHA-256 hashed API keys and refresh tokens |
| AI | Official OpenAI SDK (Responses API, or Chat Completions for compatible APIs) |
| Tests | Vitest: unit, integration (real PostgreSQL), concurrency, load tests |

## Quick start (Docker)

Requirements: Docker with Compose v2.

```bash
docker compose up --build
```

| What | URL |
|---|---|
| API | http://localhost:3000/api/v1 |
| Swagger UI / OpenAPI JSON | http://localhost:3000/docs · http://localhost:3000/api/v1/openapi.json |
| PostgreSQL | `localhost:5433`, user / password / database `metering` |

Startup applies the migrations and seeds the predefined plans and the demo data (all idempotent).

**Local demo credentials**

| Credential | Value |
|---|---|
| Operator token | `local-admin-token-change-me` (exchanged for an operator JWT at `POST /auth/token`) |
| Demo account API key | `umk_demo_local_only_do_not_use_in_production` (account "شركة الوثائق التجريبية") |

Reset everything with `docker compose down -v`.

## Local development (without Docker)

Requirements: Node.js 22+ and PostgreSQL 15+.

```bash
npm install
cp .env.example .env
npm run dev
```

Set `DATABASE_URL`, `ADMIN_API_TOKEN` and `JWT_SECRET` in `.env` first. The server listens on
http://localhost:3000 and migrates and seeds on start. `npm run db:migrate` and `npm run db:seed` run
those steps by hand.

## Configuration

All settings are environment variables, validated at startup. The full list with comments is in
[.env.example](.env.example).

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | – | PostgreSQL connection string (required) |
| `PORT` / `HOST` | `3000` / `0.0.0.0` | Listen address (`::` for IPv6 private networks) |
| `ADMIN_API_TOKEN` | – | Operator credential, at least 16 characters (required) |
| `JWT_SECRET` | – | HS256 signing secret, at least 32 characters (required) |
| `ACCESS_TOKEN_TTL_SECONDS` / `REFRESH_TOKEN_TTL_DAYS` | `900` / `7` | Token lifetimes |
| `COOKIE_SECURE` | `false` | `true` behind HTTPS |
| `CORS_ORIGINS` | localhost origins | Only needed when a browser calls the API directly |
| `TRUST_PROXY_HOPS` / `TRUST_PROXY_ADDRESSES` | `0` / – | Which reverse proxies may set `X-Forwarded-For` |
| `RUN_MIGRATIONS` / `SEED_DEMO_DATA` | `true` / `true` | Startup steps |
| `AI_PROVIDER` | `mock` | `mock` or `openai` (with `OPENAI_API_KEY`, `OPENAI_MODEL`, optional `OPENAI_BASE_URL`) |

## API

Base path `/api/v1`. Full contract: [docs/API_CONTRACTS.md](docs/API_CONTRACTS.md), live at `/docs`.

| Area | Endpoints |
|---|---|
| Health | `GET /health`, `GET /ready` |
| Auth | `POST /auth/token`, `POST /auth/refresh`, `POST /auth/logout` |
| Sign-up | `POST /signup-requests`, `POST /signup-requests/status`, `POST /signup-requests/claim` |
| Operator | `/admin/accounts`, `/admin/users`, `/admin/plans`, `/admin/usage`, `/admin/analytics/*`, `/admin/search`, `/admin/signup-requests` |
| Account | `GET /account`, `PUT /account/plan`, `GET /account/usage-summary` |
| Users | `GET/POST /users`, `GET /users/:userId`, `PUT /users/:userId/plan`, `GET /users/:userId/usage-summary` |
| Plans | `GET/POST /plans`, `GET/PATCH /plans/:planId` |
| Quotas | `GET /quotas`, `POST /quotas/check` (dry run) |
| Usage | `POST /usage/consume`, `GET /usage`, `GET /usage/:usageId` |
| AI | `POST /ai/advisor` |

Errors always look like `{"error": {"code", "message", "requestId", "details?"}}`; lists return
`{items, page, pageSize, totalItems, totalPages, hasNextPage, hasPreviousPage}`.

```bash
API=http://localhost:3000/api/v1
KEY=umk_demo_local_only_do_not_use_in_production

curl -s -X POST $API/usage/consume -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"userId":"ahmad","units":10,"requestId":"doc-1001"}'
```

## Quota rules

Both rules apply on both levels, checked in this order (the first failure is the reported `rejection`,
all failures are listed in `violations`):

1. user `MAX_UNITS_PER_REQUEST`
2. account `MAX_UNITS_PER_REQUEST`
3. user `MONTHLY_ALLOWANCE` (UTC calendar month)
4. account `MONTHLY_ALLOWANCE`

A rejection consumes nothing. Predefined plans: **Starter** (account, 10,000/month, 100 per request),
**Pro** (account, 100,000 / 500), **Basic** (user, 1,000 / 20), **Premium** (user, 10,000 / 200).

## Idempotency and concurrency

- `requestId` is unique per account. The same key with the same payload replays the stored decision
  (`Idempotent-Replayed: true`); a different payload answers `409 IDEMPOTENCY_KEY_REUSED`.
- Each decision is one transaction: advisory lock on the request id, `SELECT … FOR UPDATE` on the
  account counter then the user counter, evaluate, update counters, store the record, commit.
- Tests verify that counters always equal the sum of accepted records and that no limit is exceeded.

## Project structure

```text
src/
  Domain/          pure business rules (QuotaPolicy, plans, periods, API keys)
  Application/     use cases (Commands / Queries), ports, DTO mapping
  Infrastructure/  PostgreSQL (migrations, seed, repositories, transactions), JWT, AI providers
  Presentation/    Fastify routes, error envelope, OpenAPI
  Shared/          errors, auth guards, rate limiting, validation
  contracts/       DTOs and zod schemas (copied unchanged to the frontend)
  container.ts     composition root
tests/             unit, integration, concurrency, performance
scripts/           CLI client and demo data generator
```

## Testing

| Suite | Command | Needs |
|---|---|---|
| Unit | `npm test` | nothing |
| Integration | `npm run test:integration` | PostgreSQL |
| Concurrency | `npm run test:concurrency` | PostgreSQL |
| Typecheck / lint / build | `npm run typecheck` · `npm run lint` · `npm run build` | nothing |

Database tests recreate the database in `TEST_DATABASE_URL` (its name must contain "test"); with the
compose database: `TEST_DATABASE_URL=postgres://metering:metering@localhost:5433/metering_test`.
Load test method and results: [tests/performance/RESULTS.md](tests/performance/RESULTS.md).

## Deploy (Render)

1. Create a **PostgreSQL** database (*New → Postgres*) and copy its **Internal Database URL**.
2. Create a **Web Service** from this repository with the **Docker** runtime. The `Dockerfile` in
   the root is used automatically.
3. Set the variables:

| Variable | Value |
|---|---|
| `NODE_ENV` | `production` |
| `DATABASE_URL` | the database's Internal Database URL |
| `ADMIN_API_TOKEN` | *Generate* (a long random value) |
| `JWT_SECRET` | *Generate* (a long random value) |
| `COOKIE_SECURE` | `true` |
| `TRUST_PROXY_HOPS` | `2` (Render's edge plus the frontend's `/api` rewrite) |
| `SEED_DEMO_DATA` | `true` for a public demo, `false` otherwise |

4. Set the health check path to `/api/v1/ready`.

Render sets `PORT`. The frontend Static Site rewrites `/api/*` to this service's public URL
(https://usage-metering-backend.onrender.com), so the browser stays same-origin. The same image runs
on any Docker host.

---

<div align="center">

Final Project of the Boot.dev Plan (TypeScript), 2026-C3 · Built by [Qais Rjoob](https://github.com/QaisRjoob)

</div>
