# AGENTS.md

Working agreement for AI agents operating in this repository.

## Read this first

`specs/` is the source of truth. Read, in order:

1. `specs/constitution.md` — axioms, security invariants, data standards, scope
2. `specs/mission.md` — personas and measurable success metrics
3. `specs/tech-stack.md` — layer boundaries and rejected alternatives
4. `specs/roadmap.md` — what belongs to which milestone

`project.md` is the original vision document. Where it conflicts with `specs/`, `specs/`
wins.

If a task requires breaking an invariant, **stop and ask**. Do not amend `specs/` to match
the code; amend the code to match `specs/`, or get explicit approval to change the spec
first.

## Current state

Milestone 1, scaffolding stage. Specification suite and directory structure exist.
No application code yet. Do not assume imports, modules, or tests exist.

## Non-negotiables (short form)

- **Deny by default.** Every non-public route declares a permission. Never add a route
  without one; a test enumerates routes to enforce this.
- **Single stock writer.** `products.stock_quantity` is mutated only by the inventory
  repository function that also writes `stock_logs` and asserts
  `stock_after = stock_before + change_qty`. Never `UPDATE` stock anywhere else.
- **Append-only history.** Never mutate `order_items.unit_price`, `options_json`, or
  existing `stock_logs` rows. Corrections are new rows.
- **Snapshot the catalog.** `options_json` stores `{id, name, extra_price}` — a rename or
  price edit must never reach backwards.
- **Server-authoritative math.** Never trust a client-submitted price or total.
- **404 vs 403.** Ownership failure returns 404 ("not yours" is indistinguishable from
  "does not exist"). Capability failure returns 403.
- **Thin tokens.** The access token carries `sub`, `sid`, `perm_epoch`, `iat`, `exp`,
  `jti` only. Never put roles or permissions in it.
- **RBAC writes are audited** in the same transaction, with `before`/`after` JSON.

## Conventions

| Concern | Rule |
| --- | --- |
| Async | Async SQLAlchemy only. No sync sessions, no blocking calls on the event loop. CPU-bound crypto goes to a threadpool. |
| Timestamps | `DATETIME(6)`, UTC only. `datetime.now(timezone.utc)`. Naive datetimes are bugs. |
| Money | `NUMERIC(12,2)`, single currency (PHP). Never floats. |
| Vocabularies | `VARCHAR(32)` + `CHECK` via `Enum(native_enum=False, create_constraint=True)`. No native MySQL `ENUM`. |
| JSON | `options_json` and audit snapshots are opaque — never join or filter them in M1. |
| Deletion | Catalog is soft-deleted with `is_active`. No hard deletes. |
| Transactions | Service layer only. Routers and repositories never commit. |
| IDs | MySQL has no `RETURNING` — flush and read `inserted_primary_key`. |
| Eager loading | Explicit `selectinload`/`joinedload`; lazy loading will raise in async. |
| Locking | Checkout locks `products` with `FOR UPDATE` in ascending `product_id` order, under `READ COMMITTED`. |

## Layering

`router/` → `schema/` → `service/` → `policy/` → `repository/` → `model/`

Forbidden: routers importing repositories; services building raw SQL; repositories
deciding permissions; `policy/` touching the session.

## Adding a permission

A new permission is a four-part change — do all four or none:

1. Add it to the permission catalog constant.
2. Add an Alembic revision seeding it and its default grants for system roles.
3. Let the startup reconciliation grant it to `admin` (additive, never revoking).
4. Add or update the route that requires it, and the RBAC matrix test.

## Testing

- Integration tests run against **real MySQL** with a schema built by Alembic. Never
  `create_all`, never SQLite — that would skip `CHECK` constraints, JSON, and locking.
- Every invariant added should come with a test that fails when the invariant is broken.
- Pin unit tests to pure functions (pricing, state machine, policies); no DB.

## Scope discipline

The out-of-scope list in `specs/constitution.md` is a prohibition, not a backlog
suggestion: no payments, refunds, caching, Docker, uploads, workers, notifications,
reporting, or admin-defined permission strings in M1. Ask before implementing anything
from that list.

## Commands (M1 target — not all exist yet)

```bash
alembic upgrade head                    # schema + seed + admin bootstrap
python -m app.cli create-admin          # recovery escape hatch
pytest                                  # unit + integration
pytest tests/unit -q                    # fast loop
python scripts/bench_products.py        # p95 KPI for GET /products
uvicorn app.main:app --reload           # local server
```

## Style

Match the existing layering and naming once code lands. Prefer explicit over clever.
If a decision is not obvious from `specs/`, write it down rather than inventing a
convention that only exists in code.
