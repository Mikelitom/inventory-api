# 06 — Validations

&gt; Input validation rules per endpoint for `inventory-api`.
&gt; Status: 📝 Draft | Last revised: 2026-10-20
&gt; All rules below are enforced by Zod schemas at the boundary (controller
&gt; middleware). Business rules that depend on database state (uniqueness,
&gt; stock availability) live in the service layer and return 409/422 — see
&gt; doc 05.

---

## 1. Conventions

- **Layer**: format/shape/type → Zod at the route boundary (400).
  State-dependent rules → service layer (409/422). Never trust one for the other.
- Error shape (see doc 04, §1.3):
  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "Request validation failed.",
    "details": [ { "field": "email", "message": "Must be a valid email address." } ] } }
  ```
- Empty strings are treated as absent (`z.string().trim().min(1)` pattern).
- All numeric inputs are validated as integers within explicit ranges.
- Unknown/extra body fields are **rejected** (`strict()`), catching typos
  like `quantiy` early.

## 2. Shared types

| Type | Rule |
|---|---|
| `UUID` | Valid UUID v4 format. |
| `Email` | RFC 5322 subset via Zod; max 255 chars; lowercased before persistence. |
| `Password` | 8–72 chars, ≥1 uppercase, ≥1 lowercase, ≥1 digit. Max 72 = bcrypt input limit. |
| `Quantity` | Integer, 1–1,000,000. |
| `Money` | Integer (cents), 0–999,999,999. |
| `Reason` (movement) | Trimmed string, 3–500 chars. |
| `Idempotency-Key` | Header, optional, 8–64 chars (UUID recommended). |
| Pagination | `page`: int ≥ 1 (default 1). `pageSize`: int 1–100 (default 20). |
| Dates | ISO 8601 (`from`, `to`); `from` ≤ `to` when both present. |
| Sorting | Whitelist per endpoint; unknown fields → 400. |

## 3. Auth

### 3.1 `POST /auth/register` (ADMIN)

| Field | Rules | Error message |
|---|---|---|
| `email` | required, Email type | "Email is required." / "Must be a valid email address." |
| `password` | required, Password type | "Password must be 8–72 characters and include uppercase, lowercase, and a digit." |
| `role` | required, enum `ADMIN\|MANAGER\|EMPLOYEE` | "Role must be one of: ADMIN, MANAGER, EMPLOYEE." |

Service layer: duplicate email → 409 `EMAIL_TAKEN`.

### 3.2 `POST /auth/login`

| Field | Rules |
|---|---|
| `email` | required, Email format (validated even though failure response is identical for both cases — prevents oracle) |
| `password` | required, 1–72 chars |

Failure (wrong email OR wrong password): 401 `INVALID_CREDENTIALS`,
identical response body and comparable response time.

### 3.3 `POST /auth/refresh`

| Field | Rules |
|---|---|
| `refreshToken` | required, string 16–512 chars |

Invalid/expired/revoked → 401 `INVALID_REFRESH_TOKEN` (same code for all).

### 3.4 `POST /auth/logout`

| Field | Rules |
|---|---|
| `refreshToken` | required, string 16–512 chars |

Always returns 204 (idempotent), even for unknown tokens.

### 3.5 `PATCH /users/:id/role` (ADMIN)

| Field | Rules | Service layer |
|---|---|---|
| `role` | required, enum | Target is last active ADMIN and role ≠ ADMIN → 422 `LAST_ACTIVE_ADMIN` |

### 3.6 `PATCH /users/:id/status` (ADMIN)

| Field | Rules | Service layer |
|---|---|---|
| `isActive` | required, boolean | Deactivating last active ADMIN → 422 `LAST_ACTIVE_ADMIN`; self-deactivation allowed (session ends at token expiry) |

## 4. Categories

### 4.1 `POST /categories` (MANAGER+)

| Field | Rules | Service layer |
|---|---|---|
| `name` | required, trimmed, 2–120 chars | — |
| `description` | optional, ≤500 chars | — |
| `parentId` | optional, UUID | Must reference existing, active category; cannot equal own id → 404 `PARENT_NOT_FOUND` / 422 `CYCLE_DETECTED` |

### 4.2 `PATCH /categories/:id` (MANAGER+)

All fields optional; same rules as POST. Setting `parentId` to a descendant
of the category → 422 `CYCLE_DETECTED`.

### 4.3 `GET /categories`

| Param | Rules |
|---|---|
| `includeInactive` | optional, boolean string (`true`/`false`), default false; EMPLOYEE always gets false regardless |
| `tree` | optional, boolean string, default false |

## 5. Suppliers

### 5.1 `POST /suppliers` (MANAGER+)

| Field | Rules | Service layer |
|---|---|---|
| `name` | required, trimmed, 2–150 chars | — |
| `contactName` | optional, trimmed, ≤150 chars | — |
| `email` | optional, Email type | — |
| `phone` | optional, 5–30 chars; digits, spaces, `+`, `-`, `()` only | "Phone contains invalid characters." |
| *(meta)* | at least one of `contactName`, `email`, `phone` must be present | 400 "Provide at least one contact method." |

Service layer: same `name` + `email` pair exists → 409 `DUPLICATE_SUPPLIER`.

### 5.2 `PATCH /suppliers/:id` (MANAGER+)

All fields optional; same rules as POST. Removing the last contact method
(all three set to null) → 400.

### 5.3 `GET /suppliers`

| Param | Rules |
|---|---|
| `search` | optional, ≤100 chars, escaped against `%`/`_` in LIKE |
| `includeInactive` | optional boolean (MANAGER+ only; EMPLOYEE forced false) |
| `page`, `pageSize` | shared rules |
| `sort` | whitelist: `name`, `-name`, `createdAt`, `-createdAt` |

## 6. Products

### 6.1 `POST /products` (MANAGER+)

| Field | Rules | Service layer |
|---|---|---|
| `sku` | required, trimmed, 3–64 chars; `[A-Z0-9-]` case-insensitive | Duplicate (even inactive) → 409 `DUPLICATE_SKU` |
| `name` | required, trimmed, 2–150 chars | — |
| `description` | optional, ≤1000 chars | — |
| `priceInCents` | required, Money type | — |
| `categoryId` | required, UUID | Must exist and be active → 422 `INVALID_CATEGORY` |
| `supplierId` | optional, UUID | If present: must exist and be active → 422 `INVALID_SUPPLIER` |
| `minStock` | required, integer 0–1,000,000 | — |
| `unit` | required, enum: `unit`, `kg`, `g`, `liter`, `ml`, `box`, `pack` | — |

&gt; No `stock` field on creation — always 0 (doc 05, §8).

### 6.2 `PATCH /products/:id` (MANAGER+)

| Field | Rules |
|---|---|
| all product fields above | optional; same rules as POST |
| `sku` | **not accepted** → 400 `IMMUTABLE_FIELD` (explicit rejection instead of silent ignore) |
| `version` | optional integer ≥ 1; if provided and mismatched → 409 `STALE_VERSION` (see doc 03, D2) |

### 6.3 `GET /products`

| Param | Rules |
|---|---|
| `search` | optional, ≤100 chars; matches `name` (ILIKE) or exact `sku` |
| `categoryId`, `supplierId` | optional UUIDs |
| `lowStock` | optional boolean |
| `includeInactive` | optional boolean (MANAGER+ only) |
| `page`, `pageSize`, `sort` | shared; sort whitelist: `name`, `sku`, `stock`, `priceInCents`, `createdAt` (±) |

### 6.4 `GET /products/:id`

| Param | Rules |
|---|---|
| `includeInactive` | optional boolean (MANAGER+ only); EMPLOYEE requesting inactive product → 404 |

## 7. Movements

### 7.1 `POST /movements` (see doc 05 §2 for the full transaction)

| Field | Rules | Service layer |
|---|---|---|
| `productId` | required, UUID | Exists + active → else 404 / 422 `PRODUCT_INACTIVE` |
| `type` | required, enum `IN\|OUT\|ADJUSTMENT` | — |
| `quantity` | required, Quantity type | For `ADJUSTMENT`: 0–1,000,000 (0 allowed: zeroing stock) |
| `reason` | `ADJUSTMENT`: required, Reason type. `IN`/`OUT`: optional Reason type | — |
| `referenceMovementId` | optional, UUID; only valid when `type = ADJUSTMENT` → 400 otherwise | Must exist and belong to same product → 422 |

Headers: `Idempotency-Key` per shared rules (§2).

Service layer outcomes: `INSUFFICIENT_STOCK` (409), role gate for
`ADJUSTMENT` (403), concurrent lock conflicts resolved per doc 05 §2
(stock guard re-evaluated under lock), idempotency replay (doc 05 §7).

### 7.2 `GET /movements`

| Param | Rules |
|---|---|
| `productId`, `userId` | optional UUIDs |
| `type` | optional enum |
| `from`, `to` | optional ISO dates; `from` ≤ `to` |
| `page`, `pageSize`, `sort` | shared; sort whitelist: `createdAt`, `quantity`, `type` (±; `createdAt` default) |

### 7.3 `GET /movements/:id`

Path `id`: UUID. 404 if not found. No filters.

## 8. Reports & inventory

### 8.1 `GET /inventory/low-stock`

Shared pagination rules only. Additional params ignored → 400 unknown-param
behavior consistent with strict mode (see §1).

### 8.2 `GET /reports/summary`

| Param | Rules |
|---|---|
| `from`, `to` | optional ISO dates; `from` ≤ `to`; default range: last 30 days |

### 8.3 `GET /reports/movements`

| Param | Rules |
|---|---|
| `groupBy` | required, enum `product\|type\|day` |
| `from`, `to` | optional ISO dates; default last 30 days |

## 9. Anti-patterns explicitly excluded

- ❌ Client-controlled `stock` on product create/update — rejected by schema.
- ❌ `isActive` toggles via PATCH on entities where a dedicated
  deactivate endpoint exists (consistent surface; one way to do things).
- ❌ Mass assignment: every body is parsed through a strict schema —
  fields like `createdById`, `version` bumps, or `role` escalation via
  product PATCH are impossible.
- ❌ Silently ignoring invalid query params — strict mode returns 400 so
  frontend bugs surface during development.