# Design Journal — issue-434-visual-yaml-builder-phase1b

## 2026-09-14 — Phase 1b implementation complete

### What was built

Visual YAML Builder Phase 1b — three workstreams completing the authoring
experience beyond the Phase 1 (#428) foundation.

**Structural Editing (#433):** Transaction API for atomic undo on compound
operations. Seven new facade operations: `addChild` (all 3 container slot
kinds), `wrapIn` (container wrapping with `defaultContentSlot`), `wrapInRow`
(page-level restructuring with flat→rows migration), `replaceWith` (type
swap with top-level property migration), `moveToSlot` (named-record
container targeting), `insertChildAt`/`insertComponentAt`/`insertColumnAt`
(positional insertion). Tree UI wired with `+` buttons on containers,
right-click context menu, keyboard shortcuts (Delete, Ctrl+D,
Ctrl+Shift+P, Ctrl+Shift+Up/Down), and drag-and-drop with drop indicators.

**Preview Data (#432):** Strategy registry with 4 strategies covering all
20 `NEEDS_DATASET_TYPES` entries. Column-aware generation (TEXT→strings,
NUMBER→numbers, DATE→dates). YAML transformation for preview — replaces
URL datasets with inline content before rendering, no runtime changes
needed. `previewHints` on catalog entries for container stub children and
content placeholders.

**Dock-Workbench (#429):** `<pages-dock-workbench>` Lit component with
named slots (left/centre/right/bottom), zone enable/disable, resize
handles, collapse/expand, localStorage persistence. Builder shell
migrated from bespoke dock layout to the component. Source/Split/Visual
column-based editor layout preserved in the centre panel.

### Key decisions

- Transaction API: `beginTransaction`/`commitTransaction`/`abortTransaction`
  with notification suppression during compound operations
- Hybrid preview strategy: strategies only for data-consuming types,
  render-as-is for everything else
- Dock scoped to builder only — runtime's 6-zone `ZoneLayoutEngine` is a
  separate, more complex system (not unified)
- `defaultContentSlot` on `ContainerChildDescriptor` — sidebar wraps into
  `content` slot, not `sidebar` slot

### Open issue

**#436 — Dock-workbench Lit component needs rework.** The current
implementation builds its own toggle bar and event contract instead of
wrapping the existing runtime `renderDockBar()`, `DockBarProps`, `DockItem`,
and `pages-dock-toggle` event infrastructure. This creates a dual
architecture. The Lit component should be a thin wrapper around the
existing dock infrastructure, not a parallel system.

### Stats

- 15 commits on branch
- 101 new tests (92 facade + 9 existing = total 92 in pages-document, 187 in pages-builder, 86 in pages-primitives)
- Net code: ~2,400 lines added across 3 packages
