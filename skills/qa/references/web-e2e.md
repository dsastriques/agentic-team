# Web e2e with Playwright

## 1. Project layout

```
tests/e2e/web/
  playwright.config.ts
  fixtures/            # test.extend: auth per role, api client, factories, axe, page objects
  pages/               # page / component objects per screen (SCR-id in the file header)
  flows/               # reusable multi-step journeys (checkout, onboarding)
  specs/<feature>/     # *.spec.ts, one file per feature or journey
  data/                # factories (API-backed), synthetic generators
  .auth/               # storage states (git-ignored)
```

Config essentials:
- `projects`: a `setup` project that signs in each role once and saves storage state; browser projects depend on it. Chromium on PR; Chromium + WebKit + Firefox and the designer's breakpoints (mobile 390, tablet 768, desktop 1440) nightly.
- `use`: `baseURL` from env, `trace: 'on-first-retry'`, `screenshot: 'only-on-failure'`, `video: 'retain-on-failure'`, `testIdAttribute` agreed with the frontend.
- `retries`: 1 in CI (failure then pass = flaky, reported), 0 locally. `fullyParallel: true`; `workers` tuned to the runner.
- `reporter`: `list` locally; `junit`, `html` (+ `blob` for merged shards) in CI.
- Shard in CI with `--shard=i/n` and merge reports with `merge-reports`.
- `expect.timeout` short and global; no per-test magic numbers.

## 2. Writing tests

- **Locators**: `getByRole(name)`, `getByLabel`, `getByText` for content; `getByTestId` only when there is no accessible handle (then log a `TEST-REQ-xx` if it should have one). Never CSS chains or XPath tied to layout.
- **Waiting**: web-first assertions (`await expect(locator).toBeVisible()`); never `waitForTimeout`. Wait for the network response that matters with `page.waitForResponse` when a UI signal isn't enough.
- **Data**: create through API fixtures (`request` context or the project's API client) with unique ids per test; never through the UI unless the UI creation is what is under test.
- **Auth**: storage state per role from the setup project; tests that exercise login itself do it through the UI once.
- **Isolation**: each test can run alone and in any order; parallel-safe tenants / users.
- **Naming and tags**: `test('TS-04 BR-07 UX-03: overlapping booking shows the conflict and keeps the form', { tag: ['@critical', '@regression'] }, …)`. Use `test.step` for Given / When / Then readability in traces.
- **Assertions**: what the user sees and what the system stored (API check after a UI action for critical paths). ARIA snapshots (`toMatchAriaSnapshot`) for structure of complex widgets; avoid full-page DOM snapshots.
- **Error paths**: `page.route` to force 4xx / 5xx / timeouts / slow responses for states the real backend can't produce on demand; the forced bodies follow the contract's problem-details shape.
- **Accessibility**: `@axe-core/playwright` on every critical screen state, tags `wcag2a, wcag2aa, wcag21aa, wcag22aa`; known exceptions listed with an issue and expiry, never global disables.
- **Visual**: `toHaveScreenshot` only for stable, design-critical areas; mask dynamic content; baselines generated in CI's container (fonts and rendering differ across OSes), reviewed in PRs.

## 3. Mocked vs real backend

- **Mocked** (Lean rung): run the web app with MSW in the browser or `page.route` handlers generated from the OpenAPI; validate every mock response against the schema in a fixture. Fast and deterministic; proves the UI against the contract, not against the real backend.
- **Real** (Previews / Staging): same specs, real API; data via factories; third parties in sandbox mode or virtualized (Prism / WireMock) per environment.
- Keep one spec set and switch with a project or env flag where possible; tag tests that only make sense against the real stack `@real-backend`.

## 4. Playwright test agents

Playwright ships planner, generator and healer agents (initialised with `npx playwright init-agents`) that explore an app, write a Markdown test plan, generate specs and repair failing locators. Use them as accelerators:
- Planner output feeds your test plan, but conditions and priorities come from requirements and risk, not from what the agent happened to explore.
- Generated specs are reviewed like any PR against the Review mode checklist (locators, data, assertions, ids).
- Healer fixes are accepted only when the change is a test bug (renamed label, moved element); if the app behaviour changed, it's a product question or a bug — never let a healer "fix" a real regression away.
- The Playwright MCP server / browser tools are good for exploratory sessions and reproducing bugs; committed tests stay in Playwright Test.

## 5. Performance of the suite

- Reuse auth state, seed through APIs, block third-party analytics and fonts not under test (`page.route`), run only `@smoke` and impacted specs on PR.
- Watch the slowest 10 tests per run; anything over ~30 s is split or moved down a level.
