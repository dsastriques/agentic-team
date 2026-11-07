# SOLID, GoF and tactical patterns in backend code

Use a pattern when a requirement or a real change pressure asks for it; name the requirement in the code review or plan. Speculative abstraction is a defect.

## 1. SOLID, applied

| Principle | In practice | Smell |
|---|---|---|
| **S** — single responsibility | one use case per class / function; controllers translate HTTP only; repositories persist only | `BookingService` with 25 methods; handlers with business rules |
| **O** — open / closed | new behaviour by adding a strategy / handler / adapter, not editing a switch in many places | `if provider == "stripe" … elif …` repeated across files |
| **L** — Liskov substitution | every adapter honours the port's contract (errors, ordering, idempotency); fakes behave like the real thing | an in-memory repo that never raises "not found" |
| **I** — interface segregation | small ports per need (`LoadBooking`, `SaveBooking`) or role interfaces; clients depend on what they use | one `Repository` with 30 methods injected everywhere |
| **D** — dependency inversion | application owns ports; adapters implement them; wiring in the composition root | domain importing SQLAlchemy / JPA / HTTP clients |

Also: composition over inheritance, tell-don't-ask, law of Demeter for aggregates, explicit dependencies (constructor injection), pure functions for calculations (pricing, availability).

## 2. GoF patterns that earn their place in services

| Pattern | Typical backend use | Notes |
|---|---|---|
| **Strategy** | pricing rules, payment providers, notification channels, LLM providers | select by config or domain data via a registry |
| **Factory / Factory Method** | building aggregates with invariants (`Booking.create(...)`), choosing an adapter per tenant | keep construction rules in the domain |
| **Builder** | complex test data (test data builders), request objects for external APIs | prefer records / dataclasses with defaults for simple cases |
| **Adapter** | every outbound port implementation; anti-corruption layer for external APIs | map external models to domain types at the boundary |
| **Facade** | a simple port over a complex SDK (payments, search, LLM) | |
| **Decorator** | cross-cutting around ports: caching repository, retrying client, metrics, tracing, authorization checks | stackable; keeps the core clean |
| **Proxy** | lazy loading, remote access, rate-limited clients | |
| **Template Method** | shared job skeleton (load → process → mark done) | prefer composition (strategy) when variants multiply |
| **Chain of Responsibility** | middleware / filters, validation pipelines, LLM guardrail chains | |
| **Command** | use case inputs, queued jobs, undoable operations, agent tool calls | serializable, idempotency key on the command |
| **Observer** | domain events → handlers | in-process for the same context, outbox + broker across contexts |
| **State** | aggregates with lifecycles (booking: requested → confirmed → active → returned / cancelled) | transitions as methods guarded by invariants; matches the designer's flow state machines |
| **Specification** (DDD) | composable business rules and query filters (`AvailableFor(dates)`) | |
| **Composite** | price breakdowns, permission trees, nested menus | |
| **Singleton** | avoid; let the DI container manage lifetimes | |

## 3. Tactical DDD in code

- **Aggregates**: one transaction changes one aggregate; reference other aggregates by id; keep them small; invariants checked in methods, never left to callers.
- **Value objects**: immutable, validated on creation, equality by value (`Money`, `DateRange`, `Email`).
- **Domain events**: past-tense names (`BookingConfirmed`), raised by aggregates, dispatched after commit (in-process) or via the outbox (cross-context).
- **Domain services**: rules that span aggregates and belong to no single one (availability across items).
- **Application services / use cases**: orchestrate — load, call domain, save, publish; no business rules.
- **Repositories**: per aggregate, collection-like, return domain objects.
- **Policies / process managers / sagas**: react to events and issue commands; compensations for multi-step processes.

## 4. Enterprise and integration patterns

- **Transactional outbox** + relay (polling or CDC) for reliable event publishing; **idempotent consumer** (processed-message table or Redis set with TTL).
- **Saga** (orchestrated for complex flows, choreographed for simple ones) with compensations; persist saga state.
- **CQRS-lite**: separate read models / queries (jOOQ, SQL views, materialized views) when reads differ from aggregates; full CQRS / event sourcing only with an ADR.
- **Circuit breaker, retry with exponential backoff and jitter, timeout, bulkhead** around every remote dependency.
- **Anti-corruption layer** for legacy or external systems.
- **Unit of Work** for transaction scope.
- **Result types** for expected failures in the application layer (sealed results / discriminated unions) vs exceptions for unexpected ones — pick one convention per codebase (G-xx).

## 5. Refactoring and code health

- Small, behaviour-preserving steps with tests green; extract method / class, introduce parameter object, replace conditional with polymorphism, move rules into the domain.
- Function size and nesting kept small; names from the ubiquitous language; no comments explaining what code does — explain why.
- Delete dead code; no feature-flagged code left after rollout.
