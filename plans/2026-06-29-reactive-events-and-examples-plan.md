# Reactive Dataset Events + Examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the DataSetManager's register/accumulate API with a unified event model, enable expression generators for inline datasets, add WebSocket push support with multiplexing, and create navigation/feature examples.

**Architecture:** The DataSetManager gains a single `apply(id, event)` method accepting a discriminated union of four event types (snapshot, append, replace, remove). All mutation sources — HTTP resolver, expression generator, WebSocket, local adapter — produce events. Schema restrictions are lifted to allow `accumulate` and `refreshTime` on content datasets. WebSocket sources use a connection pool keyed by base URL for multiplexing.

**Tech Stack:** TypeScript 5, Vitest, Zod, JSONata, WebSocket API

## Global Constraints

- TDD: every production change requires a failing test first
- Breaking changes are free — fix the design, don't protect callers
- Test commands: `yarn workspace @casehubio/pages-data run test -- --run` and `yarn workspace @casehubio/pages-ui run test -- --run` and `yarn workspace @casehubio/pages-runtime run test -- --run`
- Type check: `yarn typecheck`
- All branded types use factories: `columnId()`, `dataSetId()` — never raw `as` casts
- Vitest environment: `node` for pages-data, `jsdom` for pages-runtime and pages-viz

---

### Task 1: DataSetEvent Model + DataSetManager.apply()

**Files:**
- Create: `packages/pages-data/src/dataset/events.ts`
- Modify: `packages/pages-data/src/dataset/manager.ts` (interface lines 23-30, impl lines 58-158)
- Test: `packages/pages-data/src/dataset/manager.test.ts`
- Test: `packages/pages-data/src/dataset/manager-callback.test.ts`

**Interfaces:**
- Produces: `DataSetEvent`, `SnapshotEvent`, `AppendEvent`, `ReplaceEvent`, `RemoveEvent` types; `DataSetManager.apply(id, event)` method
- Produces: `DataSetEventListener` type alias `(event: DataSetEvent) => void`

- [ ] **Step 1: Write event type definitions**

Create `packages/pages-data/src/dataset/events.ts`:

```typescript
import type { TypedDataSet, TypedRow, ColumnId } from "./types.js";

export interface SnapshotEvent {
  readonly type: "snapshot";
  readonly dataset: TypedDataSet;
}

export interface AppendEvent {
  readonly type: "append";
  readonly rows: readonly TypedRow[];
  readonly maxRows?: number;
}

export interface ReplaceEvent {
  readonly type: "replace";
  readonly keyColumn: ColumnId;
  readonly key: string;
  readonly row: TypedRow;
}

export interface RemoveEvent {
  readonly type: "remove";
  readonly keyColumn: ColumnId;
  readonly key: string;
}

export type DataSetEvent =
  | SnapshotEvent
  | AppendEvent
  | ReplaceEvent
  | RemoveEvent;

export type DataSetEventListener = (event: DataSetEvent) => void;
```

- [ ] **Step 2: Write failing tests for apply() — snapshot event**

In `manager.test.ts`, add a new describe block after the existing "Accumulate" block:

```typescript
import type { DataSetEvent } from "./events.js";

describe("apply()", () => {
  it("snapshot replaces entire dataset", () => {
    const mgr = createDataSetManager();
    const ds1 = testDataSet([["Sales", "100", "2024-01-01"]]);
    const ds2 = testDataSet([["Engineering", "200", "2024-02-01"]]);

    mgr.apply(ID_A, { type: "snapshot", dataset: ds1 });
    expect(mgr.get(ID_A)?.rows).toHaveLength(1);

    mgr.apply(ID_A, { type: "snapshot", dataset: ds2 });
    expect(mgr.get(ID_A)?.rows).toHaveLength(1);
    expect(mgr.get(ID_A)?.rows[0]!.text(columnId("dept"))).toBe("Engineering");
  });
});
```

- [ ] **Step 3: Run test — verify it fails**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: FAIL — `mgr.apply is not a function`

- [ ] **Step 4: Add apply() to interface and implement snapshot**

In `manager.ts`, add to the `DataSetManager` interface (after `accumulate`):

```typescript
apply(id: DataSetId, event: DataSetEvent): void;
```

Add import at top:
```typescript
import type { DataSetEvent } from "./events.js";
```

In `DataSetManagerImpl`, add after the `accumulate` method:

```typescript
apply(id: DataSetId, event: DataSetEvent): void {
  switch (event.type) {
    case "snapshot":
      this.datasets.set(id, event.dataset);
      this.options?.onChanged?.(id, event.dataset);
      break;
    case "append":
    case "replace":
    case "remove":
      throw new Error(`Event type "${event.type}" not yet implemented`);
  }
}
```

- [ ] **Step 5: Run test — verify it passes**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: PASS

- [ ] **Step 6: Write failing tests for append event**

Add to the `apply()` describe block:

```typescript
it("append adds rows to END of existing dataset", () => {
  const mgr = createDataSetManager();
  const seed = testDataSet([["Sales", "100", "2024-01-01"]]);
  mgr.apply(ID_A, { type: "snapshot", dataset: seed });

  const newRow = testDataSet([["Engineering", "200", "2024-02-01"]]).rows[0]!;
  mgr.apply(ID_A, { type: "append", rows: [newRow] });

  const result = mgr.get(ID_A)!;
  expect(result.rows).toHaveLength(2);
  expect(result.rows[0]!.text(columnId("dept"))).toBe("Sales");
  expect(result.rows[1]!.text(columnId("dept"))).toBe("Engineering");
});

it("append trims from START when maxRows exceeded", () => {
  const mgr = createDataSetManager();
  const seed = testDataSet([
    ["Sales", "100", "2024-01-01"],
    ["Engineering", "200", "2024-02-01"],
  ]);
  mgr.apply(ID_A, { type: "snapshot", dataset: seed });

  const newRow = testDataSet([["Marketing", "50", "2024-03-01"]]).rows[0]!;
  mgr.apply(ID_A, { type: "append", rows: [newRow], maxRows: 2 });

  const result = mgr.get(ID_A)!;
  expect(result.rows).toHaveLength(2);
  expect(result.rows[0]!.text(columnId("dept"))).toBe("Engineering");
  expect(result.rows[1]!.text(columnId("dept"))).toBe("Marketing");
});

it("append to non-existent dataset is a no-op", () => {
  const mgr = createDataSetManager();
  const row = testDataSet([["Sales", "100", "2024-01-01"]]).rows[0]!;
  mgr.apply(ID_A, { type: "append", rows: [row] });
  expect(mgr.has(ID_A)).toBe(false);
});
```

- [ ] **Step 7: Run test — verify append tests fail**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: FAIL — `Event type "append" not yet implemented`

- [ ] **Step 8: Implement append event**

Replace the `case "append"` throw in `apply()`:

```typescript
case "append": {
  const existing = this.datasets.get(id);
  if (existing === undefined) {
    return;
  }
  const combined = [...existing.rows, ...event.rows];
  const trimmed = event.maxRows !== undefined && event.maxRows >= 0
    ? combined.slice(-event.maxRows)
    : combined;
  const result: TypedDataSet = { columns: existing.columns, rows: trimmed };
  this.datasets.set(id, result);
  this.options?.onChanged?.(id, result);
  break;
}
```

- [ ] **Step 9: Run test — verify append tests pass**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: PASS

- [ ] **Step 10: Write failing tests for replace event**

```typescript
it("replace updates all matching rows by key", () => {
  const mgr = createDataSetManager();
  const seed = testDataSet([
    ["Sales", "100", "2024-01-01"],
    ["Engineering", "200", "2024-02-01"],
  ]);
  mgr.apply(ID_A, { type: "snapshot", dataset: seed });

  const replacement = testDataSet([["Sales", "999", "2024-06-01"]]).rows[0]!;
  mgr.apply(ID_A, {
    type: "replace",
    keyColumn: columnId("dept"),
    key: "Sales",
    row: replacement,
  });

  const result = mgr.get(ID_A)!;
  expect(result.rows).toHaveLength(2);
  expect(result.rows[0]!.number(columnId("revenue"))).toBe(999);
  expect(result.rows[1]!.text(columnId("dept"))).toBe("Engineering");
});

it("replace is a no-op when no rows match", () => {
  const callback = vi.fn();
  const mgr = createDataSetManager({ onChanged: callback });
  const seed = testDataSet([["Sales", "100", "2024-01-01"]]);
  mgr.apply(ID_A, { type: "snapshot", dataset: seed });
  callback.mockClear();

  const replacement = testDataSet([["Unknown", "0", "2024-01-01"]]).rows[0]!;
  mgr.apply(ID_A, {
    type: "replace",
    keyColumn: columnId("dept"),
    key: "Unknown",
    row: replacement,
  });

  expect(callback).not.toHaveBeenCalled();
});
```

- [ ] **Step 11: Implement replace event**

```typescript
case "replace": {
  const existing = this.datasets.get(id);
  if (existing === undefined) return;
  let matched = false;
  const rows = existing.rows.map(row => {
    const cell = row.cell(event.keyColumn);
    if (cell.type !== "NULL" && String(cell.value) === event.key) {
      matched = true;
      return event.row;
    }
    return row;
  });
  if (!matched) return;
  const result: TypedDataSet = { columns: existing.columns, rows };
  this.datasets.set(id, result);
  this.options?.onChanged?.(id, result);
  break;
}
```

- [ ] **Step 12: Run test — verify replace tests pass**

- [ ] **Step 13: Write failing tests for remove event**

```typescript
it("remove filters out all matching rows by key", () => {
  const mgr = createDataSetManager();
  const seed = testDataSet([
    ["Sales", "100", "2024-01-01"],
    ["Engineering", "200", "2024-02-01"],
    ["Sales", "150", "2024-03-01"],
  ]);
  mgr.apply(ID_A, { type: "snapshot", dataset: seed });

  mgr.apply(ID_A, {
    type: "remove",
    keyColumn: columnId("dept"),
    key: "Sales",
  });

  const result = mgr.get(ID_A)!;
  expect(result.rows).toHaveLength(1);
  expect(result.rows[0]!.text(columnId("dept"))).toBe("Engineering");
});

it("remove is a no-op when no rows match", () => {
  const callback = vi.fn();
  const mgr = createDataSetManager({ onChanged: callback });
  const seed = testDataSet([["Sales", "100", "2024-01-01"]]);
  mgr.apply(ID_A, { type: "snapshot", dataset: seed });
  callback.mockClear();

  mgr.apply(ID_A, {
    type: "remove",
    keyColumn: columnId("dept"),
    key: "Unknown",
  });

  expect(callback).not.toHaveBeenCalled();
});
```

- [ ] **Step 14: Implement remove event**

```typescript
case "remove": {
  const existing = this.datasets.get(id);
  if (existing === undefined) return;
  const rows = existing.rows.filter(row => {
    const cell = row.cell(event.keyColumn);
    return cell.type === "NULL" || String(cell.value) !== event.key;
  });
  if (rows.length === existing.rows.length) return;
  const result: TypedDataSet = { columns: existing.columns, rows };
  this.datasets.set(id, result);
  this.options?.onChanged?.(id, result);
  break;
}
```

- [ ] **Step 15: Run all tests — verify everything passes**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: all tests PASS

- [ ] **Step 16: Update callback tests**

In `manager-callback.test.ts`, add tests verifying `apply()` fires `onChanged` for snapshot, append (when dataset exists), replace (when matched), remove (when matched). Add tests verifying no callback on append-to-nonexistent, replace-no-match, remove-no-match.

- [ ] **Step 17: Run full test suite**

Run: `yarn workspace @casehubio/pages-data run test -- --run`
Expected: all PASS

- [ ] **Step 18: Commit**

```
feat: DataSetEvent model and DataSetManager.apply()

Unified event API with snapshot, append, replace, remove semantics.
Append uses chronological order (existing + new) and trims from start.

Refs #36, #52
```

---

### Task 2: Caller Migration — Remove register() and accumulate()

**Files:**
- Modify: `packages/pages-data/src/dataset/manager.ts` — remove `register()`, `accumulate()` from interface and impl
- Modify: `packages/pages-data/src/dataset/external/resolver.ts` — `registerOrAccumulate` → apply(); join route → apply()
- Modify: `packages/pages-runtime/src/adapters/local-adapter.ts` — register() → apply() for save/delete/create
- Modify: `packages/pages-data/src/dataset/manager.test.ts` — migrate old tests
- Modify: `packages/pages-data/src/dataset/manager-callback.test.ts` — migrate old tests
- Modify: `packages/pages-data/src/dataset/external/resolver.test.ts` — migrate tests
- Modify: `packages/pages-runtime/src/data-pipeline.test.ts` — migrate any register() calls

**Interfaces:**
- Consumes: `DataSetManager.apply()` from Task 1
- Produces: Updated `DataSetManager` interface without `register`/`accumulate`

- [ ] **Step 1: Update manager.test.ts — migrate register() calls to apply(snapshot)**

Replace all `mgr.register(id, ds)` calls with `mgr.apply(id, { type: "snapshot", dataset: ds })` throughout the test file. The existing "Registry" and "Lookup Pipeline" describe blocks use `register()` to set up test data.

- [ ] **Step 2: Update manager.test.ts — migrate accumulate tests**

The "Accumulate" describe block tests become append tests. Key changes:
- `mgr.accumulate(id, ds, maxRows)` → `mgr.apply(id, { type: "append", rows: ds.rows, maxRows })`
- Ordering assertion changes: old was `[...new, ...existing]`, new is `[...existing, ...new]`
- First-accumulate-as-register: change to snapshot + append pattern (first call must be snapshot)
- Schema mismatch test: append inherits columns from existing dataset, so column validation is on the row cell level, not schema comparison

- [ ] **Step 3: Update manager-callback.test.ts — migrate register/accumulate calls**

Replace `mgr.register()` → `mgr.apply(id, { type: "snapshot", dataset })` and `mgr.accumulate()` → `mgr.apply(id, { type: "append", rows, maxRows })`.

- [ ] **Step 4: Run tests — verify they fail (register/accumulate still exist but tests no longer call them)**

Run: `yarn workspace @casehubio/pages-data run test -- --run`
Expected: PASS (tests migrated but old methods still exist)

- [ ] **Step 5: Remove register() and accumulate() from DataSetManager interface and DataSetManagerImpl**

In `manager.ts`:
- Remove `register` and `accumulate` from the `DataSetManager` interface (lines 24, 28)
- Remove the `register()` method implementation (lines 66-69)
- Remove the `accumulate()` method implementation (lines 83-134)

- [ ] **Step 6: Run typecheck — find all broken callers**

Run: `yarn typecheck`
Expected: errors in `resolver.ts`, `local-adapter.ts`, `data-pipeline.test.ts`, and any other callers

- [ ] **Step 7: Migrate resolver.ts**

Replace `registerOrAccumulate()` function (lines 142-152):

```typescript
function applyResolvedDataSet(
  def: ExternalDataSetDef,
  dataset: TypedDataSet,
  manager: DataSetManager,
): void {
  if (def.accumulate && manager.has(def.uuid)) {
    manager.apply(def.uuid, { type: "append", rows: dataset.rows, maxRows: def.cacheMaxRows });
  } else {
    manager.apply(def.uuid, { type: "snapshot", dataset });
  }
}
```

Replace the join route (line 103):
```typescript
ctx.manager.apply(def.uuid, { type: "snapshot", dataset });
```

Update the call site at line 137: `registerOrAccumulate(` → `applyResolvedDataSet(`

Add import: `import type { DataSetEvent } from "../events.js";` (if needed for types)

- [ ] **Step 8: Migrate local-adapter.ts**

Replace all three `manager.register(dataSetId, newDataset)` calls:
- Save (line 73): `manager.apply(dataSetId, { type: "snapshot", dataset: newDataset });`
- Delete (line 96): `manager.apply(dataSetId, { type: "snapshot", dataset: newDataset });`
- Create (line 128): `manager.apply(dataSetId, { type: "snapshot", dataset: newDataset });`

All three are full-dataset replacements, so snapshot is correct.

- [ ] **Step 9: Migrate data-pipeline.test.ts**

Replace any `manager.register()` calls with `manager.apply(id, { type: "snapshot", dataset })`.

- [ ] **Step 10: Migrate resolver.test.ts**

Update test assertions that check `manager.get()` after resolution. The accumulate test (lines 141-166) needs to set up initial data with `apply(snapshot)` before testing the accumulate path.

- [ ] **Step 11: Run full test suites + typecheck**

Run: `yarn typecheck && yarn workspace @casehubio/pages-data run test -- --run && yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all PASS, zero type errors

- [ ] **Step 12: Commit**

```
refactor: remove register/accumulate — all callers use apply()

Breaking change: DataSetManager.register() and accumulate() removed.
All mutation goes through apply(id, event) with typed events.

Refs #36, #52
```

---

### Task 3: Schema Changes — Lift Restrictions + keyColumn

**Files:**
- Modify: `packages/pages-data/src/dataset/external/schema.ts` (lines 52-57 refinements)
- Modify: `packages/pages-data/src/dataset/external/types.ts` (ExternalDataSetDef interface, line ~16-40)
- Test: `packages/pages-data/src/dataset/external/schema.test.ts` (find or create)

**Interfaces:**
- Produces: `keyColumn` field on `ExternalDataSetDef`; relaxed validation for accumulate/refreshTime

- [ ] **Step 1: Write failing tests for lifted restrictions**

Find or create `schema.test.ts`. Add tests:

```typescript
import { describe, it, expect } from "vitest";
import { parseExternalDataSetDef } from "./schema.js";

describe("schema refinements", () => {
  it("accepts accumulate on content datasets", () => {
    const def = parseExternalDataSetDef({
      uuid: "test",
      content: '[["a"]]',
      accumulate: true,
    });
    expect(def.accumulate).toBe(true);
  });

  it("accepts refreshTime on content + expression + accumulate", () => {
    const def = parseExternalDataSetDef({
      uuid: "test",
      content: '[["a"]]',
      expression: '[["b"]]',
      accumulate: true,
      refreshTime: "1second",
    });
    expect(def.refreshTime).toBe("1second");
  });

  it("rejects refreshTime on bare content (no expression)", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "test",
      content: '[["a"]]',
      refreshTime: "1second",
    })).toThrow();
  });

  it("rejects refreshTime on WebSocket URLs", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "test",
      url: "ws://localhost:8080/ws",
      refreshTime: "1second",
    })).toThrow();
  });

  it("accepts keyColumn field", () => {
    const def = parseExternalDataSetDef({
      uuid: "test",
      url: "ws://localhost:8080/ws",
      keyColumn: "id",
    });
    expect(def.keyColumn).toBe("id");
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Expected: FAIL — accumulate on content rejected, keyColumn unknown field

- [ ] **Step 3: Update schema.ts**

1. Add `keyColumn: z.string().optional()` to the base schema (after `accumulate`)

2. Replace refinement at lines 52-54 (accumulate requires url) — remove entirely

3. Replace refinement at lines 55-57 (refreshTime requires url):

```typescript
.refine(
  d => !d.refreshTime || (
    (d.url !== undefined && !d.url.startsWith("ws://") && !d.url.startsWith("wss://"))
    || (d.content !== undefined && d.expression !== undefined && d.accumulate === true)
  ),
  { message: "refreshTime requires a non-WebSocket url, or content + expression + accumulate" },
)
```

- [ ] **Step 4: Update types.ts**

Add to `ExternalDataSetDef` interface (after `accumulate`):

```typescript
readonly keyColumn?: string;
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-data run test -- --run`
Expected: all PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: clean

- [ ] **Step 7: Commit**

```
feat: lift schema restrictions, add keyColumn field

accumulate no longer requires url. refreshTime accepts content +
expression + accumulate, rejects WebSocket URLs. keyColumn added
for replace/remove event routing.

Refs #36, #52
```

---

### Task 4: Expression Generator + Pipeline Integration (#36)

**Files:**
- Create: `packages/pages-data/src/dataset/external/expression-generator.ts`
- Create: `packages/pages-data/src/dataset/external/expression-generator.test.ts`
- Modify: `packages/pages-runtime/src/data-pipeline.ts` (scheduleRefresh, lines 350-374)
- Modify: `packages/pages-runtime/src/site.ts` (expose presetRegistry for generator)
- Modify: `packages/pages-data/src/dataset/external/index.ts` (export evaluateGenerator)

**Interfaces:**
- Consumes: `DataSetManager.apply()` from Task 1; schema changes from Task 3
- Produces: `evaluateGenerator(expression, columns, presetRegistry)` → `Promise<TypedDataSet>`

- [ ] **Step 1: Write failing tests for evaluateGenerator**

Create `expression-generator.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { evaluateGenerator } from "./expression-generator.js";
import { createPresetRegistry } from "./presets/registry.js";

describe("evaluateGenerator", () => {
  const presetRegistry = createPresetRegistry();

  it("evaluates a JSONata expression that produces rows", async () => {
    const result = await evaluateGenerator(
      '[["hello", 42]]',
      [{ id: "name", type: "LABEL" }, { id: "value", type: "NUMBER" }],
      presetRegistry,
    );

    expect(result.columns).toHaveLength(2);
    expect(result.rows).toHaveLength(1);
    expect(result.rows[0]!.text(result.columns[0]!.id)).toBe("hello");
    expect(result.rows[0]!.number(result.columns[1]!.id)).toBe(42);
  });

  it("returns empty dataset for expression that produces no rows", async () => {
    const result = await evaluateGenerator(
      "[]",
      [{ id: "name", type: "LABEL" }],
      presetRegistry,
    );
    expect(result.rows).toHaveLength(0);
  });

  it("works without explicit column definitions (infers from data)", async () => {
    const result = await evaluateGenerator(
      '[["test", 1]]',
      undefined,
      presetRegistry,
    );
    expect(result.columns.length).toBeGreaterThan(0);
    expect(result.rows).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Implement evaluateGenerator**

Create `expression-generator.ts`:

```typescript
import type { TypedDataSet } from "../types.js";
import type { ExternalColumnDef } from "./types.js";
import type { PresetRegistry } from "./presets/registry.js";
import { extractDataSet } from "./extraction.js";

export async function evaluateGenerator(
  expression: string,
  columns: readonly ExternalColumnDef[] | undefined,
  presetRegistry: PresetRegistry,
): Promise<TypedDataSet> {
  return extractDataSet({
    data: null,
    contentType: undefined,
    def: {
      expression,
      columns,
    },
    presetRegistry,
  });
}
```

Note: `extractDataSet` already handles expression evaluation against raw data. When `data` is null and an expression is provided, it evaluates the expression as a generator (no input). The exact integration depends on `extractDataSet`'s signature — verify and adapt. If `extractDataSet` requires a non-null data input, create a minimal wrapper that evaluates the JSONata expression directly using the existing `evaluateExpression` helper, then tabulates the result.

- [ ] **Step 4: Run test — verify it passes**

- [ ] **Step 5: Write failing test for scheduleRefresh branching**

In `packages/pages-runtime/src/data-pipeline.test.ts`, add:

```typescript
it("scheduleRefresh calls evaluateGenerator for content + expression + accumulate", async () => {
  // Setup: register a dataset definition with content + expression + accumulate + refreshTime
  // Verify: after the refresh interval fires, new rows are appended via apply()
  // This is an integration test — use vi.useFakeTimers() to control the interval
});
```

The exact test shape depends on how `scheduleRefresh` is exposed. If it's internal to the pipeline closure, test via `handleDataRequest` with a dataset definition that has the right flags, then advance fake timers.

- [ ] **Step 6: Implement scheduleRefresh branching**

In `data-pipeline.ts`, modify `scheduleRefresh` (lines 350-374):

```typescript
function scheduleRefresh(def: ExternalDataSetDef, dataSetId: DataSetId): void {
  if (!def.refreshTime || refreshTimers.has(dataSetId)) return;

  // Guard: WebSocket datasets use server push, not polling
  if (def.url?.startsWith("ws://") || def.url?.startsWith("wss://")) return;

  const ms = parseRefreshTime(def.refreshTime);

  // Content + expression + accumulate: generator path
  if (def.content !== undefined && def.expression !== undefined && def.accumulate) {
    const timerId = setInterval(async () => {
      try {
        const generated = await evaluateGenerator(
          def.expression!,
          def.columns,
          resolverCtx!.presetRegistry,
        );
        manager.apply(dataSetId, {
          type: "append",
          rows: generated.rows,
          maxRows: def.cacheMaxRows,
        });
        // Push updated data to all subscribing components
        for (const [compId, entry] of registry) {
          if (entry.lookup.dataSetId === dataSetId) {
            pushData(entry.target, entry.lookup, entry.pagePath, entry.filterGroup, compId);
          }
        }
      } catch (e) {
        console.warn(`Expression generator failed for ${dataSetId}:`, e);
      }
    }, ms);
    refreshTimers.set(dataSetId, timerId);
    return;
  }

  // Existing URL refresh path (unchanged)
  // ...existing code...
}
```

Add import: `import { evaluateGenerator } from "@casehubio/pages-data/dist/dataset/external/expression-generator.js";`

- [ ] **Step 7: Run tests — verify they pass**

- [ ] **Step 8: Export evaluateGenerator from index.ts**

Add to `packages/pages-data/src/dataset/external/index.ts`:
```typescript
export { evaluateGenerator } from "./expression-generator.js";
```

- [ ] **Step 9: Run full suite + typecheck**

Run: `yarn typecheck && yarn workspace @casehubio/pages-data run test -- --run && yarn workspace @casehubio/pages-runtime run test -- --run`

- [ ] **Step 10: Commit**

```
feat: expression generator and pipeline integration

evaluateGenerator() evaluates JSONata expressions as row generators.
scheduleRefresh branches: content+expression+accumulate uses the
generator path; WebSocket URLs are skipped (server push).

Closes #36
```

---

### Task 5: WebSocketSource (#52)

**Files:**
- Create: `packages/pages-data/src/dataset/external/sources/websocket-source.ts`
- Create: `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`

**Interfaces:**
- Consumes: `DataSetEvent`, `DataSetEventListener` from Task 1
- Produces: `WebSocketSource` interface with `subscribe()`, `unsubscribe()`, `close()`; `createWebSocketSource(url)` factory

- [ ] **Step 1: Write failing tests for WebSocketSource**

Create `websocket-source.test.ts`. Use a mock WebSocket class (Vitest runs in node, no native WebSocket):

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { createWebSocketSource } from "./websocket-source.js";
import type { DataSetEvent } from "../../events.js";
import { dataSetId, columnId } from "../../types.js";

// Mock WebSocket for node environment
class MockWebSocket {
  static instances: MockWebSocket[] = [];
  onopen: (() => void) | null = null;
  onmessage: ((e: { data: string }) => void) | null = null;
  onclose: ((e: { code: number; reason: string }) => void) | null = null;
  onerror: ((e: unknown) => void) | null = null;
  readyState = 0; // CONNECTING
  sent: string[] = [];

  constructor(public url: string) {
    MockWebSocket.instances.push(this);
    setTimeout(() => { this.readyState = 1; this.onopen?.(); }, 0);
  }
  send(data: string) { this.sent.push(data); }
  close() { this.readyState = 3; this.onclose?.({ code: 1000, reason: "" }); }
}

describe("WebSocketSource", () => {
  beforeEach(() => { MockWebSocket.instances = []; });

  it("dispatches snapshot event to subscribed listener", async () => {
    const source = createWebSocketSource("ws://localhost/ws", MockWebSocket as any);
    const events: DataSetEvent[] = [];
    const def = { uuid: dataSetId("chat"), url: "ws://localhost/ws?dataset=messages" };

    source.subscribe(dataSetId("chat"), def as any, (e) => events.push(e));

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;

    ws.onmessage?.({ data: JSON.stringify({
      dataset: "messages",
      type: "snapshot",
      columns: [{ id: "text", type: "TEXT" }],
      rows: [["hello"]],
    })});

    expect(events).toHaveLength(1);
    expect(events[0]!.type).toBe("snapshot");
  });

  it("sends subscribe message on subscribe", async () => {
    const source = createWebSocketSource("ws://localhost/ws", MockWebSocket as any);
    const def = { uuid: dataSetId("chat"), url: "ws://localhost/ws?dataset=messages" };

    source.subscribe(dataSetId("chat"), def as any, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;

    expect(ws.sent).toContainEqual(JSON.stringify({ type: "subscribe", dataset: "messages" }));
  });

  it("ignores events for unsubscribed datasets", async () => {
    const source = createWebSocketSource("ws://localhost/ws", MockWebSocket as any);
    const events: DataSetEvent[] = [];
    const def = { uuid: dataSetId("chat"), url: "ws://localhost/ws?dataset=messages" };

    source.subscribe(dataSetId("chat"), def as any, (e) => events.push(e));

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;

    ws.onmessage?.({ data: JSON.stringify({
      dataset: "other",
      type: "snapshot",
      columns: [{ id: "x", type: "TEXT" }],
      rows: [["ignored"]],
    })});

    expect(events).toHaveLength(0);
  });

  it("closes connection when last subscriber unsubscribes", async () => {
    const source = createWebSocketSource("ws://localhost/ws", MockWebSocket as any);
    const def = { uuid: dataSetId("chat"), url: "ws://localhost/ws?dataset=messages" };

    source.subscribe(dataSetId("chat"), def as any, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));

    source.unsubscribe(dataSetId("chat"));
    expect(MockWebSocket.instances[0]!.readyState).toBe(3);
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement WebSocketSource**

Create `websocket-source.ts` with:
- Wire-name extraction from URL query parameter `?dataset=`
- Bidirectional mapping: wire name ↔ DataSetId
- Message parsing (single event + batch array)
- Event tabulation using existing `toTypedDataSet()`
- Connection lifecycle with exponential backoff reconnection
- Close code handling per spec (1000 = no reconnect, 1001/1006/1011 = reconnect, 4000+ = no reconnect)
- Malformed message handling (log + skip)

- [ ] **Step 4: Add more tests — reconnection, malformed messages, close codes**

Add tests for:
- Malformed JSON → no crash, no event dispatched
- Server close with code 1000 → no reconnection
- Server close with code 1006 → exponential backoff reconnection
- Batch message array → multiple events dispatched
- Replace/remove events populate keyColumn from stored def

- [ ] **Step 5: Run full suite**

- [ ] **Step 6: Commit**

```
feat: WebSocketSource — push event dispatch with reconnection

Parses incoming WebSocket JSON, tabulates rows, dispatches typed
DataSetEvents to per-dataset listeners. Exponential backoff reconnect
with close-code awareness. Wire-name routing for multiplexing.

Refs #52
```

---

### Task 6: WebSocketPool + Pipeline Integration (#52, #53)

**Files:**
- Create: `packages/pages-data/src/dataset/external/sources/websocket-pool.ts`
- Create: `packages/pages-data/src/dataset/external/sources/websocket-pool.test.ts`
- Modify: `packages/pages-runtime/src/data-pipeline.ts` — source routing for ws:// URLs
- Modify: `packages/pages-runtime/src/site.ts` — pool creation and disposal
- Modify: `packages/pages-data/src/dataset/external/index.ts` — exports

**Interfaces:**
- Consumes: `WebSocketSource` from Task 5; `DataSetManager.apply()` from Task 1
- Produces: `WebSocketPool` interface with `acquire()`, `releaseAll()`; `createWebSocketPool()` factory

- [ ] **Step 1: Write failing tests for WebSocketPool**

```typescript
import { describe, it, expect } from "vitest";
import { createWebSocketPool } from "./websocket-pool.js";

describe("WebSocketPool", () => {
  it("returns same source for same base URL", () => {
    const pool = createWebSocketPool(MockWebSocket as any);
    const def1 = { uuid: "d1", url: "ws://host/ws?dataset=a" };
    const def2 = { uuid: "d2", url: "ws://host/ws?dataset=b" };

    const s1 = pool.acquire("ws://host/ws", def1 as any);
    const s2 = pool.acquire("ws://host/ws", def2 as any);
    expect(s1).toBe(s2);
  });

  it("returns different sources for different base URLs", () => {
    const pool = createWebSocketPool(MockWebSocket as any);
    const def1 = { uuid: "d1", url: "ws://host/ws1" };
    const def2 = { uuid: "d2", url: "ws://host/ws2" };

    const s1 = pool.acquire("ws://host/ws1", def1 as any);
    const s2 = pool.acquire("ws://host/ws2", def2 as any);
    expect(s1).not.toBe(s2);
  });

  it("releaseAll closes all sources", () => {
    const pool = createWebSocketPool(MockWebSocket as any);
    const def = { uuid: "d1", url: "ws://host/ws" };
    pool.acquire("ws://host/ws", def as any);
    pool.releaseAll();
    // Verify connection closed
  });
});
```

- [ ] **Step 2: Implement WebSocketPool**

```typescript
import type { ExternalDataSetDef } from "../types.js";
import { createWebSocketSource, type WebSocketSource } from "./websocket-source.js";

export interface WebSocketPool {
  acquire(baseUrl: string, def: ExternalDataSetDef): WebSocketSource;
  releaseAll(): void;
}

export function createWebSocketPool(
  WS: typeof WebSocket = WebSocket,
): WebSocketPool {
  const sources = new Map<string, WebSocketSource>();

  return {
    acquire(baseUrl: string, def: ExternalDataSetDef): WebSocketSource {
      let source = sources.get(baseUrl);
      if (source === undefined) {
        source = createWebSocketSource(baseUrl, WS);
        sources.set(baseUrl, source);
      }
      return source;
    },

    releaseAll(): void {
      for (const source of sources.values()) {
        source.close();
      }
      sources.clear();
    },
  };
}
```

- [ ] **Step 3: Integrate into data-pipeline.ts — source routing**

In `handleDataRequest`, add WebSocket detection before the existing fetch path:

```typescript
// Route by source type
if (def.url?.startsWith("ws://") || def.url?.startsWith("wss://")) {
  const baseUrl = new URL(def.url);
  baseUrl.search = "";
  const source = pool.acquire(baseUrl.toString(), def);
  source.subscribe(dataSetId, def, (event) => {
    manager.apply(dataSetId, event);
    for (const [compId, entry] of registry) {
      if (entry.lookup.dataSetId === dataSetId) {
        pushData(entry.target, entry.lookup, entry.pagePath, entry.filterGroup, compId);
      }
    }
  });
  return;
}
```

The `pool` variable needs to be created in the pipeline factory and exposed for disposal.

- [ ] **Step 4: Integrate into site.ts — pool lifecycle**

In `loadSite()`, create the pool and pass it to the pipeline. In `dispose()`, add `pool.releaseAll()`.

- [ ] **Step 5: Export from index.ts**

Add to `packages/pages-data/src/dataset/external/index.ts`:
```typescript
export { createWebSocketSource, type WebSocketSource } from "./sources/websocket-source.js";
export { createWebSocketPool, type WebSocketPool } from "./sources/websocket-pool.js";
```

- [ ] **Step 6: Run full suite + typecheck**

Run: `yarn typecheck && yarn workspace @casehubio/pages-data run test -- --run && yarn workspace @casehubio/pages-runtime run test -- --run`

- [ ] **Step 7: Commit**

```
feat: WebSocketPool and pipeline integration

Connection pool keyed by base URL for multiplexing. Pipeline routes
ws:// URLs to pool.acquire → subscribe → apply events. Site disposes
pool on cleanup.

Closes #52, Closes #53
```

---

### Task 7: Accumulate Flag Example Update

**Files:**
- Modify: `examples/dashboards/Basic Usage/Accumulate Flag.ts`
- Modify: `examples/dashboards/Basic Usage/Accumulate Flag.dash.yaml`

**Interfaces:**
- Consumes: Schema changes from Task 3 (refreshTime on content+expression+accumulate)

- [ ] **Step 1: Update Accumulate Flag.ts**

Move the refresh responsibility from the component to the dataset definition:

```typescript
inlineDataset("products", productsData, {
  accumulate: true,
  cacheMaxRows: 20,
  refreshTime: "1second",
  expression: '[["test", $now() ~> $toMillis(), ($random() * 100)]]',
});

export default page(
  timeseries({
    resizable: true,
    // Remove: refresh: { interval: 1 },
    lookup: createLookup("products", []),
  })
);
```

- [ ] **Step 2: Update Accumulate Flag.dash.yaml**

Add `refreshTime: 1second` to the dataset definition. Remove `refresh.interval` from the displayer props.

- [ ] **Step 3: Commit**

```
chore: update Accumulate Flag example — dataset-level refresh

Move refresh from component to dataset definition using the new
content + expression + accumulate + refreshTime pipeline.

Refs #36
```

---

### Task 8: NAV_TYPE_MAP Fix + Navigation Gallery (#33)

**Files:**
- Modify: `packages/pages-ui/src/parser/component-desugar.ts` — add ACCORDION, APP_GRID to NAV_TYPE_MAP
- Test: `packages/pages-ui/src/parser/component-desugar.test.ts`
- Modify: `examples/dashboards/Basic Usage/Navigation Rebinding.dash.yml` — add accordion, appGrid views
- Modify: `examples/dashboards/Basic Usage/Navigation Rebinding.ts` — add builder calls

**Interfaces:**
- Produces: YAML support for `navType: ACCORDION` and `navType: APP_GRID`

- [ ] **Step 1: Write failing tests for ACCORDION and APP_GRID desugaring**

In `component-desugar.test.ts`, find the nav type test section and add:

```typescript
it("desugars navType: ACCORDION", () => {
  const result = desugarComponent({ navType: "ACCORDION", groupId: "g1" });
  expect(result.type).toBe("accordion");
});

it("desugars navType: APP_GRID", () => {
  const result = desugarComponent({ navType: "APP_GRID", groupId: "g1" });
  expect(result.type).toBe("app-grid");
});
```

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Add entries to NAV_TYPE_MAP**

In `component-desugar.ts`, find `NAV_TYPE_MAP` and add:

```typescript
ACCORDION: "accordion",
APP_GRID: "app-grid",
```

- [ ] **Step 4: Run test — verify it passes**

- [ ] **Step 5: Update Navigation Rebinding example**

Add Accordion View and App Grid View sections to both `.dash.yml` and `.ts` files, following the existing pattern of switching navigation container type while keeping the same content pages.

- [ ] **Step 6: Commit**

```
feat: ACCORDION + APP_GRID YAML support and navigation gallery

Add missing nav types to NAV_TYPE_MAP. Extend Navigation Rebinding
example with accordion and app-grid views.

Closes #33
```

---

### Task 9: Team Management Example — withAccess, JOIN, Includes (#34)

**Files:**
- Create: `examples/dashboards/Basic Usage/Team Management.dash.yaml`
- Create: `examples/dashboards/Basic Usage/includes/team-detail.dash.yml`
- Create: `examples/dashboards/Basic Usage/includes/admin-settings.dash.yml`
- Create: `examples/dashboards/Basic Usage/Team Management.ts`

**Interfaces:**
- Consumes: Existing `withAccess()`, `join()` DSL builders; existing YAML `src:` includes support

- [ ] **Step 1: Create main dashboard YAML**

`Team Management.dash.yaml` with:
- Inline dataset: team, member, role, email, department columns (10-15 rows across 3 teams)
- Overview page: table grouped by team with JOIN on member names and COUNT
- Team Detail page: included via `src: includes/team-detail.dash.yml`
- Admin Settings page: included via `src: includes/admin-settings.dash.yml`, with access control (`roles: [admin]`)

- [ ] **Step 2: Create included pages**

`includes/team-detail.dash.yml` — table showing all members with role and email columns.

`includes/admin-settings.dash.yml` — simple settings page with a few metric cards.

- [ ] **Step 3: Create TypeScript companion**

`Team Management.ts` using DSL builders:
- `inlineDataset()` for the team data
- `withAccess({ roles: ["admin"] }, component)` for access-gated components
- `join("member", ", ")` for the JOIN aggregation
- Standard page/table/metric builders for layout

- [ ] **Step 4: Verify example renders**

Run: `yarn build:examples` and check the gallery includes the new example.

- [ ] **Step 5: Commit**

```
feat: Team Management example — withAccess, JOIN, YAML includes

Demonstrates role-based access control, JOIN aggregation for
concatenating grouped values, and modular dashboard organization
with included sub-pages.

Closes #34
```

---

## Dependency Graph

```
Task 1 (events + apply)
  ├── Task 2 (caller migration) ──┐
  │                                ├── Task 4 (generator + pipeline #36) ── Task 7 (example update)
  │   Task 3 (schema) ────────────┘
  │
  └── Task 5 (WebSocketSource) ── Task 6 (pool + pipeline #52/#53)

Task 8 (nav types #33) — independent
Task 9 (team mgmt #34) — independent
```

Tasks 8 and 9 are independent of Tasks 1-7 and can be done in parallel.
