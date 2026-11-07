# API system tests and contracts

## 1. Who tests what

| Level | Owner | Where | Against |
|---|---|---|---|
| API per endpoint (handler, validation, status codes) | backend | service repo / module | the service with Testcontainers |
| Provider conformance to the OpenAPI | backend (wired by you in setup) | service CI | the running service |
| Consumer mocks match the OpenAPI | frontend (validation fixture by you) | web / mobile test setup | the contract file |
| Cross-service / journey-level API system tests, authz matrix | **qa** | `tests/api/` | a running environment (local stack, preview, staging) |
| Property-based / fuzz from the contract | **qa** (Schemathesis) | CI against preview or local stack | the running API |

## 2. Contract strategy

**Default — OpenAPI as the published contract** (fits one backend owning its API, as in this team where the architect owns the contracts):
1. Contracts live in `docs/architecture/api/*.yaml` (architect-owned; backend proposes changes as `x-status: proposed`).
2. **Breaking-change check**: `oasdiff breaking <base> <head>` on every PR that touches a contract; breaking changes need a version bump or an ADR.
3. **Provider side**: the backend validates requests / responses against the contract in its API tests (or Schemathesis), and CI fails on drift between generated and published spec.
4. **Consumer side**: MSW handlers and Playwright route mocks are generated from the contract (typed) and every mocked response is validated against the schema at test time (e.g. Ajv or an OpenAPI response validator in a shared fixture). A mock field the contract doesn't have fails the test.
5. With 3 and 4 both green, mocked frontend tests and real backend tests are talking about the same API — the property that makes the Lean rung safe.

**Pact (consumer-driven)** when: several consumers with different needs, services owned by different teams, or async messages between services where the provider can't see consumer usage. Then: consumer tests publish pacts to a broker, providers verify in CI, `can-i-deploy` gates promotion. It's an ADR-level choice; propose it to the architect with the reason.

**AsyncAPI / events**: schema validation of produced and consumed messages with the same producer / consumer split; test idempotent consumers by redelivering the same message.

## 3. API system tests (qa)

- Black-box over HTTP with the project's language choice (pytest + httpx for Python services, REST Assured / JUnit 5 for Java, or Playwright `request` to share fixtures with web e2e).
- Organised by journey (`tests/api/<feature>/`), named with TS / BR ids, data via factories with unique ids.
- Cover the team's API conventions explicitly:
  - **Errors**: RFC 9457 problem details, stable `code` per error, correct status, no stack traces or internal ids leaked.
  - **Pagination**: cursor stability under inserts, last page, invalid cursor, page size limits.
  - **Idempotency**: same `Idempotency-Key` + same body → same result, no duplicate side effect; same key + different body → the documented error.
  - **Concurrency**: `ETag` / `If-Match` → 412 on stale writes; double submit races on critical operations.
  - **AuthN / AuthZ**: the authorization matrix (§4).
  - **Validation**: boundaries, types, unknown fields per the contract's policy, Unicode, very long strings.
  - **Time**: time zones and DST boundaries for anything scheduled; controlled clock through the backend's test hook when needed.
- Webhooks from third parties (payments, email): replay sandbox events or signed fixtures; test signature failures and duplicates.

## 4. Authorization matrix

For every endpoint (or use case) and every role, including anonymous and "other tenant's user": expected allow / deny. Generate parameterized tests from a table in `tests/api/authz/matrix.yaml`. IDOR checks: user A requesting user B's resource by id must get 404 or 403 per the design. This is one of the cheapest high-value suites; it belongs in every approach rung.

## 5. Schemathesis

- Run against a seeded environment with an auth hook; start with `--checks all` on the PR's affected operations and the full spec nightly.
- Findings are triaged: server errors (5xx) and response-schema violations are bugs; contract gaps (undocumented status codes) go to the architect.
- Stateful testing (links between operations) for CRUD resources once the basic run is clean.

## 6. Service virtualization

Prism (mock server from OpenAPI) or WireMock for third parties or services not deployed in an environment. Virtualized responses are recorded or derived from the provider's published spec, not invented; mark which suites run against virtual services in reports.
