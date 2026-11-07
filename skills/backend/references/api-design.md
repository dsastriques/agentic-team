# REST and API design

Conventions shared with the frontend skill (its `references/api-and-data.md` consumes exactly these). The architect's `02-implementation.md` and ADRs override them where they differ.

## 1. Contract first

- OpenAPI 3.1 in `api_contracts`, one file per service or bounded context, reviewed like code; AsyncAPI for events.
- Every operation has: `operationId` (verb + noun, used by generated clients), summary, tags, security, parameters, request / response schemas with examples, every status code it can return, problem types.
- Changes are drafted with `x-status: proposed` and accepted by the architect; CI lints the contract (Spectral / Redocly rules) and fails on breaking changes against `main` (oasdiff), and checks the implementation matches (FastAPI's generated schema diff, generated Spring interfaces, schemathesis).
- Examples in the contract feed the frontend's MSW mocks.

## 2. Resources and URLs

- Nouns, plural, lowercase, kebab-case: `/bookings`, `/bookings/{bookingId}`, `/bookings/{bookingId}/items`. Nest at most one level; beyond that, top-level resources with filters.
- IDs opaque strings (UUIDs); never expose sequential internal ids if they leak business data.
- JSON fields `camelCase` (what the frontend uses) — or the convention in the ADR, consistently.
- Actions that aren't CRUD: model a sub-resource or a state change (`POST /bookings/{id}/cancellation`, or `PATCH` the `status` when the transition is simple); avoid RPC verbs in paths.
- Avoid screen-shaped endpoints in core services; a BFF composes for screens if the architecture has one. Do support what screens need: sparse fieldsets (`fields=`), embedding related summaries (`include=`) where it prevents N+1 from clients.

## 3. Methods and semantics

| Method | Semantics | Success |
|---|---|---|
| GET | safe, cacheable | 200 (`ETag`, `Cache-Control`) |
| POST (create) | not idempotent → support `Idempotency-Key` for side effects | 201 + `Location` + the resource |
| POST (action / async) | | 200 with result, or 202 + status resource for long work |
| PUT | full replace, idempotent | 200 / 204 |
| PATCH | partial update (JSON Merge Patch `application/merge-patch+json`, or JSON Patch) | 200 with the resource |
| DELETE | idempotent | 204 (second call 204 or 404 — choose and document) |

## 4. Status codes and errors

Errors are **RFC 9457** `application/problem+json`:

```json
{
  "type": "https://api.example.com/problems/booking-overlap",
  "title": "Items are not available for these dates",
  "status": 409,
  "detail": "Stroller S-12 is booked from 2026-12-03 to 2026-12-05.",
  "instance": "/bookings",
  "code": "booking_overlap",
  "requestId": "01JC…",
  "errors": [{ "field": "period.to", "code": "after_from", "message": "…" }]
}
```

- `type` and `code` are stable and documented (the frontend maps them to copy keys); `title` / `detail` are for humans and logs, not for UI logic.
- 400 malformed request · 401 unauthenticated (with `WWW-Authenticate`) · 403 not allowed · 404 not found (also for resources the caller may not see, to avoid leaking existence) · 409 conflict with current state / invariant · 412 `If-Match` failed · 415 · 422 semantic validation (field errors) · 429 with `Retry-After` · 500 unexpected (no stack traces) · 502 / 503 / 504 upstream or overload (with `Retry-After` when meaningful).
- Domain exceptions map to problem types in one handler; unknown exceptions → 500 with the request id, logged with the stack trace.

## 5. Pagination, filtering, sorting

- Cursor pagination by default: `GET /items?limit=20&cursor=<opaque>` → `{ "data": [...], "page": { "nextCursor": "…", "hasMore": true } }` (or `Link` headers — pick one per API). Cursors encode the keyset (base64 JSON), are opaque to clients and validated.
- Max `limit` enforced; default sensible (20).
- Filters as query params named after fields (`status=active`, `from=2026-12-01`); ranges with `from` / `to` or `[gte]`-style consistently; sorting `sort=-createdAt,name` with an allowlist.
- Totals only when needed and cheap (or estimated).

## 6. Idempotency and concurrency

- `Idempotency-Key` header on POSTs with side effects: store key + request fingerprint + response for 24 h (Redis or a table); replay returns the stored response; same key with a different body → 422; concurrent duplicate → 409 until the first completes.
- Optimistic concurrency: `ETag` (from `version`) on GET, `If-Match` required on PUT / PATCH / DELETE of user-editable resources → 412 on mismatch, 428 when missing (if required).

## 7. Data formats

- Dates and times ISO 8601 / RFC 3339 with offset (`2026-12-01T10:00:00Z`); dates without time as `YYYY-MM-DD`; durations ISO 8601.
- Money: `{ "amount": "49.90", "currency": "EUR" }` (decimal string) or minor units integer — one convention, documented.
- Enums as lowercase strings; clients must tolerate new values (document it).
- Nulls vs absent fields: define it (absent = not requested / unknown; `null` = no value).
- Localized content: `Accept-Language` → `Content-Language`; error `detail` may be localized but clients use `code`.

## 8. Versioning and evolution

- Additive changes (new optional fields, new endpoints, new enum values documented as extensible) don't need a new version.
- Breaking changes need a new version (`/v2/...` path or media-type version, per ADR), a deprecation period, `Deprecation` and `Sunset` headers on the old one, and a migration note for the frontend.
- Never repurpose a field.

## 9. Security

- OAuth2 / OIDC bearer tokens (or the BFF's session) validated on every request: signature (JWKS), `iss`, `aud`, `exp`, scopes.
- **Object-level authorization** in the use case for every resource id (OWASP API1 BOLA) and **property-level** control (API3): response schemas expose only allowed fields; request schemas reject unknown / read-only fields (mass assignment).
- Rate limits per client and per expensive endpoint (API4); pagination limits; request body size limits; timeouts.
- Validate all input against schemas; parameterized queries only; SSRF protection on any URL the user supplies (allowlist, no internal ranges).
- CORS allowlist; security headers; no sensitive data in URLs; webhook signatures (HMAC with timestamp) for inbound and outbound webhooks.
- Audit log for sensitive operations.

## 10. Async APIs and events

- Long-running work: `202 Accepted` + `Location: /operations/{id}`; the operation resource reports status and the result link; or webhooks / SSE when the architecture chooses them.
- Events (AsyncAPI): past-tense names (`booking.confirmed`), envelope with `id`, `type`, `occurredAt`, `source`, `subject`, `dataVersion` (CloudEvents-compatible), payload without sensitive data unless required; consumers idempotent by event `id`; schema evolution additive.

## 11. Answering a frontend request (`REQ-API-xx`)

Checklist: which use case and context owns it · existing endpoint extendable? · resource shape and fields the screen listed (no more) · filters, sort, pagination · status codes and problem `code`s the UI must distinguish · authz rule · caching / ETag · idempotency · performance target (NFR) and the index that supports it · examples for mocks · contract diff marked `proposed` · entry status updated.
