# View State Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Centralize sort/pagination state alongside existing filter state, persist to URL, fix the initialization race condition, make CasehubTable a stateless renderer.

**Architecture:** New `ComponentViewState` map (per-component sort + pagination) joins existing `FilterState` (per-page filters) and `ActiveSlots` (navigation). The data pipeline becomes the single place where all state is applied. Components emit events; the runtime stores state, syncs URLs, and pushes data. The initialization order changes to populate state before rendering.

**Tech Stack:** TypeScript 5, Vitest, JSDOM, Playwright (spot tests only)

## Global Constraints

- `SortColumn` and `SortOrder` types from `@casehubio/pages-data/dist/dataset/sort.js` — reuse, don't duplicate
- `ColumnId` is a branded type (`string & { __brand: "ColumnId" }`) — cast with `as ColumnId` at URL parsing boundaries
- All workspaces use `vitest run` for tests — run per-workspace: `yarn workspace @casehubio/pages-runtime run test`
- Build order: `yarn build:packages` rebuilds all packages; needed after type changes that cross package boundaries
- Spec: `docs/superpowers/specs/2026-06-25-view-state-persistence-design.md`

---

### Task 1: ComponentViewState Module + clearPageFilters

**Files:**
- Create: `packages/pages-runtime/src/component-view-state.ts`
- Create: `packages/pages-runtime/src/component-view-state.test.ts`
- Modify: `packages/pages-runtime/src/cross-filter.ts` — add `clearPageFilters` export
- Modify: `packages/pages-runtime/src/cross-filter.test.ts` — add clearPageFilters tests

**Interfaces:**
- Consumes: `SortColumn` from `@casehubio/pages-data/dist/dataset/sort.js`
- Produces:
  - `ComponentState` interface (`{ readonly sort?: SortColumn; readonly page?: number }`)
  - `ComponentViewState` type (`Map<string, ComponentState>`)
  - `createComponentViewState(): ComponentViewState`
  - `updateSort(state: ComponentViewState, componentId: string, sort: SortColumn | undefined): void`
  - `updatePage(state: ComponentViewState, componentId: string, page: number | undefined): void`
  - `getComponentState(state: ComponentViewState, componentId: string): ComponentState | undefined`
  - `clearPageFilters(filterState: FilterState, pagePath: string): void`

- [ ] **Step 1: Write failing tests for ComponentViewState**

Create `packages/pages-runtime/src/component-view-state.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { ColumnId } from "@casehubio/pages-data/dist/dataset/types.js";
import type { SortColumn } from "@casehubio/pages-data/dist/dataset/sort.js";
import {
  createComponentViewState,
  updateSort,
  updatePage,
  getComponentState,
} from "./component-view-state.js";

const sort = (col: string, order: "ASCENDING" | "DESCENDING"): SortColumn => ({
  columnId: col as ColumnId,
  order,
});

describe("ComponentViewState", () => {
  it("createComponentViewState returns empty map", () => {
    const state = createComponentViewState();
    expect(state.size).toBe(0);
  });

  it("getComponentState returns undefined for unknown component", () => {
    const state = createComponentViewState();
    expect(getComponentState(state, "unknown")).toBeUndefined();
  });

  it("updateSort sets sort for a component", () => {
    const state = createComponentViewState();
    updateSort(state, "t1", sort("Revenue", "DESCENDING"));
    const cs = getComponentState(state, "t1");
    expect(cs?.sort?.columnId).toBe("Revenue");
    expect(cs?.sort?.order).toBe("DESCENDING");
  });

  it("updateSort with undefined clears sort", () => {
    const state = createComponentViewState();
    updateSort(state, "t1", sort("Revenue", "ASCENDING"));
    updateSort(state, "t1", undefined);
    const cs = getComponentState(state, "t1");
    expect(cs?.sort).toBeUndefined();
  });

  it("updatePage sets page for a component", () => {
    const state = createComponentViewState();
    updatePage(state, "t1", 3);
    expect(getComponentState(state, "t1")?.page).toBe(3);
  });

  it("updatePage with undefined clears page", () => {
    const state = createComponentViewState();
    updatePage(state, "t1", 5);
    updatePage(state, "t1", undefined);
    expect(getComponentState(state, "t1")?.page).toBeUndefined();
  });

  it("sort and page are independent per component", () => {
    const state = createComponentViewState();
    updateSort(state, "t1", sort("A", "ASCENDING"));
    updatePage(state, "t2", 7);
    expect(getComponentState(state, "t1")?.sort?.columnId).toBe("A");
    expect(getComponentState(state, "t1")?.page).toBeUndefined();
    expect(getComponentState(state, "t2")?.sort).toBeUndefined();
    expect(getComponentState(state, "t2")?.page).toBe(7);
  });

  it("updateSort replaces previous sort (no accumulation)", () => {
    const state = createComponentViewState();
    updateSort(state, "t1", sort("A", "ASCENDING"));
    updateSort(state, "t1", sort("B", "DESCENDING"));
    expect(getComponentState(state, "t1")?.sort?.columnId).toBe("B");
    expect(getComponentState(state, "t1")?.sort?.order).toBe("DESCENDING");
  });

  it("updateSort preserves existing page", () => {
    const state = createComponentViewState();
    updatePage(state, "t1", 5);
    updateSort(state, "t1", sort("X", "ASCENDING"));
    expect(getComponentState(state, "t1")?.page).toBe(5);
  });

  it("updatePage preserves existing sort", () => {
    const state = createComponentViewState();
    updateSort(state, "t1", sort("X", "ASCENDING"));
    updatePage(state, "t1", 3);
    expect(getComponentState(state, "t1")?.sort?.columnId).toBe("X");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | head -40`

Expected: FAIL — module `./component-view-state.js` not found

- [ ] **Step 3: Implement ComponentViewState**

Create `packages/pages-runtime/src/component-view-state.ts`:

```typescript
import type { SortColumn } from "@casehubio/pages-data/dist/dataset/sort.js";

export interface ComponentState {
  readonly sort?: SortColumn;
  readonly page?: number;
}

export type ComponentViewState = Map<string, ComponentState>;

export function createComponentViewState(): ComponentViewState {
  return new Map();
}

export function updateSort(
  state: ComponentViewState,
  componentId: string,
  sort: SortColumn | undefined,
): void {
  const existing = state.get(componentId);
  state.set(componentId, { ...existing, sort });
}

export function updatePage(
  state: ComponentViewState,
  componentId: string,
  page: number | undefined,
): void {
  const existing = state.get(componentId);
  state.set(componentId, { ...existing, page });
}

export function getComponentState(
  state: ComponentViewState,
  componentId: string,
): ComponentState | undefined {
  return state.get(componentId);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -20`

Expected: All component-view-state tests PASS

- [ ] **Step 5: Write failing test for clearPageFilters**

Add to `packages/pages-runtime/src/cross-filter.test.ts`:

```typescript
import { createFilterState, updateFilter, collectAncestorFilterOps, clearPageFilters } from "./cross-filter.js";

describe("clearPageFilters", () => {
  it("clears all filters for a specific page", () => {
    const fs = createFilterState();
    updateFilter(fs, "Sales", undefined, "region", ["North"], false);
    updateFilter(fs, "Sales", undefined, "year", ["2024"], false);
    clearPageFilters(fs, "Sales");
    const ops = collectAncestorFilterOps(fs, "Sales", undefined);
    expect(ops).toHaveLength(0);
  });

  it("does not affect other pages", () => {
    const fs = createFilterState();
    updateFilter(fs, "Sales", undefined, "region", ["North"], false);
    updateFilter(fs, "HR", undefined, "dept", ["Eng"], false);
    clearPageFilters(fs, "Sales");
    const salesOps = collectAncestorFilterOps(fs, "Sales", undefined);
    const hrOps = collectAncestorFilterOps(fs, "HR", undefined);
    expect(salesOps).toHaveLength(0);
    expect(hrOps).toHaveLength(1);
  });

  it("no-op for page with no filters", () => {
    const fs = createFilterState();
    clearPageFilters(fs, "Unknown");
    expect(fs.size).toBe(0);
  });

  it("clears grouped filters", () => {
    const fs = createFilterState();
    updateFilter(fs, "Sales", "g1", "region", ["North"], false);
    updateFilter(fs, "Sales", undefined, "year", ["2024"], false);
    clearPageFilters(fs, "Sales");
    const ops = collectAncestorFilterOps(fs, "Sales", "g1");
    expect(ops).toHaveLength(0);
  });
});
```

- [ ] **Step 6: Implement clearPageFilters**

Add to `packages/pages-runtime/src/cross-filter.ts`:

```typescript
export function clearPageFilters(
  filterState: FilterState,
  pagePath: string,
): void {
  const pageFilters = filterState.get(pagePath);
  if (pageFilters) {
    for (const [, columnMap] of pageFilters) columnMap.clear();
  }
}
```

- [ ] **Step 7: Run all tests and verify pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -20`

Expected: All tests PASS (component-view-state + cross-filter)

- [ ] **Step 8: Commit**

```
git add packages/pages-runtime/src/component-view-state.ts packages/pages-runtime/src/component-view-state.test.ts packages/pages-runtime/src/cross-filter.ts packages/pages-runtime/src/cross-filter.test.ts
git commit -m "feat: ComponentViewState module + clearPageFilters

Centralized per-component state (sort + pagination) using
SortColumn from pages-data. clearPageFilters added to cross-filter
for popstate full state restoration.

Refs #24"
```

---

### Task 2: DeepLink Redesign + URL Serialization

**Files:**
- Modify: `packages/pages-ui/src/model/page-types.ts` — DeepLink, remove DrillDownStep/LayoutOverride
- Modify: `packages/pages-runtime/src/url.ts` — extend serialization for sort + pagination
- Modify: `packages/pages-runtime/src/url.test.ts` — full round-trip test matrix

**Interfaces:**
- Consumes: `SortOrder` from `@casehubio/pages-data/dist/dataset/sort.js`
- Produces:
  - Revised `DeepLink` (page, filters?, sort?, pagination?)
  - `serializeToUrl(link: DeepLink): string` — extended
  - `parseFromUrl(hash: string): DeepLink` — extended

- [ ] **Step 1: Modify DeepLink type**

Edit `packages/pages-ui/src/model/page-types.ts`. Remove `DrillDownStep`, `LayoutOverride`, and the `parameters`/`drillDown`/`sort` fields from `DeepLink`. Add new `sort` and `pagination` fields:

```typescript
import type { SortOrder } from "@casehubio/pages-data/dist/dataset/sort.js";

export interface DeepLink {
  readonly page: string;
  readonly filters?: Readonly<Record<string, readonly string[]>>;
  readonly sort?: Readonly<Record<string, { readonly columnId: string; readonly order: SortOrder }>>;
  readonly pagination?: Readonly<Record<string, number>>;
}
```

Remove the `DrillDownStep` and `LayoutOverride` interfaces. Keep `ViewState` unchanged for now (Task 6 updates it).

- [ ] **Step 2: Build packages to verify type changes compile**

Run: `yarn build:packages 2>&1 | tail -10`

Expected: Build succeeds (removed types had no consumers beyond the interface definition)

- [ ] **Step 3: Write failing URL round-trip tests**

Add to `packages/pages-runtime/src/url.test.ts` (after existing tests):

```typescript
describe("serializeToUrl — sort", () => {
  it("single component sort", () => {
    const link: DeepLink = {
      page: "Sales",
      sort: { "t1": { columnId: "Revenue", order: "DESCENDING" } },
    };
    expect(serializeToUrl(link)).toBe("#/page/Sales?sort=t1:Revenue:DESCENDING");
  });

  it("multiple component sorts", () => {
    const link: DeepLink = {
      page: "Sales",
      sort: {
        "t1": { columnId: "Revenue", order: "DESCENDING" },
        "t2": { columnId: "Name", order: "ASCENDING" },
      },
    };
    const url = serializeToUrl(link);
    expect(url).toContain("sort=");
    expect(url).toContain("t1:Revenue:DESCENDING");
    expect(url).toContain("t2:Name:ASCENDING");
  });

  it("sort with special characters in component ID and column", () => {
    const link: DeepLink = {
      page: "Sales",
      sort: { "my:table": { columnId: "R&D", order: "ASCENDING" } },
    };
    const url = serializeToUrl(link);
    expect(url).toContain("sort=");
    expect(url).not.toContain("my:table:");
  });

  it("empty sort object omitted", () => {
    const link: DeepLink = { page: "Sales", sort: {} };
    expect(serializeToUrl(link)).toBe("#/page/Sales");
  });
});

describe("serializeToUrl — pagination", () => {
  it("single component page", () => {
    const link: DeepLink = { page: "Sales", pagination: { "t1": 3 } };
    expect(serializeToUrl(link)).toBe("#/page/Sales?page=t1:3");
  });

  it("page 0 omitted", () => {
    const link: DeepLink = { page: "Sales", pagination: { "t1": 0 } };
    expect(serializeToUrl(link)).toBe("#/page/Sales");
  });

  it("multiple components", () => {
    const link: DeepLink = { page: "Sales", pagination: { "t1": 3, "t2": 7 } };
    const url = serializeToUrl(link);
    expect(url).toContain("page=");
    expect(url).toContain("t1:3");
    expect(url).toContain("t2:7");
  });
});

describe("serializeToUrl — combined", () => {
  it("all four dimensions", () => {
    const link: DeepLink = {
      page: "Sales/Revenue",
      filters: { region: ["North"] },
      sort: { "t1": { columnId: "Revenue", order: "DESCENDING" } },
      pagination: { "t1": 2 },
    };
    const url = serializeToUrl(link);
    expect(url).toContain("#/page/Sales/Revenue?");
    expect(url).toContain("filter=region:North");
    expect(url).toContain("sort=t1:Revenue:DESCENDING");
    expect(url).toContain("page=t1:2");
  });
});

describe("parseFromUrl — sort", () => {
  it("parses single sort", () => {
    const link = parseFromUrl("#/page/Sales?sort=t1:Revenue:DESCENDING");
    expect(link.sort).toEqual({ t1: { columnId: "Revenue", order: "DESCENDING" } });
  });

  it("parses multiple sorts", () => {
    const link = parseFromUrl("#/page/Sales?sort=t1:Revenue:DESCENDING,t2:Name:ASCENDING");
    expect(link.sort?.t1).toEqual({ columnId: "Revenue", order: "DESCENDING" });
    expect(link.sort?.t2).toEqual({ columnId: "Name", order: "ASCENDING" });
  });

  it("no sort param returns undefined sort", () => {
    const link = parseFromUrl("#/page/Sales");
    expect(link.sort).toBeUndefined();
  });

  it("malformed sort entry skipped", () => {
    const link = parseFromUrl("#/page/Sales?sort=bad");
    expect(link.sort).toEqual({});
  });
});

describe("parseFromUrl — pagination", () => {
  it("parses single page", () => {
    const link = parseFromUrl("#/page/Sales?page=t1:3");
    expect(link.pagination).toEqual({ t1: 3 });
  });

  it("parses multiple pages", () => {
    const link = parseFromUrl("#/page/Sales?page=t1:3,t2:7");
    expect(link.pagination?.t1).toBe(3);
    expect(link.pagination?.t2).toBe(7);
  });

  it("no page param returns undefined pagination", () => {
    const link = parseFromUrl("#/page/Sales");
    expect(link.pagination).toBeUndefined();
  });
});

describe("round-trip — sort + pagination", () => {
  it("full round-trip with all dimensions", () => {
    const original: DeepLink = {
      page: "Sales/Revenue",
      filters: { region: ["North", "South"], year: ["2024"] },
      sort: { "t1": { columnId: "Revenue", order: "DESCENDING" } },
      pagination: { "t1": 2 },
    };
    const url = serializeToUrl(original);
    const parsed = parseFromUrl(url);
    expect(parsed.page).toBe(original.page);
    expect(parsed.filters).toEqual(original.filters);
    expect(parsed.sort).toEqual(original.sort);
    expect(parsed.pagination).toEqual(original.pagination);
  });

  it("round-trips special characters in component IDs", () => {
    const original: DeepLink = {
      page: "Sales",
      sort: { "my-table": { columnId: "R&D Cost", order: "ASCENDING" } },
      pagination: { "my-table": 5 },
    };
    const url = serializeToUrl(original);
    const parsed = parseFromUrl(url);
    expect(parsed.sort).toEqual(original.sort);
    expect(parsed.pagination).toEqual(original.pagination);
  });

  it("backwards compatibility — old URL without sort/page", () => {
    const link = parseFromUrl("#/page/Overview?filter=region:North");
    expect(link.page).toBe("Overview");
    expect(link.filters).toEqual({ region: ["North"] });
    expect(link.sort).toBeUndefined();
    expect(link.pagination).toBeUndefined();
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | head -40`

Expected: New sort/pagination tests FAIL (serializeToUrl doesn't handle sort/pagination yet)

- [ ] **Step 5: Implement extended serializeToUrl**

Edit `packages/pages-runtime/src/url.ts`. Update the import to use the new `DeepLink` shape, extend `serializeToUrl` to handle `sort` and `pagination`, extend `parseFromUrl` to parse them:

```typescript
import type { DeepLink } from "@casehubio/pages-ui/dist/model/page-types.js";

function encodePagePath(page: string): string {
  return page.split("/").filter(Boolean).map(encodeURIComponent).join("/");
}

function decodePagePath(encoded: string): string {
  return encoded.split("/").filter(Boolean).map(decodeURIComponent).join("/");
}

export function serializeToUrl(link: DeepLink): string {
  let url = `#/page/${encodePagePath(link.page)}`;
  const params: string[] = [];

  if (link.filters) {
    const entries = Object.entries(link.filters).filter(([, v]) => v.length > 0);
    if (entries.length > 0) {
      const filterStr = entries
        .map(([col, values]) => `${encodeURIComponent(col)}:${values.map(encodeURIComponent).join("|")}`)
        .join(",");
      params.push(`filter=${filterStr}`);
    }
  }

  if (link.sort) {
    const entries = Object.entries(link.sort);
    if (entries.length > 0) {
      const sortStr = entries
        .map(([id, s]) => `${encodeURIComponent(id)}:${encodeURIComponent(s.columnId)}:${s.order}`)
        .join(",");
      params.push(`sort=${sortStr}`);
    }
  }

  if (link.pagination) {
    const entries = Object.entries(link.pagination).filter(([, p]) => p > 0);
    if (entries.length > 0) {
      const pageStr = entries
        .map(([id, p]) => `${encodeURIComponent(id)}:${String(p)}`)
        .join(",");
      params.push(`page=${pageStr}`);
    }
  }

  if (params.length > 0) {
    url += `?${params.join("&")}`;
  }
  return url;
}

export function parseFromUrl(hash: string): DeepLink {
  if (!hash || !hash.startsWith("#/page/")) {
    return { page: "" };
  }

  const withoutPrefix = hash.substring("#/page/".length);
  const qIndex = withoutPrefix.indexOf("?");
  const rawPage = qIndex === -1 ? withoutPrefix : withoutPrefix.substring(0, qIndex);
  const page = decodePagePath(rawPage);

  let filters: Record<string, readonly string[]> | undefined;
  let sort: Record<string, { readonly columnId: string; readonly order: "ASCENDING" | "DESCENDING" }> | undefined;
  let pagination: Record<string, number> | undefined;

  if (qIndex !== -1) {
    const queryStr = withoutPrefix.substring(qIndex + 1);
    const params = new URLSearchParams(queryStr);

    const filterStr = params.get("filter");
    if (filterStr) {
      filters = {};
      for (const entry of filterStr.split(",")) {
        const colonIdx = entry.indexOf(":");
        if (colonIdx === -1) continue;
        const col = decodeURIComponent(entry.substring(0, colonIdx));
        const values = entry.substring(colonIdx + 1).split("|").map(decodeURIComponent);
        filters[col] = values;
      }
    }

    const sortStr = params.get("sort");
    if (sortStr) {
      sort = {};
      for (const entry of sortStr.split(",")) {
        const parts = entry.split(":");
        if (parts.length < 3) continue;
        const id = decodeURIComponent(parts[0]!);
        const columnId = decodeURIComponent(parts[1]!);
        const order = parts[2] as "ASCENDING" | "DESCENDING";
        if (order !== "ASCENDING" && order !== "DESCENDING") continue;
        sort[id] = { columnId, order };
      }
    }

    const pageStr = params.get("page");
    if (pageStr) {
      pagination = {};
      for (const entry of pageStr.split(",")) {
        const colonIdx = entry.indexOf(":");
        if (colonIdx === -1) continue;
        const id = decodeURIComponent(entry.substring(0, colonIdx));
        const num = parseInt(entry.substring(colonIdx + 1), 10);
        if (!isNaN(num)) {
          pagination[id] = num;
        }
      }
    }
  }

  return {
    page,
    ...(filters ? { filters } : {}),
    ...(sort ? { sort } : {}),
    ...(pagination ? { pagination } : {}),
  };
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -30`

Expected: All url.test.ts tests PASS (existing + new)

- [ ] **Step 7: Commit**

```
git add packages/pages-ui/src/model/page-types.ts packages/pages-runtime/src/url.ts packages/pages-runtime/src/url.test.ts
git commit -m "feat: DeepLink redesign + URL sort/pagination serialization

Remove unused parameters, drillDown, DrillDownStep, LayoutOverride.
Extend URL format with sort=id:col:order and page=id:num params.
Round-trip tests cover all 8 combinations of state dimensions.

Refs #24"
```

---

### Task 3: Grid Builder ID Fix

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts` — remove grid auto-ID assignment
- Modify: `packages/pages-ui/src/dsl/builders.test.ts` — add/update grid ID tests

**Interfaces:**
- Consumes: none
- Produces: Grid items without `withId()` have `component.id === undefined`

This is a prerequisite for `hasExplicitId` detection in Task 4. Without it, grid items incorrectly appear as user-ID'd.

- [ ] **Step 1: Write failing test**

Add to `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
describe("grid — component IDs", () => {
  it("grid items without withId have component.id undefined", () => {
    const g = grid(12,
      at(0, 0, 6, 1, { type: "bar-chart", props: { lookup: { dataSetId: "x", operations: [] } } }),
    );
    const item = g.items![0]!;
    expect(item.component.id).toBeUndefined();
  });

  it("grid items with withId preserve their ID", () => {
    const g = grid(12,
      at(0, 0, 6, 1, withId("my-chart", { type: "bar-chart", props: { lookup: { dataSetId: "x", operations: [] } } })),
    );
    const item = g.items![0]!;
    expect(item.component.id).toBe("my-chart");
  });

  it("grid container itself gets auto-ID", () => {
    const g = grid(12, at(0, 0, 6, 1, { type: "bar-chart" }));
    expect(g.id).toBeDefined();
    expect(g.id).toMatch(/^grid_/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter verbose 2>&1 | head -30`

Expected: "grid items without withId have component.id undefined" FAILS (currently auto-assigned)

- [ ] **Step 3: Remove grid auto-ID assignment**

Edit `packages/pages-ui/src/dsl/builders.ts`. In the `grid()` function, remove the `itemsWithIds` mapping that auto-assigns IDs, use `items` directly:

```typescript
export function grid(columns: number, ...items: GridItem[]): TypedComponent<"grid"> {
  const gridId = `grid_${String(gridCounter++)}`;
  const props: GridProps = { columns };
  return freeze({
    type: "grid" as const,
    id: gridId,
    props,
    items,
  });
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter verbose 2>&1 | tail -20`

Expected: All tests PASS. Existing grid tests that relied on auto-IDs may need updating — adjust assertions to match new behavior.

- [ ] **Step 5: Commit**

```
git add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts
git commit -m "fix: remove grid auto-ID assignment — component.id reserved for withId

Grid items without withId() now have component.id === undefined.
The renderer's generateId() fallback produces DOM IDs for them.
Prerequisite for hasExplicitId detection in view state persistence.

Refs #24"
```

---

### Task 4: VizTarget + CasehubElement + Registry Changes

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts` — extend `VizTarget` interface
- Modify: `packages/pages-runtime/src/registry.ts` — add `hasExplicitId` to `ComponentEntry`
- Modify: `packages/pages-runtime/src/activation.ts` — set `hasExplicitId` on entry creation
- Modify: `packages/pages-viz/src/base/CasehubElement.ts` — add `activeSort`, `activePage` properties

**Interfaces:**
- Consumes: `SortColumn` from `@casehubio/pages-data/dist/dataset/sort.js`
- Produces:
  - `VizTarget` with `activeSort?: SortColumn` and `activePage?: number`
  - `ComponentEntry` with `hasExplicitId: boolean`
  - `CasehubElement` with `activeSort` and `activePage` getter/setters (no-update)

- [ ] **Step 1: Extend VizTarget interface**

Edit `packages/pages-runtime/src/data-pipeline.ts`. Add imports and extend `VizTarget`:

```typescript
import type { SortColumn } from "@casehubio/pages-data/dist/dataset/sort.js";

export interface VizTarget {
  dataSet: unknown;
  totalRows: number;
  theme: string;
  error: string;
  activeSort?: SortColumn;
  activePage?: number;
}
```

- [ ] **Step 2: Add hasExplicitId to ComponentEntry**

Edit `packages/pages-runtime/src/registry.ts`:

```typescript
export interface ComponentEntry {
  readonly element: HTMLElement;
  readonly vizElement?: CasehubElement<VizComponentProps>;
  readonly component: Component;
  readonly pagePath: string;
  readonly originalLookup?: DataSetLookup;
  readonly hasExplicitId: boolean;
}
```

- [ ] **Step 3: Set hasExplicitId in activation callback**

Edit `packages/pages-runtime/src/activation.ts`. In the `DATA_COMPONENT_TYPES` branch where registry entries are created, add `hasExplicitId`:

```typescript
const hasExplicitId = component.id !== undefined;

const entry = {
  element: el,
  vizElement: vizEl,
  component,
  pagePath,
  hasExplicitId,
  ...(lookup !== undefined && { originalLookup: lookup }),
};
registry.set(componentId, entry);
```

Also set `hasExplicitId: false` on any other registry entries (non-data-component entries if they exist — check the activation callback for other `registry.set` calls).

- [ ] **Step 4: Add activeSort and activePage to CasehubElement**

Edit `packages/pages-viz/src/base/CasehubElement.ts`. Add private backing fields and getter/setter pairs that do NOT call `update()`:

```typescript
import type { SortColumn } from "@casehubio/pages-data/dist/dataset/sort.js";

// Add after existing private fields:
private _activeSort: SortColumn | undefined;
private _activePage: number | undefined;

// Add after existing property getter/setters:
get activeSort(): SortColumn | undefined {
  return this._activeSort;
}

set activeSort(value: SortColumn | undefined) {
  this._activeSort = value;
}

get activePage(): number | undefined {
  return this._activePage;
}

set activePage(value: number | undefined) {
  this._activePage = value;
}
```

- [ ] **Step 5: Build packages to verify all changes compile**

Run: `yarn build:packages 2>&1 | tail -10`

Expected: Build succeeds

- [ ] **Step 6: Run existing tests to verify no regressions**

Run: `yarn workspace @casehubio/pages-runtime run test 2>&1 | tail -10 && yarn workspace @casehubio/pages-viz run test 2>&1 | tail -10`

Expected: All existing tests PASS

- [ ] **Step 7: Commit**

```
git add packages/pages-runtime/src/data-pipeline.ts packages/pages-runtime/src/registry.ts packages/pages-runtime/src/activation.ts packages/pages-viz/src/base/CasehubElement.ts
git commit -m "feat: VizTarget + CasehubElement + registry for view state

VizTarget gains activeSort and activePage metadata fields.
ComponentEntry gains hasExplicitId for URL persistence gating.
CasehubElement gains no-update property setters for sort/page.

Refs #24"
```

---

### Task 5: Pipeline Sort + Pagination Application

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts` — `createDataPipeline` accepts `ComponentViewState`, `pushData` applies sort + pagination
- Modify: `packages/pages-runtime/src/data-pipeline.test.ts` — sort/pagination pipeline tests

**Interfaces:**
- Consumes: `ComponentViewState` (Task 1), `VizTarget` (Task 4), `SortColumn` from pages-data
- Produces: Extended `createDataPipeline(manager, scope, registry, filterState, dataScopeRegistry, componentViewState)` — `pushData` applies sort ops from `ComponentViewState`, applies pagination from `ComponentViewState` + component `pageSize` prop, sets `activeSort` and `activePage` on `VizTarget`

- [ ] **Step 1: Write failing pipeline tests**

Add to `packages/pages-runtime/src/data-pipeline.test.ts`:

```typescript
import { createComponentViewState, updateSort, updatePage } from "./component-view-state.js";
import type { SortColumn } from "@casehubio/pages-data/dist/dataset/sort.js";

// Add to existing helpers:
function makeTarget(): VizTarget {
  return { dataSet: undefined, totalRows: -1, theme: "", error: "", activeSort: undefined, activePage: undefined };
}

describe("pipeline — sort from ComponentViewState", () => {
  it("applies centralized sort to pushed data", () => {
    const manager = createDataSetManager();
    const ds = toTypedDataSet({
      columns: [col("name", "Name", ColumnType.LABEL), col("value", "Value", ColumnType.NUMBER)],
      data: [["B", "2"], ["A", "1"], ["C", "3"]],
    });
    manager.register("test" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("t1", {
      element: document.createElement("div"),
      component: { type: "table" },
      pagePath: "",
      hasExplicitId: true,
    });

    const cvs = createComponentViewState();
    updateSort(cvs, "t1", { columnId: "name" as ColumnId, order: "ASCENDING" } as SortColumn);

    const pipeline = createDataPipeline(
      manager, new Map() as DataSetScope, registry,
      createFilterState(), createDataScopeRegistry(), cvs,
    );

    const target = makeTarget();
    pipeline.handleDataRequest(target, { dataSetId: "test" as DataSetId, operations: [] }, "t1");

    const rows = (target.dataSet as { rows: { cells: { value: unknown }[] }[] }).rows;
    expect(rows[0]!.cells[0]!.value).toBe("A");
    expect(rows[1]!.cells[0]!.value).toBe("B");
    expect(rows[2]!.cells[0]!.value).toBe("C");
  });

  it("sets activeSort on VizTarget", () => {
    const manager = createDataSetManager();
    const ds = toTypedDataSet({
      columns: [col("name", "Name", ColumnType.LABEL)],
      data: [["A"]],
    });
    manager.register("test" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("t1", {
      element: document.createElement("div"),
      component: { type: "table" },
      pagePath: "",
      hasExplicitId: true,
    });

    const cvs = createComponentViewState();
    const sortCol: SortColumn = { columnId: "name" as ColumnId, order: "DESCENDING" };
    updateSort(cvs, "t1", sortCol);

    const pipeline = createDataPipeline(
      manager, new Map() as DataSetScope, registry,
      createFilterState(), createDataScopeRegistry(), cvs,
    );

    const target = makeTarget();
    pipeline.handleDataRequest(target, { dataSetId: "test" as DataSetId, operations: [] }, "t1");

    expect(target.activeSort).toEqual(sortCol);
  });

  it("no centralized sort preserves original lookup sort", () => {
    const manager = createDataSetManager();
    const ds = toTypedDataSet({
      columns: [col("name", "Name", ColumnType.LABEL), col("value", "Value", ColumnType.NUMBER)],
      data: [["B", "2"], ["A", "1"]],
    });
    manager.register("test" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("t1", {
      element: document.createElement("div"),
      component: { type: "table" },
      pagePath: "",
      hasExplicitId: true,
    });

    const cvs = createComponentViewState();
    // No sort in CVS — original lookup sort should be preserved

    const pipeline = createDataPipeline(
      manager, new Map() as DataSetScope, registry,
      createFilterState(), createDataScopeRegistry(), cvs,
    );

    const sortOp = { type: "sort" as const, columns: [{ columnId: "name" as ColumnId, order: "ASCENDING" as const }] };
    const target = makeTarget();
    pipeline.handleDataRequest(target, { dataSetId: "test" as DataSetId, operations: [sortOp] }, "t1");

    const rows = (target.dataSet as { rows: { cells: { value: unknown }[] }[] }).rows;
    expect(rows[0]!.cells[0]!.value).toBe("A");
  });

  it("centralized sort replaces original lookup sort", () => {
    const manager = createDataSetManager();
    const ds = toTypedDataSet({
      columns: [col("name", "Name", ColumnType.LABEL)],
      data: [["B"], ["A"], ["C"]],
    });
    manager.register("test" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("t1", {
      element: document.createElement("div"),
      component: { type: "table" },
      pagePath: "",
      hasExplicitId: true,
    });

    const cvs = createComponentViewState();
    updateSort(cvs, "t1", { columnId: "name" as ColumnId, order: "DESCENDING" } as SortColumn);

    const pipeline = createDataPipeline(
      manager, new Map() as DataSetScope, registry,
      createFilterState(), createDataScopeRegistry(), cvs,
    );

    // Original lookup has ASCENDING sort — centralized DESCENDING should win
    const sortOp = { type: "sort" as const, columns: [{ columnId: "name" as ColumnId, order: "ASCENDING" as const }] };
    const target = makeTarget();
    pipeline.handleDataRequest(target, { dataSetId: "test" as DataSetId, operations: [sortOp] }, "t1");

    const rows = (target.dataSet as { rows: { cells: { value: unknown }[] }[] }).rows;
    expect(rows[0]!.cells[0]!.value).toBe("C");
  });
});

describe("pipeline — pagination from ComponentViewState", () => {
  it("applies pagination when pageSize prop exists", () => {
    const manager = createDataSetManager();
    const ds = toTypedDataSet({
      columns: [col("name", "Name", ColumnType.LABEL)],
      data: [["A"], ["B"], ["C"], ["D"], ["E"]],
    });
    manager.register("test" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("t1", {
      element: document.createElement("div"),
      component: { type: "table", props: { pageSize: 2, lookup: { dataSetId: "test", operations: [] } } },
      pagePath: "",
      hasExplicitId: true,
    });

    const cvs = createComponentViewState();
    updatePage(cvs, "t1", 1); // page 1 = rows 2-3

    const pipeline = createDataPipeline(
      manager, new Map() as DataSetScope, registry,
      createFilterState(), createDataScopeRegistry(), cvs,
    );

    const target = makeTarget();
    pipeline.handleDataRequest(target, { dataSetId: "test" as DataSetId, operations: [] }, "t1");

    const rows = (target.dataSet as { rows: { cells: { value: unknown }[] }[] }).rows;
    expect(rows).toHaveLength(2);
    expect(rows[0]!.cells[0]!.value).toBe("C");
    expect(rows[1]!.cells[0]!.value).toBe("D");
    expect(target.activePage).toBe(1);
  });

  it("clamps page when beyond total", () => {
    const manager = createDataSetManager();
    const ds = toTypedDataSet({
      columns: [col("name", "Name", ColumnType.LABEL)],
      data: [["A"], ["B"], ["C"]],
    });
    manager.register("test" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("t1", {
      element: document.createElement("div"),
      component: { type: "table", props: { pageSize: 2, lookup: { dataSetId: "test", operations: [] } } },
      pagePath: "",
      hasExplicitId: true,
    });

    const cvs = createComponentViewState();
    updatePage(cvs, "t1", 5); // way beyond total

    const pipeline = createDataPipeline(
      manager, new Map() as DataSetScope, registry,
      createFilterState(), createDataScopeRegistry(), cvs,
    );

    const target = makeTarget();
    pipeline.handleDataRequest(target, { dataSetId: "test" as DataSetId, operations: [] }, "t1");

    const rows = (target.dataSet as { rows: { cells: { value: unknown }[] }[] }).rows;
    expect(rows).toHaveLength(1); // last page: row C
    expect(rows[0]!.cells[0]!.value).toBe("C");
    expect(target.activePage).toBe(1); // clamped to last page
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | head -40`

Expected: New pipeline tests FAIL (createDataPipeline doesn't accept componentViewState)

- [ ] **Step 3: Implement pipeline sort + pagination**

Edit `packages/pages-runtime/src/data-pipeline.ts`. Add `componentViewState` parameter to `createDataPipeline`. Modify `pushData` to read sort/pagination from `ComponentViewState` and apply them. The implementation follows the spec §5 logic:

1. Read `ComponentState` for the component
2. Strip existing sort ops from lookup, append centralized sort (if any)
3. Compute pagination options from `ComponentState.page` + component `pageSize` prop
4. Execute `manager.lookup` with pagination options
5. Clamp page if result is empty but totalRows > 0
6. Set `activeSort`, `activePage` on target BEFORE `dataSet`

This is a substantial edit to `data-pipeline.ts` — modify `pushData` and `handleDataRequest`. The componentId parameter is already passed to `handleDataRequest`; thread it to `pushData`.

- [ ] **Step 4: Update existing data-pipeline tests for new createDataPipeline signature**

Add `createComponentViewState()` as the 6th argument to all existing `createDataPipeline` calls in `data-pipeline.test.ts`. Update `makeTarget()` to include `activeSort: undefined, activePage: undefined`.

- [ ] **Step 5: Run all pipeline tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -30`

Expected: All tests PASS (existing + new sort/pagination)

- [ ] **Step 6: Commit**

```
git add packages/pages-runtime/src/data-pipeline.ts packages/pages-runtime/src/data-pipeline.test.ts
git commit -m "feat: pipeline applies sort + pagination from ComponentViewState

pushData reads centralized sort/page state, applies sort ops
(replacing original lookup sort), applies pagination via
LookupOptions, clamps page when beyond total, sets activeSort
and activePage on VizTarget.

Refs #24"
```

---

### Task 6: CasehubTable Stateless Refactor

**Files:**
- Modify: `packages/pages-viz/src/components/CasehubTable.ts` — remove private sort/page state, always emit events, read from VizTarget
- Modify: `packages/pages-viz/src/components/CasehubTable.test.ts` — update tests for stateless behavior

**Interfaces:**
- Consumes: `VizTarget.activeSort`, `VizTarget.activePage` (Task 4)
- Produces: CasehubTable always emits `casehub-sort` and `casehub-page` events (no `serverSide` fork)

- [ ] **Step 1: Write failing tests for stateless table behavior**

Add to `packages/pages-viz/src/components/CasehubTable.test.ts`:

```typescript
describe("stateless sort/pagination", () => {
  it("always emits casehub-sort on column header click", () => {
    const ds = makeDataSet([["name", "LABEL"], ["value", "NUMBER"]], [["A", 1], ["B", 2]]);
    const props: TableProps = { lookup: mockLookup("test"), sortable: true };

    el.props = props;
    document.body.appendChild(el);
    el.dataSet = ds;

    const events: CustomEvent[] = [];
    el.addEventListener("casehub-sort", ((e: Event) => { events.push(e as CustomEvent); }) as EventListener);

    const headers = queryHeaders(el);
    headers[0]!.click();

    expect(events).toHaveLength(1);
    expect(events[0]!.detail.columnId).toBe("name");
    expect(events[0]!.detail.order).toBe("ASCENDING");
  });

  it("reads activeSort for sort indicator", () => {
    const ds = makeDataSet([["name", "LABEL"], ["value", "NUMBER"]], [["A", 1], ["B", 2]]);
    const props: TableProps = { lookup: mockLookup("test"), sortable: true };

    el.props = props;
    document.body.appendChild(el);
    el.activeSort = { columnId: "name" as ColumnId, order: "DESCENDING" };
    el.dataSet = ds;

    const headers = queryHeaders(el);
    expect(headers[0]!.textContent).toContain("▼");
  });

  it("toggles sort order based on activeSort", () => {
    const ds = makeDataSet([["name", "LABEL"]], [["A"], ["B"]]);
    const props: TableProps = { lookup: mockLookup("test"), sortable: true };

    el.props = props;
    document.body.appendChild(el);
    el.activeSort = { columnId: "name" as ColumnId, order: "ASCENDING" };
    el.dataSet = ds;

    const events: CustomEvent[] = [];
    el.addEventListener("casehub-sort", ((e: Event) => { events.push(e as CustomEvent); }) as EventListener);

    const headers = queryHeaders(el);
    headers[0]!.click(); // same column → should toggle to DESCENDING

    expect(events[0]!.detail.order).toBe("DESCENDING");
  });

  it("always emits casehub-page on page button click", () => {
    const ds = makeDataSet(
      [["name", "LABEL"]],
      [["A"], ["B"], ["C"], ["D"], ["E"]],
    );
    const props: TableProps = { lookup: mockLookup("test"), pageSize: 2 };

    el.props = props;
    document.body.appendChild(el);
    el.activePage = 0;
    el.totalRows = 5;
    el.dataSet = ds;

    const events: CustomEvent[] = [];
    el.addEventListener("casehub-page", ((e: Event) => { events.push(e as CustomEvent); }) as EventListener);

    const nextBtn = el.shadowRoot.querySelector(".paging button:nth-child(5)") as HTMLButtonElement;
    if (nextBtn && !nextBtn.disabled) nextBtn.click();

    expect(events.length).toBeGreaterThan(0);
    expect(events[0]!.detail.offset).toBe(2);
    expect(events[0]!.detail.count).toBe(2);
  });

  it("renders pagination controls from activePage", () => {
    const ds = makeDataSet(
      [["name", "LABEL"]],
      [["C"], ["D"]],
    );
    const props: TableProps = { lookup: mockLookup("test"), pageSize: 2 };

    el.props = props;
    document.body.appendChild(el);
    el.activePage = 1;
    el.totalRows = 5;
    el.dataSet = ds;

    const pageInput = el.shadowRoot.querySelector(".paging input") as HTMLInputElement;
    expect(pageInput?.value).toBe("2"); // page 1 → display "2"
  });

  it("renders all received rows without local slicing", () => {
    const ds = makeDataSet(
      [["name", "LABEL"]],
      [["A"], ["B"]], // pipeline already paginated — 2 rows
    );
    const props: TableProps = { lookup: mockLookup("test"), pageSize: 2 };

    el.props = props;
    document.body.appendChild(el);
    el.activePage = 0;
    el.totalRows = 5;
    el.dataSet = ds;

    const rows = queryRows(el);
    expect(rows).toHaveLength(2); // renders exactly what pipeline provided
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-viz run test -- --reporter verbose 2>&1 | head -40`

Expected: New tests FAIL (table doesn't have `activeSort`/`activePage`, still uses private fields)

- [ ] **Step 3: Refactor CasehubTable**

Edit `packages/pages-viz/src/components/CasehubTable.ts`:

1. Remove private fields: `_currentPage`, `_sortColumn`, `_sortOrder`, `_lastDataSet`
2. Remove methods: `getSortedRows()`, `isServerSide()`
3. Modify `handleSort()`: remove `serverSide` parameter, read toggle from `this.activeSort`, always emit event
4. Modify `goToPage()`: remove `serverSide` and extra params, always emit event, no `_currentPage` update
5. Modify `render()`: use `this.activePage ?? 0` instead of `_currentPage`, use `this.activeSort` for indicators, `displayRows = dataset.rows` (no local sort/slice), `totalCount = this.totalRows` always
6. Remove the `dataSet` setter override that checked `_lastDataSet`

- [ ] **Step 4: Update existing CasehubTable tests**

Some existing tests may assert local sorting or pagination behavior. Update them to work with the stateless model: set `activeSort`/`activePage` before `dataSet`, verify event emissions rather than internal state changes.

- [ ] **Step 5: Run all viz tests**

Run: `yarn workspace @casehubio/pages-viz run test -- --reporter verbose 2>&1 | tail -20`

Expected: All tests PASS

- [ ] **Step 6: Commit**

```
git add packages/pages-viz/src/components/CasehubTable.ts packages/pages-viz/src/components/CasehubTable.test.ts
git commit -m "refactor: CasehubTable — stateless sort/pagination

Remove _sortColumn, _sortOrder, _currentPage, _lastDataSet,
getSortedRows(), isServerSide(). Always emit casehub-sort and
casehub-page events. Read sort/page from VizTarget activeSort
and activePage. Table renders exactly what the pipeline provides.

Refs #24"
```

---

### Task 7: Site.ts Orchestration

**Files:**
- Modify: `packages/pages-ui/src/model/page-types.ts` — `ViewState` non-optional fields, add sort/pagination
- Modify: `packages/pages-runtime/src/site.ts` — initialization reorder, `navigateInternal`, simplified event handlers, `syncUrl` extended, `popstate` full restore, `restoreFromUrl`
- Modify: `packages/pages-runtime/src/index.ts` — export new types
- Modify: `packages/pages-runtime/src/site.test.ts` — orchestration tests

**Interfaces:**
- Consumes: Tasks 1–6 (all)
- Produces: Complete orchestrated system — `loadSite` with state-before-render initialization, `navigateInternal` for popstate, simplified event handlers, extended `syncUrl`, `restoreFromUrl`, `ViewState` live snapshot with sort/pagination

This is the integration task. It wires everything together.

- [ ] **Step 1: Update ViewState in page-types.ts**

```typescript
import type { SortOrder } from "@casehubio/pages-data/dist/dataset/sort.js";

export interface ViewState {
  readonly currentPage: string;
  readonly activeFilters: Readonly<Record<string, readonly string[]>>;
  readonly sort: Readonly<Record<string, { readonly columnId: string; readonly order: SortOrder }>>;
  readonly pagination: Readonly<Record<string, number>>;
}
```

- [ ] **Step 2: Write failing site tests**

Add to `packages/pages-runtime/src/site.test.ts`:

```typescript
describe("view state persistence", () => {
  it("sort event updates URL", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    // Build a site with a table that has an explicit ID
    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [{ uuid: "ds" as DataSetId, content: JSON.stringify([{ name: "A", value: 1 }, { name: "B", value: 2 }]) }] },
      slots: { default: [{ type: "table", id: "t1", props: { lookup: { dataSetId: "ds" as DataSetId, operations: [] }, sortable: true } }] },
    };

    const site = await loadSite(target, root);

    // Simulate sort event
    const tableEl = target.querySelector("[data-component-id='t1']") as HTMLElement;
    tableEl.dispatchEvent(new CustomEvent("casehub-sort", {
      bubbles: true, composed: true,
      detail: { columnId: "name", order: "ASCENDING" },
    }));

    expect(location.hash).toContain("sort=t1:name:ASCENDING");

    site.dispose();
    document.body.removeChild(target);
  });

  it("page event updates URL", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [{ uuid: "ds" as DataSetId, content: JSON.stringify([{ a: 1 }, { a: 2 }, { a: 3 }, { a: 4 }, { a: 5 }]) }] },
      slots: { default: [{ type: "table", id: "t1", props: { lookup: { dataSetId: "ds" as DataSetId, operations: [] }, pageSize: 2 } }] },
    };

    const site = await loadSite(target, root);

    const tableEl = target.querySelector("[data-component-id='t1']") as HTMLElement;
    tableEl.dispatchEvent(new CustomEvent("casehub-page", {
      bubbles: true, composed: true,
      detail: { offset: 4, count: 2 },
    }));

    expect(location.hash).toContain("page=t1:2");

    site.dispose();
    document.body.removeChild(target);
  });

  it("sort resets pagination to 0", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [{ uuid: "ds" as DataSetId, content: JSON.stringify([{ a: 1 }, { a: 2 }, { a: 3 }]) }] },
      slots: { default: [{ type: "table", id: "t1", props: { lookup: { dataSetId: "ds" as DataSetId, operations: [] }, pageSize: 2, sortable: true } }] },
    };

    const site = await loadSite(target, root);
    const tableEl = target.querySelector("[data-component-id='t1']") as HTMLElement;

    // Set page to 1
    tableEl.dispatchEvent(new CustomEvent("casehub-page", {
      bubbles: true, composed: true,
      detail: { offset: 2, count: 2 },
    }));
    expect(location.hash).toContain("page=t1:1");

    // Sort should reset page
    tableEl.dispatchEvent(new CustomEvent("casehub-sort", {
      bubbles: true, composed: true,
      detail: { columnId: "a", order: "DESCENDING" },
    }));
    expect(location.hash).toContain("sort=t1:a:DESCENDING");
    expect(location.hash).not.toContain("page=");

    site.dispose();
    document.body.removeChild(target);
  });

  it("ViewState exposes sort and pagination", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [{ uuid: "ds" as DataSetId, content: JSON.stringify([{ a: 1 }]) }] },
      slots: { default: [{ type: "table", id: "t1", props: { lookup: { dataSetId: "ds" as DataSetId, operations: [] }, sortable: true } }] },
    };

    const site = await loadSite(target, root);
    const tableEl = target.querySelector("[data-component-id='t1']") as HTMLElement;

    tableEl.dispatchEvent(new CustomEvent("casehub-sort", {
      bubbles: true, composed: true,
      detail: { columnId: "a", order: "ASCENDING" },
    }));

    expect(site.state.sort).toEqual({ t1: { columnId: "a", order: "ASCENDING" } });

    site.dispose();
    document.body.removeChild(target);
  });

  it("table without explicit ID — sort works but not in URL", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [{ uuid: "ds" as DataSetId, content: JSON.stringify([{ a: 1 }]) }] },
      slots: { default: [{ type: "table", props: { lookup: { dataSetId: "ds" as DataSetId, operations: [] }, sortable: true } }] },
    };

    const site = await loadSite(target, root);

    // Find the auto-ID'd table
    const tableEl = target.querySelector("[data-component-type='table']") as HTMLElement;
    tableEl.dispatchEvent(new CustomEvent("casehub-sort", {
      bubbles: true, composed: true,
      detail: { columnId: "a", order: "ASCENDING" },
    }));

    expect(location.hash).not.toContain("sort=");

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 3: Implement site.ts changes**

This is the largest edit. Modify `packages/pages-runtime/src/site.ts`:

1. Import `createComponentViewState`, `updateSort`, `updatePage`, `getComponentState` from `./component-view-state.js`
2. Import `clearPageFilters` from `./cross-filter.js`
3. Create `componentViewState` in `loadSite`
4. Add `restoreFromUrl` function
5. Reorder initialization: parse URL → populate state → listeners → render → navigate → canonicalize
6. Extract `navigateInternal` from `navigate`
7. Extend `syncUrl` with `deriveUrlSort` and `deriveUrlPagination`
8. Simplify `casehub-sort` handler: update state → reset page → re-push → syncUrl
9. Simplify `casehub-page` handler: derive page → update state → re-push → syncUrl
10. Add pagination reset in `casehub-filter` handler's re-push loop
11. Update `popstate` handler: `navigateInternal` → clear → restore → re-push all
12. Update `ViewState` snapshot with sort/pagination getters
13. Pass `componentViewState` to `createDataPipeline`
14. Add `componentViewState.clear()` to `dispose()`

- [ ] **Step 4: Update exports in index.ts**

Add new exports to `packages/pages-runtime/src/index.ts`:

```typescript
export { createComponentViewState, updateSort, updatePage, getComponentState } from "./component-view-state.js";
export type { ComponentState, ComponentViewState } from "./component-view-state.js";
export { clearPageFilters } from "./cross-filter.js";
```

- [ ] **Step 5: Build everything**

Run: `yarn build:packages 2>&1 | tail -10`

Expected: Build succeeds

- [ ] **Step 6: Run all runtime tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -30`

Expected: All tests PASS

- [ ] **Step 7: Commit**

```
git add packages/pages-ui/src/model/page-types.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/index.ts packages/pages-runtime/src/site.test.ts
git commit -m "feat: site.ts — view state orchestration

Initialization reorder: state before render fixes filter race.
navigateInternal extracted for popstate (no history corruption).
casehub-sort/page handlers simplified to state update + re-push.
Filter handler resets pagination for affected components.
syncUrl serializes sort/pagination for explicit-ID components.
popstate does full state restoration from URL.
ViewState snapshot exposes sort and pagination.

Refs #24"
```

---

### Task 8: Integration Lifecycle Tests

**Files:**
- Modify: `packages/pages-runtime/src/site.test.ts` — deep combinatorial lifecycle tests

**Interfaces:**
- Consumes: Full integrated system (Tasks 1–7)
- Produces: Comprehensive test coverage for nested pages, linked components, multiple tables, popstate, edge cases

This task adds the deep combinatorial tests from spec §9 categories D–L. Tests in Task 7 covered the flat-page basics. This task covers nesting, cross-filter interactions, and edge cases.

- [ ] **Step 1: Build nested page test fixtures**

Add test helper functions to `site.test.ts` for 2-level and 3-level nested component trees with explicit-ID'd tables, selectors, and filter-linked components.

- [ ] **Step 2: Write 2-level nesting tests (spec §9E)**

Tests: load URL with deep page path + sort, sort on inactive tab, navigate between tabs preserving state, filters scoped to correct page.

- [ ] **Step 3: Write 3-level nesting tests (spec §9F)**

Tests: load URL with 3-level path + sort, full path in URL after sort, navigate away and back.

- [ ] **Step 4: Write cross-filter + sort interaction tests (spec §9G)**

Tests: sort then filter (sort preserved, page resets), filter then sort, sort + filter + paginate then clear filter, filter group isolation.

- [ ] **Step 5: Write multiple tables same page tests (spec §9I)**

Tests: independent sort, independent pagination, filter affects both, filter affects one (different groups), URL contains both tables' state.

- [ ] **Step 6: Write popstate tests (spec §9K)**

Tests: navigate + back restores state, sort + navigate + back, two levels of back, navigateInternal used (no duplicate history entries).

- [ ] **Step 7: Write edge case tests (spec §9L)**

Tests: pagination clamp on filter, stale sort column, empty dataset, dispose cleanup, ID collision across pages.

- [ ] **Step 8: Write race condition verification test (spec §9D addition)**

Assert pipeline `handleDataRequest` is called once per component during initial render with correct state — not twice.

- [ ] **Step 9: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -40`

Expected: All tests PASS

- [ ] **Step 10: Commit**

```
git add packages/pages-runtime/src/site.test.ts
git commit -m "test: deep lifecycle tests — nesting, cross-filter, popstate, edge cases

2-level and 3-level nested page tests, cross-filter + sort
interaction matrix, multiple tables same page, popstate full
restore, pagination clamp, stale sort column, race condition
verification.

Refs #24"
```

---

### Task 9: Playwright Spot Tests + Documentation

**Files:**
- Create: `examples/e2e/view-state.spec.ts` (or extend existing Playwright test file)
- Modify: `docs/CASEHUB-PAGES.md` — update ViewState, DeepLink documentation

**Interfaces:**
- Consumes: Full system (Tasks 1–8), examples gallery with tables that have explicit IDs
- Produces: 6 Playwright spot tests, updated LLM guide

- [ ] **Step 1: Add withId to an example dashboard table**

The examples gallery needs at least one table with `withId()` for the Playwright tests to target. Find an existing example with a table and add `withId("test-table", ...)` to it.

- [ ] **Step 2: Write Playwright spot tests**

Create `examples/e2e/view-state.spec.ts` with 6 tests:

1. Sort table column → URL contains sort param
2. Sort table → reload page → sort indicator present
3. Sort table → navigate to different tab → click back → sort indicator restored
4. Paginate table → URL contains page param
5. Sort + filter + paginate → reload → all three present
6. Sort table without `withId` → reload → sort gone

- [ ] **Step 3: Update CASEHUB-PAGES.md**

Update the LiveSite, ViewState, and DeepLink sections in `docs/CASEHUB-PAGES.md` to reflect the new types and URL format. Document `withId()` as the opt-in for URL state persistence.

- [ ] **Step 4: Run Playwright tests**

Run: `yarn workspace @casehubio/pages-examples run test 2>&1 | tail -20`

Expected: All 6 spot tests PASS

- [ ] **Step 5: Commit**

```
git add examples/ docs/CASEHUB-PAGES.md
git commit -m "test: Playwright spot tests + docs update for view state

6 E2E browser tests: sort/page URL serialization, reload
restoration, back/forward, explicit ID gating.
CASEHUB-PAGES.md updated with new ViewState, DeepLink, withId docs.

Closes #24"
```

---

## Self-Review

**Spec coverage:**
- §1 State Model → Task 1
- §2 Type Changes → Tasks 2 (DeepLink), 4 (VizTarget, registry), 7 (ViewState)
- §3 URL Format → Task 2
- §4 State Lifecycle → Tasks 7 (site.ts), 8 (integration tests)
- §5 Pipeline Changes → Task 5
- §6 Component Changes → Task 6
- §7 syncUrl + Restoration → Task 7
- §8 Explicit ID Detection → Tasks 3 (grid fix), 4 (hasExplicitId)
- §9 Testing Strategy → Tasks 1–9 (each task has TDD tests)

**Placeholder scan:** All steps contain actual code or explicit instructions. No "TBD", "TODO", "similar to", or "add appropriate" patterns.

**Type consistency:** `SortColumn` used consistently from `@casehubio/pages-data/dist/dataset/sort.js`. `activePage` (not `currentPage`) on VizTarget and CasehubElement. `columnId` (not `column`) in DeepLink sort. `ComponentState` fields readonly. `hasExplicitId` on ComponentEntry.
