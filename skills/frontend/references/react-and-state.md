# React, React Native and state

How the frontend skill structures apps, components and state. Check current stable versions of each library before scaffolding; follow the project's ADRs where they differ from these defaults.

## 1. Default stacks (proposals to the architect)

| Need | Default | Alternatives and when |
|---|---|---|
| Web app behind login (SPA) | Vite + React + TypeScript, React Router or TanStack Router | Rsbuild / Rspack for large apps or Module Federation |
| Public, SEO-relevant or content pages | Next.js (App Router, server components) | Astro for mostly static content; React Router framework mode |
| Mobile | Expo (managed, prebuild + config plugins), Expo Router, EAS Build / Update | bare React Native only for native needs Expo can't cover |
| Web + mobile from one codebase | monorepo with shared `packages/` (tokens, api-client, i18n, domain logic, hooks); separate UI per platform | React Native Web / Expo web when the UI is genuinely the same |
| Server state | TanStack Query | RTK Query only if Redux is already there |
| Forms | React Hook Form + Zod (`@hookform/resolvers`) | TanStack Form |
| Global client state | none; then context; then Zustand | Redux Toolkit for large, event-heavy client state |
| Styling (web) | CSS Modules or Tailwind on CSS-variable tokens; headless primitives (React Aria, Radix) for complex widgets | vanilla-extract for typed themes |
| Styling (native) | `StyleSheet` + typed theme from tokens | Unistyles / Tamagui when themes and breakpoints get heavy |
| Motion | Motion (motion.dev) on web, Reanimated on native | CSS transitions for simple feedback |
| Lists (native) | FlashList | FlatList for short lists |
| Testing | Vitest (web) / Jest (native), Testing Library, MSW, Playwright, Maestro or Detox, Storybook | |

## 2. Project structure

Feature-based, with a shared layer and enforced import boundaries (ESLint `import/no-restricted-paths` or `eslint-plugin-boundaries`):

```
apps/web/src/
  app/            # providers, router, error boundaries, app shell
  features/
    checkout/
      api/        # queryOptions, mutations, mappers (DTO → view model)
      components/ # feature UI, presentational
      routes/     # route components (lazy), one per SCR-id
      hooks/      # feature logic (headless)
      model/      # types, zod schemas, pure functions
      index.ts    # public API of the feature; other features import only this
  shared/         # cross-feature hooks, utils, lib adapters
packages/
  ui/             # design-system components (props = Figma properties)
  tokens/         # generated from tokens_path
  api-client/     # generated OpenAPI types + fetch wrapper + query key factory
  i18n/           # setup, formatters, message loading
  motion/         # catalog presets keyed by motion id
```

Rules: features never import another feature's internals; `ui` never imports features or the API; domain logic in `model/` is pure and framework-free (hexagonal: API and platform services behind small adapter interfaces so they can be mocked).

Naming: route component named after the screen (`SCR-checkout-payment` → `CheckoutPaymentScreen`, file `routes/payment.tsx`), layer names from Figma become component and test ids (`payButton` → `PayButton` / `testID="payButton"`).

## 3. Components

- Function components with typed props; props mirror the design-system spec (`variant`, `size`, `tone`, `disabled`, `loading`, `selected`, `value`).
- Composition over configuration: `children`, slots, compound components (`<Tabs><Tabs.List/>…`) instead of growing booleans.
- Headless hooks for behaviour (`useCombobox`), presentational components for markup; prefer proven headless libraries (React Aria) for complex APG widgets rather than hand-rolling keyboard logic.
- Controlled and uncontrolled support for inputs (`value` / `defaultValue` / `onChange`); forward refs (React 19: `ref` as a prop).
- No outer margins in components; layout belongs to the parent.
- Effects only to sync with external systems. Derive state during render; don't mirror props into state; data fetching never in `useEffect` (use TanStack Query or the framework loader).
- Memoization: with the React Compiler enabled, don't add `useMemo` / `useCallback` / `memo` by hand except where profiling shows a need; without it, memoize only measured hot paths and stable callbacks passed to memoized children or lists.
- Error boundaries at route level and around independent widgets, with a retry that resets the related queries (`QueryErrorResetBoundary`).
- Suspense for code and data where the router or `useSuspenseQuery` supports it; `startTransition` / `useDeferredValue` for non-urgent updates (filters, search).

## 4. State model

| Kind | Lives in | Examples |
|---|---|---|
| Server state | TanStack Query cache | orders, user profile, availability |
| URL state | router (path, search params) | selected tab, filters, page, open item id |
| Form state | React Hook Form | checkout inputs, validation |
| Local UI state | `useState` / `useReducer` | open / closed, hover, step within a component |
| Global client state | context or Zustand (small) | theme, auth session flags, feature flags |
| Persistent client state | storage adapter (MMKV / AsyncStorage on native, IndexedDB / localStorage for non-sensitive prefs on web) | onboarding seen, last locale |

Never copy server data into a client store. Reducers or state machines (XState) for flows with many states — the designer's flow state machine is the model.

## 5. TanStack Query patterns

- **Query key factory** per resource, keys hierarchical and serializable:
  ```ts
  export const orderKeys = {
    all: ['orders'] as const,
    lists: () => [...orderKeys.all, 'list'] as const,
    list: (f: OrderFilters) => [...orderKeys.lists(), f] as const,
    detail: (id: string) => [...orderKeys.all, 'detail', id] as const,
  };
  export const orderQuery = (id: string) =>
    queryOptions({ queryKey: orderKeys.detail(id), queryFn: ({ signal }) => api.getOrder(id, { signal }), staleTime: 30_000 });
  ```
  Use `queryOptions` / `infiniteQueryOptions` everywhere so `useQuery`, `prefetchQuery`, `ensureQueryData` and loaders share one definition.
- **QueryClient defaults**: sensible `staleTime` per resource (not 0 for stable data), `gcTime` default, `retry` only for network errors and 5xx (never 4xx), `refetchOnWindowFocus` on for data that changes, off for static data. On native wire `focusManager` to `AppState` and `onlineManager` to NetInfo.
- **Loading UX**: `isPending` for first load (skeleton), `isFetching` for background refresh (subtle indicator), `placeholderData: keepPreviousData` for pagination and filters, no spinners that flash (< 300 ms).
- **Mutations**: `useMutation` with `onSuccess` invalidation of the narrowest keys; optimistic updates (`onMutate` snapshot → `setQueryData`, `onError` rollback, `onSettled` invalidate) only where the UX-xx asks for instant feedback and rollback is clear; `mutationKey` + `useMutationState` for pending indicators elsewhere; idempotency key generated once per user intent.
- **Prefetching**: on route loaders, hover / focus intent for links, and the next step of a flow.
- **Infinite lists**: `useInfiniteQuery` with cursor pagination; virtualize long lists.
- **Dependent and parallel queries**: `enabled` for dependencies, `useQueries` / `useSuspenseQueries` for parallel.
- **Offline (native)**: persist the cache with the persister plugin for read data the requirements say must work offline; queue mutations with `networkMode` and resume paused mutations on reconnect.
- **Selectors**: `select` to derive view data and limit re-renders; map DTOs to view models in the feature's `api/` layer, not in components.
- **Devtools** in development only.

## 6. Forms

- Zod schema per form in `model/`, from the JSON `inputs[].validation`; the same schema validates and types.
- Validation timing per the designer: on blur, then on change after the first error; on submit show an error summary and move focus to it or to the first invalid field.
- Server errors (problem details with field errors) mapped back with `setError` to the field and its `error_keys`.
- Inputs keep their values after errors; correct `type`, `inputMode`, `autoComplete` (web) / `autoComplete` + `textContentType` (native); submit disabled only while submitting, not as validation.

## 7. Routing and navigation

- One route per `route` in the screen JSON; states of the same route are UI state or search params, not routes.
- Transition grammar → code: `push` = navigate, `replace` = navigate with replace, `modal` = modal route or dialog (web: dialog route or overlay; Expo Router: `presentation: 'modal'`), `dismiss` = close overlay, `back` = history back with a fallback route, `state` = same route new state.
- Lazy-load every route component; prefetch the next route of a flow.
- Typed routes and search params (TanStack Router or Expo Router typed routes); validate search params with Zod.
- Auth guards in loaders / layouts, not in each component; deep links and universal links on native map to the same routes.
- On web route change: update `document.title` from the screen's `a11y.title`, move focus per `focus_on_enter`, announce the new page (see `a11y-and-i18n.md`); restore scroll on back.

## 8. React Native specifics

- New Architecture on; Hermes; Expo SDK on the current stable line; config plugins over manual native edits; EAS for builds, EAS Update for OTA of JS-only changes (respect store rules).
- Safe areas (`react-native-safe-area-context`), keyboard handling (`KeyboardAvoidingView` or a keyboard controller library), platform files (`.ios.tsx` / `.android.tsx`) only for real differences.
- Animations on the UI thread with Reanimated; gestures with Gesture Handler; respect `useReducedMotion()`.
- Lists with FlashList (stable keys, `estimatedItemSize` where needed, memoized row components), images with `expo-image` (caching, placeholders).
- Avoid re-renders across the bridge-heavy paths: stable callbacks for list rows, no inline objects in hot lists, `InteractionManager` / deferred work after navigation.
- Secure storage for tokens (`expo-secure-store`), MMKV for fast non-sensitive storage.
- Test on a low-end Android device profile; measure startup and frame drops.

## 9. TypeScript and quality

- `strict: true`, `noUncheckedIndexedAccess`, no `any` (use `unknown` and narrow); discriminated unions for states (`{ status: 'error'; error: Problem }`).
- Runtime validation (Zod) for data crossing trust boundaries that aren't covered by generated types (storage, deep links, postMessage, untrusted API responses when the architect requires it).
- ESLint: `react-hooks`, `jsx-a11y` (web), import boundaries, `@tanstack/eslint-plugin-query`; Prettier; no lint disables without a reason comment.

## 10. Testing

- Testing Library: query by role and accessible name (`getByRole('button', { name: /pay/i })`), `user-event` for interactions; test behaviour and UX-xx, not implementation.
- MSW for API in unit, Storybook and e2e-against-mocks; one handler per state (ok, empty, 4xx problem, 5xx, slow, offline).
- Storybook: one story per component state and per screen state; interaction tests and a11y addon in CI.
- Playwright: flows per UX-xx, `@axe-core/playwright` on each state, viewport projects per breakpoint, `emulateMedia({ reducedMotion: 'reduce' })` checks.
- Native: React Native Testing Library; Maestro (simple YAML flows) or Detox (gray-box) for e2e.
- Snapshot tests only as a complement, never as the only test.
