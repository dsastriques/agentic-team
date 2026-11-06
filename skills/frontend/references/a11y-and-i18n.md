# Accessibility and localization in code

The designer defines the accessible behaviour (annotations, `a11y` block, component specs); this reference is how it's built. The target is WCAG 2.2 AA at least (`a11y_target`).

## 1. Web semantics first

- Native elements before ARIA: `<button>`, `<a href>`, `<input>`, `<select>`, `<dialog>`, `<details>`, `<fieldset>` / `<legend>`, `<table>`. The first rule of ARIA: don't use it when HTML does the job.
- Landmarks per the screen's `a11y.landmarks` (`header`, `nav`, `main`, `aside`, `footer`); one `main`; a skip link to it.
- One `h1` per page (the screen title), headings in order; `document.title` = `<screen title> – <app>`; `<html lang>` follows the active locale, `lang` on inline text in another language.
- Links navigate, buttons act. No `div` with `onClick`.
- Every control has an accessible name (visible label, `aria-label` only when there's no visible text, and the visible text is contained in the name — WCAG 2.5.3).
- Icons: decorative → `aria-hidden="true"`; meaningful icon-only buttons → name from a message key.
- Images: `alt` from a message key when meaningful, `alt=""` when decorative.

## 2. Widgets (APG)

Use a headless, tested implementation (React Aria, Radix, Ariakit) for: dialog / modal, menu and menu button, combobox / autocomplete, listbox, tabs, tooltip, disclosure / accordion, date picker, slider, grid. Hand-rolled widgets must follow the WAI-ARIA APG pattern exactly (roles, states, keyboard map, focus handling) and carry tests for each keyboard interaction.

Key behaviours:
- **Dialog**: focus moves into the dialog (first field or the title per the spec), focus is trapped, `Escape` closes, focus returns to the trigger, background is inert (`inert` attribute or the native `<dialog>` modal).
- **Menus and listboxes**: arrow keys move, `Home` / `End`, type-ahead, roving `tabindex` or `aria-activedescendant`.
- **Tabs**: arrows switch tabs, `Tab` goes into the panel.
- **Tooltips** on hover and focus, dismissible with `Escape`, never the only place for essential information (1.4.13).

## 3. Focus management in an SPA

- On route change: move focus per the screen's `focus_on_enter` (usually the `h1` with `tabIndex={-1}`), update the title, and announce the page via a polite live region if focus doesn't land on the title.
- After actions: focus the result (new item, error summary, confirmation) per the design; after delete, focus the next logical item.
- Form submit with errors: focus the error summary (with links to fields) or the first invalid field, per the spec; `aria-invalid` and `aria-describedby` pointing to the error text.
- Focus is always visible (`:focus-visible` style from tokens, ≥ 3:1 contrast, not obscured by sticky headers — 2.4.11; use `scroll-margin` / `scroll-padding`).
- No positive `tabindex`; DOM order = visual order = reading order.

## 4. Announcements

- One app-level polite live region and one assertive region, mounted at start (live regions added at the moment of the change are often not read); a small `announce(messageKey, { politeness })` helper.
- Polite: results updated, item added, saved. Assertive: only errors that block the task.
- Status messages per 4.1.3: loading complete, "3 results", form errors summary.
- Announce at the start of the change, not after the motion ends.

## 5. Visual and input

- Colours only from tokens (the designer verified contrast); never convey state with colour alone.
- Support 200% zoom and reflow at 320 CSS px (no horizontal scroll except data tables, maps); text spacing overrides (1.4.12) don't break layout: avoid fixed heights on text containers.
- Target size ≥ 24×24 CSS px (2.5.8), the design system's sizes are usually larger; spacing for small inline targets.
- Pointer gestures (drag, swipe, pinch) always have a single-pointer alternative (2.5.1, 2.5.7).
- Respect `prefers-reduced-motion`, `prefers-color-scheme`, `forced-colors` (Windows high contrast: use system colours, keep borders visible), `prefers-contrast`.
- Never `maximum-scale=1` or `user-scalable=no`.
- Accessible authentication (3.3.8): allow paste and password managers, `autocomplete="current-password"` / `one-time-code`.
- Redundant entry (3.3.7): prefill values the user already gave in the flow.

## 6. React Native accessibility

- `accessible`, `accessibilityRole` / `role`, `accessibilityLabel`, `accessibilityHint` (only when the label isn't enough), `accessibilityState` (`disabled`, `selected`, `checked`, `expanded`, `busy`), `accessibilityValue` for sliders and progress.
- Group related elements (`accessible` on the container) so a card reads as one item; hide decorative views (`importantForAccessibility="no-hide-descendants"` on Android, `accessibilityElementsHidden` on iOS).
- Custom actions (`accessibilityActions` + `onAccessibilityAction`) for swipe-only gestures.
- Announcements: `AccessibilityInfo.announceForAccessibility`; move focus with `AccessibilityInfo.setAccessibilityFocus` (via `findNodeHandle`) after navigation or errors when the spec says so; headings with `accessibilityRole="header"`.
- Dynamic type: no fixed heights on text, `allowFontScaling` stays on, cap with `maxFontSizeMultiplier` only for chrome where the design says so; test at the largest size.
- Minimum touch targets 44×44 pt (iOS) / 48×48 dp (Android) using `hitSlop` when the visual is smaller.
- Reduced motion: `AccessibilityInfo.isReduceMotionEnabled` or Reanimated's `useReducedMotion()`.
- Test with VoiceOver and TalkBack on real devices for new patterns.

## 7. Accessibility testing

- Lint: `eslint-plugin-jsx-a11y` (web).
- Unit: Testing Library queries by role and name (a failing `getByRole` is an a11y bug); `jest-axe` / `vitest-axe` on rendered states.
- Storybook a11y addon (axe) on every story; fail CI on violations.
- E2E: `@axe-core/playwright` per screen state; keyboard-only flow tests; reduced-motion emulation.
- Manual for new patterns: screen reader (NVDA + Firefox or Chrome, VoiceOver + Safari, VoiceOver iOS, TalkBack), keyboard-only, 200% / 400% zoom, forced colours.
- Automated tools catch only part of the issues: record manual checks in the PR.

## 8. Localization

### Setup
- Library per config: i18next + react-i18next (namespaces, lazy backends, wide ecosystem), FormatJS / react-intl (ICU-native), or Lingui (ICU, compile-time extraction, small runtime). All use ICU MessageFormat or support it; use ICU syntax.
- Message files per locale and namespace (`locales/<locale>/<feature>.json`), keys exactly the designer's copy keys (`checkout.payment.title`); the default-locale file is the source; types generated from it so a missing or misspelled key fails typecheck.
- Lazy-load locale bundles per namespace with the route; preload the user's locale for the shell.
- Locale negotiation: user setting → stored preference → `Accept-Language` / `navigator.languages` / `expo-localization` → default; persist the choice; set `<html lang dir>`.

### Messages
- Full sentences with placeholders; never concatenate fragments or build sentences in code (word order differs per language).
- Plurals and selects in ICU: `{count, plural, one {# night} other {# nights}}`; gender or variant with `select`.
- Rich text via the library's component interpolation (`<Trans>` / `FormattedMessage` with tags), not HTML strings.
- Context for translators in descriptions; no text inside images.
- Errors from the API: map problem-detail `type` / codes to message keys; never show raw server messages unless the contract says they're localized.

### Formatting
- `Intl.DateTimeFormat`, `Intl.NumberFormat` (currency, percent, units, compact), `Intl.RelativeTimeFormat`, `Intl.ListFormat`, `Intl.PluralRules`, `Intl.Collator` for sorting, through shared helpers in `packages/i18n` that take the active locale.
- Dates: store and send ISO 8601 with offset or UTC; format in the user's time zone (or the venue's, when the requirement says so, e.g. a booking's local time); `Temporal` once supported by the targets, or a small date library for arithmetic.
- Money: amounts as integer minor units or decimal strings from the API plus currency code; never floats for arithmetic; format with `Intl.NumberFormat(locale, { style: 'currency', currency })`.

### Layout
- Logical CSS properties (`margin-inline-start`, `padding-block`, `inset-inline-end`, `text-align: start`) so RTL works; mirror directional icons; `dir="rtl"` on the root when the locale needs it. On native: `I18nManager` and `start` / `end` style props.
- Room for text expansion (German, Dutch ~30% longer): no fixed widths on text, test with the longest translation the designer used and a pseudo-locale (accented, +40% length, brackets) that makes truncation and hard-coded strings visible.

### Process
- Extraction and missing-key checks in CI; no hard-coded user-facing strings (lint rule such as `i18next/no-literal-string` or Lingui's extraction report).
- New keys added to the default locale with the designer's copy; translations flow through the configured translation process (TMS or files); untranslated keys fall back to the default locale and are reported.
