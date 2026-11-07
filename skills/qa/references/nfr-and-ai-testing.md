# Non-functional and AI testing

Every NFR test traces to an `NFR-xx` with a number. If the NFR has no number, ask the architect; don't invent thresholds.

## 1. Performance (k6 by default)

Scenarios in `tests/perf/`, one per journey or API group, with thresholds from the NFRs / SLOs (e.g. `http_req_duration{group:checkout}: p(95)<300`, `http_req_failed: rate<0.001`).

| Test | Shape | Where | When |
|---|---|---|---|
| Smoke | 1–5 VUs, 1 min | CI-ephemeral or preview | every PR touching the backend |
| Load | expected peak (from NFRs), 15–30 min | staging or prod-clone | before releases with backend changes, weekly |
| Stress | ramp past peak until errors | prod-clone / staging | before launch, after architecture changes |
| Spike | sudden 5–10× | staging / prod-clone | when the NFRs mention bursts (campaigns, events) |
| Soak | expected load, 2–8 h | prod-clone / staging | before launch, quarterly; finds leaks and pool exhaustion |

- Data: realistic volume in the DB before the test (pagination and query plans change with size); unique users per VU.
- Results on staging-sized infra are extrapolated with care and labelled; capacity claims need prod-clone.
- Report p50 / p95 / p99, error rate, throughput and the bottleneck observed in devops' dashboards (CPU, DB connections, cache hit rate) — attach the dashboard links for the test window.
- Browser-side performance: the frontend's Lighthouse CI and bundle budgets; k6 browser for a few critical journeys under load if needed.
- Load generation from the decided region; never against production without the owner's approval and devops coordination.

## 2. Resilience

- Dependency failures with Toxiproxy (latency, timeouts, resets) in CI-ephemeral or staging: the app degrades as designed (timeouts, retries only on idempotent operations, circuit breaker, fallback UI states).
- Pod / instance kills and zone failure simulation (Chaos Mesh, Litmus, or cloud fault injection) only in staging or prod-clone, scheduled with devops.
- DR: devops' `dr-drill` measures RPO / RTO; you verify the app works on the restored environment with the smoke suite.

## 3. Security testing (in addition to devops' SAST, dependency and image scanning)

- **Authorization matrix and IDOR** (see `api-and-contract.md` §4) — every rung.
- **OWASP ZAP**: baseline scan (passive) on previews; API scan from the OpenAPI and authenticated full scan on staging, nightly or pre-release. Findings triaged with false-positive rules committed and reasons; real findings → `BUG-xx` to the owner.
- **Business-logic abuse** in the test plan for critical journeys: negative amounts, replaying requests, skipping steps, changing ids in the URL, rate limits on login / OTP / password reset.
- **Session and auth flows**: logout invalidation, token expiry, refresh, password reset token single-use, cookie flags.
- Pen tests by third parties are an owner decision; you prepare scope and environment.

## 4. Accessibility

- axe-core in web e2e (`@axe-core/playwright`) on every critical screen state; mobile: accessibility labels asserted in flows, largest text size and screen-reader smoke on critical screens.
- Automated checks find roughly a third to half of WCAG issues; the designer's manual review covers the rest. Failures are `BUG-xx` with the WCAG criterion.

## 5. Visual regression

- Only for design-critical, stable surfaces (design-system components in Storybook, key landing screens). Tooling per config: Playwright `toHaveScreenshot`, Vitest 4 visual tests for components, Chromatic / Argos when the frontend runs Storybook-driven reviews.
- Baselines generated in the CI container, dynamic content masked, diffs reviewed by the frontend and the designer.

## 6. AI features (LLM calls, agents, RAG)

The backend implements AI features; you own their evaluation suite with them.
- **Golden dataset** per feature in `tests/evals/<feature>/`: inputs with expected properties (not exact strings): required facts, forbidden content, structure (JSON schema), tool calls expected, refusal expected. Built from requirements and, with the owner's approval, anonymized real queries.
- **Checks**, cheapest first: deterministic (schema, regex, tool-call names and arguments, length, latency, cost per call) → reference-based (similarity to an expected answer) → rubric-based LLM-as-judge with a fixed judge model and a written rubric, calibrated against a small human-labelled set.
- **Safety cases**: prompt injection through user content and retrieved documents, data exfiltration attempts, jailbreak patterns relevant to the product, PII leakage, off-topic requests.
- **Agents**: task success rate on scripted scenarios, step limits, tool misuse, behaviour when tools fail.
- **Regression**: run the eval suite on every prompt, model, retrieval or tool change; report pass rate per category vs the baseline, with thresholds the owner agrees (they're product decisions). Tools per config: promptfoo, DeepEval or Inspect.
- Cost: evals call paid models — budget them with the owner, cache responses for deterministic reruns, run the full set on change and nightly, a small smoke set per PR.

## 7. Mutation testing (audit, not a gate)

Periodically (or on critical modules) run Stryker (TS / JS), PIT (Java) or mutmut (Python) on domain code. Surviving mutants in business-rule code are review findings for the owner of the code: missing assertions, not missing coverage.
