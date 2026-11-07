# HANDOFF — casehub-pages

## Status

On `issue-618-grid-edit-placeholder-drag`. POC core validated, continuing POC iteration before framework integration.

## What happened this session

Built standalone POC (`examples/poc/grid-drag-poc.html`) proving placeholder-based grid drag with structural CSS Grid items. Iterated through 4 algorithm versions based on live UX feedback. Final single-rule compaction algorithm (15 lines, 40 tests) handles move, resize, expand, shrink, and delete with round-trip stability.

Key decisions: no absolute positioning (CSS Grid only), pluggable compactors (vertical/float), drag handle on title bar (24px), two-stage delete, sizing modes (proportional/fixed width, rows/fixed/content height). Upgraded Playwright to 1.65.0 — found `viewport: null` fix for macOS Retina select popup displacement.

## Next action

Continue POC iteration — the core interaction (move, resize, delete, compaction, sizing modes) is validated, but several features need building and validating in the POC before framework integration.

## POC items remaining (in priority order)

1. **Collision popup** — build the split/replace/swap/displace popup for `prompt` mode on occupied-cell drop. Algorithm is defined, UI not built.
2. **Top resize upward** — broken in POC. Dragging top border upward doesn't resize correctly. Needs debugging.
3. **Row breaks** — design question: how to add logical row grouping to a flat CSS Grid. Three candidates: full-width break item (`grid-column: 1/-1`), row container with internal flexbox, or CSS Subgrid. Need to POC the preferred approach. User leaning toward "a block IS the break."
4. **Rows/columns layout relationship** — the other layout system has more limitations but better reflow. How do the two relate? Can a row container be a grid item? This bridges the two systems.
5. **Expand to fill / sticky** — discussed features. Expand: panel fills available space (revertible). Sticky: panel is immovable, others reflow around it.
6. **Undo/redo** — layout state stack. Essential for any editor.
7. **Sizing modes validation** — added to POC but need more browser testing with container resize.

## After POC is complete

Rewrite implementation plan based on POC learnings, then integrate into `grid-edit-controller.ts`. Stale plan at `plans/2026-11-06-grid-edit-placeholder-drag.md` predates POC.

## References

- POC: `examples/poc/grid-drag-poc.html`
- Algorithm: `packages/pages-component/src/renderer/grid-compaction.ts` (40 tests)
- Spec: `specs/issue-618-grid-edit-placeholder-drag/2026-11-06-grid-edit-placeholder-drag-design.md`
- Decisions: `specs/issue-618-grid-edit-placeholder-drag/decisions.md` (D1-D8)
- Blog: `blog/2026-11-07-mdp01-tetris-not-pixels.md`
