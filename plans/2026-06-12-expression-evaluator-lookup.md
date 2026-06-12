# Expression Evaluator & DataSetLookup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the JSONata expression bridge, DataSetLookup query model, and YAML parser — covering issues #4 and #5.

**Architecture:** Two new modules (`expression/` for JSONata bridge + evaluator, `dataset/` extensions for lookup + parser + resolution) with a foundational refactor of FilterExpression into a parameterized tree type that separates resolved and unresolved filter leaves at the type level.

**Tech Stack:** TypeScript 5.6+, Vitest, JSONata 2.x, Zod 3.x

**Spec:** `docs/superpowers/specs/2026-06-12-expression-evaluator-lookup-design.md`

---

### Task 1: Refactor FilterExpression to parameterized tree

**Files:**
- Modify: `packages/core/src/dataset/filter.ts`
- Modify: `packages/core/src/dataset/filter-eval.ts`
- Modify: `packages/core/src/dataset/filter-eval.test.ts`
- Modify: `packages/core/src/dataset/ops.ts`
- Modify: `packages/core/src/dataset/ops.test.ts`

- [ ] **Step 1: Replace FilterExpression with parameterized FilterExprTree in filter.ts**

Replace the entire `FilterExpression` type definition and add new types. Keep `FilterOp` unchanged for now.

```typescript
// Add after the DateFilter type

export type FilterExprTree<Leaf> =
  | Leaf
  | { readonly type: "and"; readonly children: readonly FilterExprTree<Leaf>[] }
  | { readonly type: "or"; readonly children: readonly FilterExprTree<Leaf>[] }
  | { readonly type: "not"; readonly child: FilterExprTree<Leaf> };

export type ResolvedLeaf =
  | { readonly type: "numeric"; readonly columnId: ColumnId; readonly filter: NumericFilter }
  | { readonly type: "string"; readonly columnId: ColumnId; readonly filter: StringFilter }
  | { readonly type: "date"; readonly columnId: ColumnId; readonly filter: DateFilter };

export type UnresolvedLeaf = {
  readonly type: "unresolved";
  readonly columnId: ColumnId;
  readonly fn: CoreFunctionType;
  readonly args: readonly string[];
};

export type ResolvedFilterExpression = FilterExprTree<ResolvedLeaf>;
export type FilterExpression = FilterExprTree<ResolvedLeaf | UnresolvedLeaf>;

export interface ResolvedFilterOp {
  readonly type: "filter";
  readonly expressions: readonly ResolvedFilterExpression[];
}

export interface FilterOp {
  readonly type: "filter";
  readonly expressions: readonly FilterExpression[];
}
```

Remove the old `FilterExpression` type definition (the 6-variant union at lines 54-61) and the old `FilterOp` interface (lines 63-66). The new definitions replace them.

- [ ] **Step 2: Update filter-eval.ts to use ResolvedFilterOp**

Change the import and function signature:

```typescript
import type { ResolvedFilterOp, ResolvedFilterExpression, NumericFilter, StringFilter, DateFilter } from "./filter.js";
```

Change `applyFilter` signature:

```typescript
export function applyFilter(
  ds: TypedDataSet,
  op: ResolvedFilterOp,
  referenceDate?: Date,
): TypedDataSet {
```

Change internal `evaluateExpression` parameter type:

```typescript
function evaluateExpression(
  row: TypedRow,
  expr: ResolvedFilterExpression,
  resolved: ResolvedTimeFrames,
): boolean {
```

Change `preResolveTimeFrames` and `walkExpressions` to use `ResolvedFilterExpression`:

```typescript
function preResolveTimeFrames(
  expressions: readonly ResolvedFilterExpression[],
  referenceDate: Date,
): ResolvedTimeFrames {

function walkExpressions(
  expressions: readonly ResolvedFilterExpression[],
  visitor: (expr: ResolvedFilterExpression) => void,
): void {
```

The switch statement in `evaluateExpression` covers `numeric | string | date | and | or | not` — exhaustive over `ResolvedFilterExpression`. No `"unresolved"` case needed.

- [ ] **Step 3: Update ops.ts to use ResolvedDataSetOp**

Add `ResolvedDataSetOp` type and update `applyOps` signature:

```typescript
import type { FilterOp, ResolvedFilterOp } from "./filter.js";
import type { GroupOp } from "./group.js";
import type { SortOp } from "./sort.js";
import { DataSetError } from "./errors.js";
import type { TypedDataSet } from "./types.js";
import { applyFilter } from "./filter-eval.js";
import { applyGroup, applyGroupSequence } from "./group-eval.js";
import { applySort } from "./sort-eval.js";

export type DataSetOp = FilterOp | GroupOp | SortOp;
export type ResolvedDataSetOp = ResolvedFilterOp | GroupOp | SortOp;

export function validateOpOrder(ops: readonly DataSetOp[]): void {
  // unchanged
}

export function applyOps(
  ds: TypedDataSet,
  ops: readonly ResolvedDataSetOp[],
): TypedDataSet {
  validateOpOrder(ops);
  // rest unchanged — the cast from ResolvedFilterOp to FilterOp
  // in the group type guard is safe since ResolvedFilterOp is a subtype
```

In the `applyOps` body, update the type assertion in the group collection loop:

```typescript
    } else if (op.type === "group") {
      const groupOps: GroupOp[] = [];
      while (i < ops.length && ops[i]!.type === "group") {
        groupOps.push(ops[i]! as GroupOp);
        i++;
      }
```

And the filter branch:

```typescript
    if (op.type === "filter") {
      current = applyFilter(current, op as ResolvedFilterOp);
      i++;
```

- [ ] **Step 4: Update filter-eval.test.ts helper return types**

Change the helper functions to return `ResolvedFilterExpression`:

```typescript
import type { ResolvedFilterExpression, NumericFilter, StringFilter, DateFilter } from "./filter.js";

function nf(filter: NumericFilter): ResolvedFilterExpression {
  return { type: "numeric", columnId: "val" as ColumnId, filter };
}

function sf(filter: StringFilter): ResolvedFilterExpression {
  return { type: "string", columnId: "name" as ColumnId, filter };
}

function df(filter: DateFilter): ResolvedFilterExpression {
  return { type: "date", columnId: "date" as ColumnId, filter };
}
```

The inline `{ type: "filter", expressions: [...] }` objects passed to `applyFilter` will be inferred as `ResolvedFilterOp` since the helper functions now return `ResolvedFilterExpression`. No other test changes needed.

- [ ] **Step 5: Update ops.test.ts imports**

Update imports to use `ResolvedFilterOp`:

```typescript
import type { ResolvedFilterOp } from "./filter.js";
```

And any explicit `FilterOp` type annotations in the test file should become `ResolvedFilterOp`.

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @melviz/core run test`
Expected: All 550+ existing tests pass. No behavioral changes — only type signatures changed.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/filter.ts packages/core/src/dataset/filter-eval.ts packages/core/src/dataset/filter-eval.test.ts packages/core/src/dataset/ops.ts packages/core/src/dataset/ops.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "refactor: parameterized FilterExprTree with resolved/unresolved leaf types  Refs #4"
```

---

### Task 2: Add DateFilter IN/NOT_IN variants

**Files:**
- Modify: `packages/core/src/dataset/filter.ts`
- Modify: `packages/core/src/dataset/filter-eval.ts`
- Modify: `packages/core/src/dataset/filter-eval.test.ts`

- [ ] **Step 1: Write the failing tests**

Add to `filter-eval.test.ts`:

```typescript
function dateDataSet(): TypedDataSet {
  return toTypedDataSet({
    columns: [col("date", "Date", ColumnType.DATE)],
    data: [
      ["2024-01-15T00:00:00.000Z"],
      ["2024-06-01T00:00:00.000Z"],
      ["2024-09-20T00:00:00.000Z"],
    ],
  });
}

describe("applyFilter — date IN/NOT_IN", () => {
  const jan15 = new Date("2024-01-15T00:00:00.000Z");
  const jun01 = new Date("2024-06-01T00:00:00.000Z");
  const dec25 = new Date("2024-12-25T00:00:00.000Z");

  it("IN matches rows with dates in the set", () => {
    const result = applyFilter(dateDataSet(), {
      type: "filter",
      expressions: [df({ fn: "IN", values: [jan15, jun01] })],
    });
    expect(result.rows).toHaveLength(2);
  });

  it("IN with no matches returns empty", () => {
    const result = applyFilter(dateDataSet(), {
      type: "filter",
      expressions: [df({ fn: "IN", values: [dec25] })],
    });
    expect(result.rows).toHaveLength(0);
  });

  it("NOT_IN excludes rows with dates in the set", () => {
    const result = applyFilter(dateDataSet(), {
      type: "filter",
      expressions: [df({ fn: "NOT_IN", values: [jan15, jun01] })],
    });
    expect(result.rows).toHaveLength(1);
  });

  it("IN with NULL cell returns false", () => {
    const ds = toTypedDataSet({
      columns: [col("date", "Date", ColumnType.DATE)],
      data: [[null], ["2024-01-15T00:00:00.000Z"]],
    });
    const result = applyFilter(ds, {
      type: "filter",
      expressions: [df({ fn: "IN", values: [jan15] })],
    });
    expect(result.rows).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- filter-eval`
Expected: FAIL — DateFilter type doesn't include IN/NOT_IN variants.

- [ ] **Step 3: Add IN/NOT_IN to DateFilter type in filter.ts**

Add two variants to `DateFilter`:

```typescript
export type DateFilter =
  | NullFilter
  | { readonly fn: "EQUALS_TO"; readonly value: Date }
  | { readonly fn: "NOT_EQUALS_TO"; readonly value: Date }
  | { readonly fn: "GREATER_THAN"; readonly value: Date }
  | { readonly fn: "GREATER_OR_EQUALS_TO"; readonly value: Date }
  | { readonly fn: "LOWER_THAN"; readonly value: Date }
  | { readonly fn: "LOWER_OR_EQUALS_TO"; readonly value: Date }
  | { readonly fn: "BETWEEN"; readonly low: Date; readonly high: Date }
  | { readonly fn: "TIME_FRAME"; readonly timeFrame: TimeFrame }
  | { readonly fn: "IN"; readonly values: readonly Date[] }
  | { readonly fn: "NOT_IN"; readonly values: readonly Date[] };
```

- [ ] **Step 4: Add evaluation logic in filter-eval.ts**

Add IN/NOT_IN cases to `evaluateDateFilter`:

```typescript
function evaluateDateFilter(cell: CellValue, filter: DateFilter, resolved: ResolvedTimeFrames): boolean {
  if (filter.fn === "IS_NULL") return cell.type === "NULL";
  if (filter.fn === "NOT_NULL") return cell.type !== "NULL";
  if (cell.type === "NULL") return false;

  const value = cell.type === "DATE" ? cell.value.getTime() : NaN;

  switch (filter.fn) {
    case "EQUALS_TO": return value === filter.value.getTime();
    case "NOT_EQUALS_TO": return value !== filter.value.getTime();
    case "GREATER_THAN": return value > filter.value.getTime();
    case "GREATER_OR_EQUALS_TO": return value >= filter.value.getTime();
    case "LOWER_THAN": return value < filter.value.getTime();
    case "LOWER_OR_EQUALS_TO": return value <= filter.value.getTime();
    case "BETWEEN": return value >= filter.low.getTime() && value <= filter.high.getTime();
    case "TIME_FRAME": {
      const range = resolved.get(filter.timeFrame);
      if (!range) return false;
      return value >= range.from.getTime() && value <= range.to.getTime();
    }
    case "IN": return filter.values.some((d) => d.getTime() === value);
    case "NOT_IN": return !filter.values.some((d) => d.getTime() === value);
  }
}
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @melviz/core run test -- filter-eval`
Expected: All tests pass including new IN/NOT_IN tests.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/filter.ts packages/core/src/dataset/filter-eval.ts packages/core/src/dataset/filter-eval.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: add IN/NOT_IN variants to DateFilter  Refs #4"
```

---

### Task 3: Fix compareValues localeCompare bug

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write the failing test**

Add to `group-eval.test.ts`:

```typescript
describe("compareValues — codepoint order", () => {
  it("sorts uppercase before lowercase (codepoint, not locale)", () => {
    const ds = toTypedDataSet({
      columns: [
        col("cat", "Category", ColumnType.LABEL),
        col("val", "Value", ColumnType.NUMBER),
      ],
      data: [
        ["banana", "1"],
        ["Apple", "2"],
        ["cherry", "3"],
      ],
    });
    const result = applyGroup(ds, {
      type: "group",
      groupingKey: {
        sourceId: "cat" as ColumnId,
        columnId: "cat" as ColumnId,
        strategy: { mode: "distinct" },
        maxIntervals: 15,
        emptyIntervals: false,
        ascendingOrder: true,
      },
      columns: [
        { kind: "key", sourceId: "cat" as ColumnId, columnId: "cat" as ColumnId },
        { kind: "aggregate", sourceId: "val" as ColumnId, columnId: "val" as ColumnId, fn: { fn: "SUM" } },
      ],
    });
    const labels = result.rows.map((r) => r.text("cat" as ColumnId));
    // Codepoint order: 'A' (65) < 'b' (98) < 'c' (99)
    expect(labels).toEqual(["Apple", "banana", "cherry"]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @melviz/core run test -- group-eval`
Expected: FAIL — localeCompare may sort "Apple" after "banana" depending on locale.

- [ ] **Step 3: Fix compareValues**

In `group-eval.ts`, replace line 191:

```typescript
// Old:
return a.value.localeCompare(b.value);

// New:
return a.value < b.value ? -1 : a.value > b.value ? 1 : 0;
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- group-eval`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/group-eval.ts packages/core/src/dataset/group-eval.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "fix: compareValues uses codepoint order, not localeCompare  Refs #4"
```

---

### Task 4: Add RESOLUTION_FAILED to DataSetErrorCode

**Files:**
- Modify: `packages/core/src/dataset/errors.ts`

- [ ] **Step 1: Add the new error code**

```typescript
export type DataSetErrorCode =
  | "FETCH_FAILED"
  | "PARSE_FAILED"
  | "SCHEMA_MISMATCH"
  | "TRANSFORM_FAILED"
  | "TIMEOUT"
  | "INVALID_REF"
  | "UNKNOWN_COLUMN"
  | "TYPE_MISMATCH"
  | "UNKNOWN_PROVIDER"
  | "INVALID_OPERATION"
  | "RESOLUTION_FAILED";
```

- [ ] **Step 2: Run tests to verify nothing breaks**

Run: `yarn workspace @melviz/core run test`
Expected: All tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/errors.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: add RESOLUTION_FAILED to DataSetErrorCode  Refs #4"
```

---

### Task 5: Create ExpressionError

**Files:**
- Create: `packages/core/src/expression/errors.ts`
- Create: `packages/core/src/expression/errors.test.ts`

- [ ] **Step 1: Write the test**

```typescript
import { describe, it, expect } from "vitest";
import { ExpressionError } from "./errors.js";

describe("ExpressionError", () => {
  it("creates a SYNTAX_ERROR with position", () => {
    const err = new ExpressionError("SYNTAX_ERROR", "bad + +", 6, "Unexpected token");
    expect(err.code).toBe("SYNTAX_ERROR");
    expect(err.expression).toBe("bad + +");
    expect(err.position).toBe(6);
    expect(err.message).toBe("SYNTAX_ERROR: Unexpected token");
    expect(err).toBeInstanceOf(Error);
    expect(err.name).toBe("ExpressionError");
  });

  it("creates EVALUATION_FAILED without position", () => {
    const err = new ExpressionError("EVALUATION_FAILED", "$unknown()", undefined, "Unknown function");
    expect(err.code).toBe("EVALUATION_FAILED");
    expect(err.position).toBeUndefined();
  });

  it("creates TYPE_COERCION_FAILED", () => {
    const err = new ExpressionError("TYPE_COERCION_FAILED", "$keys(value)", undefined, "Result is an array");
    expect(err.code).toBe("TYPE_COERCION_FAILED");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @melviz/core run test -- expression/errors`
Expected: FAIL — module not found.

- [ ] **Step 3: Create the expression directory and errors module**

```typescript
// packages/core/src/expression/errors.ts

export type ExpressionErrorCode =
  | "SYNTAX_ERROR"
  | "EVALUATION_FAILED"
  | "TYPE_COERCION_FAILED";

export class ExpressionError extends Error {
  constructor(
    readonly code: ExpressionErrorCode,
    readonly expression: string,
    readonly position: number | undefined,
    message?: string,
  ) {
    super(`${code}: ${message ?? "Expression error"}`);
    this.name = "ExpressionError";
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- expression/errors`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/expression/
git -C /Users/mdproctor/claude/melviz commit -m "feat: ExpressionError type for expression evaluation domain  Refs #4"
```

---

### Task 6: Create JSONata bridge

**Files:**
- Modify: `packages/core/package.json` (add jsonata dependency)
- Create: `packages/core/src/expression/jsonata-bridge.ts`
- Create: `packages/core/src/expression/jsonata-bridge.test.ts`

- [ ] **Step 1: Add jsonata dependency**

```bash
yarn workspace @melviz/core add jsonata
```

- [ ] **Step 2: Write the tests**

```typescript
// packages/core/src/expression/jsonata-bridge.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { compile, compileOrCached, clearCache } from "./jsonata-bridge.js";
import { ExpressionError } from "./errors.js";

describe("compile", () => {
  it("compiles a valid expression", () => {
    const expr = compile("2 + 3");
    expect(expr).toBeDefined();
    expect(typeof expr.evaluate).toBe("function");
  });

  it("throws ExpressionError with SYNTAX_ERROR for invalid expression", () => {
    expect(() => compile("2 + +")).toThrow(ExpressionError);
    try {
      compile("2 + +");
    } catch (e) {
      const err = e as ExpressionError;
      expect(err.code).toBe("SYNTAX_ERROR");
      expect(err.expression).toBe("2 + +");
      expect(err.position).toBeTypeOf("number");
    }
  });
});

describe("evaluate", () => {
  it("evaluates simple arithmetic", async () => {
    const expr = compile("2 + 3");
    const result = await expr.evaluate({});
    expect(result).toBe(5);
  });

  it("evaluates with data binding", async () => {
    const expr = compile("account.name");
    const result = await expr.evaluate({ account: { name: "Alice" } });
    expect(result).toBe("Alice");
  });

  it("evaluates with variable bindings", async () => {
    const expr = compile("$x * $y");
    const result = await expr.evaluate({}, { x: 3, y: 4 });
    expect(result).toBe(12);
  });

  it("returns undefined for missing path", async () => {
    const expr = compile("missing.path");
    const result = await expr.evaluate({ other: 1 });
    expect(result).toBeUndefined();
  });

  it("evaluates string functions", async () => {
    const expr = compile("$uppercase(name)");
    const result = await expr.evaluate({ name: "hello" });
    expect(result).toBe("HELLO");
  });
});

describe("compileOrCached", () => {
  beforeEach(() => {
    clearCache();
  });

  it("returns the same compiled object for the same expression string", () => {
    const a = compileOrCached("1 + 1");
    const b = compileOrCached("1 + 1");
    expect(a).toBe(b);
  });

  it("returns different objects for different expressions", () => {
    const a = compileOrCached("1 + 1");
    const b = compileOrCached("2 + 2");
    expect(a).not.toBe(b);
  });

  it("clearCache causes recompilation", () => {
    const a = compileOrCached("1 + 1");
    clearCache();
    const b = compileOrCached("1 + 1");
    expect(a).not.toBe(b);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- jsonata-bridge`
Expected: FAIL — module not found.

- [ ] **Step 4: Implement the bridge**

```typescript
// packages/core/src/expression/jsonata-bridge.ts
import jsonata from "jsonata";
import { ExpressionError } from "./errors.js";

export interface CompiledExpression {
  evaluate(data: unknown, bindings?: Record<string, unknown>): Promise<unknown>;
}

const cache = new Map<string, CompiledExpression>();

export function compile(expression: string): CompiledExpression {
  try {
    const compiled = jsonata(expression);
    return {
      evaluate: (data: unknown, bindings?: Record<string, unknown>) =>
        compiled.evaluate(data, bindings).catch((err: unknown) => {
          throw new ExpressionError(
            "EVALUATION_FAILED",
            expression,
            (err as { position?: number }).position,
            err instanceof Error ? err.message : String(err),
          );
        }),
    };
  } catch (err: unknown) {
    throw new ExpressionError(
      "SYNTAX_ERROR",
      expression,
      (err as { position?: number }).position,
      err instanceof Error ? err.message : String(err),
    );
  }
}

export function compileOrCached(expression: string): CompiledExpression {
  const existing = cache.get(expression);
  if (existing) return existing;
  const compiled = compile(expression);
  cache.set(expression, compiled);
  return compiled;
}

export function clearCache(): void {
  cache.clear();
}
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @melviz/core run test -- jsonata-bridge`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/package.json packages/core/src/expression/jsonata-bridge.ts packages/core/src/expression/jsonata-bridge.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: JSONata bridge with expression cache  Refs #4"
```

---

### Task 7: Create expression evaluator

**Files:**
- Create: `packages/core/src/expression/evaluator.ts`
- Create: `packages/core/src/expression/evaluator.test.ts`

- [ ] **Step 1: Write the tests**

```typescript
// packages/core/src/expression/evaluator.test.ts
import { describe, it, expect, vi } from "vitest";
import { evaluateExpression } from "./evaluator.js";
import type { ExpressionError } from "./errors.js";

describe("evaluateExpression — fast paths", () => {
  it("returns value when expression is undefined", async () => {
    expect(await evaluateExpression("hello", undefined)).toBe("hello");
  });

  it("returns value when expression is empty string", async () => {
    expect(await evaluateExpression("hello", "")).toBe("hello");
  });

  it("returns value when expression is 'value'", async () => {
    expect(await evaluateExpression("hello", "value")).toBe("hello");
  });

  it("returns null when value is null and no expression", async () => {
    expect(await evaluateExpression(null, undefined)).toBeNull();
  });

  it("returns null when value is null and expression is 'value'", async () => {
    expect(await evaluateExpression(null, "value")).toBeNull();
  });
});

describe("evaluateExpression — transforms", () => {
  it("evaluates $uppercase", async () => {
    expect(await evaluateExpression("hello", "$uppercase(value)")).toBe("HELLO");
  });

  it("evaluates arithmetic", async () => {
    expect(await evaluateExpression("42", "value * 100")).toBe("4200");
  });

  it("evaluates $trim", async () => {
    expect(await evaluateExpression("  hello  ", "$trim(value)")).toBe("hello");
  });

  it("evaluates $substring", async () => {
    expect(await evaluateExpression("hello world", "$substring(value, 0, 5)")).toBe("hello");
  });
});

describe("evaluateExpression — null handling", () => {
  it("null value with null-coalescing expression", async () => {
    expect(await evaluateExpression(null, "value ? value : 'N/A'")).toBe("N/A");
  });

  it("null value with arithmetic returns null", async () => {
    expect(await evaluateExpression(null, "value * 100")).toBeNull();
  });
});

describe("evaluateExpression — coercion", () => {
  it("coerces number result to string", async () => {
    const result = await evaluateExpression("5", "value + 10");
    expect(result).toBe("15");
    expect(typeof result).toBe("string");
  });

  it("coerces boolean result to string", async () => {
    const result = await evaluateExpression("5", "value > 3");
    expect(result).toBe("true");
  });

  it("array result triggers onError and returns original value", async () => {
    const onError = vi.fn();
    const result = await evaluateExpression("hello", "$split(value, '')", onError);
    expect(result).toBe("hello");
    expect(onError).toHaveBeenCalledOnce();
    expect(onError.mock.calls[0]![0]!.code).toBe("TYPE_COERCION_FAILED");
  });
});

describe("evaluateExpression — error handling", () => {
  it("invalid expression returns original value and calls onError", async () => {
    const onError = vi.fn();
    const result = await evaluateExpression("hello", "bad + +", onError);
    expect(result).toBe("hello");
    expect(onError).toHaveBeenCalledOnce();
    expect(onError.mock.calls[0]![0]!.code).toBe("SYNTAX_ERROR");
  });

  it("without onError, invalid expression still returns original value", async () => {
    const result = await evaluateExpression("hello", "bad + +");
    expect(result).toBe("hello");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- evaluator`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the evaluator**

```typescript
// packages/core/src/expression/evaluator.ts
import { compileOrCached } from "./jsonata-bridge.js";
import { ExpressionError } from "./errors.js";

export async function evaluateExpression(
  value: string | null,
  expression: string | undefined,
  onError?: (error: ExpressionError) => void,
): Promise<string | null> {
  if (!expression || expression === "value") {
    return value;
  }

  try {
    const compiled = compileOrCached(expression);
    const result = await compiled.evaluate({}, { value });
    return coerceResult(result, value, expression, onError);
  } catch (err: unknown) {
    if (onError) {
      onError(
        err instanceof ExpressionError
          ? err
          : new ExpressionError("EVALUATION_FAILED", expression, undefined, String(err)),
      );
    }
    return value;
  }
}

function coerceResult(
  result: unknown,
  originalValue: string | null,
  expression: string,
  onError?: (error: ExpressionError) => void,
): string | null {
  if (result === undefined || result === null) {
    return null;
  }
  if (typeof result === "string") {
    return result;
  }
  if (typeof result === "number" || typeof result === "boolean") {
    return String(result);
  }
  if (onError) {
    onError(
      new ExpressionError(
        "TYPE_COERCION_FAILED",
        expression,
        undefined,
        `Expression produced ${typeof result}, expected scalar`,
      ),
    );
  }
  return originalValue;
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- evaluator`
Expected: PASS.

- [ ] **Step 5: Run full test suite**

Run: `yarn workspace @melviz/core run test`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/expression/evaluator.ts packages/core/src/expression/evaluator.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: expression evaluator with null semantics and coercion  Closes #4"
```

---

### Task 8: Create DataSetLookup model

**Files:**
- Create: `packages/core/src/dataset/lookup.ts`
- Create: `packages/core/src/dataset/lookup.test.ts`

- [ ] **Step 1: Write the tests**

```typescript
// packages/core/src/dataset/lookup.test.ts
import { describe, it, expect } from "vitest";
import { createLookup } from "./lookup.js";
import type { DataSetLookup } from "./lookup.js";
import type { ColumnId } from "./types.js";
import type { DataSetId } from "./types.js";
import type { ResolvedFilterOp } from "./filter.js";
import type { GroupOp } from "./group.js";
import type { SortOp } from "./sort.js";
import { DataSetError } from "./errors.js";

function dsId(id: string): DataSetId {
  return id as DataSetId;
}

describe("createLookup", () => {
  it("creates a lookup with valid ops", () => {
    const filter: ResolvedFilterOp = {
      type: "filter",
      expressions: [{ type: "numeric", columnId: "x" as ColumnId, filter: { fn: "IS_NULL" } }],
    };
    const lookup = createLookup(dsId("test"), [filter]);
    expect(lookup.dataSetId).toBe("test");
    expect(lookup.operations).toHaveLength(1);
    expect(lookup.operations[0]!.type).toBe("filter");
  });

  it("creates a lookup with empty ops", () => {
    const lookup = createLookup(dsId("test"), []);
    expect(lookup.operations).toHaveLength(0);
  });

  it("throws on invalid op order (sort before group)", () => {
    const sort: SortOp = { type: "sort", columns: [{ columnId: "x" as ColumnId, order: "ASCENDING" }] };
    const group: GroupOp = {
      type: "group",
      groupingKey: null,
      columns: [{ kind: "aggregate", sourceId: "x" as ColumnId, columnId: "x" as ColumnId, fn: { fn: "COUNT" } }],
    };
    expect(() => createLookup(dsId("test"), [sort, group])).toThrow(DataSetError);
  });

  it("lookup is frozen", () => {
    const lookup = createLookup(dsId("test"), []);
    expect(Object.isFrozen(lookup)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- lookup.test`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the lookup model**

```typescript
// packages/core/src/dataset/lookup.ts
import type { DataSetId } from "./types.js";
import type { DataSetOp } from "./ops.js";
import { validateOpOrder } from "./ops.js";

export interface DataSetLookup {
  readonly dataSetId: DataSetId;
  readonly operations: readonly DataSetOp[];
}

export function createLookup(
  dataSetId: DataSetId,
  operations: readonly DataSetOp[],
): DataSetLookup {
  validateOpOrder(operations);
  return Object.freeze({ dataSetId, operations: Object.freeze([...operations]) });
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- lookup.test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/lookup.ts packages/core/src/dataset/lookup.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: DataSetLookup model with validated op ordering  Refs #5"
```

---

### Task 9: Create DataSetLookupConstraints

**Files:**
- Create: `packages/core/src/dataset/lookup-constraints.ts`
- Create: `packages/core/src/dataset/lookup-constraints.test.ts`

- [ ] **Step 1: Write the tests**

```typescript
// packages/core/src/dataset/lookup-constraints.test.ts
import { describe, it, expect } from "vitest";
import { validateLookup, DEFAULT_CONSTRAINTS } from "./lookup-constraints.js";
import type { DataSetLookupConstraints } from "./lookup-constraints.js";
import { createLookup } from "./lookup.js";
import type { ColumnId, DataSetId, Column } from "./types.js";
import { ColumnType } from "./types.js";
import type { ResolvedFilterOp } from "./filter.js";
import type { GroupOp } from "./group.js";

const dsId = "test" as DataSetId;

function filterOp(): ResolvedFilterOp {
  return { type: "filter", expressions: [{ type: "numeric", columnId: "x" as ColumnId, filter: { fn: "IS_NULL" } }] };
}

function groupOp(): GroupOp {
  return {
    type: "group",
    groupingKey: {
      sourceId: "x" as ColumnId, columnId: "x" as ColumnId,
      strategy: { mode: "distinct" }, maxIntervals: 15,
      emptyIntervals: false, ascendingOrder: true,
    },
    columns: [
      { kind: "key", sourceId: "x" as ColumnId, columnId: "x" as ColumnId },
      { kind: "aggregate", sourceId: "y" as ColumnId, columnId: "y" as ColumnId, fn: { fn: "COUNT" } },
    ],
  };
}

describe("validateLookup", () => {
  it("valid lookup against DEFAULT_CONSTRAINTS returns no violations", () => {
    const lookup = createLookup(dsId, [filterOp(), groupOp()]);
    expect(validateLookup(lookup, DEFAULT_CONSTRAINTS)).toEqual([]);
  });

  it("FILTER_NOT_ALLOWED when filterAllowed is false", () => {
    const constraints: DataSetLookupConstraints = { ...DEFAULT_CONSTRAINTS, filterAllowed: false };
    const lookup = createLookup(dsId, [filterOp()]);
    const violations = validateLookup(lookup, constraints);
    expect(violations).toHaveLength(1);
    expect(violations[0]!.code).toBe("FILTER_NOT_ALLOWED");
  });

  it("GROUP_NOT_ALLOWED when groupAllowed is false", () => {
    const constraints: DataSetLookupConstraints = { ...DEFAULT_CONSTRAINTS, groupAllowed: false };
    const lookup = createLookup(dsId, [groupOp()]);
    const violations = validateLookup(lookup, constraints);
    expect(violations).toHaveLength(1);
    expect(violations[0]!.code).toBe("GROUP_NOT_ALLOWED");
  });

  it("GROUP_REQUIRED when groupRequired but no groups", () => {
    const constraints: DataSetLookupConstraints = { ...DEFAULT_CONSTRAINTS, groupRequired: true };
    const lookup = createLookup(dsId, []);
    const violations = validateLookup(lookup, constraints);
    expect(violations).toHaveLength(1);
    expect(violations[0]!.code).toBe("GROUP_REQUIRED");
  });

  it("TOO_MANY_GROUPS when exceeding maxGroups", () => {
    const constraints: DataSetLookupConstraints = { ...DEFAULT_CONSTRAINTS, maxGroups: 1 };
    const lookup = createLookup(dsId, [groupOp(), groupOp()]);
    const violations = validateLookup(lookup, constraints);
    expect(violations).toHaveLength(1);
    expect(violations[0]!.code).toBe("TOO_MANY_GROUPS");
  });

  it("maxGroups absent means unlimited", () => {
    const lookup = createLookup(dsId, [groupOp(), groupOp()]);
    expect(validateLookup(lookup, DEFAULT_CONSTRAINTS)).toEqual([]);
  });

  it("multiple violations returned in one call", () => {
    const constraints: DataSetLookupConstraints = {
      ...DEFAULT_CONSTRAINTS,
      filterAllowed: false,
      groupAllowed: false,
    };
    const lookup = createLookup(dsId, [filterOp(), groupOp()]);
    const violations = validateLookup(lookup, constraints);
    expect(violations.length).toBeGreaterThanOrEqual(2);
    const codes = violations.map((v) => v.code);
    expect(codes).toContain("FILTER_NOT_ALLOWED");
    expect(codes).toContain("GROUP_NOT_ALLOWED");
  });

  it("TOO_FEW_COLUMNS when below minColumns", () => {
    const constraints: DataSetLookupConstraints = { ...DEFAULT_CONSTRAINTS, minColumns: 5 };
    const lookup = createLookup(dsId, [groupOp()]);
    const violations = validateLookup(lookup, constraints);
    expect(violations.some((v) => v.code === "TOO_FEW_COLUMNS")).toBe(true);
  });

  it("TOO_MANY_COLUMNS when above maxColumns", () => {
    const constraints: DataSetLookupConstraints = { ...DEFAULT_CONSTRAINTS, maxColumns: 1 };
    const lookup = createLookup(dsId, [groupOp()]);
    const violations = validateLookup(lookup, constraints);
    expect(violations.some((v) => v.code === "TOO_MANY_COLUMNS")).toBe(true);
  });

  it("INVALID_COLUMN_TYPE with columns provided for full validation", () => {
    const constraints: DataSetLookupConstraints = {
      ...DEFAULT_CONSTRAINTS,
      columnTypes: [[ColumnType.NUMBER], [ColumnType.NUMBER]],
    };
    const columns: readonly Column[] = [
      { id: "x" as ColumnId, name: "X", type: ColumnType.TEXT },
      { id: "y" as ColumnId, name: "Y", type: ColumnType.NUMBER },
    ];
    const lookup = createLookup(dsId, [groupOp()]);
    const violations = validateLookup(lookup, constraints, columns);
    expect(violations.some((v) => v.code === "INVALID_COLUMN_TYPE")).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- lookup-constraints`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement constraints validation**

```typescript
// packages/core/src/dataset/lookup-constraints.ts
import type { Column, ColumnType } from "./types.js";
import type { DataSetLookup } from "./lookup.js";
import type { GroupOp, ResultColumn, Aggregation } from "./group.js";

export interface DataSetLookupConstraints {
  readonly filterAllowed: boolean;
  readonly groupAllowed: boolean;
  readonly groupRequired: boolean;
  readonly maxGroups?: number;
  readonly minColumns?: number;
  readonly maxColumns?: number;
  readonly columnTypes?: readonly (readonly ColumnType[])[];
  readonly uniqueColumnIds: boolean;
  readonly extraColumnsAllowed: boolean;
  readonly extraColumnsType?: ColumnType;
}

export const DEFAULT_CONSTRAINTS: DataSetLookupConstraints = Object.freeze({
  filterAllowed: true,
  groupAllowed: true,
  groupRequired: false,
  uniqueColumnIds: false,
  extraColumnsAllowed: true,
});

export type LookupViolationCode =
  | "FILTER_NOT_ALLOWED"
  | "GROUP_NOT_ALLOWED"
  | "GROUP_REQUIRED"
  | "TOO_MANY_GROUPS"
  | "TOO_FEW_COLUMNS"
  | "TOO_MANY_COLUMNS"
  | "INVALID_COLUMN_TYPE"
  | "DUPLICATE_COLUMN_ID"
  | "EXTRA_COLUMN_NOT_ALLOWED"
  | "EXTRA_COLUMN_WRONG_TYPE";

export interface LookupViolation {
  readonly code: LookupViolationCode;
  readonly message: string;
}

export function validateLookup(
  lookup: DataSetLookup,
  constraints: DataSetLookupConstraints,
  columns?: readonly Column[],
): readonly LookupViolation[] {
  const violations: LookupViolation[] = [];
  const ops = lookup.operations;

  const filterOps = ops.filter((o) => o.type === "filter");
  const groupOps = ops.filter((o) => o.type === "group") as GroupOp[];

  if (!constraints.filterAllowed && filterOps.length > 0) {
    violations.push({ code: "FILTER_NOT_ALLOWED", message: "Filters are not allowed" });
  }

  if (!constraints.groupAllowed && groupOps.length > 0) {
    violations.push({ code: "GROUP_NOT_ALLOWED", message: "Groups are not allowed" });
  }

  if (constraints.groupRequired && groupOps.length === 0) {
    violations.push({ code: "GROUP_REQUIRED", message: "At least one group is required" });
  }

  if (constraints.maxGroups !== undefined && groupOps.length > constraints.maxGroups) {
    violations.push({ code: "TOO_MANY_GROUPS", message: `Max ${constraints.maxGroups} groups, found ${groupOps.length}` });
  }

  if (groupOps.length > 0) {
    const lastGroup = groupOps[groupOps.length - 1]!;
    const resultColumns = lastGroup.columns;
    validateColumns(resultColumns, constraints, columns, violations);
  }

  return violations;
}

function validateColumns(
  resultColumns: readonly ResultColumn[],
  constraints: DataSetLookupConstraints,
  sourceColumns: readonly Column[] | undefined,
  violations: LookupViolation[],
): void {
  const count = resultColumns.length;

  if (constraints.minColumns !== undefined && count < constraints.minColumns) {
    violations.push({ code: "TOO_FEW_COLUMNS", message: `Min ${constraints.minColumns} columns, found ${count}` });
  }

  if (constraints.maxColumns !== undefined && count > constraints.maxColumns) {
    violations.push({ code: "TOO_MANY_COLUMNS", message: `Max ${constraints.maxColumns} columns, found ${count}` });
  }

  if (constraints.uniqueColumnIds) {
    const ids = new Set<string>();
    for (const col of resultColumns) {
      if (ids.has(col.columnId)) {
        violations.push({ code: "DUPLICATE_COLUMN_ID", message: `Duplicate column ID: ${col.columnId}` });
      }
      ids.add(col.columnId);
    }
  }

  if (constraints.columnTypes) {
    for (let i = 0; i < resultColumns.length; i++) {
      const col = resultColumns[i]!;
      if (i < constraints.columnTypes.length) {
        const allowed = constraints.columnTypes[i]!;
        const inferredType = inferResultColumnType(col, sourceColumns);
        if (inferredType !== undefined && !allowed.includes(inferredType)) {
          violations.push({ code: "INVALID_COLUMN_TYPE", message: `Column ${i} type ${inferredType} not in [${allowed.join(", ")}]` });
        }
      } else if (!constraints.extraColumnsAllowed) {
        violations.push({ code: "EXTRA_COLUMN_NOT_ALLOWED", message: `Extra column at position ${i}` });
      } else if (constraints.extraColumnsType !== undefined) {
        const inferredType = inferResultColumnType(col, sourceColumns);
        if (inferredType !== undefined && inferredType !== constraints.extraColumnsType) {
          violations.push({ code: "EXTRA_COLUMN_WRONG_TYPE", message: `Extra column ${i} type ${inferredType}, expected ${constraints.extraColumnsType}` });
        }
      }
    }
  }
}

function inferResultColumnType(
  col: ResultColumn,
  sourceColumns: readonly Column[] | undefined,
): ColumnType | undefined {
  if (col.kind === "key") return "LABEL" as ColumnType;
  if (col.kind === "aggregate") return inferAggregateType(col.fn, col.sourceId, sourceColumns);
  if (col.kind === "select") return resolveSourceType(col.sourceId, sourceColumns);
  return undefined;
}

function inferAggregateType(
  fn: Aggregation,
  sourceId: string,
  sourceColumns: readonly Column[] | undefined,
): ColumnType | undefined {
  switch (fn.fn) {
    case "COUNT": return "NUMBER" as ColumnType;
    case "DISTINCT": return "NUMBER" as ColumnType;
    case "SUM": return "NUMBER" as ColumnType;
    case "AVERAGE": return "NUMBER" as ColumnType;
    case "MEDIAN": return "NUMBER" as ColumnType;
    case "JOIN": return "TEXT" as ColumnType;
    case "MIN":
    case "MAX":
      return resolveSourceType(sourceId, sourceColumns);
  }
}

function resolveSourceType(
  sourceId: string,
  sourceColumns: readonly Column[] | undefined,
): ColumnType | undefined {
  if (!sourceColumns) return undefined;
  const col = sourceColumns.find((c) => c.id === sourceId);
  return col?.type;
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- lookup-constraints`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/lookup-constraints.ts packages/core/src/dataset/lookup-constraints.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: DataSetLookupConstraints with partial/full type validation  Refs #5"
```

---

### Task 10: Create filter type resolution

**Files:**
- Create: `packages/core/src/dataset/filter-resolve.ts`
- Create: `packages/core/src/dataset/filter-resolve.test.ts`

- [ ] **Step 1: Write the tests**

```typescript
// packages/core/src/dataset/filter-resolve.test.ts
import { describe, it, expect } from "vitest";
import { resolveFilterTypes } from "./filter-resolve.js";
import type { FilterExpression, ResolvedFilterExpression } from "./filter.js";
import type { Column, ColumnId } from "./types.js";
import { ColumnType } from "./types.js";
import { DataSetError } from "./errors.js";

const columns: readonly Column[] = [
  { id: "price" as ColumnId, name: "Price", type: ColumnType.NUMBER },
  { id: "name" as ColumnId, name: "Name", type: ColumnType.TEXT },
  { id: "date" as ColumnId, name: "Date", type: ColumnType.DATE },
  { id: "tag" as ColumnId, name: "Tag", type: ColumnType.LABEL },
];

describe("resolveFilterTypes — leaf resolution", () => {
  it("resolves unresolved on NUMBER column to numeric filter", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "price" as ColumnId, fn: "EQUALS_TO", args: ["42"] };
    const resolved = resolveFilterTypes(expr, columns);
    expect(resolved.type).toBe("numeric");
    if (resolved.type === "numeric") {
      expect(resolved.filter).toEqual({ fn: "EQUALS_TO", value: 42 });
    }
  });

  it("resolves unresolved on TEXT column to string filter", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "name" as ColumnId, fn: "EQUALS_TO", args: ["hello"] };
    const resolved = resolveFilterTypes(expr, columns);
    expect(resolved.type).toBe("string");
    if (resolved.type === "string") {
      expect(resolved.filter).toEqual({ fn: "EQUALS_TO", value: "hello" });
    }
  });

  it("resolves unresolved on LABEL column to string filter", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "tag" as ColumnId, fn: "EQUALS_TO", args: ["red"] };
    const resolved = resolveFilterTypes(expr, columns);
    expect(resolved.type).toBe("string");
  });

  it("resolves unresolved on DATE column to date filter", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "date" as ColumnId, fn: "EQUALS_TO", args: ["2024-01-15T00:00:00.000Z"] };
    const resolved = resolveFilterTypes(expr, columns);
    expect(resolved.type).toBe("date");
    if (resolved.type === "date") {
      expect(resolved.filter.fn).toBe("EQUALS_TO");
    }
  });

  it("resolves BETWEEN on NUMBER", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "price" as ColumnId, fn: "BETWEEN", args: ["10", "50"] };
    const resolved = resolveFilterTypes(expr, columns);
    if (resolved.type === "numeric") {
      expect(resolved.filter).toEqual({ fn: "BETWEEN", low: 10, high: 50 });
    }
  });

  it("resolves IN on NUMBER", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "price" as ColumnId, fn: "IN", args: ["10", "20", "30"] };
    const resolved = resolveFilterTypes(expr, columns);
    if (resolved.type === "numeric") {
      expect(resolved.filter).toEqual({ fn: "IN", values: [10, 20, 30] });
    }
  });

  it("resolves IN on DATE", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "date" as ColumnId, fn: "IN", args: ["2024-01-01T00:00:00.000Z", "2024-06-01T00:00:00.000Z"] };
    const resolved = resolveFilterTypes(expr, columns);
    expect(resolved.type).toBe("date");
    if (resolved.type === "date" && resolved.filter.fn === "IN") {
      expect(resolved.filter.values).toHaveLength(2);
    }
  });

  it("resolves IS_NULL (no args needed)", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "price" as ColumnId, fn: "IS_NULL", args: [] };
    const resolved = resolveFilterTypes(expr, columns);
    if (resolved.type === "numeric") {
      expect(resolved.filter).toEqual({ fn: "IS_NULL" });
    }
  });
});

describe("resolveFilterTypes — combinators", () => {
  it("resolves AND children recursively", () => {
    const expr: FilterExpression = {
      type: "and",
      children: [
        { type: "unresolved", columnId: "price" as ColumnId, fn: "GREATER_THAN", args: ["10"] },
        { type: "unresolved", columnId: "price" as ColumnId, fn: "LOWER_THAN", args: ["100"] },
      ],
    };
    const resolved = resolveFilterTypes(expr, columns);
    expect(resolved.type).toBe("and");
    if (resolved.type === "and") {
      expect(resolved.children).toHaveLength(2);
      expect(resolved.children[0]!.type).toBe("numeric");
      expect(resolved.children[1]!.type).toBe("numeric");
    }
  });

  it("passes through already-resolved leaves", () => {
    const resolved: FilterExpression = { type: "numeric", columnId: "price" as ColumnId, filter: { fn: "IS_NULL" } };
    const result = resolveFilterTypes(resolved, columns);
    expect(result).toEqual(resolved);
  });
});

describe("resolveFilterTypes — errors", () => {
  it("throws RESOLUTION_FAILED for bad numeric arg", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "price" as ColumnId, fn: "EQUALS_TO", args: ["not-a-number"] };
    expect(() => resolveFilterTypes(expr, columns)).toThrow(DataSetError);
    try { resolveFilterTypes(expr, columns); } catch (e) {
      expect((e as DataSetError).code).toBe("RESOLUTION_FAILED");
    }
  });

  it("throws RESOLUTION_FAILED for invalid DATE arg", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "date" as ColumnId, fn: "EQUALS_TO", args: ["not-a-date"] };
    expect(() => resolveFilterTypes(expr, columns)).toThrow(DataSetError);
  });

  it("throws UNKNOWN_COLUMN for unknown column ID", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "missing" as ColumnId, fn: "IS_NULL", args: [] };
    expect(() => resolveFilterTypes(expr, columns)).toThrow(DataSetError);
    try { resolveFilterTypes(expr, columns); } catch (e) {
      expect((e as DataSetError).code).toBe("UNKNOWN_COLUMN");
    }
  });

  it("throws RESOLUTION_FAILED for LIKE_TO on NUMBER column", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "price" as ColumnId, fn: "LIKE_TO", args: ["%test%"] };
    expect(() => resolveFilterTypes(expr, columns)).toThrow(DataSetError);
  });

  it("throws RESOLUTION_FAILED for TIME_FRAME on TEXT column", () => {
    const expr: FilterExpression = { type: "unresolved", columnId: "name" as ColumnId, fn: "TIME_FRAME", args: ["now till end[MONTH]"] };
    expect(() => resolveFilterTypes(expr, columns)).toThrow(DataSetError);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- filter-resolve`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement filter resolution**

Create `packages/core/src/dataset/filter-resolve.ts`. This is a substantial file — the implementation walks the tree, checks column types against the compatibility matrix, parses args to typed values, and constructs resolved leaf expressions. Key structure:

```typescript
// packages/core/src/dataset/filter-resolve.ts
import type { Column, ColumnId } from "./types.js";
import { ColumnType } from "./types.js";
import type { FilterExpression, ResolvedFilterExpression, CoreFunctionType, NumericFilter, StringFilter, DateFilter } from "./filter.js";
import { DataSetError } from "./errors.js";
import { parseTimeFrame } from "./timeframe.js";

export function resolveFilterTypes(
  expression: FilterExpression,
  columns: readonly Column[],
): ResolvedFilterExpression {
  switch (expression.type) {
    case "numeric":
    case "string":
    case "date":
      return expression;
    case "and":
      return { type: "and", children: expression.children.map((c) => resolveFilterTypes(c, columns)) };
    case "or":
      return { type: "or", children: expression.children.map((c) => resolveFilterTypes(c, columns)) };
    case "not":
      return { type: "not", child: resolveFilterTypes(expression.child, columns) };
    case "unresolved":
      return resolveLeaf(expression.columnId, expression.fn, expression.args, columns);
  }
}

function resolveLeaf(
  columnId: ColumnId,
  fn: CoreFunctionType,
  args: readonly string[],
  columns: readonly Column[],
): ResolvedFilterExpression {
  const column = columns.find((c) => c.id === columnId);
  if (!column) {
    throw new DataSetError("UNKNOWN_COLUMN", `Filter references unknown column: ${columnId}`);
  }

  switch (column.type) {
    case ColumnType.NUMBER:
      return { type: "numeric", columnId, filter: resolveNumericFilter(fn, args) };
    case ColumnType.TEXT:
    case ColumnType.LABEL:
      return { type: "string", columnId, filter: resolveStringFilter(fn, args) };
    case ColumnType.DATE:
      return { type: "date", columnId, filter: resolveDateFilter(fn, args) };
  }
}

function resolveNumericFilter(fn: CoreFunctionType, args: readonly string[]): NumericFilter {
  switch (fn) {
    case "IS_NULL": return { fn: "IS_NULL" };
    case "NOT_NULL": return { fn: "NOT_NULL" };
    case "EQUALS_TO": return { fn: "EQUALS_TO", value: parseNum(args[0]!) };
    case "NOT_EQUALS_TO": return { fn: "NOT_EQUALS_TO", value: parseNum(args[0]!) };
    case "GREATER_THAN": return { fn: "GREATER_THAN", value: parseNum(args[0]!) };
    case "GREATER_OR_EQUALS_TO": return { fn: "GREATER_OR_EQUALS_TO", value: parseNum(args[0]!) };
    case "LOWER_THAN": return { fn: "LOWER_THAN", value: parseNum(args[0]!) };
    case "LOWER_OR_EQUALS_TO": return { fn: "LOWER_OR_EQUALS_TO", value: parseNum(args[0]!) };
    case "BETWEEN": return { fn: "BETWEEN", low: parseNum(args[0]!), high: parseNum(args[1]!) };
    case "IN": return { fn: "IN", values: args.map(parseNum) };
    case "NOT_IN": return { fn: "NOT_IN", values: args.map(parseNum) };
    default:
      throw new DataSetError("RESOLUTION_FAILED", `${fn} is not valid for NUMBER columns`);
  }
}

function resolveStringFilter(fn: CoreFunctionType, args: readonly string[]): StringFilter {
  switch (fn) {
    case "IS_NULL": return { fn: "IS_NULL" };
    case "NOT_NULL": return { fn: "NOT_NULL" };
    case "EQUALS_TO": return { fn: "EQUALS_TO", value: args[0]! };
    case "NOT_EQUALS_TO": return { fn: "NOT_EQUALS_TO", value: args[0]! };
    case "GREATER_THAN": return { fn: "GREATER_THAN", value: args[0]! };
    case "GREATER_OR_EQUALS_TO": return { fn: "GREATER_OR_EQUALS_TO", value: args[0]! };
    case "LOWER_THAN": return { fn: "LOWER_THAN", value: args[0]! };
    case "LOWER_OR_EQUALS_TO": return { fn: "LOWER_OR_EQUALS_TO", value: args[0]! };
    case "BETWEEN": return { fn: "BETWEEN", low: args[0]!, high: args[1]! };
    case "LIKE_TO": return { fn: "LIKE_TO", pattern: args[0]!, caseSensitive: args[1] !== "false" };
    case "IN": return { fn: "IN", values: [...args] };
    case "NOT_IN": return { fn: "NOT_IN", values: [...args] };
    default:
      throw new DataSetError("RESOLUTION_FAILED", `${fn} is not valid for TEXT/LABEL columns`);
  }
}

function resolveDateFilter(fn: CoreFunctionType, args: readonly string[]): DateFilter {
  switch (fn) {
    case "IS_NULL": return { fn: "IS_NULL" };
    case "NOT_NULL": return { fn: "NOT_NULL" };
    case "EQUALS_TO": return { fn: "EQUALS_TO", value: parseDate(args[0]!) };
    case "NOT_EQUALS_TO": return { fn: "NOT_EQUALS_TO", value: parseDate(args[0]!) };
    case "GREATER_THAN": return { fn: "GREATER_THAN", value: parseDate(args[0]!) };
    case "GREATER_OR_EQUALS_TO": return { fn: "GREATER_OR_EQUALS_TO", value: parseDate(args[0]!) };
    case "LOWER_THAN": return { fn: "LOWER_THAN", value: parseDate(args[0]!) };
    case "LOWER_OR_EQUALS_TO": return { fn: "LOWER_OR_EQUALS_TO", value: parseDate(args[0]!) };
    case "BETWEEN": return { fn: "BETWEEN", low: parseDate(args[0]!), high: parseDate(args[1]!) };
    case "TIME_FRAME": return { fn: "TIME_FRAME", timeFrame: parseTimeFrame(args[0]!) };
    case "IN": return { fn: "IN", values: args.map(parseDate) };
    case "NOT_IN": return { fn: "NOT_IN", values: args.map(parseDate) };
    default:
      throw new DataSetError("RESOLUTION_FAILED", `${fn} is not valid for DATE columns`);
  }
}

function parseNum(s: string): number {
  const n = parseFloat(s);
  if (Number.isNaN(n)) {
    throw new DataSetError("RESOLUTION_FAILED", `Cannot parse "${s}" as number`);
  }
  return n;
}

function parseDate(s: string): Date {
  const d = new Date(s);
  if (Number.isNaN(d.getTime())) {
    throw new DataSetError("RESOLUTION_FAILED", `Cannot parse "${s}" as date`);
  }
  return d;
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- filter-resolve`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/filter-resolve.ts packages/core/src/dataset/filter-resolve.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: filter type resolution with compatibility matrix  Refs #5"
```

---

### Task 11: Create YAML lookup parser

**Files:**
- Create: `packages/core/src/dataset/lookup-parser.ts`
- Create: `packages/core/src/dataset/lookup-parser.test.ts`

- [ ] **Step 1: Write the tests**

```typescript
// packages/core/src/dataset/lookup-parser.test.ts
import { describe, it, expect } from "vitest";
import { parseLookup } from "./lookup-parser.js";
import type { ColumnId, DataSetId } from "./types.js";

describe("parseLookup — minimal", () => {
  it("parses uuid-only lookup", () => {
    const lookup = parseLookup({ uuid: "my-dataset" });
    expect(lookup.dataSetId).toBe("my-dataset");
    expect(lookup.operations).toHaveLength(0);
  });
});

describe("parseLookup — filters", () => {
  it("parses flat filter list as implicit AND of unresolved expressions", () => {
    const lookup = parseLookup({
      uuid: "ds",
      filter: [
        { column: "status", function: "EQUALS_TO", args: ["active"] },
        { column: "price", function: "GREATER_THAN", args: ["100"] },
      ],
    });
    expect(lookup.operations).toHaveLength(1);
    const filterOp = lookup.operations[0]!;
    expect(filterOp.type).toBe("filter");
    if (filterOp.type === "filter") {
      expect(filterOp.expressions).toHaveLength(2);
      expect(filterOp.expressions[0]!.type).toBe("unresolved");
    }
  });

  it("parses nested OR combinator", () => {
    const lookup = parseLookup({
      uuid: "ds",
      filter: [
        {
          or: [
            { column: "region", function: "EQUALS_TO", args: ["US"] },
            { column: "region", function: "EQUALS_TO", args: ["EU"] },
          ],
        },
      ],
    });
    const filterOp = lookup.operations[0]!;
    if (filterOp.type === "filter") {
      const first = filterOp.expressions[0]!;
      expect(first.type).toBe("or");
      if (first.type === "or") {
        expect(first.children).toHaveLength(2);
      }
    }
  });

  it("parses NOT combinator", () => {
    const lookup = parseLookup({
      uuid: "ds",
      filter: [{ not: { column: "archived", function: "EQUALS_TO", args: ["true"] } }],
    });
    const filterOp = lookup.operations[0]!;
    if (filterOp.type === "filter") {
      expect(filterOp.expressions[0]!.type).toBe("not");
    }
  });

  it("rejects unknown function via ZodError", () => {
    expect(() =>
      parseLookup({ uuid: "ds", filter: [{ column: "x", function: "BOGUS", args: [] }] }),
    ).toThrow();
  });
});

describe("parseLookup — groups", () => {
  it("parses group with key + aggregate columns", () => {
    const lookup = parseLookup({
      uuid: "ds",
      group: [
        {
          columnGroup: { source: "region" },
          columns: [
            { source: "region" },
            { source: "revenue", function: "SUM" },
          ],
        },
      ],
    });
    expect(lookup.operations).toHaveLength(1);
    const groupOp = lookup.operations[0]!;
    expect(groupOp.type).toBe("group");
    if (groupOp.type === "group") {
      expect(groupOp.groupingKey).not.toBeNull();
      expect(groupOp.columns).toHaveLength(2);
      expect(groupOp.columns[0]!.kind).toBe("key");
      expect(groupOp.columns[1]!.kind).toBe("aggregate");
    }
  });

  it("omitted columnGroup produces groupingKey: null", () => {
    const lookup = parseLookup({
      uuid: "ds",
      group: [
        {
          columns: [{ source: "revenue", function: "SUM" }],
        },
      ],
    });
    const groupOp = lookup.operations[0]!;
    if (groupOp.type === "group") {
      expect(groupOp.groupingKey).toBeNull();
    }
  });

  it("parses fixedCalendar strategy", () => {
    const lookup = parseLookup({
      uuid: "ds",
      group: [
        {
          columnGroup: { source: "date", strategy: "fixedCalendar", unit: "MONTH" },
          columns: [{ source: "date" }, { source: "count", function: "COUNT" }],
        },
      ],
    });
    const groupOp = lookup.operations[0]!;
    if (groupOp.type === "group" && groupOp.groupingKey) {
      expect(groupOp.groupingKey.strategy).toEqual({ mode: "fixedCalendar", unit: "MONTH" });
    }
  });
});

describe("parseLookup — sort", () => {
  it("parses sort with default ASCENDING", () => {
    const lookup = parseLookup({
      uuid: "ds",
      sort: [{ column: "revenue" }],
    });
    const sortOp = lookup.operations[0]!;
    expect(sortOp.type).toBe("sort");
    if (sortOp.type === "sort") {
      expect(sortOp.columns[0]!.order).toBe("ASCENDING");
    }
  });

  it("parses sort with DESCENDING", () => {
    const lookup = parseLookup({
      uuid: "ds",
      sort: [{ column: "revenue", order: "DESCENDING" }],
    });
    const sortOp = lookup.operations[0]!;
    if (sortOp.type === "sort") {
      expect(sortOp.columns[0]!.order).toBe("DESCENDING");
    }
  });
});

describe("parseLookup — full pipeline", () => {
  it("parses filter + group + sort in correct F*G*S order", () => {
    const lookup = parseLookup({
      uuid: "ds",
      filter: [{ column: "status", function: "EQUALS_TO", args: ["active"] }],
      group: [
        {
          columnGroup: { source: "region" },
          columns: [{ source: "region" }, { source: "revenue", function: "SUM" }],
        },
      ],
      sort: [{ column: "revenue", order: "DESCENDING" }],
    });
    expect(lookup.operations).toHaveLength(3);
    expect(lookup.operations[0]!.type).toBe("filter");
    expect(lookup.operations[1]!.type).toBe("group");
    expect(lookup.operations[2]!.type).toBe("sort");
  });
});

describe("parseLookup — malformed input", () => {
  it("missing uuid throws ZodError", () => {
    expect(() => parseLookup({})).toThrow();
  });

  it("filter missing column throws ZodError", () => {
    expect(() => parseLookup({ uuid: "ds", filter: [{ function: "IS_NULL" }] })).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- lookup-parser`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the YAML lookup parser**

Create `packages/core/src/dataset/lookup-parser.ts` with Zod schemas for filter nodes (leaf / and / or / not — mutually exclusive), group entries, sort entries, and the top-level lookup. The parser:

1. Validates via Zod (structural + enum)
2. Converts filter entries to `FilterExpression` (unresolved leaves + combinator tree)
3. Converts group entries to `GroupOp` (inferring ResultColumn kind)
4. Converts sort entries to `SortOp`
5. Assembles ops in F*G*S order
6. Delegates to `createLookup` for op validation

```typescript
// packages/core/src/dataset/lookup-parser.ts
import { z } from "zod";
import type { ColumnId, DataSetId } from "./types.js";
import type { FilterExpression, FilterOp, CoreFunctionType } from "./filter.js";
import type { GroupOp, GroupingKey, GroupStrategy, ResultColumn, Aggregation } from "./group.js";
import type { SortOp, SortColumn } from "./sort.js";
import type { DataSetLookup } from "./lookup.js";
import { createLookup } from "./lookup.js";
import type { FixedCalendarUnit } from "./group.js";
import type { DateIntervalType, Month, DayOfWeek } from "./date-interval.js";

const coreFunctionTypes = [
  "IS_NULL", "NOT_NULL", "EQUALS_TO", "NOT_EQUALS_TO", "LIKE_TO",
  "GREATER_THAN", "GREATER_OR_EQUALS_TO", "LOWER_THAN", "LOWER_OR_EQUALS_TO",
  "BETWEEN", "TIME_FRAME", "IN", "NOT_IN",
] as const;

const filterLeafSchema = z.object({
  column: z.string(),
  function: z.enum(coreFunctionTypes),
  args: z.array(z.union([z.string(), z.number()])).default([]),
});

type FilterNodeInput = z.infer<typeof filterLeafSchema> | { and: FilterNodeInput[] } | { or: FilterNodeInput[] } | { not: FilterNodeInput };

const filterNodeSchema: z.ZodType<FilterNodeInput> = z.union([
  filterLeafSchema,
  z.object({ and: z.lazy(() => z.array(filterNodeSchema)) }).strict(),
  z.object({ or: z.lazy(() => z.array(filterNodeSchema)) }).strict(),
  z.object({ not: z.lazy(() => filterNodeSchema) }).strict(),
]);

const aggregationFunctions = ["COUNT", "DISTINCT", "SUM", "AVERAGE", "MEDIAN", "MIN", "MAX", "JOIN"] as const;

const resultColumnSchema = z.object({
  source: z.string(),
  function: z.enum(aggregationFunctions).optional(),
  column: z.string().optional(),
  separator: z.string().optional(),
});

const strategies = ["distinct", "fixedCalendar", "dynamicRange", "dynamic"] as const;
const calendarUnits = ["QUARTER", "MONTH", "DAY_OF_WEEK", "HOUR", "MINUTE", "SECOND"] as const;
const months = ["JANUARY", "FEBRUARY", "MARCH", "APRIL", "MAY", "JUNE", "JULY", "AUGUST", "SEPTEMBER", "OCTOBER", "NOVEMBER", "DECEMBER"] as const;
const days = ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY", "SATURDAY", "SUNDAY"] as const;

const columnGroupSchema = z.object({
  source: z.string(),
  column: z.string().optional(),
  strategy: z.enum(strategies).default("distinct"),
  unit: z.string().optional(),
  maxIntervals: z.number().default(15),
  emptyIntervals: z.boolean().default(false),
  ascendingOrder: z.boolean().default(true),
  firstMonthOfYear: z.enum(months).optional(),
  firstDayOfWeek: z.enum(days).optional(),
});

const groupEntrySchema = z.object({
  columnGroup: columnGroupSchema.optional(),
  columns: z.array(resultColumnSchema),
  selectedIntervals: z.array(z.string()).optional(),
  join: z.boolean().optional(),
});

const sortEntrySchema = z.object({
  column: z.string(),
  order: z.enum(["ASCENDING", "DESCENDING"]).default("ASCENDING"),
});

const lookupSchema = z.object({
  uuid: z.string(),
  filter: z.array(filterNodeSchema).optional(),
  group: z.array(groupEntrySchema).optional(),
  sort: z.array(sortEntrySchema).optional(),
});

export function parseLookup(raw: unknown): DataSetLookup {
  const parsed = lookupSchema.parse(raw);
  const ops = [];

  if (parsed.filter && parsed.filter.length > 0) {
    ops.push(parseFilterOp(parsed.filter));
  }

  if (parsed.group) {
    for (const entry of parsed.group) {
      ops.push(parseGroupOp(entry));
    }
  }

  if (parsed.sort && parsed.sort.length > 0) {
    ops.push(parseSortOp(parsed.sort));
  }

  return createLookup(parsed.uuid as DataSetId, ops);
}

function parseFilterOp(nodes: FilterNodeInput[]): FilterOp {
  const expressions = nodes.map(parseFilterNode);
  return { type: "filter", expressions };
}

function parseFilterNode(node: FilterNodeInput): FilterExpression {
  if ("column" in node) {
    return {
      type: "unresolved",
      columnId: node.column as ColumnId,
      fn: node.function as CoreFunctionType,
      args: node.args.map(String),
    };
  }
  if ("and" in node) {
    return { type: "and", children: node.and.map(parseFilterNode) };
  }
  if ("or" in node) {
    return { type: "or", children: node.or.map(parseFilterNode) };
  }
  if ("not" in node) {
    return { type: "not", child: parseFilterNode(node.not) };
  }
  throw new Error("Unreachable: invalid filter node");
}

function parseGroupOp(entry: z.infer<typeof groupEntrySchema>): GroupOp {
  const groupingKey = entry.columnGroup ? parseGroupingKey(entry.columnGroup) : null;
  const keySourceId = entry.columnGroup?.source;

  const columns: ResultColumn[] = entry.columns.map((col) => {
    const sourceId = col.source as ColumnId;
    const columnId = (col.column ?? col.source) as ColumnId;

    if (keySourceId && col.source === keySourceId && !col.function) {
      return { kind: "key" as const, sourceId, columnId };
    }

    if (col.function) {
      const fn = parseAggregation(col.function, col.separator);
      return { kind: "aggregate" as const, sourceId, columnId, fn };
    }

    return { kind: "select" as const, sourceId, columnId };
  });

  return {
    type: "group",
    groupingKey,
    columns,
    selectedIntervals: entry.selectedIntervals,
    join: entry.join,
  };
}

function parseGroupingKey(cg: z.infer<typeof columnGroupSchema>): GroupingKey {
  const sourceId = cg.source as ColumnId;
  const columnId = (cg.column ?? cg.source) as ColumnId;

  let strategy: GroupStrategy;
  switch (cg.strategy) {
    case "distinct":
      strategy = { mode: "distinct" };
      break;
    case "fixedCalendar":
      strategy = { mode: "fixedCalendar", unit: (cg.unit ?? "MONTH") as FixedCalendarUnit };
      break;
    case "dynamicRange":
      strategy = { mode: "dynamicRange", preferredUnit: cg.unit as DateIntervalType | undefined };
      break;
    case "dynamic":
      strategy = { mode: "dynamic", preferredUnit: cg.unit as DateIntervalType | undefined };
      break;
  }

  return {
    sourceId,
    columnId,
    strategy,
    maxIntervals: cg.maxIntervals,
    emptyIntervals: cg.emptyIntervals,
    ascendingOrder: cg.ascendingOrder,
    firstMonthOfYear: cg.firstMonthOfYear as Month | undefined,
    firstDayOfWeek: cg.firstDayOfWeek as DayOfWeek | undefined,
  };
}

function parseAggregation(fn: string, separator?: string): Aggregation {
  switch (fn) {
    case "COUNT": return { fn: "COUNT" };
    case "DISTINCT": return { fn: "DISTINCT" };
    case "SUM": return { fn: "SUM" };
    case "AVERAGE": return { fn: "AVERAGE" };
    case "MEDIAN": return { fn: "MEDIAN" };
    case "MIN": return { fn: "MIN" };
    case "MAX": return { fn: "MAX" };
    case "JOIN": return { fn: "JOIN", separator: separator ?? ", " };
    default: throw new Error(`Unknown aggregation: ${fn}`);
  }
}

function parseSortOp(entries: z.infer<typeof sortEntrySchema>[]): SortOp {
  const columns: SortColumn[] = entries.map((e) => ({
    columnId: e.column as ColumnId,
    order: e.order,
  }));
  return { type: "sort", columns };
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- lookup-parser`
Expected: PASS.

- [ ] **Step 5: Run full test suite**

Run: `yarn workspace @melviz/core run test`
Expected: All tests pass — existing + all new tests.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/core/src/dataset/lookup-parser.ts packages/core/src/dataset/lookup-parser.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat: YAML lookup parser with full filter expression tree  Closes #5"
```

---

### Self-Review

**1. Spec coverage:** Every spec section maps to a task:
- §1 (bridge) → Task 6
- §2 (evaluator) → Task 7
- §3 (DataSetLookup) → Task 8
- §4 (constraints) → Task 9
- §5 (YAML parser) → Task 11
- §6 (parameterized tree, unresolved, resolution, DateFilter, compatibility matrix, lifecycle) → Tasks 1, 2, 10
- §7 (errors) → Tasks 4, 5
- §8 (compareValues fix) → Task 3
- §9 (file organization) → reflected across all tasks
- §10 (testing) → covered in each task

**2. Placeholder scan:** No TBDs, TODOs, or vague "handle edge cases." Every step has code.

**3. Type consistency:** Verified: `FilterExprTree<Leaf>`, `ResolvedLeaf`, `UnresolvedLeaf`, `ResolvedFilterExpression`, `FilterExpression`, `ResolvedFilterOp`, `FilterOp`, `ResolvedDataSetOp`, `DataSetOp` — used consistently across Tasks 1, 2, 8, 10, 11. `ExpressionError` / `ExpressionErrorCode` — defined in Task 5, consumed in Tasks 6, 7. `DataSetLookup` / `createLookup` — defined in Task 8, consumed in Tasks 9, 11. `resolveFilterTypes` — defined in Task 10, tested there.
