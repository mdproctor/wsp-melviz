# mutableRestSource — Production Write Path for DataAction Dispatch

**Issue:** casehubio/casehub-pages#144
**Date:** 2026-07-25
**Status:** Approved

## Problem

The DataSource spec defines `MutableDataSource` with `dispatch(action: DataAction)`, but only `simulated()` implements it — for demo/test mode only. Production components handle mutations via direct `fetch()` calls. There is no pipeline-integrated write path for production REST APIs.

## Solution

A `mutableRestSource(readUrl, writeConfig, options?)` factory function that returns `MutableDataSource`. It composes a `restSource` internally for reads and adds an HTTP-backed `dispatch()` for writes.

## Interface Changes

### MutableDataSource.dispatch becomes async

```typescript
export interface MutableDataSource extends DataSource {
  dispatch(action: DataAction): Promise<void>;
}
```

`simulated()` updated to return `Promise.resolve()` — contract alignment, no behavioral change.

### WriteConfig

```typescript
type UrlTemplate = string | ((action: DataAction) => string);

interface WriteEndpoint {
  url: UrlTemplate;
  method?: string;  // defaults: PATCH for update, POST for create, DELETE for delete
}

interface WriteConfig {
  update?: WriteEndpoint;
  create?: WriteEndpoint;
  delete?: WriteEndpoint;
  headers?: Record<string, string>;
  refreshAfterWrite?: boolean;  // default false
  keyColumn?: string;
}
```

- Each endpoint is optional — dispatching an unsupported action type rejects.
- `headers` shared across all write endpoints (auth tokens, content-type).
- `keyColumn` identifies which column holds the row key for update/delete matching in response merge.
- String URLs support `:key` substitution. Function URLs receive the full action.
- `refreshAfterWrite: true` re-fetches the read URL instead of merging the response body.

## Architecture

```
                    ┌──────────────────────────┐
                    │   mutableRestSource()     │
                    │                           │
  connect(sink) ──► │  ┌─────────────────────┐  │
  disconnect()  ──► │  │  restSource (read)   │  │ ──► sink.apply(snapshot)
                    │  └─────────────────────┘  │
                    │                           │
  dispatch(action)─►│  ┌─────────────────────┐  │
                    │  │  writeDispatcher     │  │ ──► HTTP request
                    │  │  (action → fetch)    │  │ ──► sink.apply(replace/append/remove)
                    │  └─────────────────────┘  │      or re-fetch if refreshAfterWrite
                    └──────────────────────────┘
```

Read path delegates to an internal `restSource`. Write path is independent — maps actions to HTTP calls, then updates the sink.

Constructed directly by callers (same pattern as `simulated()`), not through the source factory. The factory stays read-only — mutability is orthogonal to URL-scheme routing.

## Write Dispatch Flow

For each `dispatch(action)` call:

1. **Resolve endpoint** — look up `writeConfig[action.type]`. If absent, reject with `"Unsupported action type: ${action.type}"`.

2. **Build URL** — string: replace `:key` with `action.key` (update/delete). Function: call with the action.

3. **Build request body** — `action.changes` for update, `action.data` for create, no body for delete.

4. **HTTP call** — `fetch(url, { method, headers, body: JSON.stringify(body) })`. Uses injected `fetchFn` from options or global `fetch`.

5. **Error handling** — non-2xx response rejects with a `SourceError` containing status code and response text. Network failures also reject.

6. **Sync local state** (on success):
   - If `refreshAfterWrite: true` → re-fetch the read URL, done.
   - Otherwise, use the response body:
     - **create** → parse response as new row, push `append` event to sink
     - **update** → parse response as updated row, push `replace` event to sink (matched by `keyColumn`)
     - **delete** → push `remove` event to sink (matched by `keyColumn` + `action.key`)
   - If response body is empty (204 No Content) and `refreshAfterWrite` is false → auto-refresh as fallback.

7. **Promise resolves** after sink events are pushed (or re-fetch completes).

Sink events are the same types the simulated source uses (`append`, `replace`, `remove`), so downstream consumers handle them identically.

## Sync Strategy

**Pessimistic with response merge** as the default. The server is the source of truth:
- Wait for the HTTP response before updating local state.
- Use the response body as the authoritative row (no re-fetch needed).
- `refreshAfterWrite: true` escape hatch for APIs that return 204 No Content or partial responses.

The simulated source stays optimistic (in-memory, no server). This is only about the REST write path.

## Files Changed

| File | Change |
|------|--------|
| `packages/pages-data/src/datasource/types.ts` | `dispatch` returns `Promise<void>` |
| `packages/pages-data/src/datasource/sources/simulated/simulated-source.ts` | Return `Promise.resolve()` from dispatch |
| `packages/pages-data/src/datasource/sources/mutable-rest-source.ts` | New file — `mutableRestSource()` factory |
| `packages/pages-data/src/datasource/sources/mutable-rest-source.test.ts` | New file — tests |
| `packages/pages-data/src/index.ts` | Export `mutableRestSource`, `WriteConfig`, `WriteEndpoint`, `UrlTemplate` |

## Testing

Unit tests with injected `fetchFn` (same pattern as restSource tests). No HTTP server needed.

- update: correct URL (`:key` substituted), PATCH method, body is `action.changes`, response merged via `replace` event
- create: POST method, body is `action.data`, response merged via `append` event
- delete: DELETE method, no body, `remove` event to sink
- function URL template: callback receives full action, returned URL used
- unsupported action: dispatching a type with no endpoint rejects
- HTTP error: non-2xx rejects promise with status/message
- refreshAfterWrite: true: re-fetches read URL instead of merging response
- 204 No Content fallback: empty response body triggers auto-refresh
- custom headers: auth token propagated to write requests
- simulated source: dispatch still works (returns resolved promise)

## Not In Scope

- Optimistic updates (simulated source covers this use case)
- Source factory integration (mutability is orthogonal to protocol routing)
- Batch/bulk operations (single action per dispatch)
- WebSocket or SSE write paths (REST only)
