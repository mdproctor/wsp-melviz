# Structural Editing — Cut/Copy/Paste + Visual Add/Insert

**Issue:** #433 (Tree/visual structural editing)
**Branch:** issue-439-dock-workbench-polish
**Date:** 2026-09-17

## Overview

Add cut/copy/paste-based structural editing to the YAML builder's tree and visual views. Replace the existing drag-and-drop with a clipboard-driven approach that works across files and shows guided valid-target highlighting.

## Architecture

### Global BuilderClipboard

A singleton state object shared across all builder views:

```typescript
interface BuilderClipboard {
  fragment: string | null;         // YAML text (also on system clipboard)
  fragmentType: string | null;     // 'component', 'row', 'column', etc.
  operation: 'cut' | 'copy' | null;
  insertMode: boolean;             // are valid targets highlighted?
}
```

- Lives in a new module: `packages/pages-builder/src/clipboard/builder-clipboard.ts`
- Observable — views subscribe to changes via a callback pattern
- `insertMode` is the transient UI state; clipboard content persists independently
- System clipboard holds the raw YAML text for cross-app/cross-file paste

### Tree View — Button Layout

Every non-section tree node gets four inline buttons, opacity-hidden until hover:

```
[▸] Label              [+] [↓] [✂] [⎘]
                         │   │   │   └─ Copy
                         │   │   └─ Cut
                         │   └─ Insert (before/after sibling)
                         └─ Add child (containers only)
```

- **Add (+):** Only rendered on container nodes (page, row, column, container components). Opens picker → appends as child. If clipboard has content and insert mode is active → pastes as child.
- **Insert (↓):** On every node. Prompts before/after, then opens picker → inserts as sibling. If clipboard has content and insert mode is active → prompts before/after → pastes.
- **Cut (✂):** Serializes node to YAML, writes to system clipboard, deletes node from document (undo-able), sets global clipboard state, enters insert mode.
- **Copy (⎘):** Serializes node to YAML, writes to system clipboard, sets global clipboard state, enters insert mode. Source node unchanged.

All buttons use `opacity: 0` / `opacity: 1` on hover (existing pattern from the tree hover-dancing fix).

### Insert Mode (Guided Paste)

When insert mode is active (`clipboard.insertMode === true`):

1. **Clipboard indicator:** Small banner at top of tree panel showing fragment type and ✕ dismiss. Example: `📋 metric  ✕`
2. **Valid targets highlight:** Tree nodes that can accept the fragment type show their Add/Insert buttons in accent color. Invalid nodes dim their buttons.
3. **Click a highlighted target:** Execute paste with position strategy (before/after for Insert, child-append for Add).
4. **Esc:** Sets `insertMode = false` globally. Clipboard content retained. Targets un-highlight.
5. **Re-enter:** Click Add/Insert on any node while clipboard has YAML content → re-enters insert mode.
6. **✕ on indicator:** Clears fragment from global state. Add/Insert revert to picker behavior.

### Validation Rules

Fragment type determines valid targets (reuses logic from `tree-dnd.ts` and `component-catalog.ts`):

| Fragment Type | Valid Add Targets | Valid Insert Targets |
|---------------|-------------------|---------------------|
| component | page, column, container-component | sibling of any component |
| row | page (rows mode) | sibling of any row |
| column | row | sibling of any column |

Cycle prevention: cannot paste a node into its own descendants (same as existing DnD validation).

### Cut/Copy Serialization

```typescript
function serializeNode(doc: PageDocument, path: readonly (string | number)[]): string
```

- Uses the YAML library's `toString()` on the node at the given path
- Preserves indentation and comments
- Writes result to system clipboard via `navigator.clipboard.writeText()`
- Stores parsed metadata in global `BuilderClipboard` state

### Paste Deserialization

```typescript
function parseClipboardFragment(yaml: string): { type: string; node: unknown } | null
```

- Parses YAML text
- Detects fragment type from structure (has `type:` key → component, has `columns:` → row, has `span:` → column, has `rows:` or `components:` → page)
- Returns null for non-YAML or unrecognized content

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+X` | Cut selected node |
| `Ctrl+C` | Copy selected node |
| `Ctrl+V` | Enter insert mode (if clipboard has YAML) or paste at selected position |
| `Esc` | Exit insert mode (retain clipboard) |
| `Delete` / `Backspace` | Delete selected node (existing) |
| `Ctrl+D` | Duplicate selected node (existing) |

### Visual Mode Overlay (Phase 2)

The visual preview renders via `loadSite()` — we don't control its DOM. To add + buttons:

1. After preview renders, query all `[data-component-type]` elements
2. Create transparent overlay `div`s positioned via `getBoundingClientRect()`
3. Each overlay shows a + button (top-right corner) on hover
4. Clicking + opens the same picker / paste flow as tree view
5. Overlays recompute on scroll, resize, and preview re-render
6. In insert mode, overlays highlight valid targets the same as tree nodes

## Implementation Phases

### Phase 1: Tree Cut/Copy/Paste
1. `BuilderClipboard` singleton with subscribe/notify
2. Serialize/deserialize YAML fragments
3. Tree buttons: Add, Insert, Cut, Copy (replace existing `_handleAddClick`)
4. Insert mode UI: clipboard indicator, target highlighting, Esc handling
5. Position strategy popup (before/after/child)
6. Keyboard shortcuts
7. Remove existing DnD code from tree

### Phase 2: Visual Overlay
1. Component overlay positioning engine
2. + button on overlay hover
3. Insert mode target highlighting in visual view
4. Cross-view sync (cut in tree, paste targets in visual)

## Files to Modify

| File | Changes |
|------|---------|
| `packages/pages-builder/src/clipboard/builder-clipboard.ts` | **New** — global clipboard state |
| `packages/pages-builder/src/clipboard/yaml-fragment.ts` | **New** — serialize/deserialize |
| `packages/pages-builder/src/tree/builder-tree.ts` | Add Insert/Cut/Copy buttons, insert mode highlighting, remove DnD |
| `packages/pages-builder/src/tree/tree-dnd.ts` | Remove (replaced by clipboard) |
| `packages/pages-builder/src/shell/builder-shell.ts` | Wire clipboard, handle paste, keyboard shortcuts |
| `packages/pages-builder/src/palette/inline-picker.ts` | Support position strategy popup |

## Testing Strategy

- Unit tests for `BuilderClipboard` state transitions
- Unit tests for YAML fragment serialization/deserialization (round-trip)
- Unit tests for validation rules (fragment type → valid targets)
- Integration tests for tree button clicks → clipboard state → paste execution
- Visual regression via Playwright for insert mode highlighting

## References

- `packages/pages-builder/src/tree/builder-tree.ts` — existing tree component
- `packages/pages-builder/src/tree/tree-dnd.ts` — existing DnD (to be replaced)
- `packages/pages-document/src/page-document.ts` — `ComponentNode.moveToIndex()`, `addComponent()`
- `packages/pages-builder/src/catalog/component-catalog.ts` — type validation
- `packages/pages-builder/src/palette/inline-picker.ts` — existing picker
- Issue #433 — Tree/visual structural editing
- Issue #426 — Visual YAML Builder epic
