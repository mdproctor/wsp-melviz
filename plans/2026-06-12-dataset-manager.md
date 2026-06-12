# DataSetManager Service Layer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement DataSetManager — dataset registration, lookup execution pipeline (resolve filters → applyOps → paginate), and the `applyOps` referenceDate threading fix.

**Architecture:** DataSetManager is a service class with a `Map<DataSetId, TypedDataSet>` registry and a `lookup()` method that runs the full pipeline: validate → resolve dataset → resolve filters → execute ops → paginate. A factory function `createDataSetManager()` is the public construction path. One prerequisite change: extend `applyOps` to thread `referenceDate` to `applyFilter`.

**Tech Stack:** TypeScript, vitest

**Spec:** `docs/superpowers/specs/2026-06-12-dataset-manager-design.md`

---

### Task 1: Extend `applyOps` to thread `referenceDate`

**Files:**
- Modify: `packages/core/src/dataset/ops.ts`
- Modify: `packages/core/src/dataset/ops.test.ts`

This is a prerequisite for the manager. `applyOps` currently calls `applyFilter(current, op)` without forwarding the `referenceDate` parameter that `applyFilter` already accepts. TIME_FRAME filters always evaluate against `new Date()`, making tests non-deterministic.

- [ ] **Step 1: Write the failing test in `ops.test.ts`**

Add at the end of the `applyOps` describe block:

```typescript
it("forwards referenceDate to filter for TIME_FRAME evaluation", () => {
  const refDate = new Date(Date.UTC(2024, 5, 15)); // June 15, 2024
  const timeFrame = parseTimeFrame("begin[year] till end[year]");

  const ds = toTypedDataSet({
    columns: [
      col("date", "Date", ColumnType.DATE),
      col("label", "Label", ColumnType.LABEL),
    ],
    data: [
      ["2024-03-01T00:00:00.000Z", "in-range"],
      ["2023-06-01T00:00:00.000Z", "out-of-range"],
      ["2024-11-01T00:00:00.000Z", "in-range"],
    ],
  });

  const filter: ResolvedFilterOp = {
    type: "filter",
    expressions: [{
      type: "date",
      columnId: "date" as ColumnId,
      filter: { fn: "TIME_FRAME", timeFrame },
    }],
  };

  const result = applyOps(ds, [filter], { referenceDate: refDate });
  expect(result.rows).toHaveLength(2);
  expect(result.rows[0]!.text("label" as ColumnId)).toBe("in-range");
  expect(result.rows[1]!.text("label" as ColumnId)).toBe("in-range");
});
```

Add the missing import at the top of the file:

```typescript
import { parseTimeFrame } from "./timeframe.js";
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @melviz/core run test -- ops.test.ts`
Expected: FAIL — `applyOps` does not accept a third argument (TypeScript compile error or ignored parameter)

- [ ] **Step 3: Add `ApplyOpsOptions` and update `applyOps` signature in `ops.ts`**

Add the interface after the existing type exports:

```typescript
export interface ApplyOpsOptions {
  readonly referenceDate?: Date;
}
```

Change the `applyOps` function signature and the filter branch:

```typescript
export function applyOps(
  ds: TypedDataSet,
  ops: readonly ResolvedDataSetOp[],
  options?: ApplyOpsOptions,
): TypedDataSet {
  validateOpOrder(ops);

  let current = ds;
  let i = 0;

  while (i < ops.length) {
    const op = ops[i]!;

    if (op.type === "filter") {
      current = applyFilter(current, op as ResolvedFilterOp, options?.referenceDate);
      i++;
    } else if (op.type === "group") {
      // Collect consecutive GroupOps for deferred materialisation
      const groupOps: GroupOp[] = [];
      while (i < ops.length && ops[i]!.type === "group") {
        groupOps.push(ops[i]! as GroupOp);
        i++;
      }
      current = groupOps.length === 1
        ? applyGroup(current, groupOps[0]!)
        : applyGroupSequence(current, groupOps);
    } else if (op.type === "sort") {
      current = applySort(current, op);
      i++;
    }
  }

  return current;
}
```

The only change is: the function now takes an optional third parameter, and line `applyFilter(current, op as ResolvedFilterOp)` becomes `applyFilter(current, op as ResolvedFilterOp, options?.referenceDate)`.

- [ ] **Step 4: Run tests to verify all pass**

Run: `yarn workspace @melviz/core run test -- ops.test.ts`
Expected: ALL PASS (new test + all existing tests unchanged)

- [ ] **Step 5: Run full test suite to confirm no regressions**

Run: `yarn workspace @melviz/core run test`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add packages/core/src/dataset/ops.ts packages/core/src/dataset/ops.test.ts
git commit -m "feat: thread referenceDate through applyOps to applyFilter  Refs #7"
```

---

### Task 2: DataSetManager — registry operations (TDD)

**Files:**
- Create: `packages/core/src/dataset/manager.ts`
- Create: `packages/core/src/dataset/manager.test.ts`

Build the registry (register/get/remove/has) and factory function. Lookup comes in Task 3.

- [ ] **Step 1: Write the registry tests in `manager.test.ts`**

```typescript
import { describe, it, expect } from "vitest";
import { createDataSetManager } from "./manager.js";
import { toTypedDataSet } from "./conversion.js";
import type { Column, ColumnId, DataSetId } from "./types.js";
import { ColumnType } from "./types.js";

function col(id: string, name: string, type: ColumnType): Column {
  return { id: id as ColumnId, name, type };
}

function testDataSet(rows: (string | null)[][]) {
  return toTypedDataSet({
    columns: [
      col("name", "Name", ColumnType.LABEL),
      col("amount", "Amount", ColumnType.NUMBER),
    ],
    data: rows,
  });
}

const ID_A = "dataset-a" as DataSetId;
const ID_B = "dataset-b" as DataSetId;
const ID_UNKNOWN = "does-not-exist" as DataSetId;

describe("DataSetManager — registry", () => {
  it("register + get returns the same dataset", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.register(ID_A, ds);
    expect(mgr.get(ID_A)).toBe(ds);
  });

  it("register overwrites existing dataset with same ID", () => {
    const mgr = createDataSetManager();
    const ds1 = testDataSet([["Alice", "100"]]);
    const ds2 = testDataSet([["Bob", "200"]]);
    mgr.register(ID_A, ds1);
    mgr.register(ID_A, ds2);
    expect(mgr.get(ID_A)).toBe(ds2);
  });

  it("get returns undefined for unknown ID", () => {
    const mgr = createDataSetManager();
    expect(mgr.get(ID_UNKNOWN)).toBeUndefined();
  });

  it("has returns true for registered ID", () => {
    const mgr = createDataSetManager();
    mgr.register(ID_A, testDataSet([["Alice", "100"]]));
    expect(mgr.has(ID_A)).toBe(true);
  });

  it("has returns false for unknown ID", () => {
    const mgr = createDataSetManager();
    expect(mgr.has(ID_UNKNOWN)).toBe(false);
  });

  it("remove returns true and deletes registered dataset", () => {
    const mgr = createDataSetManager();
    mgr.register(ID_A, testDataSet([["Alice", "100"]]));
    expect(mgr.remove(ID_A)).toBe(true);
    expect(mgr.get(ID_A)).toBeUndefined();
  });

  it("remove returns false for unknown ID", () => {
    const mgr = createDataSetManager();
    expect(mgr.remove(ID_UNKNOWN)).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- manager.test.ts`
Expected: FAIL — `createDataSetManager` does not exist

- [ ] **Step 3: Implement registry and factory in `manager.ts`**

```typescript
import type { DataSetId, TypedDataSet, Column } from "./types.js";
import type { DataSetLookup } from "./lookup.js";
import type { DataSetOp, ResolvedDataSetOp } from "./ops.js";
import { applyOps } from "./ops.js";
import { resolveFilterTypes } from "./filter-resolve.js";
import { DataSetError } from "./errors.js";

export interface LookupOptions {
  readonly rowOffset?: number;
  readonly rowCount?: number;
  readonly referenceDate?: Date;
}

export interface DataSetManager {
  register(id: DataSetId, dataset: TypedDataSet): void;
  get(id: DataSetId): TypedDataSet | undefined;
  remove(id: DataSetId): boolean;
  has(id: DataSetId): boolean;
  lookup(query: DataSetLookup, options?: LookupOptions): TypedDataSet;
}

function resolveOps(
  ops: readonly DataSetOp[],
  columns: readonly Column[],
): ResolvedDataSetOp[] {
  return ops.map(op => {
    if (op.type !== "filter") return op;
    return {
      type: "filter" as const,
      expressions: op.expressions.map(expr => resolveFilterTypes(expr, columns)),
    };
  });
}

function paginate(
  ds: TypedDataSet,
  offset: number,
  count: number,
): TypedDataSet {
  if (offset === 0 && count < 0) return ds;
  const start = Math.min(offset, ds.rows.length);
  const rows = count < 0
    ? ds.rows.slice(start)
    : ds.rows.slice(start, start + count);
  return { columns: ds.columns, rows };
}

class DataSetManagerImpl implements DataSetManager {
  private readonly datasets = new Map<DataSetId, TypedDataSet>();

  register(id: DataSetId, dataset: TypedDataSet): void {
    this.datasets.set(id, dataset);
  }

  get(id: DataSetId): TypedDataSet | undefined {
    return this.datasets.get(id);
  }

  remove(id: DataSetId): boolean {
    return this.datasets.delete(id);
  }

  has(id: DataSetId): boolean {
    return this.datasets.has(id);
  }

  lookup(query: DataSetLookup, options?: LookupOptions): TypedDataSet {
    const offset = options?.rowOffset ?? 0;
    if (offset < 0) {
      throw new DataSetError("INVALID_OPERATION", `rowOffset cannot be negative: ${offset}`);
    }

    const dataset = this.datasets.get(query.dataSetId);
    if (!dataset) {
      throw new DataSetError("UNKNOWN_PROVIDER", `Dataset "${query.dataSetId}" not registered`);
    }

    const resolvedOps = resolveOps(query.operations, dataset.columns);
    const result = applyOps(dataset, resolvedOps, { referenceDate: options?.referenceDate });
    return paginate(result, offset, options?.rowCount ?? -1);
  }
}

export function createDataSetManager(): DataSetManager {
  return new DataSetManagerImpl();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- manager.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/core/src/dataset/manager.ts packages/core/src/dataset/manager.test.ts
git commit -m "feat: DataSetManager registry — register, get, remove, has  Refs #7"
```

---

### Task 3: DataSetManager — lookup pipeline tests (TDD)

**Files:**
- Modify: `packages/core/src/dataset/manager.test.ts`

The implementation already exists from Task 2. This task adds lookup tests exercising the full pipeline: no-ops, resolved filters, unresolved filters, group, sort, full pipeline, and TIME_FRAME with referenceDate.

- [ ] **Step 1: Add lookup pipeline tests to `manager.test.ts`**

Append after the registry describe block:

```typescript
import { createLookup } from "./lookup.js";
import type { ResolvedFilterOp, FilterOp } from "./filter.js";
import type { GroupOp } from "./group.js";
import type { SortOp } from "./sort.js";
import { parseTimeFrame } from "./timeframe.js";

function salesDataSet() {
  return toTypedDataSet({
    columns: [
      col("dept", "Department", ColumnType.LABEL),
      col("revenue", "Revenue", ColumnType.NUMBER),
      col("date", "Date", ColumnType.DATE),
    ],
    data: [
      ["Sales", "100", "2024-01-15T00:00:00.000Z"],
      ["Engineering", "200", "2024-04-01T00:00:00.000Z"],
      ["Sales", "150", "2024-07-01T00:00:00.000Z"],
      ["Marketing", "50", "2023-06-01T00:00:00.000Z"],
      ["Engineering", "300", "2024-10-01T00:00:00.000Z"],
    ],
  });
}

const SALES_ID = "sales" as DataSetId;

describe("DataSetManager — lookup pipeline", () => {
  it("no operations returns full dataset unchanged", () => {
    const mgr = createDataSetManager();
    const ds = salesDataSet();
    mgr.register(SALES_ID, ds);
    const result = mgr.lookup(createLookup(SALES_ID, []));
    expect(result).toBe(ds);
  });

  it("resolved filter ops applied correctly", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const filter: ResolvedFilterOp = {
      type: "filter",
      expressions: [{
        type: "numeric",
        columnId: "revenue" as ColumnId,
        filter: { fn: "GREATER_THAN", value: 100 },
      }],
    };
    const result = mgr.lookup(createLookup(SALES_ID, [filter]));
    expect(result.rows).toHaveLength(3);
  });

  it("unresolved filter ops resolved against column schema then applied", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const filter: FilterOp = {
      type: "filter",
      expressions: [{
        type: "unresolved",
        columnId: "revenue" as ColumnId,
        fn: "GREATER_THAN",
        args: ["100"],
      }],
    };
    const result = mgr.lookup(createLookup(SALES_ID, [filter]));
    expect(result.rows).toHaveLength(3);
  });

  it("group ops applied correctly", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const group: GroupOp = {
      type: "group",
      groupingKey: {
        sourceId: "dept" as ColumnId,
        columnId: "dept" as ColumnId,
        strategy: { mode: "distinct" },
        maxIntervals: 15,
        emptyIntervals: false,
        ascendingOrder: true,
      },
      columns: [
        { kind: "key", sourceId: "dept" as ColumnId, columnId: "dept" as ColumnId },
        { kind: "aggregate", sourceId: "revenue" as ColumnId, columnId: "total" as ColumnId, fn: { fn: "SUM" } },
      ],
    };
    const result = mgr.lookup(createLookup(SALES_ID, [group]));
    expect(result.rows).toHaveLength(3);
  });

  it("sort ops applied correctly", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const sort: SortOp = {
      type: "sort",
      columns: [{ columnId: "revenue" as ColumnId, order: "DESCENDING" }],
    };
    const result = mgr.lookup(createLookup(SALES_ID, [sort]));
    expect(result.rows[0]!.number("revenue" as ColumnId)).toBe(300);
    expect(result.rows[4]!.number("revenue" as ColumnId)).toBe(50);
  });

  it("filter + group + sort full pipeline", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const filter: ResolvedFilterOp = {
      type: "filter",
      expressions: [{
        type: "numeric",
        columnId: "revenue" as ColumnId,
        filter: { fn: "GREATER_OR_EQUALS_TO", value: 100 },
      }],
    };
    const group: GroupOp = {
      type: "group",
      groupingKey: {
        sourceId: "dept" as ColumnId,
        columnId: "dept" as ColumnId,
        strategy: { mode: "distinct" },
        maxIntervals: 15,
        emptyIntervals: false,
        ascendingOrder: true,
      },
      columns: [
        { kind: "key", sourceId: "dept" as ColumnId, columnId: "dept" as ColumnId },
        { kind: "aggregate", sourceId: "revenue" as ColumnId, columnId: "total" as ColumnId, fn: { fn: "SUM" } },
      ],
    };
    const sort: SortOp = {
      type: "sort",
      columns: [{ columnId: "total" as ColumnId, order: "DESCENDING" }],
    };
    const result = mgr.lookup(createLookup(SALES_ID, [filter, group, sort]));
    expect(result.rows).toHaveLength(2);
    expect(result.rows[0]!.text("dept" as ColumnId)).toBe("Engineering");
    expect(result.rows[0]!.number("total" as ColumnId)).toBe(500);
    expect(result.rows[1]!.text("dept" as ColumnId)).toBe("Sales");
    expect(result.rows[1]!.number("total" as ColumnId)).toBe(250);
  });

  it("TIME_FRAME filter with explicit referenceDate — deterministic", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const timeFrame = parseTimeFrame("begin[year] till end[year]");
    const filter: ResolvedFilterOp = {
      type: "filter",
      expressions: [{
        type: "date",
        columnId: "date" as ColumnId,
        filter: { fn: "TIME_FRAME", timeFrame },
      }],
    };
    const refDate = new Date(Date.UTC(2024, 5, 1));
    const result = mgr.lookup(
      createLookup(SALES_ID, [filter]),
      { referenceDate: refDate },
    );
    expect(result.rows).toHaveLength(4);
  });
});
```

Note: the imports for `createLookup`, `ResolvedFilterOp`, `FilterOp`, `GroupOp`, `SortOp`, and `parseTimeFrame` must be added to the top of the file alongside the existing imports.

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- manager.test.ts`
Expected: ALL PASS (implementation already in place from Task 2)

- [ ] **Step 3: Commit**

```bash
git add packages/core/src/dataset/manager.test.ts
git commit -m "test: DataSetManager lookup pipeline — filter resolve, group, sort, TIME_FRAME  Refs #7"
```

---

### Task 4: DataSetManager — pagination tests (TDD)

**Files:**
- Modify: `packages/core/src/dataset/manager.test.ts`

Pagination is already implemented. This task adds comprehensive edge-case tests.

- [ ] **Step 1: Add pagination tests to `manager.test.ts`**

Append after the lookup pipeline describe block:

```typescript
describe("DataSetManager — pagination", () => {
  function setupManager() {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    return mgr;
  }

  it("no options returns all rows", () => {
    const result = setupManager().lookup(createLookup(SALES_ID, []));
    expect(result.rows).toHaveLength(5);
  });

  it("explicit defaults return all rows", () => {
    const result = setupManager().lookup(
      createLookup(SALES_ID, []),
      { rowOffset: 0, rowCount: -1 },
    );
    expect(result.rows).toHaveLength(5);
  });

  it("rowOffset + rowCount slices correctly", () => {
    const result = setupManager().lookup(
      createLookup(SALES_ID, []),
      { rowOffset: 1, rowCount: 2 },
    );
    expect(result.rows).toHaveLength(2);
    expect(result.rows[0]!.number("revenue" as ColumnId)).toBe(200);
    expect(result.rows[1]!.number("revenue" as ColumnId)).toBe(150);
  });

  it("rowCount: 0 returns zero rows", () => {
    const result = setupManager().lookup(
      createLookup(SALES_ID, []),
      { rowOffset: 0, rowCount: 0 },
    );
    expect(result.rows).toHaveLength(0);
  });

  it("rowOffset beyond dataset length returns zero rows", () => {
    const result = setupManager().lookup(
      createLookup(SALES_ID, []),
      { rowOffset: 100, rowCount: 10 },
    );
    expect(result.rows).toHaveLength(0);
  });

  it("rowCount: -1 with offset returns all rows from offset", () => {
    const result = setupManager().lookup(
      createLookup(SALES_ID, []),
      { rowOffset: 3, rowCount: -1 },
    );
    expect(result.rows).toHaveLength(2);
  });

  it("pagination applies after ops", () => {
    const mgr = setupManager();
    const sort: SortOp = {
      type: "sort",
      columns: [{ columnId: "revenue" as ColumnId, order: "ASCENDING" }],
    };
    const result = mgr.lookup(
      createLookup(SALES_ID, [sort]),
      { rowOffset: 0, rowCount: 2 },
    );
    expect(result.rows).toHaveLength(2);
    expect(result.rows[0]!.number("revenue" as ColumnId)).toBe(50);
    expect(result.rows[1]!.number("revenue" as ColumnId)).toBe(100);
  });
});
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- manager.test.ts`
Expected: ALL PASS

- [ ] **Step 3: Commit**

```bash
git add packages/core/src/dataset/manager.test.ts
git commit -m "test: DataSetManager pagination — offset, count, zero, beyond-length, post-ops  Refs #7"
```

---

### Task 5: DataSetManager — error path tests (TDD)

**Files:**
- Modify: `packages/core/src/dataset/manager.test.ts`

All error paths are already implemented. This task adds tests confirming each one.

- [ ] **Step 1: Add error path tests to `manager.test.ts`**

Add the `DataSetError` import at the top:

```typescript
import { DataSetError } from "./errors.js";
```

Append after the pagination describe block:

```typescript
describe("DataSetManager — error paths", () => {
  it("unknown dataset ID throws UNKNOWN_PROVIDER", () => {
    const mgr = createDataSetManager();
    expect(() => mgr.lookup(createLookup(ID_UNKNOWN, []))).toThrow(DataSetError);
    expect(() => mgr.lookup(createLookup(ID_UNKNOWN, []))).toThrow("UNKNOWN_PROVIDER");
  });

  it("filter referencing unknown column throws UNKNOWN_COLUMN", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const filter: FilterOp = {
      type: "filter",
      expressions: [{
        type: "unresolved",
        columnId: "nonexistent" as ColumnId,
        fn: "EQUALS_TO",
        args: ["x"],
      }],
    };
    expect(() => mgr.lookup(createLookup(SALES_ID, [filter]))).toThrow("UNKNOWN_COLUMN");
  });

  it("invalid function/type combo throws RESOLUTION_FAILED", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const filter: FilterOp = {
      type: "filter",
      expressions: [{
        type: "unresolved",
        columnId: "revenue" as ColumnId,
        fn: "LIKE_TO",
        args: ["%test%"],
      }],
    };
    expect(() => mgr.lookup(createLookup(SALES_ID, [filter]))).toThrow("RESOLUTION_FAILED");
  });

  it("negative rowOffset throws INVALID_OPERATION", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    expect(() => mgr.lookup(
      createLookup(SALES_ID, []),
      { rowOffset: -1 },
    )).toThrow("INVALID_OPERATION");
  });

  it("raw-object DataSetLookup with invalid op order throws INVALID_OPERATION", () => {
    const mgr = createDataSetManager();
    mgr.register(SALES_ID, salesDataSet());
    const sort: SortOp = { type: "sort", columns: [{ columnId: "revenue" as ColumnId, order: "ASCENDING" }] };
    const group: GroupOp = { type: "group", groupingKey: null, columns: [] };
    const rawLookup = { dataSetId: SALES_ID, operations: [sort, group] } as const;
    expect(() => mgr.lookup(rawLookup)).toThrow("INVALID_OPERATION");
  });
});
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- manager.test.ts`
Expected: ALL PASS

- [ ] **Step 3: Run full test suite**

Run: `yarn workspace @melviz/core run test`
Expected: ALL PASS — no regressions anywhere

- [ ] **Step 4: Commit**

```bash
git add packages/core/src/dataset/manager.test.ts
git commit -m "test: DataSetManager error paths — unknown ID, bad filter, negative offset, raw lookup  Refs #7"
```

---

### Task 6: Final verification and cleanup

**Files:**
- Verify: all files in `packages/core/src/dataset/`

- [ ] **Step 1: Run the full test suite**

Run: `yarn workspace @melviz/core run test`
Expected: ALL PASS

- [ ] **Step 2: Run TypeScript compilation check**

Run: `yarn workspace @melviz/core run build`
Expected: No errors — confirms all type imports and exports are correct

- [ ] **Step 3: Verify test count**

Run: `yarn workspace @melviz/core run test -- --reporter=verbose 2>&1 | tail -5`
Expected: Manager tests add ~20 new tests. Total should be ~426+ (406 prior + 20 new).

- [ ] **Step 4: Commit the spec update if not already committed**

The spec at `docs/superpowers/specs/2026-06-12-dataset-manager-design.md` was already committed during brainstorming. Verify it's up to date:

```bash
git -C /Users/mdproctor/claude/melviz status docs/superpowers/specs/
```

If there are uncommitted changes, commit them.
