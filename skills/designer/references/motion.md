# Micro-interactions and motion

Motion makes an interface feel responsive, polished and modern, but only when every animation has a job. This reference covers how the designer chooses, names, specifies, builds (in Figma when available) and hands off micro-animations.

## 1. Principles

1. **Purpose first.** Every motion serves one of:
   - **Feedback**: confirms an action (press, toggle, submit, copy, add to cart).
   - **State change**: shows something changed (loading → success, error, selected, expanded).
   - **Orientation and continuity**: shows where something came from or went (navigation, sheet from the bottom, shared element, list item added or removed).
   - **Attention**: points to what needs action now (a validation error, a new badge). Use sparingly.
   - **Delight**: brand character at key moments (first success, completion). At most one per flow, never in a repeated task.

   If a motion has none of these, drop it.
2. **Fast and calm.** Micro-interactions 100–200 ms, component transitions 200–300 ms, screen and large-surface transitions 300–500 ms. Nothing in a frequent task above 500 ms. Exits are about 20–30% shorter than entrances.
3. **Natural easing.** Ease-out (decelerate) for things entering or responding, ease-in (accelerate) for things leaving, ease-in-out for things moving on screen, gentle springs for direct manipulation and playful feedback. Never linear for movement (linear only for opacity loops, progress and colour).
4. **Small distances.** Entrances move 8–24 px, scale from 0.95–0.98 (never from 0), press scales to 0.96–0.98. Large travel and big rotations look cheap and slow.
5. **Choreography.** Related items stagger by 30–60 ms (cap the total at about 300 ms; past 8 items, animate the group). Hierarchy order: container → heading → primary content → secondary. One focal motion at a time.
6. **Interruptible and non-blocking.** Users never wait for an animation to finish before they can act; a new trigger reverses or retargets the running motion from its current value.
7. **Performance.** Animate `transform` and `opacity` (and colour); avoid animating width, height, top/left, box-shadow blur or layout on large areas. Use layout or shared-element transitions from the platform instead of animating size by hand.
8. **Accessible.**
   - Every motion has a **reduced-motion alternative** (`prefers-reduced-motion`, iOS Reduce Motion, Android "Remove animations"): replace movement and scale with a short opacity fade (or an instant change); keep the information.
   - No auto-playing motion longer than 5 s without pause / stop / hide (WCAG 2.2.2); nothing flashes more than 3 times a second (2.3.1); avoid large parallax and zoom (vestibular triggers, 2.3.3).
   - Motion never carries meaning alone: a success animation comes with text or an icon and a screen-reader announcement.
   - Focus moves and announcements happen when the motion starts, not when it ends.
9. **Consistency.** The same kind of change always uses the same named motion. Motions are design-system assets, not per-screen improvisation.

## 2. Motion tokens

Defined in System mode, in the `Motion` variable collection (and `tokens_path`), with code syntax for each platform:

| Token | Value | Use |
|---|---|---|
| `motion/duration/instant` | 0.1 s | hover and colour changes, press down |
| `motion/duration/fast` | 0.15 s | press release, toggles, small icons |
| `motion/duration/base` | 0.25 s | most component transitions, fade-ups |
| `motion/duration/slow` | 0.35 s | dialogs, sheets, expand / collapse |
| `motion/duration/page` | 0.45 s | screen transitions, large surfaces |
| `motion/easing/standard` | cubic-bezier(0.2, 0, 0, 1) | moving on screen |
| `motion/easing/enter` | cubic-bezier(0, 0, 0.2, 1) (ease-out) | entering, responding |
| `motion/easing/exit` | cubic-bezier(0.4, 0, 1, 1) (ease-in) | leaving |
| `motion/easing/spring` | spring, bounce 0.15–0.25 | press release, drag, playful feedback |
| `motion/distance/sm` · `md` | 8 px · 16 px | entrance offsets |
| `motion/stagger` | 0.04 s | delay between related items |

Durations are `TIMING` variables (seconds), distances `FLOAT`. Create `EASING` variables when the file supports them; otherwise keep easing as documented tokens in `motion.md` and the DTCG file (`cubicBezier` type). Adjust values to the brand (calm vs energetic), but keep the scale small.

## 3. Naming: the motion catalog

Every motion has a stable, descriptive id: `motion/<category>/<name>`, lowercase kebab case, used **identically** in the catalog, the Figma custom animation style name, annotations, the screen JSON and code (`motionCatalog["enter/fade-up"]`, `--motion-enter-fade-up-*`).

Categories and the starter catalog (add, rename or drop to fit the product; reuse before creating):

| id | What it does | Trigger | Spec (from → to, duration, easing) | Reduced motion |
|---|---|---|---|---|
| `motion/feedback/press` | Button or card dips under the finger | press down / release | scale 1 → 0.97 (instant, enter), back to 1 (fast, spring) | none (keep colour state) |
| `motion/feedback/hover-lift` | Card lifts on hover (pointer devices only) | hover in / out | translateY 0 → -2 px, shadow elevation 1 → 2 (fast, enter) | shadow only |
| `motion/feedback/toggle` | Switch knob slides, track recolours | value change | knob translateX, track colour (fast, spring) | instant |
| `motion/feedback/shake-error` | Field nudges on rejected submit | submit with errors | translateX 0 → 6 → -6 → 3 → 0 px (0.3 s, standard) | border colour + icon only |
| `motion/feedback/copy-confirm` | Icon swaps to a check, then back | copy | cross-fade + scale 0.8 → 1 (fast, spring), revert after 1.5 s | cross-fade |
| `motion/enter/fade-up` | Content appears from slightly below | mount / screen enter / in view | opacity 0 → 1, translateY 12 → 0 px (base, enter) | opacity only |
| `motion/enter/stagger-list` | List items fade-up in sequence | first load | `enter/fade-up` per item, `motion/stagger` delay, total ≤ 0.3 s | group fade |
| `motion/enter/scale-in` | Popover, menu, tooltip grows from its anchor | open | opacity 0 → 1, scale 0.96 → 1, origin at anchor (fast, enter) | opacity only |
| `motion/exit/fade-down` | Content leaves | unmount / dismiss | opacity 1 → 0, translateY 0 → 8 px (fast, exit) | opacity only |
| `motion/overlay/sheet` | Bottom sheet slides up, scrim fades | open / close / drag | translateY 100% → 0 (slow, enter; spring when dragged), scrim opacity 0 → 0.4 | fade |
| `motion/overlay/dialog` | Dialog fades and scales in | open / close | opacity, scale 0.98 → 1 (base, enter); exit fast, exit | fade |
| `motion/state/skeleton-shimmer` | Loading placeholder shimmer | while loading > 300 ms | gradient translateX loop 1.2 s, linear | static placeholder |
| `motion/state/spinner-to-check` | Button loading → success | submit success | spinner fades, check path draws (path trim 0 → 1, base, enter) | instant check |
| `motion/state/expand` | Accordion / details opens | toggle | height via layout transition, chevron rotates 0 → 180° (slow, standard) | instant |
| `motion/state/count-up` | Number ticks to new value | value change | tween the number (base, enter) | instant |
| `motion/nav/push` | Screen enters from the trailing edge | navigate forward | translateX 24 → 0 px + fade (page, standard) | cross-fade |
| `motion/nav/shared-element` | Card expands into its detail screen | open detail | shared-element / layout transition (page, standard) | cross-fade |
| `motion/attention/badge-pop` | New badge or count appears | new item | scale 0.6 → 1 (fast, spring bounce 0.3) | appear |
| `motion/delight/success` | Completion moment (brand) | first booking, payment done | per brand; ≤ 1 s; once per flow | static illustration |

Each motion in `docs/design/motion.md` has a full spec block:

```markdown
### motion/enter/fade-up
- purpose: orientation — content arrives without a hard cut
- trigger: on-mount | on-screen-enter | in-view (once, 20% visible)
- targets: any block-level content; not for text inside buttons
- properties: opacity 0 → 1; translateY motion/distance/sm (12 px) → 0
- duration: motion/duration/base (0.25 s) · easing: motion/easing/enter · delay: 0
- stagger: when applied to siblings, + motion/stagger per item, total ≤ 0.3 s
- interruption: if unmounted mid-way, play motion/exit/fade-down from current values
- reduced motion: opacity only, motion/duration/fast
- a11y: no announcement; content is in the accessibility tree from the start
- figma: custom animation style "motion/enter/fade-up" (style id …), demo frame "MOTION: enter/fade-up"
- code notes: CSS @keyframes + transition tokens; React: motion.dev variants; RN: Reanimated entering FadeInUp; SwiftUI: .transition(.opacity.combined(with: .move(edge: .bottom)))
```

## 4. Where motion lives

**In Figma (when motion is supported)**
- Runtime check: motion APIs throw `"<name>" is not a supported API` when not enabled for the user. Probe once with a read-only call (`figma.motion.figmaAnimationStyles()`, `figma.getLocalCustomAnimationStylesAsync()`); on that error, stop using Figma motion, tell the owner once, and use the text catalog only. Custom animation styles can be disabled even when keyframes work: check separately.
- Load Figma's `figma-use-motion` skill with `figma-use` before any motion write and pass both in `skillNames`.
- **Reusable motions = local custom animation styles** in the library file, named exactly `motion/<category>/<name>`, composed of Figma's first-party definitions (`Opacity`, `Position`, `Scale`, `Rotation`, `Size`, `Path`, `TextStagger`). Read each definition's `props` before configuring entries; durations and easing live inside the entry props. Discover existing styles first and match by id, never create near-duplicates.
- **Node-specific choreography = manual keyframes** (`applyManualKeyframeTrack`) with public field names (`OPACITY`, `TRANSLATION_X/Y`, `SCALE_XY`, `ROTATION`, `PATH_TRIM_END`, fills / effects by index) and public easing names (`EASE_OUT`, `EASE_IN_AND_OUT`, `GENTLE`, `CUSTOM_CUBIC_BEZIER`, `CUSTOM_SPRING`, `HOLD`).
- Never animate a top-level frame (a direct child of a page); animate its children. Extend the frame's timeline with `setTimelineDuration` when the motion runs past it.
- **Figma motion is a timeline that loops**; it has no notion of hover, press or "on enter". So in Figma:
  - each motion gets a demo frame `MOTION: <category>/<name>` on the library's `Motion` page: the component or element at rest, the motion applied at a clear offset, a caption with the trigger and tokens, and an annotation linking the spec;
  - interaction-triggered motion (hover, press, open / close) is also expressed as component **variants** (default / hover / pressed / open) linked by prototype reactions with Smart Animate, the matching easing and duration, so it plays on interaction in prototype mode;
  - on screens, apply the motion (style or keyframes) to the elements that use it, offset on the timeline to show the choreography (stagger order), and record the trigger in the annotation and metadata.
- **Verify motion visually**: screenshots show only the resting state. For anything non-trivial, `export_video` the top-level frame (small width, low quality, fps 5–10), poll the job, then extract 4–6 frames with `ffmpeg -ss <t>` (start, mid, settle per phase) and check order, timing and end state. Batch all fixes before re-exporting.

**In text (always)**
- `docs/design/motion.md` is the source of truth: principles applied, tokens, and every catalog entry with its full spec block. Figma styles and demos mirror it; if Figma motion isn't available, the catalog alone is complete enough to build from.
- Component specs reference motions by id in a **Motion** section (state → motion id).
- Screens reference them in the screen JSON and the `Motion` annotations.

## 5. Handoff to the frontend

- The FE builds each catalog entry **once** (e.g. a `motion.ts` / CSS module of named presets keyed by id, tokens from the token pipeline) and applies it by id; it never copies per-screen animation code.
- `get_motion_context` on a Figma node returns code snippets (CSS `@keyframes`, motion.dev) and keyframe values. Use them for the **values and curves** (exact offsets, spring curves, path trims). Their timing is the frame's looping demo timeline: take **trigger, duration, delay, repeat and reduced motion from the catalog and the screen JSON**, not from the snippet (snippets default to `repeat: Infinity` over the frame's timeline length).
- The screen JSON lists every motion on the screen (see the contract): element, motion id, trigger, delay or stagger order, and overrides. Overrides are rare and must be justified in the feature doc.

## 6. Reviewing motion

Check, per screen or component:
- Every motion has a purpose, an id from the catalog and the right category; no orphan animations.
- Durations and easing come from tokens; nothing in a frequent task above 500 ms; exits faster than entrances.
- Only transform / opacity / colour animate (or platform layout transitions).
- Interruptible; no input blocked during motion; focus and announcements not delayed.
- Reduced-motion alternative defined and implemented; WCAG 2.2.2 and 2.3.1 respected; no meaning carried by motion alone.
- Choreography: one focal motion at a time; stagger total ≤ 0.3 s.
- Built UI matches the catalog values (record a short screen capture or step through with the browser's animation tools when reviewing a running app).

Findings use the normal review format with category `motion`.
