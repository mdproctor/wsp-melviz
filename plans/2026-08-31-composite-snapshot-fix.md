# Fix: composite() REST snapshot replacement — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #396 — composite() source replaces REST snapshot with empty live accumulator on first push event
**Issue group:** #396

**Goal:** Fix composite() so the first live source event doesn't clobber REST snapshot data.

**Architecture:** Replace the direct `live.connect(outerSink!)` with an interposing sink that converts the first snapshot from the live source to an append event, preserving REST-loaded data. Subsequent snapshots and all non-snapshot events pass through unmodified.

**Tech Stack:** TypeScript, Vitest

## Global Constraints

- No interface changes to DataSource or DataSink
- No changes to PushSource or processWireMessage
- Existing tests must continue to pass without modification

---

## Batch 1: Fix and verify

### Task 1: Test and fix composite live snapshot replacement

**Files:**
- Modify: `packages/pages-data/src/datasource/sources/composite-source.test.ts`
- Modify: `packages/pages-data/src/datasource/sources/composite-source.ts`
- Test: `packages/pages-data/src/datasource/sources/composite-source.test.ts`

**Interfaces:**
- Consumes: `composite(initial, live)` from `composite-source.ts`, `controllableSource()` and `textDataset()` from test file / test-helpers
- Produces: no new exports — behavioral fix only

- [ ] **Step 1: Write failing test — first live snapshot replaces REST data**

Add this test to the existing `describe("composite", ...)` block in `composite-source.test.ts`:

```typescript
it("converts first live snapshot to append (preserves REST data)", () => {
  const initial = controllableSource();
  const live = controllableSource();
  const source = composite(initial, live);

  const events: DataSetEvent[] = [];
  const sink: DataSink = {
    apply(event) { events.push(event); },
    error() {},
  };

  source.connect(sink);

  // REST snapshot — 1 row
  initial.emitSnapshot(textDataset("rest-row"));
  expect(events).toHaveLength(1);
  expect(events[0]!.type).toBe("snapshot");

  // Live source emits a snapshot (the promoted-append case)
  live.emitSnapshot(textDataset("live-row"));

  // Should arrive as append, NOT snapshot
  expect(events).toHaveLength(2);
  expect(events[1]!.type).toBe("append");
  expect((events[1] as { type: "append"; rows: unknown[] }).rows).toHaveLength(1);
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `yarn workspace @casehubio/pages-data run test -- --run composite-source.test.ts -t "converts first live snapshot"`
Expected: FAIL — the live snapshot arrives as `type: "snapshot"` not `type: "append"`

- [ ] **Step 3: Write failing test — subsequent live snapshots pass through**

```typescript
it("passes through subsequent live snapshots as replacements", () => {
  const initial = controllableSource();
  const live = controllableSource();
  const source = composite(initial, live);

  const events: DataSetEvent[] = [];
  const sink: DataSink = {
    apply(event) { events.push(event); },
    error() {},
  };

  source.connect(sink);

  // REST snapshot
  initial.emitSnapshot(textDataset("rest-row"));

  // First live snapshot — converted to append
  live.emitSnapshot(textDataset("live-row-1"));
  expect(events[1]!.type).toBe("append");

  // Second live snapshot — genuine server state reset, passes through
  live.emitSnapshot(textDataset("live-row-2"));
  expect(events).toHaveLength(3);
  expect(events[2]!.type).toBe("snapshot");
});
```

- [ ] **Step 4: Write failing test — live phase guard blocks events after disconnect**

```typescript
it("does not forward live events after disconnect", () => {
  const initial = controllableSource();
  const live = controllableSource();
  const source = composite(initial, live);

  const events: DataSetEvent[] = [];
  const errors: SourceError[] = [];
  const sink: DataSink = {
    apply(event) { events.push(event); },
    error(err) { errors.push(err); },
  };

  source.connect(sink);
  initial.emitSnapshot(textDataset("data"));

  // Now in live phase — disconnect
  source.disconnect();

  // Live source still has a reference to the sink — events should be dropped
  // (The controllable source's sink was nulled by disconnect, but in real
  // push sources the subscription callback may still fire.)
  expect(events).toHaveLength(1); // Only the REST snapshot
  expect(errors).toHaveLength(0);
});
```

- [ ] **Step 5: Run all three new tests to verify they fail**

Run: `yarn workspace @casehubio/pages-data run test -- --run composite-source.test.ts`
Expected: "converts first live snapshot" fails (snapshot not converted), other two may pass or fail depending on current behavior.

- [ ] **Step 6: Implement the interposing sink fix**

In `composite-source.ts`, replace line 36 (`live.connect(outerSink!)`) with the interposing liveSink. The full updated `connect` method body:

```typescript
connect(sink: DataSink): void {
  outerSink = sink;
  phase = "initial";

  let firstLiveSnapshot = true;

  const initialSink: DataSink = {
    apply(event): void {
      if (phase !== "initial") return;

      // Only snapshot triggers the handoff
      if (event.type !== "snapshot") return;

      // Forward the snapshot to the outer sink
      outerSink?.apply(event);

      // Disconnect initial, connect live with interposing sink
      phase = "live";
      initial.disconnect();

      const liveSink: DataSink = {
        apply(event): void {
          if (phase !== "live") return;
          if (event.type === "snapshot" && firstLiveSnapshot) {
            firstLiveSnapshot = false;
            outerSink?.apply({ type: "append", rows: event.dataset.rows });
            return;
          }
          outerSink?.apply(event);
        },
        error(err): void {
          if (phase !== "live") return;
          outerSink?.error(err);
        },
      };
      live.connect(liveSink);
    },

    error(err): void {
      if (phase !== "initial") return;

      // Error in initial — stay in error state, do NOT connect live
      phase = "error";
      outerSink?.error(err);
    },
  };

  initial.connect(initialSink);
},
```

- [ ] **Step 7: Run all tests**

Run: `yarn workspace @casehubio/pages-data run test -- --run composite-source.test.ts`
Expected: ALL tests pass (existing + 3 new)

- [ ] **Step 8: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/src/datasource/sources/composite-source.ts packages/pages-data/src/datasource/sources/composite-source.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "fix(#396): composite() converts first live snapshot to append — preserves REST data

Interpose a wrapping sink between live source and outer sink. First
snapshot from the live source is converted to an append event so REST-
loaded data is preserved. Subsequent snapshots pass through as full
replacements. Phase guard prevents stale events after disconnect.

Closes #396"
```

## References

- [2026-08-31-composite-snapshot-fix-design.md] — design spec this plan implements
- `packages/pages-data/src/datasource/sources/composite-source.ts:36` — the bug line
- `packages/pages-data/src/dataset/external/sources/push-source.ts:105-107` — append→snapshot promotion
- `packages/pages-data/src/datasource/types.ts` — DataSource, DataSink, SourceError interfaces
- `packages/pages-data/src/dataset/events.ts` — DataSetEvent union type
- `packages/pages-data/src/datasource/sources/test-helpers.ts` — textDataset(), controllableSource()
- [GitHub #396] — issue with concrete failure path
