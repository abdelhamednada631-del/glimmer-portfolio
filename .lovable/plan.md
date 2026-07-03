
## 1. Lighthouse + Core Web Vitals report

- Build the app (`bun run build`) and serve the production output locally on `http://localhost:4173`.
- Run headless Lighthouse (mobile preset, throttled) via `npx lighthouse` against `/`, `/about`, `/projects`, `/projects/portfolio`, `/contact`, `/dev` — both EN and AR. Save the JSON + HTML reports to `/tmp/browser/lh/`.
- Extract for each route: Performance score, LCP, TBT, CLS, INP (from the trace), TTFB, and the top 5 flagged opportunities/diagnostics.
- Also capture live Core Web Vitals via `web-vitals` in a Playwright run (real user timing after scroll/interaction) so INP is real, not synthetic.
- Report back in a single clean table:

  ```
  Route      Perf  LCP    TBT    CLS    INP    Top issues
  /          --    --     --     --     --     …
  /projects  --    --     --     --     --     …
  ```

- For every issue flagged (unused JS, render-blocking, image sizing, main-thread work, unused CSS, LCP element, etc.), list it with severity and the file it points to. No fixes applied in this pass — the report is the deliverable. Any fix that requires design or behavior changes is called out for approval before touching code.

- The only auto-fixes applied in this pass are strictly "zero-risk, zero-visual-change":
  - Add `<link rel="preload" as="font" type="font/woff2" crossorigin>` for the Instrument Serif 400 file the H1 uses (in `__root.tsx` head links), only if Lighthouse flags "font not preloaded" or LCP is font-blocked.
  - Add `decoding="async"` / `loading="lazy"` to non-LCP `<img>` in `projects.$slug.tsx` gallery, only if flagged.
  - Add `content-visibility: auto; contain-intrinsic-size: 800px` on the very tall off-screen sections in `index.tsx`, only if TBT/INP flags long paints.

  Anything else stays as a written recommendation for user approval.

## 2. Normalize icon/FAB spacing (visuals unchanged, only rhythm)

Symptom: the two FABs stack correctly, but the gap between them and the gap under the WhatsApp halo don't read as intentional. Also spot-checked chip/icon rows in `hero.tsx`, `section-header.tsx` kicker dot, footer socials — a couple use ad-hoc gap values that don't match the rest of the site.

Non-destructive spacing normalization (no design, color, or size changes):

- **`src/components/fab-stack.tsx`**: change the flex column gap from `gap-2.5` (10px) to `gap-3` (12px) so the two disks breathe evenly; the WhatsApp halo (blur-xl) currently visually overlaps the top of the Dev Mode disk at `gap-2.5`. Keep `bottom-5 right-4` and `safe-bottom` exactly as-is.
- **`src/components/whatsapp-fab.tsx`**: the halo `<span>` uses `inset-0` + `blur-xl`, which extends ~24px past the disk. Constrain the halo to the disk footprint by adding `inset-[-2px]` → `inset-0` stays but the blur radius drops from `blur-xl` to `blur-lg` **only** if step above still shows halo bleed. Zero visual change to the disk itself — this is only to keep the halo from crowding the second FAB.
- **`src/components/dev-mode.tsx`** (button portion only): confirm it drops its own `fixed`/positioning and inherits from FabStack (should already be the case). If any leftover margin (`mb-*`, `mt-*`) exists on the button root, remove it so FabStack's `gap-*` is the single source of truth.
- **`src/components/hero.tsx`** chips row: unify to `gap-2` across all breakpoints (currently mixes `gap-2` and `gap-3` at different breakpoints per read).
- **`src/components/footer.tsx`** social icons row: unify to `gap-3` (matches nav's icon rhythm).
- **`src/components/section-header.tsx`** kicker: no change (already `gap-2`).

Rule enforced (mental grid): FABs = `gap-3`, inline icon rows = `gap-2`, icon-with-label rows = `gap-3`. Applied everywhere it currently drifts; nothing else touched.

## 3. Verification

- `bun run build` clean.
- Playwright 390×844 + 1280×900, EN + AR: screenshot bottom-right corner of `/`, `/about`, `/projects`, `/contact`, `/dev`. Confirm:
  - WhatsApp halo no longer touches the Dev Mode disk.
  - Gap between the two FABs looks intentional and identical on every route.
  - Chip and social icon rows have consistent spacing.
- Report Lighthouse table back to the user with issues + recommended (but not-yet-applied) fixes for their approval.

## Out of scope

- No visual redesign, no icon/color/size changes to the FABs themselves.
- No feature removals, no dependency changes.
- Any Lighthouse fix that requires touching design or behavior is written as a recommendation, not silently applied.
