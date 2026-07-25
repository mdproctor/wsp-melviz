# Server-Query Push-Down Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the backend `/api/dataset/query` SQL push-down endpoint to the frontend data pipeline so components can delegate filter/group/sort operations to the database.

**Architecture:** A new `serverQuery` source type on `ExternalDataSetDef` triggers a dedicated resolver path that POSTs `DataSetLookup` to the backend. The data pipeline tracks server-query datasets and skips client-side re-application of YAML-defined operations — only runtime interactive ops (cross-filters, user sort, text filter) apply client-side.

**Tech Stack:** TypeScript 5, Vitest, `@casehubio/pages-data`, `@casehubio/pages-runtime`

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-03-server-query-pushdown-design.md`
- Test framework: Vitest (colocated `*.test.ts` files)
- Branded types: use `dataSetId()` and `columnId()` constructors from `@casehubio/pages-data`
- Imports use `.js` extensions (ESM)
- All new types are `readonly`
- Tests colocated alongside source files (e.g., `resolver.test.ts` next to `resolver.ts`)

---

### Task 1: Types and schema validation

**Files:**
- Modify: `packages/pages-data/src/dataset/external/types.ts`
- Modify: `packages/pages-data/src/dataset/external/schema.ts`
- Test: `packages/pages-data/src/dataset/external/schema.test.ts`

**Produces:**
- `ExternalDataSetDef.serverQuery?: boolean`
- `DataProviderConfig.serverQuery?: { readonly endpoint: string; readonly tokenFn?: () => string | null }`
- `ResolveResult.source` union includes `"serverQuery"`
- Schema: `parseExternalDataSetDef` accepts `serverQuery` as fourth source type

- [ ] **Step 1: Write failing schema tests**

Add to `schema.test.ts`:

```typescript
it("accepts serverQuery-based definition", () => {
  const result = parseExternalDataSetDef({ uuid: "sq-1", serverQuery: true });
  expect(result.serverQuery).toBe(true);
});

it("rejects serverQuery combined with url", () => {
  expect(() => parseExternalDataSetDef({
    uuid: "sq-1", serverQuery: true, url: "https://x.com",
  })).toThrow(/Exactly one/);
});

it("rejects extraction fields with serverQuery", () => {
  expect(() => parseExternalDataSetDef({
    uuid: "sq-1", serverQuery: true, dataPath: "results",
  })).toThrow(/not valid with serverQuery/);
});

it("accepts refreshTime with serverQuery", () => {
  const result = parseExternalDataSetDef({
    uuid: "sq-1", serverQuery: true, refreshTime: "30second",
  });
  expect(result.refreshTime).toBe("30second");
});

it("rejects http options with serverQuery", () => {
  expect(() => parseExternalDataSetDef({
    uuid: "sq-1", serverQuery: true, method: "POST",
  })).toThrow(/only valid when url is set/);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/external/schema.test.ts`
Expected: 5 new tests FAIL

- [ ] **Step 3: Update types**

In `types.ts`, add `serverQuery` to `ExternalDataSetDef`:

```typescript
export interface ExternalDataSetDef {
  // ... existing fields ...
  readonly serverQuery?: boolean;
}
```

Add `serverQuery` block to `DataProviderConfig`:

```typescript
export interface DataProviderConfig {
  // ... existing fields ...
  readonly serverQuery?: {
    readonly endpoint: string;
    readonly tokenFn?: () => string | null;
  };
}
```

Update `ResolveResult.source` union:

```typescript
export interface ResolveResult {
  readonly dataset: TypedDataSet;
  readonly inferredColumns: boolean;
  readonly source: "url" | "content" | "join" | "serverQuery";
}
```

- [ ] **Step 4: Update schema validation**

In `schema.ts`, add `serverQuery` to the object schema:

```typescript
serverQuery: z.boolean().optional(),
```

Update the source-count refinement to include `serverQuery`:

```typescript
.refine(
  d => [d.url, d.content, d.join, d.serverQuery].filter(Boolean).length === 1,
  { message: "Exactly one of url, content, join, or serverQuery is required" },
)
```

Add extraction field restriction for `serverQuery` (after the existing `join` restriction):

```typescript
.refine(
  d => !d.serverQuery || [d.dataPath, d.type, d.expression, d.accumulate]
    .every(v => v === undefined),
  { message: "dataPath, type, expression, accumulate are not valid with serverQuery" },
)
```

Update the `refreshTime` refinement to include `serverQuery`:

```typescript
.refine(
  d => !d.refreshTime || (
    (d.url !== undefined
      && !d.url.startsWith("ws://") && !d.url.startsWith("wss://")
      && !d.url.startsWith("sse://") && !d.url.startsWith("sses://"))
    || (d.content !== undefined && d.expression !== undefined && d.accumulate === true)
    || d.serverQuery === true
  ),
  { message: "refreshTime requires a non-push-source url, content + expression + accumulate, or serverQuery" },
)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/external/schema.test.ts`
Expected: All tests PASS (existing + 5 new)

- [ ] **Step 6: Commit**

```bash
git add packages/pages-data/src/dataset/external/types.ts packages/pages-data/src/dataset/external/schema.ts packages/pages-data/src/dataset/external/schema.test.ts
git commit -m "feat(pages-data): add serverQuery source type and schema validation

Refs #21"
```

---

### Task 2: ServerQueryClient

**Files:**
- Create: `packages/pages-data/src/dataset/external/providers/server-query.ts`
- Create: `packages/pages-data/src/dataset/external/providers/server-query.test.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts`

**Consumes:** `DataSetLookup` from `../../../lookup.js`, `DataSet`/`TypedDataSet`/`Column`/`ColumnType` from `../../../types.js`, `toTypedDataSet` from `../../../conversion.js`, `DataSetError` from `../../../errors.js`

**Produces:** `ServerQueryClient` class with `query(lookup: DataSetLookup): Promise<TypedDataSet>`

- [ ] **Step 1: Write failing tests**

Create `server-query.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { ServerQueryClient } from "./server-query.js";
import { dataSetId } from "../../../types.js";
import type { DataSetLookup } from "../../../lookup.js";

function makeLookup(id: string): DataSetLookup {
  return { dataSetId: dataSetId(id), operations: [] };
}

function mockFetch(body: unknown, status = 200): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue({
    ok: status >= 200 && status < 300,
    status,
    json: () => Promise.resolve(body),
    text: () => Promise.resolve(JSON.stringify(body)),
  }) as unknown as typeof globalThis.fetch;
}

describe("ServerQueryClient", () => {
  it("POSTs DataSetLookup and returns TypedDataSet", async () => {
    const response = {
      columns: [{ id: "name", name: "Name", type: "LABEL" }],
      rows: [["Alice"], ["Bob"]],
    };
    const fetchFn = mockFetch(response);
    const client = new ServerQueryClient("/api/dataset/query", fetchFn);

    const result = await client.query(makeLookup("ds-1"));

    expect(fetchFn).toHaveBeenCalledWith("/api/dataset/query", expect.objectContaining({
      method: "POST",
      headers: { "Content-Type": "application/json" },
    }));
    expect(result.columns).toHaveLength(1);
    expect(result.columns[0].name).toBe("Name");
    expect(result.rows).toHaveLength(2);
  });

  it("adds Authorization header when tokenFn returns a token", async () => {
    const response = { columns: [], rows: [] };
    const fetchFn = mockFetch(response);
    const tokenFn = () => "jwt-token-123";
    const client = new ServerQueryClient("/api/dataset/query", fetchFn, tokenFn);

    await client.query(makeLookup("ds-1"));

    expect(fetchFn).toHaveBeenCalledWith("/api/dataset/query", expect.objectContaining({
      headers: { "Content-Type": "application/json", "Authorization": "Bearer jwt-token-123" },
    }));
  });

  it("omits Authorization header when tokenFn returns null", async () => {
    const response = { columns: [], rows: [] };
    const fetchFn = mockFetch(response);
    const tokenFn = () => null;
    const client = new ServerQueryClient("/api/dataset/query", fetchFn, tokenFn);

    await client.query(makeLookup("ds-1"));

    const calledHeaders = (fetchFn as ReturnType<typeof vi.fn>).mock.calls[0][1].headers as Record<string, string>;
    expect(calledHeaders["Authorization"]).toBeUndefined();
  });

  it("dispatches pages-auth-expired on 401 and throws", async () => {
    const fetchFn = mockFetch({ error: "Unauthorized" }, 401);
    const client = new ServerQueryClient("/api/dataset/query", fetchFn);

    const dispatchSpy = vi.spyOn(document, "dispatchEvent");

    await expect(client.query(makeLookup("ds-1"))).rejects.toThrow("FETCH_FAILED");
    expect(dispatchSpy).toHaveBeenCalledWith(expect.objectContaining({ type: "pages-auth-expired" }));

    dispatchSpy.mockRestore();
  });

  it("throws DataSetError on non-ok response", async () => {
    const fetchFn = mockFetch({ error: "Server error" }, 500);
    const client = new ServerQueryClient("/api/dataset/query", fetchFn);

    await expect(client.query(makeLookup("ds-1"))).rejects.toThrow("FETCH_FAILED");
  });

  it("maps null values in rows correctly", async () => {
    const response = {
      columns: [
        { id: "name", name: "Name", type: "LABEL" },
        { id: "age", name: "Age", type: "NUMBER" },
      ],
      rows: [["Alice", "30"], [null, null]],
    };
    const fetchFn = mockFetch(response);
    const client = new ServerQueryClient("/api/dataset/query", fetchFn);

    const result = await client.query(makeLookup("ds-1"));

    expect(result.rows).toHaveLength(2);
    expect(result.rows[1].cells[0].type).toBe("NULL");
    expect(result.rows[1].cells[1].type).toBe("NULL");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/external/providers/server-query.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement ServerQueryClient**

Create `server-query.ts`:

```typescript
import type { DataSetLookup } from "../../../lookup.js";
import type { Column, DataSet, TypedDataSet } from "../../../types.js";
import { ColumnType } from "../../../types.js";
import { toTypedDataSet } from "../../../conversion.js";
import { DataSetError } from "../../../errors.js";

interface ServerQueryResponse {
  readonly columns: readonly { readonly id: string; readonly name: string; readonly type: string }[];
  readonly rows: readonly (readonly (string | null)[])[];
}

function mapColumnType(type: string): ColumnType {
  switch (type) {
    case "NUMBER": return ColumnType.NUMBER;
    case "DATE": return ColumnType.DATE;
    case "TEXT": return ColumnType.TEXT;
    default: return ColumnType.LABEL;
  }
}

function toDataSet(response: ServerQueryResponse): DataSet {
  const columns: Column[] = response.columns.map(c => ({
    id: c.id as Column["id"],
    name: c.name,
    type: mapColumnType(c.type),
  }));
  return { columns, data: response.rows };
}

export class ServerQueryClient {
  constructor(
    private readonly endpoint: string,
    private readonly fetchFn: typeof globalThis.fetch,
    private readonly tokenFn?: () => string | null,
  ) {}

  async query(lookup: DataSetLookup): Promise<TypedDataSet> {
    const headers: Record<string, string> = { "Content-Type": "application/json" };
    const token = this.tokenFn?.();
    if (token) {
      headers["Authorization"] = `Bearer ${token}`;
    }

    let response: Response;
    try {
      response = await this.fetchFn(this.endpoint, {
        method: "POST",
        headers,
        body: JSON.stringify(lookup),
      });
    } catch (e) {
      throw new DataSetError(
        "FETCH_FAILED",
        `Server query failed for "${String(lookup.dataSetId)}": ${e instanceof Error ? e.message : String(e)}`,
        e,
      );
    }

    if (response.status === 401) {
      if (typeof document !== "undefined") {
        document.dispatchEvent(new CustomEvent("pages-auth-expired"));
      }
      throw new DataSetError("FETCH_FAILED", "Authentication expired");
    }

    if (!response.ok) {
      const text = await response.text().catch(() => "");
      throw new DataSetError(
        "FETCH_FAILED",
        `Server query failed for "${String(lookup.dataSetId)}": HTTP ${String(response.status)} ${text}`,
      );
    }

    const body = await response.json() as ServerQueryResponse;
    return toTypedDataSet(toDataSet(body));
  }
}
```

- [ ] **Step 4: Add export to index.ts**

In `index.ts`, add after the existing `ServerRelayProvider` export:

```typescript
export { ServerQueryClient } from "./providers/server-query.js";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/external/providers/server-query.test.ts`
Expected: All 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-data/src/dataset/external/providers/server-query.ts packages/pages-data/src/dataset/external/providers/server-query.test.ts packages/pages-data/src/dataset/external/index.ts
git commit -m "feat(pages-data): add ServerQueryClient for SQL push-down

Refs #21"
```

---

### Task 3: Resolver serverQuery route

**Files:**
- Modify: `packages/pages-data/src/dataset/external/resolver.ts`
- Test: `packages/pages-data/src/dataset/external/resolver.test.ts`

**Consumes:** `ServerQueryClient` from `./providers/server-query.js`, `DataSetLookup` from `../lookup.js`, `DataProviderConfig` from `./types.js`

**Produces:** `resolveExternalDataSet(def, ctx, lookup?)` — updated signature with optional `lookup` parameter

- [ ] **Step 1: Write failing resolver tests**

Add to `resolver.test.ts`:

```typescript
import { dataSetId } from "../types.js";

describe("resolveExternalDataSet — serverQuery", () => {
  it("routes serverQuery to ServerQueryClient and stores snapshot", async () => {
    const manager = createDataSetManager();
    const lookup = { dataSetId: dataSetId("sq-ds"), operations: [] };
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      status: 200,
      json: () => Promise.resolve({
        columns: [{ id: "name", name: "Name", type: "LABEL" }],
        rows: [["Alice"]],
      }),
    }) as unknown as typeof globalThis.fetch;

    const ctx = makeCtx({
      manager,
      providerConfig: {
        serverQuery: { endpoint: "/api/dataset/query" },
      },
    });

    const def: ExternalDataSetDef = {
      uuid: dataSetId("sq-ds"),
      serverQuery: true,
    };

    const result = await resolveExternalDataSet(def, ctx, lookup, mockFetch);

    expect(result.source).toBe("serverQuery");
    expect(result.inferredColumns).toBe(false);
    expect(manager.has(dataSetId("sq-ds"))).toBe(true);
  });

  it("throws CONFIG_MISSING when serverQuery is true but config is absent", async () => {
    const ctx = makeCtx({ providerConfig: {} });
    const def: ExternalDataSetDef = {
      uuid: dataSetId("sq-ds"),
      serverQuery: true,
    };

    await expect(resolveExternalDataSet(def, ctx)).rejects.toThrow("CONFIG_MISSING");
  });

  it("passes lookup operations to ServerQueryClient", async () => {
    const capturedBodies: string[] = [];
    const mockFetch = vi.fn().mockImplementation((_url: string, init: RequestInit) => {
      capturedBodies.push(init.body as string);
      return Promise.resolve({
        ok: true,
        status: 200,
        json: () => Promise.resolve({ columns: [], rows: [] }),
      });
    }) as unknown as typeof globalThis.fetch;

    const ctx = makeCtx({
      providerConfig: {
        serverQuery: { endpoint: "/api/dataset/query" },
      },
    });

    const lookup = {
      dataSetId: dataSetId("sq-ds"),
      operations: [{ type: "sort" as const, columns: [{ columnId: "name" as any, order: "ASCENDING" as const }] }],
    };

    const def: ExternalDataSetDef = {
      uuid: dataSetId("sq-ds"),
      serverQuery: true,
    };

    await resolveExternalDataSet(def, ctx, lookup, mockFetch);

    const parsed = JSON.parse(capturedBodies[0]);
    expect(parsed.operations).toHaveLength(1);
    expect(parsed.operations[0].type).toBe("sort");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/external/resolver.test.ts`
Expected: New tests FAIL (signature mismatch or missing route)

- [ ] **Step 3: Implement resolver serverQuery route**

In `resolver.ts`, add imports:

```typescript
import type { DataSetLookup } from "../lookup.js";
import { ServerQueryClient } from "./providers/server-query.js";
```

Update `resolveExternalDataSet` signature to accept optional `lookup` and `fetchFn`:

```typescript
export async function resolveExternalDataSet(
  def: ExternalDataSetDef,
  ctx: ResolverContext,
  lookup?: DataSetLookup,
  fetchFn?: typeof globalThis.fetch,
): Promise<ResolveResult> {
  // ---- Server-query route (early return — bypasses validate/determineSource) ----
  if (def.serverQuery) {
    if (!ctx.providerConfig.serverQuery) {
      throw new DataSetError(
        "CONFIG_MISSING" as any,
        `Dataset "${String(def.uuid)}" uses serverQuery but no serverQuery config is provided`,
      );
    }
    const config = ctx.providerConfig.serverQuery;
    const client = new ServerQueryClient(
      config.endpoint,
      fetchFn ?? globalThis.fetch.bind(globalThis),
      config.tokenFn,
    );
    const effectiveLookup = lookup ?? { dataSetId: def.uuid, operations: [] };
    const dataset = await client.query(effectiveLookup);
    ctx.manager.apply(def.uuid, { type: "snapshot", dataset });
    return { dataset, inferredColumns: false, source: "serverQuery" };
  }

  // ---- Existing code (unchanged) ----
  validate(def);
  // ... rest of the function
```

Note: The `"CONFIG_MISSING"` error code is cast to `any` because `DataSetErrorCode` may not include it. Add `"CONFIG_MISSING"` to the error code union in `errors.ts`:

```typescript
export type DataSetErrorCode =
  | "INVALID_DEFINITION"
  | "FETCH_FAILED"
  // ... existing codes ...
  | "CONFIG_MISSING";
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/external/resolver.test.ts`
Expected: All tests PASS (existing + 3 new)

- [ ] **Step 5: Run full pages-data test suite**

Run: `yarn workspace @casehub/pages-data run test`
Expected: All tests PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git add packages/pages-data/src/dataset/external/resolver.ts packages/pages-data/src/dataset/external/resolver.test.ts packages/pages-data/src/dataset/errors.ts
git commit -m "feat(pages-data): add serverQuery resolver route with CONFIG_MISSING guard

Refs #21"
```

---

### Task 4: Data pipeline — operation separation and refresh

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts`
- Test: `packages/pages-runtime/src/data-pipeline.test.ts` (create if absent, or add to existing)

**Consumes:** `resolveExternalDataSet` updated signature from Task 3, `ExternalDataSetDef.serverQuery` from Task 1

**Produces:** Data pipeline that tracks server-query datasets, strips YAML ops in `pushData`, and supports server-query refresh

The data pipeline changes are concentrated in `data-pipeline.ts`. Four modifications:

1. **Server-query tracking Set** — populated at resolution time
2. **pushData operation separation** — both the standard path and the expandable bypass
3. **Stored lookups for refresh** — `scheduleRefresh` gains a server-query path
4. **Resolution call site** — passes lookup to resolver

- [ ] **Step 1: Add server-query tracking Set and stored lookups**

Near the top of `createDataPipeline`, after the existing declarations, add:

```typescript
const serverQueryDatasets = new Set<DataSetId>();
const serverQueryLookups = new Map<DataSetId, DataSetLookup>();
```

- [ ] **Step 2: Update resolution call site to pass lookup**

In `handleDataRequest`, update the resolution call (around line 505) from:

```typescript
pending = resolveExternalDataSet(def, resolverCtx);
```

to:

```typescript
if (def.serverQuery) {
  serverQueryDatasets.add(lookup.dataSetId);
  serverQueryLookups.set(lookup.dataSetId, lookup);
}
pending = resolveExternalDataSet(def, resolverCtx, def.serverQuery ? lookup : undefined);
```

- [ ] **Step 3: Update pushData — expandable bypass**

In `pushData`, update the expandable bypass block (around line 198-211). Change:

```typescript
if (expandable) {
  const result = manager.lookup(lookup, options);
```

to:

```typescript
if (expandable) {
  const effectiveLookup = serverQueryDatasets.has(lookup.dataSetId)
    ? { ...lookup, operations: [] as readonly DataSetOp[] }
    : lookup;
  const result = manager.lookup(effectiveLookup, options);
```

Add the import for `DataSetOp` at the top if not already present:

```typescript
import type { DataSetOp } from "@casehubio/pages-data/dist/dataset/ops.js";
```

- [ ] **Step 4: Update pushData — standard sort/ops logic**

In `pushData`, update the standard path sort logic (around lines 232-243). Change:

```typescript
let sortOps: readonly DataSetOp[] = lookup.operations.filter((op) => op.type !== "sort");

if (compState?.sort) {
  sortOps = [...sortOps, { type: "sort" as const, columns: [compState.sort] }];
  target.activeSort = compState.sort;
} else {
  // Preserve original lookup sort
  sortOps = lookup.operations;
  target.activeSort = undefined;
}
```

to:

```typescript
const isServerQuery = serverQueryDatasets.has(lookup.dataSetId);
let sortOps: readonly DataSetOp[] = isServerQuery
  ? []
  : lookup.operations.filter((op) => op.type !== "sort");

if (compState?.sort) {
  sortOps = [...sortOps, { type: "sort" as const, columns: [compState.sort] }];
  target.activeSort = compState.sort;
} else if (!isServerQuery) {
  sortOps = lookup.operations;
  target.activeSort = undefined;
} else {
  target.activeSort = undefined;
}
```

- [ ] **Step 5: Update scheduleRefresh — add server-query path**

In `scheduleRefresh`, add a new path at the top of the function body, before the push-source guard:

```typescript
function scheduleRefresh(def: ExternalDataSetDef, dataSetId: DataSetId): void {
  if (!def.refreshTime || refreshTimers.has(dataSetId)) return;

  // Server-query refresh: re-send the stored lookup to the backend
  if (def.serverQuery) {
    const interval = parseRefreshTime(def.refreshTime);
    const timerId = setInterval(() => {
      if (!resolverCtx) return;
      const storedLookup = serverQueryLookups.get(dataSetId);
      if (!storedLookup) return;
      resolveExternalDataSet(def, resolverCtx, storedLookup)
        .then(() => {
          for (const [compId, entry] of registry) {
            if (entry.originalLookup?.dataSetId === dataSetId && entry.vizElement) {
              const filterGroup = (entry.component.props as Record<string, unknown> | undefined)
                ?.filter as { group?: string } | undefined;
              pushData(entry.vizElement, entry.originalLookup, entry.pagePath, filterGroup?.group, compId);
            }
          }
        })
        .catch(() => {});
    }, interval);
    refreshTimers.set(dataSetId, timerId);
    return;
  }

  // Guard: Push source datasets use server push, not polling
  // ... existing code continues ...
```

- [ ] **Step 6: Update import of resolveExternalDataSet**

At the top of `data-pipeline.ts`, the import is already present. No change needed — the resolver's new optional parameters are backward-compatible.

- [ ] **Step 7: Clean up dispose**

In the `dispose()` method, add cleanup for the new collections:

```typescript
serverQueryDatasets.clear();
serverQueryLookups.clear();
```

- [ ] **Step 8: Run full test suite**

Run: `yarn workspace @casehub/pages-data run test && yarn workspace @casehub/pages-runtime run test`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-runtime/src/data-pipeline.ts
git commit -m "feat(pages-runtime): server-query operation separation in data pipeline

Track server-query datasets, strip YAML ops in pushData (standard + expandable
bypass), add scheduleRefresh server-query path with stored lookups.

Refs #21"
```

---

### Task 5: Site wiring and PLATFORM.md update

**Files:**
- Modify: `packages/pages-runtime/src/site.ts`
- Modify: `../../casehub-parent/docs/PLATFORM.md` (external repo)

**Consumes:** `DataProviderConfig.serverQuery` from Task 1, `createDevAuthTokenFn` from `./dev-auth.js`

- [ ] **Step 1: Wire serverQuery config in loadSite**

In `site.ts`, add the import for `createDevAuthTokenFn` if not already present:

```typescript
import { createDevAuthTokenFn } from "./dev-auth.js";
```

In the `pipeline.setResolverCtx` call (around line 155-160), update `providerConfig` to include `serverQuery` when `options?.providerConfig?.serverQuery` is present:

```typescript
pipeline.setResolverCtx({
  manager,
  providerFactory: createDataProviderFactory(options?.fetch ?? globalThis.fetch.bind(globalThis), options?.baseUrl),
  providerConfig: {
    ...options?.providerConfig,
    ...(options?.providerConfig?.serverQuery ? {
      serverQuery: {
        ...options.providerConfig.serverQuery,
        tokenFn: options.providerConfig.serverQuery.tokenFn ?? createDevAuthTokenFn(),
      },
    } : {}),
  },
  presetRegistry: createPresetRegistry(),
});
```

This ensures that if a `serverQuery` config is provided but `tokenFn` is omitted, it defaults to the dev-auth token function.

- [ ] **Step 2: Update PLATFORM.md**

In `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`, find the casehub-pages capability entry and update the `data` description from `data` (scaffold) to the actual module description. The relevant text to update is in the capability ownership table row for "Web application framework (pages)".

Replace `data` (scaffold)` with:

`data` (DataProvider SPI + REST relay at `/api/dataset/fetch` + SQL push-down at `/api/dataset/query`, `@DefaultBean` no-op, no JDBC), `data-sql` (SQL provider via Quarkus Agroal named datasources, `SqlQueryBuilder` + `ResultSetMapper` with push-down filter/group/sort/aggregation, `@ApplicationScoped`).

- [ ] **Step 3: Run full build**

Run: `yarn build:packages`
Expected: All packages build successfully

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehub/pages-data run test && yarn workspace @casehub/pages-runtime run test`
Expected: All tests PASS

- [ ] **Step 5: Commit site.ts**

```bash
git add packages/pages-runtime/src/site.ts
git commit -m "feat(pages-runtime): wire serverQuery config with dev-auth tokenFn

Refs #21"
```

- [ ] **Step 6: Commit PLATFORM.md (separate repo)**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: update casehub-pages data module description — no longer scaffold

Refs casehubio/casehub-pages#21"
```

---

## Verification

After all tasks are complete:

1. `yarn workspace @casehub/pages-data run test` — all pages-data tests pass
2. `yarn workspace @casehub/pages-runtime run test` — all pages-runtime tests pass (if test suite exists)
3. `yarn build:packages` — packages build cleanly
4. `yarn typecheck` — no type errors
5. `mvn -f backend/pom.xml test` — backend tests still pass (no backend changes in this plan)
