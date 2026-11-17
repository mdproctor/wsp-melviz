# Remove Old Rows/Columns Grid Model — Design Spec

**Issue:** casehubio/casehub-pages#674
**Date:** 2026-11-17
**Depends on:** #625 (unified grid layout model)

## Goal

Remove every remnant of the old `rows:` / `columns:` page-level grid layout and the `type: "columns"` / `type: "rows"` layout components from the pages runtime. After this work, the only layout model is `items:` / `grid:` with coordinate-based `GridPlacement`.

## What Gets Removed

### 1. Page-level rows/columns model

The old format where pages define layout as:
```yaml
rows:
  - columns:
    - span: 6
      components: [...]
    - span: 6
      components: [...]
```

### 2. Layout components (type: columns, type: rows)

Component-level layout wrappers:
```yaml
type: columns
properties:
  distribution: [6, 6]
slots:
  column-0: [...]
  column-1: [...]
```

### 3. LayoutMode enum values

`'rows'` and `'columns'` from `LayoutMode = 'grid' | 'flat' | 'items' | 'rows' | 'columns'`. After removal: `LayoutMode = 'grid' | 'flat' | 'items'`.

## Packages Affected (bottom-up order)

### pages-document (foundation)

| File | Remove |
|------|--------|
| `src/types.ts` | `'rows' \| 'columns'` from `LayoutMode` |
| `src/page-document.ts` | `RowNode` class (~50 lines), `ColumnNode` class (~40 lines), `getRows()`, `getColumns()`, `addRow()`, `insertRowAt()`, `wrapInRow()`, `removeChild()` row/column branches, insertion dispatch for `'rows'`/`'columns'` |
| `src/index.ts` | `RowNode`, `ColumnNode` exports |
| `src/insertion-constraints.ts` | `'row'`/`'column'` from `structuralTypes`, `ROW_CONSTRAINT`, row/column cases in `allowedTypesAt()` and `allowedTypesForSiblingOf()` |
| `src/page-document.test.ts` | `ROWS_PAGE`/`COLUMNS_PAGE` fixtures, `'rows and columns'` test suite, `insertColumnAt`/`insertRowAt` tests, `wrapInRow` tests |
| `src/grid-item-node.test.ts` | Any `RowNode`/`ColumnNode` references |

### pages-schema (validation)

| File | Remove |
|------|--------|
| `src/document-schema.ts` | `columnSchema`, `rowSchema` definitions; `rows`/`columns` from `pageEntrySchema`; `"columns"`/`"rows"` from component type discriminated union |
| `src/component-schemas.generated.ts` | `columnsPropsSchema`, `rowsPropsSchema` |
| `src/document-schema.test.ts` | `"columns"`/`"rows"` as valid component types in tests |

### pages-ui (parser/desugarer)

| File | Remove |
|------|--------|
| `src/parser/page-parser.ts` | Stale JSDoc documenting `rows:` format (lines 13, 86, 259-270) |
| `src/parser/component-desugar.ts` | `type: "columns"` desugaring branch (lines 432-459) |
| `src/parser/desugar-new-types.test.ts` | `type: "columns"` desugaring tests |
| `src/parser/layout-placement.test.ts` | Placeholder test about rows/columns removal |

### pages-component (renderer/migration)

| File | Remove |
|------|--------|
| `src/renderer/grid-migration.ts` | Entire file (migration script — no longer needed after YAML migration) |
| `src/renderer/grid-migration.test.ts` | Entire file |
| `src/renderer/layout.ts` | `"columns"` and `"rows"` from `LAYOUT_TYPES` set; `case "columns":` and `case "rows":` from `applyLayoutCSS` |
| `src/model/component-props.ts` | `ColumnsProps`, `RowsProps` types |
| `src/model/type-guards.ts` | `isColumns()`, `isRows()` functions |

### pages-builder (UI)

| File | Remove |
|------|--------|
| `src/shell/builder-shell.ts` | `RowNode`/`ColumnNode` imports, `_injectEmptyColumnPlaceholders()`, row/column branches in insert handler, property panel for `'row'`/`'column'`, drop handler row/column dispatch, `_findRowAtPath()`, `_findColumnAtPath()`, row/column navigation in `_findComponentAtPath()` |
| `src/tree/builder-tree.ts` | `buildColumnNode()`, `buildRowNode()`, `'rows'`/`'columns'` branches in `buildPageNode()`, `'row'`/`'column'` in `TreeNodeType`, `_isContainerNode()` row/column check, `_isPasteTarget()` row/column fragment types |
| `src/shell/scope-collector.ts` | Row/column traversal |
| `src/shell/yaml-paste-indicators.ts` | Row/column paste location iteration |
| `src/data/preview-yaml-transform.ts` | Row/column snapshot iteration |
| `src/shell/builder-shell.test.ts` | `getRows()`/`getColumns()`/`RowNode`/`ColumnNode` references |

### YAML files (migration)

| File | Action |
|------|--------|
| `backend/data-json/src/test/resources/org/apps/tasks/tasks.page.yaml` | Migrate to `grid:` format |
| `backend/data-json/src/test/resources/org/apps/contacts/contacts.page.yaml` | Migrate to `grid:` format |
| `backend/data-json/src/test/resources/org/apps/platform/platform.page.yaml` | Migrate to `grid:` format |
| `examples/samples/Layout/includes/team-detail.dash.yml` | Migrate to `grid:` format |
| `examples/samples/Layout/includes/reports.dash.yml` | Migrate to `grid:` format |
| `examples/samples/Layout/includes/admin-settings.dash.yml` | Migrate to `grid:` format |
| `examples/samples/Custom Components/Code Editor.page.yaml` | Remove old-format YAML from HTML sample block |
| `packages/graph-stencil-swf/src/test-fixtures/test.page.yaml` (if exists) | Migrate or remove |

### Stale build artifacts

| Path | Action |
|------|--------|
| `backend/data-json/target/` | Rebuild (not committed) |
| `examples/server/target/` | Rebuild (not committed) |
| `examples/dist/` | Rebuild from source |

## Approach

1. **YAML migration** — run existing `migrateRowsToItems()` on the 7 source files, verify output, commit
2. **pages-document** — remove `RowNode`, `ColumnNode`, `LayoutMode` values, related methods and exports
3. **pages-schema** — remove `rowSchema`, `columnSchema`, old format acceptance in `pageEntrySchema`, layout component schemas
4. **pages-ui** — remove stale JSDoc, columns desugaring branch, related tests
5. **pages-component** — delete `grid-migration.ts` and test, remove layout type entries, props, type guards
6. **pages-builder** — remove all row/column traversal, tree building, path resolution, property panels, paste indicators
7. **Tests** — update/remove all affected test files, verify `yarn test` passes
8. **Dist rebuild** — regenerate `examples/dist/` from migrated sources

## What Stays

- `type: "grid"` component with `grid?: readonly GridItem[]`
- `GridPlacement` (`positioned` and `auto` modes)
- `GridItem` (`{ at: GridPlacement; component?: Component }`)
- `GridProps` (`{ columns: number; rowHeight?: number }`)
- `LayoutMode` values: `'grid' | 'flat' | 'items'`
- All grid editing infrastructure (#625, #639, #653, #654, #665)

## References

- docs/specs/issue-625-unified-grid-layout-model/2026-11-08-unified-grid-layout-model-design.md — D9: unify on items
- docs/specs/issue-625-unified-grid-layout-model/decisions.md — D9: clean break
- packages/pages-component/src/renderer/grid-migration.ts — existing migration script
- packages/pages-component/src/model/types.ts — current GridItem/GridPlacement types
- packages/pages-document/src/page-document.ts — RowNode/ColumnNode to remove
- packages/pages-schema/src/document-schema.ts — schema validators to remove
