# Visual YAML Builder Phase 1b — Design Specification

**Date:** 2026-09-14
**Status:** Draft
**Scope:** Structural editing (#433), preview data provider (#432), dock-workbench Lit extraction (#429)
**Parent:** Phase 1 spec at `docs/specs/issue-visual-yaml-builder/2026-09-12-visual-yaml-builder-design.md`
**Epic:** #434

## Problem

Phase 1 (#428) built the foundation: a CST-backed facade (`PageDocument`), an outline tree with selection, a 56-component categorised palette, bidirectional YAML sync, and append-only component insertion. The builder is a viewer with an append button — it cannot insert at a position, move components, wrap in containers, or show meaningful previews. Phase 1b fills these gaps across three workstreams.

**#433 — Structural Editing (P1):** The core interaction model. Without position-aware insertion, move, wrap, and replace, the builder is not a usable authoring tool.

**#432 — Preview Data Provider (P2):** Components render as empty boxes without data. Users cannot judge layout or verify component configuration without seeing realistic sample content.

**#429 — Dock-Workbench Lit Extraction (P3):** The builder shell has a bespoke hardcoded dock layout (~200 lines of resize handlers, toggle bar, CSS). Extract this into a reusable `<pages-dock-workbench>` Lit component. The runtime's dock-workbench pipeline (6-zone model with per-panel drag and constraints) is a different, more complex system and is NOT being unified — it continues unchanged.

---

## 1. Structural Editing (#433)

### 1.1 Facade Extensions — Transaction API

Compound operations (wrap, replace, cross-container move) decompose into multiple CST mutations that must be a single undo step. Without a transaction model, each sub-mutation pushes its own undo snapshot — requiring N undo operations to reverse one user action.

```typescript
// On PageDocument
beginTransaction(): void
commitTransaction(): void
abortTransaction(): void
```

`beginTransaction()` saves the current `toString()` as the undo snapshot and sets an internal `_inTransaction` flag. During a transaction, both `_pushUndoInternal()` and `_notifyInternal()` are suppressed — no intermediate undo snapshots and no intermediate change notifications. `commitTransaction()` clears the flag and fires a single coalesced `_notifyInternal()`. The pre-transaction snapshot is already on the undo stack — undo restores to the pre-compound state in one step.

`abortTransaction()` clears the flag and restores the document to the pre-transaction state by re-parsing the saved snapshot. The undo entry is popped (the operation never happened). This handles mid-compound errors — a failed wrap doesn't leave a partially mutated document.

Nested transactions are not needed — compound operations do not compose into deeper compounds. Every compound operation wraps in try/catch:

```typescript
wrapIn(containerType: string): ComponentNode {
  this.doc.beginTransaction();
  try {
    // remove self, create container, insert self, place container
    this.doc.commitTransaction();
    return newContainer;
  } catch (e) {
    this.doc.abortTransaction();
    throw e;
  }
}
```

### 1.2 Facade Extensions — New Operations

All new operations on existing node interfaces. Compound operations use the transaction API.

**ComponentNode — new methods:**

```typescript
addChild(slot: string, type: string, props?: Record<string, unknown>): ComponentNode
```
Insert a child into a container slot. Uses the descriptor registry to determine the YAML mutation: for named-record slots (tabs), creates a named entry; for array slots (sidebar), appends to the array; for nested-array slots (split), appends to the child array. Throws if the component is not a container or the slot name is invalid.

```typescript
wrapIn(containerType: string): ComponentNode
```
Compound (transaction-wrapped): remove self from parent, create a new container of the given type, insert self as first child in the container's first slot, place the container at self's original position. Returns the new container node.

For container types (tabs, accordion, etc.): uses the descriptor registry to determine the correct slot structure. Each `ContainerChildDescriptor` gains a `defaultContentSlot` field identifying the semantically correct slot for content insertion (e.g., `content` for sidebar, not `sidebar`; the first slot for tabs). The wrapped component becomes the first entry in the `defaultContentSlot` with an auto-generated slot name ("Tab 1", "Section 1").

```typescript
replaceWith(type: string): ComponentNode
```
Compound (transaction-wrapped): create a new component of the target type at the same position, migrate properties, remove the original. Returns the new component node.

Property migration is top-level key matching with type compatibility: for each top-level property key present in both the source component and the target schema, transfer the value only if the schema types are compatible (both primitives, both objects, or both arrays). Incompatible types (e.g., source has an object `lookup` but target expects a string) are dropped. Source-only properties are dropped. Target-only properties get schema defaults from `componentSchemaRegistry`; if no default exists, the key is omitted.

```typescript
moveToSlot(target: { path: readonly (string | number)[]; slotName: string }, index: number): void
```
Move into a named-record container slot. Creates the named entry if it doesn't exist (with the provided `slotName` as key). For existing slots, inserts at the given index within the slot's child array. Compound (transaction-wrapped) when the source and target are different containers.

**PageNode — new methods:**

```typescript
insertChildAt(index: number, type: string, props?: Record<string, unknown>): ComponentNode
```
Insert a component at a specific index in the page's component/row/column array (depending on layout mode).

```typescript
wrapInRow(componentIndices: number[]): RowNode
```
Page-level restructuring (transaction-wrapped): remove the specified components from their current positions, create a new row with a single full-width column, insert the components into that column, place the row at the position of the first removed component. Behaviour by layout mode: in `flat` mode, transitions to `rows` mode; in `rows` mode, inserts the new row among existing rows; in `columns` mode, throws (wrapping in a row is semantically invalid when the page uses column-based layout — use the context menu to restructure first).

**RowNode — new methods:**

```typescript
insertColumnAt(index: number, span?: number): ColumnNode
```

**ColumnNode — new methods:**

```typescript
insertComponentAt(index: number, type: string, props?: Record<string, unknown>): ComponentNode
```

### 1.3 Tree UI — Add Picker

Container nodes (pages, rows, columns, container components) show a `+` icon button on hover. Clicking opens an inline popover anchored to the button.

The popover contains a compact palette: category tabs, search input, filtered component grid. Context is derived from the clicked node (not the current selection):

The context extends the existing `PaletteContext` interface with an optional `parentSlot` field:

```typescript
interface PaletteContext {
  parentType: string | undefined
  parentSlot?: string | undefined   // NEW — slot name for container-targeted insertion
  acceptsComponents: boolean
  availableDatasets: string[]
  siblingTypes: string[]
}
```

Selecting an entry calls the appropriate insert method — `addChild()` for container slots, `addComponent()` / `insertComponentAt()` for columns/pages.

**Shared filtering logic:** A `PaletteFilter` utility is extracted from `builder-palette.ts`. Both the dock palette and the inline popover consume it. The filter takes a `PaletteContext` and the full catalog, returns entries with relevance scores (promoted / normal / hidden / needs-prereq). The dock palette leaves `parentSlot` undefined (full catalog view); the inline popover sets it from the clicked container node.

### 1.4 Tree UI — Context Menu

Right-click on any tree node opens a `<pages-context-menu>` Lit component (new, in `pages-primitives`). ARIA role: `menu` with `menuitem` children. `FocusTrapMixin` for keyboard containment. Dismiss on `Escape`, click outside, or item selection.

Menu items are context-dependent:

| Node Type | Actions |
|-----------|---------|
| Component | Move up, Move down, Move to…, Wrap in…, Replace with…, Duplicate, Delete |
| Container component | Same + Add child |
| Row | Move up, Move down, Add column, Delete |
| Column | Move left, Move right, Add component, Delete |
| Page | Add row, Add component, Delete |
| Dataset | Duplicate, Delete |

**Submenus:**
- "Wrap in…" → Layout section (Row, Column) + Container section (Tabs). Row/Column invoke `PageNode.wrapInRow()`. Tabs invokes `ComponentNode.wrapIn('tabs')`.
- "Replace with…" → opens the palette popover filtered to the same category as the current component type.
- "Move to…" → opens a tree-position picker showing valid drop targets. Selecting a target calls `moveToIndex()` or `moveToSlot()`.

**Delete confirmation:** Deleting a container with children shows a confirmation dialog via `<pages-modal>`. Deleting a leaf component does not confirm.

### 1.5 Tree UI — Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Delete` / `Backspace` | Delete selected node (with confirmation for containers with children) |
| `Ctrl+D` | Duplicate selected node |
| `Ctrl+Shift+P` | Open palette popover at selected node |
| `Ctrl+Shift+Up` | Move selected node up among siblings |
| `Ctrl+Shift+Down` | Move selected node down among siblings |
| `Ctrl+Z` | Undo (already wired) |
| `Ctrl+Shift+Z` | Redo (already wired) |

`Ctrl+Shift+Up/Down` provides keyboard-accessible sibling reorder as an alternative to drag-and-drop. `Ctrl+Shift+P` avoids the browser `Ctrl+P` (Print) conflict.

Wired via `KeyboardShortcutMixin` on `builder-tree`.

### 1.6 Tree UI — Drag-and-Drop

Drag handle icon on each tree node, visible on hover. Drag operations use the HTML Drag and Drop API with custom drop indicators.

**Visual feedback:**
- Horizontal line between siblings: reorder position indicator
- Container highlight: cross-container move target
- Named-record slot highlight: specific tab/section as drop target
- Ghost image: semi-transparent copy of the dragged node label

**Drop logic:**

| Drop Target | Operation |
|-------------|-----------|
| Between siblings in same parent | `moveToIndex()` — reorder |
| On a column or flat-component array | `moveToIndex()` — cross-container move |
| On a named-record slot (e.g., "Tab 1") | `moveToSlot()` — move into existing slot |
| On a named-record container (e.g., tabs node itself) | Auto-generate slot name ("Tab N"), `moveToSlot()` — user renames via tree inline-edit or property panel |
| Invalid target (row, dataset) | No drop indicator, drop rejected |

**Edge behaviours:**
- Auto-scroll when dragging near tree top/bottom edges (16px threshold, 4px/frame scroll)
- ARIA: drag start/end announced via `LiveRegionMixin` ("Moving bar-chart", "Dropped into Column 2, position 3")
- Cross-page drag: supported — `moveToIndex()` accepts any valid path

### 1.7 Structural Editing Surface Scope

All structural operations are invoked from the tree panel only in Phase 1b. The facade API is surface-agnostic — the same `addChild()`, `wrapIn()`, `moveToIndex()` methods could be called from a visual preview direct-manipulation layer or from LSP refactoring commands in a future phase. The tree is the sole UI surface for now; the API does not couple to it.

**Partial #433 scope note:** Issue #433 also specifies visual-view insertion points ("+ indicators between rendered components on hover") and `Ctrl+X` move mode (keyboard-accessible click-to-move alternative to DnD). These are deferred — they require the visual preview to become an editing surface, which is beyond Phase 1b's tree-only scope. This is a partial close of #433; the remaining visual-view interactions will be tracked separately.

---

## 2. Preview Data Provider (#432)

### 2.1 Strategy Registry

The preview system uses a three-tier cascade to ensure no component renders as an empty box:

1. **Real data** — if the dataset has an inline `content` source, use it directly
2. **Strategy-generated data** — for the 21 types in `NEEDS_DATASET_TYPES`, a registered strategy generates synthetic data from the component's props and dataset column definitions
3. **Preview hints** — for types that render empty without data or children (containers with no children, components with no configuration), `previewHints` on `ComponentCatalogEntry` provides placeholder content

**Strategy registry:** A `PreviewDataRegistry` maps component types to `PreviewDataStrategy` implementations for the 21 data-consuming types.

```typescript
interface PreviewDataStrategy {
  generate(
    props: Record<string, unknown>,
    datasets: DatasetNode[],
  ): DatasetSnapshot[]
}

interface DatasetSnapshot {
  uuid: string
  rows: Record<string, unknown>[]
}
```

The registry is a `Map<string, PreviewDataStrategy>`. Lookup: `registry.get(componentType)`. If absent, the component is not data-consuming — tiers 1 and 2 don't apply.

**Preview hints:** `ComponentCatalogEntry` gains an optional `previewHints` field:

```typescript
interface PreviewHints {
  placeholderText?: string       // text to show when component has no content
  sampleChildren?: number        // number of stub children to generate for containers
  displayDefaults?: Record<string, unknown>  // props to apply in preview when not user-configured
}
```

Container types (tabs, accordion, sidebar, etc.) use `sampleChildren` to generate stub child components so the container renders with visible structure. Content types (html, markdown) use `placeholderText` to show representative content.

Strategies import from `@casehubio/pages-schema` (for `componentSchemaRegistry` — column type metadata) and `@casehubio/pages-data` (for column type definitions). `pages-schema` is already in `pages-builder`'s dependency chain via `pages-document`.

### 2.2 Strategy Implementations

Four strategy classes cover all 21 data-consuming types (from the `NEEDS_DATASET_TYPES` set in `component-catalog.ts`):

**ChartDataStrategy** — `bar-chart`, `line-chart`, `area-chart`, `pie-chart`, `scatter-chart`, `bubble-chart`, `timeseries`, `heatmap-chart`, `treemap-chart`, `density-heatmap`

Reads `lookup.uuid` from props → finds the referenced `DatasetNode` → reads column definitions → generates 5-8 rows with typed values. String columns get region/product/category names. Number columns get plausible values in the 1,000-100,000 range. Date columns get dates spanning the last 12 months. `bubble-chart` adds a third numeric column for bubble size. `heatmap-chart` and `density-heatmap` generate a grid matrix. `timeseries` generates time-series data with 20 points.

**TableDataStrategy** — `data-table`, `grid-table`, `grouped-view`

Same dataset lookup. Generates 10-15 rows with varied types per column. Includes edge cases: one null value, one long string, one negative number — so the preview reveals how the table handles real data. `grouped-view` generates rows with a groupable string column.

**MetricDataStrategy** — `metric`, `meter`

Generates a single numeric value (e.g., 42,350). `meter` additionally generates min/max bounds. The value is derived from the metric's `lookup` column type if available.

**DataComponentStrategy** — `selector`, `map`, `timeline`, `graph`, `event-timeline`

Generates component-appropriate data from the referenced dataset. `selector` produces 4-6 distinct categorical values for filter options. `map` generates lat/lng coordinate pairs. `timeline` and `event-timeline` generate timestamped event entries spanning the last 7 days. `graph` generates node/edge pairs for network rendering.

### 2.3 Column-Aware Generation

When a dataset has column definitions in the YAML:

```yaml
datasets:
  - uuid: sales_tx
    columns:
      - { id: region, type: TEXT }
      - { id: revenue, type: NUMBER }
      - { id: date, type: DATE }
```

The strategy reads those definitions and generates appropriately typed values. When no column definitions exist (dataset has only a URL), the strategy falls back to generic sample data: 3 string columns and 2 number columns with plausible names.

### 2.4 Integration with Existing Preview Infrastructure

The builder shell has a `renderPreview` callback and `_refreshPreview()`. The builder does NOT call `loadSite()` directly — the host application provides the `renderPreview` callback, which typically calls `loadSite()` internally.

**Integration via YAML transformation (no runtime changes required):**

1. On document change, the builder runs strategies for all data-consuming components in the current document
2. For each dataset referenced by a component with a registered strategy, the builder generates a `DatasetSnapshot`
3. Before passing YAML to `renderPreview`, the builder transforms it: for each dataset with a URL source that has a generated snapshot, the dataset's `url` source is replaced with an inline `content` source containing the strategy-generated rows
4. The runtime's existing `InlineProvider` (in `provider-factory.ts`) handles `content`-based datasets natively — no new runtime extension points needed
5. The existing preview-click synchronization (via `data-component-type` attributes) continues working — components render normally with synthetic data

**Container preview hints:** Before calling `renderPreview`, the builder also applies `previewHints`: containers with no children get stub children injected into the transformed YAML; components with `displayDefaults` get those defaults merged under their properties.

**Re-generation triggers:**
- Component props change (different dataset, different columns) → strategy regenerates for that component
- Dataset column definitions change → all strategies referencing that dataset regenerate
- New component added → strategy generates if the type has a registered strategy
- All behind the existing 300ms debounce on `_refreshPreview()`

---

## 3. Dock-Workbench Lit Extraction (#429)

### 3.1 Extraction Approach

The runtime's dock-workbench is a 6-zone model (`left-top`, `left-bottom`, `right-top`, `right-bottom`, `bottom-left`, `bottom-right`) with per-panel drag rearrangement, zone constraints, and a `Component` tree renderer. The builder shell's dock is a simpler 4-zone model (left, centre, right, bottom) with resize handles and toggle bar. These serve different purposes and are NOT being unified.

**Layer 1: `<pages-dock-workbench>` Lit component** — in `pages-ui` (existing package for interactive UI components). Extracts the builder shell's dock layout logic: 4-zone sizing, resize handles, panel collapse/expand, toggle bar, localStorage persistence. This is a general-purpose dock layout component, not a wrapper around the runtime's ZoneLayoutEngine.

**Layer 2: Builder shell migration** — `builder-shell.ts` replaces its bespoke dock (~200 lines of layout CSS, resize event handlers, toggle bar) with `<pages-dock-workbench>`.

The runtime dock-workbench pipeline (`findDockConfig()` → `createZoneLayoutEngine()` → `buildTree()`) continues unchanged — it is a separate, more complex system.

### 3.2 Component API

```typescript
@customElement('pages-dock-workbench')
class PagesDockWorkbench extends LitElement {
  // Zone sizing
  @property({ type: Number }) leftWidth = 260
  @property({ type: Number }) rightWidth = 320
  @property({ type: Number }) bottomHeight = 200
  @property({ type: Number }) minPanelSize = 120
  @property({ type: Number }) maxPanelSize = 600

  // Zone visibility — collapse/expand
  @property({ type: Boolean }) leftCollapsed = false
  @property({ type: Boolean }) rightCollapsed = false
  @property({ type: Boolean }) bottomCollapsed = true

  // Zone enable/disable — disabled zones are not rendered at all
  @property({ type: Boolean }) leftEnabled = true
  @property({ type: Boolean }) rightEnabled = true
  @property({ type: Boolean }) bottomEnabled = true

  // Chrome
  @property({ type: Boolean }) showToggleBar = true
  @property() persistKey?: string
}
```

**Zone enable/disable:** When a zone is disabled (`leftEnabled=false`), its slot, resize handle, and toggle bar icon are not rendered — the centre panel expands to fill the space. This lets consumers configure the workbench per content type: a YAML-only editor disables left and right zones; a diagram editor might disable the bottom panel. Disabled zones differ from collapsed zones — collapsed zones can be re-expanded by the user; disabled zones are removed from the layout entirely.

**Named slots:** `left`, `centre`, `right`, `bottom`, `status-bar`, `toggle-bar`

**Events:**
- `dock-panel-toggle` — `{ zone: string, collapsed: boolean }`
- `dock-panel-resize` — `{ zone: string, size: number }`

**Resize handles:** CSS `resize` or pointer-event-based drag on zone borders. Handles constrain to `minPanelSize` / `maxPanelSize`. Debounced persistence to localStorage under `persistKey`.

**Toggle bar:** Vertical icon strip (default slot: `toggle-bar`). Each icon toggles a zone's collapsed state. When `showToggleBar` is false, consumers manage collapse via properties directly.

**ARIA:** `role="region"` on each zone with `aria-label` (e.g., "Left panel"). Resize handles have `role="separator"` with `aria-orientation` and `aria-valuenow`.

### 3.3 Builder Shell After Migration

```html
<pages-dock-workbench
  left-width="260" right-width="320" bottom-height="200"
  persist-key="pages-builder"
  .bottomCollapsed=${!this._yamlExpanded}>

  <pages-builder-tree slot="left"
    .document=${this._doc}
    @node-select=${this._handleNodeSelect}>
  </pages-builder-tree>

  <div slot="centre">
    ${this._renderEditorArea()}
  </div>

  <div slot="right">
    ${this._renderPropertiesPanel()}
  </div>

  <pages-code-editor slot="bottom"
    .value=${this._yaml}
    @input=${this._handleYamlInput}>
  </pages-code-editor>
</pages-dock-workbench>
```

The shell's `render()` method simplifies from ~150 lines of dock layout to declarative slot assignment. Resize, collapse, and toggle logic moves into the component.

**View mode migration:** The current shell has three view modes (source, split, visual) implemented as conditional rendering of the YAML editor and preview. With the dock's collapsible bottom panel, these become:

| Old Mode | New Equivalent |
|----------|---------------|
| visual | Bottom collapsed (YAML hidden), centre shows preview |
| split | Bottom expanded, centre shows preview above YAML |
| source | Left and right collapsed, bottom expanded full-height — YAML-dominant layout |

The three-mode toolbar toggle becomes a two-state YAML toggle (show/hide bottom panel). "Source mode" is achieved by collapsing left + right panels and expanding the bottom panel — no dedicated mode needed.

---

## 4. Package Structure

No new packages. All changes in existing packages:

```
packages/
  pages-document/src/               # Facade extensions
    page-document.ts                 # +beginTransaction, +commitTransaction
                                     # +addChild, +wrapIn, +replaceWith, +moveToSlot
                                     # +insertChildAt, +insertComponentAt, +insertColumnAt
                                     # +wrapInRow

  pages-builder/src/                 # UI interactions + preview data
    tree/
      builder-tree.ts                # +DnD handlers, +context menu trigger, +add button
      tree-dnd.ts                    # NEW — DnD logic (drag/drop handlers, hit testing)
      tree-context-menu.ts           # NEW — context menu item computation
    palette/
      builder-palette.ts             # Refactor: extract PaletteFilter
      palette-filter.ts              # NEW — shared filtering logic
      inline-picker.ts              # NEW — popover picker component
    data/
      preview-data-registry.ts       # NEW — strategy registry
      preview-yaml-transform.ts      # NEW — YAML transformation for preview
      strategies/
        chart-strategy.ts            # NEW — 10 chart types
        table-strategy.ts            # NEW — data-table, grid-table, grouped-view
        metric-strategy.ts           # NEW — metric, meter
        data-component-strategy.ts   # NEW — selector, map, timeline, graph, event-timeline
    shell/
      builder-shell.ts               # Migration: replace bespoke dock with <pages-dock-workbench>

  pages-primitives/src/              # Shared a11y components
    context-menu/
      context-menu.ts                # NEW — <pages-context-menu>

  pages-ui/src/                      # Interactive UI components
    dock/
      dock-workbench.ts              # NEW — <pages-dock-workbench>
```

---

## 5. Testing Strategy

**Facade tests** (`pages-document`):
- Transaction API: begin/commit, abort on error restores pre-transaction state, notifications suppressed during transaction, single coalesced notification on commit
- `addChild()`: all 3 slot kinds (named-record, array, nested-array), invalid slot rejection
- `wrapIn()`: flat component → tabs, row component → tabs, undo restores original position
- `replaceWith()`: same-category (bar→line preserves lookup), cross-category (bar→metric drops lookup), undo restores original type and all properties
- `moveToSlot()`: into existing named slot, into new named slot, cross-container
- `insertChildAt()` / `insertComponentAt()` / `insertColumnAt()`: at start, middle, end, out-of-bounds
- `wrapInRow()`: flat→rows migration, multi-component selection, single component

**Tree interaction tests** (`pages-builder`):
- Add picker: popover opens on `+` click, context filtering matches parent type, selection inserts at correct position
- Context menu: correct items per node type, wrap submenu shows Layout and Container sections, delete confirms for containers with children
- DnD: reorder within column, cross-column move, move into named-record slot (tab), auto-scroll, invalid drop rejection
- Keyboard shortcuts: Delete removes, Ctrl+D duplicates, Ctrl+Shift+P opens picker, Ctrl+Shift+Up/Down reorders siblings

**Preview data tests** (`pages-builder`):
- Strategy registry: registered types get data, unregistered types render as-is
- ChartDataStrategy: generates correct column types from dataset definitions, falls back to generic
- Column-aware generation: TEXT→strings, NUMBER→numbers, DATE→dates
- Integration: preview re-renders on component prop change, debounce works

**Dock workbench tests** (`pages-ui`):
- Resize: drag updates zone width/height, constrained to min/max
- Collapse/expand: toggle updates layout, content hidden when collapsed
- Persistence: sizes saved to localStorage under persistKey, restored on mount
- Slots: left/centre/right/bottom/status-bar content renders in correct zones

**Integration tests** (Playwright):
- Build a page: add components via `+` picker, arrange with DnD, verify YAML output
- Preview: add a chart referencing a dataset, verify sample data renders in preview
- Structural edits: wrap component in tabs via context menu, undo, verify original state restored

---

## 6. Accessibility

| Component | ARIA Pattern | Primitives |
|-----------|-------------|------------|
| Context menu | `menu` / `menuitem` with submenus | `FocusTrapMixin`, `KeyboardShortcutMixin` |
| Inline picker popover | `listbox` / `option` | `RovingTabindexMixin`, `FocusTrapMixin` |
| Tree DnD | Live announcements on drag start/end/drop | `LiveRegionMixin` |
| Dock workbench | `region` per zone, `separator` for resize handles | — |
| Delete confirmation | `alertdialog` | `<pages-modal>` (existing) |

---

## References

- `docs/specs/issue-visual-yaml-builder/2026-09-12-visual-yaml-builder-design.md` — Phase 1 design spec (parent)
- `docs/specs/issue-visual-yaml-builder/decisions.md` — Phase 1 design decisions
- `packages/pages-document/src/page-document.ts` — existing facade with CRUD + undo/redo
- `packages/pages-document/src/container-descriptors.ts` — container slot structure registry (10 types, 3 slot kinds)
- `packages/pages-builder/src/tree/builder-tree.ts` — existing tree with selection and expand/collapse
- `packages/pages-builder/src/palette/builder-palette.ts` — existing palette with 56 components, context filtering
- `packages/pages-builder/src/shell/builder-shell.ts` — existing shell with bespoke dock layout
- `packages/pages-builder/src/catalog/component-catalog.ts` — 56 component entries across 7 categories
- `packages/pages-builder/src/catalog/palette-context.ts` — existing PaletteContext interface
- `packages/pages-runtime/src/site.ts` — runtime dock-workbench detection and ZoneLayoutEngine (NOT being unified — see §3.1)
- `packages/pages-runtime/src/provider-factory.ts` — `InlineProvider` for content-based datasets (used by preview YAML transformation)
- `packages/pages-primitives/` — a11y mixins (RovingTabindex, FocusTrap, LiveRegion, KeyboardShortcut)
- `packages/pages-ui/` — interactive UI components (dock-workbench target package)
- `docs/protocols/casehub/web-component-strategy.md` — Lit conventions, `pages-` prefix
- `docs/protocols/casehub/aria-interaction-contract.md` — ARIA role + accessible name requirements
- `docs/protocols/casehub/css-design-tokens.md` — `--pages-` token vocabulary
- `docs/specs/issue-285-dock-workbench/` — original dock-workbench design spec (runtime model, not builder model)
