# Viz Barrel Crash + Shadow DOM CSS Isolation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #318 — pages-viz: @drdreo/heatmap not available to downstream Maven SNAPSHOT consumers
**Issue group:** #318, #319

**Goal:** Fix two independent packaging/styling bugs — make the pages-viz barrel export safe when `@drdreo/heatmap` isn't installed, and make graph-renderer CSS isolation work inside shadow DOM hosts.

**Architecture:** Dynamic `import()` replaces the static `@drdreo/heatmap` import so the barrel never eagerly pulls in the dependency. Separately, `injectIsolationStyles()` gains shadow-root awareness via `getRootNode()` and per-root ref-counting via `WeakMap`.

**Tech Stack:** TypeScript, Lit, Vitest, `@drdreo/heatmap`, `@xyflow/react`

## Global Constraints

- All Web Components use Lit per protocol PP-20260705-c7687d
- CSS tokens use `--pages-` prefix per protocol PP-20260705-2ae91d
- Existing callers of `injectIsolationStyles()` without a `host` argument must continue working unchanged (backward compatibility)
- `vi.mock()` in vitest intercepts both static and dynamic imports — existing test mocks continue to work

---

## Batch 1: Shadow-aware CSS injection (#319)

### Task 1: Refactor css-isolation.ts — per-root injection and ref-counting

**Files:**
- Modify: `packages/graph-renderer/src/bridge/css-isolation.ts`
- Modify: `packages/graph-renderer/src/bridge/css-isolation.test.ts`

**Interfaces:**
- Produces: `injectIsolationStyles(host?: HTMLElement): HTMLStyleElement` — updated signature, backward compatible
- Produces: `releaseIsolationStyles(host?: HTMLElement): void` — updated signature, backward compatible
- Produces: `resetIsolationState(): void` — test-only helper to clear the WeakMap between tests

- [ ] **Step 1: Write failing tests for shadow-root injection**

Add new test cases to `packages/graph-renderer/src/bridge/css-isolation.test.ts`. These tests create a real shadow root and verify styles are injected there instead of `document.head`.

```typescript
describe('shadow root injection', () => {
  let hostEl: HTMLElement;
  let shadowRoot: ShadowRoot;

  beforeEach(() => {
    hostEl = document.createElement('div');
    document.body.appendChild(hostEl);
    shadowRoot = hostEl.attachShadow({ mode: 'open' });
  });

  afterEach(() => {
    hostEl.remove();
  });

  it('injects into shadow root when host is inside one', () => {
    const child = document.createElement('div');
    shadowRoot.appendChild(child);
    const style = injectIsolationStyles(child);
    expect(style.parentNode).toBe(shadowRoot);
    expect(document.head.querySelector('style[data-graph-isolation]')).toBeNull();
    releaseIsolationStyles(child);
  });

  it('injects into document.head when no host is provided', () => {
    const style = injectIsolationStyles();
    expect(style.parentElement).toBe(document.head);
    releaseIsolationStyles();
  });

  it('injects into document.head when host is in light DOM', () => {
    const lightChild = document.createElement('div');
    document.body.appendChild(lightChild);
    const style = injectIsolationStyles(lightChild);
    expect(style.parentElement).toBe(document.head);
    releaseIsolationStyles(lightChild);
    lightChild.remove();
  });

  it('maintains independent ref-counts per root', () => {
    const child1 = document.createElement('div');
    shadowRoot.appendChild(child1);

    const host2 = document.createElement('div');
    document.body.appendChild(host2);
    const shadow2 = host2.attachShadow({ mode: 'open' });
    const child2 = document.createElement('div');
    shadow2.appendChild(child2);

    injectIsolationStyles(child1);
    injectIsolationStyles(child2);
    injectIsolationStyles(child1);

    releaseIsolationStyles(child1);
    expect(shadowRoot.querySelector('style[data-graph-isolation]')).not.toBeNull();

    releaseIsolationStyles(child1);
    expect(shadowRoot.querySelector('style[data-graph-isolation]')).toBeNull();

    expect(shadow2.querySelector('style[data-graph-isolation]')).not.toBeNull();
    releaseIsolationStyles(child2);
    expect(shadow2.querySelector('style[data-graph-isolation]')).toBeNull();

    host2.remove();
  });

  it('two instances in same shadow root share one style element', () => {
    const child1 = document.createElement('div');
    const child2 = document.createElement('div');
    shadowRoot.appendChild(child1);
    shadowRoot.appendChild(child2);

    const style1 = injectIsolationStyles(child1);
    const style2 = injectIsolationStyles(child2);
    expect(style2).toBe(style1);
    expect(shadowRoot.querySelectorAll('style[data-graph-isolation]').length).toBe(1);

    releaseIsolationStyles(child1);
    releaseIsolationStyles(child2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run src/bridge/css-isolation.test.ts`
Expected: FAIL — `injectIsolationStyles` doesn't accept arguments yet

- [ ] **Step 3: Implement shadow-aware injection**

Replace the global `refCount` variable with a `WeakMap`-based per-root tracker. Add `getStyleRoot` helper. Update `injectIsolationStyles` and `releaseIsolationStyles` signatures to accept optional `host`.

New implementation for `packages/graph-renderer/src/bridge/css-isolation.ts`:

```typescript
import { getRegisteredStyles } from '../registry/stencil-registry.js';

export const DIAGRAM_ROOT_CLASS = 'diagram-root';

export function getIsolationCSS(): string {
  const pluginStyles = getRegisteredStyles();

  return `
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

interface StyleEntry {
  count: number;
  style: HTMLStyleElement;
}

const styleRoots = new WeakMap<Document | ShadowRoot, StyleEntry>();

function getStyleRoot(host?: HTMLElement): Document | ShadowRoot {
  if (host) {
    const root = host.getRootNode();
    if (root instanceof ShadowRoot) return root;
  }
  return document;
}

export function injectIsolationStyles(host?: HTMLElement): HTMLStyleElement {
  const root = getStyleRoot(host);
  const target = root instanceof ShadowRoot ? root : document.head;
  const entry = styleRoots.get(root);

  if (entry) {
    entry.count++;
    entry.style.textContent = getIsolationCSS();
    return entry.style;
  }

  const style = document.createElement('style');
  style.setAttribute('data-graph-isolation', 'true');
  style.textContent = getIsolationCSS();
  target.appendChild(style);
  styleRoots.set(root, { count: 1, style });
  return style;
}

export function releaseIsolationStyles(host?: HTMLElement): void {
  const root = getStyleRoot(host);
  const entry = styleRoots.get(root);
  if (!entry) return;
  entry.count--;
  if (entry.count === 0) {
    entry.style.remove();
    styleRoots.delete(root);
  }
}

export function resetIsolationState(): void {
  // WeakMap entries are auto-collected; this exists so tests
  // can force a clean slate when the same Document object
  // persists across test cases.
  const docEntry = styleRoots.get(document);
  if (docEntry) {
    docEntry.style.remove();
    styleRoots.delete(document);
  }
}
```

- [ ] **Step 4: Update existing tests for per-root cleanup**

The existing tests call `injectIsolationStyles()` / `releaseIsolationStyles()` without arguments. These must continue to pass (backward compat). Update the `afterEach` to call `resetIsolationState()` for a clean slate:

```typescript
import {
  getIsolationCSS,
  injectIsolationStyles,
  releaseIsolationStyles,
  resetIsolationState,
  DIAGRAM_ROOT_CLASS,
} from './css-isolation.js';

// In the top-level afterEach:
afterEach(() => {
  resetIsolationState();
  document.head.querySelectorAll('style[data-graph-isolation]')
    .forEach(el => el.remove());
});
```

- [ ] **Step 5: Run all css-isolation tests**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run src/bridge/css-isolation.test.ts`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/bridge/css-isolation.ts packages/graph-renderer/src/bridge/css-isolation.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): shadow-aware CSS isolation with per-root ref-counting

injectIsolationStyles() now accepts an optional host element and
uses getRootNode() to detect shadow boundaries. Ref-counting is
per-root via WeakMap instead of a global counter.

Refs #319"
```

### Task 2: Wire GraphCanvas to pass host element

**Files:**
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts`

**Interfaces:**
- Consumes: `injectIsolationStyles(host?: HTMLElement): HTMLStyleElement` from Task 1
- Consumes: `releaseIsolationStyles(host?: HTMLElement): void` from Task 1

- [ ] **Step 1: Write failing test**

Add to `packages/graph-renderer/src/bridge/css-isolation.test.ts` (or a new `GraphCanvas.test.ts` if one exists — `bridge.test.ts` exists, check if it covers this):

The key behavioural test is that GraphCanvas passes `this` when calling inject/release. Since GraphCanvas creates its own DOM and appends to `this` (light DOM via `createRenderRoot() → this`), the simplest verification is to confirm the call signatures are correct. The unit tests in Task 1 already verify that passing a host in a shadow root works correctly at the css-isolation level. The GraphCanvas change is a two-line wiring fix — verify via the full test suite.

- [ ] **Step 2: Update GraphCanvas.ts**

In `packages/graph-renderer/src/bridge/GraphCanvas.ts`, change line 38:

```typescript
// Before:
injectIsolationStyles();

// After:
injectIsolationStyles(this);
```

And in `disconnectedCallback()`, change the `releaseIsolationStyles()` call:

```typescript
// Before:
releaseIsolationStyles();

// After:
releaseIsolationStyles(this);
```

- [ ] **Step 3: Run full graph-renderer test suite**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/bridge/GraphCanvas.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): pass host to CSS isolation for shadow DOM support

GraphCanvas now passes 'this' to injectIsolationStyles() and
releaseIsolationStyles() so styles reach shadow DOM hosts.

Closes #319"
```

---

## Batch 2: Dynamic import for @drdreo/heatmap (#318)

### Task 3: Convert PagesDensityHeatmap to dynamic import

**Files:**
- Modify: `packages/pages-viz/src/charts/PagesDensityHeatmap.ts`
- Modify: `packages/pages-viz/src/charts/PagesDensityHeatmap.test.ts`

**Interfaces:**
- Produces: `PagesDensityHeatmap` class — same public API, internal async init

- [ ] **Step 1: Write failing test for dynamic import behaviour**

Add a test to `packages/pages-viz/src/charts/PagesDensityHeatmap.test.ts` that verifies the component still works when the module is loaded dynamically. The existing `vi.mock("@drdreo/heatmap", ...)` intercepts dynamic imports too, so the mock setup stays the same.

Add under the `lifecycle` describe block:

```typescript
it("handles resize by recreating via async path", async () => {
  const ds = makeDataSet(
    [["x", "NUMBER"], ["y", "NUMBER"], ["value", "NUMBER"]],
    [[10, 20, 5], [50, 60, 8]],
  );
  await renderChart(el, { lookup: mockLookup("test") }, ds);

  vi.mocked(createHeatmap).mockClear();
  mockHeatmap.destroy.mockClear();

  el.onResize();
  await new Promise(r => { setTimeout(r, 0); });
  await el.updateComplete;

  expect(mockHeatmap.destroy).toHaveBeenCalled();
  expect(createHeatmap).toHaveBeenCalled();
});
```

- [ ] **Step 2: Run test to verify it passes with current static import**

Run: `yarn workspace @casehubio/pages-viz run test -- --run src/charts/PagesDensityHeatmap.test.ts`
Expected: PASS (this is a characterisation test — confirms current behaviour before refactoring)

- [ ] **Step 3: Convert to dynamic import**

Replace the static import and refactor `updated()`, `createInstance()`, and `onResize()` in `packages/pages-viz/src/charts/PagesDensityHeatmap.ts`.

Remove line 4 (static import):
```typescript
// DELETE: import { createHeatmap, withTooltip, withLegend } from "@drdreo/heatmap";
```

Add module-level lazy loader after the existing imports:

```typescript
type HeatmapModule = typeof import("@drdreo/heatmap");
let heatmapModule: HeatmapModule | undefined;

async function loadHeatmapModule(): Promise<HeatmapModule> {
  if (!heatmapModule) {
    heatmapModule = await import("@drdreo/heatmap");
  }
  return heatmapModule;
}
```

Replace `updated()`:

```typescript
override updated(): void {
  const container = this._containerRef.value;
  if (!container || !this.props || !this.dataSet) return;
  if (container.offsetWidth === 0 || container.offsetHeight === 0) return;
  void this._updateHeatmap(container);
}

private async _updateHeatmap(container: HTMLDivElement): Promise<void> {
  const mod = await loadHeatmapModule();
  if (!this.props || !this.dataSet) return;
  const points = this.extractAndNormalize(this.dataSet, this.props, container);
  if (this._heatmap) {
    this._heatmap.setData(points);
  } else {
    this._heatmap = this.createInstance(mod, container, points, this.props);
  }
}
```

Update `createInstance` to accept the module as first parameter:

```typescript
private createInstance(
  mod: HeatmapModule,
  container: HTMLDivElement,
  data: HeatmapPoint[],
  props: DensityHeatmapProps,
): HeatmapInstance {
  const config: Record<string, unknown> = { container, data };

  if (props.gradient) {
    config.gradient = props.gradient;
  }
  if (props.aggregation) {
    config.aggregationMode = props.aggregation;
  }

  const features: unknown[] = [];
  if (props.showTooltip) {
    features.push(mod.withTooltip());
  }
  if (props.showLegend) {
    features.push(mod.withLegend());
  }

  return mod.createHeatmap(config as never, ...features as never[]) as unknown as HeatmapInstance;
}
```

Update `onResize()` to use the async path:

```typescript
override onResize(): void {
  if (!this._heatmap || !this._containerRef.value || !this.props || !this.dataSet) return;
  this._heatmap.destroy();
  this._heatmap = undefined;
  void this._updateHeatmap(this._containerRef.value);
}
```

- [ ] **Step 4: Run all PagesDensityHeatmap tests**

Run: `yarn workspace @casehubio/pages-viz run test -- --run src/charts/PagesDensityHeatmap.test.ts`
Expected: ALL PASS — `vi.mock()` intercepts dynamic imports in vitest

- [ ] **Step 5: Run full pages-viz test suite**

Run: `yarn workspace @casehubio/pages-viz run test -- --run`
Expected: ALL PASS

- [ ] **Step 6: Type-check both packages**

Run: `yarn typecheck`
Expected: No new errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/charts/PagesDensityHeatmap.ts packages/pages-viz/src/charts/PagesDensityHeatmap.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): dynamic import for @drdreo/heatmap

Replace static top-level import with lazy dynamic import() so the
barrel export doesn't crash downstream consumers that don't have
@drdreo/heatmap installed (Maven SNAPSHOT packaging).

Closes #318"
```

---

## blocks-ui instructions

After both fixes land, provide this to the blocks-ui session:

> **@drdreo/heatmap packaging fix**
>
> `casehub-pages` PR [link] converted `PagesDensityHeatmap` to use
> `import("@drdreo/heatmap")` dynamically, so the barrel no longer
> crashes. For belt-and-suspenders, add `@drdreo/heatmap` to the Maven
> SNAPSHOT extraction in blocks-ui:
>
> 1. In the pom.xml or build script that unpacks `@casehubio/pages-viz`
>    from the WebJar into `.casehub-packages`, add `@drdreo/heatmap` to
>    the list of npm dependencies that get extracted alongside the
>    `@casehubio` packages.
>
> 2. Remove the Vite alias stub at
>    `examples/src/stubs/drdreo-heatmap.ts` and its reference in
>    `examples/vite.config.ts` — the dynamic import means the stub is
>    no longer needed even without the Maven packaging fix.

## References

- [2026-08-18-viz-shadow-dom-fixes-design.md] — design spec this plan implements
- [packages/pages-viz/src/charts/PagesDensityHeatmap.ts:4] — static import site
- [packages/pages-viz/src/index.ts:48] — barrel re-export
- [packages/graph-renderer/src/bridge/css-isolation.ts] — current injection
- [packages/graph-renderer/src/bridge/GraphCanvas.ts:38] — current call site
- [GE-20260802-01] — CSS isolation pattern
- [GE-20260810-02] — @drdreo/heatmap integration
- [GE-20260810-03] — Dynamic imports + lazy loading pattern
- [PP-20260705-c7687d] — Web component strategy protocol
- [GitHub #318] — barrel crash issue
- [GitHub #319] — shadow DOM CSS isolation issue
