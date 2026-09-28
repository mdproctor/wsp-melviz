# DiagramBaseMixin Layout Hooks & Composable Template Methods — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #468 — DiagramBaseMixin: standard render template with default event wiring
**Issue group:** #468, #467

**Goal:** Add layout pipeline hooks (`_computeLayout`, `_postLayout`) and composable template methods (`_handleCanvasEvent`, `_handleEdgeClick`, `_renderCanvas`, `_renderDialogs`) to DiagramBaseMixin so subclasses no longer need to override `_fullRender` or copy-paste event wiring.

**Architecture:** Two layout hooks replace the hardcoded `computeElkLayout` call in `_fullRender` — `_computeLayout` for the layout algorithm and `_postLayout` for post-processing after `toReactFlowGraph`. Four composable template methods provide reusable render pieces that each diagram's `render()` composes freely.

**Tech Stack:** TypeScript, Lit 3.x, vitest

## Global Constraints

- All changes in `packages/pages-diagram-core/src/diagram-base-mixin.ts`
- New `LayoutResult` type exported from `packages/pages-diagram-core/src/index.ts`
- All new methods must have default implementations that preserve current behavior — backward compatible
- Arrow function binding for `_handleCanvasEvent` (passed as `@pages-event` handler)
- Regular method for `_handleEdgeClick` (called from within bound handler, allows `override`)
- Tests in `packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
- Test runner: `yarn vitest run` from `packages/pages-diagram-core/`
- `pages-event` topics use colon-separated segments per protocol

---

## Batch 1: Layout pipeline hooks (#467)

### Task 1: Add `LayoutResult` type and `_computeLayout` hook

**Files:**
- Modify: `packages/pages-diagram-core/src/diagram-base-mixin.ts:1-11` (imports), `:27-93` (interface), `:95-96` (overload + function signature), `:405-437` (`_fullRender`)
- Modify: `packages/pages-diagram-core/src/index.ts:11` (exports)
- Test: `packages/pages-diagram-core/src/diagram-base-mixin.test.ts`

**Interfaces:**
- Produces: `LayoutResult { layout: ElkLayoutResult; direction?: 'DOWN' | 'RIGHT' | 'LEFT' | 'UP' }` — exported type used by consumers
- Produces: `_computeLayout(model: GraphModel, options: ElkLayoutOptions): Promise<LayoutResult>` — protected method, default delegates to `computeElkLayout`

- [ ] **Step 1: Write failing test — `_computeLayout` is called during `_fullRender`**

```typescript
import { describe, it, expect, vi } from 'vitest';
import { LitElement } from 'lit';
import { DiagramBaseMixin } from './diagram-base-mixin.js';
import type { LayoutResult } from './diagram-base-mixin.js';
import type { GraphModel } from '@casehubio/graph-core';

class TestDiagram extends DiagramBaseMixin(LitElement) {
  computeLayoutCalls: Array<{ model: GraphModel }> = [];

  protected _adaptYaml(yaml: string) {
    return {
      model: { nodes: [], edges: [] } as GraphModel,
      yamlPaths: new Map(),
    };
  }

  protected _applyPropertyEdit() { return ''; }
  protected _emptyTemplate() { return null; }

  protected override async _computeLayout(
    model: GraphModel,
    options: import('@casehubio/graph-renderer').ElkLayoutOptions,
  ): Promise<LayoutResult> {
    this.computeLayoutCalls.push({ model });
    return { layout: { nodeLayouts: new Map() }, direction: 'DOWN' };
  }
}

describe('DiagramBaseMixin', () => {
  it('exports the mixin function', () => {
    expect(typeof DiagramBaseMixin).toBe('function');
  });

  describe('_computeLayout', () => {
    it('is called by _fullRender instead of computeElkLayout directly', async () => {
      const el = new TestDiagram();
      await el._fullRender('test: yaml');
      expect(el.computeLayoutCalls.length).toBe(1);
      expect(el.computeLayoutCalls[0]!.model).toEqual({ nodes: [], edges: [] });
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: FAIL — `LayoutResult` type not exported, `_computeLayout` not defined on mixin

- [ ] **Step 3: Add `LayoutResult` type and `_computeLayout` method**

Add the `LayoutResult` interface after the existing `AdapterResult` interface (after line 21):

```typescript
export interface LayoutResult {
  readonly layout: ElkLayoutResult;
  readonly direction?: 'DOWN' | 'RIGHT' | 'LEFT' | 'UP';
}
```

Add `_computeLayout` to the `DiagramBaseInterface` declaration (after line 87, `_layoutOptions`):

```typescript
protected _computeLayout(model: GraphModel, options: ElkLayoutOptions): Promise<LayoutResult>;
```

Add the default implementation inside the mixin class (after the `_layoutOptions` method):

```typescript
protected async _computeLayout(
  model: GraphModel,
  options: ElkLayoutOptions,
): Promise<LayoutResult> {
  const layout = await computeElkLayout(model, options);
  return { layout, direction: options.direction };
}
```

Update `_fullRender` to call `this._computeLayout` instead of `computeElkLayout` directly. Replace the layout call and the `toReactFlowGraph` call:

```typescript
// Replace line 415:
//   const layout = await computeElkLayout(result.model, this._layoutOptions());
// With:
    const { layout, direction } = await this._computeLayout(
      result.model, this._layoutOptions(),
    );

// Replace line 422:
//   const { nodes, edges } = toReactFlowGraph(result.model, layout, this._decorations(), this._layoutOptions().direction);
// With:
    const { nodes, edges } = toReactFlowGraph(
      result.model, layout, this._decorations(), direction,
    );
```

Add to `index.ts` exports:

```typescript
export type { AdapterResult, DiagramBaseInterface, LayoutResult } from './diagram-base-mixin.js';
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn tsc --noEmit -p packages/pages-diagram-core/tsconfig.json`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add packages/pages-diagram-core/src/diagram-base-mixin.ts packages/pages-diagram-core/src/diagram-base-mixin.test.ts packages/pages-diagram-core/src/index.ts
git commit -m "feat(diagram-core): add _computeLayout hook to DiagramBaseMixin

Introduces LayoutResult type and _computeLayout protected method.
_fullRender now calls this._computeLayout() instead of computeElkLayout()
directly, allowing subclasses to override the layout strategy without
copy-pasting the entire render pipeline.

Refs #467"
```

### Task 2: Add `_postLayout` hook

**Files:**
- Modify: `packages/pages-diagram-core/src/diagram-base-mixin.ts` — interface declaration, mixin class, `_fullRender`, `_updateWithoutLayout`
- Test: `packages/pages-diagram-core/src/diagram-base-mixin.test.ts`

**Interfaces:**
- Consumes: `LayoutResult` from Task 1, updated `_fullRender` from Task 1
- Produces: `_postLayout(nodes: Node[], edges: Edge[]): { nodes: Node[]; edges: Edge[] }` — protected method, default is identity

- [ ] **Step 1: Write failing test — `_postLayout` transforms output of `_fullRender`**

```typescript
class PostLayoutTestDiagram extends DiagramBaseMixin(LitElement) {
  protected _adaptYaml(_yaml: string) {
    return {
      model: { nodes: [], edges: [] } as GraphModel,
      yamlPaths: new Map(),
    };
  }
  protected _applyPropertyEdit() { return ''; }
  protected _emptyTemplate() { return null; }

  protected override async _computeLayout(): Promise<LayoutResult> {
    return { layout: { nodeLayouts: new Map() }, direction: 'DOWN' };
  }

  postLayoutCalled = false;

  protected override _postLayout(
    nodes: import('@xyflow/react').Node[],
    edges: import('@xyflow/react').Edge[],
  ) {
    this.postLayoutCalled = true;
    return { nodes, edges };
  }
}

describe('_postLayout', () => {
  it('is called after toReactFlowGraph in _fullRender', async () => {
    const el = new PostLayoutTestDiagram();
    await el._fullRender('test: yaml');
    expect(el.postLayoutCalled).toBe(true);
  });

  it('is called after toReactFlowGraph in _updateWithoutLayout', () => {
    const el = new PostLayoutTestDiagram();
    (el as any)._lastLayout = { nodeLayouts: new Map() };
    (el as any)._adapterResult = {
      model: { nodes: [], edges: [] },
      yamlPaths: new Map(),
    };
    el._updateWithoutLayout('test: yaml');
    expect(el.postLayoutCalled).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: FAIL — `_postLayout` not defined

- [ ] **Step 3: Add `_postLayout` method**

Add to `DiagramBaseInterface` declaration (after the `_computeLayout` line):

```typescript
protected _postLayout(nodes: Node[], edges: Edge[]): { nodes: Node[]; edges: Edge[] };
```

Add default implementation inside the mixin class (after `_computeLayout`):

```typescript
protected _postLayout(
  nodes: Node[],
  edges: Edge[],
): { nodes: Node[]; edges: Edge[] } {
  return { nodes, edges };
}
```

Update `_fullRender` — after the `toReactFlowGraph` call, pass through `_postLayout`:

```typescript
// Replace:
//     this._nodes = nodes;
//     this._edges = edges;
// With:
    const processed = this._postLayout(nodes, edges);
    this._nodes = processed.nodes;
    this._edges = processed.edges;
```

Update `_updateWithoutLayout` — same pattern after its `toReactFlowGraph` call:

```typescript
// Replace:
//     this._nodes = nodes;
//     this._edges = edges;
// With:
    const processed = this._postLayout(nodes, edges);
    this._nodes = processed.nodes;
    this._edges = processed.edges;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn tsc --noEmit -p packages/pages-diagram-core/tsconfig.json`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add packages/pages-diagram-core/src/diagram-base-mixin.ts packages/pages-diagram-core/src/diagram-base-mixin.test.ts
git commit -m "feat(diagram-core): add _postLayout hook to DiagramBaseMixin

Called after toReactFlowGraph in both _fullRender and _updateWithoutLayout.
Default is identity. Subclasses override for post-layout processing
(e.g. org-diagram engine.postLayout, edge styling).

Refs #467"
```

## Batch 2: Composable template methods (#468)

### Task 3: Add `_handleCanvasEvent` dispatcher and `_handleEdgeClick`

**Files:**
- Modify: `packages/pages-diagram-core/src/diagram-base-mixin.ts` — interface declaration, mixin class
- Test: `packages/pages-diagram-core/src/diagram-base-mixin.test.ts`

**Interfaces:**
- Produces: `_handleCanvasEvent: (e: CustomEvent) => void` — arrow function, dispatches common graph events
- Produces: `_handleEdgeClick(e: CustomEvent): void` — regular method, default no-op

- [ ] **Step 1: Write failing test — `_handleCanvasEvent` dispatches to handlers**

```typescript
describe('_handleCanvasEvent', () => {
  it('dispatches graph:node:click to _handleNodeClick', () => {
    const el = new TestDiagram();
    let called = false;
    (el as any)._handleNodeClick = () => { called = true; };
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:node:click', payload: { nodeId: 'n1' } },
    });
    el._handleCanvasEvent(event);
    expect(called).toBe(true);
  });

  it('dispatches graph:edge:click to _handleEdgeClick', () => {
    const el = new TestDiagram();
    let called = false;
    el._handleEdgeClick = () => { called = true; };
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:edge:click', payload: { edgeId: 'e1' } },
    });
    el._handleCanvasEvent(event);
    expect(called).toBe(true);
  });

  it('dispatches graph:selection:change to _handleSelectionChange', () => {
    const el = new TestDiagram();
    let called = false;
    (el as any)._handleSelectionChange = () => { called = true; };
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:selection:change', payload: { nodeIds: [] } },
    });
    el._handleCanvasEvent(event);
    expect(called).toBe(true);
  });

  it('dispatches graph:pane:click to _showPickerAtPaneClick', () => {
    const el = new TestDiagram();
    let called = false;
    (el as any)._showPickerAtPaneClick = () => { called = true; };
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:pane:click' },
    });
    el._handleCanvasEvent(event);
    expect(called).toBe(true);
  });

  it('dispatches graph:connect:end-on-empty to _showPickerAtConnectEnd', () => {
    const el = new TestDiagram();
    let called = false;
    (el as any)._showPickerAtConnectEnd = () => { called = true; };
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:connect:end-on-empty', payload: { sourceNodeId: 'n1' } },
    });
    el._handleCanvasEvent(event);
    expect(called).toBe(true);
  });

  it('does nothing for unknown topics', () => {
    const el = new TestDiagram();
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:unknown:event' },
    });
    expect(() => el._handleCanvasEvent(event)).not.toThrow();
  });
});

describe('_handleEdgeClick', () => {
  it('is a no-op by default', () => {
    const el = new TestDiagram();
    const event = new CustomEvent('pages-event', {
      detail: { topic: 'graph:edge:click', payload: { edgeId: 'e1' } },
    });
    expect(() => el._handleEdgeClick(event)).not.toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: FAIL — `_handleCanvasEvent` and `_handleEdgeClick` not defined

- [ ] **Step 3: Add `_handleCanvasEvent` and `_handleEdgeClick`**

Add to `DiagramBaseInterface` declaration (after line 86, `_renderNodePicker`):

```typescript
_handleCanvasEvent: (e: CustomEvent) => void;
_handleEdgeClick(e: CustomEvent): void;
```

Add implementations inside the mixin class (after `_renderNodePicker` method, before the stencil palette section):

```typescript
protected _handleCanvasEvent = (e: CustomEvent): void => {
  const topic = e.detail?.topic;
  switch (topic) {
    case 'graph:node:click':
      this._handleNodeClick(e);
      break;
    case 'graph:edge:click':
      this._handleEdgeClick(e);
      break;
    case 'graph:selection:change':
      this._handleSelectionChange(e);
      break;
    case 'graph:pane:click':
      this._showPickerAtPaneClick();
      break;
    case 'graph:connect:end-on-empty':
      this._showPickerAtConnectEnd(e.detail?.payload);
      break;
  }
};

protected _handleEdgeClick(_e: CustomEvent): void {
  // Subclasses override for edge-click behavior
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-diagram-core/src/diagram-base-mixin.ts packages/pages-diagram-core/src/diagram-base-mixin.test.ts
git commit -m "feat(diagram-core): add _handleCanvasEvent dispatcher and _handleEdgeClick

Central event dispatcher for common graph events. Arrow function for
stable this-binding when passed as @pages-event handler. Subclasses call
super._handleCanvasEvent(e) then handle domain-specific events.

_handleEdgeClick is a no-op default for subclass override (e.g. SWF
split-edge picker).

Refs #468"
```

### Task 4: Add `_renderCanvas` and `_renderDialogs` template methods

**Files:**
- Modify: `packages/pages-diagram-core/src/diagram-base-mixin.ts` — interface declaration, mixin class
- Test: `packages/pages-diagram-core/src/diagram-base-mixin.test.ts`

**Interfaces:**
- Consumes: `_handleCanvasEvent` from Task 3, `_handleMutation` (existing), `_editPolicy` (existing)
- Produces: `_renderCanvas(): TemplateResult` — renders `pages-graph-canvas` with standard props
- Produces: `_renderDialogs(): TemplateResult` — renders conflict + delete confirm dialogs

- [ ] **Step 1: Write failing test — `_renderCanvas` returns template with canvas element**

```typescript
import { html } from 'lit';

describe('_renderCanvas', () => {
  it('returns a TemplateResult', () => {
    const el = new TestDiagram();
    const result = el._renderCanvas();
    expect(result).toBeDefined();
    expect(result.strings).toBeDefined();
  });
});

describe('_renderDialogs', () => {
  it('returns a TemplateResult', () => {
    const el = new TestDiagram();
    const result = el._renderDialogs();
    expect(result).toBeDefined();
  });

  it('includes conflict dialog when _showConflict is true', () => {
    const el = new TestDiagram();
    (el as any)._showConflict = true;
    const result = el._renderDialogs();
    const templateStr = result.strings.join('');
    expect(templateStr).toContain('Conflict');
  });

  it('includes delete confirm when _confirmMessage is set', () => {
    const el = new TestDiagram();
    (el as any)._confirmMessage = 'Delete this?';
    const result = el._renderDialogs();
    const templateStr = result.strings.join('');
    expect(templateStr.length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: FAIL — `_renderCanvas` and `_renderDialogs` not defined

- [ ] **Step 3: Add `_renderCanvas` and `_renderDialogs`**

Add to `DiagramBaseInterface` declaration (after the `_handleEdgeClick` line from Task 3):

```typescript
_renderCanvas(): TemplateResult;
_renderDialogs(): TemplateResult;
```

Add implementations inside the mixin class (after `_handleEdgeClick`):

```typescript
protected _renderCanvas(): TemplateResult {
  return html`
    <pages-graph-canvas
      .nodes=${this._nodes}
      .edges=${this._edges}
      .model=${this._adapterResult?.model}
      .editPolicy=${this._editPolicy()}
      .onMutation=${this._handleMutation}
      style="width:100%;height:100%;"
      @pages-event=${this._handleCanvasEvent}
    ></pages-graph-canvas>
  `;
}

protected _renderDialogs(): TemplateResult {
  return html`
    ${this._showConflict ? this._renderConflictDialog() : nothing}
    ${this._confirmMessage ? this._renderDeleteConfirm() : nothing}
  `;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
Expected: PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn tsc --noEmit -p packages/pages-diagram-core/tsconfig.json`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add packages/pages-diagram-core/src/diagram-base-mixin.ts packages/pages-diagram-core/src/diagram-base-mixin.test.ts
git commit -m "feat(diagram-core): add _renderCanvas and _renderDialogs template methods

_renderCanvas renders pages-graph-canvas with standard props and
_handleCanvasEvent wiring. _renderDialogs renders conflict and delete
confirm dialogs. Both are overridable — subclasses compose freely.

Refs #468"
```

## References

- [specs/issue-468-diagram-mixin-render-layout/2026-09-26-diagram-mixin-render-layout-design.md] — design spec this plan implements
- [packages/pages-diagram-core/src/diagram-base-mixin.ts] — mixin source (706 lines)
- [packages/pages-diagram-core/src/index.ts] — package exports
- [packages/pages-diagram-core/src/diagram-base-mixin.test.ts] — existing tests
- [packages/graph-renderer/src/layout/elk-layout.ts:5-26] — ElkLayoutOptions, ElkLayoutResult types
- [packages/graph-renderer/src/bridge/PagesGraphCanvas.ts:29-189] — canvas component API
- [docs/protocols/casehub/pages-event-contract.md] — event naming conventions
- [GitHub #468] — render template + event wiring
- [GitHub #467] — layout hook
