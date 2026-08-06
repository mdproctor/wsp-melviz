# Server-Side Pagination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #12 — Add lazy on-demand pagination for datasets
**Issue group:** #12

**Goal:** Add server-side pagination so datasets fetch only the visible page from the server, with LRU caching of previously fetched pages.

**Architecture:** Pipeline-level pagination (Approach 3). No new DataSource types. The pipeline detects `serverPagination` config on `ExternalDataSetDef`, manages page fetches directly via template URLs, and caches results in a per-dataset `PageCache`. Existing `pages-page`, `pages-sort`, `pages-filter` events drive the fetch cycle.

**Tech Stack:** TypeScript, Vitest, Lit (pages-table), pages-data, pages-runtime

## Global Constraints

- No changes to the `DataSource` interface (`connect`/`disconnect`)
- No `SourceConnector` created for server-paged datasets
- Client-side sort/filter operations must be blocked on server-paged datasets with a warning
- `totalRows` always comes from the server response via `totalPath`
- Template URL param names are user-configurable (not hardcoded)

---

### Task 1: Types — ServerPaginationConfig and ExternalDataSetDef extension

**Files:**
- Modify: `packages/pages-data/src/dataset/external/types.ts`
- Modify: `packages/pages-data/src/datasource/types.ts`
- Test: `packages/pages-data/src/dataset/external/types.test.ts` (if exists, else inline type-check)

**Interfaces:**
- Consumes: nothing (foundational types)
- Produces: `ServerPaginationConfig` interface, `serverPagination?: ServerPaginationConfig` on `ExternalDataSetDef`, `serverPagination?: ServerPaginationConfig` on `DataSourceBinding`

- [ ] **Step 1: Add ServerPaginationConfig interface to types.ts**

```typescript
export interface ServerPaginationConfig {
  readonly offsetParam: string;
  readonly limitParam: string;
  readonly sortParam?: string;
  readonly orderParam?: string;
  readonly filterParam?: string;
  readonly defaultPageSize: number;
  readonly maxCachedPages?: number;
}
```

Add to `ExternalDataSetDef`:
```typescript
readonly serverPagination?: ServerPaginationConfig;
```

Use `ide_insert_member` to add the field after the existing `serverQuery` field at line 32 of `packages/pages-data/src/dataset/external/types.ts`.

- [ ] **Step 2: Add serverPagination to DataSourceBinding**

In `packages/pages-data/src/datasource/types.ts`, add to `DataSourceBinding`:
```typescript
readonly serverPagination?: ServerPaginationConfig;
```

Import `ServerPaginationConfig` from `../dataset/external/types.js`.

- [ ] **Step 3: Export ServerPaginationConfig from package index**

In `packages/pages-data/src/index.ts`, add `ServerPaginationConfig` to the type exports from `./dataset/external/types.js`.

- [ ] **Step 4: Verify typecheck passes**

Run: `yarn typecheck`
Expected: clean (no errors)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/dataset/external/types.ts packages/pages-data/src/datasource/types.ts packages/pages-data/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#12): add ServerPaginationConfig type and ExternalDataSetDef extension"
```

---

### Task 2: PageCache — LRU page cache

**Files:**
- Create: `packages/pages-data/src/dataset/page-cache.ts`
- Create: `packages/pages-data/src/dataset/page-cache.test.ts`
- Modify: `packages/pages-data/src/index.ts` (export)

**Interfaces:**
- Consumes: `TypedDataSet` from `./types.js`
- Produces: `PageCache` class with `get(key): CachedPage | undefined`, `store(key, page): void`, `clear(): void`, `size: number`

- [ ] **Step 1: Write failing tests for PageCache**

```typescript
// packages/pages-data/src/dataset/page-cache.test.ts
import { describe, it, expect } from "vitest";
import { PageCache } from "./page-cache.js";
import type { TypedDataSet } from "./types.js";

function mockDataSet(label: string): TypedDataSet {
  return { columns: [], rows: [] } as unknown as TypedDataSet;
}

describe("PageCache", () => {
  it("returns undefined on cache miss", () => {
    const cache = new PageCache(5);
    expect(cache.get({ offset: 0, limit: 25 })).toBeUndefined();
  });

  it("returns stored page on cache hit", () => {
    const cache = new PageCache(5);
    const ds = mockDataSet("page0");
    cache.store({ offset: 0, limit: 25 }, { dataset: ds, totalRows: 100 });
    const result = cache.get({ offset: 0, limit: 25 });
    expect(result).toBeDefined();
    expect(result!.totalRows).toBe(100);
  });

  it("distinguishes keys by sort and filter", () => {
    const cache = new PageCache(5);
    const ds1 = mockDataSet("unsorted");
    const ds2 = mockDataSet("sorted");
    cache.store({ offset: 0, limit: 25 }, { dataset: ds1, totalRows: 100 });
    cache.store({ offset: 0, limit: 25, sort: "name", order: "asc" }, { dataset: ds2, totalRows: 100 });
    expect(cache.get({ offset: 0, limit: 25 })!.dataset).toBe(ds1);
    expect(cache.get({ offset: 0, limit: 25, sort: "name", order: "asc" })!.dataset).toBe(ds2);
  });

  it("evicts LRU entry when capacity exceeded", () => {
    const cache = new PageCache(2);
    const ds0 = mockDataSet("p0");
    const ds1 = mockDataSet("p1");
    const ds2 = mockDataSet("p2");
    cache.store({ offset: 0, limit: 25 }, { dataset: ds0, totalRows: 100 });
    cache.store({ offset: 25, limit: 25 }, { dataset: ds1, totalRows: 100 });
    // Access page 0 to make page 1 the LRU
    cache.get({ offset: 0, limit: 25 });
    // Add page 2 — should evict page 1 (LRU)
    cache.store({ offset: 50, limit: 25 }, { dataset: ds2, totalRows: 100 });
    expect(cache.get({ offset: 0, limit: 25 })).toBeDefined();
    expect(cache.get({ offset: 25, limit: 25 })).toBeUndefined();
    expect(cache.get({ offset: 50, limit: 25 })).toBeDefined();
  });

  it("clear() removes all entries", () => {
    const cache = new PageCache(5);
    cache.store({ offset: 0, limit: 25 }, { dataset: mockDataSet("p0"), totalRows: 50 });
    cache.store({ offset: 25, limit: 25 }, { dataset: mockDataSet("p1"), totalRows: 50 });
    cache.clear();
    expect(cache.get({ offset: 0, limit: 25 })).toBeUndefined();
    expect(cache.size).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-data run test -- --testPathPattern page-cache`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PageCache**

```typescript
// packages/pages-data/src/dataset/page-cache.ts
import type { TypedDataSet } from "./types.js";

export interface PageCacheKey {
  readonly offset: number;
  readonly limit: number;
  readonly sort?: string;
  readonly order?: string;
  readonly filter?: string;
}

export interface CachedPage {
  readonly dataset: TypedDataSet;
  readonly totalRows: number;
}

function serializeKey(key: PageCacheKey): string {
  return `${key.offset}:${key.limit}:${key.sort ?? ""}:${key.order ?? ""}:${key.filter ?? ""}`;
}

export class PageCache {
  private readonly _maxSize: number;
  private readonly _entries = new Map<string, CachedPage>();
  private readonly _accessOrder: string[] = [];

  constructor(maxSize: number) {
    this._maxSize = maxSize;
  }

  get(key: PageCacheKey): CachedPage | undefined {
    const k = serializeKey(key);
    const entry = this._entries.get(k);
    if (entry) {
      const idx = this._accessOrder.indexOf(k);
      if (idx !== -1) this._accessOrder.splice(idx, 1);
      this._accessOrder.push(k);
    }
    return entry;
  }

  store(key: PageCacheKey, page: CachedPage): void {
    const k = serializeKey(key);
    if (this._entries.has(k)) {
      const idx = this._accessOrder.indexOf(k);
      if (idx !== -1) this._accessOrder.splice(idx, 1);
    } else if (this._entries.size >= this._maxSize) {
      const evict = this._accessOrder.shift();
      if (evict) this._entries.delete(evict);
    }
    this._entries.set(k, page);
    this._accessOrder.push(k);
  }

  clear(): void {
    this._entries.clear();
    this._accessOrder.length = 0;
  }

  get size(): number {
    return this._entries.size;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-data run test -- --testPathPattern page-cache`
Expected: PASS (all 5 tests)

- [ ] **Step 5: Export from package index**

In `packages/pages-data/src/index.ts`, add:
```typescript
export { PageCache } from "./dataset/page-cache.js";
export type { PageCacheKey, CachedPage } from "./dataset/page-cache.js";
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/dataset/page-cache.ts packages/pages-data/src/dataset/page-cache.test.ts packages/pages-data/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#12): PageCache — LRU page cache for server-side pagination"
```

---

### Task 3: DSL builder — serverPaginated()

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/dsl/builders.test.ts`
- Modify: `packages/pages-ui/src/dsl/index.ts`

**Interfaces:**
- Consumes: `ServerPaginationConfig` from `@casehubio/pages-data`
- Produces: `serverPaginated(options?): ServerPaginationConfig` builder function, `ServerPaginationOptions` interface

- [ ] **Step 1: Write failing test**

In `packages/pages-ui/src/dsl/builders.test.ts`, add:

```typescript
describe("serverPaginated()", () => {
  it("returns config with defaults", () => {
    const config = serverPaginated();
    expect(config.offsetParam).toBe("offset");
    expect(config.limitParam).toBe("limit");
    expect(config.defaultPageSize).toBe(25);
    expect(config.maxCachedPages).toBe(5);
    expect(config.sortParam).toBeUndefined();
  });

  it("accepts custom param names", () => {
    const config = serverPaginated({
      offsetParam: "skip",
      limitParam: "take",
      sortParam: "sortBy",
      orderParam: "dir",
      defaultPageSize: 50,
    });
    expect(config.offsetParam).toBe("skip");
    expect(config.limitParam).toBe("take");
    expect(config.sortParam).toBe("sortBy");
    expect(config.defaultPageSize).toBe(50);
  });
});
```

Import `serverPaginated` in the test file's import block.

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui run test -- --testPathPattern builders`
Expected: FAIL — `serverPaginated` is not exported

- [ ] **Step 3: Implement serverPaginated builder**

In `packages/pages-ui/src/dsl/builders.ts`, add after the `dockWorkbench` function:

```typescript
import type { ServerPaginationConfig } from "@casehubio/pages-data";

export interface ServerPaginationOptions {
  readonly offsetParam?: string;
  readonly limitParam?: string;
  readonly sortParam?: string;
  readonly orderParam?: string;
  readonly filterParam?: string;
  readonly defaultPageSize?: number;
  readonly maxCachedPages?: number;
}

export function serverPaginated(options?: ServerPaginationOptions): ServerPaginationConfig {
  return {
    offsetParam: options?.offsetParam ?? "offset",
    limitParam: options?.limitParam ?? "limit",
    sortParam: options?.sortParam,
    orderParam: options?.orderParam,
    filterParam: options?.filterParam,
    defaultPageSize: options?.defaultPageSize ?? 25,
    maxCachedPages: options?.maxCachedPages ?? 5,
  };
}
```

- [ ] **Step 4: Export from index**

In `packages/pages-ui/src/dsl/index.ts`, add `serverPaginated` and `type ServerPaginationOptions` to the exports from `./builders.js`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --testPathPattern builders`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts packages/pages-ui/src/dsl/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#12): serverPaginated() DSL builder with configurable param names"
```

---

### Task 4: Pipeline integration — server-paged fetch and cache

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts`
- Create: `packages/pages-runtime/src/server-pagination.ts` (extracted helper)
- Create: `packages/pages-runtime/src/server-pagination.test.ts`
- Modify: `packages/pages-runtime/src/site.test.ts`

**Interfaces:**
- Consumes: `ServerPaginationConfig` from `@casehubio/pages-data`, `PageCache` from `@casehubio/pages-data`, `resolveTemplate` from `./context-wiring.js`, `extractDataSet` from `@casehubio/pages-data`
- Produces: `ServerPaginationManager` — manages per-dataset page caches, URL construction, and fetch execution. Called from `pushData()` and `handleDefRequest()`.

- [ ] **Step 1: Write failing tests for ServerPaginationManager**

```typescript
// packages/pages-runtime/src/server-pagination.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { ServerPaginationManager } from "./server-pagination.js";
import type { ServerPaginationConfig } from "@casehubio/pages-data";
import { dataSetId } from "@casehubio/pages-data";

const dsId = dataSetId("test-ds");
const config: ServerPaginationConfig = {
  offsetParam: "offset",
  limitParam: "limit",
  sortParam: "sort",
  orderParam: "order",
  defaultPageSize: 25,
  maxCachedPages: 3,
};

describe("ServerPaginationManager", () => {
  let fetchFn: ReturnType<typeof vi.fn>;
  let manager: ServerPaginationManager;

  beforeEach(() => {
    fetchFn = vi.fn().mockResolvedValue({
      ok: true,
      headers: { get: () => "application/json" },
      json: () => Promise.resolve({
        data: [{ name: "Alice" }, { name: "Bob" }],
        total: 100,
      }),
    });
    manager = new ServerPaginationManager(fetchFn);
  });

  it("registers a dataset config", () => {
    manager.register(dsId, config, "https://api.example.com/items?offset={offset}&limit={limit}", "total");
    expect(manager.has(dsId)).toBe(true);
  });

  it("fetches page on cache miss", async () => {
    manager.register(dsId, config, "https://api.example.com/items?offset={offset}&limit={limit}", "total");
    await manager.fetchPage(dsId, 0, 25);
    expect(fetchFn).toHaveBeenCalledTimes(1);
    expect(fetchFn.mock.calls[0][0]).toBe("https://api.example.com/items?offset=0&limit=25");
  });

  it("returns cached page on hit", async () => {
    manager.register(dsId, config, "https://api.example.com/items?offset={offset}&limit={limit}", "total");
    await manager.fetchPage(dsId, 0, 25);
    fetchFn.mockClear();
    const result = await manager.fetchPage(dsId, 0, 25);
    expect(fetchFn).not.toHaveBeenCalled();
    expect(result).toBeDefined();
  });

  it("clears cache on sort change", async () => {
    manager.register(dsId, config, "https://api.example.com/items?offset={offset}&limit={limit}&sort={sort}&order={order}", "total");
    await manager.fetchPage(dsId, 0, 25);
    manager.updateSort(dsId, "name", "asc");
    fetchFn.mockClear();
    await manager.fetchPage(dsId, 0, 25, { sort: "name", order: "asc" });
    expect(fetchFn).toHaveBeenCalledTimes(1);
  });

  it("builds URL with sort and filter params", async () => {
    manager.register(dsId, config, "https://api.example.com/items?offset={offset}&limit={limit}&sort={sort}&order={order}", "total");
    await manager.fetchPage(dsId, 0, 25, { sort: "name", order: "desc" });
    expect(fetchFn.mock.calls[0][0]).toBe("https://api.example.com/items?offset=0&limit=25&sort=name&order=desc");
  });

  it("strips empty query params from URL", async () => {
    manager.register(dsId, config, "https://api.example.com/items?offset={offset}&limit={limit}&sort={sort}&order={order}", "total");
    await manager.fetchPage(dsId, 0, 25);
    const url = fetchFn.mock.calls[0][0] as string;
    expect(url).not.toContain("sort=&");
    expect(url).not.toContain("order=");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --testPathPattern server-pagination`
Expected: FAIL — module not found

- [ ] **Step 3: Implement ServerPaginationManager**

```typescript
// packages/pages-runtime/src/server-pagination.ts
import type { DataSetId, TypedDataSet, ServerPaginationConfig } from "@casehubio/pages-data";
import { PageCache } from "@casehubio/pages-data";
import type { CachedPage, PageCacheKey } from "@casehubio/pages-data";
import { extractDataSet } from "@casehubio/pages-data";

interface DatasetRegistration {
  config: ServerPaginationConfig;
  urlTemplate: string;
  totalPath: string | undefined;
  cache: PageCache;
}

export interface FetchPageOptions {
  sort?: string;
  order?: string;
  filter?: string;
}

export class ServerPaginationManager {
  private readonly _datasets = new Map<DataSetId, DatasetRegistration>();
  private readonly _fetchFn: typeof globalThis.fetch;

  constructor(fetchFn?: typeof globalThis.fetch) {
    this._fetchFn = fetchFn ?? globalThis.fetch.bind(globalThis);
  }

  register(
    id: DataSetId,
    config: ServerPaginationConfig,
    urlTemplate: string,
    totalPath: string | undefined,
  ): void {
    this._datasets.set(id, {
      config,
      urlTemplate,
      totalPath,
      cache: new PageCache(config.maxCachedPages ?? 5),
    });
  }

  has(id: DataSetId): boolean {
    return this._datasets.has(id);
  }

  updateSort(id: DataSetId, _sort: string, _order: string): void {
    const reg = this._datasets.get(id);
    if (reg) reg.cache.clear();
  }

  updateFilter(id: DataSetId, _filter: string): void {
    const reg = this._datasets.get(id);
    if (reg) reg.cache.clear();
  }

  async fetchPage(
    id: DataSetId,
    offset: number,
    limit: number,
    options?: FetchPageOptions,
  ): Promise<CachedPage | undefined> {
    const reg = this._datasets.get(id);
    if (!reg) return undefined;

    const key: PageCacheKey = {
      offset,
      limit,
      sort: options?.sort,
      order: options?.order,
      filter: options?.filter,
    };

    const cached = reg.cache.get(key);
    if (cached) return cached;

    const url = this._buildUrl(reg, offset, limit, options);
    const response = await this._fetchFn(url);
    const contentType = response.headers?.get("content-type") ?? undefined;
    const data = contentType?.includes("application/json")
      ? await response.json()
      : await response.text();

    let totalRows = 0;
    if (reg.totalPath) {
      let current: unknown = data;
      for (const segment of reg.totalPath.split(".")) {
        if (current === null || current === undefined || typeof current !== "object") break;
        current = (current as Record<string, unknown>)[segment];
      }
      if (typeof current === "number" && Number.isFinite(current)) totalRows = current;
    }

    const { dataset } = await extractDataSet(
      { data, ...(contentType ? { contentType } : {}) },
      { url },
      undefined,
    );

    const page: CachedPage = { dataset, totalRows };
    reg.cache.store(key, page);
    return page;
  }

  clear(id: DataSetId): void {
    this._datasets.get(id)?.cache.clear();
  }

  private _buildUrl(
    reg: DatasetRegistration,
    offset: number,
    limit: number,
    options?: FetchPageOptions,
  ): string {
    const config = reg.config;
    let url = reg.urlTemplate;

    url = url.replace(`{${config.offsetParam}}`, String(offset));
    url = url.replace(`{${config.limitParam}}`, String(limit));

    if (config.sortParam) {
      url = url.replace(`{${config.sortParam}}`, options?.sort ?? "");
    }
    if (config.orderParam) {
      url = url.replace(`{${config.orderParam}}`, options?.order ?? "");
    }
    if (config.filterParam) {
      url = url.replace(`{${config.filterParam}}`, options?.filter ?? "");
    }

    // Strip empty query params
    const parsed = new URL(url);
    const toDelete: string[] = [];
    parsed.searchParams.forEach((v, k) => {
      if (v === "") toDelete.push(k);
    });
    for (const k of toDelete) parsed.searchParams.delete(k);

    return parsed.toString();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --testPathPattern server-pagination`
Expected: PASS (all 6 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/server-pagination.ts packages/pages-runtime/src/server-pagination.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#12): ServerPaginationManager — URL construction, fetch, LRU caching"
```

---

### Task 5: Pipeline wiring — connect ServerPaginationManager to pushData and event handlers

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts`
- Modify: `packages/pages-runtime/src/site.ts`
- Modify: `packages/pages-runtime/src/site.test.ts`

**Interfaces:**
- Consumes: `ServerPaginationManager` from `./server-pagination.js`, `ServerPaginationConfig` from `@casehubio/pages-data`
- Produces: integrated pipeline that routes server-paged datasets through `ServerPaginationManager` instead of client-side slicing

- [ ] **Step 1: Write failing integration test**

In `packages/pages-runtime/src/site.test.ts`, add a test for the full round-trip:

```typescript
describe("server-side pagination (#12)", () => {
  it("fetches only page 0 on initial load for serverPagination datasets", async () => {
    // Setup: create a site with a dataset that has serverPagination config
    // Verify: fetch is called with offset=0&limit=25
    // Verify: table receives dataset + totalRows
  });
});
```

(Full test body depends on existing test helpers in site.test.ts — adapt to the existing fixture pattern.)

- [ ] **Step 2: Wire ServerPaginationManager into data-pipeline.ts**

In `createDataPipeline()`, add:

```typescript
import { ServerPaginationManager } from "./server-pagination.js";

// Inside the factory function, after existing Maps:
const serverPagination = new ServerPaginationManager();
```

In `handleDefRequest()`, add a branch before the existing routing:

```typescript
if (def.serverPagination && def.url) {
  if (!serverPagination.has(lookup.dataSetId)) {
    serverPagination.register(
      lookup.dataSetId,
      def.serverPagination,
      def.url,
      def.totalPath,
    );
  }
  // Fetch page 0
  target.loading = true;
  serverPagination.fetchPage(lookup.dataSetId, 0, def.serverPagination.defaultPageSize)
    .then((page) => {
      if (!page) return;
      manager.apply(lookup.dataSetId, { type: "snapshot", dataset: page.dataset, totalRows: page.totalRows });
      pushData(target, lookup, _entry.pagePath, filterGroup, componentId);
    })
    .catch((err: unknown) => {
      target.error = err instanceof Error ? err.message : String(err);
    });
  return;
}
```

- [ ] **Step 3: Modify pushData for server-paged datasets**

In `pushData()`, add a branch at the top of the non-textFilter path:

```typescript
// Inside the else branch (no text filter), before existing pagination:
if (serverPagination.has(lookup.dataSetId)) {
  const compState = getComponentState(componentViewState, componentId);
  const requestedPage = compState?.page ?? 0;
  const entry = registry.get(componentId);
  const pageSize = (entry?.component.props as { pageSize?: number } | undefined)?.pageSize
    ?? serverPagination._datasets.get(lookup.dataSetId)?.config.defaultPageSize ?? 25;
  const offset = requestedPage * pageSize;
  const sortState = compState?.sort;

  const cached = serverPagination.fetchPage(lookup.dataSetId, offset, pageSize, {
    sort: sortState ? String(sortState.columnId) : undefined,
    order: sortState?.order,
  });

  void cached.then((page) => {
    if (!page) return;
    target.activePage = requestedPage;
    target.totalRows = page.totalRows;
    target.dataSet = page.dataset;
    target.loading = false;
  });
  return;
}
```

- [ ] **Step 4: Wire sort/filter invalidation in site.ts event handlers**

In the `pages-sort` handler, after `updateSort()`:

```typescript
// If this is a server-paged dataset, clear the cache and re-fetch
if (serverPagination.has(entry.originalLookup.dataSetId)) {
  serverPagination.updateSort(entry.originalLookup.dataSetId, String(columnId), order);
  updatePage(componentViewState, componentId, 0);
}
```

Expose `serverPagination` from the pipeline interface so `site.ts` can access it, or pass it via a method on the pipeline.

- [ ] **Step 5: Add corrupted view protection**

In `pushData()`, when `serverPagination.has(lookup.dataSetId)` is true:

```typescript
// Strip client-side sort/filter ops with warning
const clientOps = lookup.operations.filter(op => op.type === "sort" || op.type === "filter");
if (clientOps.length > 0) {
  console.warn(`[DataPipeline] Dataset "${String(lookup.dataSetId)}" uses server pagination — client-side ${clientOps.map(o => o.type).join(", ")} ignored`);
}
```

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 7: Run full typecheck**

Run: `yarn typecheck`
Expected: clean

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/data-pipeline.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/site.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#12): wire server-side pagination into pipeline — fetch, cache, sort/filter invalidation

Refs #12"
```

---

### Task 6: Full build verification and cleanup

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: all prior tasks
- Produces: verified clean build

- [ ] **Step 1: Run full test suite**

Run: `export GH_PACKAGES_TOKEN=<token> && yarn build && yarn typecheck`
Expected: clean build, clean typecheck

- [ ] **Step 2: Run all package tests**

Run: `yarn workspace @casehubio/pages-data run test && yarn workspace @casehubio/pages-ui run test && yarn workspace @casehubio/pages-runtime run test`
Expected: all pass

- [ ] **Step 3: Final commit if any cleanup needed**

If lint or typecheck surfaced issues, fix and commit.
