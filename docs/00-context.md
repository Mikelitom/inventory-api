# 00 — Project Context

> Executive summary for session continuity. Read this first in any new session.
> Status: ✅ Active | Last updated: 2026-10-23

---

## What this project is

`inventory-api` — a public portfolio backend project by **Miguel**. It is
project **#1 of 3** (backend → frontend → full stack) built to demonstrate
transferable backend skills beyond his professional Python/FastAPI experience.

Core domain value: **stock integrity** — movements are immutable audit records,
stock never goes negative, and concurrent movements are handled
transactionally.

## Tech stack (confirmed, ADR-001/002/003)

Node.js 20 LTS (via mise) · TypeScript strict · Express · Prisma · PostgreSQL 16
(Docker) · JWT (access + refresh) · Zod · Vitest + Supertest · Swagger/OpenAPI ·
GitHub Actions · Docker Compose

## Environment notes

- Dev machines: **CachyOS (Arch)** and **Ubuntu 24.04** — all setup commands
in doc 08 cover both.
- Runtime versions managed with **mise** (`.mise.toml` pins Node 20).
- Auth to GitHub: **SSH keys** (git operations); `gh` CLI installed as of
2026-10-23 for PR workflows.

## Current state

| Phase | Status |
| --- | --- |
| Phase 0 — Documentation | ✅ **Complete and merged to `main`** |
| Phase 1 — Foundation | ⬜ Next (scaffolding: tsconfig, ESLint, docker-compose, CI, `/health` endpoint) |
| Phase 2 — Auth | ⬜ |
| Phase 3 — Catalog CRUD | ⬜ |
| Phase 4 — Stock engine (core) | ⬜ |
| Phase 5 — Reports + OpenAPI | ⬜ |
| Phase 6 — Polish, README, v1.0.0 | ⬜ |

All design decisions live in `docs/01`–`docs/09`. Key invariants
(stock ≥ 0, movement immutability, SKU uniqueness, ≥1 active ADMIN) are in
doc 05 §1 and must never be violated by implementation.

## Workflow conventions

- Language: **docs, code, commits in English**; conversation may be Spanish.
- Branches: `main` (protected) ← PRs from `develop`; short feature branches off
`develop` (`feat/auth`, `docs/data-model`, …).
- Commits: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`).
- Docs are the source of truth: when code and docs disagree, **fix the code or
consciously update the docs in the same PR** — never let them drift.

## Next session: Phase 1 — Foundation

Concrete first steps (from doc 09):

1. Repo scaffolding: `src/` structure (config, lib, middlewares, modules),
`tests/`, `prisma/` folders.
2. Config files: `tsconfig.json` (strict), ESLint + Prettier, `.mise.toml`,
`docker-compose.yml` (db + db-test), `.env.example`, CI workflow
`.github/workflows/ci.yml`.
3. `config/env.ts` — Zod-validated environment (fail-fast at boot).
4. First endpoint `GET /api/v1/health` + error middleware + first integration
test. CI must be green before Phase 2 starts.

Exit criterion: `npm run dev` boots the app, `/health` responds, CI passes on
an empty-but-real test suite.

## For a new assistant session

Load context in this order: this file → doc 01 (scope) → doc 05 (business
rules, the heart) → doc 04 + 06 (API contract & validations) → rest as
needed. Confirm with Miguel which phase is active before writing code.
