# Test approach and budget

How to build the approach options in Strategy mode and agree one with the owner.

## 1. The ladder

Each step adds what the previous one can't catch, and costs more. Options presented to the owner are usually two adjacent rungs plus the recommendation; tailor each to the product.

| | **Lean** | **Previews** | **Staging** | **Prod-clone** |
|---|---|---|---|---|
| Idea | Mock the boundary, prove the boundary | Real stack per PR, short-lived | One persistent production-like environment | Production-sized copy for load, DR and release rehearsal |
| Frontend | Component + integration tests with MSW (mocks validated against OpenAPI); e2e of critical journeys against the mocked API or a local stack | Lean + e2e critical journeys on the PR's preview | Previews (if any) + full regression e2e nightly / pre-release, cross-browser | Staging + release rehearsal |
| Backend | Unit, integration (Testcontainers), API per endpoint, conformance to OpenAPI (Schemathesis) | Lean + API system tests on the preview | + cross-service API system tests, authz matrix, ZAP API scan | + load / soak at production scale, DR restore tests |
| Mobile | Maestro on emulator in CI against a mock or local API | + flows against the preview API | + real-device cloud matrix nightly | + release candidate on store-like builds across the full matrix |
| Perf | k6 smoke in CI (single VU, thresholds on latency) | k6 smoke on preview | k6 load / stress on staging, scaled-down targets extrapolated | load, stress, soak at real scale |
| Catches | Logic, contract drift, UI states, most regressions | + integration and deploy config of the real stack, migrations, real auth flow | + environment-specific config, third-party sandboxes end to end, data volume effects, cross-browser / device | + capacity, scaling, failover, real data shapes (anonymized) |
| Misses | Real infra config, auth provider quirks, CDN / headers, migrations on real data, third parties | Long-lived data drift, scale, real devices | Scale, real-size data, rare failovers | Little; production itself (shift-right covers that) |
| Cost drivers | CI minutes only | Preview compute and DB per open PR × TTL | 24/7 environment (often scaled to zero or down off-hours) + device cloud | Production-scale infra while it runs (on demand, hours not months) + data anonymization pipeline |
| Typical fit | Pre-launch, solo / small team, tight budget | Small team shipping often, moderate budget | Paying customers, several services, mobile on many devices | Strict SLOs, regulated, high traffic, contractual DR |

Rungs combine: "Lean + staging nightly, no previews" is valid. Prod-clone is almost always on demand (spun up for a load test or DR drill and torn down), never 24/7.

## 2. What the options must show the owner

For each option, in one table row or short block:
- **What runs where** (level × environment × trigger).
- **Residual risk** in concrete terms tied to their critical journeys ("a payment webhook misconfiguration would reach production undetected").
- **Monthly cost**: environments (from devops, or a public-pricing estimate marked `ASSUMPTION`), device cloud, tool licences, CI minutes. Give a range, not false precision.
- **Speed**: time from push to the PR gate result; time for full regression.
- **Effort**: maintenance load (e2e suites and environments both decay), who maintains what.
- **Upgrade trigger**: the event that should move them up a rung.

Then one recommendation with the reason. Keep it to one screen.

## 3. Cost questions for devops

Send as a request in `docs/devops/requests.md` (or ask the devops subagent directly):
- Preview per PR: compute + database per preview per hour on the decided provider, expected open PRs, TTL → €/month.
- Staging: always-on vs scheduled off-hours vs scale-to-zero; managed DB tier; €/month.
- Prod-clone on demand: €/hour at production size; time to provision and seed.
- CI: minutes per month per suite tier; larger runners; artifacts storage.
- Device cloud and load-generation runners if the provider offers them.

Devops supplies facts; neither you nor devops picks the hosting — the architect records environments after the owner chooses the approach.

## 4. Levers that cut cost without cutting confidence

- **Contract testing** (validated mocks + backend conformance) is what makes the Lean rung trustworthy; invest there before buying environments.
- **Test impact analysis / affected-only runs** in monorepos.
- **Smoke on PR, full regression nightly**; cross-browser and device matrices nightly, not per PR.
- **Preview TTL** and auto-teardown on merge; scale-to-zero staging off-hours.
- **Emulators for breadth, real devices for depth**: a small real-device matrix nightly, emulators per PR.
- **On-demand prod-clone** for load and DR, torn down after.
- **Shift-right**: post-deploy smoke, synthetic monitoring, feature flags and canaries catch what pre-prod can't, often cheaper than another environment.

## 5. Questions to ask the owner (one batch, max 4)

1. Which approach (options, recommendation first).
2. Monthly budget ceiling for test environments and tools.
3. Real-device testing: device cloud subscription yes / no (and which platforms matter most).
4. Production-derived data: never (default) / anonymized snapshots allowed for load and data-shape tests.

## 6. Revisit triggers

Record them with the decision: first paying customers, an escaped S1 bug, a new platform (mobile, new market), a new critical integration (payments, health data), budget change, team growth, SLO tightening, a regulator or contract asking for evidence.
