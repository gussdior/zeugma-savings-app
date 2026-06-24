# Bundle Redesign — Design Spec

**Date:** 2026-06-24
**Status:** Self-approved (owner delegated full autonomy: "finish to perfection, go by recommendation, don't wait")

## Context

Zeugma sells bundle packages often, and the owner reported the bundle screen looked
"broken" with wrong numbers. Investigation found no crash and no math error with the seed
prices, but two real weaknesses: (1) the headline savings sum unrounded per-service values
while each card rounds, so the figures can fail to add up in some price configs; and (2)
the bundle screen is static and read-only, using each service's dashboard session count
with no way to adjust per client. The owner wants bundles to "work like the single service
page, with more detail."

The single-service screen (`#client`) is interactive: a savings hero with count-up, a
Regular-vs-Package comparison card, a live 1..N session selector + tier ladder, and a CTA
that updates as sessions change. The bundle screen (`#bundleClient`) has only a savings
hero, static cards, and a CTA.

## Goals

1. **Correctness:** bundle headline totals always equal the sum of the rounded per-service
   dollars shown on the cards. No "doesn't add up."
2. **Interactive like single-service:** each treatment in the bundle has a live session
   control; changing it updates that treatment plus the combined hero, comparison, and CTA
   in real time.
3. **More detail:** add a combined comparison (Total regular vs Bundle price vs You save)
   and show per-treatment package total, regular (struck), savings, and per-session price.
4. **Pricing untouched:** keep the exact `calc()` model. Do NOT invent a bundle-level
   discount (that would change what clients pay). Flag it as an optional future setting.

## Decisions (autonomous, by recommendation)

- **Session control per treatment:** a stepper (− N +), range 1..service.maxSessions,
  default maxSessions (mirrors single-service, which defaults to the best-value max). A
  stepper (not the 1..N segmented track) because multiple cards must stay compact; it also
  matches the dashboard's existing session stepper. A "Best value" tag shows when the
  treatment is at its max.
- **Bundle session state:** module-level `bundleSessions = {}` keyed by service key.
  `bundleSessFor(s)` returns `clampSess(bundleSessions[s.key] ?? s.maxSessions)`. Adjusting
  a card writes to this map; it does NOT mutate the service's dashboard `maxSessions`.
- **Rounded totals:** `bundleTotals()` sums `Math.round()` of each service's `packageTotal`,
  `regGross`, `totalSave`, `packageSave` so the headline equals the sum of displayed cards.
- **Invoice consistency:** `invoiceForBundle()` uses `bundleSessFor()` and the same rounded
  sums, so a bundle invoice matches the adjusted on-screen bundle exactly.
- **Combined comparison:** reuse the single-service `.compare`/`.cmp`/`.vs`/`.cmp-right`
  classes for visual consistency: Total regular price (struck) VS Bundle price, with
  Package savings and Total savings on the right.

## Components (all in `index.html`)

1. **Markup:** add a combined comparison block (`#bCompare`) between `.bhero` and `#bList`
   in `#bundleClient`. Keep the existing hero and CTA bar.
2. **State + math:** `bundleSessions`, `bundleSessFor()`, rounded `bundleTotals()`.
3. **Render:** rewrite `buildBundleClient()` to render interactive treatment cards (name +
   promo tag, session stepper, package/regular/save/per-session), wire steppers, then call
   `updateBundle()`.
4. **Live update:** new `updateBundle()` recomputes from `bundleSessFor`, updates each
   card's figures + best-value tag, the hero (animated count-up), the comparison block, and
   the CTA. Called on open and on every stepper change.
5. **Invoice:** `invoiceForBundle()` reads `bundleSessFor()` and rounded sums.
6. **CSS:** styles for the enriched card internals (`.bc-top`, `.bc-figs`, `.bc-fig`,
   per-card stepper) and the comparison block spacing; reuse existing palette + classes.

## Out of scope

- A bundle-level extra discount (pricing decision; left as a flagged recommendation).
- Persisting bundle session choices across app reloads (kept in-memory like the rest of the
  client-facing flow).
- Reworking the single-service screen.

## Verification

Browser (Playwright, landscape 1180x820):
1. Select 2-3 services, open bundle. Each treatment shows a stepper, package/regular/save/
   per-session, and a Best-value tag at max.
2. Lower one treatment's sessions: that card's figures, the hero count, the comparison, and
   the CTA all update, and remain internally consistent (sum of cards == headline).
3. Per-treatment figures equal the single-service screen's figures for the same service at
   the same session count.
4. "Send invoice" from the bundle uses the adjusted sessions; invoice totals tie out to the
   bundle screen.
5. No console errors; layout holds for 2 and 5 services (scroll if needed).
