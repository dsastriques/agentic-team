# From the designer's handoff contract to code

The designer's contract (`docs/design/fe-handoff-contract.md`, `contract_version: 2`) puts every screen in three layers: structure in Figma, machine-readable metadata (`docs/design/screens/<SCR-id>.json`, also shared plugin data `agentic_team` / `screen` on the frame) and Dev Mode annotations. The JSON is the build input; Figma gives exact layout and visuals.

## 1. Reading order

1. The screen JSON for the default state at mobile, then the other states and breakpoints (`SCR-<area>-<name>:<state>`).
2. The feature doc: UX-xx criteria, copy deck, analytics, assumptions.
3. Component specs (`docs/design/components/<name>.md`) for each component used: props, states, a11y, motion.
4. `docs/design/motion.md` for every motion id the screen lists.
5. Figma, when needed: `get_design_context` (after loading `figma-design-to-code`) for structure, `get_variable_defs` for token names, `get_screenshot` for the visual target, a read-only `use_figma` (with `figma-use`) to read shared plugin data or annotations if the JSON is missing.

## 2. Field → code mapping

| Contract field | Code |
|---|---|
| `id` | route component name and file (`SCR-checkout-payment` → `CheckoutPaymentScreen`), test file name, story title |
| `state` | a branch of the screen's view-state union (`'default' \| 'loading' \| 'error-declined' \| 'offline'`), a story and a test each |
| `breakpoint` + `responsive` | layout rules: media / container queries on web, `useWindowDimensions` breakpoints on native; values from the `Layout` tokens |
| `status` | build only `ready`; `changed` → update existing code |
| `route` | router entry; params typed; search params for filters and tabs |
| `flow`, `entry` | navigation sources; prefetch targets; back behaviour |
| `requirements`, `ux_ac` | test names and the PR's traceability table |
| `data[]` | `queryOptions` per `source` endpoint; view-model fields; `required` → render guard and empty state |
| `inputs[]` | Zod schema (`validation`), field component and props (`type`, `autocomplete`), error messages from `error_keys` |
| `actions[]` | event handlers following the transition grammar `trigger [condition] / effect → how` |
| `states[]` | every listed state reachable and covered by a story and an MSW scenario |
| `a11y` | `document.title` / header title, landmarks, focus target on enter, live-region messages |
| `motion[]` | preset by id on the element, with trigger and order (see §4) |
| `analytics[]` | tracking calls through the analytics adapter, event names exact |
| `copy_keys` | message keys in the feature's namespace |
| `open_questions` | carried into the report; build behind a stub if they block |

Transition grammar → code:

| `how` | Web | Native (Expo Router) |
|---|---|---|
| `push` | `navigate(to)` | `router.push(to)` |
| `replace` | `navigate(to, { replace: true })` | `router.replace(to)` |
| `modal` | dialog route or overlay component | route with `presentation: 'modal'` (or a sheet) |
| `dismiss` | close the overlay; return focus to its trigger | `router.dismiss()` |
| `back` | `navigate(-1)` with a fallback route when there's no history | `router.back()` with `canGoBack()` check |
| `state` | set view state / search param on the same route | same |

`[condition]` becomes a guard (form valid, feature flag, permission); `effect` becomes the mutation or side effect; navigation happens on success unless the contract says otherwise.

## 3. Structure and names

- Figma auto layout ≈ flexbox: direction, gap, padding, alignment and hug / fill / fixed sizing translate directly; grid layout → CSS grid.
- Instances → design-system components with the same property names and values; never re-create a component from its visual.
- Layer names → component names, `data-testid` / `testID` (only where role and name aren't enough for tests) and analytics element ids.
- Variables → tokens by code syntax (`var(--color-text-primary)` / `theme.color.text.primary`); never hard-coded values. A value with no token is a `DESIGN-GAP`.
- Text layers named by i18n key → `t('checkout.payment.title')`.
- Images: assets from the designer's exports or `download_assets`; named placeholders become `DESIGN-GAP`s.

## 4. Motion

Build the catalog **once**, in `packages/motion` (or `src/ui/motion`), keyed by the catalog id without the `motion/` prefix:

```ts
// web: Motion (motion.dev) variants + CSS variables from the motion tokens
export const motionCatalog = {
  'enter/fade-up': {
    initial: { opacity: 0, y: 12 },
    animate: { opacity: 1, y: 0, transition: { duration: tokens.motion.duration.base, ease: tokens.motion.easing.enter } },
    exit: { opacity: 0, y: 8, transition: { duration: tokens.motion.duration.fast, ease: tokens.motion.easing.exit } },
    reduced: { initial: { opacity: 0 }, animate: { opacity: 1, transition: { duration: tokens.motion.duration.fast } } },
  },
  // ...one entry per id in docs/design/motion.md
} as const satisfies Record<MotionId, MotionPreset>;
```

- `MotionId` is a union generated from `docs/design/motion.md` (or a typed list kept in sync) so an unknown id fails typecheck.
- A `useMotionPreset(id)` hook (or `<Motion id="enter/fade-up">` wrapper) picks the reduced variant when `useReducedMotion()` is true (Motion's hook on web, Reanimated's on native) and wires the trigger: mount, in-view (once, 20%), press, hover (pointer devices only), state change, route enter.
- Stagger: parent variants with `staggerChildren = motion/stagger`, capped so the total stays ≤ 0.3 s; the JSON's `order` sets which group goes first.
- Native: Reanimated layout animations (`entering` / `exiting` built from the same tokens, e.g. a configured `FadeInUp`), shared values for feedback, `withSpring` for spring tokens; screen transitions through the navigator's animation options mapped to `nav/*` ids.
- CSS-only motions (hover, press) can use CSS transitions with the token variables (`--motion-duration-fast`, `--motion-easing-enter`) under `@media (prefers-reduced-motion: no-preference)`.
- Animate `transform` and `opacity` only (or layout animations from the library); motions are interruptible; never block input; never delay focus moves or announcements until the end of a motion.
- `get_motion_context` snippets: take exact keyframe values and curves from them, but trigger, duration, delay, repeat and reduced motion from the catalog and the JSON — the snippet's timing is Figma's looping demo timeline.
- Per-screen overrides only when the JSON lists one and the feature doc justifies it.

## 5. Verifying against the design

- Screenshot each state at each breakpoint (Playwright viewports or the browser tools; simulator on native) and compare with `get_screenshot` of the matching frame: spacing, alignment, typography, colours (tokens), states.
- Optional visual regression (Storybook + Chromatic / Playwright screenshots) for design-system components.
- Differences you can't resolve become `DESIGN-GAP`s with both screenshots; the designer decides.
- After merging, tell the designer which screens are built so they can link Code Connect and run `designer review`.

## 6. Report gaps in a fixed shape

```markdown
| id | type | screen / component | issue | blocking? | owner |
|---|---|---|---|---|---|
| DG-01 | DESIGN-GAP | SCR-checkout-payment:error-declined | no copy key for "card expired" (code card_expired) | no — generic error shown | designer |
| TB-01 | TBD-BE | SCR-search-results | GET /items not deployed; built against MSW | yes for e2e | backend |
| TA-01 | TBD-ARCH | SCR-checkout-payment | idempotency key not in the payments contract | no | architect |
```
