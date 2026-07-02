
## 1. Restack the two FABs (WhatsApp + Developer Mode)

Symptom (IMG_9674): the `</>` disk and the WhatsApp disk sit on opposite corners of the screen and look unrelated. They also don't match in size or weight.

Fix:
- Merge both buttons into a single vertical stack anchored to the physical **bottom-right** (`right: 1rem; bottom: 1.25rem`, respects `safe-bottom`), WhatsApp on top, Developer Mode directly under it with a small 10px gap.
- WhatsApp stays `size-12` (48px). Developer Mode shrinks to `size-10` (40px) so it reads as the secondary action, with the icon proportionally scaled (`size-4`).
- Keep the WhatsApp green halo behind its own disk only (already outside the glass). Developer Mode keeps the plain `glass-strong` disk — no ring, no halo.
- To avoid divergence, extract a tiny `FabStack` wrapper in `src/components/fab-stack.tsx` that renders `WhatsAppFab` + `DevModeButton` as children. `WhatsAppFab` and `DevModeButton` drop their own `fixed` positioning (they become inline items in the stack) but keep every existing animation, halo, and drawer behavior untouched. `__root.tsx` renders `<FabStack />` in place of the two direct components.

## 2. Kill the tall empty gaps (IMG_9675 / IMG_9676)

Root cause: `Footer` has `mt-32` (128px) and every route ends with `pb-16` (64px) — combined ≈192px of empty air between the last card and the footer glass, which reads as a "dead zone" on short pages like `/contact` and `/dev`.

Fix (spacing only — no layout removed):
- `src/components/footer.tsx`: change `mt-32` → `mt-12 sm:mt-16`.
- `src/routes/contact.tsx`, `src/routes/dev.tsx`, `src/routes/projects.index.tsx`: change the section's `pb-16` → `pb-8 sm:pb-10`.
- `src/routes/projects.$slug.tsx`: change the final section's `py-10 pb-20` → `py-8 pb-10`.
- `src/routes/about.tsx`: leave rhythm alone (already tight) but change last section `py-10 sm:py-14` → `py-8 sm:py-10` to match the same footer approach.
- Home (`src/routes/index.tsx`) stays with its `py-12 sm:py-16` rhythm since sections there stack — only the last-child gap changes via the new footer margin.

Net effect: the empty band shrinks from ~192px to ~64–96px — content-to-footer feels intentional instead of blank.

## 3. Stop the device heating / freezing

The heat comes from three always-on GPU/CPU consumers on the home page, plus a couple of everywhere loops. We reduce cost dramatically **without removing any effect** — every animation, the 3D torus, glass, gradients, Lenis, and the intro all stay visible and identical to a casual viewer.

### 3a. Hero WebGL (`src/components/hero-canvas.tsx`) — biggest offender
- Lower `dpr` from `[1, 1.6]` → `[1, 1.25]` (still crisp on retina, ~40% fewer fragment ops).
- Add `frameloop="demand"` on the Canvas and drive one frame per RAF via a tiny `useFrame(() => invalidate())` only while visible — the transmission material and rotation still animate smoothly, but React-Three-Fiber stops re-rendering when the tab is hidden or the hero is fully scrolled out of view. Combine with an `IntersectionObserver` gate: unmount `<HeroCanvas />` when the hero section leaves the viewport, remount when it re-enters. GPU idles the moment the user scrolls past the fold.
- Reduce `torusKnotGeometry` tessellation from `[1, 0.32, 220, 32]` → `[1, 0.32, 160, 24]` (visually identical at hero scale, ~45% fewer triangles).
- Drop `MeshTransmissionMaterial` `samples` from 6 → 4 and `temporalDistortion` from 0.2 → 0.15 (transmission still refracts; the shader loop is the single hottest cost).
- Add `<PerformanceMonitor>` from drei with a `factor` fallback that halves `dpr` if the device drops below 45fps — keeps the visual on strong devices, auto-cools weak ones.
- Add `power-preference: "low-power"` as a **fallback branch** when `navigator.hardwareConcurrency <= 4` OR `matchMedia("(pointer: coarse)").matches` (phones); desktops keep `high-performance`. No visual change; mobile GPU uses the efficiency core.

### 3b. Aurora & gradient loops
- `.aurora` and `.animate-gradient` currently animate `background-position` on a 200%×200% gradient — that composites the whole viewport every frame. Add `@media (prefers-reduced-motion: reduce)` fallback to freeze it (already partly there for Lenis; extend to CSS animations).
- Add `will-change: transform` + wrap the aurora blob in `transform: translateZ(0)` and animate `transform` instead of `background-position` where possible (CSS only, no visual change) — moves the work off the CPU compositor onto the GPU layer, cutting main-thread paint.
- Pause the aurora animation via `animation-play-state: paused` when the document is `hidden` (visibilitychange listener in `SmoothScroll` or a new 15-line `usePauseWhenHidden` hook).

### 3c. Lenis + RAF
- Guard the RAF loop in `SmoothScroll` with `document.visibilityState === "visible"` — currently the loop keeps ticking in background tabs. No behavior change when the tab is active.

### 3d. Framer Motion cost
- `SectionReveal` fires `whileInView` with blur filters on every section — blur is expensive. Keep the blur on desktop; on `(pointer: coarse)` OR reduced-motion, swap the blur for a plain opacity/translate (visually near-identical on small screens where blur is barely perceptible anyway). No feature loss.

### 3e. Analytics/vitals
- `@vercel/speed-insights` and `@vercel/analytics` load synchronously. Wrap both in `React.lazy` + `Suspense` so hydration finishes before they boot. Removes a measurable INP hit already logged in the console.

Together these changes typically drop steady-state GPU load 60–80% and eliminate the "hot phone" symptom while keeping every visual identical. Nothing is deleted.

## 4. Lighthouse pass + preload wins

- Run headless Lighthouse mobile against `http://localhost:8080` for `/`, `/about`, `/projects`, `/contact`, save JSON to `/tmp/browser/lh/`, and act on flagged items only.
- Confirmed easy wins to apply proactively:
  - Preload the Instrument Serif 400 woff2 that the H1 uses (`<link rel="preload" as="font" crossorigin>` in `__root.tsx`'s `head().links`).
  - Add `decoding="async"` and `loading="lazy"` to the portfolio gallery `<img>`s in `projects.$slug.tsx` (except the first, which stays eager for LCP on the case-study page).
  - Set `content-visibility: auto` on the very tall, off-screen sections in `index.tsx` (`about`, `projects preview`, `stack`, `testimonials-esque`, `cta`) to defer their paint work — pure CSS, no visual change, huge INP win.
- Report the four core scores back to the user after the run.

## 5. Code review sweep (non-destructive)

- `src/routes/__root.tsx`: the `theme-color` + `viewport-fit=cover` block is fine; verify the `dark` class is still applied by `themeBootstrap` after the FabStack refactor (no change expected, just a sanity check).
- `src/routes/dev.tsx`: STACK/ARCH arrays duplicate the ones in `src/components/dev-mode.tsx`. Extract to `src/lib/dev-manifest.ts` (new file) and import from both places — behavior unchanged, but the two lists can no longer drift.
- `src/components/hero.tsx`: the `chips` array is inline; not a bug, just a note — leave as is unless larger refactor asked.
- `src/components/section-reveal.tsx`: verify it respects `prefers-reduced-motion` (per §3d).
- Look for straggling console warnings after the changes (dev server logs) and address any that appear.
- No package removals this turn (last turn already removed 9). Do a fresh `rg` scan for zero-import packages only if new dead code surfaces; otherwise leave dependencies untouched.

## 6. Verification

- `bun run build` must pass.
- Playwright script (mobile 390×844, desktop 1280×900, EN + AR):
  - Screenshot `/`, `/about`, `/projects`, `/projects/portfolio`, `/contact`, `/dev` — save to `/tmp/browser/verify/`. Confirm the empty band under content shrank and the two FABs are stacked bottom-right with WhatsApp on top.
  - Open the Dev Mode drawer — confirm still bottom-centered.
  - Assert zero horizontal overflow (`scrollWidth === clientWidth`) — regression guard from last turn.
- Chrome DevTools performance manual: scroll home page for 5s, confirm frame time stays under 16ms on desktop after the R3F changes; GPU process CPU drops noticeably in Task Manager.
- Report Lighthouse scores back.

## Out of scope

No feature removals. No copy rewrites. No dependency deletions. No design-token recoloring. The intro, the 3D hero, glass, aurora, Lenis, i18n, all animations, and every route stay exactly as they are — only cheaper.
