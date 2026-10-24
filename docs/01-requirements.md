# 01 — Requirements

&gt; Scope definition document for `inventory-api`.
&gt; Status: ✅ Approved | Last revised: 2026-10-19

---

## 1. Overview

`inventory-api` is a REST API that allows an organization to manage its product
inventory: register products with their category and supplier, track stock
through auditable movements (incoming, outgoing, and adjustments), receive
low-stock alerts, and generate activity reports.

The core value of the system is not the CRUD itself, but **stock integrity**:
every movement is recorded as immutable history, outgoing movements never
produce negative stock, and concurrent operations on the same product are
handled transactionally.

## 2. Objectives

1. Demonstrate the design and implementation of a professional REST API in
   **Node.js + TypeScript**.
2. Demonstrate handling of **non-trivial business rules** (transactions,
   concurrency, auditability).
3. Demonstrate engineering practices: testing, documentation, CI/CD,
   containerization, and semantic commits.

## 3. Out of scope (MVP)

To keep the project focused, the MVP **does not include**:

- Frontend (that is portfolio project #2).
- Multi-warehouse / multi-location (future extension).
- Invoicing or payment gateway integration.
- Real-time notifications (websockets, email).
- Internationalization (the API responds in English only).
- Multi-tenancy (a single organization per deployment).

## 4. Actors and roles

| Role | Description |
|---|---|
| `ADMIN` | Full control: user management, suppliers, logical deletion, stock adjustments. |
| `MANAGER` | Manages products, categories and suppliers; views reports. |
| `EMPLOYEE` | Records stock movements (IN/OUT) and consults products. |

Authentication via **JWT** (access token + refresh token).
Role-based authorization enforced on every protected endpoint.

## 5. Features

### 5.1 Authentication
- [ ] User registration (only `ADMIN` can create users with a role).
- [ ] Login with email + password (bcrypt).
- [ ] Issue access token (short-lived, ~15 min) and refresh token (long-lived, rotating).
- [ ] Session renewal via refresh token.
- [ ] Logout (refresh token revocation).
- [ ] User management (`ADMIN`): list, change role, deactivate.

### 5.2 Catalog
- [ ] **Categories**: CRUD with hierarchy support (optional parent category).
- [ ] **Suppliers**: CRUD with name, contact, email, phone.
- [ ] **Products**: CRUD with name, unique SKU, description, unit price,
      category, supplier, current stock (derived), minimum stock, and unit
      of measure.

### 5.3 Stock movements (core)
- [ ] Record a movement of type `IN` (incoming), `OUT` (outgoing), or `ADJUSTMENT` (manual adjustment).
- [ ] Every movement records: product, type, quantity, reason/notes, user, timestamp.
- [ ] Product stock is updated **in the same transaction** as the movement.
- [ ] An `OUT` that would cause negative stock is rejected → HTTP 409.
- [ ] Movements are **immutable**: they cannot be edited or deleted.
      Corrections are made with an `ADJUSTMENT` referencing the original movement.
- [ ] Manual adjustments require the `ADMIN` role and a mandatory reason.

### 5.4 Queries and reports
- [ ] Product listing with search (name, SKU), filters (category, supplier,
      status) and pagination.
- [ ] Product detail including paginated movement history.
- [ ] **Low stock**: products with stock ≤ minimum stock.
- [ ] **Movement history** filterable by product, type, user and date range.
- [ ] **Summary**: total inventory value, total movements by type for a period.

### 5.5 Logical deletion
- [ ] Products, categories and suppliers are never physically deleted:
      they are deactivated (soft delete).
- [ ] A product with stock &gt; 0 cannot be deactivated without a prior adjustment
      (exact rule to be defined in doc 05).
- [ ] Deactivated entities do not appear in listings by default (optional
      filter to include them).

## 6. Non-functional requirements

| Category | Requirement |
|---|---|
| **Performance** | Response &lt; 200 ms for common queries (datasets ≤ 10k products). |
| **Security** | Hashed passwords (bcrypt, cost ≥ 12); tokens in `Authorization: Bearer`; rate limiting on `/auth/*`; configurable CORS. |
| **Integrity** | Stock never negative, guaranteed at both the database **and** application level. |
| **Traceability** | Every movement records its user and timestamp (audit trail). |
| **Availability** | Health check (`GET /api/v1/health`) for monitoring. |
| **Maintainability** | Test coverage ≥ 70% on business logic; strict TypeScript typing. |

## 7. Tech stack (confirmed)

| Layer | Technology |
|---|---|
| Language | TypeScript (strict) |
| Runtime | Node.js 20 LTS (via mise) |
| Framework | Express |
| ORM | Prisma |
| Database | PostgreSQL 16 (Docker) |
| Auth | JWT (access + refresh) |
| Validation | Zod |
| Testing | Vitest + Supertest |
| API Docs | OpenAPI + Swagger UI |
| DevOps | Docker Compose, GitHub Actions |
| Node version management | mise (`.mise.toml`) |

## 8. Assumptions

- Single organization/monolith per deployment (no multi-tenancy).
- Units of measure are validated free text (e.g. `"unit"`, `"kg"`, `"liter"`)
  to avoid over-engineering the MVP.
- Money is handled as integer cents (`priceInCents`) to avoid floating-point errors.
