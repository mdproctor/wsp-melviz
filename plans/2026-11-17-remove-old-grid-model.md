# Remove Old Grid Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #674 — Remove old rows/columns grid model — migrate to unified items model
**Issue group:** #674

**Goal:** Remove every remnant of the old `rows:` / `columns:` page-level grid layout and the `type: "columns"` / `type: "rows"` layout components, leaving only the unified `items:` / `grid:` placement model.

**Architecture:** Bottom-up removal across 6 packages: migrate YAML files first, then delete type definitions (pages-document, pages-schema, pages-component model), then delete consumers (pages-ui parser, pages-component renderer, pages-builder). This is deletion work — no new features, no new tests. Existing tests for the new grid model validate nothing was broken.

**Tech Stack:** TypeScript monorepo (Yarn workspaces), Vitest for testing, Zod for schema validation

## Global Constraints

- Pre-release repo — no backward compatibility needed
- `type: "grid"` component with `GridItem[]`, `GridPlacement`, `GridProps` all stay untouched
- `LayoutMode` retains `'grid' | 'flat' | 'items'` — only `'rows' | 'columns'` removed
- Layout components `type: "columns"` and `type: "rows"` are removed alongside the page-level model
- IntelliJ MCP required for all code operations — no bash grep/sed on source files
- After each batch: `yarn build` must succeed and `yarn test` must pass

---

## Batch 1: YAML Migration

### Task 1: Migrate all old-format YAML files to grid format

**Files:**
- Modify: `backend/data-json/src/test/resources/org/apps/tasks/tasks.page.yaml`
- Modify: `backend/data-json/src/test/resources/org/apps/contacts/contacts.page.yaml`
- Modify: `backend/data-json/src/test/resources/org/apps/platform/platform.page.yaml`
- Modify: `examples/samples/Layout/includes/team-detail.dash.yml`
- Modify: `examples/samples/Layout/includes/reports.dash.yml`
- Modify: `examples/samples/Layout/includes/admin-settings.dash.yml`
- Modify: `examples/samples/Custom Components/Code Editor.page.yaml`

**Interfaces:**
- Consumes: `packages/pages-component/src/renderer/grid-migration.ts` — `migrateRowsToItems()` function
- Produces: All YAML files using `grid:` format with `at: { x, y, w, h }` placement

- [ ] **Step 1: Write a migration runner script**

Write to `/tmp/migrate-yaml.ts`:
```typescript
import { readFileSync, writeFileSync } from 'fs';
import { parse, stringify } from 'yaml';
import { migrateRowsToItems, needsMigration } from './packages/pages-component/src/renderer/grid-migration.js';

const files = [
  'backend/data-json/src/test/resources/org/apps/tasks/tasks.page.yaml',
  'backend/data-json/src/test/resources/org/apps/contacts/contacts.page.yaml',
  'backend/data-json/src/test/resources/org/apps/platform/platform.page.yaml',
  'examples/samples/Layout/includes/team-detail.dash.yml',
  'examples/samples/Layout/includes/reports.dash.yml',
  'examples/samples/Layout/includes/admin-settings.dash.yml',
];

for (const file of files) {
  const content = readFileSync(file, 'utf-8');
  const doc = parse(content);
  if (doc.pages) {
    for (const page of doc.pages) {
      if (needsMigration(page)) {
        migrateRowsToItems(page);
      }
    }
  }
  writeFileSync(file, stringify(doc, { lineWidth: 120 }));
  console.log(`Migrated: ${file}`);
}
```

If the migration script can't be imported directly (it's TS, not built), manually migrate each file. The conversion is mechanical: each `rows:` entry with `columns:` containing `span:` values maps to `grid:` entries with `at: { x: <cumulative-span>, y: <row-index>, w: <span>, h: 1 }` using a 12-column grid.

- [ ] **Step 2: Migrate tasks.page.yaml**

Read the file. For each page, convert `rows:` / `columns:` / `span:` to `grid:` with `at:` placement. Example conversion:
```yaml
# Old:
rows:
  - columns:
    - span: 6
      components:
        - type: html
          properties: { content: "left" }
    - span: 6
      components:
        - type: html
          properties: { content: "right" }

# New:
grid:
  - at: { x: 0, y: 0, w: 6, h: 1 }
    type: html
    properties: { content: "left" }
  - at: { x: 6, y: 0, w: 6, h: 1 }
    type: html
    properties: { content: "right" }
```

- [ ] **Step 3: Migrate contacts.page.yaml**

Same conversion pattern as Step 2.

- [ ] **Step 4: Migrate platform.page.yaml**

Same conversion pattern as Step 2.

- [ ] **Step 5: Migrate the three .dash.yml includes**

Read and convert `team-detail.dash.yml`, `reports.dash.yml`, `admin-settings.dash.yml`. Same rows→grid conversion.

- [ ] **Step 6: Fix Code Editor.page.yaml**

This file has old-format YAML inside an HTML `<pre>` sample block. Update the sample to show the new grid format, or remove the old-format example.

- [ ] **Step 7: Verify all migrated files parse correctly**

Run: `yarn build` (should succeed — parser already handles grid format)
Run: `yarn test` (all existing tests should still pass)

- [ ] **Step 8: Commit**

```bash
git add backend/data-json/src/test/resources/ examples/samples/
git commit -m "feat(#674): migrate all YAML files from rows/columns to grid format Refs #674"
```

---

## Batch 2: Remove Type Definitions and Schemas

### Task 2: Remove RowNode, ColumnNode, and old LayoutMode from pages-document

**Files:**
- Modify: `packages/pages-document/src/types.ts:17`
- Modify: `packages/pages-document/src/page-document.ts:303-568`
- Modify: `packages/pages-document/src/index.ts:1`
- Modify: `packages/pages-document/src/insertion-constraints.ts:9,41,47`

**Interfaces:**
- Consumes: nothing
- Produces: `LayoutMode` without `'rows' | 'columns'`; `PageNode` without `getRows()`, `getColumns()`, `addRow()`, `insertRowAt()`, `wrapInRow()`; no `RowNode`/`ColumnNode` exports

- [ ] **Step 1: Remove 'rows' | 'columns' from LayoutMode**

In `packages/pages-document/src/types.ts` line 17, change:
```typescript
export type LayoutMode = 'grid' | 'flat' | 'items' | 'rows' | 'columns';
```
to:
```typescript
export type LayoutMode = 'grid' | 'flat' | 'items';
```

- [ ] **Step 2: Remove rows/columns checks from getLayoutMode()**

In `packages/pages-document/src/page-document.ts`, remove the two lines in `getLayoutMode()` that check for `rows` and `columns` sequences (lines 308-309).

- [ ] **Step 3: Remove getRows() and getColumns() methods**

Delete `getRows(): RowNode[]` (lines 313-317) and `getColumns(): ColumnNode[]` (lines 319-323) from `PageNode`.

- [ ] **Step 4: Remove addRow(), insertRowAt(), wrapInRow()**

Delete these methods from `PageNode`. They create rows/columns YAML structures.

- [ ] **Step 5: Remove row/column branches from removeChild()**

In `removeChild()`, the dispatch on layout mode should only handle `'grid'`, `'items'`, and `'components'` (flat). Remove any rows/columns path.

- [ ] **Step 6: Delete RowNode class entirely**

Delete the `RowNode` class (~lines 477-527). It has `getColumns()`, `addColumn()`, `insertColumnAt()`, `removeColumn()`.

- [ ] **Step 7: Delete ColumnNode class entirely**

Delete the `ColumnNode` class (~lines 529-568). It has `span` getter/setter, `getComponents()`, `addComponent()`, `insertComponentAt()`, `removeComponent()`.

- [ ] **Step 8: Remove RowNode, ColumnNode from index.ts exports**

In `packages/pages-document/src/index.ts` line 1, change:
```typescript
export { PageDocument, PageNode, RowNode, ColumnNode, DatasetNode, ComponentNode, NavTreeNode, GridItemNode } from './page-document.js';
```
to:
```typescript
export { PageDocument, PageNode, DatasetNode, ComponentNode, NavTreeNode, GridItemNode } from './page-document.js';
```

- [ ] **Step 9: Clean up insertion-constraints.ts**

In `packages/pages-document/src/insertion-constraints.ts`:
- Line 9: remove `'columns'` from `ALL_COMPONENT_TYPES`
- Line 41: remove `if (parentNodeType === 'column')` branch from `allowedTypesAt()`
- Line 47: in `allowedTypesForSiblingOf()`, change `allowedTypesAt('column')` to `allowedTypesAt('page')` (or use `COMPONENT_CONTAINER_CONSTRAINT` directly)

- [ ] **Step 10: Commit (will not build yet — downstream packages still import removed exports)**

```bash
git add packages/pages-document/
git commit -m "wip(#674): remove RowNode, ColumnNode, old LayoutMode from pages-document Refs #674"
```

### Task 3: Remove old schemas, parser branches, component types, and layout entries

**Files:**
- Modify: `packages/pages-schema/src/document-schema.ts:14,37-38,107-122`
- Modify: `packages/pages-schema/src/component-schemas.generated.ts:42-46`
- Modify: `packages/pages-ui/src/parser/page-parser.ts:3,86,259-270`
- Modify: `packages/pages-ui/src/parser/component-desugar.ts:432-460`
- Modify: `packages/pages-component/src/renderer/layout.ts:2,5,31-45`
- Modify: `packages/pages-component/src/model/component-props.ts:9-13`
- Modify: `packages/pages-component/src/model/type-guards.ts:4-5,69-70,165-171`
- Delete: `packages/pages-component/src/renderer/grid-migration.ts`
- Delete: `packages/pages-component/src/renderer/grid-migration.test.ts`

**Interfaces:**
- Consumes: Task 2's removal of `RowNode`/`ColumnNode` from pages-document
- Produces: No `columnSchema`/`rowSchema` in pages-schema; no `isColumns()`/`isRows()` guards; no `ColumnsProps`/`RowsProps`; no columns desugaring in parser; `LAYOUT_TYPES` contains only `"grid"` and `"stack"`

- [ ] **Step 1: Remove old schemas from pages-schema**

In `packages/pages-schema/src/document-schema.ts`:
- Line 14: remove `columnsPropsSchema, rowsPropsSchema` from imports
- Lines 37-38: delete the two `z.literal("columns")` and `z.literal("rows")` entries from the component discriminated union
- Lines 107-116: delete `columnSchema` and `rowSchema` definitions
- Lines 121-122: remove `rows` and `columns` from `pageEntrySchema`

In `packages/pages-schema/src/component-schemas.generated.ts`:
- Lines 42-46: delete `columnsPropsSchema` and `rowsPropsSchema`

- [ ] **Step 2: Remove stale JSDoc from page-parser.ts**

In `packages/pages-ui/src/parser/page-parser.ts`:
- Line 3: update comment to remove "rows+columns" reference
- Line 86: update comment to remove "rows" reference
- Lines 259-270: delete the entire `**2. rows explicit layout**` JSDoc block

- [ ] **Step 3: Remove columns desugaring from component-desugar.ts**

In `packages/pages-ui/src/parser/component-desugar.ts`:
- Lines 432-460: delete the entire `type: "columns"` desugaring branch that converts columns layout components to grid items

- [ ] **Step 4: Remove layout component entries from pages-component**

In `packages/pages-component/src/renderer/layout.ts`:
- Line 2: remove `isColumns` from import
- Line 5: remove `"columns"` and `"rows"` from `LAYOUT_TYPES` set
- Lines 31-45: delete `case "columns":` and `case "rows":` from `applyLayoutCSS`

- [ ] **Step 5: Remove ColumnsProps, RowsProps from component-props.ts**

In `packages/pages-component/src/model/component-props.ts`:
- Delete `ColumnsProps` interface (lines 9-11) and `RowsProps` type (line 13)

- [ ] **Step 6: Remove isColumns(), isRows() from type-guards.ts**

In `packages/pages-component/src/model/type-guards.ts`:
- Lines 4-5: remove `ColumnsProps`, `RowsProps` from imports
- Lines 69-70: remove `columns: ColumnsProps` and `rows: RowsProps` from `ComponentTypeRegistry`
- Lines 165-171: delete `isColumns()` and `isRows()` functions

- [ ] **Step 7: Delete grid-migration.ts and its test**

Delete `packages/pages-component/src/renderer/grid-migration.ts` (the migration script — no longer needed after Batch 1).
Delete `packages/pages-component/src/renderer/grid-migration.test.ts`.

Also check for and remove any barrel exports of `grid-migration` from `packages/pages-component/src/index.ts` or similar.

- [ ] **Step 8: Commit**

```bash
git add packages/pages-schema/ packages/pages-ui/ packages/pages-component/
git commit -m "wip(#674): remove old schemas, parser branches, layout components, migration script Refs #674"
```

### Task 4: Remove old grid model from pages-builder

**Files:**
- Modify: `packages/pages-builder/src/tree/builder-tree.ts:4,12,85-103,172-175`
- Modify: `packages/pages-builder/src/shell/scope-collector.ts:48-53,68-86,97-112,130-133`
- Modify: `packages/pages-builder/src/shell/yaml-paste-indicators.ts:24-61`
- Modify: `packages/pages-builder/src/data/preview-yaml-transform.ts:14-21`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts:7,492-519,860-887,1028-1039,1167-1194,1429-1451,1496-1537`

**Interfaces:**
- Consumes: Task 2's removal of `RowNode`/`ColumnNode` from pages-document
- Produces: Builder that only handles `'grid'`, `'items'`, and `'flat'` layout modes

- [ ] **Step 1: Clean up builder-tree.ts**

In `packages/pages-builder/src/tree/builder-tree.ts`:
- Line 4: remove `RowNode`, `ColumnNode` from import
- Line 12: remove `'row'` and `'column'` from `TreeNodeType` union
- Lines 85-103: delete `buildColumnNode()` and `buildRowNode()` functions
- Lines 172-175: delete `mode === 'rows'` and `mode === 'columns'` branches from `buildPageNode()`
- Remove `'row'`/`'column'` from `_isContainerNode()` and `_isPasteTarget()` checks

- [ ] **Step 2: Clean up scope-collector.ts**

In `packages/pages-builder/src/shell/scope-collector.ts`:
- Lines 48-53: delete `mode === 'rows'` and `mode === 'columns'` branches in `collectFromPage`
- Lines 68-86: delete `nodeType === 'row'` and `nodeType === 'column'` branches entirely
- Lines 97-112: delete `path[2] === 'rows'` and `path[2] === 'columns'` branches in component resolution
- Lines 130-133: delete rows/columns branches in dataset traversal

- [ ] **Step 3: Simplify yaml-paste-indicators.ts**

In `packages/pages-builder/src/shell/yaml-paste-indicators.ts`:
- Lines 24-37: delete the `mode === 'rows'` branch under `fragmentType === 'component'`
- Lines 45-50: delete the `fragmentType === 'row'` branch entirely
- Lines 51-61: delete the `fragmentType === 'column'` branch entirely
- Keep only `'component'` with `mode === 'flat'` and `'grid-item'` handling

- [ ] **Step 4: Clean up preview-yaml-transform.ts**

In `packages/pages-builder/src/data/preview-yaml-transform.ts`:
- Lines 14-21: delete the `page.getRows()` and `page.getColumns()` iteration blocks
- Verify that grid/items mode traversal exists (via `page.getItems()` or similar). If missing, add it — this may be a pre-existing gap.

- [ ] **Step 5: Clean up builder-shell.ts**

In `packages/pages-builder/src/shell/builder-shell.ts`:
- Line 7: remove `RowNode`, `ColumnNode` from import
- Lines 492-519: delete `_injectEmptyColumnPlaceholders()` method
- Lines 860-887: delete row/column insert handler branches
- Lines 1028-1039: delete add-to-row logic
- Lines 1167-1194: delete property panel for `'row'` and `'column'` node types
- Lines 1429-1451: delete row/column dispatch in drop handler
- Lines 1496-1537: delete `_findRowAtPath()`, `_findColumnAtPath()`, and row/column navigation in `_findComponentAtPath()`

- [ ] **Step 6: Build and fix any remaining compile errors**

Run: `yarn build`

Fix any remaining import errors, type errors, or references to removed symbols. The TypeScript compiler will flag everything that still references `RowNode`, `ColumnNode`, `isColumns`, `isRows`, `ColumnsProps`, `RowsProps`, or old `LayoutMode` values.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-builder/
git commit -m "wip(#674): remove old grid model from pages-builder Refs #674"
```

---

## Batch 3: Tests and Verification

### Task 5: Update tests and verify clean build

**Files:**
- Modify: `packages/pages-document/src/page-document.test.ts`
- Modify: `packages/pages-document/src/grid-item-node.test.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.test.ts`
- Modify: `packages/pages-ui/src/parser/desugar-new-types.test.ts`
- Modify: `packages/pages-ui/src/parser/layout-placement.test.ts`
- Modify: `packages/pages-schema/src/document-schema.test.ts`
- Modify: `packages/pages-component/src/renderer/grid-migration.test.ts` (already deleted in Task 3)

**Interfaces:**
- Consumes: All removals from Tasks 2-4
- Produces: Clean test suite with no references to old model

- [ ] **Step 1: Clean up page-document.test.ts**

In `packages/pages-document/src/page-document.test.ts`:
- Delete `ROWS_PAGE` and `COLUMNS_PAGE` test fixtures (lines 12-35)
- Delete the `'rows and columns'` test suite (~lines 193-257, ~7 tests)
- Delete `insertColumnAt`/`insertRowAt` tests (~lines 414-461)
- Delete `wrapInRow` tests (~lines 660-754)
- Keep all tests for the new grid/items/flat models

- [ ] **Step 2: Clean up grid-item-node.test.ts**

In `packages/pages-document/src/grid-item-node.test.ts`:
- Remove any `RowNode`/`ColumnNode` references or assertions

- [ ] **Step 3: Clean up builder-shell.test.ts**

In `packages/pages-builder/src/shell/builder-shell.test.ts`:
- Remove all `getRows()`/`getColumns()` chains in assertions
- Remove `RowNode`/`ColumnNode` imports and type references
- Remove any test cases that set up pages with `rows:` layout

- [ ] **Step 4: Clean up desugar-new-types.test.ts**

In `packages/pages-ui/src/parser/desugar-new-types.test.ts`:
- Lines 152-206: delete the `type: "columns"` desugaring tests (the desugaring branch was removed in Task 3)

- [ ] **Step 5: Clean up layout-placement.test.ts**

In `packages/pages-ui/src/parser/layout-placement.test.ts`:
- Remove the placeholder test about "rows/columns format removed — migration script converts to items"
- If the file is now empty, delete it

- [ ] **Step 6: Clean up document-schema.test.ts**

In `packages/pages-schema/src/document-schema.test.ts`:
- Remove `"columns"` and `"rows"` from valid component type assertions (lines 69-70)

- [ ] **Step 7: Search for any remaining references**

Use `ide_search_text` to find any remaining references across the codebase:
- Search for `RowNode` (should find 0 results)
- Search for `ColumnNode` (should find 0 results)
- Search for `getRows()` (should find 0 source results — may appear in git history or docs)
- Search for `getColumns()` (should find 0 source results — `DatasetNode.getColumns()` is different and stays)
- Search for `isColumns` (should find 0 results)
- Search for `isRows` (should find 0 results)
- Search for `ColumnsProps` (should find 0 results)
- Search for `RowsProps` (should find 0 results)
- Search for `rowSchema` (should find 0 results)
- Search for `columnSchema` (should find 0 results)
- Search for `grid-migration` (should find 0 source results)

Flag any results that aren't in docs, comments, or git history.

- [ ] **Step 8: Run full build and test suite**

Run: `yarn build`
Expected: Clean build with 0 errors

Run: `yarn test`
Expected: All tests pass

- [ ] **Step 9: Verify graph-stencil-swf test fixture**

Check if `packages/graph-stencil-swf/src/test-fixtures/test.page.yaml` exists. If so, migrate it from `pages:\n- columns:\n` to valid grid format or remove it.

- [ ] **Step 10: Commit**

```bash
git add .
git commit -m "feat(#674): remove all old grid model tests, verify clean build Refs #674"
```

---

## References

- [specs/issue-674-remove-old-grid-model/2026-11-17-remove-old-grid-model-design.md] — design spec
- [specs/issue-674-remove-old-grid-model/decisions.md] — D1 (clean break), D2 (remove layout components), D3 (YAML first, TS bottom-up)
- [docs/specs/issue-625-unified-grid-layout-model/decisions.md] — D9: unify on items, remove rows/columns
- [packages/pages-document/src/page-document.ts] — RowNode/ColumnNode classes
- [packages/pages-document/src/types.ts:17] — LayoutMode definition
- [packages/pages-schema/src/document-schema.ts:107-123] — rowSchema/columnSchema
- [packages/pages-component/src/renderer/grid-migration.ts] — migration script
- [packages/pages-component/src/renderer/layout.ts] — LAYOUT_TYPES, applyLayoutCSS
- [packages/pages-component/src/model/type-guards.ts:165-171] — isColumns/isRows
- [packages/pages-builder/src/tree/builder-tree.ts] — TreeNodeType, buildRowNode/buildColumnNode
- [packages/pages-builder/src/shell/scope-collector.ts] — row/column traversal
- [packages/pages-builder/src/shell/builder-shell.ts] — row/column path resolution
- [GitHub #674] — focal issue
- [GitHub #625] — unified grid layout model (predecessor)
