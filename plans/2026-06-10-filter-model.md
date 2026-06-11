# Filter Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the filter model — null CellValue, all 13 CoreFunctionType filter operations, recursive FilterExpression (AND/OR/NOT), TimeFrame parser, and filter evaluation engine.

**Architecture:** Five files: `types.ts` (null variant + nullable wire format), `conversion.ts` (null handling), `filter.ts` (type definitions), `timeframe.ts` (TimeFrame/TimeInstant types + parser + resolution), `filter-eval.ts` (applyFilter + evaluators + LIKE_TO regex). TDD throughout — tests first, then implementation, then commit.

**Tech Stack:** TypeScript 5.6+ (strict mode, `noUncheckedIndexedAccess`), Vitest, no browser APIs, UTC-only date arithmetic.

**Spec:** `docs/superpowers/specs/2026-06-10-filter-model-design.md`

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `packages/core/src/dataset/types.ts` | Modify | Add `"NULL"` CellValue variant, change `DataSet.data` to `(string \| null)[][]` |
| `packages/core/src/dataset/conversion.ts` | Modify | Handle null/undefined raw values, serialize null cells |
| `packages/core/src/dataset/conversion.test.ts` | Modify | Add null cell tests, fix broken tests from type changes |
| `packages/core/src/dataset/filter.ts` | Create | CoreFunctionType, NumericFilter, StringFilter, DateFilter, FilterExpression, FilterOp |
| `packages/core/src/dataset/timeframe.ts` | Create | DateIntervalType, Month, TruncationUnit, OffsetUnit, TimeFrame, TimeInstant, TimeOffset, parseTimeFrame, resolveTimeFrame, resolveInstant |
| `packages/core/src/dataset/timeframe.test.ts` | Create | Tests for parsing and resolution |
| `packages/core/src/dataset/filter-eval.ts` | Create | applyFilter, evaluateExpression, evaluateNumericFilter, evaluateStringFilter, evaluateDateFilter, compileLikePattern, preResolveTimeFrames |
| `packages/core/src/dataset/filter-eval.test.ts` | Create | Tests for all 13 filter functions, null semantics, logical composition, LIKE_TO, TIME_FRAME |

---

### Task 1: Null CellValue Variant and Wire Format

**Files:**
- Modify: `packages/core/src/dataset/types.ts`
- Modify: `packages/core/src/dataset/conversion.ts`
- Modify: `packages/core/src/dataset/conversion.test.ts`

- [ ] **Step 1: Write failing tests for null cell handling**

Add to `packages/core/src/dataset/conversion.test.ts`:

```typescript
it("produces NULL cell for undefined raw value (short row)", () => {
  const ds: DataSet = {
    columns: [
      col("a", "A", ColumnType.TEXT),
      col("b", "B", ColumnType.NUMBER),
    ],
    data: [["hello"]],  // row is shorter than columns — b is undefined
  };

  const result = toTypedDataSet(ds);
  const cell = result.rows[0]!.cell("b" as ColumnId);
  expect(cell.type).toBe("NULL");
});

it("produces NULL cell for explicit null in data array", () => {
  const ds: DataSet = {
    columns: [col("x", "X", ColumnType.TEXT)],
    data: [[null]],
  };

  const result = toTypedDataSet(ds);
  expect(result.rows[0]!.cell("x" as ColumnId).type).toBe("NULL");
});

it("preserves empty string as valid TEXT value, not null", () => {
  const ds: DataSet = {
    columns: [col("x", "X", ColumnType.TEXT)],
    data: [[""]],
  };

  const result = toTypedDataSet(ds);
  const cell = result.rows[0]!.cell("x" as ColumnId);
  expect(cell.type).toBe(ColumnType.TEXT);
  expect(cell.value).toBe("");
});

it("text() throws on NULL cell", () => {
  const ds: DataSet = {
    columns: [col("x", "X", ColumnType.TEXT)],
    data: [[null]],
  };

  const result = toTypedDataSet(ds);
  expect(() => result.rows[0]!.text("x" as ColumnId)).toThrow();
});

it("serializes NULL cell as null in wire format", () => {
  const ds: DataSet = {
    columns: [col("x", "X", ColumnType.TEXT)],
    data: [[null]],
  };

  const typed = toTypedDataSet(ds);
  const wire = toWireDataSet(typed);
  expect(wire.data[0]![0]).toBeNull();
});

it("round-trips null cells through toTypedDataSet → toWireDataSet", () => {
  const ds: DataSet = {
    columns: [
      col("a", "A", ColumnType.TEXT),
      col("b", "B", ColumnType.NUMBER),
    ],
    data: [["hello", null], [null, "42"]],
  };

  const wire = toWireDataSet(toTypedDataSet(ds));
  expect(wire.data[0]![0]).toBe("hello");
  expect(wire.data[0]![1]).toBeNull();
  expect(wire.data[1]![0]).toBeNull();
  expect(wire.data[1]![1]).toBe("42");
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- conversion.test`
Expected: Multiple failures — `"NULL"` variant doesn't exist, `DataSet.data` doesn't accept `null`.

- [ ] **Step 3: Update types.ts**

In `packages/core/src/dataset/types.ts`, add the `"NULL"` variant to `CellValue` and change `DataSet.data`:

```typescript
export type CellValue =
  | { readonly type: ColumnType.TEXT; readonly value: string }
  | { readonly type: ColumnType.NUMBER; readonly value: number }
  | { readonly type: ColumnType.DATE; readonly value: Date }
  | { readonly type: ColumnType.LABEL; readonly value: string }
  | { readonly type: "NULL" };

export interface DataSet {
  readonly columns: readonly Column[];
  readonly data: readonly (readonly (string | null)[])[];
}
```

- [ ] **Step 4: Update conversion.ts — toTypedDataSet null handling**

In `packages/core/src/dataset/conversion.ts`, change the cell parsing loop in `toTypedDataSet`:

```typescript
// Replace:
const rawValue = rawRow[colIdx] ?? "";
cells.push(parseCell(rawValue, column, rowIdx));

// With:
const rawValue = rawRow[colIdx];
if (rawValue === undefined || rawValue === null) {
  cells.push({ type: "NULL" as const });
} else {
  cells.push(parseCell(rawValue, column, rowIdx));
}
```

- [ ] **Step 5: Update conversion.ts — cellToString null handling**

Change the `cellToString` function return type and add the NULL case:

```typescript
function cellToString(cell: CellValue): string | null {
  switch (cell.type) {
    case ColumnType.TEXT:
    case ColumnType.LABEL:
      return cell.value;
    case ColumnType.NUMBER:
      return String(cell.value);
    case ColumnType.DATE:
      return cell.value.toISOString();
    case "NULL":
      return null;
  }
}
```

Update `toWireDataSet` return type to match the new `DataSet` interface:

```typescript
export function toWireDataSet(ds: TypedDataSet): DataSet {
  const data: (string | null)[][] = [];

  for (const row of ds.rows) {
    const rawRow: (string | null)[] = [];
    for (const cell of row.cells) {
      rawRow.push(cellToString(cell));
    }
    data.push(rawRow);
  }

  return { columns: ds.columns, data };
}
```

- [ ] **Step 6: Run all tests to verify they pass**

Run: `yarn workspace @melviz/core run test`
Expected: All tests pass (new null tests + existing tests).

- [ ] **Step 7: Type-check**

Run: `yarn workspace @melviz/core run build`
Expected: No type errors.

- [ ] **Step 8: Commit**

```
git add packages/core/src/dataset/types.ts packages/core/src/dataset/conversion.ts packages/core/src/dataset/conversion.test.ts
git commit -m "feat: add NULL CellValue variant and nullable wire format"
```

---

### Task 2: Filter Type Definitions

**Files:**
- Create: `packages/core/src/dataset/filter.ts`

- [ ] **Step 1: Create filter.ts with all type definitions**

Create `packages/core/src/dataset/filter.ts`:

```typescript
import type { ColumnId } from "./types.js";
import type { TimeFrame } from "./timeframe.js";

export type CoreFunctionType =
  | "IS_NULL" | "NOT_NULL"
  | "EQUALS_TO" | "NOT_EQUALS_TO"
  | "LIKE_TO"
  | "GREATER_THAN" | "GREATER_OR_EQUALS_TO"
  | "LOWER_THAN" | "LOWER_OR_EQUALS_TO"
  | "BETWEEN"
  | "TIME_FRAME"
  | "IN" | "NOT_IN";

type NullFilter =
  | { readonly fn: "IS_NULL" }
  | { readonly fn: "NOT_NULL" };

export type NumericFilter =
  | NullFilter
  | { readonly fn: "EQUALS_TO"; readonly value: number }
  | { readonly fn: "NOT_EQUALS_TO"; readonly value: number }
  | { readonly fn: "GREATER_THAN"; readonly value: number }
  | { readonly fn: "GREATER_OR_EQUALS_TO"; readonly value: number }
  | { readonly fn: "LOWER_THAN"; readonly value: number }
  | { readonly fn: "LOWER_OR_EQUALS_TO"; readonly value: number }
  | { readonly fn: "BETWEEN"; readonly low: number; readonly high: number }
  | { readonly fn: "IN"; readonly values: readonly number[] }
  | { readonly fn: "NOT_IN"; readonly values: readonly number[] };

export type StringFilter =
  | NullFilter
  | { readonly fn: "EQUALS_TO"; readonly value: string }
  | { readonly fn: "NOT_EQUALS_TO"; readonly value: string }
  | { readonly fn: "GREATER_THAN"; readonly value: string }
  | { readonly fn: "GREATER_OR_EQUALS_TO"; readonly value: string }
  | { readonly fn: "LOWER_THAN"; readonly value: string }
  | { readonly fn: "LOWER_OR_EQUALS_TO"; readonly value: string }
  | { readonly fn: "BETWEEN"; readonly low: string; readonly high: string }
  | { readonly fn: "LIKE_TO"; readonly pattern: string; readonly caseSensitive: boolean }
  | { readonly fn: "IN"; readonly values: readonly string[] }
  | { readonly fn: "NOT_IN"; readonly values: readonly string[] };

export type DateFilter =
  | NullFilter
  | { readonly fn: "EQUALS_TO"; readonly value: Date }
  | { readonly fn: "NOT_EQUALS_TO"; readonly value: Date }
  | { readonly fn: "GREATER_THAN"; readonly value: Date }
  | { readonly fn: "GREATER_OR_EQUALS_TO"; readonly value: Date }
  | { readonly fn: "LOWER_THAN"; readonly value: Date }
  | { readonly fn: "LOWER_OR_EQUALS_TO"; readonly value: Date }
  | { readonly fn: "BETWEEN"; readonly low: Date; readonly high: Date }
  | { readonly fn: "TIME_FRAME"; readonly timeFrame: TimeFrame };

export type FilterExpression =
  | { readonly type: "numeric"; readonly columnId: ColumnId; readonly filter: NumericFilter }
  | { readonly type: "string"; readonly columnId: ColumnId; readonly filter: StringFilter }
  | { readonly type: "date"; readonly columnId: ColumnId; readonly filter: DateFilter }
  | { readonly type: "and"; readonly children: readonly FilterExpression[] }
  | { readonly type: "or"; readonly children: readonly FilterExpression[] }
  | { readonly type: "not"; readonly child: FilterExpression };

export interface FilterOp {
  readonly type: "filter";
  readonly expressions: readonly FilterExpression[];
}
```

- [ ] **Step 2: Type-check**

Run: `yarn workspace @melviz/core run build`
Expected: Fails — `timeframe.ts` doesn't exist yet. This is expected; the import will resolve after Task 3. For now, verify no syntax errors by inspecting TypeScript output. Alternatively, create a stub `timeframe.ts` with just the `TimeFrame` interface:

Create `packages/core/src/dataset/timeframe.ts` (stub — will be fully implemented in Task 3):

```typescript
export interface TimeFrame {
  readonly from: TimeInstant;
  readonly to: TimeInstant;
}

export type TimeInstant =
  | { readonly mode: "now"; readonly offset?: TimeOffset }
  | { readonly mode: "begin"; readonly unit: TruncationUnit; readonly firstMonthOfYear?: Month; readonly offset?: TimeOffset }
  | { readonly mode: "end"; readonly unit: TruncationUnit; readonly firstMonthOfYear?: Month; readonly offset?: TimeOffset }
  | { readonly mode: "relative"; readonly offset: TimeOffset };

export interface TimeOffset {
  readonly amount: number;
  readonly unit: OffsetUnit;
}

export type DateIntervalType =
  | "MILLISECOND" | "HUNDRETH" | "TENTH"
  | "SECOND" | "MINUTE" | "HOUR"
  | "DAY" | "DAY_OF_WEEK" | "WEEK"
  | "MONTH" | "QUARTER" | "YEAR"
  | "DECADE" | "CENTURY" | "MILLENIUM";

export type TruncationUnit =
  | "MINUTE" | "HOUR" | "DAY"
  | "MONTH" | "QUARTER" | "YEAR"
  | "DECADE" | "CENTURY" | "MILLENIUM";

export type OffsetUnit =
  | "SECOND" | "MINUTE" | "HOUR"
  | "DAY" | "WEEK"
  | "MONTH" | "QUARTER" | "YEAR"
  | "DECADE" | "CENTURY" | "MILLENIUM";

export type Month =
  | "JANUARY" | "FEBRUARY" | "MARCH" | "APRIL"
  | "MAY" | "JUNE" | "JULY" | "AUGUST"
  | "SEPTEMBER" | "OCTOBER" | "NOVEMBER" | "DECEMBER";
```

Run: `yarn workspace @melviz/core run build`
Expected: Clean compile.

- [ ] **Step 3: Commit**

```
git add packages/core/src/dataset/filter.ts packages/core/src/dataset/timeframe.ts
git commit -m "feat: filter type definitions and timeframe type stubs"
```

---

### Task 3: TimeFrame Parser and Resolution

**Files:**
- Modify: `packages/core/src/dataset/timeframe.ts` (replace stub with full implementation)
- Create: `packages/core/src/dataset/timeframe.test.ts`

- [ ] **Step 1: Write failing tests for TimeFrame parsing**

Create `packages/core/src/dataset/timeframe.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { parseTimeFrame, resolveTimeFrame, resolveInstant } from "./timeframe.js";
import type { TimeInstant } from "./timeframe.js";

describe("parseTimeFrame", () => {
  it("parses 'now till 10second'", () => {
    const tf = parseTimeFrame("now till 10second");
    expect(tf.from).toEqual({ mode: "now" });
    expect(tf.to).toEqual({ mode: "relative", offset: { amount: 10, unit: "SECOND" } });
  });

  it("parses 'begin[year] till now'", () => {
    const tf = parseTimeFrame("begin[year] till now");
    expect(tf.from).toEqual({ mode: "begin", unit: "YEAR" });
    expect(tf.to).toEqual({ mode: "now" });
  });

  it("parses 'begin[year March] -1year till now'", () => {
    const tf = parseTimeFrame("begin[year March] -1year till now");
    expect(tf.from).toEqual({
      mode: "begin",
      unit: "YEAR",
      firstMonthOfYear: "MARCH",
      offset: { amount: -1, unit: "YEAR" },
    });
    expect(tf.to).toEqual({ mode: "now" });
  });

  it("parses 'end[quarter] till 1quarter'", () => {
    const tf = parseTimeFrame("end[quarter] till 1quarter");
    expect(tf.from).toEqual({ mode: "end", unit: "QUARTER" });
    expect(tf.to).toEqual({ mode: "relative", offset: { amount: 1, unit: "QUARTER" } });
  });

  it("parses single instant without 'till' — pairs with now", () => {
    const tf = parseTimeFrame("begin[month]");
    expect(tf.from.mode).toBe("begin");
    expect(tf.to).toEqual({ mode: "now" });
  });

  it("parses 'now +2day'", () => {
    const tf = parseTimeFrame("now +2day");
    expect(tf.from).toEqual({ mode: "now", offset: { amount: 2, unit: "DAY" } });
    expect(tf.to).toEqual({ mode: "now" });
  });

  it("throws on empty string", () => {
    expect(() => parseTimeFrame("")).toThrow();
  });

  it("throws on invalid interval type", () => {
    expect(() => parseTimeFrame("begin[invalid] till now")).toThrow();
  });

  it("parses negative offset '-7day'", () => {
    const tf = parseTimeFrame("-7day");
    expect(tf.from).toEqual({ mode: "relative", offset: { amount: -7, unit: "DAY" } });
    expect(tf.to).toEqual({ mode: "now" });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- timeframe.test`
Expected: FAIL — `parseTimeFrame` not exported.

- [ ] **Step 3: Implement parseTimeFrame, parseTimeInstant, parseTimeOffset**

Replace the stub `packages/core/src/dataset/timeframe.ts` with the full implementation. Keep all type definitions already in the stub. Add:

```typescript
export function parseTimeOffset(expr: string): TimeOffset {
  const trimmed = expr.trim();
  if (trimmed.length === 0) throw new Error("Empty time offset expression");

  const isNegative = trimmed.startsWith("-");
  const isPositive = trimmed.startsWith("+");
  let i = isNegative || isPositive ? 1 : 0;

  let numberStr = "";
  for (; i < trimmed.length; i++) {
    const ch = trimmed[i]!;
    if (ch >= "0" && ch <= "9") numberStr += ch;
    else break;
  }

  if (numberStr.length === 0) throw new Error(`Missing quantity in time offset: "${expr}"`);

  const unitStr = trimmed.substring(i).trim().toUpperCase();
  if (!isOffsetUnit(unitStr)) throw new Error(`Invalid offset unit: "${unitStr}"`);

  const amount = parseInt(numberStr, 10) * (isNegative ? -1 : 1);
  return { amount, unit: unitStr };
}

function parseTimeInstant(expr: string): TimeInstant {
  const trimmed = expr.trim().toLowerCase();
  if (trimmed.length === 0) throw new Error("Empty time instant expression");

  const isBegin = trimmed.startsWith("begin");
  const isEnd = trimmed.startsWith("end");

  if (!isBegin && !isEnd) {
    if (trimmed.startsWith("now")) {
      if (trimmed.length > 3) {
        const offsetExpr = trimmed.substring(3);
        return { mode: "now", offset: parseTimeOffset(offsetExpr) };
      }
      return { mode: "now" };
    }
    return { mode: "relative", offset: parseTimeOffset(trimmed) };
  }

  const bracketStart = trimmed.indexOf("[");
  const bracketEnd = trimmed.indexOf("]");
  if (bracketStart === -1 || bracketEnd === -1 || bracketStart >= bracketEnd) {
    throw new Error(`Missing brackets in time instant: "${expr}"`);
  }

  const bracketContent = trimmed.substring(bracketStart + 1, bracketEnd);
  const parts = bracketContent.split(/\s+/);
  if (parts.length > 2) {
    throw new Error(`Too many settings in brackets: "${expr}"`);
  }

  const unitStr = parts[0]!.toUpperCase();
  if (!isTruncationUnit(unitStr)) {
    throw new Error(`Invalid truncation unit: "${unitStr}"`);
  }

  let firstMonthOfYear: Month | undefined;
  if (parts.length === 2) {
    const monthStr = parts[1]!.toUpperCase();
    if (!isMonth(monthStr)) {
      throw new Error(`Invalid month: "${monthStr}"`);
    }
    firstMonthOfYear = monthStr;
  }

  let offset: TimeOffset | undefined;
  const afterBracket = trimmed.substring(bracketEnd + 1).trim();
  if (afterBracket.length > 0) {
    offset = parseTimeOffset(afterBracket);
  }

  const base = {
    mode: isBegin ? "begin" as const : "end" as const,
    unit: unitStr,
    ...(firstMonthOfYear !== undefined && { firstMonthOfYear }),
    ...(offset !== undefined && { offset }),
  };
  return base;
}

export function parseTimeFrame(expr: string): TimeFrame {
  if (!expr || expr.trim().length === 0) {
    throw new Error("Empty time frame expression");
  }

  const lower = expr.toLowerCase().trim();
  const tillIndex = lower.indexOf("till");

  if (tillIndex === -1) {
    const instant = parseTimeInstant(lower);
    return { from: instant, to: { mode: "now" } };
  }

  const fromExpr = lower.substring(0, tillIndex);
  const toExpr = lower.substring(tillIndex + 4);
  return { from: parseTimeInstant(fromExpr), to: parseTimeInstant(toExpr) };
}
```

Add type guard helpers (private):

```typescript
const TRUNCATION_UNITS: ReadonlySet<string> = new Set([
  "MINUTE", "HOUR", "DAY", "MONTH", "QUARTER", "YEAR",
  "DECADE", "CENTURY", "MILLENIUM",
]);

const OFFSET_UNITS: ReadonlySet<string> = new Set([
  "SECOND", "MINUTE", "HOUR", "DAY", "WEEK",
  "MONTH", "QUARTER", "YEAR", "DECADE", "CENTURY", "MILLENIUM",
]);

const MONTHS: ReadonlySet<string> = new Set([
  "JANUARY", "FEBRUARY", "MARCH", "APRIL", "MAY", "JUNE",
  "JULY", "AUGUST", "SEPTEMBER", "OCTOBER", "NOVEMBER", "DECEMBER",
]);

function isTruncationUnit(s: string): s is TruncationUnit {
  return TRUNCATION_UNITS.has(s);
}

function isOffsetUnit(s: string): s is OffsetUnit {
  return OFFSET_UNITS.has(s);
}

function isMonth(s: string): s is Month {
  return MONTHS.has(s);
}
```

- [ ] **Step 4: Run parse tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- timeframe.test`
Expected: All parse tests pass.

- [ ] **Step 5: Write failing tests for TimeInstant resolution**

Add to `packages/core/src/dataset/timeframe.test.ts`:

```typescript
describe("resolveInstant", () => {
  const ref = new Date(Date.UTC(2024, 5, 15, 10, 30, 45, 500)); // 2024-06-15T10:30:45.500Z

  it("resolves 'now' to referenceDate", () => {
    const result = resolveInstant({ mode: "now" }, ref);
    expect(result.getTime()).toBe(ref.getTime());
  });

  it("resolves 'now +2day'", () => {
    const result = resolveInstant({ mode: "now", offset: { amount: 2, unit: "DAY" } }, ref);
    expect(result.toISOString()).toBe("2024-06-17T10:30:45.500Z");
  });

  it("resolves begin[year] — truncates to Jan 1", () => {
    const result = resolveInstant({ mode: "begin", unit: "YEAR" }, ref);
    expect(result.toISOString()).toBe("2024-01-01T00:00:00.000Z");
  });

  it("resolves begin[month] — truncates to 1st of month", () => {
    const result = resolveInstant({ mode: "begin", unit: "MONTH" }, ref);
    expect(result.toISOString()).toBe("2024-06-01T00:00:00.000Z");
  });

  it("resolves end[month] — first ms of next month", () => {
    const result = resolveInstant({ mode: "end", unit: "MONTH" }, ref);
    expect(result.toISOString()).toBe("2024-07-01T00:00:00.000Z");
  });

  it("resolves begin[day] — truncates to midnight", () => {
    const result = resolveInstant({ mode: "begin", unit: "DAY" }, ref);
    expect(result.toISOString()).toBe("2024-06-15T00:00:00.000Z");
  });

  it("resolves end[day] — first ms of next day", () => {
    const result = resolveInstant({ mode: "end", unit: "DAY" }, ref);
    expect(result.toISOString()).toBe("2024-06-16T00:00:00.000Z");
  });

  it("resolves begin[hour] — truncates to hour", () => {
    const result = resolveInstant({ mode: "begin", unit: "HOUR" }, ref);
    expect(result.toISOString()).toBe("2024-06-15T10:00:00.000Z");
  });

  it("resolves begin[minute] — truncates to minute", () => {
    const result = resolveInstant({ mode: "begin", unit: "MINUTE" }, ref);
    expect(result.toISOString()).toBe("2024-06-15T10:30:00.000Z");
  });

  it("resolves begin[quarter] — Q2 starts April", () => {
    const result = resolveInstant({ mode: "begin", unit: "QUARTER" }, ref);
    expect(result.toISOString()).toBe("2024-04-01T00:00:00.000Z");
  });

  it("resolves begin[year March] — fiscal year starting March", () => {
    const result = resolveInstant(
      { mode: "begin", unit: "YEAR", firstMonthOfYear: "MARCH" },
      ref,
    );
    expect(result.toISOString()).toBe("2024-03-01T00:00:00.000Z");
  });

  it("resolves begin[year March] when before March — goes to previous year", () => {
    const feb = new Date(Date.UTC(2024, 1, 15)); // 2024-02-15
    const result = resolveInstant(
      { mode: "begin", unit: "YEAR", firstMonthOfYear: "MARCH" },
      feb,
    );
    expect(result.toISOString()).toBe("2023-03-01T00:00:00.000Z");
  });

  it("resolves begin[year] -1year — previous year start", () => {
    const result = resolveInstant(
      { mode: "begin", unit: "YEAR", offset: { amount: -1, unit: "YEAR" } },
      ref,
    );
    expect(result.toISOString()).toBe("2023-01-01T00:00:00.000Z");
  });

  it("resolves relative offset '+5day'", () => {
    const result = resolveInstant(
      { mode: "relative", offset: { amount: 5, unit: "DAY" } },
      ref,
    );
    expect(result.toISOString()).toBe("2024-06-20T10:30:45.500Z");
  });

  it("resolves relative offset '-2month'", () => {
    const result = resolveInstant(
      { mode: "relative", offset: { amount: -2, unit: "MONTH" } },
      ref,
    );
    expect(result.toISOString()).toBe("2024-04-15T10:30:45.500Z");
  });

  it("offset +1month from Jan 31 → Feb 28 (non-leap)", () => {
    const jan31 = new Date(Date.UTC(2023, 0, 31));
    const result = resolveInstant(
      { mode: "relative", offset: { amount: 1, unit: "MONTH" } },
      jan31,
    );
    expect(result.getUTCMonth()).toBe(1); // February
    expect(result.getUTCDate()).toBe(28);
  });

  it("offset +1week adds 7 days", () => {
    const result = resolveInstant(
      { mode: "relative", offset: { amount: 1, unit: "WEEK" } },
      ref,
    );
    expect(result.toISOString()).toBe("2024-06-22T10:30:45.500Z");
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- timeframe.test`
Expected: FAIL — `resolveInstant` not exported.

- [ ] **Step 7: Implement resolveInstant**

Add to `packages/core/src/dataset/timeframe.ts`:

```typescript
const MONTH_INDEX: Record<Month, number> = {
  JANUARY: 0, FEBRUARY: 1, MARCH: 2, APRIL: 3,
  MAY: 4, JUNE: 5, JULY: 6, AUGUST: 7,
  SEPTEMBER: 8, OCTOBER: 9, NOVEMBER: 10, DECEMBER: 11,
};

function applyOffset(date: Date, offset: TimeOffset): Date {
  const d = new Date(date.getTime());
  const { amount, unit } = offset;

  switch (unit) {
    case "MILLENIUM": d.setUTCFullYear(d.getUTCFullYear() + amount * 1000); break;
    case "CENTURY": d.setUTCFullYear(d.getUTCFullYear() + amount * 100); break;
    case "DECADE": d.setUTCFullYear(d.getUTCFullYear() + amount * 10); break;
    case "YEAR": d.setUTCFullYear(d.getUTCFullYear() + amount); break;
    case "QUARTER": d.setUTCMonth(d.getUTCMonth() + amount * 3); break;
    case "MONTH": d.setUTCMonth(d.getUTCMonth() + amount); break;
    case "WEEK": d.setUTCDate(d.getUTCDate() + amount * 7); break;
    case "DAY": d.setUTCDate(d.getUTCDate() + amount); break;
    case "HOUR": d.setUTCHours(d.getUTCHours() + amount); break;
    case "MINUTE": d.setUTCMinutes(d.getUTCMinutes() + amount); break;
    case "SECOND": d.setUTCSeconds(d.getUTCSeconds() + amount); break;
  }
  return d;
}

function truncate(date: Date, unit: TruncationUnit, mode: "begin" | "end", firstMonthOfYear?: Month): Date {
  const d = new Date(date.getTime());
  const firstMonth = firstMonthOfYear ? MONTH_INDEX[firstMonthOfYear] : 0;

  switch (unit) {
    case "MILLENIUM": {
      const base = Math.floor(d.getUTCFullYear() / 1000);
      const inc = mode === "end" ? 1 : 0;
      d.setUTCFullYear((base + inc) * 1000, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "CENTURY": {
      const base = Math.floor(d.getUTCFullYear() / 100);
      const inc = mode === "end" ? 1 : 0;
      d.setUTCFullYear((base + inc) * 100, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "DECADE": {
      const base = Math.floor(d.getUTCFullYear() / 10);
      const inc = mode === "end" ? 1 : 0;
      d.setUTCFullYear((base + inc) * 10, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "YEAR": {
      const month = d.getUTCMonth();
      let yearInc: number;
      if (mode === "begin") yearInc = month < firstMonth ? -1 : 0;
      else yearInc = month < firstMonth ? 0 : 1;
      d.setUTCFullYear(d.getUTCFullYear() + yearInc, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "QUARTER": {
      const month = d.getUTCMonth();
      const quarterFirstMonth = getQuarterFirstMonth(firstMonth, month);
      if (mode === "begin") {
        const yearInc = quarterFirstMonth > month ? -1 : 0;
        d.setUTCFullYear(d.getUTCFullYear() + yearInc);
        d.setUTCMonth(quarterFirstMonth, 1);
      } else {
        d.setUTCMonth(quarterFirstMonth + 3, 1);
      }
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "MONTH":
      d.setUTCDate(1);
      d.setUTCHours(0, 0, 0, 0);
      if (mode === "end") d.setUTCMonth(d.getUTCMonth() + 1);
      break;
    case "DAY":
      d.setUTCHours(0, 0, 0, 0);
      if (mode === "end") d.setUTCDate(d.getUTCDate() + 1);
      break;
    case "HOUR":
      d.setUTCMinutes(0, 0, 0);
      if (mode === "end") d.setUTCHours(d.getUTCHours() + 1);
      break;
    case "MINUTE":
      d.setUTCSeconds(0, 0);
      if (mode === "end") d.setUTCMinutes(d.getUTCMinutes() + 1);
      break;
  }
  return d;
}

function getQuarterFirstMonth(firstMonthOfYear: number, currentMonth: number): number {
  for (let q = 3; q >= 0; q--) {
    const qStart = (firstMonthOfYear + q * 3) % 12;
    if (currentMonth >= qStart) return qStart;
  }
  return (firstMonthOfYear + 9) % 12;
}

export function resolveInstant(instant: TimeInstant, referenceDate: Date): Date {
  switch (instant.mode) {
    case "now": {
      const d = new Date(referenceDate.getTime());
      return instant.offset ? applyOffset(d, instant.offset) : d;
    }
    case "begin":
    case "end": {
      const d = truncate(referenceDate, instant.unit, instant.mode, instant.firstMonthOfYear);
      return instant.offset ? applyOffset(d, instant.offset) : d;
    }
    case "relative":
      return applyOffset(new Date(referenceDate.getTime()), instant.offset);
  }
}
```

- [ ] **Step 8: Run resolution tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- timeframe.test`
Expected: All tests pass.

- [ ] **Step 9: Write failing tests for resolveTimeFrame**

Add to `packages/core/src/dataset/timeframe.test.ts`:

```typescript
describe("resolveTimeFrame", () => {
  const ref = new Date(Date.UTC(2024, 5, 15, 10, 30, 0)); // 2024-06-15T10:30:00Z

  it("resolves 'begin[year] till now'", () => {
    const tf = parseTimeFrame("begin[year] till now");
    const { from, to } = resolveTimeFrame(tf, ref);
    expect(from.toISOString()).toBe("2024-01-01T00:00:00.000Z");
    expect(to.toISOString()).toBe("2024-06-15T10:30:00.000Z");
  });

  it("resolves relative 'to' using resolved 'from' as start time", () => {
    const tf = parseTimeFrame("begin[month] till 10day");
    const { from, to } = resolveTimeFrame(tf, ref);
    expect(from.toISOString()).toBe("2024-06-01T00:00:00.000Z");
    expect(to.toISOString()).toBe("2024-06-11T00:00:00.000Z");
  });

  it("swaps from/to when from > to", () => {
    const tf = parseTimeFrame("end[year] till begin[year]");
    const { from, to } = resolveTimeFrame(tf, ref);
    expect(from < to).toBe(true);
  });

  it("single instant without 'till' — pairs with now, orders correctly", () => {
    const tf = parseTimeFrame("begin[month]");
    const { from, to } = resolveTimeFrame(tf, ref);
    expect(from.toISOString()).toBe("2024-06-01T00:00:00.000Z");
    expect(to.toISOString()).toBe("2024-06-15T10:30:00.000Z");
  });

  it("zero-width range is valid (equal instants)", () => {
    const tf = parseTimeFrame("now till now");
    const { from, to } = resolveTimeFrame(tf, ref);
    expect(from.getTime()).toBe(to.getTime());
  });
});
```

- [ ] **Step 10: Run tests to verify they pass (resolveTimeFrame already implemented in Step 7's resolveInstant)**

Wait — `resolveTimeFrame` needs to be implemented too. Add:

```typescript
export function resolveTimeFrame(
  tf: TimeFrame,
  referenceDate: Date,
): { from: Date; to: Date } {
  let from = resolveInstant(tf.from, referenceDate);
  let to = tf.to.mode === "relative"
    ? resolveInstant(tf.to, from)
    : resolveInstant(tf.to, referenceDate);

  if (from > to) [from, to] = [to, from];
  return { from, to };
}
```

Run: `yarn workspace @melviz/core run test -- timeframe.test`
Expected: All tests pass.

- [ ] **Step 11: Type-check**

Run: `yarn workspace @melviz/core run build`
Expected: No type errors.

- [ ] **Step 12: Commit**

```
git add packages/core/src/dataset/timeframe.ts packages/core/src/dataset/timeframe.test.ts
git commit -m "feat: TimeFrame parser and resolution with UTC date arithmetic"
```

---

### Task 4: Filter Evaluation — Numeric Filters

**Files:**
- Create: `packages/core/src/dataset/filter-eval.ts`
- Create: `packages/core/src/dataset/filter-eval.test.ts`

- [ ] **Step 1: Write failing tests for numeric filter evaluation**

Create `packages/core/src/dataset/filter-eval.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { applyFilter } from "./filter-eval.js";
import { toTypedDataSet } from "./conversion.js";
import type { DataSet, Column, ColumnId, TypedDataSet } from "./types.js";
import { ColumnType } from "./types.js";
import type { FilterOp, FilterExpression } from "./filter.js";

function col(id: string, name: string, type: ColumnType): Column {
  return { id: id as ColumnId, name, type };
}

function numericDataSet(): TypedDataSet {
  return toTypedDataSet({
    columns: [col("val", "Value", ColumnType.NUMBER)],
    data: [["10"], ["20"], ["30"], ["40"], ["50"]],
  });
}

function numericFilter(filter: FilterExpression["type"] extends "numeric" ? FilterExpression : never): FilterOp;
function numericFilter(expr: FilterExpression): FilterOp {
  return { type: "filter", expressions: [expr] };
}

function nf(filter: import("./filter.js").NumericFilter): FilterExpression {
  return { type: "numeric", columnId: "val" as ColumnId, filter };
}

describe("applyFilter — numeric", () => {
  it("EQUALS_TO filters to matching rows", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "EQUALS_TO", value: 30 })] });
    expect(result.rows).toHaveLength(1);
    expect(result.rows[0]!.number("val" as ColumnId)).toBe(30);
  });

  it("NOT_EQUALS_TO excludes matching rows", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "NOT_EQUALS_TO", value: 30 })] });
    expect(result.rows).toHaveLength(4);
  });

  it("GREATER_THAN filters correctly", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "GREATER_THAN", value: 30 })] });
    expect(result.rows).toHaveLength(2);
  });

  it("GREATER_OR_EQUALS_TO includes boundary", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "GREATER_OR_EQUALS_TO", value: 30 })] });
    expect(result.rows).toHaveLength(3);
  });

  it("LOWER_THAN filters correctly", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "LOWER_THAN", value: 30 })] });
    expect(result.rows).toHaveLength(2);
  });

  it("LOWER_OR_EQUALS_TO includes boundary", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "LOWER_OR_EQUALS_TO", value: 30 })] });
    expect(result.rows).toHaveLength(3);
  });

  it("BETWEEN inclusive both ends", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "BETWEEN", low: 20, high: 40 })] });
    expect(result.rows).toHaveLength(3);
  });

  it("IN matches any value in set", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "IN", values: [10, 30, 50] })] });
    expect(result.rows).toHaveLength(3);
  });

  it("NOT_IN excludes all values in set", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "NOT_IN", values: [10, 30, 50] })] });
    expect(result.rows).toHaveLength(2);
  });

  it("IS_NULL returns no rows (no nulls in dataset)", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "IS_NULL" })] });
    expect(result.rows).toHaveLength(0);
  });

  it("NOT_NULL returns all rows (no nulls in dataset)", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [nf({ fn: "NOT_NULL" })] });
    expect(result.rows).toHaveLength(5);
  });
});

describe("applyFilter — null semantics", () => {
  function dataSetWithNull(): TypedDataSet {
    return toTypedDataSet({
      columns: [col("val", "Value", ColumnType.NUMBER)],
      data: [["10"], [null], ["30"]],
    });
  }

  it("IS_NULL matches null cells", () => {
    const result = applyFilter(dataSetWithNull(), { type: "filter", expressions: [nf({ fn: "IS_NULL" })] });
    expect(result.rows).toHaveLength(1);
  });

  it("NOT_NULL excludes null cells", () => {
    const result = applyFilter(dataSetWithNull(), { type: "filter", expressions: [nf({ fn: "NOT_NULL" })] });
    expect(result.rows).toHaveLength(2);
  });

  it("EQUALS_TO returns false for null (not true)", () => {
    const result = applyFilter(dataSetWithNull(), { type: "filter", expressions: [nf({ fn: "EQUALS_TO", value: 10 })] });
    expect(result.rows).toHaveLength(1);
  });

  it("NOT_EQUALS_TO returns false for null (fixes Java bug)", () => {
    const result = applyFilter(dataSetWithNull(), { type: "filter", expressions: [nf({ fn: "NOT_EQUALS_TO", value: 10 })] });
    expect(result.rows).toHaveLength(1); // only row with 30, NOT the null row
  });

  it("LOWER_THAN returns false for null (fixes Java bug)", () => {
    const result = applyFilter(dataSetWithNull(), { type: "filter", expressions: [nf({ fn: "LOWER_THAN", value: 100 })] });
    expect(result.rows).toHaveLength(2); // 10 and 30, NOT null
  });

  it("NOT_IN returns false for null (fixes Java bug)", () => {
    const result = applyFilter(dataSetWithNull(), { type: "filter", expressions: [nf({ fn: "NOT_IN", values: [999] })] });
    expect(result.rows).toHaveLength(2); // 10 and 30, NOT null
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: FAIL — `filter-eval.ts` doesn't exist.

- [ ] **Step 3: Implement applyFilter and evaluateNumericFilter**

Create `packages/core/src/dataset/filter-eval.ts`:

```typescript
import type { CellValue, TypedDataSet, TypedRow } from "./types.js";
import type { FilterOp, FilterExpression, NumericFilter, StringFilter, DateFilter } from "./filter.js";
import type { TimeFrame } from "./timeframe.js";
import { resolveTimeFrame } from "./timeframe.js";

type ResolvedTimeFrames = Map<TimeFrame, { from: Date; to: Date }>;

export function applyFilter(
  ds: TypedDataSet,
  op: FilterOp,
  referenceDate?: Date,
): TypedDataSet {
  const ref = referenceDate ?? new Date();
  const resolved = preResolveTimeFrames(op.expressions, ref);
  const rows = ds.rows.filter((row) =>
    op.expressions.every((expr) => evaluateExpression(row, expr, resolved)),
  );
  return { columns: ds.columns, rows };
}

function preResolveTimeFrames(
  expressions: readonly FilterExpression[],
  referenceDate: Date,
): ResolvedTimeFrames {
  const resolved: ResolvedTimeFrames = new Map();
  walkExpressions(expressions, (expr) => {
    if (expr.type === "date" && expr.filter.fn === "TIME_FRAME") {
      const tf = expr.filter.timeFrame;
      if (!resolved.has(tf)) {
        resolved.set(tf, resolveTimeFrame(tf, referenceDate));
      }
    }
  });
  return resolved;
}

function walkExpressions(
  expressions: readonly FilterExpression[],
  visitor: (expr: FilterExpression) => void,
): void {
  for (const expr of expressions) {
    visitor(expr);
    switch (expr.type) {
      case "and": walkExpressions(expr.children, visitor); break;
      case "or": walkExpressions(expr.children, visitor); break;
      case "not": walkExpressions([expr.child], visitor); break;
    }
  }
}

function evaluateExpression(
  row: TypedRow,
  expr: FilterExpression,
  resolved: ResolvedTimeFrames,
): boolean {
  switch (expr.type) {
    case "numeric":
      return evaluateNumericFilter(row.cell(expr.columnId), expr.filter);
    case "string":
      return evaluateStringFilter(row.cell(expr.columnId), expr.filter);
    case "date":
      return evaluateDateFilter(row.cell(expr.columnId), expr.filter, resolved);
    case "and":
      return expr.children.every((c) => evaluateExpression(row, c, resolved));
    case "or":
      return expr.children.some((c) => evaluateExpression(row, c, resolved));
    case "not":
      return !evaluateExpression(row, expr.child, resolved);
  }
}

function evaluateNumericFilter(cell: CellValue, filter: NumericFilter): boolean {
  if (filter.fn === "IS_NULL") return cell.type === "NULL";
  if (filter.fn === "NOT_NULL") return cell.type !== "NULL";
  if (cell.type === "NULL") return false;

  const value = cell.type === "NUMBER" ? cell.value : NaN;

  switch (filter.fn) {
    case "EQUALS_TO": return value === filter.value;
    case "NOT_EQUALS_TO": return value !== filter.value;
    case "GREATER_THAN": return value > filter.value;
    case "GREATER_OR_EQUALS_TO": return value >= filter.value;
    case "LOWER_THAN": return value < filter.value;
    case "LOWER_OR_EQUALS_TO": return value <= filter.value;
    case "BETWEEN": return value >= filter.low && value <= filter.high;
    case "IN": return filter.values.includes(value);
    case "NOT_IN": return !filter.values.includes(value);
  }
}

function evaluateStringFilter(cell: CellValue, filter: StringFilter): boolean {
  // Stub — implemented in Task 5
  return false;
}

function evaluateDateFilter(cell: CellValue, filter: DateFilter, resolved: ResolvedTimeFrames): boolean {
  // Stub — implemented in Task 6
  return false;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: All numeric and null semantics tests pass.

- [ ] **Step 5: Type-check**

Run: `yarn workspace @melviz/core run build`
Expected: No type errors.

- [ ] **Step 6: Commit**

```
git add packages/core/src/dataset/filter-eval.ts packages/core/src/dataset/filter-eval.test.ts
git commit -m "feat: applyFilter with numeric filter evaluation and null semantics"
```

---

### Task 5: String Filter Evaluation + LIKE_TO

**Files:**
- Modify: `packages/core/src/dataset/filter-eval.ts`
- Modify: `packages/core/src/dataset/filter-eval.test.ts`

- [ ] **Step 1: Write failing tests for string filter evaluation**

Add to `packages/core/src/dataset/filter-eval.test.ts`:

```typescript
function stringDataSet(): TypedDataSet {
  return toTypedDataSet({
    columns: [col("name", "Name", ColumnType.LABEL)],
    data: [["Alice"], ["Bob"], ["Charlie"], ["David"]],
  });
}

function sf(filter: import("./filter.js").StringFilter): FilterExpression {
  return { type: "string", columnId: "name" as ColumnId, filter };
}

describe("applyFilter — string", () => {
  it("EQUALS_TO matches exact string", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "EQUALS_TO", value: "Bob" })] });
    expect(result.rows).toHaveLength(1);
    expect(result.rows[0]!.text("name" as ColumnId)).toBe("Bob");
  });

  it("NOT_EQUALS_TO excludes matching string", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "NOT_EQUALS_TO", value: "Bob" })] });
    expect(result.rows).toHaveLength(3);
  });

  it("GREATER_THAN uses Unicode code point order", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "GREATER_THAN", value: "Charlie" })] });
    expect(result.rows).toHaveLength(1); // David
  });

  it("LOWER_THAN uses Unicode code point order", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "LOWER_THAN", value: "Bob" })] });
    expect(result.rows).toHaveLength(1); // Alice
  });

  it("BETWEEN inclusive on string range", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "BETWEEN", low: "Bob", high: "David" })] });
    expect(result.rows).toHaveLength(3); // Bob, Charlie, David
  });

  it("IN matches any string in set", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "IN", values: ["Alice", "Charlie"] })] });
    expect(result.rows).toHaveLength(2);
  });

  it("NOT_IN excludes all strings in set", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "NOT_IN", values: ["Alice", "Charlie"] })] });
    expect(result.rows).toHaveLength(2);
  });
});

describe("applyFilter — LIKE_TO", () => {
  it("% matches zero or more characters", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "LIKE_TO", pattern: "%li%", caseSensitive: true })] });
    expect(result.rows).toHaveLength(2); // Alice, Charlie
  });

  it("_ matches exactly one character", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "LIKE_TO", pattern: "Bo_", caseSensitive: true })] });
    expect(result.rows).toHaveLength(1); // Bob
  });

  it("case insensitive matching", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "LIKE_TO", pattern: "bob", caseSensitive: false })] });
    expect(result.rows).toHaveLength(1); // Bob
  });

  it("exact match without wildcards (anchored)", () => {
    const result = applyFilter(stringDataSet(), { type: "filter", expressions: [sf({ fn: "LIKE_TO", pattern: "Ali", caseSensitive: true })] });
    expect(result.rows).toHaveLength(0); // "Ali" does not match "Alice"
  });

  it("pattern with literal dot is escaped", () => {
    const ds = toTypedDataSet({
      columns: [col("v", "V", ColumnType.TEXT)],
      data: [["a.b"], ["axb"]],
    });
    const result = applyFilter(ds, { type: "filter", expressions: [
      { type: "string", columnId: "v" as ColumnId, filter: { fn: "LIKE_TO", pattern: "a.b", caseSensitive: true } },
    ]});
    expect(result.rows).toHaveLength(1); // only "a.b", not "axb"
  });

  it("bracket expression [charlist] passes through", () => {
    const ds = toTypedDataSet({
      columns: [col("v", "V", ColumnType.TEXT)],
      data: [["cat"], ["cut"], ["cot"], ["cit"]],
    });
    const result = applyFilter(ds, { type: "filter", expressions: [
      { type: "string", columnId: "v" as ColumnId, filter: { fn: "LIKE_TO", pattern: "c[ao]t", caseSensitive: true } },
    ]});
    expect(result.rows).toHaveLength(2); // cat, cot
  });

  it("% and _ inside brackets are not replaced", () => {
    const ds = toTypedDataSet({
      columns: [col("v", "V", ColumnType.TEXT)],
      data: [["a%b"], ["a_b"], ["axb"]],
    });
    const result = applyFilter(ds, { type: "filter", expressions: [
      { type: "string", columnId: "v" as ColumnId, filter: { fn: "LIKE_TO", pattern: "a[%_]b", caseSensitive: true } },
    ]});
    expect(result.rows).toHaveLength(2); // a%b, a_b — NOT axb
  });

  it("LIKE_TO returns false for null cell", () => {
    const ds = toTypedDataSet({
      columns: [col("v", "V", ColumnType.TEXT)],
      data: [[null]],
    });
    const result = applyFilter(ds, { type: "filter", expressions: [
      { type: "string", columnId: "v" as ColumnId, filter: { fn: "LIKE_TO", pattern: "%", caseSensitive: true } },
    ]});
    expect(result.rows).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: String and LIKE_TO tests fail (stub returns false).

- [ ] **Step 3: Implement compileLikePattern and evaluateStringFilter**

Add to `packages/core/src/dataset/filter-eval.ts`:

```typescript
function compileLikePattern(pattern: string): string {
  let result = "";
  let inBracket = false;

  for (let i = 0; i < pattern.length; i++) {
    const ch = pattern[i]!;

    if (inBracket) {
      result += ch;
      if (ch === "]") inBracket = false;
      continue;
    }

    if (ch === "[") {
      inBracket = true;
      result += ch;
      continue;
    }

    switch (ch) {
      case ".": result += "\\."; break;
      case "%": result += ".*"; break;
      case "_": result += "."; break;
      default: result += ch;
    }
  }

  return result;
}
```

Replace the `evaluateStringFilter` stub:

```typescript
function evaluateStringFilter(cell: CellValue, filter: StringFilter): boolean {
  if (filter.fn === "IS_NULL") return cell.type === "NULL";
  if (filter.fn === "NOT_NULL") return cell.type !== "NULL";
  if (cell.type === "NULL") return false;

  const value = (cell.type === "TEXT" || cell.type === "LABEL") ? cell.value : String(cell.value);

  switch (filter.fn) {
    case "EQUALS_TO": return value === filter.value;
    case "NOT_EQUALS_TO": return value !== filter.value;
    case "GREATER_THAN": return value > filter.value;
    case "GREATER_OR_EQUALS_TO": return value >= filter.value;
    case "LOWER_THAN": return value < filter.value;
    case "LOWER_OR_EQUALS_TO": return value <= filter.value;
    case "BETWEEN": return value >= filter.low && value <= filter.high;
    case "IN": return filter.values.includes(value);
    case "NOT_IN": return !filter.values.includes(value);
    case "LIKE_TO": {
      const patternStr = filter.caseSensitive ? filter.pattern : filter.pattern.toLowerCase();
      const strValue = filter.caseSensitive ? value : value.toLowerCase();
      const regex = new RegExp("^" + compileLikePattern(patternStr) + "$");
      return regex.test(strValue);
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: All string and LIKE_TO tests pass.

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/filter-eval.ts packages/core/src/dataset/filter-eval.test.ts
git commit -m "feat: string filter evaluation with bracket-aware LIKE_TO"
```

---

### Task 6: Date Filter Evaluation + TIME_FRAME

**Files:**
- Modify: `packages/core/src/dataset/filter-eval.ts`
- Modify: `packages/core/src/dataset/filter-eval.test.ts`

- [ ] **Step 1: Write failing tests for date filter evaluation**

Add import at the top of `packages/core/src/dataset/filter-eval.test.ts`:

```typescript
import { parseTimeFrame } from "./timeframe.js";
```

Then add tests:

```typescript
function df(filter: import("./filter.js").DateFilter): FilterExpression {
  return { type: "date", columnId: "date" as ColumnId, filter };
}

function dateDataSet(): TypedDataSet {
  return toTypedDataSet({
    columns: [col("date", "Date", ColumnType.DATE)],
    data: [
      ["2024-01-15T00:00:00.000Z"],
      ["2024-03-15T00:00:00.000Z"],
      ["2024-06-15T00:00:00.000Z"],
      ["2024-09-15T00:00:00.000Z"],
      ["2024-12-15T00:00:00.000Z"],
    ],
  });
}

describe("applyFilter — date", () => {
  const march = new Date(Date.UTC(2024, 2, 15));

  it("EQUALS_TO matches exact date", () => {
    const result = applyFilter(dateDataSet(), { type: "filter", expressions: [df({ fn: "EQUALS_TO", value: march })] });
    expect(result.rows).toHaveLength(1);
  });

  it("GREATER_THAN filters by timestamp", () => {
    const result = applyFilter(dateDataSet(), { type: "filter", expressions: [df({ fn: "GREATER_THAN", value: march })] });
    expect(result.rows).toHaveLength(3); // Jun, Sep, Dec
  });

  it("BETWEEN inclusive on date range", () => {
    const low = new Date(Date.UTC(2024, 2, 1));
    const high = new Date(Date.UTC(2024, 8, 30));
    const result = applyFilter(dateDataSet(), { type: "filter", expressions: [df({ fn: "BETWEEN", low, high })] });
    expect(result.rows).toHaveLength(3); // Mar, Jun, Sep
  });

  it("TIME_FRAME filters using resolved time range", () => {
    const timeFrame = parseTimeFrame("begin[year] till end[year]");
    const refDate = new Date(Date.UTC(2024, 5, 1));
    const result = applyFilter(
      dateDataSet(),
      { type: "filter", expressions: [df({ fn: "TIME_FRAME", timeFrame })] },
      refDate,
    );
    expect(result.rows).toHaveLength(5); // all dates are in 2024
  });

  it("TIME_FRAME excludes dates outside range", () => {
    const timeFrame = parseTimeFrame("begin[quarter] till end[quarter]");
    const refDate = new Date(Date.UTC(2024, 5, 1)); // Q2: Apr-Jun
    const result = applyFilter(
      dateDataSet(),
      { type: "filter", expressions: [df({ fn: "TIME_FRAME", timeFrame })] },
      refDate,
    );
    expect(result.rows).toHaveLength(1); // only Jun 15
  });

  it("IS_NULL / NOT_NULL on date column with null", () => {
    const ds = toTypedDataSet({
      columns: [col("date", "Date", ColumnType.DATE)],
      data: [["2024-01-01T00:00:00.000Z"], [null]],
    });
    const nullResult = applyFilter(ds, { type: "filter", expressions: [df({ fn: "IS_NULL" })] });
    expect(nullResult.rows).toHaveLength(1);
    const notNullResult = applyFilter(ds, { type: "filter", expressions: [df({ fn: "NOT_NULL" })] });
    expect(notNullResult.rows).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: Date tests fail (stub returns false).

- [ ] **Step 3: Implement evaluateDateFilter**

Replace the stub in `packages/core/src/dataset/filter-eval.ts`:

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
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: All date tests pass.

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/filter-eval.ts packages/core/src/dataset/filter-eval.test.ts
git commit -m "feat: date filter evaluation with TIME_FRAME pre-resolution"
```

---

### Task 7: Logical Composition (AND/OR/NOT)

**Files:**
- Modify: `packages/core/src/dataset/filter-eval.test.ts`

- [ ] **Step 1: Write failing tests for logical composition**

Add to `packages/core/src/dataset/filter-eval.test.ts`:

```typescript
describe("applyFilter — logical composition", () => {
  it("AND requires all children to pass", () => {
    const ds = numericDataSet(); // 10, 20, 30, 40, 50
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "and",
      children: [
        nf({ fn: "GREATER_THAN", value: 15 }),
        nf({ fn: "LOWER_THAN", value: 45 }),
      ],
    }]});
    expect(result.rows).toHaveLength(3); // 20, 30, 40
  });

  it("OR requires any child to pass", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "or",
      children: [
        nf({ fn: "EQUALS_TO", value: 10 }),
        nf({ fn: "EQUALS_TO", value: 50 }),
      ],
    }]});
    expect(result.rows).toHaveLength(2);
  });

  it("NOT inverts child", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "not",
      child: nf({ fn: "EQUALS_TO", value: 30 }),
    }]});
    expect(result.rows).toHaveLength(4);
  });

  it("nested: NOT(AND(GT(20), LT(40)))", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "not",
      child: {
        type: "and",
        children: [
          nf({ fn: "GREATER_THAN", value: 20 }),
          nf({ fn: "LOWER_THAN", value: 40 }),
        ],
      },
    }]});
    expect(result.rows).toHaveLength(4); // everything except 30
  });

  it("multiple top-level expressions are implicitly ANDed", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [
      nf({ fn: "GREATER_THAN", value: 15 }),
      nf({ fn: "LOWER_THAN", value: 35 }),
    ]});
    expect(result.rows).toHaveLength(2); // 20, 30
  });

  it("mixed column types in OR", () => {
    const ds = toTypedDataSet({
      columns: [
        col("val", "Value", ColumnType.NUMBER),
        col("name", "Name", ColumnType.LABEL),
      ],
      data: [["10", "Alice"], ["20", "Bob"], ["30", "Charlie"]],
    });
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "or",
      children: [
        nf({ fn: "EQUALS_TO", value: 10 }),
        sf({ fn: "EQUALS_TO", value: "Charlie" }),
      ],
    }]});
    expect(result.rows).toHaveLength(2);
  });

  it("empty AND returns all rows", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "and", children: [],
    }]});
    expect(result.rows).toHaveLength(5);
  });

  it("empty OR returns no rows", () => {
    const ds = numericDataSet();
    const result = applyFilter(ds, { type: "filter", expressions: [{
      type: "or", children: [],
    }]});
    expect(result.rows).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests**

Run: `yarn workspace @melviz/core run test -- filter-eval.test`
Expected: All tests pass — the AND/OR/NOT logic is already implemented in `evaluateExpression` from Task 4. If any test fails, fix the evaluator.

- [ ] **Step 3: Commit**

```
git add packages/core/src/dataset/filter-eval.test.ts
git commit -m "test: logical composition tests for FilterExpression (AND/OR/NOT)"
```

---

### Task 8: Full Test Suite Verification and Type Check

**Files:**
- All files

- [ ] **Step 1: Run full test suite**

Run: `yarn workspace @melviz/core run test`
Expected: All tests pass.

- [ ] **Step 2: Run type check**

Run: `yarn workspace @melviz/core run build`
Expected: Clean compile, no type errors.

- [ ] **Step 3: Verify file structure matches spec**

Confirm these files exist:
- `packages/core/src/dataset/types.ts` — has `"NULL"` CellValue and nullable `DataSet.data`
- `packages/core/src/dataset/conversion.ts` — handles null cells both directions
- `packages/core/src/dataset/filter.ts` — all type definitions
- `packages/core/src/dataset/timeframe.ts` — types + parser + resolution
- `packages/core/src/dataset/filter-eval.ts` — applyFilter + all evaluators

- [ ] **Step 4: Commit (if any fixups needed)**

```
git add -A packages/core/
git commit -m "chore: filter model verification — all tests passing, types clean"
```
