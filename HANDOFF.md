# HANDOFF — casehub-pages

## Last Session

Closed #587 (race and select step kind rendering). Built two new Lit components:
- `race-view.ts` — swim-lane grid columns with amber header and "first to complete wins" badge
- `select-view.ts` — row-based dispatch table with purple subscribe / teal wait type badges

Also exported `RaceStep` from yaml-core (was missing from public API), wired both into step-dispatch, and added gallery sections 31-32. Landed as e30aa813 on main.

## Immediate Next Step

Start #563 — Unified semantic diff overlay for step viewer (scale: L, complexity: High). This is the last remaining v1 child of epic #558. Some groundwork exists: `swim-lane-view` and `step-card` already have `diffState` props, and gallery section 21 demonstrates adding a 4th branch with diff overlays. The task is extending diff support uniformly across all components and adding semantic diff computation.

## References

- `docs/mockups/step-review-viewer.html` — design mockup (sections 1-12 pages scope)
- `docs/specs/2026-10-17-step-review-viewer-design.md` — design spec
- `packages/pages-step-viewer/gallery.html` — component gallery (sections 24-32)
- Epic #558 — only #563 remains open in v1 tier
