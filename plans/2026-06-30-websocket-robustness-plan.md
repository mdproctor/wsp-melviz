# WebSocket Robustness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix two WebSocket bugs (#61, #62) and add three features: incremental reconnect (#56), query-param auth (#58), and transparent relay proxy (#57).

**Architecture:** Extend `DataProviderConfig` with a `webSocket` section for auth and relay config. The pool receives config lazily via `configure()` and passes it to sources on creation. Sources build final connection URLs from base URL + relay rewrite + auth token. `seq` tracking in the source enables incremental reconnect. Bug fixes are localized to `processMessage()` (fallback routing), `subscribe()` (duplicate guard), and `apply()` (column validation).

**Tech Stack:** TypeScript 5, Vitest, URL API

**Spec:** `docs/superpowers/specs/2026-06-29-websocket-robustness-design.md`

## Global Constraints

- All tests use Vitest (`vitest run` in `packages/pages-data`)
- Branded types: use `dataSetId("x")` and `columnId("x")` constructors — never cast raw strings
- `WireMessage` is a module-private interface — not exported
- WebSocket tests use a `MockWebSocket` class (see existing tests for shape)
- Run `yarn workspace @casehub/pages-data run test` for unit tests
- Run `yarn typecheck` for cross-package type checking after interface changes

---

### Task 1: DataProviderConfig Extension + Exports

**Files:**
- Modify: `packages/pages-data/src/dataset/external/types.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `WebSocketAuthConfig` interface, `DataProviderConfig.webSocket` field — consumed by Tasks 3, 6, 7

- [ ] **Step 1: Add `WebSocketAuthConfig` and extend `DataProviderConfig` in types.ts**

Add after the existing `DataProviderConfig` interface (around line 88):

```typescript
export interface WebSocketAuthConfig {
  readonly type: "query-param";
  readonly paramName?: string;
  readonly token: string;
}
```

Then add the `webSocket` field to the existing `DataProviderConfig` interface:

```typescript
export interface DataProviderConfig {
  readonly defaultProvider?: "browser" | "server-relay";
  readonly corsProxy?: {
    readonly url: string;
    readonly enabled: boolean;
  };
  readonly serverRelay?: {
    readonly endpoint: string;
  };
  readonly webSocket?: {
    readonly relay?: { readonly endpoint: string };
    readonly auth?: WebSocketAuthConfig;
  };
}
```

- [ ] **Step 2: Add exports in index.ts**

Add to the existing re-exports section at the top of `packages/pages-data/src/dataset/external/index.ts`:

```typescript
export type { WebSocketAuthConfig } from "./types.js";
```

Also add the `WebSocketSourceConfig` export (the type will be created in Task 3, but the export line can be added now — TypeScript will error until Task 3 creates it):

Actually — defer this export to Task 3 when the type exists. For now, only export `WebSocketAuthConfig`.

- [ ] **Step 3: Run typecheck to verify no breakage**

Run: `yarn typecheck`
Expected: PASS — the new interface is additive, no existing code breaks.

- [ ] **Step 4: Commit**

```
feat: add WebSocketAuthConfig and webSocket config to DataProviderConfig

Extends DataProviderConfig with a webSocket section for auth and relay
deployment config.  Refs #58
```

---

### Task 2: Append Column-Count Validation (#62b)

**Files:**
- Modify: `packages/pages-data/src/dataset/manager.ts:84-94`
- Test: `packages/pages-data/src/dataset/manager.test.ts`

**Interfaces:**
- Consumes: `DataSetManager.apply()`, `createTypedRow()`, `toTypedDataSet()`
- Produces: no interface changes — behavioral fix only

- [ ] **Step 1: Write failing test — append with mismatched column count is rejected**

Add to `packages/pages-data/src/dataset/manager.test.ts` inside a new describe block:

```typescript
describe("DataSetManager — append validation", () => {
  it("rejects append when row cell count does not match column count", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });

    // Create a row with 3 cells instead of 2
    const badRow = toTypedDataSet({
      columns: [
        col("name", "Name", ColumnType.LABEL),
        col("amount", "Amount", ColumnType.NUMBER),
        col("extra", "Extra", ColumnType.TEXT),
      ],
      data: [["Bob", "200", "surplus"]],
    });

    mgr.apply(ID_A, { type: "append", rows: badRow.rows });

    // Dataset should still have only the original row
    const result = mgr.get(ID_A);
    expect(result?.rows).toHaveLength(1);
  });

  it("accepts append when row cell count matches column count", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });

    const goodRow = testDataSet([["Bob", "200"]]);
    mgr.apply(ID_A, { type: "append", rows: goodRow.rows });

    const result = mgr.get(ID_A);
    expect(result?.rows).toHaveLength(2);
  });

  it("rejects entire append if any row has wrong cell count", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });

    const goodRow = testDataSet([["Bob", "200"]]).rows[0]!;
    const badRow = toTypedDataSet({
      columns: [
        col("name", "Name", ColumnType.LABEL),
        col("amount", "Amount", ColumnType.NUMBER),
        col("extra", "Extra", ColumnType.TEXT),
      ],
      data: [["Carol", "300", "surplus"]],
    }).rows[0]!;

    mgr.apply(ID_A, { type: "append", rows: [goodRow, badRow] });

    // Neither row should be appended — reject-all semantics
    const result = mgr.get(ID_A);
    expect(result?.rows).toHaveLength(1);
  });

  it("does not fire onChanged when append is rejected", () => {
    const onChange = vi.fn();
    const mgr = createDataSetManager({ onChanged: onChange });
    const ds = testDataSet([["Alice", "100"]]);
    mgr.apply(ID_A, { type: "snapshot", dataset: ds });
    onChange.mockClear();

    const badRow = toTypedDataSet({
      columns: [
        col("name", "Name", ColumnType.LABEL),
        col("amount", "Amount", ColumnType.NUMBER),
        col("extra", "Extra", ColumnType.TEXT),
      ],
      data: [["Bob", "200", "surplus"]],
    });

    mgr.apply(ID_A, { type: "append", rows: badRow.rows });
    expect(onChange).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose manager.test`
Expected: 2 FAIL (reject tests), 1 PASS (accept test), 1 FAIL (onChanged test)

- [ ] **Step 3: Implement column-count validation in manager.ts**

In `packages/pages-data/src/dataset/manager.ts`, replace the `case "append"` block (lines 84-94):

```typescript
case "append": {
  const existing = this.datasets.get(id);
  if (existing === undefined) return;
  const colCount = existing.columns.length;
  if (event.rows.some(row => row.cells.length !== colCount)) {
    console.warn(
      `[DataSetManager] append rejected: row cell count mismatch (expected ${String(colCount)})`,
    );
    return;
  }
  const combined = [...existing.rows, ...event.rows];
  const trimmed = event.maxRows !== undefined && event.maxRows >= 0
    ? combined.slice(-event.maxRows)
    : combined;
  const result: TypedDataSet = { columns: existing.columns, rows: trimmed };
  this.datasets.set(id, result);
  this.options?.onChanged?.(id, result);
  break;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose manager.test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
fix: reject append events with column-count mismatch

Append events where any row has a different cell count than the existing
dataset's column schema are now rejected entirely. Partial application
of a corrupt event creates worse inconsistency than rejecting it.

Closes #62
```

Note: #62 has two sub-issues. This commit addresses 62b. Task 4 addresses 62a. The `Closes` will apply when both are done on the branch.

---

### Task 3: WebSocketSource Config, buildConnectionUrl, and Signature Migration (#57, #58)

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts`

**Interfaces:**
- Consumes: `WebSocketAuthConfig` from Task 1
- Produces: `WebSocketSourceConfig` interface, `createWebSocketSource(baseUrl, config?, WSConstructor?)` — consumed by Tasks 4, 5, 6

- [ ] **Step 1: Write failing tests for auth URL, relay URL, and combined**

Add to `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`:

```typescript
describe("WebSocketSource — connection URL", () => {
  it("appends auth token as query parameter", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      { auth: { type: "query-param", token: "secret123" } },
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    const url = new URL(ws.url);
    expect(url.searchParams.get("token")).toBe("secret123");
  });

  it("uses custom auth param name", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      { auth: { type: "query-param", paramName: "api_key", token: "key456" } },
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    const url = new URL(ws.url);
    expect(url.searchParams.get("api_key")).toBe("key456");
    expect(url.searchParams.has("token")).toBe(false);
  });

  it("rewrites URL through relay endpoint", async () => {
    const source = createWebSocketSource(
      "ws://upstream:8080/ws",
      { relay: { endpoint: "wss://relay.example.com/ws-relay" } },
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://upstream:8080/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    const url = new URL(ws.url);
    expect(url.hostname).toBe("relay.example.com");
    expect(url.pathname).toBe("/ws-relay");
    expect(url.searchParams.get("target")).toBe("ws://upstream:8080/ws");
  });

  it("applies auth to relay URL when both configured", async () => {
    const source = createWebSocketSource(
      "ws://upstream:8080/ws",
      {
        relay: { endpoint: "wss://relay.example.com/ws-relay" },
        auth: { type: "query-param", token: "relay-token" },
      },
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://upstream:8080/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    const url = new URL(ws.url);
    expect(url.hostname).toBe("relay.example.com");
    expect(url.searchParams.get("target")).toBe("ws://upstream:8080/ws");
    expect(url.searchParams.get("token")).toBe("relay-token");
  });

  it("connects to baseUrl directly when no config", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    expect(MockWebSocket.instances[0]!.url).toBe("ws://localhost/ws");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: FAIL — `createWebSocketSource` doesn't accept config parameter yet; existing tests also fail because MockWebSocket is now in the wrong argument position.

- [ ] **Step 3: Implement WebSocketSourceConfig, buildConnectionUrl, and signature change**

In `packages/pages-data/src/dataset/external/sources/websocket-source.ts`:

Add import for `WebSocketAuthConfig`:
```typescript
import type { WebSocketAuthConfig } from "../types.js";
```

Add the config interface after the existing `WebSocketSource` interface:
```typescript
export interface WebSocketSourceConfig {
  readonly relay?: { readonly endpoint: string };
  readonly auth?: WebSocketAuthConfig;
}
```

Change the function signature:
```typescript
export function createWebSocketSource(
  baseUrl: string,
  config?: WebSocketSourceConfig,
  WSConstructor: typeof WebSocket = WebSocket,
): WebSocketSource {
```

Add `buildConnectionUrl` as the first function inside the closure:
```typescript
function buildConnectionUrl(): string {
  let url = new URL(baseUrl);

  if (config?.relay) {
    url = new URL(config.relay.endpoint);
    url.searchParams.set("target", baseUrl);
  }

  if (config?.auth?.type === "query-param") {
    url.searchParams.set(config.auth.paramName ?? "token", config.auth.token);
  }

  return url.toString();
}
```

Change the `connect()` function to use `buildConnectionUrl()`:
```typescript
ws = new WSConstructor(buildConnectionUrl());
```
(was: `ws = new WSConstructor(baseUrl);`)

- [ ] **Step 4: Migrate all existing test callsites**

In `websocket-source.test.ts`, update every `createWebSocketSource` call that passes `MockWebSocket` as the second argument. Change:
```typescript
createWebSocketSource("ws://localhost/ws", MockWebSocket as unknown as typeof WebSocket)
```
to:
```typescript
createWebSocketSource("ws://localhost/ws", undefined, MockWebSocket as unknown as typeof WebSocket)
```

There are 14 existing callsites to update. Search for `createWebSocketSource(` and add `undefined,` before `MockWebSocket` in each.

- [ ] **Step 5: Add export to index.ts**

In `packages/pages-data/src/dataset/external/index.ts`, add:
```typescript
export type { WebSocketSourceConfig } from "./sources/websocket-source.js";
```

- [ ] **Step 6: Run all tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: ALL PASS (14 migrated tests + 5 new tests)

- [ ] **Step 7: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 8: Commit**

```
feat: WebSocketSource config — auth query-param and relay URL rewriting

Adds WebSocketSourceConfig with relay and auth support. Uses URL API for
parameter handling. Signature change: config is the second parameter,
WSConstructor moves to third.

Refs #57 Refs #58
```

---

### Task 4: Bug Fixes — Single-Subscriber Fallback + Duplicate Subscribe Guard (#61, #62a)

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`

**Interfaces:**
- Consumes: `createWebSocketSource` from Task 3 (new signature)
- Produces: no interface changes — behavioral fixes only

- [ ] **Step 1: Write failing test — messages without dataset field route to sole subscriber**

Add to `websocket-source.test.ts`:

```typescript
describe("WebSocketSource — single-subscriber fallback (#61)", () => {
  it("routes message without dataset field to sole subscriber", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const events: DataSetEvent[] = [];
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, (e) => events.push(e));

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    ws.open();

    ws.onmessage?.({
      data: JSON.stringify({
        type: "snapshot",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["hello"]],
      }),
    });

    expect(events).toHaveLength(1);
    expect(events[0]!.type).toBe("snapshot");
  });

  it("drops message without dataset field when multiple subscribers exist", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const events: DataSetEvent[] = [];
    const def1: ExternalDataSetDef = {
      uuid: dataSetId("d1"),
      url: "ws://localhost/ws?dataset=a",
    };
    const def2: ExternalDataSetDef = {
      uuid: dataSetId("d2"),
      url: "ws://localhost/ws?dataset=b",
    };

    source.subscribe(dataSetId("d1"), def1, (e) => events.push(e));
    source.subscribe(dataSetId("d2"), def2, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    ws.open();

    ws.onmessage?.({
      data: JSON.stringify({
        type: "snapshot",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["hello"]],
      }),
    });

    expect(events).toHaveLength(0);
  });

  it("does not fallback when dataset field is present but unrecognized", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const events: DataSetEvent[] = [];
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, (e) => events.push(e));

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    ws.open();

    ws.onmessage?.({
      data: JSON.stringify({
        dataset: "unknown",
        type: "snapshot",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["hello"]],
      }),
    });

    expect(events).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: First test FAIL (message dropped), second PASS (already drops), third PASS (already drops)

- [ ] **Step 3: Implement single-subscriber fallback in processMessage**

In `websocket-source.ts`, replace the existing `processMessage` function's opening lines:

```typescript
function processMessage(msg: WireMessage): void {
  const wireName = msg.dataset;
  let dataSetId = wireName !== undefined ? wireNameToId.get(wireName) : undefined;

  if (dataSetId === undefined) {
    if (wireName === undefined && subscriptions.size === 1) {
      dataSetId = subscriptions.keys().next().value;
    } else {
      return;
    }
  }

  const subscription = subscriptions.get(dataSetId);
  if (!subscription) {
    return;
  }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: ALL PASS

- [ ] **Step 5: Write failing test — duplicate subscribe sends no extra upstream message**

Add to `websocket-source.test.ts`:

```typescript
describe("WebSocketSource — duplicate subscribe guard (#62a)", () => {
  it("does not send duplicate subscribe message for same dataSetId", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    ws.open();

    const subscribeCount = ws.sent.filter(
      (m) => JSON.parse(m).type === "subscribe",
    ).length;
    expect(subscribeCount).toBe(1);

    // Second subscribe for same ID — should be ignored
    source.subscribe(dataSetId("chat"), def, vi.fn());
    const afterCount = ws.sent.filter(
      (m) => JSON.parse(m).type === "subscribe",
    ).length;
    expect(afterCount).toBe(1);
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: FAIL — second subscribe sends duplicate message (afterCount === 2)

- [ ] **Step 7: Implement duplicate subscribe guard**

In `websocket-source.ts`, add early-return at the top of the `subscribe` method:

```typescript
subscribe(dataSetId: DataSetId, def: ExternalDataSetDef, listener: DataSetEventListener): void {
  if (subscriptions.has(dataSetId)) return;
  // ... existing code
```

- [ ] **Step 8: Run all tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
fix: single-subscriber fallback routing and duplicate subscribe guard

Messages without a dataset field now route to the sole subscriber in
non-multiplexed mode. Messages with an unrecognized dataset field are
still silently dropped (unsubscribed broadcast).

Duplicate subscribe calls for the same dataSetId are now ignored,
preventing duplicate upstream subscribe messages.

Closes #61 Refs #62
```

---

### Task 5: Incremental Reconnect via seq (#56)

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`

**Interfaces:**
- Consumes: `createWebSocketSource` from Task 3
- Produces: no interface changes — behavioral addition only

- [ ] **Step 1: Write failing tests for seq tracking and since in reconnect**

Add to `websocket-source.test.ts`:

```typescript
describe("WebSocketSource — incremental reconnect (#56)", () => {
  it("includes since in subscribe message on reconnect when seq was received", async () => {
    vi.useFakeTimers();
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws1 = MockWebSocket.instances[0]!;
    ws1.open();

    // Receive a message with seq
    ws1.onmessage?.({
      data: JSON.stringify({
        dataset: "messages",
        type: "snapshot",
        seq: "42",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["hello"]],
      }),
    });

    // Simulate abnormal close and reconnect
    ws1.readyState = MockWebSocket.CLOSED;
    ws1.onclose?.({ code: 1006, reason: "abnormal" });
    await vi.advanceTimersByTimeAsync(1000);

    expect(MockWebSocket.instances).toHaveLength(2);
    const ws2 = MockWebSocket.instances[1]!;
    ws2.open();

    // Check that resubscribe includes since
    const resubscribe = ws2.sent.find((m) => JSON.parse(m).type === "subscribe");
    expect(resubscribe).toBeDefined();
    const parsed = JSON.parse(resubscribe!);
    expect(parsed.since).toBe("42");

    vi.useRealTimers();
  });

  it("does not include since on first connection (no prior seq)", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    ws.open();

    const subscribeMsg = ws.sent.find((m) => JSON.parse(m).type === "subscribe");
    const parsed = JSON.parse(subscribeMsg!);
    expect(parsed.since).toBeUndefined();
  });

  it("tracks seq across multiple events, uses latest", async () => {
    vi.useFakeTimers();
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws1 = MockWebSocket.instances[0]!;
    ws1.open();

    ws1.onmessage?.({
      data: JSON.stringify({
        dataset: "messages",
        type: "snapshot",
        seq: "10",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["first"]],
      }),
    });

    ws1.onmessage?.({
      data: JSON.stringify({
        dataset: "messages",
        type: "append",
        seq: "15",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["second"]],
      }),
    });

    ws1.readyState = MockWebSocket.CLOSED;
    ws1.onclose?.({ code: 1006, reason: "abnormal" });
    await vi.advanceTimersByTimeAsync(1000);

    const ws2 = MockWebSocket.instances[1]!;
    ws2.open();

    const resubscribe = ws2.sent.find((m) => JSON.parse(m).type === "subscribe");
    expect(JSON.parse(resubscribe!).since).toBe("15");

    vi.useRealTimers();
  });

  it("does not advance seq for events without seq field", async () => {
    vi.useFakeTimers();
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws1 = MockWebSocket.instances[0]!;
    ws1.open();

    // First event with seq
    ws1.onmessage?.({
      data: JSON.stringify({
        dataset: "messages",
        type: "snapshot",
        seq: "10",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["first"]],
      }),
    });

    // Second event without seq — lastSeq should stay at "10"
    ws1.onmessage?.({
      data: JSON.stringify({
        dataset: "messages",
        type: "append",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["second"]],
      }),
    });

    ws1.readyState = MockWebSocket.CLOSED;
    ws1.onclose?.({ code: 1006, reason: "abnormal" });
    await vi.advanceTimersByTimeAsync(1000);

    const ws2 = MockWebSocket.instances[1]!;
    ws2.open();

    const resubscribe = ws2.sent.find((m) => JSON.parse(m).type === "subscribe");
    expect(JSON.parse(resubscribe!).since).toBe("10");

    vi.useRealTimers();
  });

  it("resets seq on explicit close()", async () => {
    const source = createWebSocketSource(
      "ws://localhost/ws",
      undefined,
      MockWebSocket as unknown as typeof WebSocket,
    );
    const def: ExternalDataSetDef = {
      uuid: dataSetId("chat"),
      url: "ws://localhost/ws?dataset=messages",
    };

    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(1));
    const ws = MockWebSocket.instances[0]!;
    ws.open();

    ws.onmessage?.({
      data: JSON.stringify({
        dataset: "messages",
        type: "snapshot",
        seq: "42",
        columns: [{ id: "text", type: "TEXT" }],
        rows: [["hello"]],
      }),
    });

    source.close();

    // Re-subscribe after close — should not have since
    source.subscribe(dataSetId("chat"), def, vi.fn());

    await vi.waitFor(() => expect(MockWebSocket.instances).toHaveLength(2));
    const ws2 = MockWebSocket.instances[1]!;
    ws2.open();

    const subscribeMsg = ws2.sent.find((m) => JSON.parse(m).type === "subscribe");
    expect(JSON.parse(subscribeMsg!).since).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: FAIL — no `since` field in subscribe messages, no `seq` field on WireMessage

- [ ] **Step 3: Implement seq tracking**

In `websocket-source.ts`:

Add `seq` to `WireMessage`:
```typescript
interface WireMessage {
  dataset?: string;
  type?: string;
  seq?: string;
  columns?: Column[];
  rows?: (string | null)[][];
  row?: (string | null)[];
  key?: string;
}
```

Add state variable after `reconnectTimer`:
```typescript
let lastSeq: string | undefined;
```

In `ws.onopen`, modify the resubscribe loop to include `since`:
```typescript
ws.onopen = () => {
  reconnectAttempt = 0;
  for (const [id] of subscriptions) {
    const wireName = idToWireName.get(id);
    if (wireName) {
      const msg: Record<string, string> = { type: "subscribe", dataset: wireName };
      if (lastSeq !== undefined) {
        msg.since = lastSeq;
      }
      ws?.send(JSON.stringify(msg));
    }
  }
};
```

In `processMessage`, add `lastSeq` update inside each case branch, after the `subscription.listener()` call. For example in `case "snapshot"`:
```typescript
case "snapshot": {
  if (!msg.columns || !msg.rows) {
    console.warn("[WebSocketSource] snapshot event missing columns or rows:", msg);
    return;
  }
  const dataset = toTypedDataSet({ columns: msg.columns, data: msg.rows });
  subscription.listener({ type: "snapshot", dataset });
  if (msg.seq !== undefined) lastSeq = msg.seq;
  break;
}
```

Add the same `if (msg.seq !== undefined) lastSeq = msg.seq;` line after the `subscription.listener()` call in each of the `append`, `replace`, and `remove` cases.

In `close()`, reset `lastSeq`:
```typescript
close(): void {
  if (reconnectTimer) {
    clearTimeout(reconnectTimer);
    reconnectTimer = null;
  }
  if (ws) {
    ws.close();
    ws = null;
  }
  subscriptions.clear();
  wireNameToId.clear();
  idToWireName.clear();
  lastSeq = undefined;
},
```

- [ ] **Step 4: Run all tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-source.test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat: incremental reconnect via seq tracking and since parameter

The WebSocket source now tracks server-provided seq values from wire
messages. On reconnect, subscribe messages include since=<lastSeq>
so servers can replay only events after the client's last position.

Servers that don't send seq get full-snapshot reconnect (unchanged).

Refs #56
```

---

### Task 6: WebSocketPool Lazy Configuration

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-pool.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-pool.test.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts`

**Interfaces:**
- Consumes: `createWebSocketSource` from Task 3, `WebSocketSourceConfig` from Task 3
- Produces: `WebSocketPool.configure(config)` method, `acquire(baseUrl)` (no longer takes `def`) — consumed by Task 7

- [ ] **Step 1: Write failing tests for pool configure and propagation**

Add to `websocket-pool.test.ts`:

```typescript
import type { WebSocketSourceConfig } from "./websocket-source.js";

describe("WebSocketPool — configuration", () => {
  it("passes config to created sources via configure()", () => {
    const constructedUrls: string[] = [];

    class ConfigTrackingWebSocket {
      static CONNECTING = 0;
      static OPEN = 1;
      static CLOSING = 2;
      static CLOSED = 3;
      readyState = 0;
      onopen: ((this: WebSocket, ev: Event) => unknown) | null = null;
      onmessage: ((this: WebSocket, ev: MessageEvent) => unknown) | null = null;
      onclose: ((this: WebSocket, ev: CloseEvent) => unknown) | null = null;
      onerror: ((this: WebSocket, ev: Event) => unknown) | null = null;
      send = vi.fn();
      close = vi.fn();

      constructor(public url: string) {
        constructedUrls.push(url);
      }
    }

    const pool = createWebSocketPool(
      ConfigTrackingWebSocket as unknown as typeof WebSocket,
    );
    const config: WebSocketSourceConfig = {
      auth: { type: "query-param", token: "pooltest" },
    };
    pool.configure(config);

    const source = pool.acquire("ws://host/ws");
    source.subscribe(dataSetId("d1"), { uuid: dataSetId("d1") }, () => {});

    expect(constructedUrls).toHaveLength(1);
    const url = new URL(constructedUrls[0]!);
    expect(url.searchParams.get("token")).toBe("pooltest");
  });

  it("acquire without configure creates source with no config", () => {
    const constructedUrls: string[] = [];

    class TrackingWebSocket {
      static CONNECTING = 0;
      static OPEN = 1;
      static CLOSING = 2;
      static CLOSED = 3;
      readyState = 0;
      onopen: ((this: WebSocket, ev: Event) => unknown) | null = null;
      onmessage: ((this: WebSocket, ev: MessageEvent) => unknown) | null = null;
      onclose: ((this: WebSocket, ev: CloseEvent) => unknown) | null = null;
      onerror: ((this: WebSocket, ev: Event) => unknown) | null = null;
      send = vi.fn();
      close = vi.fn();

      constructor(public url: string) {
        constructedUrls.push(url);
      }
    }

    const pool = createWebSocketPool(
      TrackingWebSocket as unknown as typeof WebSocket,
    );

    const source = pool.acquire("ws://host/ws");
    source.subscribe(dataSetId("d1"), { uuid: dataSetId("d1") }, () => {});

    expect(constructedUrls).toHaveLength(1);
    expect(constructedUrls[0]).toBe("ws://host/ws");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-pool.test`
Expected: FAIL — `pool.configure` is not a function, `acquire` still takes two params

- [ ] **Step 3: Implement pool with configure() and simplified acquire()**

Replace `packages/pages-data/src/dataset/external/sources/websocket-pool.ts`:

```typescript
import type { WebSocketSourceConfig } from "./websocket-source.js";
import { createWebSocketSource, type WebSocketSource } from "./websocket-source.js";

export interface WebSocketPool {
  configure(config: WebSocketSourceConfig): void;
  acquire(baseUrl: string): WebSocketSource;
  releaseAll(): void;
}

export function createWebSocketPool(
  WS: typeof WebSocket = WebSocket,
): WebSocketPool {
  const sources = new Map<string, WebSocketSource>();
  let config: WebSocketSourceConfig | undefined;

  return {
    configure(cfg: WebSocketSourceConfig): void {
      config = cfg;
    },

    acquire(baseUrl: string): WebSocketSource {
      let source = sources.get(baseUrl);
      if (source === undefined) {
        source = createWebSocketSource(baseUrl, config, WS);
        sources.set(baseUrl, source);
      }
      return source;
    },

    releaseAll(): void {
      for (const source of sources.values()) {
        source.close();
      }
      sources.clear();
    },
  };
}
```

- [ ] **Step 4: Fix existing pool tests — remove `def` from acquire() calls**

In `websocket-pool.test.ts`, update existing tests. The `acquire()` call signature changes from `acquire(url, def)` to `acquire(url)`. Update each:

```typescript
// Before:
const s1 = pool.acquire("ws://host/ws", def1);
// After:
const s1 = pool.acquire("ws://host/ws");
```

- [ ] **Step 5: Run all pool tests to verify they pass**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose websocket-pool.test`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat: WebSocketPool lazy configuration via configure()

The pool accepts config lazily — created eagerly at pipeline
construction, configured when DataProviderConfig arrives via
setResolverCtx. acquire() no longer takes def (was unused).

Refs #57 Refs #58
```

---

### Task 7: Pipeline Integration

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts`

**Interfaces:**
- Consumes: `WebSocketPool.configure()` from Task 6, `WebSocketPool.acquire(baseUrl)` from Task 6
- Produces: fully wired pipeline — no new interfaces

- [ ] **Step 1: Update setResolverCtx to configure the pool**

In `packages/pages-runtime/src/data-pipeline.ts`, modify the `setResolverCtx` method:

```typescript
setResolverCtx(ctx: ResolverContext): void {
  resolverCtx = ctx;
  if (ctx.providerConfig.webSocket) {
    pool.configure(ctx.providerConfig.webSocket);
  }
},
```

- [ ] **Step 2: Simplify the acquire() call in handleDataRequest**

In `handleDataRequest`, change the WebSocket source routing block (around line 237):

```typescript
const source = pool.acquire(baseUrl.toString());
```

(was: `const source = pool.acquire(baseUrl.toString(), def);`)

- [ ] **Step 3: Run typecheck to verify cross-package types**

Run: `yarn typecheck`
Expected: PASS — `pool.configure()` and `pool.acquire()` signatures match

- [ ] **Step 4: Run full test suite**

Run: `yarn workspace @casehub/pages-data run test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat: wire WebSocket config through pipeline to pool

setResolverCtx passes providerConfig.webSocket to the pool via
configure(). acquire() call simplified to single baseUrl argument.

Closes #56 Closes #57 Closes #58
```

---

## Implementation Order Summary

```
Task 1: Types (no deps)
Task 2: Manager append validation (#62b — independent)
Task 3: WebSocketSource config + buildConnectionUrl (#57, #58 — depends on Task 1)
Task 4: Bug fixes (#61, #62a — depends on Task 3 for new signature)
Task 5: Seq tracking (#56 — depends on Task 3 for new signature)
Task 6: Pool config (depends on Task 3 for WebSocketSourceConfig)
Task 7: Pipeline integration (depends on Task 6)
```

Tasks 1 and 2 are independent and can run in parallel. Tasks 3-5 are sequential (same file). Task 6 depends on Task 3. Task 7 depends on Task 6.
