# mutableRestSource Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #144 — feat: production mutableRestSource write path for DataAction dispatch
**Issue group:** #144

**Goal:** Add a `mutableRestSource()` factory that composes restSource for reads and maps DataAction dispatch to HTTP write requests with response-merge sync.

**Architecture:** `mutableRestSource(readUrl, writeConfig, options?)` wraps an internal `restSource` for the read path. `dispatch(action)` maps each DataAction type to an HTTP request via WriteConfig, then syncs local state by pushing sink events (append/replace/remove) from the response body, or re-fetching when `refreshAfterWrite` is set.

**Tech Stack:** TypeScript, Vitest, pages-data datasource module

## Global Constraints

- `dispatch()` returns `Promise<void>` (async interface change)
- Pessimistic sync — server response is source of truth, no optimistic local mutations
- Direct construction only — not integrated into source factory
- Uses injected `fetchFn` from options (same pattern as `restSource`)

---

### Task 1: Make MutableDataSource.dispatch async

**Files:**
- Modify: `packages/pages-data/src/datasource/types.ts:29-31`
- Modify: `packages/pages-data/src/datasource/sources/simulated/simulated-source.ts:193-195`
- Test: `packages/pages-data/src/datasource/sources/simulated/simulated-source.test.ts`

**Interfaces:**
- Consumes: nothing new
- Produces: `MutableDataSource.dispatch(action: DataAction): Promise<void>`

- [ ] **Step 1: Write the failing test**

Add to simulated-source.test.ts:

```typescript
it("dispatch returns a Promise", async () => {
  const source = simulated({ initial: snapshotSource, controller, interval: 0, mutations: [], keyColumn: "id" });
  source.connect(sink);
  await vi.waitFor(() => expect(events.length).toBeGreaterThan(0));
  const result = source.dispatch({ type: "create", data: { id: "new", name: "test" } });
  expect(result).toBeInstanceOf(Promise);
  await result;
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/simulated/simulated-source.test.ts`
Expected: FAIL — dispatch returns `void`, not `Promise`

- [ ] **Step 3: Update MutableDataSource interface**

In `packages/pages-data/src/datasource/types.ts`, change line 30:

```typescript
// Before:
dispatch(action: DataAction): void;

// After:
dispatch(action: DataAction): Promise<void>;
```

- [ ] **Step 4: Update simulated source dispatch**

In `packages/pages-data/src/datasource/sources/simulated/simulated-source.ts`, change the dispatch method (line 193-195):

```typescript
// Before:
dispatch(action: DataAction): void {
  applyAction(action);
},

// After:
dispatch(action: DataAction): Promise<void> {
  applyAction(action);
  return Promise.resolve();
},
```

- [ ] **Step 5: Run test to verify it passes**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/simulated/simulated-source.test.ts`
Expected: PASS

- [ ] **Step 6: Run full typecheck**

Run: `GH_PACKAGES_TOKEN=dummy yarn typecheck`
Expected: clean (no callers break — `void` is assignable from `Promise<void>` at call sites that ignore the return)

- [ ] **Step 7: Commit**

```
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/datasource/types.ts packages/pages-data/src/datasource/sources/simulated/simulated-source.ts packages/pages-data/src/datasource/sources/simulated/simulated-source.test.ts
git commit -m "refactor(#144): make MutableDataSource.dispatch async  Refs #144"
```

---

### Task 2: Implement mutableRestSource with core dispatch

**Files:**
- Create: `packages/pages-data/src/datasource/sources/mutable-rest-source.ts`
- Test: `packages/pages-data/src/datasource/sources/mutable-rest-source.test.ts`

**Interfaces:**
- Consumes: `DataSource`, `DataSink`, `DataAction`, `MutableDataSource`, `SourceError` from `../types.js`; `restSource` from `./rest-source.js`; `extractDataSet` from `../../dataset/external/extraction.js`; `DataSetEvent`, `ReplaceEvent`, `RemoveEvent` from `../../dataset/events.js`; `ColumnId` from `../../dataset/types.js`; `createTypedRow` from `../../dataset/conversion.js`
- Produces: `mutableRestSource(readUrl: string, writeConfig: WriteConfig, options?: MutableRestSourceOptions): MutableDataSource`; types `WriteConfig`, `WriteEndpoint`, `UrlTemplate`, `MutableRestSourceOptions`

- [ ] **Step 1: Write failing tests for update, create, delete**

Create `packages/pages-data/src/datasource/sources/mutable-rest-source.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from "vitest";
import { mutableRestSource } from "./mutable-rest-source.js";
import type { DataSink, SourceError, DataAction } from "../types.js";
import type { DataSetEvent } from "../../dataset/events.js";
import { ColumnType, col, makeDataset } from "./test-helpers.js";

function jsonResponse(data: unknown, status = 200): Response {
  return new Response(JSON.stringify(data), {
    status,
    headers: { "content-type": "application/json" },
  });
}

function collectSink(): { sink: DataSink; events: DataSetEvent[]; errors: SourceError[] } {
  const events: DataSetEvent[] = [];
  const errors: SourceError[] = [];
  return {
    sink: {
      apply(event) { events.push(event); },
      error(err) { errors.push(err); },
    },
    events,
    errors,
  };
}

const READ_URL = "https://api.example.com/items";
const COLUMNS = [col("id", ColumnType.TEXT), col("name", ColumnType.TEXT), col("status", ColumnType.LABEL)];

function snapshotFetch(): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue(jsonResponse([["1", "Alice", "Active"], ["2", "Bob", "Inactive"]]));
}

describe("mutableRestSource", () => {
  afterEach(() => { vi.restoreAllMocks(); });

  it("update dispatches PATCH with :key substitution and emits replace event", async () => {
    const writeFetch = vi.fn().mockResolvedValue(jsonResponse({ id: "1", name: "Alice Updated", status: "Active" }));
    const source = mutableRestSource(READ_URL, {
      update: { url: "https://api.example.com/items/:key" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "update", key: "1", changes: { name: "Alice Updated" } });

    expect(writeFetch).toHaveBeenCalledOnce();
    const [url, init] = writeFetch.mock.calls[0]!;
    expect(url).toBe("https://api.example.com/items/1");
    expect(init.method).toBe("PATCH");
    expect(JSON.parse(init.body as string)).toEqual({ name: "Alice Updated" });

    const replaceEvent = events.find(e => e.type === "replace");
    expect(replaceEvent).toBeDefined();
    expect(replaceEvent!.type).toBe("replace");
  });

  it("create dispatches POST and emits append event", async () => {
    const writeFetch = vi.fn().mockResolvedValue(jsonResponse({ id: "3", name: "Charlie", status: "Active" }));
    const source = mutableRestSource(READ_URL, {
      create: { url: "https://api.example.com/items" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "create", data: { id: "3", name: "Charlie", status: "Active" } });

    expect(writeFetch).toHaveBeenCalledOnce();
    const [url, init] = writeFetch.mock.calls[0]!;
    expect(url).toBe("https://api.example.com/items");
    expect(init.method).toBe("POST");

    const appendEvent = events.find(e => e.type === "append");
    expect(appendEvent).toBeDefined();
  });

  it("delete dispatches DELETE and emits remove event", async () => {
    const writeFetch = vi.fn().mockResolvedValue(new Response(null, { status: 204 }));
    const source = mutableRestSource(READ_URL, {
      delete: { url: "https://api.example.com/items/:key" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "delete", key: "2" });

    expect(writeFetch).toHaveBeenCalledOnce();
    const [url, init] = writeFetch.mock.calls[0]!;
    expect(url).toBe("https://api.example.com/items/2");
    expect(init.method).toBe("DELETE");
    expect(init.body).toBeUndefined();

    const removeEvent = events.find(e => e.type === "remove");
    expect(removeEvent).toBeDefined();
    expect((removeEvent as any).key).toBe("2");
  });

  it("unsupported action type rejects", async () => {
    const source = mutableRestSource(READ_URL, { keyColumn: "id" }, {
      columns: COLUMNS, fetchFn: snapshotFetch(),
    });
    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await expect(source.dispatch({ type: "update", key: "1", changes: {} }))
      .rejects.toThrow("Unsupported action type");
  });

  it("HTTP error rejects promise", async () => {
    const writeFetch = vi.fn().mockResolvedValue(new Response("Not Found", { status: 404 }));
    const source = mutableRestSource(READ_URL, {
      update: { url: "https://api.example.com/items/:key" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await expect(source.dispatch({ type: "update", key: "1", changes: {} }))
      .rejects.toThrow("404");
  });

  it("function URL template receives full action", async () => {
    const urlFn = vi.fn().mockReturnValue("https://api.example.com/custom/1");
    const writeFetch = vi.fn().mockResolvedValue(jsonResponse({ id: "1", name: "X", status: "Y" }));
    const source = mutableRestSource(READ_URL, {
      update: { url: urlFn },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    const action: DataAction = { type: "update", key: "1", changes: { name: "X" } };
    await source.dispatch(action);

    expect(urlFn).toHaveBeenCalledWith(action);
    expect(writeFetch.mock.calls[0]![0]).toBe("https://api.example.com/custom/1");
  });

  it("custom headers are sent with write requests", async () => {
    const writeFetch = vi.fn().mockResolvedValue(jsonResponse({ id: "1", name: "X", status: "Y" }));
    const source = mutableRestSource(READ_URL, {
      update: { url: "https://api.example.com/items/:key" },
      headers: { "Authorization": "Bearer tok123" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "update", key: "1", changes: { name: "X" } });

    const headers = writeFetch.mock.calls[0]![1].headers;
    expect(headers["Authorization"]).toBe("Bearer tok123");
  });

  it("refreshAfterWrite re-fetches read URL instead of merging", async () => {
    const readFetch = vi.fn()
      .mockResolvedValueOnce(jsonResponse([["1", "Alice", "Active"]]))
      .mockResolvedValueOnce(jsonResponse([["1", "Alice Updated", "Active"]]));
    const writeFetch = vi.fn().mockResolvedValue(new Response(null, { status: 204 }));
    const source = mutableRestSource(READ_URL, {
      update: { url: "https://api.example.com/items/:key" },
      refreshAfterWrite: true,
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: readFetch, writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "update", key: "1", changes: { name: "Alice Updated" } });

    expect(readFetch).toHaveBeenCalledTimes(2);
    const snapshotEvents = events.filter(e => e.type === "snapshot");
    expect(snapshotEvents.length).toBe(2);
  });

  it("204 No Content auto-refreshes when refreshAfterWrite is false", async () => {
    const readFetch = vi.fn()
      .mockResolvedValueOnce(jsonResponse([["1", "Alice", "Active"]]))
      .mockResolvedValueOnce(jsonResponse([["1", "Alice", "Active"], ["3", "Charlie", "Active"]]));
    const writeFetch = vi.fn().mockResolvedValue(new Response(null, { status: 204 }));
    const source = mutableRestSource(READ_URL, {
      create: { url: "https://api.example.com/items" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: readFetch, writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "create", data: { id: "3", name: "Charlie", status: "Active" } });

    expect(readFetch).toHaveBeenCalledTimes(2);
  });

  it("custom method override works", async () => {
    const writeFetch = vi.fn().mockResolvedValue(jsonResponse({ id: "1", name: "X", status: "Y" }));
    const source = mutableRestSource(READ_URL, {
      update: { url: "https://api.example.com/items/:key", method: "PUT" },
      keyColumn: "id",
    }, { columns: COLUMNS, fetchFn: snapshotFetch(), writeFetchFn: writeFetch });

    const { sink, events } = collectSink();
    source.connect(sink);
    await vi.waitFor(() => expect(events.length).toBe(1));

    await source.dispatch({ type: "update", key: "1", changes: { name: "X" } });
    expect(writeFetch.mock.calls[0]![1].method).toBe("PUT");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/mutable-rest-source.test.ts`
Expected: FAIL — module `./mutable-rest-source.js` not found

- [ ] **Step 3: Implement mutableRestSource**

Create `packages/pages-data/src/datasource/sources/mutable-rest-source.ts`:

```typescript
import type { DataSource, DataSink, MutableDataSource, DataAction, SourceError } from "../types.js";
import type { DataSetEvent } from "../../dataset/events.js";
import type { ReplaceEvent, RemoveEvent } from "../../dataset/events.js";
import type { Column, ColumnId, TypedRow } from "../../dataset/types.js";
import type { ExternalColumnDef } from "../../dataset/external/types.js";
import { restSource } from "./rest-source.js";
import { extractDataSet } from "../../dataset/external/extraction.js";
import { createTypedRow } from "../../dataset/conversion.js";

export type UrlTemplate = string | ((action: DataAction) => string);

export interface WriteEndpoint {
  readonly url: UrlTemplate;
  readonly method?: string;
}

export interface WriteConfig {
  readonly update?: WriteEndpoint;
  readonly create?: WriteEndpoint;
  readonly delete?: WriteEndpoint;
  readonly headers?: Record<string, string>;
  readonly refreshAfterWrite?: boolean;
  readonly keyColumn?: string;
}

export interface MutableRestSourceOptions {
  readonly columns?: readonly ExternalColumnDef[];
  readonly dataPath?: string;
  readonly fetchFn?: typeof globalThis.fetch;
  readonly writeFetchFn?: typeof globalThis.fetch;
}

const DEFAULT_METHODS: Record<string, string> = {
  update: "PATCH",
  create: "POST",
  delete: "DELETE",
};

export function mutableRestSource(
  readUrl: string,
  writeConfig: WriteConfig,
  options?: MutableRestSourceOptions,
): MutableDataSource {
  const readFetchFn = options?.fetchFn ?? globalThis.fetch;
  const writeFetchFn = options?.writeFetchFn ?? options?.fetchFn ?? globalThis.fetch;
  const keyColumnId = writeConfig.keyColumn as ColumnId | undefined;

  let currentSink: DataSink | null = null;
  let columns: readonly Column[] = [];

  const inner = restSource(readUrl, "" as any, {
    columns: options?.columns,
    dataPath: options?.dataPath,
    fetchFn: readFetchFn,
  });

  const wrapperSink: DataSink = {
    apply(event: DataSetEvent): void {
      if (event.type === "snapshot") {
        columns = event.dataset.columns;
      }
      currentSink?.apply(event);
    },
    error(err: SourceError): void {
      currentSink?.error(err);
    },
  };

  function resolveUrl(endpoint: WriteEndpoint, action: DataAction): string {
    if (typeof endpoint.url === "function") return endpoint.url(action);
    if ("key" in action) return endpoint.url.replace(":key", action.key);
    return endpoint.url;
  }

  async function doRefresh(): Promise<void> {
    return new Promise<void>((resolve, reject) => {
      const refreshSink: DataSink = {
        apply(event: DataSetEvent): void {
          wrapperSink.apply(event);
          resolve();
        },
        error(err: SourceError): void {
          wrapperSink.error(err);
          reject(new Error(err.message));
        },
      };
      const refreshSource = restSource(readUrl, "" as any, {
        columns: options?.columns,
        dataPath: options?.dataPath,
        fetchFn: readFetchFn,
      });
      refreshSource.connect(refreshSink);
    });
  }

  function responseToRow(data: Record<string, unknown>): TypedRow {
    const cells = columns.map(col => {
      const raw = data[col.id as string];
      if (raw === null || raw === undefined) return { type: "NULL" as const };
      return { type: col.type, value: raw } as any;
    });
    return createTypedRow(cells, columns);
  }

  async function dispatchAction(action: DataAction): Promise<void> {
    const endpoint = writeConfig[action.type];
    if (!endpoint) throw new Error(`Unsupported action type: ${action.type}`);

    const url = resolveUrl(endpoint, action);
    const method = endpoint.method ?? DEFAULT_METHODS[action.type] ?? "POST";
    const headers: Record<string, string> = {
      "Content-Type": "application/json",
      ...writeConfig.headers,
    };

    const body = action.type === "delete"
      ? undefined
      : JSON.stringify(action.type === "create" ? action.data : action.changes);

    const response = await writeFetchFn(url, { method, headers, body });

    if (!response.ok) {
      const text = await response.text().catch(() => "");
      throw new Error(`${String(response.status)}: ${text}`);
    }

    if (writeConfig.refreshAfterWrite) {
      await doRefresh();
      return;
    }

    const contentLength = response.headers.get("content-length");
    const hasBody = response.status !== 204 && contentLength !== "0";

    if (!hasBody) {
      await doRefresh();
      return;
    }

    const responseData = await response.json() as Record<string, unknown>;

    if (!currentSink || !keyColumnId) return;

    switch (action.type) {
      case "create":
        currentSink.apply({ type: "append", rows: [responseToRow(responseData)] });
        break;
      case "update":
        currentSink.apply({
          type: "replace",
          keyColumn: keyColumnId,
          key: action.key,
          row: responseToRow(responseData),
        } satisfies ReplaceEvent);
        break;
      case "delete":
        currentSink.apply({
          type: "remove",
          keyColumn: keyColumnId,
          key: action.key,
        } satisfies RemoveEvent);
        break;
    }
  }

  return {
    connect(sink: DataSink): void {
      currentSink = sink;
      inner.connect(wrapperSink);
    },
    disconnect(): void {
      inner.disconnect();
      currentSink = null;
    },
    dispatch(action: DataAction): Promise<void> {
      return dispatchAction(action);
    },
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-data run test -- --run src/datasource/sources/mutable-rest-source.test.ts`
Expected: all 10 tests PASS

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/datasource/sources/mutable-rest-source.ts packages/pages-data/src/datasource/sources/mutable-rest-source.test.ts
git commit -m "feat(#144): implement mutableRestSource with dispatch  Refs #144"
```

---

### Task 3: Export and typecheck

**Files:**
- Modify: `packages/pages-data/src/datasource/index.ts`

**Interfaces:**
- Consumes: `mutableRestSource`, `WriteConfig`, `WriteEndpoint`, `UrlTemplate`, `MutableRestSourceOptions` from Task 2
- Produces: public API exports from `@casehubio/pages-data`

- [ ] **Step 1: Add exports to datasource index**

In `packages/pages-data/src/datasource/index.ts`, add after the `restSource` export block:

```typescript
export { mutableRestSource } from "./sources/mutable-rest-source.js";
export type { WriteConfig, WriteEndpoint, UrlTemplate, MutableRestSourceOptions } from "./sources/mutable-rest-source.js";
```

- [ ] **Step 2: Run full typecheck**

Run: `GH_PACKAGES_TOKEN=dummy yarn typecheck`
Expected: clean

- [ ] **Step 3: Run full test suite for pages-data**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-data run test -- --run`
Expected: all tests pass (including existing simulated source tests with the async dispatch change)

- [ ] **Step 4: Commit**

```
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/datasource/index.ts
git commit -m "feat(#144): export mutableRestSource from pages-data  Closes #144"
```
