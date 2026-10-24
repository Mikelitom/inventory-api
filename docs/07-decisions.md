# 07 — Architecture Decision Records

&gt; Log of significant design decisions for `inventory-api`.
&gt; Status: 📝 Draft | Last revised: 2026-10-23
&gt; Format: each ADR states context → decision → consequences. ADRs are
&gt; immutable once accepted; revisiting a decision means a new ADR that
&gt; supersedes the old one.

---

## ADR-001: Node.js + TypeScript + Express as the stack

**Context.** The author already has professional experience with Python/FastAPI
and wants to demonstrate that backend fundamentals (layering, transactions,
auth, validation) transfer across languages. The next portfolio project
(frontend) and the final one (full stack) will live in the JavaScript
ecosystem.

**Decision.** Build the API on Node.js 20 LTS, TypeScript in strict mode, and
Express as the HTTP framework, with Prisma as ORM and PostgreSQL as the store.

**Alternatives rejected.**
- *NestJS*: excellent framework, but its DI/module system does so much for the
  developer that the repo would showcase the framework more than the author.
- *Spring Boot / .NET*: strong enterprise signal, but the slower iteration
  would stretch the roadmap for no domain benefit.
- *Go*: attractive, but smaller job market in the target region and would not
  compound with portfolio projects #2 and #3.

**Consequences.** Structure, middleware, and error handling are hand-rolled —
more code, but every line is a deliberate decision that can be defended.
TypeScript strictness is non-negotiable (see ADR-006).

---

## ADR-002: PostgreSQL as the single store, via Docker everywhere

**Context.** The system needs ACID transactions, row-level locking, CHECK
constraints, and partial indexes (docs 03–05).

**Decision.** One PostgreSQL 16 instance per environment, launched with
docker-compose in development and CI; identical container in production.

**Alternatives rejected.**
- *SQLite*: no real concurrency control — disqualifying for the movement
  engine.
- *MongoDB / document DB*: the domain is relational (products, categories,
  suppliers, movements); transactions exist but constraints and migrations are
  weaker.

**Consequences.** The `docker compose up -d db` step becomes the universal
setup path on both developer machines (CachyOS, Ubuntu) and CI. No
environment-specific database quirks.

---

## ADR-003: Prisma as ORM

**Context.** Raw SQL maximizes control but duplicates the data model (doc 03)
in code and makes migrations manual. An ORM should keep schema and code in
sync with type safety.

**Decision.** Prisma, with migrations versioned in `prisma/migrations/`.
CHECK constraints and partial indexes added via hand-edited migration SQL
(Prisma has no native support), documented in the migration files.

**Alternatives rejected.**
- *TypeORM / Drizzle*: viable; Prisma chosen for schema readability and
  mature migration tooling. Re-evaluate in a future project.
- *Raw SQL (`pg`)**: full control, but the project would spend its budget on
  boilerplate instead of business rules.

**Consequences.** A few advanced SQL features require manual migration edits —
an accepted, documented trade-off.

---

## ADR-004: Denormalized stock counter with movement history as audit trail

**Context.** Stock can be (a) derived from movement history, (b) stored as a
counter updated per movement, or (c) both.

**Decision.** (c) — see doc 03, decision D1. The counter serves reads; the
immutable movement log serves audit and rebuilds.

**Consequences.** Introduces a theoretical risk of counter/history drift.
Mitigation: DB CHECK guards the counter's range; a reconciliation routine
(summing history vs counter) is roadmap item R-6 (doc 09).

---

## ADR-005: Hybrid concurrency control for stock movements

**Context.** Concurrent movements on one product must never corrupt stock
(doc 05 §2).

**Decision.** Row lock (`SELECT ... FOR NO KEY UPDATE`) inside the movement
transaction, plus a DB CHECK as final net; the `version` column is kept for
optimistic control on product PATCH.

**Alternatives rejected.**
- *Pure optimistic (version check)*: loses the read-then-write race — two
  concurrent `OUT` can both pass the guard.
- *Pure pessimistic everywhere*: serializes all writers even when no conflict
  exists, and offers no protection for lock-free PATCH flows.

**Consequences.** Correctness is guaranteed with minimal contention; the code
must hold the transaction discipline documented in doc 05.

---

## ADR-006: TypeScript strict mode, strict Zod parsing, strict API surface

**Context.** The project's value is its engineering quality; silent leniency
(any-casting, unknown fields ignored, implicit any) erodes that signal.

**Decision.** `strict: true` in tsconfig; Zod `.strict()` on all request
bodies; unknown query params rejected (400); unknown body fields rejected.

**Consequences.** Slightly more friction during development (a typo in a query
param fails loudly). Accepted: fail-fast surfaces bugs in development that
would otherwise become 404s in production.

---

## ADR-007: JWT access + refresh tokens, rotation with reuse detection

**Context.** Stateful sessions (server-side session store) are simpler to
revoke but require shared state; JWTs are stateless but hard to revoke.

**Decision.** Short-lived access tokens (15 min) + long-lived rotating
refresh tokens (30 days) persisted in the DB. Refresh reuse of a revoked
token revokes the whole family (theft detection). Deactivated users' access
tokens expire naturally (≤ 15 min exposure) — acceptable for MVP.

**Alternatives rejected.**
- *Opaque sessions (Redis)*: adds infrastructure; revisit at scale.
- *Long-lived JWTs only*: unacceptable revocation story.

**Consequences.** Two tables/queries per auth flow; the security posture is
strong enough for a portfolio-grade system without extra infrastructure.

---

## ADR-008: Documentation-first, English, in-repo

**Context.** The repository is a public portfolio artifact read by
recruiters and engineers who may not speak Spanish.

**Decision.** All repo artifacts (docs, README, code comments, commit
messages) in English; conversation/planning can stay in Spanish. Docs are
written before code and versioned in `docs/`.

**Consequences.** Higher writing effort up front; the payoff is a repo that is
self-explanatory without the author present.