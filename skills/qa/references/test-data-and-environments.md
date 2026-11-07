# Test data and environments

## 1. Test data principles

- **Create, don't share.** Each test creates what it needs through factories, with unique identifiers (run id + worker id + counter). Shared fixtures are read-only reference data (countries, plans, feature flags).
- **Through the front door.** Factories call the public API (or the backend's seed hook when the API can't create the state, e.g. a booking in the past). Direct DB inserts only in backend integration tests, never in e2e.
- **Reset by design, not by luck.** Per environment: local and CI-ephemeral → recreate the database; previews → seeded on creation, destroyed with the PR; staging → tenant-per-run or cleanup by run id plus a scheduled sweep; never a "reset staging" that breaks other people's runs.
- **Synthetic by default.** Faker-style generators with locale-aware data (names, addresses, IBANs in valid test formats, phone numbers from reserved ranges), deterministic seeds so failures reproduce.
- **No real personal data.** GDPR applies to test environments too. Production-derived data needs the owner's explicit approval, anonymization (not just masking of obvious fields — free text, addresses, rare combinations re-identify people), access control and retention like production. Record the decision in `00-strategy.md`.
- **Edge data on purpose**: long names, non-Latin scripts, RTL text if supported, emoji, zero / negative / max amounts, empty collections, thousands of items for pagination and performance.

## 2. Testability requests (`TEST-REQ-xx`)

Ask the backend / frontend in `docs/qa/testability-requests.md`:

| id | to | need | why (TS / risk) | environments | status |

Common needs:
- Seed and reset endpoints or CLI commands, enabled only in non-production builds / environments, protected by a test-only credential.
- Test users per role created by seed, with secrets stored where devops says.
- Clock control (an injectable clock with a test override header or endpoint in non-prod).
- Deterministic ids or a way to look up created resources by a client-supplied reference.
- Email / SMS capture (Mailpit or provider sandbox) and an API to read captured messages.
- Feature-flag overrides per request or per test user.
- Stable `testID`s and accessible names (frontend), MSW handler sets per screen state.
- Webhook replay or signed fixture support for third-party callbacks.

Statuses: `proposed` → `accepted` → `done` / `rejected` (with an alternative).

## 3. Environment types

| Environment | Good for | Data | Owner of decision |
|---|---|---|---|
| Local stack (Compose / devcontainer from devops) | developing tests, reproducing bugs | seeded locally | devops (exists already) |
| CI-ephemeral (services in the CI job: Testcontainers, Compose) | API system tests, e2e against the built images on every PR | seeded per job | devops builds, you specify |
| Preview per PR | e2e against the real deployed stack per change | seeded on create | architect decides, devops builds |
| Staging (persistent) | full regression, cross-browser / device, NFR, third-party sandboxes end to end | seeded baseline + per-run tenants | architect decides, devops builds |
| Prod-clone (on demand) | load / soak at scale, DR, release rehearsal | synthetic at scale or approved anonymized snapshot | architect decides, owner approves cost, devops builds |
| Production | post-deploy smoke, synthetic monitoring | dedicated synthetic accounts, read-only or self-cleaning | strategy + owner |

Parity matters more than size: the same images (by digest), migrations, config shape and auth provider type as production; differences documented in `00-strategy.md`.

## 4. Requests to devops

In `docs/devops/requests.md`, one entry per need:
- environment(s), purpose (suite and TS / risk it serves), lifetime (TTL, schedule)
- commands to run, environment variable names, secret names (never values)
- seed / reset hook wiring, test user provisioning
- runners (browsers, Android emulators with KVM, macOS for iOS, load generators), parallelism
- reports and artifacts: format, retention, where they're published
- cost ceiling from the approach decision

## 5. Third parties

- Use provider sandboxes (Stripe test mode with test cards and test clocks, payment webhooks via the provider's CLI or signed fixtures, email provider sandboxes, SMS test numbers).
- Virtualize when no sandbox exists or it's unreliable; contract-test the virtual service against the provider's published spec.
- Never call a production third-party endpoint from a test environment.
