# distinctJoin Aggregation + Doc Updates — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `DISTINCTJOIN` aggregation function to pages-data and update stale "dashboard rendering" references in historical docs.

**Architecture:** Extend the existing aggregation switch dispatch with a new `DISTINCTJOIN` case. Extract shared cell-to-string conversion from `joinValues` to eliminate code duplication. Add DSL helper and parser support. Update 2 historical docs.

**Tech Stack:** TypeScript 5, Vitest, pages-data, pages-ui

## Global Constraints

- `noImplicitReturns: true` enforces exhaustive switch coverage — adding a new `Aggregation` variant without handling it in every switch will fail typecheck.
- `AggregationFnType` in `lookup-parser.ts` will be replaced with a derived type `Aggregation["fn"]` — no manual sync needed after this change.
- `getBucketName` in `group-eval.ts` has different NULL semantics (`"null"` string vs skip) — it is intentionally separate from the new `cellValueToString`.

---

### Task 1: Extract `cellValueToString` and add `DISTINCTJOIN` to the type system and aggregation engine

**Files:**
- Modify: `packages/pages-data/src/dataset/group.ts:9-14` (UniversalAggregation type)
- Modify: `packages/pages-data/src/dataset/group-eval.ts:14-44` (computeAggregation switch), `:199-217` (joinValues refactor), `:518-528` (inferAggregateColumnType)
- Test: `packages/pages-data/src/dataset/group-eval.test.ts`

**Interfaces:**
- Produces: `cellValueToString(val: CellValue): string | null` (exported for test verification), `DISTINCTJOIN` case in `computeAggregation`, `DISTINCTJOIN` case in `inferAggregateColumnType`

- [ ] **Step 1: Write failing tests for `DISTINCTJOIN` aggregation**

Add a new `describe("DISTINCTJOIN")` block in `group-eval.test.ts` after the existing aggregation tests. The test file already has helpers `num()`, `text()`, `label()`, `date()`, and `NULL` at the top.

```typescript
describe("DISTINCTJOIN", () => {
  it("deduplicates text values", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [text("a"), text("b"), text("a"), text("c")],
    )).toEqual(text("a, b, c"));
  });

  it("deduplicates number values converted to string", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [num(1), num(2), num(1)],
    )).toEqual(text("1, 2"));
  });

  it("deduplicates date values converted to ISO string", () => {
    const d1 = date(2024, 6, 15);
    const d2 = date(2024, 7, 1);
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [d1, d2, d1],
    )).toEqual(text("2024-06-15T00:00:00.000Z, 2024-07-01T00:00:00.000Z"));
  });

  it("deduplicates label values", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [label("x"), label("y"), label("x")],
    )).toEqual(text("x, y"));
  });

  it("skips NULLs", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [text("a"), NULL, text("b"), NULL],
    )).toEqual(text("a, b"));
  });

  it("returns empty string for empty input", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [],
    )).toEqual(text(""));
  });

  it("returns empty string for all-NULL input", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [NULL, NULL, NULL],
    )).toEqual(text(""));
  });

  it("handles mixed types with dedup", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [text("a"), num(1), label("b"), text("a"), num(1)],
    )).toEqual(text("a, 1, b"));
  });

  it("deduplicates NUMBER and TEXT producing same string", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [num(1), text("1")],
    )).toEqual(text("1"));
  });

  it("deduplicates LABEL and TEXT producing same string", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [label("x"), text("x")],
    )).toEqual(text("x"));
  });

  it("passes through single value", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [text("only")],
    )).toEqual(text("only"));
  });

  it("uses custom separator", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: " | " },
      [text("a"), text("b"), text("a")],
    )).toEqual(text("a | b"));
  });

  it("preserves insertion order (first occurrence wins)", () => {
    expect(computeAggregation(
      { fn: "DISTINCTJOIN", separator: ", " },
      [text("c"), text("a"), text("b"), text("a"), text("c")],
    )).toEqual(text("c, a, b"));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --run group-eval.test.ts`
Expected: FAIL — TypeScript compilation error on `{ fn: "DISTINCTJOIN", separator: ", " }` because `DISTINCTJOIN` is not in the `Aggregation` type union.

- [ ] **Step 3: Add `DISTINCTJOIN` to the type system**

In `packages/pages-data/src/dataset/group.ts`, add to `UniversalAggregation`:

```typescript
export type UniversalAggregation =
  | { readonly fn: "COUNT" }
  | { readonly fn: "DISTINCT" }
  | { readonly fn: "MIN" }
  | { readonly fn: "MAX" }
  | { readonly fn: "JOIN"; readonly separator: string }
  | { readonly fn: "DISTINCTJOIN"; readonly separator: string };
```

- [ ] **Step 4: Extract `cellValueToString` and implement `distinctJoinValues`**

In `packages/pages-data/src/dataset/group-eval.ts`:

First, add and export a shared conversion helper after `cellValueKey`:

```typescript
export function cellValueToString(val: CellValue): string | null {
  if (val.type === "NULL") return null;
  if (val.type === ColumnType.NUMBER) return String(val.value);
  if (val.type === ColumnType.DATE) return val.value.toISOString();
  return val.value;
}
```

Then refactor `joinValues` to use it:

```typescript
function joinValues(values: readonly CellValue[], separator: string): CellValue {
  const parts: string[] = [];
  for (const val of values) {
    const str = cellValueToString(val);
    if (str !== null) parts.push(str);
  }
  return { type: ColumnType.TEXT, value: parts.join(separator) };
}
```

Add `distinctJoinValues` after `joinValues`:

```typescript
function distinctJoinValues(values: readonly CellValue[], separator: string): CellValue {
  const seen = new Set<string>();
  const parts: string[] = [];
  for (const val of values) {
    const str = cellValueToString(val);
    if (str !== null && !seen.has(str)) {
      seen.add(str);
      parts.push(str);
    }
  }
  return { type: ColumnType.TEXT, value: parts.join(separator) };
}
```

Add the switch case in `computeAggregation`:

```typescript
    case "DISTINCTJOIN":
      return distinctJoinValues(values, fn.separator);
```

Add the switch case in `inferAggregateColumnType`:

```typescript
    case "DISTINCTJOIN":
      return ColumnType.TEXT;
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --run group-eval.test.ts`
Expected: ALL PASS

- [ ] **Step 6: Run typecheck to verify exhaustive switch coverage**

Run: `yarn typecheck`
Expected: PASS — `noImplicitReturns` would fail if any switch missed `DISTINCTJOIN`. The lookup-parser switch will fail here because it hasn't been updated yet — that's expected and fixed in Task 2.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-data/src/dataset/group.ts packages/pages-data/src/dataset/group-eval.ts packages/pages-data/src/dataset/group-eval.test.ts
git commit -m "feat: add DISTINCTJOIN aggregation with cellValueToString extraction

Refs #84"
```

---

### Task 2: Parser and DSL helper

**Files:**
- Modify: `packages/pages-data/src/dataset/lookup-parser.ts:45` (AggregationFnType), `:282-304` (parseAggregation switch)
- Modify: `packages/pages-ui/src/dsl/lookup-helpers.ts` (add distinctJoin helper)
- Modify: `packages/pages-ui/src/dsl/index.ts:82-84` (barrel re-export)
- Test: `packages/pages-ui/src/dsl/lookup-helpers.test.ts`

**Interfaces:**
- Consumes: `Aggregation` type from `group.ts` (Task 1)
- Produces: `distinctJoin(source: string, separator?: string): ResultColumn`

- [ ] **Step 1: Write failing tests for `distinctJoin()` DSL helper**

In `packages/pages-ui/src/dsl/lookup-helpers.test.ts`, add to the `"result column helpers"` describe block:

```typescript
  it("distinctJoin() with default separator", () => {
    const c = distinctJoin("names");
    if (c.kind === "aggregate") {
      expect(c.fn).toEqual({ fn: "DISTINCTJOIN", separator: ", " });
    }
  });

  it("distinctJoin() with custom separator", () => {
    const c = distinctJoin("names", "; ");
    if (c.kind === "aggregate") {
      expect(c.fn).toEqual({ fn: "DISTINCTJOIN", separator: "; " });
    }
  });
```

Update the import at the top of the test file to include `distinctJoin`:

```typescript
import {
  lookup,
  groupBy,
  groupByCalendar,
  filterBy,
  and,
  or,
  not,
  sortBy,
  col,
  sum,
  avg,
  count,
  min,
  max,
  distinct,
  join,
  distinctJoin,
} from "./lookup-helpers.js";
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-ui run test -- --run lookup-helpers.test.ts`
Expected: FAIL — `distinctJoin` is not exported from `lookup-helpers.js`.

- [ ] **Step 3: Replace `AggregationFnType` with derived type and add `DISTINCTJOIN` to parser**

In `packages/pages-data/src/dataset/lookup-parser.ts`:

Replace line 45:
```typescript
type AggregationFnType = "COUNT" | "DISTINCT" | "SUM" | "AVERAGE" | "MEDIAN" | "MIN" | "MAX" | "JOIN";
```
With:
```typescript
type AggregationFnType = Aggregation["fn"];
```

Add case in `parseAggregation` function (after the `JOIN` case):
```typescript
    case "DISTINCTJOIN":
      return { fn: "DISTINCTJOIN", separator: separator ?? ", " };
```

- [ ] **Step 4: Add `distinctJoin` helper to lookup-helpers.ts**

In `packages/pages-ui/src/dsl/lookup-helpers.ts`, add after the `join` function:

```typescript
export function distinctJoin(source: string, separator: string = ", "): ResultColumn {
  const fn: Aggregation = Object.freeze({ fn: "DISTINCTJOIN" as const, separator });
  return Object.freeze({
    kind: "aggregate" as const,
    sourceId: source as ColumnId,
    columnId: source as ColumnId,
    fn,
  });
}
```

- [ ] **Step 5: Add barrel re-export in `dsl/index.ts`**

In `packages/pages-ui/src/dsl/index.ts`, add `distinctJoin` to the lookup-helpers re-export block:

```typescript
export {
  // Main lookup builder
  lookup,
  // Group builders
  groupBy,
  groupByCalendar,
  // Filter builders
  filterBy,
  and,
  or,
  not,
  // Sort builder
  sortBy,
  // Result column helpers
  col,
  sum,
  avg,
  count,
  min,
  max,
  distinct,
  join,
  distinctJoin,
} from "./lookup-helpers.js";
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-ui run test -- --run lookup-helpers.test.ts`
Expected: ALL PASS

- [ ] **Step 7: Run full typecheck**

Run: `yarn typecheck`
Expected: PASS — derived `AggregationFnType` resolves correctly, all switches are exhaustive.

- [ ] **Step 8: Commit**

```bash
git add packages/pages-data/src/dataset/lookup-parser.ts packages/pages-ui/src/dsl/lookup-helpers.ts packages/pages-ui/src/dsl/lookup-helpers.test.ts packages/pages-ui/src/dsl/index.ts
git commit -m "feat: add distinctJoin DSL helper and parser support

Replace manual AggregationFnType with derived Aggregation['fn'] for
compile-time sync.

Refs #84"
```

---

### Task 3: Update stale doc references

**Files:**
- Modify: `docs/superpowers/plans/2026-06-20-casehub-pages-rename-plan.md:498,557,671,692`
- Modify: `docs/superpowers/specs/2026-06-19-casehub-pages-rename-design.md:232`

**Interfaces:**
- Consumes: nothing
- Produces: nothing (docs only)

- [ ] **Step 1: Update rename plan — line 498**

Replace:
```
- Project name and description (casehub-pages, foundational dashboard rendering runtime)
```
With:
```
- Project name and description (casehub-pages, web application framework)
```

- [ ] **Step 2: Update rename plan — line 557**

Replace:
```
gh repo create casehubio/casehub-pages --public --description "CaseHub Pages — foundational dashboard rendering runtime"
```
With:
```
gh repo create casehubio/casehub-pages --public --description "CaseHub Pages — web application framework"
```

- [ ] **Step 3: Update rename plan — line 671**

Replace:
```
Add foundation tier entry for casehub-pages. Document capabilities (YAML dashboard rendering, component API, data binding, forms). Document `casehub-pages-dataset` as a wire contract. Note this is the first non-Maven foundation module. Add runtime consumption pattern to Cross-Repo Dependency Map.
```
With:
```
Add foundation tier entry for casehub-pages. Document capabilities (web application framework — layouts, data pipelines, component hosting, forms, event bus). Document `casehub-pages-dataset` as a wire contract. Note this is the first non-Maven foundation module. Add runtime consumption pattern to Cross-Repo Dependency Map.
```

- [ ] **Step 4: Update rename plan — line 692**

Replace:
```
Add table row: `| pages/ | casehubio/casehub-pages | YAML dashboard rendering, component API, forms |`
```
With:
```
Add table row: `| pages/ | casehubio/casehub-pages | web application framework, component API, forms |`
```

- [ ] **Step 5: Update rename design — line 232**

Replace:
```
- `CLAUDE.md` table: `| pages/ | casehubio/casehub-pages | YAML dashboard rendering, component API, forms |`
```
With:
```
- `CLAUDE.md` table: `| pages/ | casehubio/casehub-pages | web application framework, component API, forms |`
```

- [ ] **Step 6: Commit**

```bash
git add docs/superpowers/plans/2026-06-20-casehub-pages-rename-plan.md docs/superpowers/specs/2026-06-19-casehub-pages-rename-design.md
git commit -m "docs: update stale 'dashboard rendering' references to 'web application framework'

Refs #78"
```
