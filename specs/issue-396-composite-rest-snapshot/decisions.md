## D1: Fix strategy for composite() snapshot replacement

**Choice:** Interposing sink in composite — convert first live snapshot to append
**Alternatives:**
- Mark snapshot-received in PushSource — fixes root cause but touches PushSource interface, wsSource, sseSource, and all their tests for a one-line behavioral fix
- Seed method on DataSource — clean abstraction but YAGNI; changes a core interface for a niche use case
**Rationale:** The fix is contained to composite-source.ts. The wrapping sink intercepts the first snapshot from the live source and converts it to an append, preserving the REST data already in the consumer's state. Subsequent genuine server snapshots pass through as replacements. No interface changes needed.
**Trade-offs:** The very first genuine server snapshot (if any) is also converted to append. In practice this edge case doesn't arise — the server's first message after WebSocket connect in a composite scenario is the promoted-append case, not a genuine full snapshot. The composite pattern exists specifically because REST provides initial state and live provides incremental updates.
**Sources:** composite-source.ts:36 (the bug line), push-source.ts:105-107 (append→snapshot promotion), issue #396
**Exploration:** quick
**Status:** captured

## D2: Genuine server snapshot semantics

**Choice:** Replace — genuine server snapshots should replace everything including REST data
**Alternatives:**
- Always incremental — even genuine server snapshots would be appended, never replacing. This would prevent server-initiated state resets from working correctly.
**Rationale:** A server snapshot means "here is the full current state." Treating it as a replacement is semantically correct. The bug is specifically about the promoted-append-to-snapshot case, not genuine snapshots.
**Trade-offs:** None — this matches the semantic contract of snapshot events.
**Sources:** DataSetEvent types (events.ts), issue #396
**Exploration:** quick
**Status:** captured
