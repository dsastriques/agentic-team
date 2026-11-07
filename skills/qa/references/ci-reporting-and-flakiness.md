# CI, reporting and flakiness

Devops builds and runs the pipelines; you specify suites, triggers, environments and reports (requests in `docs/devops/requests.md`) and own the results.

## 1. Suite tiers

| Tier | Trigger | Budget (wall time) | Contents |
|---|---|---|---|
| PR gate | every PR | ≤ 10–15 min | backend / frontend unit + integration (theirs), contract checks (oasdiff, mock validation, provider conformance on affected operations), API system smoke, web e2e `@smoke` + impacted `@critical` (Chromium), mobile `smoke` on one emulator if mobile changed, k6 smoke if backend changed, axe in those e2e |
| Merge / main | merge to `main` | ≤ 30 min | full `@critical`, Schemathesis on affected operations, ZAP baseline on the preview / dev deploy |
| Nightly | schedule | hours ok | full regression: all browsers and viewports, device matrix, full Schemathesis, ZAP API scan, AI eval full set, visual |
| Release | release candidate | as needed | regression on staging (or the approach's top environment), load test, release report |
| Post-deploy | after each production deploy | ≤ 5 min | production smoke with synthetic accounts; synthetic monitoring keeps running |

Required checks for merge are the PR gate suites listed in `04-testing.md`; anything else reports but doesn't block unless the strategy says so.

## 2. Speed

- Sharding (Playwright `--shard`, parallel Maestro runs per device), merged reports.
- Affected-only: path filters and the monorepo tool's affected graph (turborepo / nx) to pick suites; tag-to-path mapping for e2e (`@feature-bookings` ↔ `apps/web/src/features/bookings/**`). Always run `@smoke` regardless.
- Build once: e2e runs against the images / app builds the pipeline already produced, never a rebuild.
- Cache browsers and dependencies; reuse auth state; seed via API.

## 3. Reports and artifacts

- JUnit XML from every suite (for the CI summary and history), Playwright HTML with traces, Maestro / Detox videos and logs, k6 summary JSON and dashboard links, ZAP HTML / JSON, eval reports. Optionally Allure or CTRF to aggregate.
- A PR comment / job summary: per suite pass / fail / flaky / skipped counts, new failures vs main, links to traces.
- Retention per config (default 14 days; release reports kept with the release record).
- Test history (pass rate, duration, flakiness per test) from the report data or the CI's test analytics; reviewed weekly in `triage flaky`.

## 4. Flakiness policy

- **Definition**: a test that both passes and fails on the same code. CI retries once (`retries: 1`); a pass after retry is recorded as flaky, never as clean.
- **Quarantine**: tag `@quarantine` (or move to a quarantine project / Maestro tag) so it runs but doesn't block; each entry has an owner, a linked issue, a root-cause guess and an expiry (default 14 days). At expiry: fixed, rewritten at a lower level, or deleted with the owner's agreement and the coverage gap recorded.
- **Budget**: quarantined tests stay under `max_quarantined` (default 5 % of the suite); over budget, fixing flakes takes priority over new e2e tests.
- **Common causes and fixes**: fixed waits → web-first assertions; shared data → per-test factories; order dependence → isolate; animations → reduced-motion in test builds or waiting for the end state; time → controlled clock; third parties → sandbox or virtualize; environment instability → `OPS-xx` to devops with evidence.
- Never raise retries or timeouts globally to hide flakes.

## 5. Production checks (shift-right)

- Post-deploy smoke: a handful of read-only or self-cleaning journeys with dedicated synthetic accounts, run by the pipeline after each deploy; failure triggers devops' rollback procedure.
- Synthetic monitoring of critical journeys every few minutes (Grafana Synthetic Monitoring, Checkly, or the cloud's equivalent — whatever the observability decision includes), alerting through devops' routing with a runbook link.
- Escaped bugs: every production bug gets a regression test at the lowest level that would have caught it, and a note in `00-strategy.md` if it reveals a gap in the approach (a revisit trigger).
