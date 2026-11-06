# Consuming REST APIs and working with the backend

The architect owns API contracts (OpenAPI / AsyncAPI in `02-implementation.md` and `api_contracts`); the backend skill implements them; the frontend consumes them and asks for what screens need. This reference covers both the consumption practices and the collaboration.

## 1. Contract-first, typed end to end

- Generate types (and optionally hooks) from the OpenAPI in CI: `openapi-typescript` + `openapi-fetch` (thin, typed `fetch`), or `orval` / `@hey-api/openapi-ts` generating TanStack Query `queryOptions` / hooks. Generated code lives in `packages/api-client` and is never edited by hand.
- A contract change regenerates the client; type errors show every affected screen.
- Map DTOs to view models in each feature's `api/` layer (dates parsed, money normalized, enums mapped); components never see raw DTOs. This is the anti-corruption layer that keeps UI stable when the API evolves.
- MSW handlers generated or written from the same contract (and its examples) for unit tests, Storybook and local development before the backend is ready.
- Where the contract isn't decided: a typed stub with `// TBD-ARCH: <question>` and an entry in `docs/frontend/api-requests.md`. Where it's decided but not live: generated types plus MSW, marked `TBD-BE` in the report.

## 2. The fetch wrapper (one per app)

Responsibilities, in one place:
- base URL per environment; `Accept: application/json`; `Content-Type` for bodies;
- auth (cookie credentials or bearer token from the auth adapter) and the refresh flow;
- `AbortSignal` from TanStack Query (cancellation) plus a timeout (`AbortSignal.timeout`, combined with `AbortSignal.any`);
- a correlation / request id header (`X-Request-Id` or `traceparent` when tracing is set up) logged with errors;
- `Accept-Language` from the active locale when the API localizes;
- parsing errors into a typed `ApiError` from **RFC 9457 problem details** (`type`, `title`, `status`, `detail`, `instance`, plus extension members like `errors` for field validation and a stable `code`);
- no business logic, no UI.

## 3. Status codes → behaviour

| Status | Frontend behaviour |
|---|---|
| 200 / 201 / 204 | success; 201 may return `Location` and the created resource — use it to seed the cache |
| 304 | handled by the HTTP cache with `ETag` / `If-None-Match` |
| 400 / 422 | validation: map field errors to form fields and their `error_keys`; no retry |
| 401 | try one token refresh, replay once; otherwise sign out and go to sign-in with a return URL |
| 403 | no-permission state from the contract; no retry |
| 404 | not-found state (or the screen's empty state if the contract says so) |
| 409 | conflict: refetch and show the conflict message; for optimistic updates roll back |
| 412 | precondition failed (stale `If-Match`): refetch, tell the user, let them retry |
| 429 | respect `Retry-After`; show a "try again shortly" message for user actions |
| 5xx / network | retry idempotent reads with backoff (TanStack Query); show the error state with retry; for mutations show an error and keep the user's input |

Map each problem `type` / `code` to a message key; unknown errors fall back to a generic message plus the request id for support.

## 4. Mutations done right

- **Idempotency**: for non-idempotent POSTs with side effects (payments, bookings, orders), send an `Idempotency-Key` generated once per user intent (kept across retries and double clicks) when the API supports it; ask the architect to add it when it doesn't.
- Disable the submit control only while the request is in flight; prevent double submit.
- Concurrency: send `If-Match` with the resource's `ETag` on updates when the API supports optimistic locking.
- `PUT` replaces, `PATCH` partially updates (JSON Merge Patch or JSON Patch per the contract); send only what the user changed for `PATCH`.
- Never retry non-idempotent mutations automatically unless they carry an idempotency key.

## 5. Reading data

- **Pagination**: prefer cursor-based (`next` cursor or `Link` headers) with `useInfiniteQuery`; offset only for small, stable lists with page numbers in the URL.
- Filtering, sorting and searching as query params mirrored in the URL search params; debounce free-text search (250–300 ms) and cancel stale requests.
- Ask for what the screen needs: sparse fieldsets or a BFF / screen-oriented endpoint when a screen would otherwise call many endpoints (N+1 from the client) — propose it in `api-requests.md`.
- Dates in ISO 8601 with offset; money as minor units or decimal strings with a currency code; IDs as strings.
- Real-time needs (SSE, WebSocket) are an architect decision; on events, update or invalidate the matching query keys.

## 6. Authentication and security

- Web: prefer a BFF or same-site session with `httpOnly`, `Secure`, `SameSite` cookies; tokens never in `localStorage` / `sessionStorage`. If the architecture uses OIDC in the browser, Authorization Code + PKCE, tokens kept in memory, refresh via the provider's recommended mechanism.
- CSRF protection for cookie-authenticated mutations (SameSite plus a CSRF token or custom header, per the architect).
- Native: OIDC with PKCE through the system browser (`expo-auth-session`), tokens in `expo-secure-store` / Keychain / Keystore, refresh with rotation.
- Never put secrets or API keys with privileges in the client; public keys only (e.g. a publishable payment key).
- Payments: use the provider's client SDK or hosted fields so card data never touches the app's code.
- Sanitize any HTML from the API (DOMPurify) before rendering; validate redirect URLs against an allowlist; set a CSP compatible with the app (no inline scripts).
- No PII in logs, analytics events or URLs.

## 7. Offline and resilience

- Detect connectivity (`navigator.onLine` plus failed requests; NetInfo on native) and show the contract's offline state.
- Cache-first reads where the requirements allow (TanStack Query persistence); queued mutations only where the requirements say so, with conflict handling agreed with the architect.
- Timeouts tuned per endpoint (e.g. 10 s default, longer for uploads); uploads with progress and resumability (tus or multipart) when files are large.

## 8. Collaborating with the backend

`docs/frontend/api-requests.md` — one entry per need, so the architect can update the contract and the backend can implement it:

```markdown
### REQ-API-03 — List available rental items for dates
- needed by: SCR-search-results (FR-04, UX-07)
- status: proposed | accepted (contract updated) | implemented | rejected
- endpoint (proposal): GET /items?from=2026-12-01&to=2026-12-08&category=stroller&cursor=…
- response fields used: id, name, thumbnailUrl, pricePerDay {amount, currency}, available
- errors the UI must distinguish: 400 invalid date range (code: invalid_range), 422 dates in the past
- paging: cursor, page size 20
- performance: p95 ≤ 300 ms (NFR-02)
- notes: ...
```

Contract tests: when the architect sets up consumer-driven contracts (e.g. Pact) the frontend publishes its expectations from the MSW / client usage; otherwise e2e against staging covers the integration.
