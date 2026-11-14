# HANDOFF — casehub-pages

## Status

On main. #653 (split/merge visual overlay) and #654 (drag-to-absorb merge) landed and closed. #608 (TS/Java YAML Runtime Parity) epic continues — next items in queue.

## What just shipped

#653/#654 — Split/merge visual overlay + drag-to-absorb merge. 43 commits compacted to 7 via semantic squash, landed as 5e0e868d.

Key features:
- Grid-item overlay toolbar: 2x3 spatial action grid (split-h, split-v, merge, delete, add-empty) with hover expand/collapse
- Drag-to-absorb merge: progressive targeting, CSS overlay for root/target highlighting, click-outside exit
- 60px grid row enforcement with visible split items (markdown content instead of typeless empty)
- Multi-page demo with pill tab switcher (Color Grid, Content Layout, Two Forms)
- Unified grid-item controls: ⋮ trigger and action buttons moved from SelectionOverlay into controller's drag handle bar — one DOM element, one display toggle, zero timing gaps
- Selection rectangle snaps on click (immediate `_applyPreviewHighlight`, 150ms handle delay to match slide)

Key decisions:
- Single ownership: grid-item controls belong to the controller, not the overlay. The overlay keeps only the selection outline for grid items.
- Event bridge pattern: controller dispatches `pages-grid-action` (bubbles, composed), shell translates to existing `selection-*` handlers on overlay-root.
- 150ms handle delay on new selection to roughly sync with the overlay's 150ms CSS transition. Not perfectly synchronized (different event sources), accepted as good enough.

## Filed Issues

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #660 | Sync handle bar and selection rectangle timing | S/Med | Enhancement — shared orchestrator for perfect sync |
| #661 | Tree selection doesn't show grid-item controls | S/Low | Bug — tree click doesn't trigger controller's _setSelectedEl |
| #662 | Overlay outline flashes at old position after cross-grid drag | S/Med | Bug — showBuilderOverlay 50ms setTimeout |
| #663 | Grid controller and selection overlay tests need DOM environment | S/Low | Testing — jsdom/happy-dom for vitest |

## Queue

| # | Issue | Scale | Notes |
|---|-------|-------|-------|
| #647 | Phase 6 — phase-aware agent simulation | M/Med | P6 from audit |
| #645 | Streaming simulation API | S/Low | Chunked corpus responses |
| #646 | Agent session showcase | S/Low | Multi-turn demo |

## Blog

`blog/2026-11-14-mdp01-overlay-dual-ownership.md` — "One Element, One Toggle": dual-ownership anti-pattern in overlay UIs, event bridge pattern, architectural lessons.

## References

- Audit: `docs/specs/2026-11-07-ts-platform-parity-audit.md`
- Epic: casehubio/casehub-pages#608
