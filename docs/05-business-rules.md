# 05 — Business Rules

&gt; Domain rules and invariants for `inventory-api`.
&gt; Status: 📝 Draft | Last revised: 2026-10-20
&gt; This document is the single source of truth for *why* the code does what
&gt; it does. Implementation must match it; tests must verify it.

---

## 1. Core invariants

These must **always** hold, regardless of what the API, a bug, or a race
condition attempts. Ordered from hardest (DB-enforced) to softest (app-enforced):

| # | Invariant | Enforced by |
|---|---|---|
| I1 | `products.stock &gt;= 0` at all times | PostgreSQL CHECK constraint (final net) |
| I2 | Every movement row corresponds to exactly one stock change of its product, applied in the same transaction | Application (only writer) |
| I3 | Movement rows are immutable: never updated, never deleted | Application (no update/delete paths) |
| I4 | `products.sku` unique across all products, including inactive ones | DB UNIQUE constraint |
| I5 | `stock` counter is consistent with the movement history (rebuildable) | Reconciliation job (future) |
| I6 | There is always at least one active `ADMIN` user | Application (role/status changes) |

## 2. The stock movement transaction (heart of the system)

This is the single most important flow in the project. `POST /movements`
must execute the following steps **inside one database transaction**:

```
BEGIN;

1.  SELECT id, stock, version, is_active
    FROM products
    WHERE id = :productId
    FOR NO KEY UPDATE;            -- lock this product row until commit

    → If not found:            ROLLBACK → 404 NOT_FOUND
    → If is_active = false:   ROLLBACK → 422 PRODUCT_INACTIVE

2.  Authorization check (application level, before any write):
    - type IN or OUT  → requires EMPLOYEE role or higher
    - type ADJUSTMENT → requires ADMIN

3.  Compute delta from type:
    - IN        → delta = +quantity
    - OUT       → delta = −quantity
    - ADJUSTMENT→ delta = quantity − stock   (target value, not relative)

4.  Guard:
    - OUT and (stock − quantity) &lt; 0 → ROLLBACK → 409 INSUFFICIENT_STOCK
      (message must include the available quantity)

5.  UPDATE products
    SET stock   = stock + :delta,
        version = version + 1,
        updated_at = now()
    WHERE id = :productId;
    -- row is already locked by step 1, so this cannot be bypassed
    -- the DB CHECK (stock &gt;= 0) still validates the final value

6.  INSERT INTO movements (...)
    VALUES (:productId, :type, :quantity, :reason,
            :referenceMovementId, :currentUserId, now());
    -- if referenceMovementId is provided, verify it exists and belongs
    -- to the same product, otherwise ROLLBACK → 422

7.  Idempotency (only if Idempotency-Key header present):
    - Look up the key in the idempotency store.
    - If found: return the stored response (no new movement is created).
    - If not found: store (key, response, status) before COMMIT,
      with a UNIQUE constraint on the key.

COMMIT;
```

**Why the row lock in step 1?** Because the optimistic `version` check alone
cannot protect a *read-then-write* gap: two concurrent `OUT` requests could
both pass step 4's guard before either commits. The lock serializes writers
per product **only for the duration of the transaction** (~milliseconds), and
`FOR NO KEY UPDATE` is the weakest lock that suffices (it still allows plain
reads and foreign-key references). In practice this is a hybrid: the lock
guarantees correctness under concurrency, while `version` (doc 03, D2) remains
useful for PATCH flows where no lock is held.

**Why is ADJUSTMENT an absolute target?** Because relative adjustments are
error-prone ("add the difference" depends on a stock value that may be stale).
An `ADJUSTMENT` with `quantity: N` means "*set* stock to N", making the
client's intent explicit and verifiable.

## 3. Movement immutability and correction flow

- Movements cannot be edited or deleted through any endpoint (rule I3).
- Human errors are corrected with a compensating `ADJUSTMENT`:

```
Scenario: an EMPLOYEE registers OUT 50 units, but meant 15.

1. The original movement stays untouched (audit trail).
2. An ADMIN registers:
   POST /movements
   {
     "productId": "...",
     "type": "ADJUSTMENT",
     "quantity": &lt;current stock + 35&gt;,   // restore what the error removed
     "reason": "Correction: OUT registered with wrong quantity (meant 15, registered 50)",
     "referenceMovementId": "&lt;id of the wrong OUT&gt;"
   }
```

- `reason` is **mandatory** for every `ADJUSTMENT` and must explain *why*.
- `referenceMovementId` links the correction to the original error.

## 4. Deactivation rules (soft delete)

| Entity | Can be deactivated when… | Blocked when… |
|---|---|---|
| Product | Always check first: `stock = 0` | `stock &gt; 0` → 422 `PRODUCT_HAS_STOCK` (adjust to zero first). |
| Category | No active products reference it, and no active children | 409 `CATEGORY_IN_USE` / 422 `HAS_ACTIVE_CHILDREN` |
| Supplier | No active products reference it | 409 `SUPPLIER_IN_USE` |
| User | Never the last active `ADMIN` | 422 `LAST_ACTIVE_ADMIN` |

Deactivation is a `PATCH ... isActive=false` (or DELETE alias), never a
physical `DELETE`. Physical deletion is blocked by FK `ON DELETE RESTRICT`.

## 5. Authentication & authorization rules

1. **Registration is closed to the public.** Users are created only by an
   `ADMIN`. The very first `ADMIN` is seeded by a setup script (doc 08).
2. **Login responses never distinguish** "wrong email" from "wrong password"
   (same `INVALID_CREDENTIALS` error, same timing) — prevents user enumeration.
3. **Refresh token rotation:** each `/auth/refresh` issues a new refresh token
   and revokes the previous one. Reuse of a *revoked* token (possible token
   theft) triggers revocation of the entire token family → user must log in
   again.
4. **Password policy:** min 8 chars, 1 upper, 1 lower, 1 digit. bcrypt cost ≥ 12.
5. **Role hierarchy:** `ADMIN &gt; MANAGER &gt; EMPLOYEE`. Role checks are
   centralized in a single middleware using a permission matrix, not scattered
   `if (role === ...)` across handlers.
6. **Deactivated users** cannot log in; their existing access tokens expire
   naturally (≤ 15 min) — acceptable for the MVP; a token blacklist is a
   documented future extension.

## 6. Derived values and reporting rules

- `inventoryValueInCents = Σ (stock × priceInCents)` over **active** products
  (inactive products are excluded from valuations).
- `unitsMoved` counts quantities by movement type within the period.
- A product appears in **low stock** when `stock &lt;= minStock` and `is_active`.
- The `stock` counter is the operational value used everywhere; movement
  history is the audit source. If they ever diverge, the history is assumed
  correct and a correction `ADJUSTMENT` is issued (rule I5).

## 7. Failure semantics

| Situation | Outcome | Code |
|---|---|---|
| Client sends `Idempotency-Key` already used | Original response replayed, no new movement | 201 (original status) |
| Concurrent OUT requests, stock suffices for only one | Second fails, reports current available stock | 409 |
| Server crash mid-transaction | Transaction rolls back: no movement, no stock change | — (atomicity) |
| DB CHECK fires (stock would go negative) | 500-class internal error logged; **never** exposed to client | 500 |
| Validation error (bad type, missing reason…) | Nothing written | 400 / 422 |

## 8. Explicitly allowed (don't "fix" these)

- Stock of `0` is valid and normal.
- An `IN` movement is the only way to introduce stock; products are created
  with `stock = 0` (no initial stock field on creation).
- A product may have no supplier (`supplierId = null`).
- Categories may be arbitrarily deep (tree) — the app validates against
  cycles, not depth.
- Reason text on `IN`/`OUT` is optional but encouraged; on `ADJUSTMENT`
  it is mandatory.