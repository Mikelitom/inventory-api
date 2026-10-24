# 04 — API Specification

&gt; REST contract for `inventory-api`.
&gt; Status: 📝 Draft | Last revised: 2026-10-20
&gt; Base URL: `/api/v1` | Format: JSON (UTF-8)

---

## 1. Conventions

### 1.1 Authentication
Protected endpoints require a valid access token:

```
Authorization: Bearer &lt;access_token&gt;
```
Auth endpoints, the health check, and Swagger UI are public.

### 1.2 Success responses
Resources are returned directly (no wrapping envelope). Collections include
pagination metadata:

```json
// Single resource → 200 | 201 (Location header included on 201)
{
  "id": "8b3a...",
  "sku": "TEE-BLK-001",
  "name": "Black T-Shirt",
  "priceInCents": 1999,
  "stock": 42,
  "minStock": 10,
  "unit": "unit",
  "categoryId": "...",
  "supplierId": "...",
  "isActive": true,
  "createdAt": "2026-10-20T15:04:22.000Z",
  "updatedAt": "2026-10-20T15:04:22.000Z"
}
```

```json
// Collection → 200
{
  "data": [ ... ],
  "meta": { "page": 1, "pageSize": 20, "total": 137, "totalPages": 7 }
}
```

### 1.3 Error responses
Uniform shape for every failure:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Available stock (6) is less than requested quantity (10).",
    "details": []
  }
}
```

| HTTP | Code (example) | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body/query fails Zod validation. `details` lists each field error. |
| 401 | `UNAUTHORIZED` | Missing, expired, or malformed token. |
| 403 | `FORBIDDEN` | Valid token, insufficient role. |
| 404 | `NOT_FOUND` | Resource doesn't exist (or is inactive and not visible to this role). |
| 409 | `CONFLICT` | Business rule violation: insufficient stock, duplicate SKU, stale version (concurrent write), entity in use. |
| 422 | `UNPROCESSABLE_ENTITY` | Semantically invalid: e.g. deactivating a product with stock &gt; 0. |
| 429 | `RATE_LIMITED` | Auth endpoints: max 10 requests/min per IP. |
| 500 | `INTERNAL_ERROR` | Unhandled failure. Never leaks internals; logged server-side. |

### 1.4 Pagination, filtering, sorting
- `?page` (default 1) and `?pageSize` (default 20, max 100).
- Filtering via query params documented per endpoint.
- Sorting: `?sort=-createdAt,name` (`-` = descending).
- Timestamps: ISO 8601 UTC (e.g. `2026-10-20T15:04:22.000Z`).

### 1.5 Money and quantities
- Money: integer cents (`priceInCents: 1999` = $19.99). Serialized as integer, never float.
- Quantities: positive integers; direction is given by the movement `type`.

### 1.6 Idempotency (movements only)
`POST /movements` accepts an optional `Idempotency-Key` header (UUID, ≤ 64 chars).
If a request with the same key is retried (network timeout, double-click), the
API returns the original response instead of creating a duplicate movement.
Unkeyed requests execute normally (no forced idempotency in MVP).

---

## 2. Endpoints

### 2.1 Health

#### `GET /health` — public
```json
// 200
{ "status": "ok", "timestamp": "2026-10-20T15:04:22.000Z", "db": "up" }
```

---

### 2.2 Auth

#### `POST /auth/register` — ADMIN only
```json
// Request
{ "email": "ana@company.com", "password": "Str0ng!Pass", "role": "EMPLOYEE" }
// 201 → { "id", "email", "role", "createdAt" }
// 409 if email already registered
```
&gt; Password requirements: min 8 chars, at least 1 uppercase, 1 lowercase, 1 digit.
&gt; Role must be a valid enum value; self-registration without a token is not
&gt; exposed in the MVP (first ADMIN is seeded manually — see doc 08).

#### `POST /auth/login` — public (rate limited)
```json
// Request
{ "email": "ana@company.com", "password": "Str0ng!Pass" }
// 200
{
  "accessToken": "eyJhbG...",   // ~15 min
  "refreshToken": "d2f1a..."    // ~30 days, rotated on use
}
// 401 → { "error": { "code": "INVALID_CREDENTIALS" } } (same message for wrong email or password)
```

#### `POST /auth/refresh` — public
```json
// Request
{ "refreshToken": "d2f1a..." }
// 200 → new accessToken + new refreshToken (old refresh token is revoked)
// 401 if token is invalid, expired, or revoked
```

#### `POST /auth/logout` — authenticated
```json
// Request: { "refreshToken": "d2f1a..." }
// 204 No Content (always, even if the token was already revoked — idempotent)
```

#### `GET /auth/me` — authenticated
```json
// 200 → { "id", "email", "role", "isActive", "createdAt" }
```

#### `GET /users` — ADMIN
Paginated user list. Filters: `?role=`, `?isActive=`, `?search=` (email).

#### `PATCH /users/:id/role` — ADMIN
```json
// Request: { "role": "MANAGER" }
// 200 → updated user | 400 if target is the last active ADMIN
```

#### `PATCH /users/:id/status` — ADMIN
```json
// Request: { "isActive": false }
// 422 → { "code": "LAST_ACTIVE_ADMIN" } if deactivating the last active ADMIN
```

---

### 2.3 Categories

| Method | Path | Role |
|---|---|---|
| GET | `/categories` | any authenticated |
| POST | `/categories` | MANAGER+ |
| GET | `/categories/:id` | any authenticated |
| PATCH | `/categories/:id` | MANAGER+ |
| DELETE | `/categories/:id` | MANAGER+ (soft delete) |

#### `GET /categories`
```json
// Query: ?includeInactive=false&tree=true
// tree=true → nested children arrays
// 200 →
[
  {
    "id": "...",
    "name": "Beverages",
    "description": "...",
    "parentId": null,
    "children": [ { "id": "...", "name": "Soft drinks", "children": [] } ]
  }
]
```

#### `POST /categories`
```json
// Request
{ "name": "Soft drinks", "description": "...", "parentId": "..." }
// parentId must reference an active category; a category cannot be its own ancestor (validated in app)
// 201 → created category
```

#### `PATCH /categories/:id`
```json
// Request (all fields optional)
{ "name": "Sodas", "parentId": null }
// 409 CYCLE_DETECTED if parentId would create a cycle
```

#### `DELETE /categories/:id`
- 204 on success (sets `isActive = false`).
- 409 `CATEGORY_IN_USE` if active products reference it (deactivate products first).
- 422 `HAS_ACTIVE_CHILDREN` — a parent with active children cannot be deactivated (decision Q2, doc 03).

---

### 2.4 Suppliers

| Method | Path | Role |
|---|---|---|
| GET | `/suppliers` | any authenticated |
| POST | `/suppliers` | MANAGER+ |
| GET | `/suppliers/:id` | any authenticated |
| PATCH | `/suppliers/:id` | MANAGER+ |
| DELETE | `/suppliers/:id` | MANAGER+ (soft delete) |

#### `GET /suppliers`
Query: `?search=` (name or contact), `?includeInactive=false`, `?page`, `?pageSize`, `?sort=name`.

#### `POST /suppliers`
```json
// Request
{ "name": "Acme Corp", "contactName": "Jane Doe", "email": "jane@acme.com", "phone": "+52 55 1234 5678" }
// 201 → created supplier | 409 if name + email combination already exists
```

#### `DELETE /suppliers/:id`
- 204 on success. 409 `SUPPLIER_IN_USE` if active products reference it.

---

### 2.5 Products

| Method | Path | Role |
|---|---|---|
| GET | `/products` | any authenticated |
| POST | `/products` | MANAGER+ |
| GET | `/products/:id` | any authenticated |
| PATCH | `/products/:id` | MANAGER+ |
| DELETE | `/products/:id` | ADMIN (soft delete) |

#### `GET /products`
```json
// Query params:
//   ?search=tee            → matches name OR sku (case-insensitive)
//   ?categoryId=...        → exact match
//   ?supplierId=...        → exact match
//   ?lowStock=true         → stock &lt;= minStock
//   ?includeInactive=false
//   ?page=1&pageSize=20&sort=-createdAt
// 200 → paginated list of product objects (see §1.2)
```

#### `POST /products`
```json
// Request
{
  "sku": "TEE-BLK-001",
  "name": "Black T-Shirt",
  "description": "100% cotton",
  "priceInCents": 1999,
  "categoryId": "uuid-of-category",
  "supplierId": "uuid-of-supplier",        // optional
  "minStock": 10,
  "unit": "unit"
}
// 201 → created product (stock starts at 0; first stock comes via an IN movement)
// 409 { "code": "DUPLICATE_SKU" } if sku exists (even on an inactive product)
// 422 if categoryId/supplierId reference inactive entities
```
&gt; `sku` is immutable: not accepted by PATCH.

#### `GET /products/:id`
```json
// 200 → product object + movement summary:
{
  "id": "...", "sku": "...", "name": "...",
  "stock": 42, "minStock": 10, "priceInCents": 1999,
  "category": { "id": "...", "name": "..." },
  "supplier": { "id": "...", "name": "..." },
  "recentMovements": [ /* last 5 movements, newest first */ ],
  "isActive": true, "createdAt": "...", "updatedAt": "..."
}
// 404 if inactive (unless ?includeInactive=true and requester is MANAGER+)
```

#### `PATCH /products/:id`
```json
// Request (all optional): name, description, priceInCents, categoryId,
//                         supplierId, minStock, unit
// 200 → updated product
// 409 STALE_VERSION if the client sends "version" and it doesn't match (optimistic locking)
```
&gt; Clients performing PATCH should send the `version` they read; the response
&gt; includes the incremented `version`.

#### `DELETE /products/:id` — ADMIN
- 204 (soft delete).
- 422 `PRODUCT_HAS_STOCK` if `stock &gt; 0` — an `ADJUSTMENT` to zero is required first (decision Q1, doc 03).

---

### 2.6 Movements (core)

| Method | Path | Role |
|---|---|---|
| POST | `/movements` | any authenticated (rules below) |
| GET | `/movements` | any authenticated |
| GET | `/movements/:id` | any authenticated |

&gt; There is **no** UPDATE or DELETE. Movements are immutable (doc 03, D-decisions).

#### `POST /movements`
```json
// Request
{
  "productId": "uuid",
  "type": "OUT",                       // IN | OUT | ADJUSTMENT
  "quantity": 4,
  "reason": "Damaged in warehouse",    // required only for ADJUSTMENT
  "referenceMovementId": "uuid"        // optional; for corrective ADJUSTMENTs
}
// Headers: Authorization (required), Idempotency-Key (optional, §1.6)

// 201 →
{
  "id": "...",
  "productId": "...",
  "type": "OUT",
  "quantity": 4,
  "reason": null,
  "referenceMovementId": null,
  "productStockAfter": 38,
  "createdBy": { "id": "...", "email": "ana@company.com" },
  "createdAt": "..."
}
```

**Rules enforced (failure → 409/422):**

| Rule | Code | Notes |
|---|---|---|
| `IN` / `OUT` allowed for EMPLOYEE+; `ADJUSTMENT` requires ADMIN | `FORBIDDEN` (403) | |
| `OUT` with `quantity &gt; available stock` | `INSUFFICIENT_STOCK` | Message includes available quantity. |
| `ADJUSTMENT` without `reason` | `VALIDATION_ERROR` (400) | |
| `referenceMovementId` on non-ADJUSTMENT | `VALIDATION_ERROR` (400) | |
| `referenceMovementId` that doesn't exist or belongs to another product | `422` | |
| Concurrent write conflict (version mismatch) | `STALE_VERSION` (409) | Client may retry. |
| Product is inactive | `422 PRODUCT_INACTIVE` | |

#### `GET /movements`
```json
// Query: ?productId=&type=OUT&userId=&from=2026-10-01&to=2026-10-31
//        &page=1&pageSize=20&sort=-createdAt
// 200 → paginated list of movement objects (shape as in POST response)
```

#### `GET /movements/:id`
Returns a single movement. 404 if not found.

---

### 2.7 Inventory & reports

#### `GET /inventory/low-stock`
```json
// Query: ?page&pageSize
// 200 → paginated products where stock &lt;= minStock, sorted by (stock - minStock) ascending
{ "data": [ { "id": "...", "sku": "...", "name": "...", "stock": 3, "minStock": 10 } ], "meta": { ... } }
```

#### `GET /reports/summary`
```json
// Query: ?from=2026-10-01&to=2026-10-31 (both optional; defaults: last 30 days)
// 200 →
{
  "period": { "from": "...", "to": "..." },
  "inventoryValueInCents": 18432050,     // Σ(stock × priceInCents) over active products
  "activeProducts": 137,
  "lowStockProducts": 6,
  "movements": { "IN": 42, "OUT": 38, "ADJUSTMENT": 3 },
  "unitsMoved": { "IN": 1250, "OUT": 1104 }
}
```

#### `GET /reports/movements`
```json
// Query: ?from&to&groupBy=product|type|day&page&pageSize
// groupBy=product → 200
{
  "data": [
    { "productId": "...", "sku": "TEE-BLK-001", "name": "Black T-Shirt",
      "IN": 200, "OUT": 158, "ADJUSTMENT": 0 }
  ],
  "meta": { ... }
}
```

---

## 3. Cross-cutting concerns

- **CORS**: configurable via env var; defaults to same-origin (no origins allowed) in production.
- **Rate limiting**: `/auth/*` → 10 req/min/IP. Other endpoints: generous default (300 req/min) to be tuned.
- **Request logging**: method, path, status, duration, user id. Bodies and tokens are never logged.
- **Swagger UI**: served at `/api/docs` in non-production environments only.
- **Versioning**: path-based (`/api/v1`). Breaking changes require `/api/v2`.

## 4. Deferred / out of scope

- Bulk operations (CSV import/export) — roadmap.
- Webhooks on low stock — roadmap.
- GraphQL gateway — not planned.