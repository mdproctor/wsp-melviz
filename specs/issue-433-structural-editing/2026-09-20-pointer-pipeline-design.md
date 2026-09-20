# Unified Pointer Event Pipeline & Drill-Down Stack

**Issue:** #456 — Unified pointer event pipeline and drill-down stack
**Branch:** issue-456-pointer-pipeline (to be created)
**Date:** 2026-09-20

## Overview

Replace the competing gesture systems in graph-renderer (React Flow handles, custom NodeMoveCoordinator, stencil buttons) with a single `NodeGestureCoordinator` that owns the initial pointerdown and routes by gesture intent. Generalize blocks-ui's drill-down protocol into graph-renderer with a visual cascade of collapsible vertical bars.

Two problems solved:
1. **Drag is broken** — React Flow's full-node-sized invisible Handle captures the pointer before the 300ms hold timer classifies the gesture, causing ghost-then-canvas-takeover.
2. **Drill-down is blocks-ui specific** — the 3-phase protocol (emit → resolve → navigate) and the navigation stack should be reusable by any graph consumer.

## Part 1: NodeGestureCoordinator

### Root Cause

`stencil-wrapper.tsx:237-240` renders a full-node-sized invisible `<Handle id="source-full" type="source" className="stencil-source-handle" style={{...fullNodeHandle, zIndex:2}}>`. Every pointerdown on a node surface hits this Handle first. React Flow's Handle immediately calls `setPointerCapture()` and starts connection drawing.

`GraphCanvas.ts:177` attaches a **bubble-phase** pointerdown listener, which fires after the Handle has already claimed the pointer. The `NodeMoveCoordinator` then tries to release the capture after 300ms (`node-move-coordinator.ts:80-84`), but React Flow's internal state already thinks a connection is being drawn.

### Architecture

Replace the bubble-phase pointerdown in `GraphCanvas.ts:177-206` with a **capture-phase** listener. The coordinator intercepts the event before React Flow's Handle sees it.

#### Gesture Classification

```
pointerdown (capture phase on container)
    │
    ├─ target is stencil button (⤢, or any .stencil-action)?
    │     → let through, don't interfere
    │
    └─ target is node surface / handle
          │
          stopImmediatePropagation()  ← blocks Handle
          │
          start 300ms hold timer, track pointer position
          │
          ├─ pointer moves >3px during hold
          │     → CONNECT: replay PointerEvent on .stencil-source-handle
          │     → React Flow draws connection normally
          │
          ├─ pointer released <300ms, movement ≤3px
          │     → CLICK: fire onNodeClick callback
          │
          └─ 300ms elapsed, pointer still down, movement ≤3px
                → MOVE: ghost node, enter drag mode
                → attach capture-phase pointermove/pointerup
                → existing NodeMoveCoordinator drag logic
```

#### Event Replay for CONNECT

When a quick drag is detected (moved >3px before 300ms):

1. Find the `.stencil-source-handle` element within the node
2. Create a new `PointerEvent('pointerdown', { clientX, clientY, pointerId, bubbles: true, ... })` from the original event's coordinates
3. Dispatch on the Handle element — React Flow processes it normally
4. Forward the current pointermove to the Handle so the connection line follows the cursor

#### Key Files

| File | Change |
|------|--------|
| `graph-renderer/src/gesture/node-gesture-coordinator.ts` | **New** — capture-phase interceptor, gesture classification, event replay |
| `graph-renderer/src/gesture/node-gesture-coordinator.test.ts` | **New** — TDD tests for all gesture paths |
| `graph-renderer/src/bridge/GraphCanvas.ts` | Modify — replace bubble-phase pointerdown (lines 177-206) with coordinator setup |
| `graph-renderer/src/editing/node-move-coordinator.ts` | Modify — remove hold timer and pointer capture release (coordinator owns this now). Keep `activateDrag`, `onDragMove`, `onDragUp` |
| `graph-renderer/src/bridge/ReactFlowApp.tsx` | No change — `nodesDraggable={false}` stays |
| `graph-renderer/src/stencil-wrapper.tsx` | No change — full-node Handle stays, coordinator handles the conflict |

#### Gesture Constants

```typescript
const HOLD_DURATION = 300;       // ms before move classification
const HOLD_MOVE_TOLERANCE = 3;   // px — movement during hold that cancels it
const DRAG_THRESHOLD = 5;        // px — movement before visual drag starts
```

These move from `node-move-coordinator.ts` to `node-gesture-coordinator.ts`.

### Stencil Button Passthrough

Stencil actions (drill-down `⤢`, and any future buttons) have CSS class `.stencil-action`. The coordinator checks `event.composedPath()` for `.stencil-action` — if found, it does nothing and lets the button's own click handler fire.

## Part 2: Drill-Down Stack

### Protocol

The 3-phase protocol from blocks-ui, generalized:

1. **Classify** — `DrillDownConfig.resolve(nodeId, model)` returns `DrillDownTarget | null`. If `null`, the node isn't drillable (no `⤢` button rendered).
2. **Resolve** — the consumer's callback fetches/constructs the sub-graph model. This is domain-specific (blocks-ui resolves case definitions and SWF YAML).
3. **Navigate** — `DrillDownStack` pushes the target, collapses the current diagram into a vertical bar, renders the sub-diagram.

### Visual Cascade

```
Level 0 (full)    → drill into Node B →

┌───┐┌─────────────────────────────────────────┐
│ L ││ Level 1 (full — Node B's sub-graph)    │
│ 0 ││                                        │
│   ││   [Step 1]──[Step 2]──[Step 3]         │
│ ▸ ││                                        │
└───┘└─────────────────────────────────────────┘

→ drill into Step 2 →

┌───┐┌───┐┌───────────────────────────────────┐
│ L ││ L ││ Level 2 (full — Step 2 sub-graph) │
│ 0 ││ 1 ││                                   │
│   ││   ││   [Detail A]──[Detail B]          │
│ ▸ ││ ▸ ││                                   │
└───┘└───┘└───────────────────────────────────┘
```

- Each collapsed bar is ~32px wide with rotated level name
- Clicking a bar navigates back: all bars to its right collapse, that level's diagram expands to fill
- The active (rightmost) diagram gets the full remaining width
- Transition: 200ms CSS width animation for collapse/expand

### API

```typescript
interface DrillDownConfig {
  resolve: (nodeId: string, model: GraphModel) => Promise<DrillDownTarget | null>;
  renderBar?: (level: DrillDownLevel) => HTMLElement;
}

interface DrillDownTarget {
  name: string;
  model: GraphModel;
  diagramType?: string;
}

interface DrillDownLevel {
  name: string;
  depth: number;
  nodeId: string;
}
```

- `resolve` returning `null` for a node means no drill-down button rendered on that stencil
- `renderBar` optional — default renders vertical bar with rotated name text
- `GraphCanvas` gains optional `drillDown?: DrillDownConfig` prop

### Key Files

| File | Change |
|------|--------|
| `graph-renderer/src/drill-down/drill-down-stack.ts` | **New** — stack state, push/pop/navigate, bar rendering |
| `graph-renderer/src/drill-down/drill-down-stack.test.ts` | **New** — TDD tests |
| `graph-renderer/src/drill-down/types.ts` | **New** — `DrillDownConfig`, `DrillDownTarget`, `DrillDownLevel` |
| `graph-renderer/src/bridge/GraphCanvas.ts` | Modify — accept `drillDown` prop, wire to stack |
| `graph-renderer/src/stencil-wrapper.tsx` | Modify — conditionally render `⤢` button based on `resolve` returning non-null |

### ARIA Compliance

The drill-down cascade must be accessible:

- **Vertical bars**: `role="navigation"` with `aria-label="Drill-down breadcrumb"`. Each bar is a `button` with `aria-label="Navigate to {level name}"` and `aria-current="location"` on the active level.
- **Drill-down button (⤢)**: `aria-label="Drill into {node name}"`, `role="button"`, keyboard-focusable (`tabindex="0"`), activated by Enter/Space.
- **Stack navigation**: `aria-live="polite"` region announces level changes: "Drilled into {name}, level {N}" on push, "Navigated back to {name}" on bar click.
- **Keyboard support**: When a node is focused, `Enter` triggers drill-down (if drillable). `Escape` navigates up one level (pops stack). `Home` returns to root level.
- **Focus management**: After drill-down, focus moves to the first node in the sub-diagram. After navigating back, focus returns to the node that was drilled into.

### blocks-ui Migration

After this lands, blocks-ui's `diagram-workbench.ts` drill-down stack becomes a thin wrapper that:
1. Passes a `resolve` callback to `GraphCanvas`
2. The callback resolves case definitions, SWF YAML, HTN trees (existing domain logic)
3. The visual cascade (bars, navigation) is now owned by graph-renderer

## Part 3: Examples Showcase

A new example in `examples/samples/Graph Editing/` demonstrating 3-level nesting:

**"Nested Pipeline"** — Pipeline → Stage → Task
- Level 0: 3 pipeline stages (Ingest → Transform → Export), each drillable
- Level 1: Stage detail — 4 processing steps, 2 drillable
- Level 2: Task detail — leaf-level operations, not drillable

The example provides a static `resolve` callback with pre-built models for each level. Demonstrates the full cascade UX: drill into a stage, then a task, click a bar to navigate back.

## Testing Strategy

**TDD is mandatory** — this area has repeated regressions from competing event systems.

### NodeGestureCoordinator tests
- Quick drag (moved >3px before 300ms) → CONNECT event replayed on Handle
- Hold 300ms then drag → MOVE classification, ghost appears
- Click (release <300ms, no movement) → onNodeClick fires
- Stencil button click → coordinator does not interfere
- Multiple rapid clicks → no double-classification
- Pointer leaves node during hold → cancels classification

### DrillDownStack tests
- `push(target)` → stack depth increases, bar rendered
- Click bar at depth N → all levels >N popped, level N expanded
- `resolve` returning null → no drill-down button on stencil
- 3-level deep drill-down → 2 bars + active diagram
- Pop to root → no bars, full diagram

### ARIA tests
- Vertical bars have `role="navigation"`, each bar is a focusable button
- Drill-down button has `aria-label` with node name
- `Enter` on focused drillable node triggers drill-down
- `Escape` pops one level, `Home` returns to root
- `aria-live` region announces level changes
- Focus moves to first node after drill-down, returns to source node on pop

### Integration tests (Playwright)
- Examples showcase: drill 3 levels deep, click bars to navigate back
- Verify no ghost/canvas-takeover on drag
- Verify connection drawing still works after coordinator intercepts

## References

- `graph-renderer/src/editing/node-move-coordinator.ts` — current hold timer and drag logic
- `graph-renderer/src/bridge/GraphCanvas.ts:177-206` — current bubble-phase pointerdown
- `graph-renderer/src/stencil-wrapper.tsx:237-240` — full-node Handle (source of conflict)
- `graph-renderer/src/bridge/ReactFlowApp.tsx:247` — `nodesDraggable={false}`
- `blocks-ui/components/diagram-workbench/src/diagram-workbench.ts:62-66` — current drill-down stack
- `blocks-ui/components/casehub-diagram/src/casehub-diagram.ts:760-805` — current drill-down resolve
- `blocks-ui/packages/graph-stencil-case/src/stencils/worker.ts:72` — drill-down button emit
- decisions.md D1-D5 — all design choices documented
