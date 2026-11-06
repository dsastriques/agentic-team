# Bundling, performance and micro frontends

## 1. Budgets and metrics

Set in the config from the NFRs; defaults when the NFRs are silent (mark them `ASSUMPTION`):

| Metric | Target (p75, mobile) |
|---|---|
| LCP | ≤ 2.5 s |
| INP | ≤ 200 ms |
| CLS | ≤ 0.1 |
| Initial JS per route | ≤ 170 kB gzip (app shell + route chunk) |
| Route chunk | ≤ 50 kB gzip |
| Native cold start (mid-range Android) | ≤ 2 s to interactive |
| Native frame rate | 60 fps on scrolling and transitions, no JS-thread stalls > 100 ms |

Measure: Lighthouse CI on key routes (lab), the `web-vitals` library reported to analytics (field), `size-limit` or bundle-analyzer in CI, React Profiler / React DevTools, Chrome Performance panel; on native, the React Native DevTools profiler, Android Studio profiler / Xcode Instruments, and Reassure for render-performance regression tests.

## 2. Bundlers

| Tool | Use |
|---|---|
| Vite | default for SPAs and libraries (dev on native ESM, production build with Rollup / Rolldown) |
| Rspack / Rsbuild | webpack-compatible and fast; large apps, webpack migrations, Module Federation |
| webpack | existing setups; mature Module Federation |
| Next.js bundler | whatever the framework uses (Turbopack); don't fight it |
| Metro | React Native; Re.Pack (Rspack) when native micro frontends or advanced splitting are needed |
| tsup / Vite library mode | internal packages (`packages/ui`), ESM output, `sideEffects` declared |

Practices:
- ESM everywhere, `"sideEffects": false` (or a precise list) in packages so tree shaking works; import per module, not barrel files that pull everything (or make barrels side-effect free).
- Modern browser targets from the support matrix; no blanket polyfills (only what the targets need).
- Content-hashed file names, long-term `Cache-Control: immutable` for hashed assets, short cache for HTML; a vendor split only when it improves cache hits (let the bundler's chunking work first).
- Source maps uploaded to error reporting, not served publicly unless the project decides so.
- Environment config via the bundler's env mechanism; only public values reach the client (`VITE_`, `NEXT_PUBLIC_`, `EXPO_PUBLIC_` prefixes); never secrets.
- Analyze every significant dependency addition (bundle analyzer, `bundlephobia`-style size check); prefer smaller alternatives and native platform APIs (`Intl` over date libraries for formatting, `fetch` over heavy clients).

## 3. Lazy loading and loading strategy

- **Route-level splitting** always: `lazy(() => import('./routes/payment'))` or the router's lazy routes; group tiny routes of one flow if they're always used together.
- **Component-level splitting** for heavy, below-the-fold or conditional UI: charts, editors, maps, dialogs opened rarely, admin-only views.
- **Prefetch with intent**: on link hover / focus / viewport (router prefetch), the next step of a flow after the current one is idle, data with `queryClient.prefetchQuery` together with the chunk. Never prefetch everything.
- **Suspense boundaries** placed where a skeleton makes sense (route, independent widgets); skeletons sized like the content to avoid CLS.
- **Images**: responsive `srcset` / `sizes`, modern formats (AVIF / WebP), explicit width and height or `aspect-ratio`, `loading="lazy"` below the fold, `fetchpriority="high"` for the LCP image, never lazy-load the LCP image; on native `expo-image` with caching and blurhash / thumbhash placeholders.
- **Fonts**: self-host, subset, `woff2`, `font-display: swap` (or `optional`), preload only the critical face, metric-compatible fallbacks to avoid layout shift.
- **Third-party scripts**: deferred or loaded after interaction; consent-gated per GDPR requirements.
- **Rendering**: SSR / SSG / streaming only where SEO or first-load needs justify it (an ADR); hydrate less (server components, islands) when the framework supports it.
- **Runtime**: virtualize long lists (TanStack Virtual / FlashList), debounce search input, move heavy computation to Web Workers, avoid layout thrash, keep long tasks < 50 ms (yield with `scheduler.yield()` or `setTimeout`), use `startTransition` for non-urgent updates to protect INP.
- **Native**: Hermes bytecode; inline requires / lazy imports for rarely used screens; defer non-critical init after first render; keep the initial navigation stack shallow.

## 4. Micro frontends

### Decide first (ADR with the architect)
Adopt only when most of these hold:
- several autonomous teams own separate product areas and need independent release cadences;
- the areas align with bounded contexts from the domain model, with little shared UI state between them;
- a monolithic frontend's build or release coordination is a measured bottleneck;
- there is capacity to run the platform (shell, shared design system, contracts, observability).

Otherwise prefer a **modular monolith frontend**: one app, features as packages with enforced boundaries, one deploy. It gives most of the isolation at a fraction of the cost. Record the decision either way.

### Composition options

| Option | How | Fits |
|---|---|---|
| Build-time packages | remotes published as versioned packages, composed at build | shared release cadence acceptable; strongest type safety |
| Runtime Module Federation | host loads remote entry points at runtime (Module Federation 2.0 on Rspack / webpack / Vite plugin, manifest-based) | independent deploys, same framework |
| Route-based split (multi-SPA) | separate apps per path behind one domain / edge router | areas with little shared chrome; simplest independence |
| Web Components | remotes expose custom elements | mixed frameworks; isolation via shadow DOM |
| Server / edge composition | fragments assembled on the server or CDN | content sites, SEO-critical |
| iframes | full isolation | third-party or legacy embeds only |
| Native (React Native) | Re.Pack with Module Federation, or separate native modules | large super-apps; usually not worth it |

### Rules when adopted
- **Shell** owns routing at the top level, auth session, app chrome (navigation, layout), error boundaries per remote, telemetry and feature flags.
- **Shared singletons**: React, React DOM, the router, TanStack Query (one `QueryClient` from the shell or one per remote with no cross-remote cache sharing — decide in the ADR), the design system and i18n runtime, with compatible version ranges; everything else bundled per remote.
- **Contracts**: each remote exposes a small typed API (mount function or component props, routes it owns, events it emits); communicate via props, the URL and typed custom events or a tiny event bus — never via each other's stores or globals.
- **Design system** shared as one versioned package (and tokens as CSS variables from the shell) so UI stays consistent; no remote ships its own copy of base components.
- **Resilience**: a remote failing to load shows a fallback and is reported; the shell keeps working; version skew handled with manifests and compatible ranges.
- **Performance**: watch duplicated dependencies and waterfalls of remote entries; preload remotes for likely routes; budgets per remote.
- **Testing**: contract tests on each remote's exposed API, e2e on the composed app in staging.
- **Ownership**: one remote per team and bounded context; cross-cutting changes go through the shell owners.

## 5. Caching and delivery

- CDN for static assets, HTTP/2 or HTTP/3, Brotli compression.
- Service worker only when offline or install is required (Workbox or the framework's PWA plugin), with a clear update strategy (prompt on new version).
- API caching with TanStack Query plus HTTP caching (`ETag` / `If-None-Match`) where the backend supports it.
- Release safety: feature flags, canary through the devops pipeline, error-rate monitoring after deploys, quick rollback; for native, staged rollouts and OTA updates with rollback.
