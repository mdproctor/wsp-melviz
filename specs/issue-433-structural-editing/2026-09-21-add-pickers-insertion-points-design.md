# Contextual Add Pickers + Position-Complete Insertion

**Issue:** #433 (Tree/visual structural editing)
**Branch:** issue-433-structural-editing
**Date:** 2026-09-21
**Depends on:** SelectionOverlay (D1-D4, implemented)

## Overview

Add schema-constrained type filtering to component insertion, and provide
position-complete insertion points (N+1 positions) in both tree and visual
preview. Users click `+` at an explicit insertion point and see only the
component types valid for that position in the document hierarchy.

Three layers of change:
1. **Constraint function** (`pages-document`) — answers "what can be added here?"
2. **Insertion points** (`pages-builder`) — `+` indicators at all valid positions
3. **Picker integration** — hard-filter by constraints, soft-order by relevance

## 1. Constraint System

### 1.1 `insertion-constraints.ts` in `pages-document`

```typescript
export interface InsertionConstraint {
  readonly structuralTypes: readonly string[];  // 'row', 'column'
  readonly componentTypes: readonly string[];   // all 55 component types, or empty
}

export function allowedTypesAt(parentNodeType: string): InsertionConstraint;
export function allowedTypesForSiblingOf(nodeType: string): InsertionConstraint;
```

### 1.2 Constraint rules

| Parent type | `structuralTypes` | `componentTypes` |
|------------|-------------------|------------------|
| `page` | `['row', 'column']` | all 55 |
| `row` | `['column']` | `[]` |
| `column` | `[]` | all 55 |
| container component (in `CONTAINER_DESCRIPTORS`) | `[]` | all 55 |
| leaf component | `[]` | `[]` |
| unknown | `[]` | `[]` |

`allowedTypesForSiblingOf` derives the parent:
- component → column's constraint
- column → row's constraint
- row → page's constraint

### 1.3 Integration with existing filtering

The inline picker receives the constraint set as a hard filter:

```
Catalog (55 entries)
  → hard-filter by allowedTypesAt(parentType) — removes structurally invalid types
  → soft-order by contextRelevance(ctx) — promotes/demotes by editorial relevance
  → display in picker
```

`PaletteContext` gains a new field:

```typescript
export interface PaletteContext {
  // existing
  parentType: string | undefined;
  parentSlot?: string | undefined;
  acceptsComponents: boolean;
  availableDatasets: string[];
  siblingTypes: string[];
  // new
  allowedTypes?: InsertionConstraint;
}
```

The inline picker and palette check `allowedTypes` first (hard filter), then
apply `contextRelevance` within the allowed set. When `allowedTypes` is
undefined, behavior is unchanged (backwards compatible).

## 2. Tree Insertion Points

### 2.1 Rendering

When a parent node is expanded, the tree renders insertion-point indicators
at all N+1 positions between its children:

```
▶ Column                    [+ ↓ ✂ ⎘]
  ─── + ───                            ← position 0
  ├─ bar-chart              [↓ ✂ ⎘ ✕]
  ─── + ───                            ← position 1
  ├─ data-table             [↓ ✂ ⎘ ✕]
  ─── + ───                            ← position 2
```

Each insertion-point indicator is a thin horizontal line with a centered `+`
icon. Default state: low opacity (0.3). On hover: full opacity, `+` icon
highlighted in accent color. On click: dispatches event and opens picker.

### 2.2 DOM structure

```html
<div class="tree-insertion-point" data-index="1" role="button"
     aria-label="Insert component at position 1">
  <span class="insertion-line"></span>
  <span class="insertion-icon">+</span>
  <span class="insertion-line"></span>
</div>
```

### 2.3 Events

Clicking an insertion point fires `tree-insert-at`:

```typescript
interface TreeInsertAtDetail {
  parentPath: readonly (string | number)[];
  index: number;
  parentNodeType: TreeNodeType;
  target: HTMLElement;  // the + element, used as picker anchor
}
```

This bypasses the position picker — the position is already explicit.

### 2.4 Which parents show insertion points

| Parent type | Shows insertion points | Children type |
|------------|----------------------|---------------|
| page (expanded) | yes | rows, columns, or components |
| row (expanded) | yes | columns |
| column (expanded) | yes | components |
| container component (expanded) | yes | components (per slot) |
| leaf component | no | — |
| section headers (Datasets, Variables, etc.) | no | — |

## 3. Visual Preview Insertion Points

### 3.1 Trigger

Insertion points appear when a container is selected in the visual preview:

1. User clicks a container (column, page, container component) in preview
2. `builder-shell` calls `selectionOverlay.update(bounds, nodeType, path, { isContainer: true })`
3. `builder-shell` queries child component bounds from the preview iframe
4. `builder-shell` calls `selectionOverlay.showInsertionPoints(childBounds, parentPath, parentNodeType)`

### 3.2 SelectionOverlay extensions

```typescript
class SelectionOverlay {
  // existing
  update(bounds: DOMRect, nodeType: TreeNodeType,
         path: readonly (string | number)[],
         options?: { isContainer?: boolean }): void
  hide(): void
  setInsertMode(fragmentType: string): void
  clearInsertMode(): void
  dispose(): void

  // new
  showInsertionPoints(childBounds: DOMRect[],
                      parentPath: readonly (string | number)[],
                      parentNodeType: TreeNodeType): void
  hideInsertionPoints(): void
}
```

### 3.3 Rendering

Each insertion point is rendered in the overlay-root as an absolutely-positioned
element between two child bounding rects:

```html
<div class="visual-insertion-point" data-index="1"
     role="button" aria-label="Insert component at position 1"
     style="position: absolute; left: Xpx; top: Ypx; width: Wpx;">
  <span class="insertion-line"></span>
  <span class="insertion-icon">+</span>
  <span class="insertion-line"></span>
</div>
```

Positioning:
- **Horizontal position:** matches the scope outline's left edge and width
- **Vertical position:** midpoint between child[i].bottom and child[i+1].top
- **Position 0:** at the scope outline's top content edge (below any padding)
- **Position N:** at the scope outline's bottom content edge

Default opacity: 0.3. On hover: full opacity with accent-colored `+`.

### 3.4 Events

Clicking dispatches `selection-insert-at` on the overlay-root:

```typescript
interface SelectionInsertAtDetail {
  parentPath: readonly (string | number)[];
  index: number;
  parentNodeType: TreeNodeType;
  target: HTMLElement;  // the + element for picker anchoring
}
```

### 3.5 Lifecycle

- `showInsertionPoints` creates/updates insertion-point DOM elements
- `hideInsertionPoints` removes them
- `hide()` calls `hideInsertionPoints()` internally
- `dispose()` cleans up all insertion-point DOM
- When `update()` is called with a non-container, insertion points are hidden

## 4. Builder-Shell Integration

### 4.1 New handler: `_handleInsertAt()`

```typescript
private _handleInsertAt(e: CustomEvent<{
  parentPath: readonly (string | number)[];
  index: number;
  parentNodeType: TreeNodeType;
  target: HTMLElement;
}>): void {
  const { parentPath, index, parentNodeType, target } = e.detail;

  const constraint = allowedTypesAt(parentNodeType);
  this._paletteContext = {
    parentType: parentNodeType,
    acceptsComponents: constraint.componentTypes.length > 0,
    availableDatasets: this._document.getDatasets().map(d => d.uuid),
    siblingTypes: [],
    allowedTypes: constraint,
  };

  this._inlinePickerOpen = true;
  this._inlinePickerPath = parentPath;
  this._inlinePickerNodeType = parentNodeType;
  this._inlinePickerAnchor = target;
  this._insertAtIndex = index;  // new state field
  this._syncTree();
}
```

### 4.2 New state field

```typescript
private _insertAtIndex: number | undefined;
```

Set by `_handleInsertAt`, consumed by `_handleInlinePickerSelect`, cleared
after insertion.

### 4.3 Mutation path in `_handleInlinePickerSelect`

Add a new branch at the top of `_handleInlinePickerSelect`, before the
existing `insertPos` branches:

```typescript
if (this._insertAtIndex !== undefined) {
  const index = this._insertAtIndex;
  this._insertAtIndex = undefined;
  this._applyEdit('tree', () => {
    if (nt === 'page') {
      const page = this._findPageAtPath(path);
      if (LAYOUT_TYPES.has(entry.type)) {
        page?.insertRowAt(index);
      } else {
        page?.insertComponentAt(index, entry.type, props);
      }
    } else if (nt === 'column') {
      this._findColumnAtPath(path)?.insertComponentAt(index, entry.type, props);
    } else if (nt === 'row') {
      this._findRowAtPath(path)?.insertColumnAt(index);
    } else if (nt === 'component') {
      // container component — insert into default slot at index
      const node = this._findComponentAtPath(path);
      node?.insertChildAt(index, entry.type, props);
    }
  });
  return;
}
```

### 4.4 Event wiring

In `_wireSelectionOverlayEvents`:

```typescript
overlayRoot.addEventListener('selection-insert-at', onInsertAt);
```

In template wiring for the tree:

```typescript
@tree-insert-at="${(e: CustomEvent) => this._handleInsertAt(e)}"
```

## 5. Mutation Layer

### 5.1 New methods on PageDocument model classes

**`PageEntry`:**
```typescript
insertComponentAt(index: number, type: string, props?: object): ComponentNode
```

**`RowNode`:**
```typescript
insertColumnAt(index: number): ColumnNode
```

**`ComponentNode` (containers only):**
```typescript
insertChildAt(index: number, type: string, props?: object, slot?: string): ComponentNode
```

Uses `getContainerDescriptor(this.type).defaultContentSlot` when `slot` is
omitted. Named-record containers (tabs, accordion, carousel, stack, menu,
tree) have individually named slots — slot-aware insertion (choosing which
tab/section to add to) is out of scope for this spec and left as follow-up.
The default slot handles the common case of "add to the currently visible
section."

### 5.2 Validation

All existing mutation methods (`addComponent`, `addRow`, `addColumn`,
`addChild`) and the new `insertAt` variants call `allowedTypesAt(this.type)`
and throw `InvalidInsertionError` if the requested type is not in the
allowed set.

```typescript
export class InvalidInsertionError extends Error {
  constructor(parentType: string, childType: string) {
    super(`Cannot insert '${childType}' into '${parentType}'`);
  }
}
```

## 6. Files Changed

| Package | File | Action |
|---------|------|--------|
| `pages-document` | `src/insertion-constraints.ts` | **New** — constraint functions |
| `pages-document` | `src/insertion-constraints.test.ts` | **New** — unit tests |
| `pages-document` | `src/page-document.ts` | Modify — add `insertComponentAt`, `insertColumnAt`, `insertChildAt`; add validation to existing mutation methods |
| `pages-builder` | `src/overlay/selection-overlay.ts` | Modify — add `showInsertionPoints`, `hideInsertionPoints` |
| `pages-builder` | `src/overlay/selection-overlay.test.ts` | Modify — tests for insertion points |
| `pages-builder` | `src/tree/builder-tree.ts` | Modify — render insertion-point indicators, dispatch `tree-insert-at` |
| `pages-builder` | `src/tree/builder-tree.test.ts` | Modify — tests for tree insertion points |
| `pages-builder` | `src/shell/builder-shell.ts` | Modify — `_handleInsertAt`, `_insertAtIndex` state, wire events |
| `pages-builder` | `src/catalog/palette-context.ts` | Modify — add `allowedTypes` to `PaletteContext` |
| `pages-builder` | `src/palette/inline-picker.ts` | Modify — hard-filter by `allowedTypes` |
| `pages-builder` | `src/palette/palette-filter.ts` | Modify — apply `allowedTypes` before `contextRelevance` |

## 7. Testing Strategy

- **Constraint function:** Unit tests for `allowedTypesAt` and `allowedTypesForSiblingOf` — verify all parent types return correct allowed sets
- **Mutation validation:** Unit tests for `InvalidInsertionError` — verify `addComponent('row')` on a column throws, `insertComponentAt` on a row throws, etc.
- **Tree insertion points:** Unit tests for DOM rendering — verify N+1 indicators for N children, correct `data-index`, event dispatch with correct detail
- **SelectionOverlay insertion points:** Unit tests for DOM rendering — verify positioning between child bounds, event dispatch, hide/show lifecycle
- **Integration (Playwright):** Select a column → verify insertion points visible → click `+` at position 1 → verify picker opens → select chart → verify chart inserted at position 1. Also test the tree path.
- **ARIA:** Insertion points have `role="button"` and `aria-label` — axe-core CI validates

## References

- `pages-document/src/container-descriptors.ts` — container type definitions
- `pages-document/src/page-document.ts` — existing mutation methods
- `pages-builder/src/overlay/selection-overlay.ts` — overlay rendering (D1-D4)
- `pages-builder/src/shell/builder-shell.ts:606-806` — current add/insert handlers
- `pages-builder/src/catalog/palette-context.ts` — PaletteContext, contextRelevance
- `pages-builder/src/palette/inline-picker.ts` — inline picker component
- `pages-builder/src/palette/palette-filter.ts` — catalog filtering
- `pages-builder/src/tree/builder-tree.ts` — tree rendering and events
- `pages-schema/src/document-schema.ts` — Zod structural hierarchy
- Protocol PP-20260916-b6f3e8 — all mutations through `_applyEdit`
- Protocol PP-20260817-a11y01 — ARIA interaction contract
- decisions.md D5-D12 — all design choices
