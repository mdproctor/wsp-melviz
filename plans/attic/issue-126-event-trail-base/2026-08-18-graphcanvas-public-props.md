# GraphCanvas Public Props + Stencil Constraints + RF CSS — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #320 — graph-renderer: GraphCanvas missing public nodes/edges properties
**Issue group:** #320

**Goal:** Make GraphCanvas consumable by blocks-ui DiagramBaseMixin — expose public nodes/edges properties, constrain stencil dimensions, and bundle React Flow CSS into the shadow-aware injection.

**Architecture:** Three independent fixes in `graph-renderer`. Fix 1 adds Lit `@property` for direct node/edge setting. Fix 2 applies ELK dimensions as constraints on stencil wrappers. Fix 3 bundles React Flow base CSS as a raw string into `getIsolationCSS()` so it reaches shadow roots.

**Tech Stack:** TypeScript, Lit, React 18, @xyflow/react v12, Vitest

## Global Constraints

- All Web Components use Lit per protocol PP-20260705-c7687d
- `injectIsolationStyles(host?)` API from #319 is the CSS injection mechanism — no parallel injection paths
- React Flow CSS is third-party and static — bundling as a raw string is acceptable

---

## Batch 1: GraphCanvas API and rendering fixes

### Task 1: Add public nodes/edges properties to GraphCanvas

**Files:**
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts`
- Modify: `packages/graph-renderer/src/bridge/bridge.test.ts`

**Interfaces:**
- Produces: `GraphCanvas.nodes: Node[] | undefined` — public `@property`
- Produces: `GraphCanvas.edges: Edge[] | undefined` — public `@property`

- [ ] **Step 1: Write failing tests for direct nodes/edges setting**

Add to `packages/graph-renderer/src/bridge/bridge.test.ts`:

```typescript
describe('direct nodes/edges properties', () => {
  it('renders nodes set via property without running layout', async () => {
    const el = document.createElement('pages-graph-canvas') as GraphCanvas;
    document.body.appendChild(el);

    const testNodes: Node[] = [{
      id: 'n1',
      type: 'default',
      position: { x: 100, y: 200 },
      data: { label: 'Test' },
    }];
    const testEdges: Edge[] = [];

    el.nodes = testNodes;
    el.edges = testEdges;
    await el.updateComplete;

    expect(el.nodes).toEqual(testNodes);
    el.remove();
  });

  it('model property takes priority over direct nodes/edges', async () => {
    const el = document.createElement('pages-graph-canvas') as GraphCanvas;
    document.body.appendChild(el);

    el.nodes = [{ id: 'direct', type: 'default', position: { x: 0, y: 0 }, data: {} }];
    await el.updateComplete;

    // Setting model triggers _runLayout which overwrites internal state
    el.model = { nodes: [], edges: [] };
    await el.updateComplete;

    // model path ran — direct nodes were overwritten
    expect(el.nodes).toBeDefined();
    el.remove();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run src/bridge/bridge.test.ts`
Expected: FAIL — `nodes` and `edges` are not reactive properties

- [ ] **Step 3: Implement public properties**

In `packages/graph-renderer/src/bridge/GraphCanvas.ts`, add the public properties alongside the existing private state:

```typescript
@property({ attribute: false }) nodes: Node[] | undefined;
@property({ attribute: false }) edges: Edge[] | undefined;
```

Update the `updated()` method to handle both input paths:

```typescript
override updated(changed: Map<string, unknown>): void {
  if (changed.has('model') || changed.has('layoutOptions')) {
    void this._runLayout();
  } else if (changed.has('nodes') || changed.has('edges')) {
    this._nodes = this.nodes ?? [];
    this._edges = this.edges ?? [];
  }
  this._renderReact();
}
```

- [ ] **Step 4: Run all bridge tests**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run src/bridge/bridge.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/bridge/GraphCanvas.ts packages/graph-renderer/src/bridge/bridge.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): add public nodes/edges properties to GraphCanvas

Adds @property for direct node/edge setting, bypassing internal ELK
layout. Model path takes priority when both are set.

Refs #320"
```

### Task 2: Constrain stencil wrapper dimensions

**Files:**
- Modify: `packages/graph-renderer/src/stencil-wrapper.tsx`
- Modify: `packages/graph-renderer/src/stencil-wrapper.test.tsx`

**Interfaces:**
- Consumes: React Flow `NodeProps.width` and `NodeProps.height` (set by ELK layout)

- [ ] **Step 1: Write failing test for dimension constraints**

Add to `packages/graph-renderer/src/stencil-wrapper.test.tsx`:

```typescript
it('applies width/height constraints from NodeProps', () => {
  registerGrammar({
    type: 'sized',
    connections: {
      inbound: { min: 0, max: 5, allowedFrom: [] },
      outbound: { min: 0, max: 5, allowedTo: [] },
    },
  });
  const renderFn: StencilRenderFn = () => html`<div class="content">sized</div>`;
  const Component = createStencilNodeComponent(renderFn);

  const { container, unmount } = mountWithProps(Component, {
    ...defaultNodeProps,
    type: 'sized',
    width: 200,
    height: 100,
  });

  const wrapper = container.querySelector('.stencil-decoration-wrapper') as HTMLElement;
  expect(wrapper).not.toBeNull();
  expect(wrapper.style.maxWidth).toBe('200px');
  expect(wrapper.style.maxHeight).toBe('100px');
  expect(wrapper.style.overflow).toBe('hidden');

  unmount();
});

it('does not apply constraints when width/height are absent', () => {
  registerGrammar({
    type: 'unsized',
    connections: {
      inbound: { min: 0, max: 5, allowedFrom: [] },
      outbound: { min: 0, max: 5, allowedTo: [] },
    },
  });
  const renderFn: StencilRenderFn = () => html`<div class="content">unsized</div>`;
  const Component = createStencilNodeComponent(renderFn);

  const { container, unmount } = mountWithProps(Component, {
    ...defaultNodeProps,
    type: 'unsized',
  });

  const wrapper = container.querySelector('.stencil-decoration-wrapper') as HTMLElement;
  expect(wrapper).not.toBeNull();
  expect(wrapper.style.maxWidth).toBe('');
  expect(wrapper.style.maxHeight).toBe('');

  unmount();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run src/stencil-wrapper.test.tsx`
Expected: FAIL — wrapper doesn't apply width/height

- [ ] **Step 3: Implement dimension constraints**

In `packages/graph-renderer/src/stencil-wrapper.tsx`, update the `StencilNode` function. Destructure `width` and `height` from `NodeProps` and apply as constraints:

```tsx
function StencilNode({ id, type, data, parentId, width, height }: NodeProps): React.JSX.Element {
  // ... existing ref, grammar, rawData, decoration code unchanged ...

  const sizeStyle: React.CSSProperties = {};
  if (width != null) {
    sizeStyle.maxWidth = width;
    sizeStyle.width = width;
  }
  if (height != null) {
    sizeStyle.maxHeight = height;
    sizeStyle.height = height;
  }
  if (width != null || height != null) {
    sizeStyle.overflow = 'hidden';
  }

  return (
    <>
      {grammar?.connections.inbound.max !== 0 && (
        <Handle type="target" position={Position.Top} />
      )}
      <div
        className="stencil-decoration-wrapper"
        style={{ position: 'relative', ...borderStyle, ...sizeStyle }}
        title={decoration?.tooltip}
      >
        {decoration?.badge && <DecorationBadge badge={decoration.badge} />}
        {decoration?.overlay && <DecorationOverlay overlay={decoration.overlay} />}
        <div ref={containerRef} />
      </div>
      {grammar?.connections.outbound.max !== 0 && (
        <Handle type="source" position={Position.Bottom} />
      )}
    </>
  );
}
```

- [ ] **Step 4: Run stencil-wrapper tests**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run src/stencil-wrapper.test.tsx`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/stencil-wrapper.tsx packages/graph-renderer/src/stencil-wrapper.test.tsx
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): constrain stencil wrapper to ELK dimensions

Apply width/height from NodeProps as max constraints with overflow
hidden. Prevents stencil content from expanding beyond ELK layout.

Refs #320"
```

### Task 3: Bundle React Flow CSS into getIsolationCSS()

**Files:**
- Modify: `packages/graph-renderer/src/bridge/css-isolation.ts`
- Modify: `packages/graph-renderer/src/bridge/css-isolation.test.ts`
- Modify: `packages/graph-renderer/src/bridge/ReactFlowApp.tsx`
- Create: `packages/graph-renderer/src/raw-css.d.ts`

**Interfaces:**
- Consumes: `getIsolationCSS()` from css-isolation.ts (extended to include RF CSS)
- Consumes: `@xyflow/react/dist/style.css?raw` — Vite raw import

- [ ] **Step 1: Create type declaration for ?raw imports**

Create `packages/graph-renderer/src/raw-css.d.ts`:

```typescript
declare module '*.css?raw' {
  const css: string;
  export default css;
}
```

- [ ] **Step 2: Write failing test for React Flow CSS inclusion**

Add to `packages/graph-renderer/src/bridge/css-isolation.test.ts`:

```typescript
it('includes React Flow base styles in isolation CSS', () => {
  const css = getIsolationCSS();
  expect(css).toContain('.react-flow');
  expect(css).toContain('.react-flow__node');
  expect(css).toContain('.react-flow__edge');
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run src/bridge/css-isolation.test.ts`
Expected: FAIL — `getIsolationCSS()` doesn't include `.react-flow__node` or `.react-flow__edge`

- [ ] **Step 4: Import React Flow CSS as raw string and include in getIsolationCSS()**

In `packages/graph-renderer/src/bridge/css-isolation.ts`, add the raw import at the top:

```typescript
import reactFlowCSS from '@xyflow/react/dist/style.css?raw';
```

Update `getIsolationCSS()` to prepend the React Flow CSS:

```typescript
export function getIsolationCSS(): string {
  const pluginStyles = getRegisteredStyles();

  return `
${reactFlowCSS}

.${DIAGRAM_ROOT_CLASS} {
  all: initial;
  display: block;
  position: relative;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
}

.${DIAGRAM_ROOT_CLASS} * {
  all: revert;
}

${pluginStyles}

.react-flow__controls {
  background: var(--pages-neutral-1, #fafafa);
  border: 1px solid var(--pages-neutral-4, #ccc);
  border-radius: var(--pages-radius-md, 8px);
}
.react-flow__controls-button {
  background: var(--pages-neutral-1, #fafafa);
  border-bottom: 1px solid var(--pages-neutral-3, #ddd);
  color: var(--pages-text-primary, #111);
  fill: var(--pages-text-primary, #111);
}
.react-flow__controls-button:hover {
  background: var(--pages-neutral-2, #f0f0f0);
}
`.trim();
}
```

- [ ] **Step 5: Remove CSS import from ReactFlowApp.tsx**

In `packages/graph-renderer/src/bridge/ReactFlowApp.tsx`, remove line 17:

```typescript
// DELETE: import '@xyflow/react/dist/style.css';
```

- [ ] **Step 6: Handle test environment (vitest mock for ?raw)**

The `?raw` import is a Vite feature not available in vitest's node environment by default. If the test fails due to the import, add a vitest mock in the test file:

```typescript
vi.mock('@xyflow/react/dist/style.css?raw', () => ({
  default: '.react-flow { display: flex; } .react-flow__node { position: absolute; } .react-flow__edge { position: absolute; }',
}));
```

Add this mock at the top of `css-isolation.test.ts` after the existing imports, before the describe block.

- [ ] **Step 7: Run all css-isolation tests**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run src/bridge/css-isolation.test.ts`
Expected: ALL PASS

- [ ] **Step 8: Run full graph-renderer test suite**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-renderer run test -- --run`
Expected: ALL PASS

- [ ] **Step 9: Type check**

Run: `GH_PACKAGES_TOKEN=dummy yarn typecheck`
Expected: No new errors (pre-existing errors in PagesEventTimeline.test.ts are acceptable)

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/bridge/css-isolation.ts packages/graph-renderer/src/bridge/css-isolation.test.ts packages/graph-renderer/src/bridge/ReactFlowApp.tsx packages/graph-renderer/src/raw-css.d.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): bundle React Flow CSS into shadow-aware injection

Import @xyflow/react/dist/style.css as raw string and include in
getIsolationCSS(). Removes the CSS import from ReactFlowApp.tsx.
React Flow base styles now reach shadow DOM hosts.

Closes #320"
```

---

## References

- [2026-08-18-graphcanvas-public-props-design.md] — design spec this plan implements
- [packages/graph-renderer/src/bridge/GraphCanvas.ts] — primary fix site (properties)
- [packages/graph-renderer/src/stencil-wrapper.tsx] — stencil dimension constraints
- [packages/graph-renderer/src/bridge/css-isolation.ts] — CSS injection
- [packages/graph-renderer/src/bridge/ReactFlowApp.tsx:17] — CSS import to remove
- [issue-259-graph-phase0/2026-08-02-phase0-react-flow-lit-bridge-design.md] — original bridge design
- [issue-265-graph-renderer/2026-08-03-stencil-wrapper-pipeline-design.md] — stencil rendering pipeline
- [GE-20260818-f0257a] — shadow-aware CSS injection technique
- [PP-20260705-c7687d] — Web component strategy protocol
- [GitHub #319] — shadow DOM CSS isolation (prerequisite)
- [GitHub #320] — this issue
