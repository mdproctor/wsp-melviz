# Unified Pointer Event Pipeline & Drill-Down Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #456 — Unified pointer event pipeline and drill-down stack
**Issue group:** #456

**Goal:** Fix broken node drag by replacing competing gesture systems with a single capture-phase coordinator, and generalize blocks-ui's drill-down into graph-renderer with a visual cascade.

**Architecture:** `NodeGestureCoordinator` intercepts pointerdown in capture phase before React Flow's Handle claims the pointer. Classifies gestures (connect/move/click) via a 300ms hold timer with touch-aware tolerances. `DrillDownStack` manages a navigation stack with collapsible vertical bars. Both integrate into `GraphCanvas` via clean delegation.

**Tech Stack:** TypeScript, React Flow (@xyflow/react), Lit (GraphCanvas), Vitest

**Pre-requisite:** Run `work start #456` to create the feature branch before executing.

## Global Constraints

- Source files in `packages/graph-renderer/src/`
- Tests use Vitest with `describe`/`it`/`expect`, environment `jsdom`
- React Flow's `nodesDraggable={false}` — must NOT be changed
- `GraphCanvas` uses light DOM (`createRenderRoot()` returns `this`)
- Full-node invisible Handle (`stencil-source-handle`, z-index 2) stays — coordinator works around it
- blocks-ui current branch: `issue-158-lsp-schema-refinements` — if blocks-ui changes needed: switch to main first, make changes, switch back and rebase

---

## Batch 1: Gesture Coordinator — fix the drag bug

### Task 1: NodeGestureCoordinator

**Files:**
- Create: `packages/graph-renderer/src/gesture/node-gesture-coordinator.ts`
- Test: `packages/graph-renderer/src/gesture/node-gesture-coordinator.test.ts`

**Interfaces:**
- Consumes: `GraphModel` from `@casehubio/graph-core`, `PointerEvent` browser API
- Produces: `NodeGestureCoordinator` — `{ attach(container: HTMLElement): void; dispose(): void }`. Constructor takes `NodeGestureConfig`:

```typescript
interface NodeGestureConfig {
  onConnect: (nodeId: string, event: PointerEvent) => void;
  onMove: (nodeId: string, event: PointerEvent, model: GraphModel) => void;
  onSegmentMove: (subject: DragSubject, event: PointerEvent, model: GraphModel) => void;
  getMultiSelectState: () => MultiSelectState;
  getModel: () => GraphModel;
}

interface MultiSelectState {
  selectedNodeIds: Set<string>;
  mode: 'none' | 'constrained';
  boundaryInput?: string;
  boundaryOutput?: string;
}
```

Dispatches: no custom events — calls config callbacks directly.

- [ ] **Step 1: Write failing tests for gesture classification**

Create `packages/graph-renderer/src/gesture/node-gesture-coordinator.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { NodeGestureCoordinator } from './node-gesture-coordinator.js';
import type { GraphModel } from '@casehubio/graph-core';

function makeConfig(overrides: Partial<NodeGestureConfig> = {}) {
  return {
    onConnect: vi.fn(),
    onMove: vi.fn(),
    onSegmentMove: vi.fn(),
    getMultiSelectState: () => ({ selectedNodeIds: new Set<string>(), mode: 'none' as const }),
    getModel: () => ({ nodes: [], edges: [] } as unknown as GraphModel),
    ...overrides,
  };
}

function makePointerEvent(type: string, opts: Partial<PointerEventInit> & { clientX?: number; clientY?: number } = {}): PointerEvent {
  return new PointerEvent(type, {
    bubbles: true, composed: true, pointerId: 1,
    pointerType: 'mouse', clientX: 100, clientY: 100,
    ...opts,
  });
}

function createNodeDOM(): { container: HTMLElement; node: HTMLElement; handle: HTMLElement } {
  const container = document.createElement('div');
  const node = document.createElement('div');
  node.className = 'react-flow__node';
  node.dataset.id = 'node-1';
  const handle = document.createElement('div');
  handle.className = 'stencil-source-handle';
  node.appendChild(handle);
  container.appendChild(node);
  document.body.appendChild(container);
  return { container, node, handle };
}

describe('NodeGestureCoordinator', () => {
  let container: HTMLElement;
  let node: HTMLElement;
  let handle: HTMLElement;
  let coordinator: NodeGestureCoordinator;

  beforeEach(() => {
    ({ container, node, handle } = createNodeDOM());
  });

  afterEach(() => {
    coordinator?.dispose();
    container?.remove();
  });

  it('CLICK: release before hold, no movement → no connect/move called', () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    node.dispatchEvent(makePointerEvent('pointerdown'));
    node.dispatchEvent(new PointerEvent('pointerup', { bubbles: true, pointerId: 1 }));
    expect(config.onConnect).not.toHaveBeenCalled();
    expect(config.onMove).not.toHaveBeenCalled();
  });

  it('CONNECT: quick drag >3px before 300ms → onConnect not called, event replayed on handle', () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    const handleCapture = vi.fn();
    handle.addEventListener('pointerdown', handleCapture);
    node.dispatchEvent(makePointerEvent('pointerdown', { clientX: 100, clientY: 100 }));
    container.dispatchEvent(makePointerEvent('pointermove', { clientX: 110, clientY: 100 }));
    expect(handleCapture).toHaveBeenCalled();
    expect(config.onMove).not.toHaveBeenCalled();
  });

  it('MOVE: hold 300ms then pointer still down → onMove called', async () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    node.dispatchEvent(makePointerEvent('pointerdown'));
    await new Promise(r => setTimeout(r, 350));
    expect(config.onMove).toHaveBeenCalledWith('node-1', expect.any(PointerEvent), expect.anything());
  });

  it('synthetic replay (isTrusted=false) passes through without interception', () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    const synthetic = new PointerEvent('pointerdown', { bubbles: true, pointerId: 1 });
    Object.defineProperty(synthetic, 'isTrusted', { value: false });
    const stopped = vi.fn();
    container.addEventListener('pointerdown', stopped);
    node.dispatchEvent(synthetic);
    expect(stopped).toHaveBeenCalled();
  });

  it('stencil-action button click passes through', () => {
    const actionBtn = document.createElement('button');
    actionBtn.className = 'stencil-action';
    node.appendChild(actionBtn);
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    const clickHandler = vi.fn();
    actionBtn.addEventListener('click', clickHandler);
    actionBtn.dispatchEvent(makePointerEvent('pointerdown'));
    actionBtn.click();
    expect(clickHandler).toHaveBeenCalled();
    expect(config.onConnect).not.toHaveBeenCalled();
    expect(config.onMove).not.toHaveBeenCalled();
  });

  it('second pointer during active classification is ignored', () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    node.dispatchEvent(makePointerEvent('pointerdown', { pointerId: 1 }));
    node.dispatchEvent(makePointerEvent('pointerdown', { pointerId: 2 }));
    // Should not crash or double-classify
    expect(config.onConnect).not.toHaveBeenCalled();
  });

  it('touch pointerType uses 10px tolerance', () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    const handleCapture = vi.fn();
    handle.addEventListener('pointerdown', handleCapture);
    node.dispatchEvent(makePointerEvent('pointerdown', { pointerType: 'touch', clientX: 100, clientY: 100 }));
    // Move 5px — under 10px touch tolerance, should NOT trigger CONNECT
    container.dispatchEvent(makePointerEvent('pointermove', { pointerType: 'touch', clientX: 105, clientY: 100 }));
    expect(handleCapture).not.toHaveBeenCalled();
  });

  it('dispose clears hold timer and listeners', async () => {
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    node.dispatchEvent(makePointerEvent('pointerdown'));
    coordinator.dispose();
    await new Promise(r => setTimeout(r, 350));
    expect(config.onMove).not.toHaveBeenCalled();
  });

  it('CONNECT on node without source handle → no action', () => {
    handle.remove(); // node has no source handle
    const config = makeConfig();
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    node.dispatchEvent(makePointerEvent('pointerdown', { clientX: 100, clientY: 100 }));
    container.dispatchEvent(makePointerEvent('pointermove', { clientX: 110, clientY: 100 }));
    // No crash, no connect
    expect(config.onConnect).not.toHaveBeenCalled();
  });

  it('multi-select constrained + selected node → onSegmentMove called immediately', () => {
    const config = makeConfig({
      getMultiSelectState: () => ({
        selectedNodeIds: new Set(['node-1']),
        mode: 'constrained',
      }),
    });
    coordinator = new NodeGestureCoordinator(config);
    coordinator.attach(container);
    node.dispatchEvent(makePointerEvent('pointerdown'));
    expect(config.onSegmentMove).toHaveBeenCalled();
    expect(config.onMove).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-renderer test -- src/gesture/node-gesture-coordinator.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement NodeGestureCoordinator**

Create `packages/graph-renderer/src/gesture/node-gesture-coordinator.ts`. Key implementation points from the spec:

- Capture-phase `pointerdown` listener on container
- `event.isTrusted === false` → pass through (re-entrancy guard)
- `composedPath().find(el => el.classList?.contains('stencil-action'))` → pass through
- `event.target.closest('.react-flow__node')` → extract nodeId from `dataset.id`
- `stopPropagation()` (NOT `stopImmediatePropagation`) — blocks Handle descendant, allows sibling capture listeners (RubberBandSelect)
- Check multi-select state: constrained + node in selection → call `onSegmentMove` immediately, no hold timer
- Otherwise: start hold timer (`HOLD_DURATION = 300`)
- During hold: capture-phase `pointermove` listener tracks distance. If distance > `holdMoveTolerance(event.pointerType)` → CONNECT path
- CONNECT: find `.stencil-source-handle` in node, null guard, dispatch synthetic `new PointerEvent('pointerdown', {...})` on handle
- Hold timer fires: call `onMove(nodeId, originalEvent, model)`
- Pointer released before hold: CLICK — do nothing, native click propagates
- `dispose()`: clear timer, remove listeners, set `_disposed` flag checked in timer callback

```typescript
export interface NodeGestureConfig {
  onConnect: (nodeId: string, event: PointerEvent) => void;
  onMove: (nodeId: string, event: PointerEvent, model: GraphModel) => void;
  onSegmentMove: (subject: { type: 'segment'; nodeIds: string[]; boundaryInput?: string; boundaryOutput?: string }, event: PointerEvent, model: GraphModel) => void;
  getMultiSelectState: () => { selectedNodeIds: Set<string>; mode: 'none' | 'constrained'; boundaryInput?: string; boundaryOutput?: string };
  getModel: () => GraphModel;
}

const HOLD_DURATION = 300;
const DRAG_THRESHOLD = 5;
const LEAVE_TIMEOUT = 500;

function holdMoveTolerance(pointerType: string): number {
  return pointerType === 'touch' ? 10 : 3;
}

export class NodeGestureCoordinator {
  // implementation per spec §Part 1
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer test -- src/gesture/node-gesture-coordinator.test.ts`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add packages/graph-renderer/src/gesture/
git commit -m "feat(graph-renderer): add NodeGestureCoordinator with TDD tests Refs #456"
```

### Task 2: Wire coordinator into GraphCanvas, refactor NodeMoveCoordinator

**Files:**
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts:177-206` — replace bubble-phase pointerdown with coordinator
- Modify: `packages/graph-renderer/src/editing/node-move-coordinator.ts` — remove `startDrag`/`startSegmentDrag`, add `activateMove`/`activateSegmentMove`

**Interfaces:**
- Consumes: `NodeGestureCoordinator` from Task 1
- Produces: Updated `NodeMoveCoordinator` with new public API: `activateMove(nodeId, event, model)`, `activateSegmentMove(subject, event, model)`

- [ ] **Step 1: Run existing graph-renderer tests to confirm baseline**

Run: `yarn workspace @casehubio/graph-renderer test`
Expected: All PASS

- [ ] **Step 2: Refactor NodeMoveCoordinator API**

In `node-move-coordinator.ts`:
- Remove `HOLD_DURATION`, `HOLD_MOVE_TOLERANCE` constants (moved to gesture coordinator)
- Keep `DRAG_THRESHOLD`, `LEAVE_TIMEOUT`
- Rename `startDrag` → `activateMove`: skip hold timer, call `confirmHold()` directly, then register drag listeners
- Rename `startSegmentDrag` → `activateSegmentMove`: same — skip hold timer, activate immediately
- Remove the hold timer setup, `onHoldMove`, `onHoldUp` logic
- Remove pointer capture release from `confirmHold()` — coordinator already blocked the Handle
- Keep `activateDrag`, `onDragMove`, `onDragUp`, splice detection, dispose

- [ ] **Step 3: Replace GraphCanvas pointerdown handler**

In `GraphCanvas.ts`:
- Remove the bubble-phase pointerdown listener (lines 177-206)
- Import `NodeGestureCoordinator` and create it in `connectedCallback` (or first render)
- Config callbacks:
  - `onConnect`: no-op (React Flow handles it via replayed event)
  - `onMove`: `this._moveCoordinator.activateMove(nodeId, event, this._model)`
  - `onSegmentMove`: `this._moveCoordinator.activateSegmentMove(subject, event, this._model)`
  - `getMultiSelectState`: `() => this._multiSelect`
  - `getModel`: `() => this._model`
- Call `coordinator.attach(this._container)` after container is available
- Call `coordinator.dispose()` in `disconnectedCallback()`

- [ ] **Step 4: Run full graph-renderer test suite**

Run: `yarn workspace @casehubio/graph-renderer test`
Expected: All PASS (existing + new gesture tests)

- [ ] **Step 5: Commit**

```bash
git add packages/graph-renderer/src/
git commit -m "feat(graph-renderer): wire NodeGestureCoordinator into GraphCanvas, refactor move coordinator API Refs #456"
```

## Batch 2: Drill-Down Foundation

### Task 3: DrillDownState — pure stack logic

**Files:**
- Create: `packages/graph-renderer/src/drill-down/types.ts`
- Create: `packages/graph-renderer/src/drill-down/drill-down-state.ts`
- Test: `packages/graph-renderer/src/drill-down/drill-down-state.test.ts`

**Interfaces:**
- Consumes: `GraphModel` from `@casehubio/graph-core`
- Produces: Types (`DrillDownConfig`, `DrillDownTarget`, `DrillDownLevel`, `StackLevel`) and `DrillDownState` class with `push(target, nodeId)`, `pop(): StackLevel | undefined`, `navigateTo(depth)`, `get levels()`, `get activeIndex()`, `get depth()`, `resolveGeneration`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DrillDownState } from './drill-down-state.js';
import type { StackLevel, DrillDownTarget } from './types.js';
import type { GraphModel } from '@casehubio/graph-core';

const MODEL_A = { nodes: [{ id: 'a' }], edges: [] } as unknown as GraphModel;
const MODEL_B = { nodes: [{ id: 'b' }], edges: [] } as unknown as GraphModel;
const MODEL_C = { nodes: [{ id: 'c' }], edges: [] } as unknown as GraphModel;

function makeTarget(name: string, model: GraphModel): DrillDownTarget {
  return { name, model };
}

describe('DrillDownState', () => {
  it('starts empty', () => {
    const state = new DrillDownState();
    expect(state.depth).toBe(0);
    expect(state.activeIndex).toBe(-1);
    expect(state.levels).toEqual([]);
  });

  it('push increases depth', () => {
    const state = new DrillDownState();
    state.push(makeTarget('Level 1', MODEL_A), 'node-1', { x: 0, y: 0, zoom: 1 }, [], []);
    expect(state.depth).toBe(1);
    expect(state.activeIndex).toBe(0);
    expect(state.levels[0]!.name).toBe('Level 1');
  });

  it('3-level deep push', () => {
    const state = new DrillDownState();
    state.push(makeTarget('L1', MODEL_A), 'n1', { x: 0, y: 0, zoom: 1 }, [], []);
    state.push(makeTarget('L2', MODEL_B), 'n2', { x: 0, y: 0, zoom: 1 }, [], []);
    state.push(makeTarget('L3', MODEL_C), 'n3', { x: 0, y: 0, zoom: 1 }, [], []);
    expect(state.depth).toBe(3);
  });

  it('pop returns popped level and decreases depth', () => {
    const state = new DrillDownState();
    state.push(makeTarget('L1', MODEL_A), 'n1', { x: 0, y: 0, zoom: 1 }, [], []);
    state.push(makeTarget('L2', MODEL_B), 'n2', { x: 0, y: 0, zoom: 1 }, [], []);
    const popped = state.pop();
    expect(popped?.name).toBe('L2');
    expect(state.depth).toBe(1);
  });

  it('navigateTo removes levels above target depth', () => {
    const state = new DrillDownState();
    state.push(makeTarget('L1', MODEL_A), 'n1', { x: 0, y: 0, zoom: 1 }, [], []);
    state.push(makeTarget('L2', MODEL_B), 'n2', { x: 0, y: 0, zoom: 1 }, [], []);
    state.push(makeTarget('L3', MODEL_C), 'n3', { x: 0, y: 0, zoom: 1 }, [], []);
    const removed = state.navigateTo(0);
    expect(removed).toHaveLength(2);
    expect(state.depth).toBe(1);
  });

  it('pop to root', () => {
    const state = new DrillDownState();
    state.push(makeTarget('L1', MODEL_A), 'n1', { x: 0, y: 0, zoom: 1 }, [], []);
    state.navigateTo(-1);
    expect(state.depth).toBe(0);
    expect(state.activeIndex).toBe(-1);
  });

  it('resolveGeneration increments on push', () => {
    const state = new DrillDownState();
    const gen1 = state.resolveGeneration;
    state.push(makeTarget('L1', MODEL_A), 'n1', { x: 0, y: 0, zoom: 1 }, [], []);
    expect(state.resolveGeneration).toBeGreaterThan(gen1);
  });

  it('resolveGeneration increments on pop (stale resolve guard)', () => {
    const state = new DrillDownState();
    state.push(makeTarget('L1', MODEL_A), 'n1', { x: 0, y: 0, zoom: 1 }, [], []);
    const genAfterPush = state.resolveGeneration;
    state.pop();
    expect(state.resolveGeneration).toBeGreaterThan(genAfterPush);
  });

  it('saved level retains viewport and layout', () => {
    const state = new DrillDownState();
    const viewport = { x: 10, y: 20, zoom: 1.5 };
    const nodes = [{ id: 'n1' }] as any[];
    const edges = [{ id: 'e1' }] as any[];
    state.push(makeTarget('L1', MODEL_A), 'n1', viewport, nodes, edges);
    const level = state.levels[0]!;
    expect(level.viewport).toEqual(viewport);
    expect(level.layoutNodes).toBe(nodes);
    expect(level.layoutEdges).toBe(edges);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-renderer test -- src/drill-down/`
Expected: FAIL — module not found

- [ ] **Step 3: Implement types.ts and drill-down-state.ts**

`types.ts`: `DrillDownConfig`, `DrillDownTarget`, `DrillDownLevel`, `StackLevel` — exact definitions from spec §Part 2.

`drill-down-state.ts`: Pure class, no DOM. Array of `StackLevel`, `push`/`pop`/`navigateTo`, `_resolveGeneration` counter incremented on every mutation.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer test -- src/drill-down/`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add packages/graph-renderer/src/drill-down/
git commit -m "feat(graph-renderer): add DrillDownState with resolve generation guard Refs #456"
```

### Task 4: DrillDownBars — DOM rendering with ARIA

**Files:**
- Create: `packages/graph-renderer/src/drill-down/drill-down-bars.ts`
- Test: `packages/graph-renderer/src/drill-down/drill-down-bars.test.ts`

**Interfaces:**
- Consumes: `DrillDownState` from Task 3
- Produces: `DrillDownBars` — `{ render(container: HTMLElement, state: DrillDownState): void; dispose(): void }`. Dispatches `drill-down-navigate` CustomEvent with `{ depth }` on bar click.

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import { DrillDownBars } from './drill-down-bars.js';
import { DrillDownState } from './drill-down-state.js';
import type { GraphModel } from '@casehubio/graph-core';

const MODEL = { nodes: [], edges: [] } as unknown as GraphModel;

describe('DrillDownBars', () => {
  let container: HTMLDivElement;
  let bars: DrillDownBars;

  afterEach(() => {
    bars?.dispose();
    container?.remove();
  });

  function setup(levels: number = 0): DrillDownState {
    container = document.createElement('div');
    document.body.appendChild(container);
    bars = new DrillDownBars();
    const state = new DrillDownState();
    for (let i = 0; i < levels; i++) {
      state.push({ name: `Level ${i}`, model: MODEL }, `n${i}`, { x: 0, y: 0, zoom: 1 }, [], []);
    }
    bars.render(container, state);
    return state;
  }

  it('renders no bars when stack is empty', () => {
    setup(0);
    expect(container.querySelectorAll('[role="button"]').length).toBe(0);
  });

  it('renders one bar per stack level', () => {
    setup(2);
    expect(container.querySelectorAll('[role="button"]').length).toBe(2);
  });

  it('bar has aria-label with level name', () => {
    setup(1);
    const bar = container.querySelector('[role="button"]') as HTMLElement;
    expect(bar.getAttribute('aria-label')).toBe('Navigate to Level 0');
  });

  it('bars wrapped in role=navigation', () => {
    setup(1);
    const nav = container.querySelector('[role="navigation"]');
    expect(nav).toBeTruthy();
    expect(nav!.getAttribute('aria-label')).toBe('Drill-down breadcrumb');
  });

  it('click bar dispatches drill-down-navigate event', () => {
    setup(2);
    const events: CustomEvent[] = [];
    container.addEventListener('drill-down-navigate', ((e: CustomEvent) => events.push(e)) as EventListener);
    (container.querySelector('[role="button"]') as HTMLElement).click();
    expect(events).toHaveLength(1);
    expect(events[0]!.detail.depth).toBe(0);
  });

  it('bar is ~32px wide', () => {
    setup(1);
    const bar = container.querySelector('[role="button"]') as HTMLElement;
    expect(bar.style.width).toBe('32px');
  });

  it('dispose removes all bars', () => {
    setup(2);
    bars.dispose();
    expect(container.querySelectorAll('[role="button"]').length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests, verify fail, implement, verify pass**

- [ ] **Step 3: Commit**

```bash
git add packages/graph-renderer/src/drill-down/
git commit -m "feat(graph-renderer): add DrillDownBars with ARIA compliance Refs #456"
```

## Batch 3: Drill-Down Wiring

### Task 5: isDrillable injection + ⤢ button

**Files:**
- Modify: `packages/graph-renderer/src/mapping.ts:23-26` — inject `_drillable` into node data
- Modify: `packages/graph-renderer/src/stencil-wrapper.tsx` — render ⤢ button with `.stencil-action` when `data._drillable`
- Test: `packages/graph-renderer/src/stencil-wrapper.test.tsx` (extend existing or create)

**Interfaces:**
- Consumes: `isDrillable` predicate from `DrillDownConfig`
- Produces: `_drillable: true` in node data, ⤢ button with `.stencil-action` class dispatching `graph:drill-down` event

- [ ] **Step 1: Write failing test for _drillable injection in mapping.ts**

Test that `toReactFlowGraph` injects `_drillable: true` when an `isDrillable` predicate is provided and returns true for a node.

- [ ] **Step 2: Add isDrillable parameter to toReactFlowGraph**

Add optional `isDrillable?: (nodeId: string, model: GraphModel) => boolean` parameter. In the node mapping loop (line 23-26), add: `if (isDrillable?.(node.id, model)) data._drillable = true;`

- [ ] **Step 3: Write failing test for ⤢ button rendering**

Test that `StencilNode` renders a button with class `stencil-action` and text `⤢` when `data._drillable === true`, and does NOT render it when `_drillable` is absent.

- [ ] **Step 4: Add ⤢ button to stencil-wrapper.tsx**

After the existing decoration rendering (line 230-232 area), add:
```tsx
{rawData._drillable && (
  <button
    className="stencil-action"
    aria-label={`Drill into ${rawData.label || id}`}
    style={{ position: 'absolute', top: 4, right: 4, zIndex: 3,
             width: 20, height: 20, border: '1px solid #dadce0',
             borderRadius: 4, background: '#fff', cursor: 'pointer',
             fontSize: 12, display: 'flex', alignItems: 'center',
             justifyContent: 'center', color: '#5f6368' }}
    onClick={(e) => {
      e.stopPropagation();
      (e.target as HTMLElement).dispatchEvent(new CustomEvent('graph:drill-down', {
        detail: { nodeId: id }, bubbles: true, composed: true,
      }));
    }}
  >⤢</button>
)}
```

- [ ] **Step 5: Run tests, verify pass, commit**

```bash
git add packages/graph-renderer/src/mapping.ts packages/graph-renderer/src/stencil-wrapper.tsx
git commit -m "feat(graph-renderer): isDrillable injection and drill-down button Refs #456"
```

### Task 6: Wire drill-down into GraphCanvas

**Files:**
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts` — accept `drillDown` prop, create state + bars, wire events, keyboard shortcuts, model swap, viewport save/restore

**Interfaces:**
- Consumes: `DrillDownState` (Task 3), `DrillDownBars` (Task 4), `isDrillable` injection (Task 5)
- Produces: `GraphCanvas.drillDown` prop of type `DrillDownConfig`

- [ ] **Step 1: Write integration tests for drill-down lifecycle**

Test cases:
- Setting `drillDown` config → `isDrillable` injected during `_runLayout` → `_drillable` in node data
- `graph:drill-down` event on container → `config.resolve` called
- Successful resolve → state pushed, bar rendered
- `drill-down-navigate` event → state navigated, bar removed
- Active gesture cancelled on push/pop
- `_layoutGeneration` incremented on push
- Keyboard: Enter on focused drillable node → drill-down triggered
- Keyboard: Escape → pop one level
- Keyboard: Escape suppressed when focus in `<input>`

- [ ] **Step 2: Add drillDown prop to GraphCanvas**

Add `@property({ attribute: false }) drillDown?: DrillDownConfig;`

- [ ] **Step 3: Create DrillDownState + DrillDownBars in connectedCallback**

When `drillDown` config is present:
- Create `_drillDownState = new DrillDownState()`
- Create `_drillDownBars = new DrillDownBars()`
- Listen for `graph:drill-down` → call `config.resolve(nodeId, model)` with generation guard
- Listen for `drill-down-navigate` → call `_drillDownState.navigateTo(depth)`, restore saved level

- [ ] **Step 4: Implement model swap on push/pop**

Push: save current viewport/nodes/edges/generation → swap model → trigger re-layout → render bar
Pop: restore saved viewport/nodes/edges → skip re-layout → remove bar → focus source node

- [ ] **Step 5: Add keyboard shortcuts to existing keydown handler**

Extend `_keyDownHandler` (line 226-231):
- `Enter` on focused drillable node → trigger drill-down
- `Escape` with stack depth > 0 → pop
- `Home` with stack depth > 0 → navigateTo(-1)
- Focus guard: suppress when `activeElement` matches `input, textarea, [contenteditable]`

- [ ] **Step 6: Add aria-live region for level announcements**

Create a visually-hidden `aria-live="polite"` div. Update text on push ("Drilled into {name}") and pop ("Navigated back to {name}").

- [ ] **Step 7: Pass isDrillable to toReactFlowGraph in _runLayout**

In `_runLayout()`, pass `this.drillDown?.isDrillable` to `toReactFlowGraph()`.

- [ ] **Step 8: Run full test suite, commit**

```bash
git add packages/graph-renderer/src/
git commit -m "feat(graph-renderer): wire DrillDownStack into GraphCanvas with keyboard and ARIA Refs #456"
```

## Batch 4: Examples Showcase

### Task 7: Nested Pipeline example (3-level drill-down)

**Files:**
- Create: `examples/samples/Graph Editing/Nested Pipeline.page.yaml`
- Modify: `examples/src/casehub-entry.ts` — export drill-down helpers if needed
- Modify: `examples/scripts/generate-samples.js` — already handles `.page.yaml` (fixed earlier)

**Interfaces:**
- Consumes: `GraphCanvas` with `drillDown` config from graph-renderer
- Produces: Example page demonstrating 3-level nested pipeline

- [ ] **Step 1: Create the example YAML and static models**

Pipeline → Stage → Task:
- Level 0: Ingest → Transform → Export (3 nodes, 2 edges, all drillable)
- Level 1 (Ingest): Fetch → Parse → Validate → Store (4 nodes, 3 edges, Parse and Validate drillable)
- Level 2 (Parse): Tokenize → Schema Check → Normalize (3 nodes, 2 edges, not drillable)

The example YAML references a custom `pages-graph-canvas` component with a `drillDown` config using pre-built static models. The `resolve` callback returns the appropriate model based on `nodeId`.

- [ ] **Step 2: Regenerate samples and verify**

```bash
yarn workspace @casehubio/pages-examples generate-samples
yarn workspace @casehubio/pages-examples build:bundle:dev
yarn workspace @casehubio/pages-examples copy-samples
```

- [ ] **Step 3: Verify in browser — drill 3 levels, navigate back via bars**

- [ ] **Step 4: Commit**

```bash
git add examples/
git commit -m "feat(examples): add 3-level nested pipeline drill-down showcase Refs #456"
```

## References

- [2026-09-20-pointer-pipeline-design.md] — design spec (post-review)
- [2026-09-20-pointer-pipeline-decisions.md] — D1-D5 design decisions
- [graph-renderer/src/editing/node-move-coordinator.ts] — current drag logic (refactored in Task 2)
- [graph-renderer/src/bridge/GraphCanvas.ts:177-206] — current pointerdown (replaced in Task 2)
- [graph-renderer/src/stencil-wrapper.tsx:237-240] — full-node Handle
- [graph-renderer/src/mapping.ts:23-26] — node data injection
- [blocks-ui/components/diagram-workbench/src/diagram-workbench.ts:62-66] — current drill-down stack
- [GitHub #456] — focal issue
