# Constitution

**Status:** Ratified v1
**Scope:** Milestone 1 (M1)
**Authority:** Where this document conflicts with `project.md`, this document wins until `project.md` §2 is rewritten.

---

## Axioms (non-negotiable)

1. **Deny by default.** A route without a declared permission is unreachable, not public. Tests prove it by enumerating `app.routes`.
2. **History is append-only.** Orders, line items, option snapshots, and stock movement are never rewritten. Corrections are new rows.
3. **Money is a snapshot.** `order_items.unit_price` and `options_json` freeze price at checkout. Catalog edits only ever affect the future.
4. **The server is the only authority.** Clients send `product_id`, `option_ids`, `quantity`. All prices, totals, and stock effects are recomputed server-side.
5. **One writer per invariant.** Every `products.stock_quantity` mutation goes through a single repository function that also writes `stock_logs` and asserts `stock_after = stock_before + change_qty`. No other code path may touch stock.
6. **Correctness before features.** An unmerged feature is acceptable; an unenforced invariant is not.

---

## Security & auth invariants

| # | Invariant |
| --- | --- |
| S1 | Access token is **thin**: `sub`, `sid`, `perm_epoch`, `iat`, `exp`, `jti`. No roles, no permissions in the payload. |
| S2 | Authorization is resolved **server-side only**. Effective permissions = union over `user_roles` → `role_permissions`, cached in-process for 30 s, invalidated on `perm_epoch` bump. Maximum staleness across workers is 30 s and is documented, not incidental. |
| S3 | Refresh tokens are opaque, stored as SHA-256 hashes, rotated on every use, grouped by `family_id`. Presenting an already-rotated token revokes the entire family. |
| S4 | Role/permission changes **do not** terminate sessions. Only `logout` and reuse detection do. |
| S5 | Ownership failure → **404** (anti-enumeration: "no such order" and "not yours" are indistinguishable). Capability failure → **403**. |
| S6 | The last `admin` cannot be demoted; self-demotion returns `409`. Guarded inside the mutating transaction with row locks; a CLI escape hatch exists for recovery. |
| S7 | `is_system` roles cannot be deleted or renamed. Roles currently assigned to at least one user cannot be deleted (`409`). |
| S8 | `admin` grants are reconciled **additively** at startup: missing catalog permissions are granted, existing ones are never revoked. A permission can never be silently lost from `admin`. |
| S9 | Every RBAC write emits an `rbac_audit_log` row **in the same transaction**, with `before`/`after` JSON. |
| S10 | The permission catalog is declared in code. The DB is validated against it at startup; drift is fatal in dev/CI and a loud warning in production. |
| S11 | The first admin is seeded from `ADMIN_EMAIL` / `ADMIN_PASSWORD_HASH`; the migration fails loudly if they are absent. No plaintext credential in version control. Admin is **never** derived from "first registered user". |
| S12 | Password storage is Argon2id (bcrypt fallback). Hashing never runs on the event loop — CPU-bound crypto is offloaded to a threadpool. |

---

## Data integrity standards

| # | Standard |
| --- | --- |
| D1 | SQLAlchemy 2.0 **async** + `asyncmy`. No sync sessions, no blocking calls inside request paths. |
| D2 | All timestamps are `DATETIME(6)`, UTC-only, with `SET time_zone='+00:00'` on connect. Naive datetimes are bugs. API output is RFC 3339 with an explicit offset. |
| D3 | Money is `NUMERIC(12,2)`, single currency (PHP). `total_amount = Σ(unit_price × quantity)`; no tax or fees in M1. |
| D4 | Fixed vocabularies are `VARCHAR(32)` + `CHECK` (`Enum(native_enum=False, create_constraint=True)`). No native `ENUM`, no lookup tables. |
| D5 | JSON columns (`options_json`, `rbac_audit_log.before/after`) are opaque snapshots — never joined, never filtered in M1. |
| D6 | The catalog is soft-deleted via `is_active`. No hard deletes of products, categories, or options. |
| D7 | Checkout runs in a single transaction under `READ COMMITTED`, locking `products` with `FOR UPDATE` in **ascending `product_id` order** to prevent lock cycles. No reliance on gap locks. |
| D8 | MySQL has no `RETURNING`: repositories obtain generated IDs via flush / `inserted_primary_key`. |
| D9 | Migrations are **not** atomic (MySQL commits DDL implicitly). Every migration must be individually re-runnable and reviewed for partial-failure left-behind state. |
| D10 | Seeding is additive and per-migration. Startup touches `admin` grants only. Runtime edits to non-admin roles are authoritative. |
| D11 | Integration tests run against real MySQL with a schema built by Alembic — never `create_all`, never SQLite. |
| D12 | Transactions are owned by the service layer only. Routers and repositories never open or commit a transaction. |

---

## M1 scope discipline

**Strictly in scope:** auth (register / login / refresh / logout / me) · dynamic RBAC administration · catalog CRUD with soft delete · checkout with server-authoritative pricing · order lifecycle plus a dedicated cancellation endpoint · inventory adjustment with ledger · RBAC audit trail · one concurrency test.

**Strictly out of scope (forbidden in M1, not "later if there is time"):** payments and refunds · reservation model or TTL sweeper · Redis or any cache layer · Docker · file uploads / `Asset` · email or notifications · reporting endpoints · health and metrics endpoints · password reset · email verification · background workers · wildcard permissions · admin-created permission strings.

Adding anything from the out-of-scope list requires amending this document first.
