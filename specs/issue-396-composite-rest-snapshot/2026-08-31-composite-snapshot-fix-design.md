# Fix: composite() replaces REST snapshot with empty live accumulator

**Issue:** casehubio/casehub-pages#396
**Date:** 2026-08-31

## Problem

`composite(initial, live)` hands the outer sink directly to the live source
after the initial REST snapshot (`live.connect(outerSink!)` at line 36 of
`composite-source.ts`). The live source starts with its own empty internal
state — specifically, `PushSource`'s per-subscription `snapshotReceived`
flag is `false`.

When the first wire message arrives as an `append` op, `processWireMessage`
(push-source.ts:105-107) promotes it to a `snapshot` event because
`snapshotReceived` is false. This 1-row snapshot replaces all REST-loaded
data in the consumer's state.

### Concrete failure path

1. REST loads 10 incidents → `sink.apply({ type: "snapshot", dataset })` → table shows 10 rows
2. Composite disconnects REST, calls `live.connect(outerSink!)` — live source receives the raw outer sink
3. WebSocket receives one incident update → PushSource promotes append to snapshot → `sink.apply({ type: "snapshot", dataset: {rows: [1 row]} })` → table shows 1 row
4. 9 incidents vanish

## Design

### Interposing sink

Replace the direct `live.connect(outerSink!)` call with a wrapping sink that
intercepts the first snapshot from the live source and converts it to an
append event:

```typescript
let firstLiveSnapshot = true;
const liveSink: DataSink = {
  apply(event): void {
    if (phase !== "live") return;
    if (event.type === "snapshot" && firstLiveSnapshot) {
      firstLiveSnapshot = false;
      outerSink?.apply({ type: "append", rows: event.dataset.rows });
      return;
    }
    if (event.type === "snapshot") firstLiveSnapshot = false;
    outerSink?.apply(event);
  },
  error(err): void {
    if (phase !== "live") return;
    outerSink?.error(err);
  },
};
live.connect(liveSink);
```

### Semantics

- **First snapshot from live → append:** The REST snapshot is already in the
  consumer's state. The first live "snapshot" is overwhelmingly the
  promoted-append case (PushSource converting append→snapshot because
  `snapshotReceived` is false). Converting it to an append preserves REST
  data and adds the new rows.

- **Subsequent snapshots from live → pass through:** A genuine server-initiated
  snapshot (op: "snapshot") is a full state reset. It replaces everything,
  including the REST data. This is correct — the server is saying "here is
  the complete current state."

- **Non-snapshot events (append, replace, remove) → pass through:** These are
  inherently incremental and work correctly regardless.

### Phase guard

The liveSink also guards against stale events after disconnect by checking
`phase !== "live"`. The existing code already guards the initial sink this
way (line 25: `if (phase !== "initial") return`), but the live path
currently has no such guard since it passes the raw outerSink. The wrapping
sink adds this guard consistently.

## Test plan

1. **Existing tests pass** — no behavioral change for current test scenarios
   (the existing test connects live directly to outerSink and emits appends,
   which still work)
2. **New test: first live snapshot is converted to append** — connect composite,
   emit REST snapshot, then emit a snapshot from the live source. Verify the
   live snapshot arrives as an append event, preserving REST data.
3. **New test: subsequent live snapshots pass through as snapshots** — after
   the first live snapshot (converted to append), emit another snapshot from
   live. Verify it arrives as a snapshot event (full replacement).
4. **New test: live phase guard** — disconnect composite during live phase,
   then emit an event from live. Verify the event is not forwarded.

## Files changed

| File | Change |
|------|--------|
| `packages/pages-data/src/datasource/sources/composite-source.ts` | Replace `live.connect(outerSink!)` with wrapping liveSink |
| `packages/pages-data/src/datasource/sources/composite-source.test.ts` | Add 3 new test cases |

## References

- `packages/pages-data/src/datasource/sources/composite-source.ts:36` — the bug line
- `packages/pages-data/src/dataset/external/sources/push-source.ts:105-107` — append→snapshot promotion
- `packages/pages-data/src/datasource/types.ts` — DataSource, DataSink interfaces
- `packages/pages-data/src/dataset/events.ts` — DataSetEvent types
- casehubio/casehub-pages#396 — issue description with failure path
- casehubio/fsitrading#32 / #30 — discovery context
