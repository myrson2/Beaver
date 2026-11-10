# Beaver — Order & Inventory Management System

A reliability-first backend for a hybrid cafe-POS / e-commerce operation: a catalog with
priced options, auditable inventory, and an order lifecycle that cannot be corrupted by
concurrency, price tampering, or history rewriting.

Built as a portfolio piece demonstrating not just working endpoints, but *enforced
invariants* and defensible architectural decisions.

---

## Status

**Milestone 1 — scaffolding.** The specification suite and directory structure exist;
application code does not yet. Nothing in the "Quickstart" section below runs today — it
is the M1 exit criterion, not a claim.

| Milestone | Theme | State |
| --- | --- | --- |
| M1 | Core systems, dynamic RBAC, audit | In progress — specs ratified, skeleton created |
| M2 | High reliability & verification depth | Planned |
| M3 | Scale & operations | Planned |

---

## What makes this different

Most portfolio backends demonstrate breadth. This one is deliberately narrow and
mechanically enforced:

- **Deny by default.** A route without a declared permission is unreachable, and a test
  enumerates every route to prove it.
- **One writer per invariant.** Every stock change goes through a single function that
  also writes an append-only ledger row with a balance snapshot.
- **History is append-only.** Prices and selected options are snapshotted onto the order
  line; catalog edits only ever affect the future.
- **Dynamic RBAC.** Roles and grants are data; the permission catalog is code. Both are
  reconciled at startup so they cannot silently drift.
- **Server-authoritative math.** The client sends ids and quantities. Prices and totals
  are always recomputed server-side.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| API | FastAPI |
| ORM | SQLAlchemy 2.0 (async) + `asyncmy` |
| Database | MySQL 8.0.46 |
| Migrations | Alembic |
| Validation | Pydantic v2 |
| Auth | Thin JWT access token + opaque rotating refresh tokens |
| Tests | pytest, `pytest-asyncio`, `httpx` against real MySQL |

No Docker in M1 by design — local MySQL with a documented bootstrap.

---

## Architecture

```
router/       HTTP only: parse, authorize, delegate. No SQL, no business rules.
  |
schema/       Pydantic in/out. Never accepts or returns ORM objects directly.
  |
service/      Owns the transaction boundary and business invariants.
  |
policy/       Pure functions: ownership and capability decisions. No I/O, no session.
  |
repository/   All queries. Single stock-writer function. No authorization logic.
  |
model/        SQLAlchemy tables, CHECK constraints, indexes.
```

Only the service layer opens, commits, or rolls back a transaction.

---

## Project structure

```
.
├── app/
│   ├── api/
│   │   ├── deps/            # shared FastAPI dependencies (auth, require(permission))
│   │   └── routes/          # one module per resource
│   ├── cli/                 # create-admin, seed, reconciliation commands
│   ├── core/                # settings, db session, security primitives
│   ├── models/              # SQLAlchemy tables + CHECK constraints
│   ├── policies/            # ownership and capability decisions (pure)
│   ├── repositories/        # all queries; the single stock-writer lives here
│   ├── schemas/             # Pydantic request/response models
│   └── services/            # business invariants + transaction boundaries
├── alembic/
│   └── versions/            # schema + seed revisions
├── scripts/                 # bench_products.py and other one-off tooling
├── specs/                   # ratified specification suite (source of truth)
└── tests/
    ├── integration/         # real MySQL, Alembic-built schema
    └── unit/                # pricing, state machine, policies, resolution
```

---

## Quickstart (M1 target)

Requires Python 3.12+ and a running MySQL 8.0.46 instance.

```bash
# 1. install and migrate (schema + seed + admin bootstrap from env)
pip install -e . && alembic upgrade head

# 2. run
uvicorn app.main:app --reload
```

Environment variables: `DATABASE_URL`, `JWT_SECRET`, `ADMIN_EMAIL`,
`ADMIN_PASSWORD_HASH`. The admin bootstrap migration fails loudly if the admin values
are absent — there is no default credential.

---

## Documentation

| File | Purpose |
| --- | --- |
| [`specs/constitution.md`](specs/constitution.md) | Non-negotiable axioms, security and data invariants, M1 scope |
| [`specs/mission.md`](specs/mission.md) | Problem statement, personas, measurable success metrics |
| [`specs/tech-stack.md`](specs/tech-stack.md) | Stack rationale, layer boundaries, rejected alternatives |
| [`specs/roadmap.md`](specs/roadmap.md) | Milestone breakdown and exit criteria |
| [`AGENTS.md`](AGENTS.md) | Working agreement for AI agents in this repository |
| [`project.md`](project.md) | Original vision document (superseded by `specs/` on conflict) |

---

## Known limitations (deliberate)

- **Abandoned carts hold stock.** Orders deduct inventory at creation; there is no
  reservation model or TTL sweeper until M3. Staff can filter pending orders by age and
  cancel them.
- **Permission propagation is bounded at 30 s** across process boundaries.
- **`total_amount` is a plain sum** of line items — no tax, fees, or delivery.
- **Payments are simulated** by staff marking an order `paid`; a provider webhook arrives
  in M3.

These are written down because a documented limitation reads as judgement, while an
unstated one reads as an oversight.
