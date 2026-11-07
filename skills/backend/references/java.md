# Java backend

Defaults when the architect hasn't decided otherwise; check current stable versions before scaffolding.

## 1. Stack

| Need | Default | Alternatives |
|---|---|---|
| Runtime | Java 21+ LTS (current LTS) | |
| Build | Gradle (Kotlin DSL, version catalog) | Maven |
| Framework | Spring Boot (current major) — Web MVC with virtual threads | WebFlux only for streaming / very high concurrency needs; Quarkus or Micronaut for fast startup / native images |
| Modularity | Spring Modulith (module boundaries, events, docs) for modular monoliths | Gradle multi-module |
| Persistence | Spring Data JPA / Hibernate for aggregates | jOOQ for complex reads and reporting; Spring JDBC `JdbcClient` for simple SQL |
| Migrations | Flyway | Liquibase |
| Validation | Jakarta Bean Validation at the edge; invariants in the domain | |
| Mapping | MapStruct or hand-written mappers | |
| HTTP client | Spring `RestClient` / declarative HTTP interfaces, with timeouts | |
| Resilience | Resilience4j (circuit breaker, retry, bulkhead, rate limiter, time limiter) | |
| Security | Spring Security OAuth2 Resource Server (JWT, JWKS) | |
| API docs | contract-first: openapi-generator interfaces (`interfaceOnly`) or springdoc + contract diff in CI | |
| Tests | JUnit 5, AssertJ, Mockito, Testcontainers (Postgres, Redis, brokers) with Spring Boot service connections, MockMvc / `RestTestClient` / RestAssured, ArchUnit, Spring Cloud Contract or Pact for contracts | |
| Observability | Micrometer + OpenTelemetry, Actuator health groups (liveness / readiness), structured JSON logging | |
| Quality | Spotless (google-java-format / palantir), Error Prone + NullAway, Checkstyle where required | |

## 2. Layout (hexagonal, per module)

```
bookings/
  src/main/java/com/acme/bookings/
    domain/                 # aggregates, value objects (records), domain events, domain services; no Spring/JPA imports
    application/
      port/in/              # use case interfaces (CreateBookingUseCase)
      port/out/             # BookingRepository, PaymentGateway, EventPublisher, Clock
      service/              # use case implementations (@Transactional here)
    adapter/
      in/web/               # controllers implementing generated API interfaces, DTO mappers, ProblemDetail handler
      in/messaging/
      out/persistence/      # JPA entities (separate from domain) or jOOQ, repository adapters
      out/payments/
    config/                 # @Configuration, wiring, properties (@ConfigurationProperties records)
  src/test/java/...         # unit, integration (Testcontainers), web, ArchUnit rules
  src/main/resources/db/migration/   # Flyway V001__create_booking.sql
```

ArchUnit tests enforce: `domain` depends on nothing outside `java.*` and itself; `application` doesn't depend on `adapter`; controllers only call `port.in`.

## 3. Practices

- **Records** for value objects, DTOs and commands; **sealed interfaces** + pattern matching for result types and domain events; immutable by default.
- **Domain separate from JPA entities** when invariants are rich (map in the persistence adapter); for simpler contexts JPA-annotated aggregates are acceptable if an ADR or G-xx allows it. No setters exposing invariants; behaviour methods instead.
- **Transactions** on application services (`@Transactional`), never on controllers; read-only transactions for queries; keep transactions short, no remote calls inside them (use the outbox).
- **JPA pitfalls**: `FetchType.LAZY` everywhere, fetch joins / entity graphs per use case to avoid N+1, no open-session-in-view (`spring.jpa.open-in-view=false`), `@Version` for optimistic locking (maps to ETag / If-Match), batch inserts configured, avoid `CascadeType.ALL` on large collections.
- **Virtual threads** (`spring.threads.virtual.enabled=true`) for blocking I/O; watch pinning in `synchronized` blocks with blocking calls on older JDKs, and size the DB pool for the database, not for the thread count.
- **Errors**: `ProblemDetail` (RFC 9457) via `@RestControllerAdvice` / `ResponseEntityExceptionHandler`, stable `type` URIs and `code` property; validation errors with field list.
- **Configuration**: `@ConfigurationProperties` records with validation; profiles only for local / test; secrets from the environment or a secret manager.
- **Events**: Spring Modulith's event publication registry (outbox) or an explicit outbox table + relay; listeners idempotent.
- **Null safety**: JSpecify annotations + NullAway; `Optional` for return values only.
- **Testing**: slice tests (`@WebMvcTest`, `@DataJpaTest` with Testcontainers) and a few full `@SpringBootTest` flows; `@ServiceConnection` for containers; reuse containers across tests; no H2 as a Postgres substitute.
- **Performance**: HikariCP pool sized deliberately, statement timeouts, `jOOQ` for heavy reads, caching via Spring Cache with Redis only where measured; JFR for profiling.
