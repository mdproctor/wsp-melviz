# Node Decoration Pills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #404 — graph-core: add pills array to NodeDecoration for runtime overlay labels
**Issue group:** #404

**Goal:** Add an optional `pills` array to `NodeDecoration` and render pills as colored labels along the bottom edge of graph nodes.

**Architecture:** Extend the `NodeDecoration` interface in graph-core with an inline readonly `pills` array type. Add a `DecorationPills` React component in graph-renderer's `stencil-wrapper.tsx` following the existing `DecorationBadge` pattern — absolute-positioned at the bottom edge of `.stencil-decoration-wrapper`.

**Tech Stack:** TypeScript, React, Lit-html, Vitest

## Global Constraints

- graph-core is a pure data package — no callbacks, no framework dependencies
- All NodeDecoration fields are readonly and optional
- Existing consumers must be unaffected (additive change only)

---

## Batch 1: Pills type and rendering

### Task 1: Add pills to NodeDecoration and test the type

**Files:**
- Modify: `packages/graph-core/src/model.ts:22-38`
- Modify: `packages/graph-core/src/model.test.ts`

**Interfaces:**
- Produces: `NodeDecoration.pills` — `readonly { readonly text: string; readonly color: string; readonly icon?: string; }[]`

- [ ] **Step 1: Write the failing test**

Add to `packages/graph-core/src/model.test.ts` inside the `NodeDecoration` describe block:

```typescript
it('accepts decoration with pills', () => {
  const decoration: NodeDecoration = {
    pills: [
      { text: '0.92', color: '#16a34a', icon: '✓' },
      { text: '45ms', color: '#2563eb' },
    ],
  };
  expect(decoration.pills).toHaveLength(2);
  expect(decoration.pills![0]!.text).toBe('0.92');
  expect(decoration.pills![0]!.icon).toBe('✓');
  expect(decoration.pills![1]!.icon).toBeUndefined();
});

it('accepts decoration with pills and badge together', () => {
  const decoration: NodeDecoration = {
    badge: { icon: 'play', color: 'green' },
    pills: [{ text: 'SLA: 2h', color: '#dc2626' }],
  };
  expect(decoration.badge?.icon).toBe('play');
  expect(decoration.pills).toHaveLength(1);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/graph-core run test -- --run 2>&1 | tail -10`
Expected: FAIL — `pills` does not exist on type `NodeDecoration`

- [ ] **Step 3: Add pills to NodeDecoration interface**

In `packages/graph-core/src/model.ts`, add after `readonly tooltip?: string;`:

```typescript
readonly pills?: readonly {
  readonly text: string;
  readonly color: string;
  readonly icon?: string;
}[];
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/graph-core run test -- --run 2>&1 | tail -10`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-core/src/model.ts packages/graph-core/src/model.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#404): add pills array to NodeDecoration interface Refs #404"
```

### Task 2: Render pills in stencil-wrapper

**Files:**
- Modify: `packages/graph-renderer/src/stencil-wrapper.tsx`
- Modify: `packages/graph-renderer/src/stencil-wrapper.test.tsx`

**Interfaces:**
- Consumes: `NodeDecoration.pills` from Task 1

- [ ] **Step 1: Write the failing tests**

Add to `packages/graph-renderer/src/stencil-wrapper.test.tsx` inside the `decoration rendering` describe block:

```typescript
it('renders pills when decoration has pills', () => {
  const renderFn: StencilRenderFn = () => html`<div>node</div>`;
  const Component = createStencilNodeComponent(renderFn);
  const decoration: NodeDecoration = {
    pills: [
      { text: '0.92', color: '#16a34a', icon: '✓' },
      { text: '45ms', color: '#2563eb' },
    ],
  };
  const { container, unmount } = mountWithProps(Component, {
    ...defaultNodeProps,
    data: { _decoration: decoration },
  });
  const pillsContainer = container.querySelector('.stencil-decoration-pills');
  expect(pillsContainer).not.toBeNull();
  const pills = pillsContainer!.querySelectorAll('.stencil-pill');
  expect(pills).toHaveLength(2);
  expect(pills[0]!.querySelector('.stencil-pill-icon')?.textContent).toBe('✓');
  expect(pills[0]!.querySelector('.stencil-pill-text')?.textContent).toBe('0.92');
  expect(pills[1]!.querySelector('.stencil-pill-icon')).toBeNull();
  expect(pills[1]!.querySelector('.stencil-pill-text')?.textContent).toBe('45ms');
  unmount();
});

it('does not render pills when absent', () => {
  const renderFn: StencilRenderFn = () => html`<div>node</div>`;
  const Component = createStencilNodeComponent(renderFn);
  const { container, unmount } = mountWithProps(Component, defaultNodeProps);
  expect(container.querySelector('.stencil-decoration-pills')).toBeNull();
  unmount();
});

it('does not render pills when array is empty', () => {
  const renderFn: StencilRenderFn = () => html`<div>node</div>`;
  const Component = createStencilNodeComponent(renderFn);
  const decoration: NodeDecoration = { pills: [] };
  const { container, unmount } = mountWithProps(Component, {
    ...defaultNodeProps,
    data: { _decoration: decoration },
  });
  expect(container.querySelector('.stencil-decoration-pills')).toBeNull();
  unmount();
});

it('renders pills alongside badge', () => {
  const renderFn: StencilRenderFn = () => html`<div>node</div>`;
  const Component = createStencilNodeComponent(renderFn);
  const decoration: NodeDecoration = {
    badge: { icon: '▶', color: '#0f0' },
    pills: [{ text: 'SLA: 2h', color: '#dc2626' }],
  };
  const { container, unmount } = mountWithProps(Component, {
    ...defaultNodeProps,
    data: { _decoration: decoration },
  });
  expect(container.querySelector('.stencil-decoration-badge')).not.toBeNull();
  expect(container.querySelector('.stencil-decoration-pills')).not.toBeNull();
  unmount();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/graph-renderer run test -- --run 2>&1 | tail -15`
Expected: FAIL — `.stencil-decoration-pills` not found

- [ ] **Step 3: Add DecorationPills component and render it**

In `packages/graph-renderer/src/stencil-wrapper.tsx`, add the `DecorationPills` component after `DecorationOverlay`:

```tsx
function DecorationPills({ pills }: { pills: NonNullable<NodeDecoration['pills']> }): React.JSX.Element {
  return (
    <div
      className="stencil-decoration-pills"
      style={{
        position: 'absolute',
        bottom: -10,
        left: '50%',
        transform: 'translateX(-50%)',
        display: 'flex',
        gap: '3px',
        zIndex: 10,
        pointerEvents: 'none',
      }}
    >
      {pills.map((pill, i) => (
        <span
          key={i}
          className="stencil-pill"
          style={{
            display: 'inline-flex',
            alignItems: 'center',
            gap: '2px',
            background: pill.color,
            color: '#fff',
            borderRadius: '8px',
            padding: '1px 6px',
            fontSize: '9px',
            fontWeight: 600,
            lineHeight: 1,
            whiteSpace: 'nowrap',
          }}
        >
          {pill.icon && <span className="stencil-pill-icon">{pill.icon}</span>}
          <span className="stencil-pill-text">{pill.text}</span>
        </span>
      ))}
    </div>
  );
}
```

Then in the `StencilNode` render return, add after the `DecorationOverlay` line and before `<div ref={containerRef} />`:

```tsx
{decoration?.pills && decoration.pills.length > 0 && <DecorationPills pills={decoration.pills} />}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/graph-renderer run test -- --run 2>&1 | tail -15`
Expected: PASS — all tests green

- [ ] **Step 5: Run full build to verify no regressions**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages build:prod 2>&1 | grep -E "error|successfully" | tail -5`
Expected: all compiled successfully

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/stencil-wrapper.tsx packages/graph-renderer/src/stencil-wrapper.test.tsx
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#404): render decoration pills on graph nodes Refs #404"
```

## References

- specs/issue-404-node-decoration-pills/2026-09-03-node-decoration-pills-design.md — design spec
- packages/graph-core/src/model.ts:22 — NodeDecoration interface
- packages/graph-renderer/src/stencil-wrapper.tsx:50-76 — DecorationBadge pattern
- packages/graph-renderer/src/stencil-wrapper.test.tsx:200-356 — decoration test patterns
- packages/graph-renderer/src/mapping.ts:9-24 — decoration data flow
- docs/protocols/casehub/graph-core-pure-data.md — pure data constraint
- GitHub #404 — focal issue
