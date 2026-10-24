# 02 — Architecture

&gt; System and application architecture for `inventory-api`.
&gt; Status: 📝 Draft | Last revised: 2026-10-20
&gt; Style: layered modular monolith. No microservices, no event bus —
&gt; deliberately boring where it should be, careful where it matters.

---

## 1. System context

```mermaid
graph TD
    Client["Frontend / API consumers&lt;br/&gt;(portfolio project #2)"]
    API["inventory-api&lt;br/&gt;Node.js 20 + Express + TypeScript"]
    DB[("PostgreSQL 16&lt;br/&gt;(Docker)")]
    Dev["Developers&lt;br/&gt;(you, on CachyOS / Ubuntu)"]
    CI["GitHub Actions&lt;br/&gt;(CI: lint, test, build)"]

    Client --&gt;|"HTTPS + JWT"| API
    Dev --&gt;|"docker compose"| DB
    API --&gt; DB
    Dev --&gt;|"git push"| CI
    CI --&gt;|"status checks"| Dev
```

Single deployable unit. The database is the only external dependency and runs
containerized in every environment.

## 2. Layered architecture

Each HTTP request flows strictly downward through four layers. Layers may only
call the layer directly below them.

```mermaid
flowchart TD
    subgraph HTTP
        R[Router / Routes] --&gt; MW[Middleware chain&lt;br/&gt;auth, roles, validate, rate-limit]
    end
    subgraph Application
        C[Controllers&lt;br/&gt;parse input → call service → shape response]
        S[Services&lt;br/&gt;business rules, transactions, orchestration]
    end
    subgraph Data
        RP[Repositories&lt;br/&gt;Prisma queries, no business logic]
        DB[("PostgreSQL")]
    end

    MW --&gt; C --&gt; S --&gt; RP --&gt; DB
```

| Layer | Responsibilities | Explicitly NOT allowed |
|---|---|---|
| **Routes** | Map URL + method → controller + middleware chain | Business logic, SQL |
| **Middleware** | Auth (JWT verify), role checks, Zod validation, rate limiting, request logging | DB access |
| **Controllers** | Extract validated input, call one service method, format HTTP response (status codes, envelopes) | Transactions, business rules, Prisma |
| **Services** | All business rules from doc 05; own transactions (`$transaction`); throw typed domain errors | HTTP concepts (req/res), Prisma outside repository calls |
| **Repositories** | Prisma queries only; return domain entities | Business decisions, transaction control |

### Why this matters (and gets tested in interviews)

- **Controllers know HTTP; services don't.** Swapping Express for Fastify
  would only touch routes/controllers. A service that returns `res.status(...)`
  is coupled to the framework — a red flag.
- **Services own transactions, not repositories.** The stock movement needs
  lock + guard + update + insert atomically (doc 05 §2). That orchestration
  is business logic and lives in the service; repositories just execute the
  statements they're given.
- **Errors are typed domain errors**, not HTTP errors. Services throw
  `InsufficientStockError`, `DuplicateSkuError`, etc. A single Express error
  middleware maps them to the error envelope from doc 04 §1.3. New error →
  one-line mapping, no `try/catch` scattered across controllers.

## 3. Request lifecycle (annotated)

```
POST /api/v1/movements
│
├─ 1. Express router matches → middleware chain:
│     rate-limit      → 429 if exceeded
│     auth(JWT)       → 401 if missing/invalid
│     role(EMPLOYEE+) → 403 if insufficient
│     validate(movementSchema) → 400 with field details if invalid
│
├─ 2. Controller: const result = await movementService.create(validatedBody, user)
│
├─ 3. Service (transaction, doc 05 §2):
│     lock product → guards → atomic stock update → insert movement
│     → idempotency handling → returns movement DTO
│
├─ 4. On domain error: throw InsufficientStockError(available: 6)
│
└─ 5. Error middleware: maps to 409 { code: "INSUFFICIENT_STOCK", ... }
```

## 4. Project structure

```
inventory-api/
├── prisma/
│   ├── schema.prisma           # data model (doc 03)
│   ├── migrations/             # generated + hand-edited CHECK/index SQL
│   └── seed.ts                 # first ADMIN + demo data
├── src/
│   ├── config/
│   │   └── env.ts              # Zod-validated env vars (fail-fast at boot)
│   ├── lib/
│   │   ├── prisma.ts           # singleton client
│   │   └── errors.ts           # typed domain errors + error catalog
│   ├── middlewares/
│   │   ├── auth.ts             # JWT verification
│   │   ├── roles.ts            # permission matrix enforcement
│   │   ├── validate.ts         # Zod schema → req.body/req.query
│   │   ├── rate-limit.ts
│   │   └── error-handler.ts    # single error → HTTP mapping
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.repository.ts
│   │   │   └── auth.schemas.ts
│   │   ├── products/           # same 5-file pattern
│   │   ├── categories/
│   │   ├── suppliers/
│   │   ├── movements/          # the core module
│   │   └── reports/
│   ├── app.ts                  # express app wiring (no listen)
│   └── server.ts               # bootstrap: env → prisma → listen
├── tests/
│   ├── unit/                   # services with mocked repositories
│   ├── integration/            # supertest against real API + test DB
│   └── fixtures/
├── docs/                       # this documentation set
├── .env.example
├── .mise.toml
├── docker-compose.yml
└── ... (config files)
```

**Module pattern**: each feature is a vertical slice (routes → controller →
service → repository → schemas) in its own folder. Shared code goes in
`lib/` and `middlewares/`. Adding a feature means adding a module, not
editing five god-files.

## 5. Cross-cutting concerns

| Concern | Where it lives |
|---|---|
| **Config** | `config/env.ts`: one Zod schema validates all env vars at boot; app refuses to start with a clear message if any are missing/invalid. |
| **Logging** | `pino` (structured JSON logs). Request id per request; bodies and tokens never logged. |
| **Security** | bcrypt for passwords; JWT verify middleware; role matrix; rate limiting; CORS allowlist; Helmet for HTTP headers. |
| **API docs** | OpenAPI spec generated from Zod schemas (`zod-to-openapi` or `@asteasolutions/zod-to-openapi`), served by Swagger UI at `/api/docs` (non-production). Single source: the Zod schemas. |
| **Idempotency** | `idempotency_keys` table (key, user_id, endpoint, response, status); enforced inside the movement transaction. |
| **DB access** | Prisma client is a singleton in `lib/prisma.ts`; no module imports Prisma directly. |

## 6. Testing strategy

```mermaid
graph LR
    U[Unit tests&lt;br/&gt;services + rules&lt;br/&gt;Vitest, repos mocked] --&gt; I
    I[Integration tests&lt;br/&gt;HTTP layer via supertest&lt;br/&gt;real Express app + test DB]
    style I fill:#cde
```

- **Test DB**: dedicated PostgreSQL database in docker-compose (`inventory_test`),
  reset per test file with `prisma migrate reset --force`.
- **Critical path coverage** (≥70% target, doc 01 §6): the movement
  transaction gets the most tests — stock math, negative-stock rejection,
  role gates, idempotency replay, concurrent OUT race (two parallel
  requests via `Promise.all`, assert only one succeeds and stock is exact).
- **What is NOT tested**: Express internals, Prisma-generated code, and
  third-party libraries — testing those is testing someone else's code.

## 7. Build & runtime

- **Dev**: `tsx watch` (no build step, instant reload); `docker compose up -d db`.
- **Prod build**: `tsc` → `dist/`; run with `node dist/server.js`; CI verifies
  the build on every PR.
- **Migrations**: applied explicitly via `prisma migrate deploy` (never
  `db push` outside development).

## 8. Deliberate trade-offs

| Decision | Trade-off accepted | Why it's still right |
|---|---|---|
| Modular monolith over microservices | Can't scale modules independently | A portfolio project must be understandable in one sitting; microservices add ops noise with zero domain benefit at this scale |
| Optimistic `version` + row lock hybrid (doc 05 §2) | Slightly more code than pure pessimistic locking | Correctness under concurrency without serializing unrelated reads |
| Env-var config everywhere | No remote config server | Twelve-factor simplicity; secrets manager only swaps *who provides* the env vars (doc 08) |
| No event bus / background jobs | Reports are computed on request | Dataset assumptions from doc 01 §6 make this fine; queue is a roadmap item if usage grows |