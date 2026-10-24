# 09 — Roadmap

&gt; Execution plan and future extensions for `inventory-api`.
&gt; Status: 📝 Draft | Last revised: 2026-10-23

---

## 1. Phases

### Phase 0 — Documentation ✅ (current)
Docs 01–09 approved and merged to `main`. No code yet. Exit criterion: every
design question answerable by reading `docs/`, not the author.

### Phase 1 — Foundation (week 1)
- [ ] Tooling: tsconfig strict, ESLint, Prettier, Vitest, folder skeleton, docker-compose, `.mise.toml`, env validation (`config/env.ts`)
- [ ] CI pipeline (doc 08 §7) green on an empty test suite
- [ ] Health endpoint + error middleware + first integration test
- [ ] Prisma schema from doc 03 + initial migration incl. CHECK constraints/partial index

### Phase 2 — Auth (week 2)
- [ ] Register/login/refresh/logout, bcrypt, JWT middleware, role middleware
- [ ] Seed script with first ADMIN
- [ ] Rate limiting on `/auth/*`; reuse detection (doc 07 ADR-007)
- [ ] Full test coverage of auth flows

### Phase 3 — Catalog CRUD (week 2–3)
- [ ] Categories (incl. hierarchy + cycle detection), suppliers, products
- [ ] All validations from doc 06 with Zod schemas
- [ ] Pagination/search/sort conventions applied uniformly

### Phase 4 — Stock engine (week 3–4) ⭐ core of the portfolio
- [ ] `POST /movements` implementing doc 05 §2 exactly (lock, guards, atomic update, immutable insert)
- [ ] Idempotency-Key support
- [ ] Concurrent-race integration test (`Promise.all` dual OUT)
- [ ] Movement listing + product detail with recent movements

### Phase 5 — Reports & alerts (week 4)
- [ ] Low stock, summary, movements-by-group reports
- [ ] OpenAPI generation from Zod schemas; Swagger UI at `/api/docs`

### Phase 6 — Polish & launch (week 5)
- [ ] README final (badges, quickstart, demo curl commands, architecture diagram)
- [ ] Coverage ≥ 70% on business logic; security notes section
- [ ] Full pass of docs 01–09 against the code; mark all docs ✅
- [ ] Merge `develop` → `main`, tag `v1.0.0`

## 2. Post-MVP extensions (backlog, unscheduled)

| ID | Item | Motivation |
|---|---|---|
| R-1 | Multi-warehouse (stock per location) | Natural scaling of the movement model |
| R-2 | Stock reconciliation job (history vs counter) | Closes drift risk from ADR-004 |
| R-3 | Low-stock notifications (email/webhook) | Operational value |
| R-4 | CSV import/export of products | Common real-world request |
| R-5 | Access-token blacklist on user deactivation | Tightens ADR-007's 15-min window |
| R-6 | API versioning policy (`/api/v2`) + deprecation headers | Long-term maintenance |
| R-7 | Webhooks for movement events | Integration surface |
| R-8 | Metrics (OpenTelemetry) + dashboards | Production readiness |

## 3. Definition of Done (applies to every item)

- [ ] Zod schema + service rule + tests (unit/integration as appropriate)
- [ ] Docs updated if behavior changed (spec, validations, business rules)
- [ ] CI green; commit(s) follow Conventional Commits
- [ ] No secrets, no `console.log`, no TODOs without linked issue

## 4. Portfolio narrative

This project is #1 of three:

1. **inventory-api** (this repo) — backend depth: transactions, auth,
   validation, testing, docs-first.
2. **Frontend** (TBD) — consumes this API; demonstrates UI/UX and state
   management against a real contract.
3. **Full stack** — integrates both ends plus deployment, closing the loop.

Each project must stand alone; together they must tell a coherent story of an
engineer who can own a feature from database schema to deployed UI.