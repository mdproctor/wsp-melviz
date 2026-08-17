# Selection-Driven Dataset Builders Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #299 — feat: selection-driven dataset builders + YAML desugaring
**Issue group:** #299

**Goal:** Add `selectionSource` field to `ExternalDataSetDef` and a `detailDataset()` builder function so authors can declare selection-driven detail datasets ergonomically in both TypeScript and YAML.

**Architecture:** `selectionSource` is a metadata field on `ExternalDataSetDef` — the runtime doesn't read it; the existing `#{selection.xxx.yyy}` template resolution pipeline on the `handleDefRequest` path handles all fetch gating and re-resolution. `detailDataset()` returns an `ExternalDataSetDef` (not a `DataSourceBinding`) so it enters the template-aware code path. YAML passthrough works without parser changes — raw dataset properties land on `ExternalDataSetDef` directly.

**Tech Stack:** TypeScript 5, Vitest, Yarn workspaces

## Global Constraints

- `ExternalDataSetDef` lives in `@casehubio/pages-data`; `detailDataset()` lives in `@casehubio/pages-ui`
- Build order matters: `packages/pages-data` must build before `packages/pages-ui`
- `detailDataset()` must return `ExternalDataSetDef` (not `DataSourceBinding`) — this is load-bearing for template resolution
- Spread ordering in `detailDataset()`: explicit values last (`{ ...options, uuid, url, selectionSource }`) to prevent runtime override

---

### Task 1: Add `selectionSource` to `ExternalDataSetDef` and `detailDataset()` builder

**Files:**
- Modify: `packages/pages-data/src/dataset/external/types.ts:38-57` — add `selectionSource` field
- Modify: `packages/pages-ui/src/dsl/builders.ts` — add `detailDataset()` function
- Modify: `packages/pages-ui/src/dsl/index.ts:105-109` — export `detailDataset`
- Test: `packages/pages-ui/src/dsl/builders.test.ts`

**Interfaces:**
- Consumes: `dataSetId(id: string): DataSetId` from `@casehubio/pages-data` (already exists in `packages/pages-data/src/dataset/constructors.ts`)
- Consumes: `ExternalDataSetDef` from `@casehubio/pages-data`
- Produces: `detailDataset(id: string, selectionSource: string, url: string, options?: Omit<ExternalDataSetDef, 'uuid' | 'url' | 'selectionSource'>): ExternalDataSetDef`

- [ ] **Step 1: Write failing tests for `detailDataset()`**

Add to `packages/pages-ui/src/dsl/builders.test.ts`. Import `detailDataset` in the existing import block from `"./builders.js"` and add a new describe block after the `bind()` tests:

```typescript
describe("detailDataset()", () => {
  it("creates an ExternalDataSetDef with uuid, url, and selectionSource", () => {
    const result = detailDataset(
      "grade-history",
      "adverse-events",
      "/api/ae/#{selection.adverse-events.id}/history",
    );

    expect(result.uuid).toBe("grade-history");
    expect(result.url).toBe("/api/ae/#{selection.adverse-events.id}/history");
    expect(result.selectionSource).toBe("adverse-events");
  });

  it("forwards optional ExternalDataSetDef fields", () => {
    const result = detailDataset(
      "grade-history",
      "adverse-events",
      "/api/ae/#{selection.adverse-events.id}/history",
      { dataPath: "data.items", refreshTime: "30second" },
    );

    expect(result.uuid).toBe("grade-history");
    expect(result.selectionSource).toBe("adverse-events");
    expect(result.dataPath).toBe("data.items");
    expect(result.refreshTime).toBe("30second");
  });

  it("explicit params override options bag (spread ordering)", () => {
    const result = detailDataset(
      "grade-history",
      "adverse-events",
      "/api/ae/#{selection.adverse-events.id}/history",
      { url: "/should-be-overridden" } as any,
    );

    expect(result.url).toBe("/api/ae/#{selection.adverse-events.id}/history");
    expect(result.selectionSource).toBe("adverse-events");
  });

  it("returns a frozen object", () => {
    const result = detailDataset(
      "grade-history",
      "adverse-events",
      "/api/ae/#{selection.adverse-events.id}/history",
    );

    expect(Object.isFrozen(result)).toBe(true);
  });

  it("throws if URL has no selection template referencing the declared source", () => {
    expect(() =>
      detailDataset("detail", "adverse-events", "/api/static-url"),
    ).toThrow("adverse-events");
  });

  it("throws on typo in selection template source name", () => {
    expect(() =>
      detailDataset(
        "detail",
        "adverse-events",
        "/api/ae/#{selection.advese-events.id}/history",
      ),
    ).toThrow("adverse-events");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter verbose 2>&1 | tail -20`
Expected: FAIL — `detailDataset` is not exported from `"./builders.js"`

- [ ] **Step 3: Add `selectionSource` to `ExternalDataSetDef`**

In `packages/pages-data/src/dataset/external/types.ts`, add `selectionSource` as the last field in `ExternalDataSetDef`:

```typescript
  readonly keyColumn?: string;
  readonly selectionSource?: string;
}
```

- [ ] **Step 4: Implement `detailDataset()` in builders.ts**

Add the import at the top of `packages/pages-ui/src/dsl/builders.ts`:

```typescript
import type { ExternalDataSetDef } from "@casehubio/pages-data";
import { dataSetId } from "@casehubio/pages-data";
```

Note: `dataSetId` may already be imported — check and add only if missing. `ExternalDataSetDef` is likely not imported yet — add it.

Add the function after the `bind()` function (after line ~462):

```typescript
/**
 * Create a selection-driven detail dataset that auto-fetches when a master
 * table row is selected. The URL must contain `#{selection.<selectionSource>.<field>}`
 * templates that resolve against RuntimeContext.selection.
 *
 * @example
 * const gradeHistory = detailDataset(
 *   "grade-history",
 *   "adverse-events",
 *   "/api/ae/#{selection.adverse-events.id}/history",
 * );
 */
export function detailDataset(
  id: string,
  selectionSource: string,
  url: string,
  options?: Omit<ExternalDataSetDef, "uuid" | "url" | "selectionSource">,
): ExternalDataSetDef {
  if (!url.includes(`#{selection.${selectionSource}.`)) {
    throw new Error(
      `detailDataset "${id}": URL must contain #{selection.${selectionSource}.<field>} template`,
    );
  }
  return Object.freeze({
    ...options,
    uuid: dataSetId(id),
    url,
    selectionSource,
  });
}
```

- [ ] **Step 5: Export `detailDataset` from DSL index**

In `packages/pages-ui/src/dsl/index.ts`, add `detailDataset` to the exports from `"./builders.js"`. Add it in the "Dataset helpers" section alongside `bind`:

```typescript
  // Dataset helpers
  bind,
  detailDataset,
  resetGridCounter,
```

Also add the re-export of `ExternalDataSetDef` type in the pages-data re-export section at the bottom:

```typescript
export type { RestSourceOptions, WsTriggerEvent, ExternalDataSetDef } from "@casehubio/pages-data";
```

- [ ] **Step 6: Build pages-data then pages-ui to verify compilation**

Run: `yarn workspace @casehubio/pages-data run build && yarn workspace @casehubio/pages-ui run build`
Expected: Both build successfully with no type errors.

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter verbose 2>&1 | tail -30`
Expected: All `detailDataset()` tests PASS. All existing tests still PASS.

- [ ] **Step 8: Run the YAML parser test to verify no regressions**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter verbose src/parser/page-parser.test.ts 2>&1 | tail -20`
Expected: All existing parser tests PASS. No YAML parser changes were made, so this is a regression check.

- [ ] **Step 9: Run full typecheck**

Run: `yarn typecheck 2>&1 | tail -10`
Expected: No type errors across the monorepo.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add packages/pages-data/src/dataset/external/types.ts packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts packages/pages-ui/src/dsl/index.ts
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "feat(#299): selectionSource on ExternalDataSetDef + detailDataset() builder

Add selectionSource metadata field to ExternalDataSetDef for declaring
selection-driven detail datasets. Add detailDataset() convenience builder
to pages-ui DSL that produces an ExternalDataSetDef (enters the
template-aware handleDefRequest path for #{selection...} resolution).

Refs #299"
```

---

### Task 2: YAML passthrough integration test

**Files:**
- Test: `packages/pages-ui/src/parser/page-parser.test.ts`

**Interfaces:**
- Consumes: `parsePage(raw: unknown): Component` from `packages/pages-ui/src/parser/page-parser.ts`
- Consumes: `ExternalDataSetDef.selectionSource` from Task 1

- [ ] **Step 1: Write the YAML passthrough test**

Add to `packages/pages-ui/src/parser/page-parser.test.ts`. Find the existing `datasets` describe block (or add one). The test verifies that a raw YAML object with `selectionSource` on a dataset passes through to the parsed root component's props unchanged:

```typescript
it("preserves selectionSource on dataset definitions", () => {
  const raw = {
    datasets: [
      { uuid: "adverse-events", url: "/api/adverse-events" },
      {
        uuid: "grade-history",
        url: "/api/ae/#{selection.adverse-events.id}/history",
        selectionSource: "adverse-events",
      },
    ],
    pages: [{ name: "Test", components: [{ html: "hello" }] }],
  };

  const result = parsePage(raw);
  const datasets = result.props?.datasets as Record<string, unknown>[];

  expect(datasets).toHaveLength(2);
  expect(datasets[1]).toMatchObject({
    uuid: "grade-history",
    url: "/api/ae/#{selection.adverse-events.id}/history",
    selectionSource: "adverse-events",
  });
});
```

- [ ] **Step 2: Run the test to verify it passes**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter verbose src/parser/page-parser.test.ts 2>&1 | tail -20`
Expected: PASS — the parser passes datasets through as-is, so `selectionSource` is preserved without any code changes.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add packages/pages-ui/src/parser/page-parser.test.ts
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "test(#299): YAML selectionSource passthrough integration test

Verify that selectionSource on dataset definitions survives the YAML
parse pipeline without transformation. No parser changes needed —
raw dataset properties pass through to the root component's props.

Refs #299"
```

- [ ] **Step 4: Update issue #299 acceptance criteria**

Update the issue body to match the actual implementation. The original AC described `restSource(url, id, { selectionSource })` which won't work (binding path has no template resolution). Replace with the actual API:

```bash
gh issue edit 299 --repo casehubio/casehub-pages --body "$(gh issue view 299 --repo casehubio/casehub-pages --json body --jq '.body')"
```

The acceptance criteria should become:
- `detailDataset("id", "master-ds", "/api/#{selection.master-ds.field}/path")` creates a selection-bound `ExternalDataSetDef`
- `selectionSource` on `ExternalDataSetDef` declares the master dataset dependency
- YAML with `selectionSource` property deserialises correctly
- `detailDataset()` validates that the URL contains a `#{selection.<source>.` template matching the declared source
- Builder API documented with JSDoc example
