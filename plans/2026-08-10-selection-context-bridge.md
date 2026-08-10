# Selection Context Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #298 — selection context bridge, RuntimeContext integration
**Issue group:** #298, #299, #300 (branch covers epic #289; this plan covers #298 only)

**Goal:** Bridge `PagesDataTable` selection events to `RuntimeContext` so that detail datasets with parameterised URL templates (`#{selection.<datasetId>.<field>}`) auto-refetch on row selection.

**Architecture:** Add `selection` field to `RuntimeContext`, add `updateSelection()` to `ContextManager`, add `selection-change` event listener in `site.ts` following the same pattern as `pages-filter`/`pages-sort`/`pages-page`. Template resolution and deferred fetch come free from existing infrastructure.

**Tech Stack:** TypeScript, Vitest, Yarn workspaces

## Global Constraints

- Template syntax is `#{var}` (not `${var}`)
- Single-row selection only — multi-select is out of scope
- `SelectionChangeDetail` is already exported from `@casehubio/pages-table`
- `pages-runtime` already depends on `@casehubio/pages-table`
- IntelliJ MCP is not available for this TypeScript project — use Read/Edit tools
- Test runner: `yarn workspace @casehubio/pages-runtime run test` / `yarn workspace @casehubio/pages-component run test`

---

### Task 1: Add `selection` to RuntimeContext and EMPTY_CONTEXT

**Files:**
- Modify: `packages/pages-component/src/context/types.ts:1-22`
- Test: `packages/pages-component/src/context/template-parser.test.ts`

**Interfaces:**
- Produces: `RuntimeContext.selection: Readonly<Record<string, Record<string, unknown>>>` — used by Tasks 2, 3, 4
- Produces: `EMPTY_CONTEXT` with `selection: {}` — used by Task 2

- [ ] **Step 1: Write the failing test**

Add to `packages/pages-component/src/context/template-parser.test.ts`:

```typescript
describe("selection template resolution", () => {
  it("resolves selection field path", () => {
    const ctx: RuntimeContext = {
      filter: {},
      datasets: {},
      page: { name: "test", path: "/test" },
      params: {},
      selection: {
        adverseEvents: { id: 42, grade: "Grade 3" },
      },
    };
    expect(resolveTemplate("#{selection.adverseEvents.id}", ctx, "none")).toBe("42");
    expect(resolveTemplate("#{selection.adverseEvents.grade}", ctx, "none")).toBe("Grade 3");
  });

  it("resolves to empty string when no selection exists", () => {
    const ctx: RuntimeContext = {
      filter: {},
      datasets: {},
      page: { name: "test", path: "/test" },
      params: {},
      selection: {},
    };
    expect(resolveTemplate("#{selection.adverseEvents.id}", ctx, "none")).toBe("");
  });

  it("allTemplateVarsResolved returns false when selection is missing", () => {
    const ctx: RuntimeContext = {
      filter: {},
      datasets: {},
      page: { name: "test", path: "/test" },
      params: {},
      selection: {},
    };
    expect(allTemplateVarsResolved("#{selection.adverseEvents.id}", ctx)).toBe(false);
  });

  it("allTemplateVarsResolved returns true when selection exists", () => {
    const ctx: RuntimeContext = {
      filter: {},
      datasets: {},
      page: { name: "test", path: "/test" },
      params: {},
      selection: {
        adverseEvents: { id: 42 },
      },
    };
    expect(allTemplateVarsResolved("#{selection.adverseEvents.id}", ctx)).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- --run template-parser.test.ts`
Expected: TypeScript error — `selection` does not exist on `RuntimeContext`

- [ ] **Step 3: Add `selection` field to RuntimeContext and EMPTY_CONTEXT**

In `packages/pages-component/src/context/types.ts`, add `selection` to the interface and update `EMPTY_CONTEXT`:

```typescript
export interface RuntimeContext {
  readonly filter: Record<string, readonly string[]>;
  readonly datasets: Record<string, DataSetSnapshot>;
  readonly page: { readonly name: string; readonly path: string };
  readonly params: Record<string, string>;
  readonly row?: Record<string, unknown>;
  readonly selection: Readonly<Record<string, Record<string, unknown>>>;
}
```

```typescript
export const EMPTY_CONTEXT: RuntimeContext = {
  filter: {},
  datasets: {},
  page: { name: "", path: "" },
  params: {},
  selection: {},
};
```

- [ ] **Step 4: Fix any existing tests that construct RuntimeContext without `selection`**

Existing tests in `template-parser.test.ts` construct `RuntimeContext` literals without `selection`. Add `selection: {}` to each. The `context-wiring.test.ts` tests use `EMPTY_CONTEXT` or spread from it, so they should pick up the new field automatically — verify by running the full test suite.

Run: `yarn workspace @casehubio/pages-component run test -- --run`
Run: `yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: All existing tests pass. New selection tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add packages/pages-component/src/context/types.ts packages/pages-component/src/context/template-parser.test.ts
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "feat(#298): add selection field to RuntimeContext

Refs #298"
```

---

### Task 2: Add `updateSelection()` to ContextManager

**Files:**
- Modify: `packages/pages-runtime/src/context-wiring.ts:38-160` (ContextManager class)
- Test: `packages/pages-runtime/src/context-wiring.test.ts`

**Interfaces:**
- Consumes: `RuntimeContext.selection` from Task 1
- Produces: `ContextManager.updateSelection(datasetId: string, row: Record<string, unknown> | null): void` — used by Tasks 3, 4

- [ ] **Step 1: Write the failing tests**

Add to `packages/pages-runtime/src/context-wiring.test.ts`, inside the `describe("ContextManager")` block:

```typescript
describe("updateSelection", () => {
  it("sets selection for a dataset", () => {
    const row = { id: 42, name: "Event A" };
    manager.updateSelection("adverseEvents", row);

    const ctx = manager.getContext();
    expect(ctx.selection["adverseEvents"]).toEqual({ id: 42, name: "Event A" });
  });

  it("clears selection when row is null", () => {
    manager.updateSelection("adverseEvents", { id: 42 });
    manager.updateSelection("adverseEvents", null);

    const ctx = manager.getContext();
    expect(ctx.selection["adverseEvents"]).toBeUndefined();
  });

  it("supports multiple independent selections", () => {
    manager.updateSelection("adverseEvents", { id: 1 });
    manager.updateSelection("deviations", { id: 2 });

    const ctx = manager.getContext();
    expect(ctx.selection["adverseEvents"]).toEqual({ id: 1 });
    expect(ctx.selection["deviations"]).toEqual({ id: 2 });
  });

  it("triggers evaluateAll for registered consumers", () => {
    const apply = vi.fn();
    const element = document.createElement("div");
    document.body.appendChild(element);

    const consumer: ContextConsumer = {
      element,
      templates: new Map([
        [
          "test",
          {
            template: "ID: #{selection.adverseEvents.id}",
            escapeMode: "none" as const,
            lastResolved: "",
            apply,
          },
        ],
      ]),
      suspended: false,
    };

    manager.registerConsumer(consumer);
    manager.updateSelection("adverseEvents", { id: 42 });

    expect(apply).toHaveBeenCalledWith("ID: 42");
    element.remove();
  });

  it("clears all selections via clearAllSelections", () => {
    manager.updateSelection("adverseEvents", { id: 1 });
    manager.updateSelection("deviations", { id: 2 });
    manager.clearAllSelections();

    const ctx = manager.getContext();
    expect(ctx.selection).toEqual({});
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run context-wiring.test.ts`
Expected: FAIL — `updateSelection` is not a function

- [ ] **Step 3: Implement `updateSelection` and `clearAllSelections` on ContextManager**

Add to `packages/pages-runtime/src/context-wiring.ts`, inside the `ContextManager` class, after `updateParams`:

```typescript
updateSelection(datasetId: string, row: Record<string, unknown> | null): void {
  const { [datasetId]: _, ...rest } = this.#context.selection;
  this.#context = {
    ...this.#context,
    selection: row ? { ...rest, [datasetId]: row } : rest,
  };
  this.evaluateAll();
}

clearAllSelections(): void {
  if (Object.keys(this.#context.selection).length === 0) return;
  this.#context = { ...this.#context, selection: {} };
  this.evaluateAll();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run context-wiring.test.ts`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add packages/pages-runtime/src/context-wiring.ts packages/pages-runtime/src/context-wiring.test.ts
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "feat(#298): add updateSelection and clearAllSelections to ContextManager

Refs #298"
```

---

### Task 3: Add `selection-change` listener and clearing logic in site.ts

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` (add event listener, clear on page nav and dataset refresh)
- Test: `packages/pages-runtime/src/site.test.ts` (or new `packages/pages-runtime/src/selection-bridge.test.ts`)

**Interfaces:**
- Consumes: `ContextManager.updateSelection(datasetId, row | null)` from Task 2
- Consumes: `ContextManager.clearAllSelections()` from Task 2
- Consumes: `SelectionChangeDetail` from `@casehubio/pages-table`
- Consumes: `findComponentId(e)`, `registry`, `contextManager` from site.ts locals
- Produces: wired event listener — no new public API

**Note:** site.ts is large and its tests are integration-heavy. The selection listener and clearing logic should be tested through the `loadSite` integration test pattern used in `site.test.ts`. However, the TypedRow → Record conversion is a pure function and should be unit-tested separately.

- [ ] **Step 1: Write unit test for TypedRow → Record conversion**

Create `packages/pages-runtime/src/selection-bridge.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { typedRowToRecord } from "./selection-bridge.js";
import { columnId, ColumnType } from "@casehubio/pages-data";
import type { TypedRow, CellValue } from "@casehubio/pages-data";

function createTypedRow(
  columns: Array<{ id: string; type: string; value: unknown }>,
): TypedRow {
  const cells: CellValue[] = columns.map((col) =>
    col.value === null
      ? ({ type: "NULL" } as CellValue)
      : ({ type: col.type, value: col.value } as CellValue),
  );
  return {
    cells,
    cell(colId) {
      const idx = columns.findIndex((c) => c.id === (colId as string));
      return idx >= 0 ? cells[idx]! : ({ type: "NULL" } as CellValue);
    },
    number(colId) {
      const cell = this.cell(colId);
      return cell.type === ColumnType.NUMBER ? cell.value : 0;
    },
    text(colId) {
      const cell = this.cell(colId);
      return cell.type === ColumnType.TEXT ? cell.value : "";
    },
    date(colId) {
      const cell = this.cell(colId);
      return cell.type === ColumnType.DATE ? cell.value : new Date(0);
    },
  };
}

describe("typedRowToRecord", () => {
  it("extracts cell values by column ID", () => {
    const columns = [
      { id: columnId("id"), name: "ID", type: ColumnType.NUMBER },
      { id: columnId("name"), name: "Name", type: ColumnType.TEXT },
    ];
    const row = createTypedRow([
      { id: "id", type: ColumnType.NUMBER, value: 42 },
      { id: "name", type: ColumnType.TEXT, value: "Event A" },
    ]);

    const result = typedRowToRecord(row, columns);
    expect(result).toEqual({ id: 42, name: "Event A" });
  });

  it("converts NULL cells to null", () => {
    const columns = [
      { id: columnId("grade"), name: "Grade", type: ColumnType.TEXT },
    ];
    const row = createTypedRow([
      { id: "grade", type: "NULL", value: null },
    ]);

    const result = typedRowToRecord(row, columns);
    expect(result).toEqual({ grade: null });
  });

  it("converts Date cells to ISO string", () => {
    const columns = [
      { id: columnId("onset"), name: "Onset", type: ColumnType.DATE },
    ];
    const d = new Date("2026-01-15T00:00:00.000Z");
    const row = createTypedRow([
      { id: "onset", type: ColumnType.DATE, value: d },
    ]);

    const result = typedRowToRecord(row, columns);
    expect(result).toEqual({ onset: "2026-01-15T00:00:00.000Z" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run selection-bridge.test.ts`
Expected: FAIL — module `./selection-bridge.js` not found

- [ ] **Step 3: Create `selection-bridge.ts` with `typedRowToRecord`**

Create `packages/pages-runtime/src/selection-bridge.ts`:

```typescript
import type { CellValue, Column, TypedRow } from "@casehubio/pages-data";

export function typedRowToRecord(
  row: TypedRow,
  columns: readonly Column[],
): Record<string, unknown> {
  const record: Record<string, unknown> = {};
  for (const col of columns) {
    const cell = row.cell(col.id);
    if (cell.type === "NULL") {
      record[col.id as string] = null;
    } else if (cell.type === "DATE") {
      record[col.id as string] = (cell.value as Date).toISOString();
    } else {
      record[col.id as string] = cell.value;
    }
  }
  return record;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run selection-bridge.test.ts`
Expected: PASS

- [ ] **Step 5: Wire the `selection-change` listener into site.ts**

In `packages/pages-runtime/src/site.ts`:

Add import at top:
```typescript
import type { SelectionChangeDetail } from "@casehubio/pages-table";
import { typedRowToRecord } from "./selection-bridge.js";
```

Add event listener after the `pages-sort` listener block (after line ~663), following the exact same pattern:

```typescript
target.addEventListener("selection-change", ((e: Event) => {
  const detail = (e as CustomEvent<SelectionChangeDetail>).detail;
  const componentId = findComponentId(e);
  if (!componentId) return;

  const entry = registry.get(componentId);
  if (!entry?.originalLookup) return;

  const datasetId = entry.originalLookup.dataSetId as string;
  if (detail.selectedRows.length > 0) {
    const row = detail.selectedRows[0]!;
    const ds = entry.vizElement?.dataSet as TypedDataSet | undefined;
    if (!ds) return;
    contextManager.updateSelection(datasetId, typedRowToRecord(row, ds.columns));
  } else {
    contextManager.updateSelection(datasetId, null);
  }
}), { signal: abortController.signal });
```

- [ ] **Step 6: Add selection clearing on page navigation**

In `packages/pages-runtime/src/site.ts`, find the `navigateInternal` function (line ~341). After `contextManager.updatePage(currentPage, currentPage)`, add:

```typescript
contextManager.clearAllSelections();
```

- [ ] **Step 7: Add selection clearing on dataset refresh**

In `packages/pages-runtime/src/site.ts`, find the `onChanged` callback in `createDataSetManager` (line ~177). After `contextManager.updateDataset(id, dataset)`, add:

```typescript
contextManager.updateSelection(id as string, null);
```

This clears any selection for the dataset that just received fresh data.

- [ ] **Step 8: Run the full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: All tests pass (existing + new)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add packages/pages-runtime/src/selection-bridge.ts packages/pages-runtime/src/selection-bridge.test.ts packages/pages-runtime/src/site.ts
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "feat(#298): wire selection-change listener and clearing in site.ts

Refs #298"
```

---

### Task 4: Update protocol and run full build

**Files:**
- Modify: `docs/protocols/casehub/pages-event-contract.md:60-88` (reserved event names table)

**Interfaces:**
- Consumes: all prior tasks

- [ ] **Step 1: Add `selection-change` to the reserved framework event names table**

In `docs/protocols/casehub/pages-event-contract.md`, add a new row to the reserved names table (after `pages-refresh-request`):

```
| `selection-change` | Master row selection for parameterised detail datasets | `PagesDataTable` component |
```

- [ ] **Step 2: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 3: Run lint**

Run: `yarn lint`
Expected: No lint errors (fix any that arise)

- [ ] **Step 4: Run full build**

Run: `yarn build`
Expected: Clean build, no errors

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "docs(#298): add selection-change to reserved framework event names

Refs #298"
```
