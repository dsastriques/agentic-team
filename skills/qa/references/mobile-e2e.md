# Mobile e2e

## 1. Choosing the tool

| Tool | Choose when | Notes |
|---|---|---|
| **Maestro** (default for Expo / React Native) | Black-box flows across iOS and Android, fast to write, runs on emulators, simulators, EAS Workflows and Maestro Cloud | YAML flows, built-in waiting and retries, `testID` / accessibility-label selectors, JS scripting for data, Maestro Studio for authoring |
| **Detox** | Gray-box RN testing where synchronization with the JS thread and animations matters, tight Jest integration | Needs native build configuration; strong determinism once set up; check current Expo compatibility before choosing |
| **Appium 2** | Native (Swift / Kotlin) or hybrid apps, or a device cloud that requires it | Drivers per platform (XCUITest, UiAutomator2), heavier and slower; WebdriverIO as client |

Record the choice in `qa.config.md`; a change of tool is an ADR draft for the architect.

## 2. Maestro conventions

```
tests/e2e/mobile/
  config.yaml                # flow inclusion, tags, platform excludes
  flows/<feature>/*.yaml     # one journey per file, header comment with TS / BR / UX ids
  subflows/                  # login-as-<role>.yaml, seed-<thing>.yaml, reset.yaml
  scripts/                   # JS helpers: API seeding with unique ids, data generators
```
- Selectors: `id:` = React Native `testID`, or the accessibility label (prefer what screen-reader users get). Agree the `testID` scheme with the frontend (`<SCR-id>.<element>`), request missing ones as `TEST-REQ-xx`.
- Start state via `launchApp: { clearState: true }`, deep links (`openLink`) into the screen under test, and API seeding through `runScript`; avoid tapping through setup screens.
- Waiting: `extendedWaitUntil` with a bound for slow operations; never fixed sleeps.
- Tags (`critical`, `smoke`, `regression`) in the flow header; run subsets with `--include-tags`.
- Permissions (`permissions:` on launch), system dialogs, backgrounding / foregrounding, rotation and slow network are explicit cases in the test plan for mobile journeys.
- Record a video / screenshots on failure; keep the logs as CI artifacts.

## 3. Builds for testing (with the frontend and devops)

- A dedicated EAS build profile (e.g. `e2e`) producing a simulator build (iOS) and an APK (Android), pointing at the test environment's API, with test hooks enabled only in that profile.
- Release-like builds for release verification (same as store builds, test API).
- OTA updates: verify the update channel the release uses, and that an update applies on a build of the previous runtime version when the release relies on it.
- Build and run on EAS Workflows (Maestro job type), GitHub Actions with emulators (Android on Linux runners with KVM; iOS needs macOS runners), or a device cloud — per the approach and budget. Requests go to devops.

## 4. Device matrix

Pick from the product's analytics or market, not habit:
- **Per PR**: one Android emulator + one iOS simulator (current OS), smoke tags.
- **Nightly**: the matrix in `qa.config.md` — at least: low-end Android (oldest supported OS, small screen), mid-range Android (common OEM skin), current Pixel, smallest supported iPhone, current iPhone, a tablet if the design supports one.
- **Release**: full matrix on real devices when the approach includes a device cloud; otherwise emulators plus a documented manual pass on owned devices.
- Include largest text size, dark mode, reduced motion and a screen-reader smoke (TalkBack / VoiceOver) for critical screens; the designer audits the rest.

## 5. API in mobile tests

- **Mocked**: MSW for React Native (`msw/native`) in the `e2e` profile, or a Prism / WireMock server the app points to; responses validated against the OpenAPI as on web.
- **Real**: the preview or staging API with seeded data; push notifications and deep-link entry tested against the provider's sandbox where feasible.
