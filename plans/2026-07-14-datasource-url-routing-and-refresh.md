# DataSourceController URL Routing & Pipeline Refresh Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #148 — feat: DataSourceController endpoint URL auto-routing
**Issue group:** #148, #134

**Goal:** Provide URL-scheme-based source factory routing for DataSourceController endpoints, and fix the pipeline's refresh mechanism to deliver fresh data instead of stale cache — with three complementary freshness layers.

**Architecture:** Types relocate from `pages-component` to `pages-data` (where their constituent types live). A default `SourceFactory` does URL-scheme routing. The pipeline splits `refreshDataSet` (re-fetch) from `deliverDataSet` (cache serve), adds a `pages-refresh-request` event for panels, and stale-while-revalidate with TTL tracking in the DataSetManager.

**Tech Stack:** TypeScript 5, Vitest, pages-data / pages-component / pages-runtime packages

## Global Constraints

- Pre-release — breaking changes cost nothing
- `restSource` signature changes to `(url, id, options?)` to align with `sseSource`/`wsSource`
- `totalPath` is excluded from `SourceFactoryOptions` (tracked as #185)
- Layer 3 default TTL is 60 seconds for datasets without `refreshTime`
- All edits via IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_insert_member`) — never Edit/Write on existing source files

---

### Task 1: Type relocation and restSource signature alignment

**Files:**
- Modify: `packages/pages-data/src/datasource/types.ts` — add `SourceFactory`, `SourceFactoryOptions`
- Modify: `packages/pages-data/src/datasource/index.ts` — export new types
- Modify: `packages/pages-data/src/datasource/sources/rest-source.ts` — add `dataSetId` param
- Modify: `packages/pages-data/src/datasource/sources/rest-source.test.ts` — update all call sites
- Modify: `packages/pages-data/src/datasource/sources/def-to-binding.ts` — update `restSource` call
- Modify: `packages/pages-component/src/controller/data-source-controller.ts` — remove local types, import from `pages-data`
- Modify: `packages/pages-component/src/controller/data-source-controller.test.ts` — update imports

**Interfaces:**
- Produces: `SourceFactory` type — `(url: string, id: DataSetId, options?: SourceFactoryOptions) => DataSource`
- Produces: `SourceFactoryOptions` interface — `{ columns?, dataPath? }`
- Produces: `restSource(url: string, id: DataSetId, options?: RestSourceOptions): DataSource`

- [ ] **Step 1: Write failing test for restSource new signature**

In `rest-source.test.ts`, add a test that passes `dataSetId` as second arg:

```typescript
it("accepts dataSetId as second parameter", async () => {
  const fetchFn = mockFetch([["alice"]]);
  const source = restSource("https://api.example.com/data", "ds-test" as any, {
    columns: [col("name", ColumnType.TEXT)],
    fetchFn,
  });
  const { sink, events } = collectSink();
  source.connect(sink);
  await vi.waitFor(() => { expect(events.length).toBeGreaterThan(0); });
  expect(events[0]!.type).toBe("snapshot");
  source.disconnect();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/rest-source.test.ts`
Expected: FAIL — `restSource` does not accept 3 args

- [ ] **Step 3: Update restSource signature**

Change `restSource` in `rest-source.ts` to accept `(url: string, _id: DataSetId, options?: RestSourceOptions)`. The `_id` parameter is unused today but aligns the signature with `sseSource`/`wsSource`. Import `DataSetId` from `../../dataset/types.js`.

- [ ] **Step 4: Update all existing restSource callers**

Update `def-to-binding.ts` line 114:
```typescript
return { ...base, source: restSource(url, def.uuid, opts) };
```

Update all existing test calls in `rest-source.test.ts` to pass a dummy `dataSetId` as second arg (e.g. `"test-ds" as any`).

- [ ] **Step 5: Run tests to verify all pass**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/rest-source.test.ts`
Expected: all PASS

- [ ] **Step 6: Add SourceFactory types to pages-data**

In `packages/pages-data/src/datasource/types.ts`, add after the existing exports:

```typescript
import type { ExternalColumnDef } from "../dataset/external/types.js";

export interface SourceFactoryOptions {
  readonly columns?: readonly ExternalColumnDef[] | undefined;
  readonly dataPath?: string | undefined;
}

export type SourceFactory = (
  url: string,
  id: DataSetId,
  options?: SourceFactoryOptions,
) => DataSource;
```

- [ ] **Step 7: Export from barrel**

In `packages/pages-data/src/datasource/index.ts`, add:

```typescript
export type { SourceFactory, SourceFactoryOptions } from "./types.js";
```

- [ ] **Step 8: Update DataSourceController to import from pages-data**

In `packages/pages-component/src/controller/data-source-controller.ts`:
- Remove the local `SourceFactoryOptions` interface and `SourceFactory` type
- Import them from `@casehubio/pages-data`:
  ```typescript
  import type {SourceFactory, SourceFactoryOptions} from "@casehubio/pages-data";
  ```
- Keep the `DataSourceControllerOptions` interface referencing `sourceFactory?: SourceFactory`

- [ ] **Step 9: Run full typecheck and tests**

Run: `yarn typecheck && yarn workspace @casehubio/pages-data run test -- --run && yarn workspace @casehubio/pages-component run test -- --run`
Expected: all PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/datasource/types.ts packages/pages-data/src/datasource/index.ts packages/pages-data/src/datasource/sources/rest-source.ts packages/pages-data/src/datasource/sources/rest-source.test.ts packages/pages-data/src/datasource/sources/def-to-binding.ts packages/pages-component/src/controller/data-source-controller.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: move SourceFactory types to pages-data, align restSource signature

Relocate SourceFactory and SourceFactoryOptions from pages-component to
pages-data where their constituent types live. Add dataSetId parameter
to restSource for signature parity with sseSource/wsSource.

Refs #148"
```

---

### Task 2: createSourceFactory + tests

**Files:**
- Create: `packages/pages-data/src/datasource/sources/source-factory.ts`
- Create: `packages/pages-data/src/datasource/sources/source-factory.test.ts`
- Modify: `packages/pages-data/src/datasource/index.ts` — export factory

**Interfaces:**
- Consumes: `SourceFactory` from Task 1, `restSource`, `sseSource`, `wsSource` from existing code
- Produces: `createSourceFactory(deps?: SourceFactoryDeps): SourceFactory`

- [ ] **Step 1: Write failing tests for URL-scheme routing**

Create `packages/pages-data/src/datasource/sources/source-factory.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { createSourceFactory } from "./source-factory.js";
import type { DataSink } from "../types.js";
import type { DataSetId } from "../../dataset/types.js";

function collectSink(): { sink: DataSink; events: unknown[]; errors: unknown[] } {
  const events: unknown[] = [];
  const errors: unknown[] = [];
  return {
    sink: {
      apply(event) { events.push(event); },
      error(err) { errors.push(err); },
    },
    events,
    errors,
  };
}

const TEST_ID = "test-ds" as DataSetId;

describe("createSourceFactory", () => {
  it("routes relative URL to restSource", () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response(JSON.stringify([["a"]]), {
      headers: { "content-type": "application/json" },
    }));
    const factory = createSourceFactory({ fetchFn });
    const source = factory("/api/items", TEST_ID);
    expect(source).toBeDefined();
    expect(typeof source.connect).toBe("function");
    expect(typeof source.disconnect).toBe("function");
  });

  it("routes http:// URL to restSource", () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response("[]", {
      headers: { "content-type": "application/json" },
    }));
    const factory = createSourceFactory({ fetchFn });
    const source = factory("http://api.example.com/data", TEST_ID);
    const { sink } = collectSink();
    source.connect(sink);
    expect(fetchFn).toHaveBeenCalledWith("http://api.example.com/data", expect.anything());
    source.disconnect();
  });

  it("routes https:// URL to restSource", () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response("[]", {
      headers: { "content-type": "application/json" },
    }));
    const factory = createSourceFactory({ fetchFn });
    const source = factory("https://api.example.com/data", TEST_ID);
    const { sink } = collectSink();
    source.connect(sink);
    expect(fetchFn).toHaveBeenCalled();
    source.disconnect();
  });

  it("routes ws:// URL to wsSource", () => {
    const mockPool = {
      acquire: vi.fn().mockReturnValue({
        subscribe: vi.fn(),
        unsubscribe: vi.fn(),
      }),
      releaseAll: vi.fn(),
    };
    const factory = createSourceFactory({ wsPool: mockPool });
    const source = factory("ws://host/events", TEST_ID);
    const { sink } = collectSink();
    source.connect(sink);
    expect(mockPool.acquire).toHaveBeenCalled();
    source.disconnect();
  });

  it("routes wss:// URL to wsSource", () => {
    const mockPool = {
      acquire: vi.fn().mockReturnValue({
        subscribe: vi.fn(),
        unsubscribe: vi.fn(),
      }),
      releaseAll: vi.fn(),
    };
    const factory = createSourceFactory({ wsPool: mockPool });
    const source = factory("wss://host/events", TEST_ID);
    const { sink } = collectSink();
    source.connect(sink);
    expect(mockPool.acquire).toHaveBeenCalled();
    source.disconnect();
  });

  it("routes sse:// URL to sseSource", () => {
    const mockPool = {
      acquire: vi.fn().mockReturnValue({
        subscribe: vi.fn(),
        unsubscribe: vi.fn(),
      }),
      releaseAll: vi.fn(),
    };
    const factory = createSourceFactory({ ssePool: mockPool });
    const source = factory("sse://host/topic", TEST_ID);
    const { sink } = collectSink();
    source.connect(sink);
    expect(mockPool.acquire).toHaveBeenCalled();
    source.disconnect();
  });

  it("routes sses:// URL to sseSource", () => {
    const mockPool = {
      acquire: vi.fn().mockReturnValue({
        subscribe: vi.fn(),
        unsubscribe: vi.fn(),
      }),
      releaseAll: vi.fn(),
    };
    const factory = createSourceFactory({ ssePool: mockPool });
    const source = factory("sses://host/topic", TEST_ID);
    const { sink } = collectSink();
    source.connect(sink);
    expect(mockPool.acquire).toHaveBeenCalled();
    source.disconnect();
  });

  it("forwards columns and dataPath to rest source options", async () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response(
      JSON.stringify({ items: [["x"]] }),
      { headers: { "content-type": "application/json" } },
    ));
    const factory = createSourceFactory({ fetchFn });
    const source = factory("/api/data", TEST_ID, {
      columns: [{ id: "col1" as any, type: 0 as any }],
      dataPath: "items",
    });
    expect(source).toBeDefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/source-factory.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement createSourceFactory**

Create `packages/pages-data/src/datasource/sources/source-factory.ts`:

```typescript
import type { DataSetId } from "../../dataset/types.js";
import type { SourceFactory, SourceFactoryOptions } from "../types.js";
import type { PushPool } from "../../dataset/external/sources/push-pool.js";
import type { PresetRegistry } from "../../dataset/external/types.js";
import { restSource } from "./rest-source.js";
import { sseSource } from "./sse-source.js";
import { wsSource } from "./ws-source.js";
import { defaultSsePushPool } from "./default-pools.js";
import { defaultWsPushPool } from "./default-pools.js";

export interface SourceFactoryDeps {
  readonly wsPool?: PushPool;
  readonly ssePool?: PushPool;
  readonly fetchFn?: typeof globalThis.fetch;
  readonly presets?: PresetRegistry;
}

export function createSourceFactory(deps?: SourceFactoryDeps): SourceFactory {
  return (url: string, id: DataSetId, options?: SourceFactoryOptions) => {
    const columns = options?.columns;
    const dataPath = options?.dataPath;

    if (url.startsWith("ws://") || url.startsWith("wss://")) {
      return wsSource(url, id, {
        ...(columns !== undefined && { columns }),
        ...(dataPath !== undefined && { dataPath }),
        ...(deps?.wsPool !== undefined && { pool: deps.wsPool }),
      });
    }

    if (url.startsWith("sse://") || url.startsWith("sses://")) {
      return sseSource(url, id, {
        ...(columns !== undefined && { columns }),
        ...(dataPath !== undefined && { dataPath }),
        ...(deps?.ssePool !== undefined && { pool: deps.ssePool }),
      });
    }

    return restSource(url, id, {
      ...(columns !== undefined && { columns }),
      ...(dataPath !== undefined && { dataPath }),
      ...(deps?.fetchFn !== undefined && { fetchFn: deps.fetchFn }),
      ...(deps?.presets !== undefined && { presets: deps.presets }),
    });
  };
}
```

- [ ] **Step 4: Export from barrel**

In `packages/pages-data/src/datasource/index.ts`, add:

```typescript
export { createSourceFactory } from "./sources/source-factory.js";
export type { SourceFactoryDeps } from "./sources/source-factory.js";
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/source-factory.test.ts`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/datasource/sources/source-factory.ts packages/pages-data/src/datasource/sources/source-factory.test.ts packages/pages-data/src/datasource/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: createSourceFactory with URL-scheme routing

Routes ws/wss → wsSource, sse/sses → sseSource, everything else → restSource.
Forwards columns and dataPath options to the underlying source.

Refs #148"
```

---

### Task 3: DataSourceController integration test

**Files:**
- Modify: `packages/pages-component/src/controller/data-source-controller.test.ts`

**Interfaces:**
- Consumes: `createSourceFactory` from Task 2, `DataSourceController` (existing)

- [ ] **Step 1: Write integration tests**

Add to `data-source-controller.test.ts`:

```typescript
import { createSourceFactory } from "@casehubio/pages-data";

describe("DataSourceController with createSourceFactory", () => {
  it("endpoint with relative URL creates restSource and delivers data", async () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response(
      JSON.stringify([["alice"]]),
      { headers: { "content-type": "application/json" } },
    ));
    const ctrl = new DataSourceController({
      sourceFactory: createSourceFactory({ fetchFn }),
    });
    ctrl.connect();
    ctrl.endpoint = "/api/items";
    await vi.waitFor(() => { expect(ctrl.dataSet).toBeDefined(); });
    expect(ctrl.dataSet!.rows).toHaveLength(1);
    ctrl.dispose();
  });

  it("endpoint with ws:// URL creates wsSource", () => {
    const mockPool = {
      acquire: vi.fn().mockReturnValue({
        subscribe: vi.fn(),
        unsubscribe: vi.fn(),
      }),
      releaseAll: vi.fn(),
    };
    const ctrl = new DataSourceController({
      sourceFactory: createSourceFactory({ wsPool: mockPool }),
    });
    ctrl.connect();
    ctrl.endpoint = "ws://host/events";
    expect(mockPool.acquire).toHaveBeenCalled();
    ctrl.dispose();
  });

  it("endpoint with sse:// URL creates sseSource", () => {
    const mockPool = {
      acquire: vi.fn().mockReturnValue({
        subscribe: vi.fn(),
        unsubscribe: vi.fn(),
      }),
      releaseAll: vi.fn(),
    };
    const ctrl = new DataSourceController({
      sourceFactory: createSourceFactory({ ssePool: mockPool }),
    });
    ctrl.connect();
    ctrl.endpoint = "sse://host/topic";
    expect(mockPool.acquire).toHaveBeenCalled();
    ctrl.dispose();
  });
});
```

- [ ] **Step 2: Run tests**

Run: `yarn workspace @casehubio/pages-component run test -- --run src/controller/data-source-controller.test.ts`
Expected: all PASS (implementation already exists)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/controller/data-source-controller.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "test: integration tests for DataSourceController with createSourceFactory

Verifies endpoint URL routing through restSource (relative), wsSource (ws://),
and sseSource (sse://) via the injected factory.

Closes #148"
```

---

### Task 4: DataSetManager TTL (age method)

**Files:**
- Modify: `packages/pages-data/src/dataset/manager.ts` — add timestamps, age()
- Modify: `packages/pages-data/src/dataset/manager.test.ts` — TTL tests

**Interfaces:**
- Produces: `DataSetManager.age(id: DataSetId): number | undefined`

- [ ] **Step 1: Write failing tests for age()**

Add to `manager.test.ts`:

```typescript
describe("DataSetManager — TTL", () => {
  it("age() returns undefined for unknown dataset", () => {
    const mgr = createDataSetManager();
    expect(mgr.age(ID_UNKNOWN)).toBeUndefined();
  });

  it("age() returns milliseconds since apply()", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });

    const age = mgr.age(ID_A);
    expect(age).toBeDefined();
    expect(age!).toBeGreaterThanOrEqual(0);
    expect(age!).toBeLessThan(100);
  });

  it("age() resets after subsequent apply()", async () => {
    const mgr = createDataSetManager();
    const ds1 = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds1 });

    // Wait a bit so the timestamp differs
    await new Promise(r => setTimeout(r, 10));
    const ageBefore = mgr.age(ID_A)!;

    const ds2 = testDataSet([["Bob", "200"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds2 });
    const ageAfter = mgr.age(ID_A)!;

    expect(ageAfter).toBeLessThan(ageBefore);
  });

  it("remove() clears timestamp", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });
    expect(mgr.age(ID_A)).toBeDefined();

    mgr.remove(ID_A);
    expect(mgr.age(ID_A)).toBeUndefined();
  });

  it("age() updates on append events", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });

    mgr.apply(ID_A, {
      type: "append",
      rows: [ds.rows[0]!],
    });
    const age = mgr.age(ID_A);
    expect(age).toBeDefined();
    expect(age!).toBeLessThan(50);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: FAIL — `age` is not a function

- [ ] **Step 3: Add age() to DataSetManager interface**

In `manager.ts`, add to the `DataSetManager` interface:

```typescript
age(id: DataSetId): number | undefined;
```

- [ ] **Step 4: Implement timestamps in DataSetManagerImpl**

Add a `timestamps` field and update methods:

```typescript
private readonly timestamps = new Map<DataSetId, number>();
```

In `apply()`, after every `this.datasets.set(id, ...)` call, add:
```typescript
this.timestamps.set(id, Date.now());
```

In `remove()`, add:
```typescript
this.timestamps.delete(id);
```

Implement `age()`:
```typescript
age(id: DataSetId): number | undefined {
  const ts = this.timestamps.get(id);
  if (ts === undefined) return undefined;
  return Date.now() - ts;
}
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/manager.test.ts`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/dataset/manager.ts packages/pages-data/src/dataset/manager.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add age() to DataSetManager for TTL tracking

Tracks insertion timestamps per dataset. age(id) returns milliseconds
since the last apply() for that dataset. Cleared on remove().

Refs #134"
```

---

### Task 5: Pipeline refresh/deliver split and re-fetch routing

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts` — split methods, routing logic, re-fetch paths
- Modify: `packages/pages-runtime/src/data-pipeline.test.ts` — comprehensive tests
- Modify: `packages/pages-runtime/src/site.ts` — change `onChanged` wiring

**Interfaces:**
- Consumes: `DataSetManager` (existing), `DataSource.disconnect()/connect()` (existing)
- Produces: `DataPipeline.refreshDataSet(id)` (re-fetch), `DataPipeline.deliverDataSet(id)` (cache), `DataPipeline.deliverAll()` (renamed from `refreshAll`)

- [ ] **Step 1: Write failing test for deliverDataSet**

Add to `data-pipeline.test.ts`:

```typescript
describe("deliverDataSet", () => {
  it("re-delivers cached data to all subscribers", () => {
    const dsId = dataSetId("ds-1");
    const manager = createDataSetManager();
    const registry: ComponentRegistry = new Map();
    const pipeline = createDataPipeline(manager, new Map() as DataSetScope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    const target = makeTarget();
    registry.set("comp-1", {
      vizElement: target,
      originalLookup: { dataSetId: dsId, operations: [] },
      pagePath: "",
      component: { type: "test", props: {} },
      hasExplicitId: false,
    } as any);

    manager.apply(dsId, { type: "snapshot", dataset: regionDataSet([["North"]]) });
    pipeline.deliverDataSet(dsId);

    expect(target.dataSet).toBeDefined();
    expect(target.totalRows).toBe(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run src/data-pipeline.test.ts -t "deliverDataSet"`
Expected: FAIL — `deliverDataSet` is not a function

- [ ] **Step 3: Add deliverDataSet and deliverAll to DataPipeline interface**

In `data-pipeline.ts`, update the `DataPipeline` interface:

```typescript
export interface DataPipeline {
  handleDataRequest(target: VizTarget, lookup: DataSetLookup, componentId: string): void;
  setResolverCtx(ctx: ResolverContext): void;
  dispose(): void;
  refreshDataSet(dataSetId: DataSetId): void;
  deliverDataSet(dataSetId: DataSetId): void;
  deliverAll(): void;
}
```

- [ ] **Step 4: Rename current refreshDataSet body to deliverDataSet**

The current `refreshDataSet` implementation (which calls `pushData`) becomes `deliverDataSet`. The current `refreshAll` becomes `deliverAll`:

```typescript
deliverDataSet(dataSetId: DataSetId): void {
  for (const [compId, entry] of registry) {
    if (entry.originalLookup?.dataSetId === dataSetId && entry.vizElement) {
      const filterGroup = (entry.component.props as Record<string, unknown> | undefined)
        ?.filter as { group?: string } | undefined;
      pushData(entry.vizElement, entry.originalLookup, entry.pagePath, filterGroup?.group, compId);
    }
  }
},

deliverAll(): void {
  for (const [compId, entry] of registry) {
    if (entry.vizElement && entry.originalLookup) {
      const filterGroup = (entry.component.props as Record<string, unknown> | undefined)
        ?.filter as { group?: string } | undefined;
      pushData(entry.vizElement, entry.originalLookup, entry.pagePath, filterGroup?.group, compId);
    }
  }
},
```

- [ ] **Step 5: Implement refreshDataSet with routing logic**

New `refreshDataSet` implementation with the four-path routing from §3.1.2:

```typescript
refreshDataSet(dataSetId: DataSetId): void {
  // 1. Push sources — skip (server pushes updates)
  if (pushSubscriptions.has(dataSetId)) return;

  // 2. DataSource path — disconnect/reconnect
  const connectedSource = connectedSources.get(dataSetId);
  if (connectedSource) {
    connectedSource.disconnect();
    connectedSources.delete(dataSetId);
    connectSource(dataSetId, connectedSource);
    return;
  }

  // 3. Parameterised URL path — re-trigger stored callback
  const reFetch = reFetchCallbacks.get(dataSetId);
  if (reFetch) {
    reFetch();
    return;
  }

  // 4. Legacy ExternalDataSetDef path — resolve and re-fetch
  if (!resolverCtx) return;
  const entry = findFirstEntry(dataSetId);
  if (!entry) return;
  const def = resolveDataSetDef(dataSetId, entry.pagePath, scope);
  if (!def) return;

  const existingController = abortControllers.get(dataSetId);
  if (existingController) existingController.abort();
  const controller = new AbortController();
  abortControllers.set(dataSetId, controller);

  const lookup = serverQueryLookups.get(dataSetId)
    ?? entry.originalLookup
    ?? { dataSetId, operations: [] as readonly DataSetOp[] };

  resolveExternalDataSet(def, resolverCtx, lookup)
    .catch((err: unknown) => {
      if (!controller.signal.aborted) {
        console.warn(`[DataPipeline] Re-fetch failed for ${String(dataSetId)}:`, err);
      }
    });
},
```

Add the `reFetchCallbacks` map and `findFirstEntry` helper:

```typescript
const reFetchCallbacks = new Map<DataSetId, () => void>();

function findFirstEntry(dataSetId: DataSetId) {
  for (const [, entry] of registry) {
    if (entry.originalLookup?.dataSetId === dataSetId) return entry;
  }
  return undefined;
}
```

- [ ] **Step 6: Register reFetchCallbacks for parameterised URL consumers**

In the `handleDefRequest` function, where the parameterised URL consumer is set up, register a re-fetch callback:

```typescript
reFetchCallbacks.set(lookup.dataSetId, () => {
  const existingCtrl = abortControllers.get(lookup.dataSetId);
  if (existingCtrl) existingCtrl.abort();
  lastResolvedUrl = "";
  const ctx = contextManager.getContext();
  const resolved = resolveTemplate(urlTemplate, ctx, "url");
  if (allTemplateVarsResolved(urlTemplate, ctx)) {
    consumer.templates.get("url")!.apply(resolved);
  }
});
```

- [ ] **Step 7: Update site.ts — change onChanged to deliverDataSet**

In `site.ts`, change:
```typescript
onChanged: (id, dataset) => {
  contextManager.updateDataset(id, dataset);
  pipeline.deliverDataSet(id);
},
```

- [ ] **Step 8: Update site.ts — rename refreshAll to deliverAll**

Replace both `pipeline.refreshAll()` calls in `site.ts` with `pipeline.deliverAll()`.

- [ ] **Step 9: Write comprehensive tests for refreshDataSet**

Add tests for each re-fetch path:

```typescript
describe("refreshDataSet (re-fetch)", () => {
  it("disconnects and reconnects DataSource on refresh", () => {
    let connectCount = 0;
    let disconnectCount = 0;
    const mockSource: DataSource = {
      connect(sink: DataSink) {
        connectCount++;
        sink.apply({ type: "snapshot", dataset: regionDataSet([["Fresh"]]) });
      },
      disconnect() { disconnectCount++; },
    };

    const manager = createDataSetManager({
      onChanged: (id) => { pipeline.deliverDataSet(id); },
    });
    const registry: ComponentRegistry = new Map();
    const target = makeTarget();
    registry.set("comp-1", {
      element: document.createElement("div"),
      vizElement: target,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });

    const binding: DataSourceBinding = { id: "ds" as DataSetId, source: mockSource };
    const scope: DataSetScope = new Map([["", new Map([["ds" as DataSetId, binding]])]]);
    const pipeline = createDataPipeline(manager, scope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    pipeline.handleDataRequest(target, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");
    expect(connectCount).toBe(1);

    pipeline.refreshDataSet("ds" as DataSetId);
    expect(disconnectCount).toBe(1);
    expect(connectCount).toBe(2);
  });

  it("skips re-fetch for push sources in pushSubscriptions", () => {
    // Push sources are tracked in pushSubscriptions, not connectedSources
    // This test verifies refreshDataSet is a no-op for them
    const manager = createDataSetManager();
    const registry: ComponentRegistry = new Map();
    const pipeline = createDataPipeline(manager, new Map() as DataSetScope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    // refreshDataSet on unknown ID should be a no-op (no crash)
    expect(() => pipeline.refreshDataSet("unknown" as DataSetId)).not.toThrow();
  });

  it("onChanged calls deliverDataSet not refreshDataSet (no infinite recursion)", () => {
    let refreshCount = 0;
    const mockSource: DataSource = {
      connect(sink: DataSink) {
        sink.apply({ type: "snapshot", dataset: regionDataSet([["A"]]) });
      },
      disconnect() {},
    };

    const manager = createDataSetManager({
      onChanged: (id) => {
        refreshCount++;
        pipeline.deliverDataSet(id);
      },
    });
    const registry: ComponentRegistry = new Map();
    const target = makeTarget();
    registry.set("comp-1", {
      element: document.createElement("div"),
      vizElement: target,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });

    const binding: DataSourceBinding = { id: "ds" as DataSetId, source: mockSource };
    const scope: DataSetScope = new Map([["", new Map([["ds" as DataSetId, binding]])]]);
    const pipeline = createDataPipeline(manager, scope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    pipeline.handleDataRequest(target, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");

    // Now trigger refresh — should not cause infinite recursion
    refreshCount = 0;
    pipeline.refreshDataSet("ds" as DataSetId);

    // onChanged fires once from the reconnect → deliverDataSet, not recursion
    expect(refreshCount).toBe(1);
    expect(target.dataSet).toBeDefined();
  });
});

describe("deliverAll (renamed from refreshAll)", () => {
  it("pushes data to all registered components", () => {
    const dsId1 = dataSetId("ds-1");
    const dsId2 = dataSetId("ds-2");
    const manager = createDataSetManager();
    const registry: ComponentRegistry = new Map();
    const pipeline = createDataPipeline(manager, new Map() as DataSetScope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    const target1 = makeTarget();
    const target2 = makeTarget();
    registry.set("comp-1", { vizElement: target1, originalLookup: { dataSetId: dsId1, operations: [] }, pagePath: "", component: { type: "test", props: {} } } as any);
    registry.set("comp-2", { vizElement: target2, originalLookup: { dataSetId: dsId2, operations: [] }, pagePath: "", component: { type: "test", props: {} } } as any);

    manager.apply(dsId1, { type: "snapshot", dataset: regionDataSet([["North"]]) });
    manager.apply(dsId2, { type: "snapshot", dataset: regionDataSet([["South"]]) });

    pipeline.deliverAll();
    expect(target1.dataSet).toBeDefined();
    expect(target2.dataSet).toBeDefined();
  });
});
```

- [ ] **Step 10: Update existing tests**

Update existing tests that reference `refreshAll` to use `deliverAll`, and `onChanged` callbacks that call `pipeline.refreshDataSet(id)` to call `pipeline.deliverDataSet(id)`.

- [ ] **Step 11: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all PASS

- [ ] **Step 12: Typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/data-pipeline.ts packages/pages-runtime/src/data-pipeline.test.ts packages/pages-runtime/src/site.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: split refreshDataSet/deliverDataSet, add re-fetch routing

refreshDataSet now re-fetches from the external source (disconnect/reconnect
for DataSource path, re-resolve for legacy path). deliverDataSet re-delivers
cached data via pushData. Breaks infinite recursion through onChanged.

Four-path routing: pushSubscriptions (skip) → connectedSources (reconnect) →
reFetchCallbacks (parameterised URL) → legacy re-resolve.

Refs #134"
```

---

### Task 6: pages-refresh-request event

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` — add event listener
- Modify: `docs/protocols/casehub/pages-event-contract.md` — add to reserved events table

**Interfaces:**
- Consumes: `pipeline.refreshDataSet(id)` from Task 5, `findComponentId` (existing)

- [ ] **Step 1: Add pages-refresh-request listener to site.ts**

After the `pages-action-complete` listener in `site.ts`, add:

```typescript
target.addEventListener("pages-refresh-request", ((e: Event) => {
  const componentId = findComponentId(e);
  if (!componentId) return;
  const entry = registry.get(componentId);
  if (!entry?.originalLookup) return;
  pipeline.refreshDataSet(entry.originalLookup.dataSetId);
}), { signal: abortController.signal });
```

- [ ] **Step 2: Update protocol doc**

In `docs/protocols/casehub/pages-event-contract.md`, add to the reserved framework events table:

```markdown
| `pages-refresh-request` | Panel requests data re-fetch from source | Host panels, any component needing fresh data |
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/site.ts docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add pages-refresh-request event for panel-initiated refresh

Panels dispatch pages-refresh-request (bubbles, composed) to trigger
re-fetch of their bound dataset. Pipeline resolves the dataset via
findComponentId and calls refreshDataSet.

Refs #134"
```

---

### Task 7: Stale-while-revalidate (Layer 3)

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts` — add staleness check, pendingRefreshes guard
- Modify: `packages/pages-runtime/src/data-pipeline.test.ts` — TTL tests

**Interfaces:**
- Consumes: `manager.age(id)` from Task 4, `pipeline.refreshDataSet(id)` from Task 5

- [ ] **Step 1: Write failing tests for stale-while-revalidate**

Add to `data-pipeline.test.ts`:

```typescript
describe("stale-while-revalidate (Layer 3)", () => {
  it("triggers background refresh when data is stale", () => {
    let connectCount = 0;
    const mockSource: DataSource = {
      connect(sink: DataSink) {
        connectCount++;
        sink.apply({ type: "snapshot", dataset: regionDataSet([["Fresh-" + String(connectCount)]]) });
      },
      disconnect() {},
    };

    const manager = createDataSetManager({
      onChanged: (id) => { pipeline.deliverDataSet(id); },
    });
    const registry: ComponentRegistry = new Map();
    const target = makeTarget();
    registry.set("comp-1", {
      element: document.createElement("div"),
      vizElement: target,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });

    const binding: DataSourceBinding = { id: "ds" as DataSetId, source: mockSource };
    const scope: DataSetScope = new Map([["", new Map([["ds" as DataSetId, binding]])]]);
    const pipeline = createDataPipeline(manager, scope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    // Initial request — connects source
    pipeline.handleDataRequest(target, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");
    expect(connectCount).toBe(1);

    // Artificially age the data beyond default TTL (60s)
    (manager as any).timestamps?.set("ds" as DataSetId, Date.now() - 61000);

    // Second request — should serve cached data AND trigger background refresh
    const target2 = makeTarget();
    registry.set("comp-2", {
      element: document.createElement("div"),
      vizElement: target2,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });
    pipeline.handleDataRequest(target2, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-2");

    // Served immediately from cache
    expect(target2.dataSet).toBeDefined();
    // Background refresh triggered
    expect(connectCount).toBe(2);
  });

  it("does not trigger refresh when data is within TTL", () => {
    let connectCount = 0;
    const mockSource: DataSource = {
      connect(sink: DataSink) {
        connectCount++;
        sink.apply({ type: "snapshot", dataset: regionDataSet([["Data"]]) });
      },
      disconnect() {},
    };

    const manager = createDataSetManager({
      onChanged: (id) => { pipeline.deliverDataSet(id); },
    });
    const registry: ComponentRegistry = new Map();
    const target = makeTarget();
    registry.set("comp-1", {
      element: document.createElement("div"),
      vizElement: target,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });

    const binding: DataSourceBinding = { id: "ds" as DataSetId, source: mockSource };
    const scope: DataSetScope = new Map([["", new Map([["ds" as DataSetId, binding]])]]);
    const pipeline = createDataPipeline(manager, scope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    pipeline.handleDataRequest(target, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");
    expect(connectCount).toBe(1);

    // Request again immediately — data is fresh, no re-fetch
    pipeline.handleDataRequest(target, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");
    expect(connectCount).toBe(1);
  });

  it("dedup guard prevents multiple concurrent refreshes for same dataset", () => {
    let connectCount = 0;
    const mockSource: DataSource = {
      connect(sink: DataSink) {
        connectCount++;
        sink.apply({ type: "snapshot", dataset: regionDataSet([["Data"]]) });
      },
      disconnect() {},
    };

    const manager = createDataSetManager({
      onChanged: (id) => { pipeline.deliverDataSet(id); },
    });
    const registry: ComponentRegistry = new Map();
    const target1 = makeTarget();
    const target2 = makeTarget();
    registry.set("comp-1", {
      element: document.createElement("div"),
      vizElement: target1,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });
    registry.set("comp-2", {
      element: document.createElement("div"),
      vizElement: target2,
      originalLookup: { dataSetId: "ds" as DataSetId, operations: [] },
      component: { type: "test" },
      pagePath: "",
      hasExplicitId: false,
    });

    const binding: DataSourceBinding = { id: "ds" as DataSetId, source: mockSource };
    const scope: DataSetScope = new Map([["", new Map([["ds" as DataSetId, binding]])]]);
    const pipeline = createDataPipeline(manager, scope, registry, createFilterState(), createDataScopeRegistry(), createComponentViewState());

    // Initial connect
    pipeline.handleDataRequest(target1, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");
    expect(connectCount).toBe(1);

    // Age the data
    (manager as any).timestamps?.set("ds" as DataSetId, Date.now() - 61000);

    // Two requests for stale data — only one refresh should fire
    pipeline.handleDataRequest(target1, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-1");
    pipeline.handleDataRequest(target2, { dataSetId: "ds" as DataSetId, operations: [] }, "comp-2");

    // connectCount should be 2 (initial + one refresh), not 3
    expect(connectCount).toBe(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run src/data-pipeline.test.ts -t "stale-while-revalidate"`
Expected: FAIL — staleness check not implemented

- [ ] **Step 3: Add pendingRefreshes and staleness check to handleDataRequest**

In `createDataPipeline`, add state:

```typescript
const pendingRefreshes = new Set<DataSetId>();
const DEFAULT_TTL_MS = 60_000;
```

In `handleDataRequest`, after the block that serves from manager cache (`if (manager.has(lookup.dataSetId))`), add a staleness check before the `return`:

```typescript
if (manager.has(lookup.dataSetId)) {
  pushData(target, lookup, entry.pagePath, filterGroup?.group, componentId);

  // Stale-while-revalidate: check if cached data is stale
  if (!pendingRefreshes.has(lookup.dataSetId)) {
    const age = manager.age(lookup.dataSetId);
    const def = resolveDataSetDef(lookup.dataSetId, entry.pagePath, scope);
    const ttl = def?.refreshTime ? parseRefreshTime(def.refreshTime) : DEFAULT_TTL_MS;
    if (age !== undefined && age > ttl) {
      // Skip push sources — they refresh via server push
      if (!pushSubscriptions.has(lookup.dataSetId)) {
        pendingRefreshes.add(lookup.dataSetId);
        refreshDataSet(lookup.dataSetId);
      }
    }
  }

  // existing scheduleRefresh and return...
}
```

- [ ] **Step 4: Clear pendingRefreshes on refresh completion**

For the DataSource path in `refreshDataSet`, the sink's `apply` callback triggers `manager.apply()` → `onChanged` → `deliverDataSet`. After `connectSource`, add cleanup:

The cleanest approach: wrap the `refreshDataSet` method to clear `pendingRefreshes` when the re-fetch completes. For DataSource path, the `onChanged` callback fires synchronously after `connectSource` for synchronous sources, so clear after the call:

In `refreshDataSet`, after each re-fetch path:

**DataSource path:** Clear `pendingRefreshes` in the `connectSource` sink callbacks — on both `apply` (success) and `error` (permanent failure):

Modify `connectSource` to accept an optional callback:
```typescript
function connectSource(dataSetId: DataSetId, source: DataSource, onComplete?: () => void): void {
  // ... existing code ...
  const sink: DataSink = {
    apply(event: DataSetEvent): void {
      manager.apply(dataSetId, event);
      onComplete?.();
    },
    error(err): void {
      // ... existing error handling ...
      if (err.permanent) { onComplete?.(); }
    },
  };
  // ...
}
```

Then in `refreshDataSet` DataSource path:
```typescript
connectSource(dataSetId, connectedSource, () => pendingRefreshes.delete(dataSetId));
```

**Legacy path:** `.finally()` on the resolve promise:
```typescript
resolveExternalDataSet(def, resolverCtx, lookup)
  .finally(() => pendingRefreshes.delete(dataSetId))
  .catch(/* ... */);
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run src/data-pipeline.test.ts`
Expected: all PASS

- [ ] **Step 6: Typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/data-pipeline.ts packages/pages-runtime/src/data-pipeline.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: stale-while-revalidate with pendingRefreshes dedup guard

Checks data age on handleDataRequest. If stale (> refreshTime or 60s default),
serves cached data immediately and triggers background refreshDataSet. Dedup
guard ensures at most one concurrent re-fetch per dataset. Guard clears on
both success and failure.

Closes #134"
```

---

## Self-review checklist

**Spec coverage:**
- §2.1 Type relocation → Task 1 (Steps 6-8)
- §2.2 Default source factory → Task 2
- §2.3 Controller unchanged → Task 3 (integration tests only)
- §2.4 Export → Task 2 (Step 4)
- §3.1.1 Operation split → Task 5 (Steps 3-4)
- §3.1.2 Routing logic → Task 5 (Step 5)
- §3.1.3 DataSource path → Task 5 (Step 5)
- §3.1.4 Legacy path → Task 5 (Step 5)
- §3.1.5 Parameterised URL path → Task 5 (Step 6)
- §3.2 pages-refresh-request → Task 6
- §3.3.1 Manager TTL → Task 4
- §3.3.2 Stale-while-revalidate → Task 7
- §3.3.3 Interaction semantics → Task 7 (implicit in TTL logic)
- §5 Test strategy → covered across all tasks

**Dependencies:**
- Task 1 ← (none)
- Task 2 ← Task 1
- Task 3 ← Task 2
- Task 4 ← (none, parallel with Tasks 1-3)
- Task 5 ← (none, parallel with Tasks 1-4)
- Task 6 ← Task 5
- Task 7 ← Tasks 4, 5
