# Python backend

Defaults when the architect hasn't decided otherwise; check current stable versions before scaffolding.

## 1. Stack

| Need | Default | Alternatives |
|---|---|---|
| Runtime | CPython 3.12+ (current stable) | |
| Packaging | uv (lockfile, `pyproject.toml`, workspaces) | Poetry |
| API | FastAPI + Pydantic v2 | Django + DRF / Django Ninja when the project wants Django's admin and ORM; Litestar |
| ORM / SQL | SQLAlchemy 2.0 (typed `Mapped[]`, async with asyncpg) | Django ORM; raw SQL via asyncpg / psycopg 3 for hot paths |
| Migrations | Alembic (autogenerate as a draft, always reviewed) | Django migrations |
| Settings | pydantic-settings | |
| Jobs | arq (async, Redis) or Celery (mature, many brokers) | Dramatiq; Temporal for long workflows |
| HTTP client | httpx (async, timeouts) + tenacity for retries | |
| Lint / format | Ruff (lint + format) | |
| Types | mypy `--strict` or pyright strict | |
| Tests | pytest, pytest-asyncio / anyio, Testcontainers, factory_boy / polyfactory, respx (httpx mocking), schemathesis (contract-based fuzzing) | Hypothesis for properties |
| Architecture checks | import-linter contracts (domain imports nothing from adapters / frameworks) | |
| Observability | structlog (JSON), OpenTelemetry auto-instrumentation for FastAPI, SQLAlchemy, httpx, Redis | |

## 2. Layout (hexagonal)

```
services/bookings/
  pyproject.toml
  src/bookings/
    domain/            # entities, value objects, events, domain services; stdlib + typing only
      model.py
      errors.py        # domain exceptions (BookingOverlap, ...) mapped to problem types at the edge
    application/
      ports.py         # Protocols: BookingRepository, Clock, PaymentGateway, EventPublisher, UnitOfWork
      commands.py      # input DTOs (dataclasses or Pydantic)
      use_cases/       # one class or function per use case
    adapters/
      inbound/http/    # FastAPI routers, request/response schemas from the contract, dependency wiring
      inbound/consumers/
      outbound/db/     # SQLAlchemy tables/mappers, repositories, UnitOfWork
      outbound/redis/
      outbound/payments/
    config/            # settings, container / composition root, app factory
  migrations/          # alembic
  tests/unit | integration | api | contract
```

## 3. Practices

- **Domain models** as plain classes / `@dataclass(slots=True)` with behaviour; value objects frozen (`frozen=True`); ids as `NewType` or small value classes; money with `Decimal` and currency, never float; time-zone-aware `datetime` (UTC) via an injected `Clock`.
- **Pydantic** at the edges (HTTP schemas, settings, external API payloads), not as domain entities; map explicitly between schemas and domain objects.
- **Ports as `typing.Protocol`**; adapters implement them; tests use in-memory fakes.
- **Unit of Work** wrapping the SQLAlchemy session (one transaction per use case); repositories return domain objects; no session leaking into the application layer.
- **Dependency injection** via FastAPI `Depends` at the edge only, building use cases from a composition root; avoid global singletons.
- **Async**: use async end to end on I/O paths (asyncpg, httpx, redis.asyncio); never call blocking code in the event loop (offload with `anyio.to_thread.run_sync`); set timeouts everywhere (`asyncio.timeout`). Choose sync for CPU-bound or simple services — don't mix without reason.
- **Errors**: domain exceptions → one exception handler mapping to RFC 9457 problem details (`application/problem+json`) with stable `type` URIs and `code`; validation errors in the same shape with `errors[]`.
- **SQLAlchemy**: `select()` 2.0 style, explicit loading (`selectinload` / `joinedload`) to avoid N+1, `expire_on_commit=False` for async, pool sized for the DB's connection budget (and PgBouncer in transaction mode → disable prepared statement caching in asyncpg).
- **Alembic**: one migration per change, reviewed, reversible `downgrade()`, data migrations separate from schema migrations, `CREATE INDEX CONCURRENTLY` in an autocommit block.
- **Background work**: never in request handlers (`BackgroundTasks` only for trivial fire-and-forget); jobs idempotent with an idempotency key; scheduled jobs guarded against double execution.
- **Security**: `secrets` for tokens, `argon2-cffi` for password hashing (if the app handles passwords at all), OAuth2 token validation with PyJWT / authlib against the provider's JWKS (cache keys), CORS restricted to known origins.
- **Typing and lint**: no untyped defs, `Any` only at boundaries with a comment; Ruff rules including `B`, `S` (bandit), `UP`, `SIM`, `ASYNC`, `PERF`.
- **Testing**: pytest fixtures for a Testcontainers Postgres per session with migrations applied, a transaction per test rolled back; HTTP tests with `httpx.AsyncClient(transport=ASGITransport(app))`; schemathesis against the contract for the API surface.

## 4. Django (when chosen)

Keep the domain out of models when rules are rich (models as persistence adapters, or a services layer with explicit domain objects); Django Ninja or DRF serializers at the edge; `select_related` / `prefetch_related`; `transaction.atomic` per use case; `django-environ` / settings from env; migrations reviewed and squashed periodically.
