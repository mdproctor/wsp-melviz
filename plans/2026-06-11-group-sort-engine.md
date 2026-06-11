# GroupOp, SortOp, and applyOps Engine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the complete Filter→Group→Sort pipeline with type-safe aggregation, calendar-aware date bucketing, and a pure-function engine.

**Architecture:** Discriminated unions for type safety (aggregation enforced at compile time, grouping strategy validated at runtime). Pure functions on immutable TypedDataSet. Date arithmetic centralised in `date-interval.ts` with UTC-only operations. Consecutive GroupOps processed as a unit with deferred materialisation via `applyGroupSequence`.

**Tech Stack:** TypeScript 5.6, Vitest, ES modules (`.js` extensions in imports)

**Spec:** `docs/superpowers/specs/2026-06-11-group-sort-engine-design.md`

**Test runner:** `yarn workspace @melviz/core run test` (vitest)

**Base path:** `packages/core/src/dataset/`

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `date-interval.ts` | CREATE | DateIntervalType, Month, DayOfWeek, APPROXIMATE_DURATION_MS, truncateToInterval(), advanceByInterval() |
| `date-interval.test.ts` | CREATE | Tests for date arithmetic |
| `timeframe.ts` | MODIFY | Refactor: import from date-interval.ts, Extract<> for subsidiary types |
| `errors.ts` | MODIFY | Add INVALID_OPERATION code |
| `group.ts` | CREATE | GroupOp, GroupingKey, GroupStrategy, ResultColumn, Aggregation types |
| `sort.ts` | CREATE | SortOp, SortColumn, SortOrder |
| `ops.ts` | CREATE | DataSetOp union, applyOps(), validateOpOrder() |
| `group-eval.ts` | CREATE | computeAggregation(), bucketing functions, applyGroup(), applyGroupSequence() |
| `group-eval.test.ts` | CREATE | Tests for aggregation, bucketing, group operations |
| `sort-eval.ts` | CREATE | applySort() |
| `sort-eval.test.ts` | CREATE | Tests for sort |
| `ops.test.ts` | CREATE | Tests for applyOps pipeline |

---

### Task 1: date-interval.ts — Types and Duration Table

**Files:**
- Create: `packages/core/src/dataset/date-interval.ts`

- [ ] **Step 1: Create date-interval.ts with types and duration table**

```typescript
// packages/core/src/dataset/date-interval.ts

export type DateIntervalType =
  | "MILLISECOND" | "HUNDRETH" | "TENTH"
  | "SECOND" | "MINUTE" | "HOUR"
  | "DAY" | "DAY_OF_WEEK" | "WEEK"
  | "MONTH" | "QUARTER" | "YEAR"
  | "DECADE" | "CENTURY" | "MILLENIUM";

export type Month = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12;
export type DayOfWeek = 1 | 2 | 3 | 4 | 5 | 6 | 7;

export const APPROXIMATE_DURATION_MS: Readonly<Record<DateIntervalType, number>> = {
  MILLISECOND: 1,
  HUNDRETH: 10,
  TENTH: 100,
  SECOND: 1_000,
  MINUTE: 60_000,
  HOUR: 3_600_000,
  DAY: 86_400_000,
  DAY_OF_WEEK: 86_400_000,
  WEEK: 604_800_000,
  MONTH: 2_678_400_000,
  QUARTER: 8_035_200_000,
  YEAR: 32_140_800_000,
  DECADE: 321_408_000_000,
  CENTURY: 3_214_080_000_000,
  MILLENIUM: 32_140_800_000_000,
};
```

- [ ] **Step 2: Verify it compiles**

Run: `yarn workspace @melviz/core run build`
Expected: success

- [ ] **Step 3: Commit**

```
git add packages/core/src/dataset/date-interval.ts
git commit -m "feat: date-interval.ts — types and duration table  Refs #2"
```

---

### Task 2: date-interval.ts — truncateToInterval()

**Files:**
- Modify: `packages/core/src/dataset/date-interval.ts`
- Create: `packages/core/src/dataset/date-interval.test.ts`

- [ ] **Step 1: Write failing tests for truncateToInterval**

```typescript
// packages/core/src/dataset/date-interval.test.ts
import { describe, it, expect } from "vitest";
import { truncateToInterval } from "./date-interval.js";

function utc(y: number, m: number, d: number, h = 0, min = 0, s = 0, ms = 0): Date {
  return new Date(Date.UTC(y, m - 1, d, h, min, s, ms));
}

describe("truncateToInterval", () => {
  const date = utc(2024, 7, 15, 14, 35, 22, 456);

  it("YEAR → Jan 1 00:00:00.000", () => {
    expect(truncateToInterval(date, "YEAR")).toEqual(utc(2024, 1, 1));
  });

  it("YEAR with fiscal start April → Apr 1 of current fiscal year", () => {
    expect(truncateToInterval(utc(2024, 7, 15), "YEAR", { firstMonthOfYear: 4 }))
      .toEqual(utc(2024, 4, 1));
  });

  it("YEAR with fiscal start April — date before April → previous fiscal year", () => {
    expect(truncateToInterval(utc(2024, 2, 15), "YEAR", { firstMonthOfYear: 4 }))
      .toEqual(utc(2023, 4, 1));
  });

  it("QUARTER → first month of quarter, 1st, 00:00:00", () => {
    expect(truncateToInterval(date, "QUARTER")).toEqual(utc(2024, 7, 1));
  });

  it("QUARTER with fiscal start April — July is Q2 of fiscal → Jul 1", () => {
    expect(truncateToInterval(utc(2024, 7, 15), "QUARTER", { firstMonthOfYear: 4 }))
      .toEqual(utc(2024, 7, 1));
  });

  it("QUARTER with fiscal start April — May is Q1 of fiscal → Apr 1", () => {
    expect(truncateToInterval(utc(2024, 5, 15), "QUARTER", { firstMonthOfYear: 4 }))
      .toEqual(utc(2024, 4, 1));
  });

  it("MONTH → 1st of month, 00:00:00", () => {
    expect(truncateToInterval(date, "MONTH")).toEqual(utc(2024, 7, 1));
  });

  it("DAY → 00:00:00 of same day", () => {
    expect(truncateToInterval(date, "DAY")).toEqual(utc(2024, 7, 15));
  });

  it("HOUR → same hour, :00:00", () => {
    expect(truncateToInterval(date, "HOUR")).toEqual(utc(2024, 7, 15, 14));
  });

  it("MINUTE → same minute, :00", () => {
    expect(truncateToInterval(date, "MINUTE")).toEqual(utc(2024, 7, 15, 14, 35));
  });

  it("SECOND → same second, .000", () => {
    expect(truncateToInterval(date, "SECOND")).toEqual(utc(2024, 7, 15, 14, 35, 22));
  });

  it("DECADE → start of decade", () => {
    expect(truncateToInterval(date, "DECADE")).toEqual(utc(2020, 1, 1));
  });

  it("CENTURY → start of century", () => {
    expect(truncateToInterval(date, "CENTURY")).toEqual(utc(2000, 1, 1));
  });

  it("does not mutate input", () => {
    const d = utc(2024, 7, 15, 14, 35, 22);
    const originalTime = d.getTime();
    truncateToInterval(d, "YEAR");
    expect(d.getTime()).toBe(originalTime);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- date-interval`
Expected: FAIL — truncateToInterval not exported

- [ ] **Step 3: Implement truncateToInterval**

Add to `date-interval.ts`:

```typescript
export function truncateToInterval(
  date: Date,
  unit: DateIntervalType,
  opts?: { firstMonthOfYear?: Month },
): Date {
  const d = new Date(date.getTime());
  const firstMonth = (opts?.firstMonthOfYear ?? 1) - 1; // 0-based for UTC methods

  switch (unit) {
    case "MILLENIUM": {
      const base = Math.floor(d.getUTCFullYear() / 1000) * 1000;
      d.setUTCFullYear(base, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "CENTURY": {
      const base = Math.floor(d.getUTCFullYear() / 100) * 100;
      d.setUTCFullYear(base, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "DECADE": {
      const base = Math.floor(d.getUTCFullYear() / 10) * 10;
      d.setUTCFullYear(base, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "YEAR": {
      const month = d.getUTCMonth();
      const yearAdj = month < firstMonth ? -1 : 0;
      d.setUTCFullYear(d.getUTCFullYear() + yearAdj, firstMonth, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "QUARTER": {
      const month = d.getUTCMonth();
      const qStart = getQuarterFirstMonth(firstMonth, month);
      const yearAdj = qStart > month ? -1 : 0;
      d.setUTCFullYear(d.getUTCFullYear() + yearAdj);
      d.setUTCMonth(qStart, 1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    }
    case "MONTH":
      d.setUTCDate(1);
      d.setUTCHours(0, 0, 0, 0);
      break;
    case "WEEK":
    case "DAY":
    case "DAY_OF_WEEK":
      d.setUTCHours(0, 0, 0, 0);
      break;
    case "HOUR":
      d.setUTCMinutes(0, 0, 0);
      break;
    case "MINUTE":
      d.setUTCSeconds(0, 0);
      break;
    case "SECOND":
      d.setUTCMilliseconds(0);
      break;
    case "TENTH":
      d.setUTCMilliseconds(Math.floor(d.getUTCMilliseconds() / 100) * 100);
      break;
    case "HUNDRETH":
      d.setUTCMilliseconds(Math.floor(d.getUTCMilliseconds() / 10) * 10);
      break;
    case "MILLISECOND":
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
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- date-interval`
Expected: all pass

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/date-interval.ts packages/core/src/dataset/date-interval.test.ts
git commit -m "feat: truncateToInterval with fiscal year support  Refs #2"
```

---

### Task 3: date-interval.ts — advanceByInterval()

**Files:**
- Modify: `packages/core/src/dataset/date-interval.ts`
- Modify: `packages/core/src/dataset/date-interval.test.ts`

- [ ] **Step 1: Write failing tests for advanceByInterval**

Add to `date-interval.test.ts`:

```typescript
import { truncateToInterval, advanceByInterval } from "./date-interval.js";

describe("advanceByInterval", () => {
  it("MONTH — normal advance", () => {
    expect(advanceByInterval(utc(2024, 3, 1), "MONTH", 1)).toEqual(utc(2024, 4, 1));
  });

  it("MONTH — Jan 31 + 1 month → Feb 28/29 (leap year)", () => {
    expect(advanceByInterval(utc(2024, 1, 31), "MONTH", 1)).toEqual(utc(2024, 2, 29));
  });

  it("MONTH — Jan 31 + 1 month → Feb 28 (non-leap year)", () => {
    expect(advanceByInterval(utc(2023, 1, 31), "MONTH", 1)).toEqual(utc(2023, 2, 28));
  });

  it("QUARTER — advance by 3 months", () => {
    expect(advanceByInterval(utc(2024, 1, 1), "QUARTER", 1)).toEqual(utc(2024, 4, 1));
  });

  it("YEAR — advance by 1 year", () => {
    expect(advanceByInterval(utc(2024, 1, 1), "YEAR", 1)).toEqual(utc(2025, 1, 1));
  });

  it("DAY — advance by 1 day", () => {
    expect(advanceByInterval(utc(2024, 1, 31), "DAY", 1)).toEqual(utc(2024, 2, 1));
  });

  it("WEEK — advance by 7 days", () => {
    expect(advanceByInterval(utc(2024, 1, 1), "WEEK", 1)).toEqual(utc(2024, 1, 8));
  });

  it("HOUR — advance by 1 hour", () => {
    expect(advanceByInterval(utc(2024, 1, 1, 23), "HOUR", 1)).toEqual(utc(2024, 1, 2, 0));
  });

  it("MINUTE — advance by 1 minute", () => {
    expect(advanceByInterval(utc(2024, 1, 1, 0, 59), "MINUTE", 1)).toEqual(utc(2024, 1, 1, 1, 0));
  });

  it("SECOND — advance by 1 second", () => {
    expect(advanceByInterval(utc(2024, 1, 1, 0, 0, 59), "SECOND", 1)).toEqual(utc(2024, 1, 1, 0, 1, 0));
  });

  it("negative count — go backward", () => {
    expect(advanceByInterval(utc(2024, 3, 1), "MONTH", -1)).toEqual(utc(2024, 2, 1));
  });

  it("count > 1 — multiple advances", () => {
    expect(advanceByInterval(utc(2024, 1, 1), "MONTH", 6)).toEqual(utc(2024, 7, 1));
  });

  it("does not mutate input", () => {
    const d = utc(2024, 1, 1);
    const originalTime = d.getTime();
    advanceByInterval(d, "MONTH", 1);
    expect(d.getTime()).toBe(originalTime);
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @melviz/core run test -- date-interval`
Expected: FAIL — advanceByInterval not exported

- [ ] **Step 3: Implement advanceByInterval**

Add to `date-interval.ts`:

```typescript
export function advanceByInterval(date: Date, unit: DateIntervalType, count: number): Date {
  const d = new Date(date.getTime());

  switch (unit) {
    case "MILLENIUM": d.setUTCFullYear(d.getUTCFullYear() + 1000 * count); break;
    case "CENTURY": d.setUTCFullYear(d.getUTCFullYear() + 100 * count); break;
    case "DECADE": d.setUTCFullYear(d.getUTCFullYear() + 10 * count); break;
    case "YEAR": d.setUTCFullYear(d.getUTCFullYear() + count); break;
    case "QUARTER": d.setUTCMonth(d.getUTCMonth() + 3 * count); break;
    case "MONTH": d.setUTCMonth(d.getUTCMonth() + count); break;
    case "WEEK": d.setUTCDate(d.getUTCDate() + 7 * count); break;
    case "DAY":
    case "DAY_OF_WEEK": d.setUTCDate(d.getUTCDate() + count); break;
    case "HOUR": d.setUTCHours(d.getUTCHours() + count); break;
    case "MINUTE": d.setUTCMinutes(d.getUTCMinutes() + count); break;
    case "SECOND": d.setUTCSeconds(d.getUTCSeconds() + count); break;
    case "TENTH": d.setUTCMilliseconds(d.getUTCMilliseconds() + 100 * count); break;
    case "HUNDRETH": d.setUTCMilliseconds(d.getUTCMilliseconds() + 10 * count); break;
    case "MILLISECOND": d.setUTCMilliseconds(d.getUTCMilliseconds() + count); break;
  }
  return d;
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- date-interval`
Expected: all pass

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/date-interval.ts packages/core/src/dataset/date-interval.test.ts
git commit -m "feat: advanceByInterval with calendar arithmetic  Refs #2"
```

---

### Task 4: Refactor timeframe.ts — import from date-interval.ts

**Files:**
- Modify: `packages/core/src/dataset/timeframe.ts`
- Existing test: `packages/core/src/dataset/timeframe.test.ts` (must continue to pass)

This is a refactoring task. The existing `truncate()` and `applyOffset()` logic moves to `date-interval.ts` (Tasks 2-3). `timeframe.ts` now imports from it.

- [ ] **Step 1: Run existing timeframe tests to capture baseline**

Run: `yarn workspace @melviz/core run test -- timeframe`
Expected: all pass (baseline)

- [ ] **Step 2: Refactor timeframe.ts**

Key changes:
1. Remove the local `DateIntervalType` — import from `date-interval.js`
2. Remove the local `Month` string type — replace with `import { type Month } from "./date-interval.js"` and use a string-to-number lookup at parse boundaries
3. Replace the local `truncate()` function with calls to imported `truncateToInterval()` and `advanceByInterval()`
4. Replace the local `applyOffset()` function with calls to imported `advanceByInterval()`
5. `TruncationUnit` and `OffsetUnit` become `Extract<>` subsets of imported `DateIntervalType`

The `Month` type change requires updating `TimeInstant.firstMonthOfYear` from string to number. This affects `parseTimeInstant()` (where the string is parsed) and `truncate()` (where it's used). The parse boundary converts `"JANUARY"` → `1` using a lookup map.

Note: the existing `MONTH_INDEX` map already does this conversion — it maps string month names to 0-based indices. We change it to produce 1-based `Month` values and use it at the parse site.

- [ ] **Step 3: Run existing tests to verify no regressions**

Run: `yarn workspace @melviz/core run test -- timeframe`
Expected: all pass — identical behavior

- [ ] **Step 4: Run all tests to verify no regressions**

Run: `yarn workspace @melviz/core run test`
Expected: all pass

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/timeframe.ts
git commit -m "refactor: timeframe.ts imports date arithmetic from date-interval.ts  Refs #2"
```

---

### Task 5: errors.ts — Add INVALID_OPERATION code

**Files:**
- Modify: `packages/core/src/dataset/errors.ts`

- [ ] **Step 1: Add INVALID_OPERATION to DataSetErrorCode**

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
  | "INVALID_OPERATION";
```

- [ ] **Step 2: Verify build**

Run: `yarn workspace @melviz/core run build`
Expected: success

- [ ] **Step 3: Commit**

```
git add packages/core/src/dataset/errors.ts
git commit -m "feat: add INVALID_OPERATION error code  Refs #2"
```

---

### Task 6: group.ts — Type Definitions

**Files:**
- Create: `packages/core/src/dataset/group.ts`

- [ ] **Step 1: Create group.ts with all types from spec §2-4**

```typescript
// packages/core/src/dataset/group.ts

import type { ColumnId } from "./types.js";
import type { DateIntervalType, Month, DayOfWeek } from "./date-interval.js";

// --- Aggregation ---

export type NumericAggregation =
  | { readonly fn: "SUM" }
  | { readonly fn: "AVERAGE" }
  | { readonly fn: "MEDIAN" };

export type UniversalAggregation =
  | { readonly fn: "COUNT" }
  | { readonly fn: "DISTINCT" }
  | { readonly fn: "MIN" }
  | { readonly fn: "MAX" }
  | { readonly fn: "JOIN"; readonly separator: string };

export type Aggregation = NumericAggregation | UniversalAggregation;

// --- Result Columns ---

export type ResultColumn =
  | { readonly kind: "key"; readonly sourceId: ColumnId; readonly columnId: ColumnId }
  | { readonly kind: "aggregate"; readonly sourceId: ColumnId; readonly columnId: ColumnId;
      readonly fn: Aggregation }
  | { readonly kind: "select"; readonly sourceId: ColumnId; readonly columnId: ColumnId };

// --- Grouping ---

export type FixedCalendarUnit =
  | "QUARTER" | "MONTH" | "DAY_OF_WEEK"
  | "HOUR" | "MINUTE" | "SECOND";

export type GroupStrategy =
  | { readonly mode: "distinct" }
  | { readonly mode: "fixedCalendar"; readonly unit: FixedCalendarUnit }
  | { readonly mode: "dynamicRange"; readonly preferredUnit?: DateIntervalType }
  | { readonly mode: "dynamic"; readonly preferredUnit?: DateIntervalType };

export interface GroupingKey {
  readonly sourceId: ColumnId;
  readonly columnId: ColumnId;
  readonly strategy: GroupStrategy;
  readonly maxIntervals: number;
  readonly emptyIntervals: boolean;
  readonly ascendingOrder: boolean;
  readonly firstMonthOfYear?: Month;
  readonly firstDayOfWeek?: DayOfWeek;
}

// --- Interval (internal, used by bucketing engine) ---

export interface Interval {
  readonly name: string;
  readonly index: number;
  readonly rowIndices: readonly number[];
  readonly minValue?: Date | number;
  readonly maxValue?: Date | number;
}

export type IntervalList = readonly Interval[];

// --- GroupOp ---

export interface GroupOp {
  readonly type: "group";
  readonly groupingKey: GroupingKey | null;
  readonly columns: readonly ResultColumn[];
  readonly selectedIntervals?: readonly string[];
  readonly join?: boolean;
}
```

- [ ] **Step 2: Verify build**

Run: `yarn workspace @melviz/core run build`
Expected: success

- [ ] **Step 3: Commit**

```
git add packages/core/src/dataset/group.ts
git commit -m "feat: group.ts — type definitions for GroupOp, aggregation, bucketing  Refs #2"
```

---

### Task 7: sort.ts — Type Definitions

**Files:**
- Create: `packages/core/src/dataset/sort.ts`

- [ ] **Step 1: Create sort.ts**

```typescript
// packages/core/src/dataset/sort.ts

import type { ColumnId } from "./types.js";

export type SortOrder = "ASCENDING" | "DESCENDING";

export interface SortColumn {
  readonly columnId: ColumnId;
  readonly order: SortOrder;
}

export interface SortOp {
  readonly type: "sort";
  readonly columns: readonly SortColumn[];
}
```

- [ ] **Step 2: Verify build**

Run: `yarn workspace @melviz/core run build`
Expected: success

- [ ] **Step 3: Commit**

```
git add packages/core/src/dataset/sort.ts
git commit -m "feat: sort.ts — SortOp, SortColumn, SortOrder types  Refs #2"
```

---

### Task 8: ops.ts — DataSetOp Union and validateOpOrder

**Files:**
- Create: `packages/core/src/dataset/ops.ts`
- Create: `packages/core/src/dataset/ops.test.ts`

- [ ] **Step 1: Write failing tests for validateOpOrder**

```typescript
// packages/core/src/dataset/ops.test.ts
import { describe, it, expect } from "vitest";
import { validateOpOrder } from "./ops.js";
import type { DataSetOp } from "./ops.js";
import type { FilterOp } from "./filter.js";
import type { GroupOp } from "./group.js";
import type { SortOp } from "./sort.js";
import type { ColumnId } from "./types.js";

const filter: FilterOp = { type: "filter", expressions: [] };
const group: GroupOp = {
  type: "group", groupingKey: null, columns: [],
};
const sort: SortOp = {
  type: "sort", columns: [{ columnId: "x" as ColumnId, order: "ASCENDING" }],
};

describe("validateOpOrder", () => {
  it("accepts empty ops", () => {
    expect(() => validateOpOrder([])).not.toThrow();
  });

  it("accepts F*G*S? patterns", () => {
    expect(() => validateOpOrder([filter])).not.toThrow();
    expect(() => validateOpOrder([filter, group])).not.toThrow();
    expect(() => validateOpOrder([filter, group, sort])).not.toThrow();
    expect(() => validateOpOrder([group, sort])).not.toThrow();
    expect(() => validateOpOrder([group, group])).not.toThrow();
    expect(() => validateOpOrder([sort])).not.toThrow();
  });

  it("rejects group before filter", () => {
    expect(() => validateOpOrder([group, filter])).toThrow("INVALID_OPERATION");
  });

  it("rejects sort before group", () => {
    expect(() => validateOpOrder([sort, group])).toThrow("INVALID_OPERATION");
  });

  it("rejects multiple sorts", () => {
    expect(() => validateOpOrder([sort, sort])).toThrow("INVALID_OPERATION");
  });

  it("rejects filter after group", () => {
    expect(() => validateOpOrder([filter, group, filter])).toThrow("INVALID_OPERATION");
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @melviz/core run test -- ops`
Expected: FAIL

- [ ] **Step 3: Implement ops.ts with validateOpOrder**

```typescript
// packages/core/src/dataset/ops.ts

import type { FilterOp } from "./filter.js";
import type { GroupOp } from "./group.js";
import type { SortOp } from "./sort.js";
import { DataSetError } from "./errors.js";

export type DataSetOp = FilterOp | GroupOp | SortOp;

export function validateOpOrder(ops: readonly DataSetOp[]): void {
  let pattern = "";
  for (const op of ops) {
    switch (op.type) {
      case "filter": pattern += "F"; break;
      case "group": pattern += "G"; break;
      case "sort": pattern += "S"; break;
    }
  }
  if (!/^F*G*S?$/.test(pattern)) {
    throw new DataSetError(
      "INVALID_OPERATION",
      `Invalid operation sequence "${pattern}". Valid pattern: (0..N) FILTER > (0..N) GROUP > (0..1) SORT`,
    );
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- ops`
Expected: all pass

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/ops.ts packages/core/src/dataset/ops.test.ts
git commit -m "feat: ops.ts — DataSetOp union and validateOpOrder  Refs #2"
```

---

### Task 9: Aggregation Functions — computeAggregation()

**Files:**
- Create: `packages/core/src/dataset/group-eval.ts`
- Create: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for computeAggregation**

```typescript
// packages/core/src/dataset/group-eval.test.ts
import { describe, it, expect } from "vitest";
import { computeAggregation } from "./group-eval.js";
import type { CellValue } from "./types.js";
import { ColumnType } from "./types.js";
import type { Aggregation } from "./group.js";

function num(v: number): CellValue { return { type: ColumnType.NUMBER, value: v }; }
function text(v: string): CellValue { return { type: ColumnType.TEXT, value: v }; }
function label(v: string): CellValue { return { type: ColumnType.LABEL, value: v }; }
function date(y: number, m: number, d: number): CellValue {
  return { type: ColumnType.DATE, value: new Date(Date.UTC(y, m - 1, d)) };
}
const NULL: CellValue = { type: "NULL" };

describe("computeAggregation", () => {
  describe("COUNT", () => {
    const fn: Aggregation = { fn: "COUNT" };
    it("counts all rows including NULLs", () => {
      expect(computeAggregation(fn, [num(1), NULL, num(3)])).toEqual(num(3));
    });
    it("0 rows → 0", () => {
      expect(computeAggregation(fn, [])).toEqual(num(0));
    });
  });

  describe("DISTINCT", () => {
    const fn: Aggregation = { fn: "DISTINCT" };
    it("counts distinct values, NULL as one", () => {
      expect(computeAggregation(fn, [num(1), num(2), num(1), NULL])).toEqual(num(3));
    });
    it("all NULL → 1", () => {
      expect(computeAggregation(fn, [NULL, NULL])).toEqual(num(1));
    });
    it("empty → 0", () => {
      expect(computeAggregation(fn, [])).toEqual(num(0));
    });
  });

  describe("SUM", () => {
    const fn: Aggregation = { fn: "SUM" };
    it("sums numeric values, skipping NULLs", () => {
      expect(computeAggregation(fn, [num(10), NULL, num(20)])).toEqual(num(30));
    });
    it("empty → 0", () => {
      expect(computeAggregation(fn, [])).toEqual(num(0));
    });
    it("all NULL → 0", () => {
      expect(computeAggregation(fn, [NULL, NULL])).toEqual(num(0));
    });
  });

  describe("AVERAGE", () => {
    const fn: Aggregation = { fn: "AVERAGE" };
    it("averages non-null values", () => {
      expect(computeAggregation(fn, [num(10), num(20), num(30)])).toEqual(num(20));
    });
    it("skips NULLs in both sum and count", () => {
      expect(computeAggregation(fn, [num(10), NULL, num(30)])).toEqual(num(20));
    });
    it("empty → NULL", () => {
      expect(computeAggregation(fn, [])).toEqual(NULL);
    });
    it("all NULL → NULL", () => {
      expect(computeAggregation(fn, [NULL, NULL, NULL])).toEqual(NULL);
    });
  });

  describe("MEDIAN", () => {
    const fn: Aggregation = { fn: "MEDIAN" };
    it("odd count → middle value", () => {
      expect(computeAggregation(fn, [num(3), num(1), num(2)])).toEqual(num(2));
    });
    it("even count → average of two middle values", () => {
      expect(computeAggregation(fn, [num(1), num(2), num(3), num(4)])).toEqual(num(2.5));
    });
    it("skips NULLs", () => {
      expect(computeAggregation(fn, [num(1), NULL, num(3)])).toEqual(num(2));
    });
    it("empty → NULL", () => {
      expect(computeAggregation(fn, [])).toEqual(NULL);
    });
    it("all NULL → NULL", () => {
      expect(computeAggregation(fn, [NULL, NULL])).toEqual(NULL);
    });
  });

  describe("MIN", () => {
    const fn: Aggregation = { fn: "MIN" };
    it("finds minimum number", () => {
      expect(computeAggregation(fn, [num(30), num(10), num(20)])).toEqual(num(10));
    });
    it("finds earliest date", () => {
      const d1 = date(2024, 1, 1);
      const d2 = date(2024, 6, 15);
      const result = computeAggregation(fn, [d2, d1]);
      expect(result).toEqual(d1);
    });
    it("finds lexicographic min for labels", () => {
      expect(computeAggregation(fn, [label("banana"), label("apple")])).toEqual(label("apple"));
    });
    it("skips NULLs", () => {
      expect(computeAggregation(fn, [num(20), NULL, num(10)])).toEqual(num(10));
    });
    it("empty → NULL", () => {
      expect(computeAggregation(fn, [])).toEqual(NULL);
    });
    it("all NULL → NULL", () => {
      expect(computeAggregation(fn, [NULL, NULL])).toEqual(NULL);
    });
  });

  describe("MAX", () => {
    const fn: Aggregation = { fn: "MAX" };
    it("finds maximum number", () => {
      expect(computeAggregation(fn, [num(10), num(30), num(20)])).toEqual(num(30));
    });
    it("finds latest date", () => {
      const d1 = date(2024, 1, 1);
      const d2 = date(2024, 6, 15);
      const result = computeAggregation(fn, [d1, d2]);
      expect(result).toEqual(d2);
    });
    it("empty → NULL", () => {
      expect(computeAggregation(fn, [])).toEqual(NULL);
    });
  });

  describe("JOIN", () => {
    const fn: Aggregation = { fn: "JOIN", separator: ", " };
    it("joins values with separator", () => {
      expect(computeAggregation(fn, [label("a"), label("b"), label("c")]))
        .toEqual(text("a, b, c"));
    });
    it("skips NULLs", () => {
      expect(computeAggregation(fn, [label("a"), NULL, label("c")]))
        .toEqual(text("a, c"));
    });
    it("empty → empty string", () => {
      expect(computeAggregation(fn, [])).toEqual(text(""));
    });
    it("all NULL → empty string", () => {
      expect(computeAggregation(fn, [NULL, NULL])).toEqual(text(""));
    });
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @melviz/core run test -- group-eval`
Expected: FAIL — computeAggregation not exported

- [ ] **Step 3: Implement computeAggregation**

Create `packages/core/src/dataset/group-eval.ts` with the aggregation function. Use a `switch` on `fn.fn` with type-specific comparison helpers. The implementation follows the spec §6 NULL handling table precisely.

Key implementation detail: for MIN/MAX comparison across types, extract a comparable value:
- NUMBER → the number
- DATE → `getTime()`
- LABEL/TEXT → the string

For JOIN, convert non-null values to string via `cellToString()` helper, then join with separator.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- group-eval`
Expected: all pass

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/group-eval.ts packages/core/src/dataset/group-eval.test.ts
git commit -m "feat: computeAggregation — all 8 functions with SQL null semantics  Refs #2"
```

---

### Task 10: Bucketing — Distinct

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for buildDistinctIntervals**

Test distinct bucketing for LABEL, NUMBER, DATE, and null values. Verify bucket names follow the spec §5 naming conventions (TEXT/LABEL: string value, NUMBER: `String(value)`, DATE: `toISOString()`, null: `"null"`).

```typescript
describe("buildDistinctIntervals", () => {
  it("creates one bucket per unique label value", () => { ... });
  it("creates one bucket per unique number value", () => { ... });
  it("names date buckets with toISOString()", () => { ... });
  it("groups null values into a 'null' bucket", () => { ... });
  it("preserves original row order within buckets", () => { ... });
});
```

- [ ] **Step 2: Implement buildDistinctIntervals**

```typescript
export function buildDistinctIntervals(
  ds: TypedDataSet,
  sourceId: ColumnId,
): IntervalList {
  // Walk column values, build Map<string, number[]>
  // Name per CellValue type: LABEL/TEXT → value, NUMBER → String(value), DATE → toISOString(), NULL → "null"
}
```

- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Commit**

---

### Task 11: Bucketing — Fixed Calendar

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for buildFixedCalendarIntervals**

Test each FixedCalendarUnit: MONTH (12 buckets), QUARTER (4 buckets), DAY_OF_WEEK (7 buckets), HOUR (24 buckets), MINUTE (60 buckets), SECOND (60 buckets). Test fiscal year alignment (firstMonthOfYear, firstDayOfWeek). Test bucket naming (index as string).

```typescript
describe("buildFixedCalendarIntervals", () => {
  it("MONTH — 12 buckets named '1' through '12'", () => { ... });
  it("MONTH — assigns dates to correct month bucket", () => { ... });
  it("MONTH with firstMonthOfYear=4 — buckets start from April", () => { ... });
  it("QUARTER — 4 buckets named '1' through '4'", () => { ... });
  it("QUARTER with firstMonthOfYear=4 — fiscal quarters", () => { ... });
  it("DAY_OF_WEEK — 7 buckets named '1' through '7'", () => { ... });
  it("DAY_OF_WEEK with firstDayOfWeek=7 — start from Sunday", () => { ... });
  it("HOUR — 24 buckets named '0' through '23'", () => { ... });
  it("ascending=false — reversed bucket order", () => { ... });
  it("emptyIntervals=true — includes buckets with no rows", () => { ... });
});
```

- [ ] **Step 2: Implement buildFixedCalendarIntervals**

Create all fixed interval buckets based on `FixedCalendarUnit`, respecting `firstMonthOfYear` and `firstDayOfWeek`. Assign rows to buckets by extracting the UTC calendar field from each date value.

- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Commit**

---

### Task 12: Bucketing — Dynamic Range (Dates)

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for buildDynamicDateIntervals**

Test the auto-sizing algorithm: date range spanning months → monthly buckets. Date range spanning days → daily buckets. preferredUnit enforcement. Calendar-aligned boundaries. Null skipping. Ascending/descending order.

```typescript
describe("buildDynamicDateIntervals", () => {
  it("auto-sizes to MONTH for multi-month span", () => { ... });
  it("auto-sizes to DAY for multi-day span within a month", () => { ... });
  it("respects preferredUnit — never finer than preferred", () => { ... });
  it("calendar-aligned boundaries — months start on 1st", () => { ... });
  it("single date → single interval", () => { ... });
  it("empty dataset → empty intervals", () => { ... });
  it("skips null dates", () => { ... });
  it("descending order — reversed intervals", () => { ... });
  it("interval names match ISO format for granularity", () => { ... });
});
```

- [ ] **Step 2: Implement buildDynamicDateIntervals**

Follow spec §5 algorithm:
1. Sort dates, find min/max
2. Compute span, walk APPROXIMATE_DURATION_MS to find interval type
3. Apply preferredUnit floor
4. Generate boundaries with truncateToInterval/advanceByInterval
5. Walk sorted values into intervals

- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Commit**

---

### Task 13: Bucketing — Dynamic Range (Numbers)

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for buildDynamicNumberIntervals**

```typescript
describe("buildDynamicNumberIntervals", () => {
  it("creates equal-width bins", () => { ... });
  it("respects maxIntervals", () => { ... });
  it("names bins as 'min-max'", () => { ... });
  it("single value → single bin", () => { ... });
  it("skips nulls", () => { ... });
  it("handles negative numbers", () => { ... });
});
```

- [ ] **Step 2: Implement buildDynamicNumberIntervals**

Find min/max numeric values (skip NULLs). Compute bin width = (max - min) / maxIntervals. Generate bins, assign values.

- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Commit**

---

### Task 14: applyGroup() — Single GroupOp

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for applyGroup**

```typescript
describe("applyGroup", () => {
  it("null groupingKey — whole-dataset SUM", () => { ... });
  it("null groupingKey — kind:key is an error", () => { ... });
  it("null groupingKey — kind:select returns first row value", () => { ... });
  it("distinct grouping — one row per unique value", () => { ... });
  it("aggregate SUM per group", () => { ... });
  it("aggregate COUNT per group", () => { ... });
  it("key column shows bucket name", () => { ... });
  it("select column shows first value in bucket", () => { ... });
  it("output columns have correct types", () => { ... });
  it("TYPE_MISMATCH — SUM on TEXT column", () => { ... });
  it("TYPE_MISMATCH — fixedCalendar on NUMBER column", () => { ... });
  it("emptyIntervals=false — excludes empty buckets", () => { ... });
  it("emptyIntervals=true — includes empty buckets", () => { ... });
  it("resolves dynamic strategy based on column type", () => { ... });
});
```

- [ ] **Step 2: Implement applyGroup**

Orchestrate: resolve dynamic strategy → dispatch to bucketing function → build output TypedDataSet. Handle null groupingKey, selectedIntervals, and full group-by.

- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Commit**

---

### Task 15: applyGroupSequence() — Consecutive GroupOps

**Files:**
- Modify: `packages/core/src/dataset/group-eval.ts`
- Modify: `packages/core/src/dataset/group-eval.test.ts`

- [ ] **Step 1: Write failing tests for applyGroupSequence**

```typescript
describe("applyGroupSequence", () => {
  it("second GroupOp without join or selectedIntervals → error", () => { ... });

  it("selectedIntervals — narrows to selected buckets", () => { ... });

  it("selectedIntervals + subsequent group — re-groups narrowed data", () => { ... });

  it("join: true — nested grouping produces parent×child rows", () => {
    // The worked example from spec §7
    // Input: East/Widgets/100, East/Gadgets/200, West/Widgets/150, West/Gadgets/50
    // GroupOp 1: group by region
    // GroupOp 2: join=true, group by product, columns=[select:region, key:product, SUM(revenue)]
    // Output: 4 rows — (East,Widgets,100), (East,Gadgets,200), (West,Widgets,150), (West,Gadgets,50)
  });

  it("selectedIntervals takes precedence over join", () => { ... });

  it("only final GroupOp columns define output shape", () => { ... });

  it("deferred materialisation — child accesses original dataset columns", () => { ... });
});
```

- [ ] **Step 2: Implement applyGroupSequence**

Process consecutive GroupOps with deferred materialisation. Maintain bucket partitions (row index sets) against the original dataset. Materialise once after the final GroupOp.

- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Commit**

---

### Task 16: sort-eval.ts — applySort()

**Files:**
- Create: `packages/core/src/dataset/sort-eval.ts`
- Create: `packages/core/src/dataset/sort-eval.test.ts`

- [ ] **Step 1: Write failing tests for applySort**

```typescript
// packages/core/src/dataset/sort-eval.test.ts
import { describe, it, expect } from "vitest";
import { applySort } from "./sort-eval.js";
import { toTypedDataSet } from "./conversion.js";
import type { Column, ColumnId } from "./types.js";
import { ColumnType } from "./types.js";
import type { SortOp } from "./sort.js";

function col(id: string, name: string, type: ColumnType): Column {
  return { id: id as ColumnId, name, type };
}

describe("applySort", () => {
  it("sorts numbers ascending", () => {
    const ds = toTypedDataSet({
      columns: [col("val", "Value", ColumnType.NUMBER)],
      data: [["30"], ["10"], ["20"]],
    });
    const op: SortOp = { type: "sort", columns: [{ columnId: "val" as ColumnId, order: "ASCENDING" }] };
    const result = applySort(ds, op);
    expect(result.rows[0]!.number("val" as ColumnId)).toBe(10);
    expect(result.rows[1]!.number("val" as ColumnId)).toBe(20);
    expect(result.rows[2]!.number("val" as ColumnId)).toBe(30);
  });

  it("sorts numbers descending", () => {
    const ds = toTypedDataSet({
      columns: [col("val", "Value", ColumnType.NUMBER)],
      data: [["10"], ["30"], ["20"]],
    });
    const op: SortOp = { type: "sort", columns: [{ columnId: "val" as ColumnId, order: "DESCENDING" }] };
    const result = applySort(ds, op);
    expect(result.rows[0]!.number("val" as ColumnId)).toBe(30);
    expect(result.rows[1]!.number("val" as ColumnId)).toBe(20);
    expect(result.rows[2]!.number("val" as ColumnId)).toBe(10);
  });

  it("multi-column sort — breaks ties with second column", () => {
    const ds = toTypedDataSet({
      columns: [
        col("dept", "Department", ColumnType.LABEL),
        col("val", "Value", ColumnType.NUMBER),
      ],
      data: [["B", "2"], ["A", "3"], ["A", "1"]],
    });
    const op: SortOp = { type: "sort", columns: [
      { columnId: "dept" as ColumnId, order: "ASCENDING" },
      { columnId: "val" as ColumnId, order: "ASCENDING" },
    ] };
    const result = applySort(ds, op);
    expect(result.rows[0]!.text("dept" as ColumnId)).toBe("A");
    expect(result.rows[0]!.number("val" as ColumnId)).toBe(1);
    expect(result.rows[1]!.text("dept" as ColumnId)).toBe("A");
    expect(result.rows[1]!.number("val" as ColumnId)).toBe(3);
    expect(result.rows[2]!.text("dept" as ColumnId)).toBe("B");
  });

  it("NULLs sort last regardless of direction", () => {
    const ds = toTypedDataSet({
      columns: [col("val", "Value", ColumnType.NUMBER)],
      data: [["20"], [null], ["10"]],
    });
    const asc: SortOp = { type: "sort", columns: [{ columnId: "val" as ColumnId, order: "ASCENDING" }] };
    const resultAsc = applySort(ds, asc);
    expect(resultAsc.rows[0]!.number("val" as ColumnId)).toBe(10);
    expect(resultAsc.rows[1]!.number("val" as ColumnId)).toBe(20);
    expect(resultAsc.rows[2]!.cell("val" as ColumnId).type).toBe("NULL");

    const desc: SortOp = { type: "sort", columns: [{ columnId: "val" as ColumnId, order: "DESCENDING" }] };
    const resultDesc = applySort(ds, desc);
    expect(resultDesc.rows[0]!.number("val" as ColumnId)).toBe(20);
    expect(resultDesc.rows[1]!.number("val" as ColumnId)).toBe(10);
    expect(resultDesc.rows[2]!.cell("val" as ColumnId).type).toBe("NULL");
  });

  it("unknown column → UNKNOWN_COLUMN error", () => {
    const ds = toTypedDataSet({
      columns: [col("val", "Value", ColumnType.NUMBER)],
      data: [["10"]],
    });
    const op: SortOp = { type: "sort", columns: [{ columnId: "missing" as ColumnId, order: "ASCENDING" }] };
    expect(() => applySort(ds, op)).toThrow("UNKNOWN_COLUMN");
  });

  it("stable sort — equal elements preserve original order", () => {
    const ds = toTypedDataSet({
      columns: [
        col("key", "Key", ColumnType.LABEL),
        col("seq", "Seq", ColumnType.NUMBER),
      ],
      data: [["A", "1"], ["A", "2"], ["A", "3"]],
    });
    const op: SortOp = { type: "sort", columns: [{ columnId: "key" as ColumnId, order: "ASCENDING" }] };
    const result = applySort(ds, op);
    expect(result.rows[0]!.number("seq" as ColumnId)).toBe(1);
    expect(result.rows[1]!.number("seq" as ColumnId)).toBe(2);
    expect(result.rows[2]!.number("seq" as ColumnId)).toBe(3);
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @melviz/core run test -- sort-eval`
Expected: FAIL

- [ ] **Step 3: Implement applySort**

```typescript
// packages/core/src/dataset/sort-eval.ts
import type { TypedDataSet, TypedRow, CellValue } from "./types.js";
import { ColumnType } from "./types.js";
import type { SortOp, SortColumn } from "./sort.js";
import { DataSetError } from "./errors.js";
import { createTypedRow } from "./conversion.js";

export function applySort(ds: TypedDataSet, op: SortOp): TypedDataSet {
  for (const sc of op.columns) {
    if (!ds.columns.find(c => c.id === sc.columnId)) {
      throw new DataSetError("UNKNOWN_COLUMN", `Sort column not found: "${sc.columnId}"`);
    }
  }

  const sorted = [...ds.rows].sort((a, b) => {
    for (const sc of op.columns) {
      const cellA = a.cell(sc.columnId);
      const cellB = b.cell(sc.columnId);
      const cmp = compareCells(cellA, cellB);
      if (cmp !== 0) return sc.order === "ASCENDING" ? cmp : -cmp;
    }
    return 0;
  });

  return { columns: ds.columns, rows: sorted };
}

function compareCells(a: CellValue, b: CellValue): number {
  // NULLs sort last
  if (a.type === "NULL" && b.type === "NULL") return 0;
  if (a.type === "NULL") return 1;
  if (b.type === "NULL") return -1;

  // Same-type comparison
  if (a.type === ColumnType.NUMBER && b.type === ColumnType.NUMBER) {
    return a.value - b.value;
  }
  if (a.type === ColumnType.DATE && b.type === ColumnType.DATE) {
    return a.value.getTime() - b.value.getTime();
  }
  if ((a.type === ColumnType.TEXT || a.type === ColumnType.LABEL) &&
      (b.type === ColumnType.TEXT || b.type === ColumnType.LABEL)) {
    return a.value < b.value ? -1 : a.value > b.value ? 1 : 0;
  }

  return 0;
}
```

**Dependency note:** `createTypedRow` is currently a private function in `conversion.ts`. `group-eval.ts` (Task 14) needs to construct new `TypedRow` instances for grouped output. Export `createTypedRow` from `conversion.ts` as the first step of Task 14 — add `export` keyword, no other changes. `applySort` doesn't need it since it reorders existing rows.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @melviz/core run test -- sort-eval`
Expected: all pass

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/sort-eval.ts packages/core/src/dataset/sort-eval.test.ts
git commit -m "feat: applySort — multi-column stable sort with NULL-last semantics  Refs #2"
```

---

### Task 17: applyOps() — End-to-End Pipeline

**Files:**
- Modify: `packages/core/src/dataset/ops.ts`
- Modify: `packages/core/src/dataset/ops.test.ts`

- [ ] **Step 1: Write failing tests for applyOps**

```typescript
describe("applyOps", () => {
  it("empty ops → returns dataset unchanged", () => { ... });

  it("filter only → applies filter", () => { ... });

  it("filter → group → sort pipeline", () => {
    // Build a 6-row dataset: dept(LABEL), revenue(NUMBER)
    // Filter: revenue > 100
    // Group: by dept, SUM(revenue)
    // Sort: SUM(revenue) DESCENDING
    // Verify correct output
  });

  it("invalid ordering → INVALID_OPERATION error", () => { ... });

  it("consecutive groups with join → delegates to applyGroupSequence", () => { ... });
});
```

- [ ] **Step 2: Implement applyOps**

```typescript
export function applyOps(
  ds: TypedDataSet,
  ops: readonly DataSetOp[],
): TypedDataSet {
  validateOpOrder(ops);

  let current = ds;
  let i = 0;

  while (i < ops.length) {
    const op = ops[i]!;

    if (op.type === "filter") {
      current = applyFilter(current, op);
      i++;
    } else if (op.type === "group") {
      // Collect consecutive GroupOps
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

- [ ] **Step 3: Run tests**

Run: `yarn workspace @melviz/core run test -- ops`
Expected: all pass

- [ ] **Step 4: Run ALL tests**

Run: `yarn workspace @melviz/core run test`
Expected: all pass — no regressions

- [ ] **Step 5: Commit**

```
git add packages/core/src/dataset/ops.ts packages/core/src/dataset/ops.test.ts
git commit -m "feat: applyOps — complete Filter→Group→Sort pipeline  Refs #2"
```
