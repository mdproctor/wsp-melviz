# Design Journal — issue-378-diagram-editing-infra

## 2026-08-26 — Session 1: Design + Batch 1 implementation

### Design phase

Brainstormed the full diagram editing interaction model — 11 interaction types covering creation, connection, insertion, deletion, and reconnection. Key insight: auto-layout (ELK) makes all drag interactions tractable because every mutation ends with re-layout, not free-position reconciliation.

Decision review (3 rounds, 17 issues) revised 4 of 8 original decisions and surfaced 3 new ones:
- D2: Palette moved from graph-renderer to casehub-diagram (Lit/React package misfit)
- D3: Removed validateConnection callback from StencilGrammar — graph-core stays pure data
- D4: Hybrid approach — React Flow handles connections natively, custom pointer events only for palette drag
- D7: Per-handler architecture instead of single coordinator — React Flow IS the coordinator for native interactions

Light spec review (17 findings) added: viewport bridge design (§4.8), mutation orchestration via onMutation callback (§4.6), edge operations in graph-core (§4.12), batch-aware delete strategies, keyboard accessibility via context menu alternatives.

Plan review (14 findings) fixed: reconnectEdge API supports both endpoints, EditPolicy is per-instance on GraphCanvas (not global singleton), splitEdge composes from primitives, applyGraphEdit executor bridges GraphEdit to graph-core operations.

### Implementation — Batch 1: graph-core edge operations

Landed 4 new functions in `packages/graph-core/src/edit.ts`:
- `addEdge(model, newEdge)` — validates duplicate ID, dangling source/target
- `removeEdge(model, edgeId)` — validates existence
- `reconnectEdge(model, edgeId, { source?, target? })` — supports both endpoints, at-least-one validation
- `splitEdge(model, edgeId, insertNode)` — composes from removeEdge + addNode + addEdge + addEdge

20 new tests, all 127 passing. Two commits on `issue-378-diagram-editing-infra`.
