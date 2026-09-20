# Unified Pointer Event Pipeline & Drill-Down Stack

**Issue:** #456 — Unified pointer event pipeline and drill-down stack
**Parent:** #433 — Tree/visual structural editing (epic — this spec and the selection toolbar spec are sibling deliverables under the same directory)
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

#### Multi-Select Awareness

The coordinator integrates with GraphCanvas's existing multi-select system. The current pointerdown handler (`GraphCanvas.ts:190-201`) branches on multi-select state before starting any drag:

- **Constrained multi-select active + pointerdown on a selected node** → start segment drag (move the selected segment as a group via `NodeMoveCoordinator.startSegmentDrag()`)
- **Multi-select active + pointerdown on a non-selected node** → clear multi-select, proceed with individual gesture classification
- **No multi-select** → individual gesture classification

The coordinator receives the current multi-select state via a callback (`getMultiSelectState()`) passed in its configuration. Segment drag is an unconditional delegation — there is no hold timer, no connect/click classification. The coordinator calls `stopPropagation()` (to block the Handle) and immediately delegates to `NodeMoveCoordinator.startSegmentDrag()`.

**RubberBandSelect coexistence:** RubberBandSelect uses a capture-phase pointerdown on the same container but only activates for Shift+click on empty canvas (`target.closest('.react-flow__node')` → return early). The coordinator's `stopPropagation()` prevents the event from reaching child elements but does NOT block other capture-phase listeners on the same element. RubberBandSelect's handler still fires — it returns early because the target is a node. Behavior is identical to today.

#### Gesture Classification

```
pointerdown (capture phase on container)
    │
    ├─ event.isTrusted === false?
    │     → let through (synthetic replay — see Event Replay)
    │
    ├─ active classification in progress (different pointerId)?
    │     → ignore — single-pointer model
    │
    ├─ target is stencil button (⤢, or any .stencil-action)?
    │     → let through, don't interfere
    │
    └─ target is node surface / handle
          │
          stopPropagation()  ← blocks Handle (descendant),
          │                    allows sibling capture listeners
          │
          start hold timer, track pointer position
          │
          ├─ pointer moves >threshold during hold
          │     → CONNECT: replay PointerEvent on .stencil-source-handle
          │     → React Flow draws connection normally
          │
          ├─ pointer released before hold duration, movement ≤threshold
          │     → CLICK: no action — native click event propagates
          │       to React Flow's onClick, which fires onNodeClick
          │
          └─ hold duration elapsed, pointer still down, movement ≤threshold
                → MOVE: enter drag mode
                → attach capture-phase pointermove/pointerup
                → existing NodeMoveCoordinator drag logic
                │
                ├─ pointer moves >DRAG_THRESHOLD (5px)
                │     → ghost node appears, splice detection starts
                │
                ├─ pointer leaves node container
                │     → LEAVE_TIMEOUT (500ms) grace period
                │     → re-enters within 500ms → continue drag
                │     → exceeds 500ms → cancel drag (no visual change)
                │
                └─ pointer released before DRAG_THRESHOLD
                      → cancelled (no visual change)
```

**Single-pointer guard:** The coordinator tracks one active classification at a time. If a `pointerdown` arrives with a different `pointerId` while a classification is active, it is ignored. This prevents multi-touch scenarios from corrupting the gesture state machine.

#### Event Replay for CONNECT

When a quick drag is detected (moved >threshold before hold duration elapses):

1. Find the `.stencil-source-handle` element within the node
2. Create a new `PointerEvent('pointerdown', { clientX, clientY, pointerId, bubbles: true, ... })` from the original event's coordinates
3. Dispatch on the Handle element — React Flow processes it normally
4. Forward the current pointermove to the Handle so the connection line follows the cursor

**CSS class management:** The replayed event triggers React Flow's Handle internal handler, which calls `setPointerCapture()` and fires `onConnectStart`. The `onConnectStart` callback in `GraphCanvas.ts:393-398` adds the `graph-connecting` CSS class, activating the extensive CSS rules in `css-isolation.ts:53-68` (handle visibility, connection line z-index, node hover effects). This class management works unmodified — the replayed event follows the exact same path as a real user event from React Flow's perspective.

**Re-entrancy guard:** The synthetic event dispatched in step 3 propagates normally and will reach the coordinator's own capture-phase listener. The coordinator checks `event.isTrusted === false` at the top of its handler (see Gesture Classification flow). Synthetic events dispatched via `dispatchEvent()` always have `isTrusted === false` per the DOM spec. This is a zero-state check — no flag management, no cleanup on dispose.

#### Key Files

| File | Change |
|------|--------|
| `graph-renderer/src/gesture/node-gesture-coordinator.ts` | **New** — capture-phase interceptor, gesture classification, event replay |
| `graph-renderer/src/gesture/node-gesture-coordinator.test.ts` | **New** — TDD tests for all gesture paths |
| `graph-renderer/src/bridge/GraphCanvas.ts` | Modify — replace bubble-phase pointerdown (lines 177-206) with coordinator setup; pass multi-select state callback to coordinator |
| `graph-renderer/src/editing/node-move-coordinator.ts` | Modify — remove hold timer and pointer capture release (coordinator owns this now). Keep `activateDrag`, `onDragMove`, `onDragUp` |
| `graph-renderer/src/bridge/ReactFlowApp.tsx` | No change — `nodesDraggable={false}` stays |
| `graph-renderer/src/stencil-wrapper.tsx` | Modify — add `.stencil-action` class to action buttons rendered by stencil wrapper (see Part 2); full-node Handle stays, coordinator handles the conflict |

#### Gesture Constants

```typescript
const HOLD_DURATION = 300;       // ms before move classification
const DRAG_THRESHOLD = 5;        // px — movement after hold before visual drag starts
const LEAVE_TIMEOUT = 500;       // ms — grace period when pointer leaves node during drag

function holdMoveTolerance(pointerType: string): number {
  return pointerType === 'touch' ? 10 : 3;
}
```

Touch devices have natural finger drift of 5–15px during a stationary press. A 3px tolerance causes touch users to almost never achieve HOLD classification — their finger drift exceeds tolerance before the timer fires, misclassifying intended moves as connects. The coordinator reads `event.pointerType` (available on all PointerEvents) and uses 10px for `'touch'`, 3px for `'mouse'`/`'pen'`.

These move from `node-move-coordinator.ts` to `node-gesture-coordinator.ts`.

#### Lifecycle Management

The coordinator must be disposed when `GraphCanvas` disconnects from the DOM, following the same pattern as the existing `NodeMoveCoordinator`:

- `GraphCanvas.disconnectedCallback()` calls `coordinator.dispose()`
- `dispose()` clears the hold timer, removes all event listeners, resets state
- The hold timer callback checks `if (this._disposed) return;` as its first line — `disconnectedCallback()` can fire before the timer, leaving the callback to run against a disposed coordinator

### Stencil Button Passthrough

Stencil actions (drill-down `⤢`, and any future buttons) have CSS class `.stencil-action`. The coordinator checks `event.composedPath()` for `.stencil-action` — if found, it does nothing and lets the button's own click handler fire.

The `.stencil-action` class is new — it does not exist in the codebase today. It is added by `stencil-wrapper.tsx` when rendering the drill-down `⤢` button (see Part 2, Key Files: `stencil-wrapper.tsx` — "conditionally render ⤢ button based on resolve returning non-null"). Any future stencil-level action buttons must also carry this class to be excluded from gesture classification.

## Part 2: Drill-Down Stack

### Protocol

The 3-phase protocol from blocks-ui, generalized:

1. **Classify** — `DrillDownConfig.resolve(nodeId, model)` returns `DrillDownTarget | null`. If `null`, the node isn't drillable (no `⤢` button rendered).
2. **Resolve** — the consumer's callback fetches domain data and produces a `GraphModel`. This is domain-specific: blocks-ui resolves case definitions and SWF YAML into `GraphModel` instances. The `resolve` callback is the parse-to-model boundary — the consumer owns the full pipeline from domain data (YAML strings, definitions) to a renderable graph model. graph-renderer never sees YAML.
3. **Navigate** — `DrillDownStack` pushes the target, collapses the current diagram into a vertical bar, renders the sub-diagram.

**Stale resolve guard:** The stack maintains a `_resolveGeneration` counter, incremented on every `resolve()` call. When a resolve completes, it checks whether the generation matches the current counter. If a newer resolve has been initiated (user clicked ⤢ on a different node while the first was pending), or the user has navigated back during the resolve, the stale result is discarded. This follows the same pattern as `GraphCanvas._layoutGeneration` in `_runLayout()`. No AbortController needed — the resolve function runs domain logic that completes normally; only the stack-push is gated.

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

### Rendering Architecture

The DrillDownStack does NOT create multiple React Flow instances. GraphCanvas has a single `_root: Root` that renders one `ReactFlowApp`. The stack works by swapping which `GraphModel` is rendered:

1. **Push (drill down):** The stack saves the current level's model and viewport state (`{ x, y, zoom }` from React Flow's `getViewport()`). `GraphCanvas` re-renders `ReactFlowApp` with the new target's model. A collapsed bar (plain DOM `<button>` element) for the previous level is inserted before the React container.

2. **Pop (navigate back via bar click):** The stack restores the saved model and viewport. `GraphCanvas` re-renders `ReactFlowApp` with the restored model. Bars for popped levels are removed from the DOM — destroyed, not hidden. The saved model and viewport provide instant restoration without re-fetching from the consumer.

3. **Container layout:** The GraphCanvas container uses CSS flexbox. Bars are fixed-width (`32px`) flex items. The React container (hosting `ReactFlowApp`) is `flex: 1` and fills the remaining width. When bars are added/removed, CSS flexbox naturally redistributes space and React Flow's `fitView` is called after the transition completes (200ms animation).

4. **State management:** The stack is a plain TypeScript class instantiated by `GraphCanvas` when a `drillDown` config is provided. Stack state (array of `StackLevel` objects) is managed internally. `push`/`pop` operations call back to `GraphCanvas` to trigger model re-rendering and bar DOM updates. Bars are plain DOM elements created/destroyed by the stack — they live outside the React root, in the GraphCanvas shadow DOM.

```typescript
interface StackLevel {
  name: string;
  nodeId: string;
  model: GraphModel;
  viewport: { x: number; y: number; zoom: number };
  diagramType?: string;
}
```

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
- `resolve` is the parse-to-model boundary: the consumer fetches domain data (YAML, definitions) and produces a `GraphModel`. The parse logic already exists in blocks-ui — `casehub-diagram.ts` already transforms case definitions and SWF YAML into renderable models today.
- `renderBar` optional — default renders vertical bar with rotated name text
- `diagramType` is consumer metadata (e.g. blocks-ui uses it to select the right stencil registry per level — case stencils vs SWF stencils). It does NOT select a different rendering engine — all levels render through the same `ReactFlowApp`
- `GraphCanvas` gains optional `drillDown?: DrillDownConfig` prop

**Model lifecycle:** The `DrillDownTarget.model` returned by `resolve()` is a snapshot owned by the consumer. The stack does not observe, invalidate, or synchronise models across levels. This is intentional — different consumers have different data flow models (static examples, live server-backed data, editable graphs). Consumers with live data should re-resolve when their source changes. Edit propagation across levels and model garbage collection are consumer concerns, not stack concerns. See GitHub issue for future live-data protocol when a concrete consumer needs it.

### Key Files

| File | Change |
|------|--------|
| `graph-renderer/src/drill-down/drill-down-stack.ts` | **New** — stack state, push/pop/navigate, bar rendering |
| `graph-renderer/src/drill-down/drill-down-stack.test.ts` | **New** — TDD tests |
| `graph-renderer/src/drill-down/types.ts` | **New** — `DrillDownConfig`, `DrillDownTarget`, `DrillDownLevel` |
| `graph-renderer/src/bridge/GraphCanvas.ts` | Modify — accept `drillDown` prop, wire to stack, extend `_keyDownHandler` with drill-down keyboard shortcuts |
| `graph-renderer/src/stencil-wrapper.tsx` | Modify — conditionally render `⤢` button based on `resolve` returning non-null |

### Keyboard Handler Ownership

Drill-down keyboard handlers are added to GraphCanvas's existing `_keyDownHandler` callback (`GraphCanvas.ts:226-231`), which already handles Delete/Backspace for multi-select. The combined handler checks:

1. Delete/Backspace with multi-select active → multi-select delete (existing)
2. `Enter` on focused drillable node + drill-down config present → trigger drill-down
3. `Escape` with stack depth > 0 → pop one level
4. `Home` with stack depth > 0 → pop to root

This consolidation avoids multiple keydown listeners on the same element. The drill-down handlers only fire when `this._drillDownStack?.depth > 0` (for Escape/Home) or when the focused node is drillable (for Enter).

### ARIA Compliance

The drill-down cascade must be accessible:

- **Vertical bars**: `role="navigation"` with `aria-label="Drill-down breadcrumb"`. Each bar is a `button` with `aria-label="Navigate to {level name}"` and `aria-current="location"` on the active level.
- **Drill-down button (⤢)**: `aria-label="Drill into {node name}"`, `role="button"`, keyboard-focusable (`tabindex="0"`), activated by Enter/Space. The button carries class `.stencil-action`.
- **Stack navigation**: `aria-live="polite"` region announces level changes: "Drilled into {name}, level {N}" on push, "Navigated back to {name}" on bar click.
- **Keyboard support**: When a node is focused, `Enter` triggers drill-down (if drillable). `Escape` navigates up one level (pops stack). `Home` returns to root level. **Focus guard:** These shortcuts only activate when focus is on a graph element (node, edge, canvas) — they are suppressed when focus is inside an `<input>`, `<textarea>`, or `[contenteditable]` element, where `Escape` and `Home` have standard text-editing semantics.
- **Focus management**: After drill-down, focus moves to the first node in the sub-diagram (first element in the model's node array). After navigating back, focus returns to the node that was drilled into. **Layout-aware:** Focus transfer is deferred until after `_runLayout()` completes for the new sub-diagram, since nodes have no positioned DOM elements until ELK layout finishes. The stack listens for the layout promise to resolve before calling `focus()` on the target node element.

### blocks-ui Migration

After this lands, blocks-ui's `diagram-workbench.ts` drill-down stack is replaced:
1. Passes a `resolve` callback to `GraphCanvas`'s `drillDown` config
2. The callback owns the domain-specific pipeline: fetch case definitions, SWF YAML, or HTN trees → parse into `GraphModel` → return as `DrillDownTarget`. This is existing domain logic in blocks-ui (`casehub-diagram.ts:760-805`) — the parse-to-model step already exists, it just needs to return a `DrillDownTarget` instead of `{ name, yaml, diagramType }`
3. The visual cascade (bars, navigation, keyboard shortcuts) is now owned by graph-renderer
4. `diagramType` on `DrillDownTarget` lets blocks-ui select the right stencil registry per level (case stencils vs SWF stencils vs HTN stencils)

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
- Quick drag (moved >threshold before hold) → CONNECT event replayed on Handle
- Hold then drag → MOVE classification, ghost appears
- Click (release before hold, no movement) → native click propagates, onNodeClick fires once
- Stencil button click (.stencil-action) → coordinator does not interfere
- Multiple rapid clicks → no double-classification
- Pointer leaves node during drag → 500ms grace period; re-enter within 500ms continues, exceeds 500ms cancels
- Synthetic replay event (isTrusted=false) → coordinator passes through, no re-entrancy
- Second pointer during active classification → ignored (single-pointer guard)
- Touch pointerType → 10px movement tolerance (vs 3px for mouse)
- Coordinator disposed during hold timer → timer callback no-ops
- Multi-select active + pointerdown on selected node → segment drag delegated to NodeMoveCoordinator
- Multi-select active + pointerdown on non-selected node → multi-select cleared, individual classification
- RubberBandSelect capture handler not blocked by coordinator's stopPropagation

### DrillDownStack tests
- `push(target)` → stack depth increases, bar rendered
- Click bar at depth N → all levels >N popped, level N expanded
- `resolve` returning null → no drill-down button on stencil
- 3-level deep drill-down → 2 bars + active diagram
- Pop to root → no bars, full diagram
- Pop destroys bar DOM elements (not hidden)
- Push saves viewport, pop restores viewport
- Single ReactFlowApp instance throughout — model swapped, not duplicated
- Concurrent resolve: second resolve supersedes first, stale result discarded
- Navigate back during pending resolve → stale result discarded

### ARIA tests
- Vertical bars have `role="navigation"`, each bar is a focusable button
- Drill-down button has `aria-label` with node name and class `.stencil-action`
- `Enter` on focused drillable node triggers drill-down
- `Escape` pops one level, `Home` returns to root
- `Escape`/`Home` suppressed when focus is in `<input>`, `<textarea>`, or `[contenteditable]`
- `aria-live` region announces level changes
- Focus moves to first node after drill-down (after layout completes), returns to source node on pop

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
- 2026-09-20-pointer-pipeline-decisions.md D1-D5 — all design choices documented
