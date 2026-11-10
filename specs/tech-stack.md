# Tech Stack

## Core stack matrix

| Layer | Technology | Version | Rationale |
| --- | --- | --- | --- |
| Language | Python | 3.12+ | Async-native, strong typing ecosystem |
| API | FastAPI | current | Dependency injection maps cleanly onto `require(permission)`; automatic OpenAPI |
| ORM | SQLAlchemy | 2.0 async | Modern typed ORM; explicit eager loading disciplines the async model |
| DB driver | `asyncmy` | current | Maintained, fastest async MySQL driver; matches SQLAlchemy's own guidance |
| Database | MySQL | 8.0.46 | The developer's actual environment. Provides `CHECK` (≥8.0.16), `SKIP LOCKED`, JSON |
| Migrations | Alembic | current | Autogenerate plus explicit seed revisions. DDL is non-transactional — acknowledged in D9 |
| Validation | Pydantic | v2 | Request/response contracts, settings management |
| Passwords | Argon2id (`argon2-cffi`), bcrypt fallback | current | Modern memory-hard default |
| Tokens | PyJWT + `secrets` | current | Thin JWT access token; opaque refresh token |
| Tests | pytest, `pytest-asyncio`, `httpx.AsyncClient` | current | Genuinely concurrent async tests against real MySQL |
| Config | `pydantic-settings`, env-driven | current | No secrets in the repository |

## Layer boundaries

```
router/         HTTP only: parse, authorize, delegate. No SQL, no business rules.
  |
schema/         Pydantic in/out models. Never accepts or returns ORM objects directly.
  |
service/        Owns the transaction boundary and business invariants.
  |
policy/         Pure functions: ownership and capability decisions. No I/O, no session.
  |
repository/     All queries. Single stock-writer function. No authorization logic.
  |
model/          SQLAlchemy tables, CHECK constraints, indexes.
```

**Hard rules**

- Routers may not import repositories.
- Services may not construct raw SQL.
- Repositories may not decide permissions.
- `policy/` may not touch the session.
- Only the service layer opens, commits, or rolls back a transaction.

## Rejected alternatives & forbidden patterns

| Rejected | Reason |
| --- | --- |
| PostgreSQL | Developer's environment is MySQL only. Accepted consequences: no `TIMESTAMPTZ`, no `jsonb`, no `RETURNING`, non-atomic DDL. |
| Static enum RBAC | The specific thing M1 exists to replace. |
| Casbin / `fastapi-users` / `authlib` | They hide the exact logic this project is meant to demonstrate. |
| Permissions embedded in the JWT | Unrevocable until expiry, and treats client-held data as authorization. |
| Wildcard `*` admin grant | Hides catalog gaps; `admin` is granted every permission explicitly. |
| Admin-defined permission strings | Typos become silent denials with no compile-time signal. |
| Cascade delete of roles | Silent privilege loss at an arbitrary moment. |
| Hard delete of catalog rows | Breaks foreign-key integrity and destroys history. |
| Native MySQL `ENUM` | Inserts `''` in non-strict mode; sorts in declaration order. |
| Lookup tables for statuses | Contradicts the state machine being a code invariant, and invites invalid states. |
| `TIMESTAMP` columns | Values depend on session `time_zone` — unacceptable in an audit trail. |
| SQLite for tests | Would not exercise `CHECK` constraints, JSON columns, or row locking. |
| Sync SQLAlchemy | Would weaken the concurrency demonstration. |
| Docker in M1 | Deferred to M3 by design; local MySQL with a documented two-step bootstrap instead. |
| Generic `audit_log` table | `stock_logs` already covers inventory; a generic table invites half-populated rows. |
