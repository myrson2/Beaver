# Roadmap

## Phased overview

| Milestone | Theme | Corresponds to |
| --- | --- | --- |
| **M1** | Core systems, dynamic RBAC, audit | `project.md` Phase 1 (re-baselined to 13 tables) |
| **M2** | High reliability & verification depth | `project.md` Phase 2 |
| **M3** | Scale & operations | `project.md` Phase 3 |

---

## M1 — Core systems, dynamic RBAC, audit

**Objective:** a secure, functional API that manages accounts, catalog, and orders with a data-driven permission model and mechanically enforced invariants.

| Workstream | Deliverables |
| --- | --- |
| Foundations | Project layout, env-driven settings, async engine with `READ COMMITTED`, `pool_pre_ping`, `pool_recycle=3600`, connection-init `SET time_zone='+00:00'`, Alembic init (non-transactional DDL noted in README) |
| Schema | 13 tables: `users`, `roles`, `permissions`, `user_roles`, `role_permissions`, `refresh_tokens`, `rbac_audit_log`, `categories`, `products`, `product_options`, `orders`, `order_items`, `stock_logs`. `VARCHAR(32)` + `CHECK` vocabularies, `DATETIME(6)`, `NUMERIC(12,2)`, JSON snapshots, `is_active` on catalog |
| Auth | Argon2id hashing offloaded to a threadpool, thin JWT access token, opaque rotating refresh tokens with `family_id` reuse detection, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me` |
| RBAC | Permission catalog constant; seed plus startup drift assertion; `GET/POST/PATCH/DELETE /roles`; `PUT /roles/{id}/permissions`; `PUT /users/{id}/roles` (replace-all); last-admin guard with row locks; `is_system` protection; `rbac_audit_log` written in-transaction |
| Enforcement | `require(permission)` dependency, router-level default-deny, `Policy` ownership checks, 403 capability / 404 ownership split |
| Catalog | Products, categories, and product options CRUD with soft delete. Nested option creation (`POST /products/{id}/options`) and top-level option mutation. Public reads filter on active product **and** active category/options |
| Checkout | Server-authoritative pricing (`price + Σ extra_price`), snapshot `unit_price` and `options_json` (`{id, name, extra_price}`), stock deduction, `stock_logs` entry, deterministic ascending lock order, full rollback if any item fails |
| Order lifecycle | `pending → paid → preparing → completed`; `pending|paid → cancelled`; illegal transitions `422`; `PATCH /orders/{id}/status` forward-only; dedicated `POST /orders/{id}/cancel` with restock; guest orders via nullable `user_id` + `created_by_id`; staff filters `status` and `created_before` |
| Inventory | `GET /inventory`, `POST /inventory/adjust` (reason `restock` / `spoilage` / `adjustment`), reconciliation assertion |
| Tests | Unit: pricing, state machine, policy, permission resolution. Integration: real MySQL, Alembic-built schema. RBAC matrix `(role × endpoint) → status`. Route-coverage enumeration test. **One concurrency test** (`asyncio.gather` on the last unit) |
| Tooling | `scripts/bench_products.py` for the p95 KPI; README with the two-command bootstrap and the documented abandoned-cart limitation |

**Exit criteria**

1. All six success metrics in `mission.md` met and demonstrated.
2. No non-public route without a declared permission.
3. Ledger reconciles for every product.
4. Concurrency test green and repeatable across runs.
5. Fresh database bootstrapped in ≤ 2 commands.
6. `project.md` §2 rewritten to match the 13-table reality (and `Asset` removed).

---

## M2 — High reliability & verification depth

**Objective:** turn the M1 invariants into a suite that would survive a hostile reviewer.

| Workstream | Deliverables |
| --- | --- |
| Contention | 1,000-checkout simulation, deadlock instrumentation, proof that lock ordering eliminates cycles |
| Ledger | Scheduled reconciliation job, drift alerting, ledger export |
| Pricing | Property-based tests over price/option combinations; tamper-attempt test matrix |
| History | `order_item_options` relational table backfilled from `options_json`, enabling option-level reporting |
| Contracts | Unified error envelope, pagination standard, documented status-code policy |
| Audit | Read endpoints over `rbac_audit_log` with actor/target filters |

**Exit criteria**

1. 1,000 simulated concurrent checkouts: zero oversell, zero deadlocks, zero unlogged stock changes.
2. Every constitution invariant has a named test that fails when the invariant is broken.
3. Option-level sales reporting returns correct results for pre-M2 data via backfill.

---

## M3 — Scale & operations

**Objective:** asynchronous processing, caching, and deployability.

| Workstream | Deliverables |
| --- | --- |
| Caching | Redis cache for `GET /products`, invalidation on any catalog write |
| Propagation | `perm_epoch` fan-out via pub/sub, reducing the 30 s staleness bound to near-instant |
| Async work | Celery/RQ workers for receipts and low-stock alerts |
| Reservations | Reservation model and TTL sweeper for abandoned carts, replacing the M1 documented limitation |
| Payments | Provider webhook replacing manual `paid` transition; refund flow |
| Media | `Asset` table reintroduced with real multipart uploads |
| Ops | Docker Compose for app + db + worker; structured logging; `/health` and `/ready` |

**Exit criteria**

1. Cache hit ratio reported and invalidation correctness demonstrated by test.
2. Sweeper demonstrably releases expired holds without overselling.
3. Whole stack starts from a single `docker compose up`.
4. Payment webhook drives the state machine end-to-end in a sandbox.
