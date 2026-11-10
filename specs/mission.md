# Mission

## Executive summary

A reliability-first backend for a hybrid cafe-POS / e-commerce operation: a catalog with priced options, auditable inventory, and an order lifecycle that cannot be corrupted by concurrency, price tampering, or history rewriting — with a dynamic role and permission model rather than a hardcoded enum.

## Problem statement

Small retail backends fail in three specific, predictable ways:

1. **They oversell.** Two customers check out the last unit at the same time; both succeed; stock goes negative or the second order is quietly unfulfillable.
2. **They rewrite history.** A product price or an option name changes, and past orders retroactively display numbers and names that were never charged or selected.
3. **Their authorization ossifies.** Roles start as an enum, then a `staff_refund` role appears, then `staff_refund_senior`, until the permission model is forty special cases nobody can reason about or audit.

Each failure is cheap to prevent at design time and expensive to retrofit. This project exists to prevent all three deliberately, and to make each prevention demonstrable.

## Target persona & workflow

**In-system personas**

| Persona | Workflow |
| --- | --- |
| `customer` | Browses the active catalog, checks out with options, cancels their own unpaid order, reads their own order history. Sees only their own orders — another user's order is indistinguishable from a nonexistent one. |
| `staff` | Sees all orders, confirms payment (`pending → paid`), advances fulfilment (`paid → preparing → completed`), reads stock levels, cancels unpaid or paid orders, and rings up guest POS orders for walk-ins with no account. |
| `admin` | Owns the catalog, manual stock adjustments, the RBAC model itself (roles, grants, role assignment), and holds every permission in the catalog by construction. |

**The actual reader.** A prospective client evaluating whether this developer can build a reliable system *and* explain why it is reliable. Every design decision should be defensible in one sentence, and every claim should be backed by a test rather than a comment.

## Value proposition & unfair advantage

Most portfolio backends demonstrate *breadth* — many endpoints, many models. This one demonstrates *judgement*: a small surface where every invariant is enforced mechanically and every tradeoff is written down, including the ones that were deliberately deferred.

Concretely: each stock unit is accountable in a ledger with a balance snapshot; each privilege change is recorded with before/after state; each permission check resolves server-side from a code-versioned catalog; and the concurrency guarantee is proven by a failing-if-broken test rather than asserted in a README.

## Measurable success metrics (M1)

| Metric | Target | How it is measured |
| --- | --- | --- |
| Oversell under concurrency | 0 units across N concurrent checkouts of the last unit; exactly one `201`, one `400` | `test_checkout_concurrency` with `asyncio.gather` |
| Stock mutations ledgered | 100% — no unlogged write path | Single-writer function + code review; reconciliation test |
| Ledger reconciliation | `stock_after` matches recomputed balance for **every** product | Reconciliation test over the full table |
| Route authorization coverage | 100% of non-public routes declare a permission | Route-enumeration test over `app.routes` |
| Catalog read latency | p95 < 100 ms on `GET /products` | `scripts/bench_products.py`, 200 sequential requests against a local server with a seeded catalog of 100 products |
| Fresh-machine bootstrap | ≤ 2 documented commands after clone | README walkthrough on a clean database |

Raising a target or dropping a metric requires amending this file.
