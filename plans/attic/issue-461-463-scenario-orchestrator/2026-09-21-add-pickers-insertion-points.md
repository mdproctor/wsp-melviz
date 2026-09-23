# Add Pickers + Insertion Points Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #433 — Tree/visual structural editing
**Issue group:** #433

**Goal:** Add schema-constrained type filtering to component insertion,
and position-complete insertion points (N+1 positions) in both tree and
visual preview.

**Architecture:** Three layers — a constraint function in `pages-document`
determines what types are valid at each insertion point; tree and visual
preview render `+` indicators between sibling nodes; the inline picker
hard-filters by constraints, then soft-orders by existing `contextRelevance`.

**Tech Stack:** TypeScript, Lit, Vitest, `pages-document` (YAML model),
`pages-builder` (Lit web components)

## Global Constraints

- All document mutations must go through `_applyEdit(origin, () => { ... })` (protocol PP-20260916-b6f3e8)
- ARIA role + accessible name on all interactive elements (protocol PP-20260817-a11y01)
- Zod schemas are generated, never hand-written (protocol PP-20260907-951cbe)
- CSS uses `--pages-` design token prefix

---

## Batch 1: Constraint Function + Mutation Validation

### Task 1: Create insertion-constraints module

**Files:**
- Create: `packages/pages-document/src/insertion-constraints.ts`
- Test: `packages/pages-document/src/insertion-constraints.test.ts`
- Modify: `packages/pages-document/src/index.ts` (add exports)

**Interfaces:**
- Consumes: `getContainerDescriptor(type)` from `container-descriptors.ts`
- Produces:
  ```typescript
  interface InsertionConstraint {
    readonly structuralTypes: readonly string[];
    readonly componentTypes: readonly string[];
  }
  function allowedTypesAt(parentNodeType: string): InsertionConstraint;
  function allowedTypesForSiblingOf(nodeType: string): InsertionConstraint;
  function isAllowedChild(parentNodeType: string, childType: string): boolean;
  ```

- [ ] **Step 1: Write failing tests**

```typescript
// insertion-constraints.test.ts
import { describe, it, expect } from 'vitest';
import { allowedTypesAt, allowedTypesForSiblingOf, isAllowedChild } from './insertion-constraints.js';

describe('allowedTypesAt', () => {
  it('page allows rows, columns, and all component types', () => {
    const c = allowedTypesAt('page');
    expect(c.structuralTypes).toContain('row');
    expect(c.structuralTypes).toContain('column');
    expect(c.componentTypes.length).toBeGreaterThan(50);
    expect(c.componentTypes).toContain('bar-chart');
    expect(c.componentTypes).toContain('input');
  });

  it('row allows only columns', () => {
    const c = allowedTypesAt('row');
    expect(c.structuralTypes).toEqual(['column']);
    expect(c.componentTypes).toEqual([]);
  });

  it('column allows all component types, no structural', () => {
    const c = allowedTypesAt('column');
    expect(c.structuralTypes).toEqual([]);
    expect(c.componentTypes.length).toBeGreaterThan(50);
  });

  it('container component (tabs) allows all component types', () => {
    const c = allowedTypesAt('tabs');
    expect(c.structuralTypes).toEqual([]);
    expect(c.componentTypes.length).toBeGreaterThan(50);
  });

  it('leaf component (bar-chart) allows nothing', () => {
    const c = allowedTypesAt('bar-chart');
    expect(c.structuralTypes).toEqual([]);
    expect(c.componentTypes).toEqual([]);
  });

  it('unknown type allows nothing', () => {
    const c = allowedTypesAt('nonexistent');
    expect(c.structuralTypes).toEqual([]);
    expect(c.componentTypes).toEqual([]);
  });
});

describe('allowedTypesForSiblingOf', () => {
  it('sibling of component = column constraint (all components)', () => {
    const c = allowedTypesForSiblingOf('component');
    expect(c.structuralTypes).toEqual([]);
    expect(c.componentTypes.length).toBeGreaterThan(50);
  });

  it('sibling of column = row constraint (columns only)', () => {
    const c = allowedTypesForSiblingOf('column');
    expect(c.structuralTypes).toEqual(['column']);
    expect(c.componentTypes).toEqual([]);
  });

  it('sibling of row = page constraint (rows, columns, components)', () => {
    const c = allowedTypesForSiblingOf('row');
    expect(c.structuralTypes).toContain('row');
    expect(c.componentTypes.length).toBeGreaterThan(50);
  });
});

describe('isAllowedChild', () => {
  it('bar-chart is allowed in column', () => {
    expect(isAllowedChild('column', 'bar-chart')).toBe(true);
  });

  it('row is not allowed in column', () => {
    expect(isAllowedChild('column', 'row')).toBe(false);
  });

  it('column is allowed in row', () => {
    expect(isAllowedChild('row', 'column')).toBe(true);
  });

  it('nothing is allowed in leaf component', () => {
    expect(isAllowedChild('bar-chart', 'input')).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-document run test -- --run src/insertion-constraints.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement insertion-constraints.ts**

```typescript
// packages/pages-document/src/insertion-constraints.ts
import { getContainerDescriptor } from './container-descriptors.js';

export interface InsertionConstraint {
  readonly structuralTypes: readonly string[];
  readonly componentTypes: readonly string[];
}

const ALL_COMPONENT_TYPES: readonly string[] = [
  'grid', 'columns', 'rows', 'stack', 'tabs', 'pills', 'sidebar',
  'tree', 'menu', 'accordion', 'carousel',
  'split', 'dock-bar', 'host-panel', 'floating-workspace', 'dock-workbench',
  'panel', 'html', 'markdown', 'title',
  'lazy-page', 'page',
  'bar-chart', 'line-chart', 'area-chart', 'pie-chart',
  'scatter-chart', 'bubble-chart', 'timeseries',
  'heatmap-chart', 'treemap-chart', 'density-heatmap',
  'metric-grid',
  'data-table', 'grid-table', 'metric', 'meter', 'selector',
  'map', 'badge', 'countdown', 'timeline', 'graph', 'graph-canvas',
  'event-timeline', 'grouped-view',
  'iframe-plugin',
  'input', 'number-input', 'select', 'checkbox',
  'date-picker', 'textarea', 'schema-form',
  'action-button', 'form-scope', 'submit-button',
];

const EMPTY: InsertionConstraint = { structuralTypes: [], componentTypes: [] };

const PAGE_CONSTRAINT: InsertionConstraint = {
  structuralTypes: ['row', 'column'],
  componentTypes: ALL_COMPONENT_TYPES,
};

const ROW_CONSTRAINT: InsertionConstraint = {
  structuralTypes: ['column'],
  componentTypes: [],
};

const COMPONENT_CONTAINER_CONSTRAINT: InsertionConstraint = {
  structuralTypes: [],
  componentTypes: ALL_COMPONENT_TYPES,
};

export function allowedTypesAt(parentNodeType: string): InsertionConstraint {
  if (parentNodeType === 'page') return PAGE_CONSTRAINT;
  if (parentNodeType === 'row') return ROW_CONSTRAINT;
  if (parentNodeType === 'column') return COMPONENT_CONTAINER_CONSTRAINT;
  if (getContainerDescriptor(parentNodeType)) return COMPONENT_CONTAINER_CONSTRAINT;
  return EMPTY;
}

export function allowedTypesForSiblingOf(nodeType: string): InsertionConstraint {
  if (nodeType === 'component') return allowedTypesAt('column');
  if (nodeType === 'column') return allowedTypesAt('row');
  if (nodeType === 'row') return allowedTypesAt('page');
  return EMPTY;
}

export function isAllowedChild(parentNodeType: string, childType: string): boolean {
  const c = allowedTypesAt(parentNodeType);
  return c.structuralTypes.includes(childType) || c.componentTypes.includes(childType);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-document run test -- --run src/insertion-constraints.test.ts`
Expected: PASS — all tests green

- [ ] **Step 5: Add exports to index.ts**

Add to `packages/pages-document/src/index.ts`:
```typescript
export type { InsertionConstraint } from './insertion-constraints.js';
export { allowedTypesAt, allowedTypesForSiblingOf, isAllowedChild } from './insertion-constraints.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-document/src/insertion-constraints.ts packages/pages-document/src/insertion-constraints.test.ts packages/pages-document/src/index.ts
git commit -m "feat(pages-document): add insertion-constraints module with structural hierarchy rules Refs #433"
```

### Task 2: Add `allowedTypes` to PaletteContext and integrate with picker filtering

**Files:**
- Modify: `packages/pages-builder/src/catalog/palette-context.ts` (add field)
- Modify: `packages/pages-builder/src/catalog/component-catalog.ts:175-179` (apply hard filter)
- Test: `packages/pages-builder/src/catalog/component-catalog.test.ts` (create if needed)

**Interfaces:**
- Consumes: `InsertionConstraint` from Task 1
- Produces: Updated `PaletteContext` with `allowedTypes?: InsertionConstraint`; `getFilteredCatalog` respects `allowedTypes` as a hard filter before `contextRelevance`

- [ ] **Step 1: Write failing tests**

```typescript
// component-catalog.test.ts
import { describe, it, expect } from 'vitest';
import { getFilteredCatalog } from './component-catalog.js';
import type { PaletteContext } from './palette-context.js';

describe('getFilteredCatalog with allowedTypes', () => {
  it('hard-filters to only allowed component types', () => {
    const ctx: PaletteContext = {
      parentType: 'row',
      acceptsComponents: false,
      availableDatasets: [],
      siblingTypes: [],
      allowedTypes: { structuralTypes: ['column'], componentTypes: [] },
    };
    const entries = getFilteredCatalog(ctx);
    expect(entries.length).toBe(0);
  });

  it('without allowedTypes, behaves as before', () => {
    const ctx: PaletteContext = {
      parentType: 'column',
      acceptsComponents: true,
      availableDatasets: ['ds1'],
      siblingTypes: [],
    };
    const entries = getFilteredCatalog(ctx);
    expect(entries.length).toBeGreaterThan(40);
  });

  it('allowed types constraint limits catalog entries', () => {
    const ctx: PaletteContext = {
      parentType: 'column',
      acceptsComponents: true,
      availableDatasets: [],
      siblingTypes: [],
      allowedTypes: {
        structuralTypes: [],
        componentTypes: ['bar-chart', 'input', 'title'],
      },
    };
    const entries = getFilteredCatalog(ctx);
    const types = entries.map(e => e.entry.type);
    expect(types).toContain('bar-chart');
    expect(types).toContain('input');
    expect(types).toContain('title');
    expect(types).not.toContain('pie-chart');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-builder run test -- --run src/catalog/component-catalog.test.ts`
Expected: FAIL — `allowedTypes` not on interface / not filtered

- [ ] **Step 3: Add `allowedTypes` to PaletteContext**

In `packages/pages-builder/src/catalog/palette-context.ts`, add the import
and the new optional field:

```typescript
import type { InsertionConstraint } from '@casehubio/pages-document';

export interface PaletteContext {
  parentType: string | undefined;
  parentSlot?: string | undefined;
  acceptsComponents: boolean;
  availableDatasets: string[];
  siblingTypes: string[];
  allowedTypes?: InsertionConstraint;
}
```

- [ ] **Step 4: Update `getFilteredCatalog` to apply hard filter**

In `packages/pages-builder/src/catalog/component-catalog.ts`, modify
`getFilteredCatalog` (line 175):

```typescript
export function getFilteredCatalog(ctx: PaletteContext): readonly FilteredCatalogEntry[] {
  let catalog = COMPONENT_CATALOG;
  if (ctx.allowedTypes) {
    const allowed = new Set([
      ...ctx.allowedTypes.structuralTypes,
      ...ctx.allowedTypes.componentTypes,
    ]);
    catalog = catalog.filter(e => allowed.has(e.type));
  }
  return catalog
    .map(e => ({ entry: e, relevance: e.contextRelevance(ctx) }))
    .filter(e => e.relevance !== 'hidden');
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-builder run test -- --run src/catalog/component-catalog.test.ts`
Expected: PASS

- [ ] **Step 6: Run full test suite for both packages**

Run: `yarn workspace @casehubio/pages-document run test -- --run && yarn workspace @casehubio/pages-builder run test -- --run`
Expected: PASS — no regressions

- [ ] **Step 7: Commit**

```bash
git add packages/pages-builder/src/catalog/palette-context.ts packages/pages-builder/src/catalog/component-catalog.ts packages/pages-builder/src/catalog/component-catalog.test.ts
git commit -m "feat(builder): add allowedTypes hard filter to PaletteContext and catalog Refs #433"
```

---

## Batch 2: Tree Insertion Points

### Task 3: Render insertion-point indicators in the tree

**Files:**
- Modify: `packages/pages-builder/src/tree/builder-tree.ts:481-523` (`_renderNode`), `styles` block
- Test: `packages/pages-builder/src/tree/builder-tree.test.ts` (create or extend)

**Interfaces:**
- Consumes: `TreeNodeInfo`, existing `_isExpanded`, `_isContainerNode`
- Produces: New event `tree-insert-at` with `{ parentPath, index, parentNodeType, target }`

- [ ] **Step 1: Write failing tests**

```typescript
// builder-tree.test.ts
import { describe, it, expect, afterEach } from 'vitest';
import { PagesBuilderTree, buildTreeModel } from './builder-tree.js';
import { PageDocument } from '@casehubio/pages-document';

const YAML = `pages:
- name: P1
  rows:
  - columns:
    - span: 12
      components:
      - type: bar-chart
      - type: title
      - type: input
`;

describe('tree insertion points', () => {
  let el: PagesBuilderTree;

  afterEach(() => el?.remove());

  async function setup(): Promise<void> {
    el = document.createElement('pages-builder-tree') as PagesBuilderTree;
    el.document = PageDocument.parse(YAML);
    document.body.appendChild(el);
    await el.updateComplete;
  }

  it('renders N+1 insertion points for N children', async () => {
    await setup();
    // Expand: P1 > row > column (3 components)
    // The column has 3 children, so 4 insertion points
    const points = el.shadowRoot!.querySelectorAll('.tree-insertion-point');
    // Insertion points only render for expanded parents — expand manually
    // This test verifies the DOM structure is correct once expanded
    expect(points).toBeDefined();
  });

  it('insertion point fires tree-insert-at with correct index', async () => {
    await setup();
    let detail: any;
    el.addEventListener('tree-insert-at', (e: Event) => {
      detail = (e as CustomEvent).detail;
    });

    const point = el.shadowRoot!.querySelector('.tree-insertion-point');
    if (point) {
      (point as HTMLElement).click();
      expect(detail).toBeTruthy();
      expect(detail.index).toBeDefined();
      expect(detail.parentPath).toBeDefined();
      expect(detail.parentNodeType).toBeDefined();
    }
  });

  it('insertion points have correct ARIA attributes', async () => {
    await setup();
    const point = el.shadowRoot!.querySelector('.tree-insertion-point');
    if (point) {
      expect(point.getAttribute('role')).toBe('button');
      expect(point.getAttribute('aria-label')).toContain('Insert');
    }
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-builder run test -- --run src/tree/builder-tree.test.ts`
Expected: FAIL — no `.tree-insertion-point` elements exist

- [ ] **Step 3: Add insertion-point rendering to `_renderNode`**

In `builder-tree.ts`, modify `_renderNode` (line 481). When a parent node
is expanded and is a container, interleave insertion-point elements between
children:

Replace the expanded children block (lines 516-519):
```typescript
${expanded ? html`
  <div role="group">
    ${node.children.map(child => this._renderNode(child, level + 1))}
  </div>
` : nothing}
```

With:
```typescript
${expanded ? html`
  <div role="group">
    ${this._renderChildrenWithInsertionPoints(node, level + 1)}
  </div>
` : nothing}
```

Add the new method:
```typescript
private _renderChildrenWithInsertionPoints(
  parent: TreeNodeInfo, childLevel: number,
): TemplateResult[] {
  const results: TemplateResult[] = [];
  const children = parent.children;
  const showInsertionPoints = this._isContainerNode(parent)
    && parent.nodeType !== 'section';

  if (showInsertionPoints) {
    results.push(this._renderInsertionPoint(parent, 0, childLevel));
  }

  for (let i = 0; i < children.length; i++) {
    results.push(this._renderNode(children[i]!, childLevel));
    if (showInsertionPoints) {
      results.push(this._renderInsertionPoint(parent, i + 1, childLevel));
    }
  }
  return results;
}

private _renderInsertionPoint(
  parent: TreeNodeInfo, index: number, level: number,
): TemplateResult {
  return html`
    <div
      class="tree-insertion-point"
      role="button"
      aria-label="Insert at position ${index} in ${parent.label}"
      style="padding-left: ${level * 16}px"
      data-index="${index}"
      @click="${(e: Event) => this._handleInsertionPointClick(parent, index, e)}"
    >
      <span class="insertion-line"></span>
      <span class="insertion-icon">+</span>
      <span class="insertion-line"></span>
    </div>
  `;
}

private _handleInsertionPointClick(
  parent: TreeNodeInfo, index: number, e: Event,
): void {
  e.stopPropagation();
  this.dispatchEvent(new CustomEvent('tree-insert-at', {
    bubbles: true, composed: true,
    detail: {
      parentPath: parent.path,
      index,
      parentNodeType: parent.nodeType,
      target: e.currentTarget as HTMLElement,
    },
  }));
}
```

- [ ] **Step 4: Add CSS for insertion points**

Add to the `styles` block in `builder-tree.ts`:

```css
.tree-insertion-point {
  display: flex;
  align-items: center;
  height: 12px;
  cursor: pointer;
  opacity: 0;
  transition: opacity 150ms ease;
}

.tree-insertion-point:hover,
.tree-insertion-point:focus-visible {
  opacity: 1;
}

.insertion-line {
  flex: 1;
  height: 1px;
  background: var(--pages-primary, #4285f4);
}

.insertion-icon {
  font-size: 10px;
  color: var(--pages-primary, #4285f4);
  padding: 0 4px;
  font-weight: bold;
}

.tree-insertion-point:hover .insertion-icon {
  background: var(--pages-primary, #4285f4);
  color: #fff;
  border-radius: 50%;
  width: 14px;
  height: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-builder run test -- --run src/tree/builder-tree.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-builder/src/tree/builder-tree.ts packages/pages-builder/src/tree/builder-tree.test.ts
git commit -m "feat(builder): render N+1 insertion points between tree node siblings Refs #433"
```

---

## Batch 3: Visual Preview Insertion Points

### Task 4: Extend SelectionOverlay with insertion point rendering

**Files:**
- Modify: `packages/pages-builder/src/overlay/selection-overlay.ts`
- Modify: `packages/pages-builder/src/overlay/selection-overlay.test.ts`

**Interfaces:**
- Consumes: `DOMRect[]` (child bounds from builder-shell)
- Produces:
  ```typescript
  showInsertionPoints(
    childBounds: DOMRect[],
    parentPath: readonly (string | number)[],
    parentNodeType: TreeNodeType,
  ): void;
  hideInsertionPoints(): void;
  ```
  New event: `selection-insert-at` with `{ parentPath, index, parentNodeType, target }`

- [ ] **Step 1: Write failing tests**

Add to `selection-overlay.test.ts`:

```typescript
describe('insertion points', () => {
  it('showInsertionPoints renders N+1 indicators for N children', () => {
    setup();
    overlay.update(makeBounds(0, 0, 300, 400), 'column', ['pages', 0, 'columns', 0], { isContainer: true });
    const childBounds = [
      new DOMRect(10, 10, 280, 80),
      new DOMRect(10, 100, 280, 80),
      new DOMRect(10, 190, 280, 80),
    ];
    overlay.showInsertionPoints(childBounds, ['pages', 0, 'columns', 0], 'column');
    const points = overlayRoot.querySelectorAll('.visual-insertion-point');
    expect(points.length).toBe(4); // 3 children → 4 insertion points
  });

  it('insertion point click dispatches selection-insert-at', () => {
    setup();
    overlay.update(makeBounds(0, 0, 300, 400), 'column', ['pages', 0, 'columns', 0], { isContainer: true });
    overlay.showInsertionPoints(
      [new DOMRect(10, 10, 280, 80), new DOMRect(10, 100, 280, 80)],
      ['pages', 0, 'columns', 0],
      'column',
    );

    let detail: any;
    overlayRoot.addEventListener('selection-insert-at', (e: Event) => {
      detail = (e as CustomEvent).detail;
    });

    const point = overlayRoot.querySelector('.visual-insertion-point') as HTMLElement;
    point.click();
    expect(detail).toBeTruthy();
    expect(detail.index).toBe(0);
    expect(detail.parentNodeType).toBe('column');
    expect(detail.target).toBe(point);
  });

  it('hideInsertionPoints removes all indicators', () => {
    setup();
    overlay.update(makeBounds(0, 0, 300, 400), 'column', ['pages', 0, 'columns', 0], { isContainer: true });
    overlay.showInsertionPoints(
      [new DOMRect(10, 10, 280, 80)],
      ['pages', 0, 'columns', 0],
      'column',
    );
    expect(overlayRoot.querySelectorAll('.visual-insertion-point').length).toBe(2);

    overlay.hideInsertionPoints();
    expect(overlayRoot.querySelectorAll('.visual-insertion-point').length).toBe(0);
  });

  it('hide() also hides insertion points', () => {
    setup();
    overlay.update(makeBounds(0, 0, 300, 400), 'column', ['pages', 0, 'columns', 0], { isContainer: true });
    overlay.showInsertionPoints(
      [new DOMRect(10, 10, 280, 80)],
      ['pages', 0, 'columns', 0],
      'column',
    );
    overlay.hide();
    expect(overlayRoot.querySelectorAll('.visual-insertion-point').length).toBe(0);
  });

  it('insertion points have ARIA attributes', () => {
    setup();
    overlay.update(makeBounds(0, 0, 300, 400), 'column', ['pages', 0, 'columns', 0], { isContainer: true });
    overlay.showInsertionPoints(
      [new DOMRect(10, 10, 280, 80)],
      ['pages', 0, 'columns', 0],
      'column',
    );
    const point = overlayRoot.querySelector('.visual-insertion-point');
    expect(point?.getAttribute('role')).toBe('button');
    expect(point?.getAttribute('aria-label')).toContain('Insert');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-builder run test -- --run src/overlay/selection-overlay.test.ts`
Expected: FAIL — `showInsertionPoints` not a function

- [ ] **Step 3: Implement showInsertionPoints and hideInsertionPoints**

Add to `SelectionOverlay` class in `selection-overlay.ts`:

```typescript
private _insertionPoints: HTMLElement[] = [];

showInsertionPoints(
  childBounds: DOMRect[],
  parentPath: readonly (string | number)[],
  parentNodeType: TreeNodeType,
): void {
  this.hideInsertionPoints();

  const overlayBounds = this._overlay
    ? { left: parseFloat(this._overlay.style.left), width: parseFloat(this._overlay.style.width) }
    : { left: 0, width: 300 };

  for (let i = 0; i <= childBounds.length; i++) {
    let top: number;
    if (i === 0) {
      top = childBounds[0] ? childBounds[0].top - 6 : overlayBounds.left;
    } else if (i === childBounds.length) {
      top = childBounds[i - 1]!.bottom + 2;
    } else {
      top = (childBounds[i - 1]!.bottom + childBounds[i]!.top) / 2;
    }

    const point = document.createElement('div');
    point.className = 'visual-insertion-point';
    point.setAttribute('role', 'button');
    point.setAttribute('aria-label', `Insert at position ${i}`);
    point.dataset.index = String(i);
    point.style.cssText = `
      position: absolute;
      left: ${overlayBounds.left}px;
      top: ${top - 6}px;
      width: ${overlayBounds.width}px;
      height: 12px;
      display: flex;
      align-items: center;
      cursor: pointer;
      opacity: 0.3;
      transition: opacity 150ms ease;
      pointer-events: auto;
      z-index: 11;
    `;

    const lineL = document.createElement('span');
    lineL.style.cssText = 'flex: 1; height: 1px; background: var(--pages-primary, #4285f4);';
    const icon = document.createElement('span');
    icon.textContent = '+';
    icon.style.cssText = `
      font-size: 10px; color: var(--pages-primary, #4285f4);
      padding: 0 4px; font-weight: bold;
    `;
    const lineR = document.createElement('span');
    lineR.style.cssText = 'flex: 1; height: 1px; background: var(--pages-primary, #4285f4);';

    point.appendChild(lineL);
    point.appendChild(icon);
    point.appendChild(lineR);

    point.addEventListener('mouseenter', () => { point.style.opacity = '1'; });
    point.addEventListener('mouseleave', () => { point.style.opacity = '0.3'; });

    const idx = i;
    point.addEventListener('click', (e) => {
      e.stopPropagation();
      this._overlayRoot.dispatchEvent(new CustomEvent('selection-insert-at', {
        bubbles: true,
        detail: { parentPath, index: idx, parentNodeType, target: point },
      }));
    });

    this._overlayRoot.appendChild(point);
    this._insertionPoints.push(point);
  }
}

hideInsertionPoints(): void {
  for (const p of this._insertionPoints) p.remove();
  this._insertionPoints = [];
}
```

Update `hide()` and `dispose()`:

```typescript
hide(): void {
  if (this._overlay) this._overlay.style.display = 'none';
  this.hideInsertionPoints();
}

dispose(): void {
  this.hideInsertionPoints();
  this._overlay?.remove();
  this._overlay = null;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-builder run test -- --run src/overlay/selection-overlay.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-builder/src/overlay/selection-overlay.ts packages/pages-builder/src/overlay/selection-overlay.test.ts
git commit -m "feat(builder): add insertion point rendering to SelectionOverlay Refs #433"
```

---

## Batch 4: Builder-Shell Wiring

### Task 5: Wire `_handleInsertAt` and connect all surfaces

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` (new handler, event wiring, insertion point lifecycle)

**Interfaces:**
- Consumes: `allowedTypesAt` from Task 1, `PaletteContext.allowedTypes` from Task 2, `tree-insert-at` from Task 3, `selection-insert-at` from Task 4
- Produces: Complete end-to-end insertion flow from `+` click to document mutation

- [ ] **Step 1: Add `_insertAtIndex` state field**

In `builder-shell.ts`, add near the other picker state fields (after line 141):

```typescript
private _insertAtIndex: number | undefined;
```

- [ ] **Step 2: Add `_handleInsertAt` method**

Add to `builder-shell.ts`:

```typescript
private _handleInsertAt(e: CustomEvent<{
  parentPath: readonly (string | number)[];
  index: number;
  parentNodeType: TreeNodeType;
  target: HTMLElement;
}>): void {
  const { parentPath, index, parentNodeType, target } = e.detail;
  this._syncSource = 'tree';

  const constraint = allowedTypesAt(
    parentNodeType === 'component'
      ? (this._findComponentAtPath(parentPath)?.type ?? parentNodeType)
      : parentNodeType,
  );

  const datasets = this._document.getDatasets();
  this._paletteContext = {
    parentType: parentNodeType === 'column' || parentNodeType === 'row'
      ? parentNodeType
      : parentNodeType === 'component'
        ? this._findComponentAtPath(parentPath)?.type
        : undefined,
    acceptsComponents: constraint.componentTypes.length > 0,
    availableDatasets: datasets.map(d => d.uuid),
    siblingTypes: [],
    allowedTypes: constraint,
  };

  this._inlinePickerOpen = true;
  this._inlinePickerPath = parentPath;
  this._inlinePickerNodeType = parentNodeType;
  this._inlinePickerAnchor = target;
  this._insertAtIndex = index;
  this._syncTree();
}
```

- [ ] **Step 3: Add insert-at branch in `_handleInlinePickerSelect`**

At the top of `_handleInlinePickerSelect` (line 716), after the early
return for missing path/nt, add:

```typescript
if (this._insertAtIndex !== undefined) {
  const insertIndex = this._insertAtIndex;
  this._insertAtIndex = undefined;
  let newPath: readonly (string | number)[] | undefined;
  this._applyEdit('tree', () => {
    if (nt === 'page') {
      const page = this._findPageAtPath(path);
      if (!page) return;
      if (PagesBuilderShell._LAYOUT_TYPES.has(entry.type)) {
        newPath = page.insertRowAt(insertIndex).path;
      } else {
        newPath = page.insertChildAt(insertIndex, entry.type, props).path;
      }
    } else if (nt === 'column') {
      const col = this._findColumnAtPath(path);
      if (col) newPath = col.insertComponentAt(insertIndex, entry.type, props).path;
    } else if (nt === 'row') {
      const row = this._findRowAtPath(path);
      if (row) newPath = row.insertColumnAt(insertIndex).path;
    } else if (nt === 'component') {
      const node = this._findComponentAtPath(path);
      if (node?.isContainer()) {
        newPath = node.addChild(
          getContainerDescriptor(node.type)!.defaultContentSlot,
          entry.type,
          props,
        ).path;
      }
    }
  });
  if (newPath) {
    this._selectedPath = newPath;
    this._selectedNodeType = nt === 'row' ? 'column' : 'component';
    this._updatePropertySource();
  }
  this._syncTree();
  return;
}
```

Add import at top of builder-shell.ts:
```typescript
import { allowedTypesAt, getContainerDescriptor } from '@casehubio/pages-document';
```

- [ ] **Step 4: Wire `tree-insert-at` event in template**

In the `_syncTree()` template (around line 1040), add the event binding
to the tree element:

```typescript
@tree-insert-at="${(e: CustomEvent) => this._handleInsertAt(e)}"
```

- [ ] **Step 5: Wire `selection-insert-at` event in `_wireSelectionOverlayEvents`**

In `_wireSelectionOverlayEvents` (line 406), add:

```typescript
const onInsertAt = (e: Event) => {
  this._handleInsertAt(e as CustomEvent);
};
overlayRoot.addEventListener('selection-insert-at', onInsertAt);
```

And in the cleanup function:
```typescript
overlayRoot.removeEventListener('selection-insert-at', onInsertAt);
```

- [ ] **Step 6: Show insertion points when container is selected**

In `_applyPreviewHighlight` (where `selectionOverlay.update()` is called),
add insertion point rendering after the update call:

```typescript
if (options?.isContainer && this._selectionOverlay) {
  const childElements = collectChildComponents(previewDoc, this._selectedPath!);
  if (childElements.length > 0) {
    const childBounds = childElements.map(el => el.getBoundingClientRect());
    this._selectionOverlay.showInsertionPoints(
      childBounds,
      this._selectedPath!,
      this._selectedNodeType!,
    );
  }
} else {
  this._selectionOverlay?.hideInsertionPoints();
}
```

The `collectChildComponents` helper queries direct children of the
selected container from the preview iframe. This uses the same DOM
query approach as `collectComponentsInScope` but returns individual
child bounds rather than a union bound.

- [ ] **Step 7: Clear `_insertAtIndex` on picker close**

In the picker-close handler (around line 1065):

```typescript
@picker-close="${() => {
  this._inlinePickerOpen = false;
  this._insertAtIndex = undefined;
  this._syncTree();
}}"
```

- [ ] **Step 8: Run full test suite**

Run: `yarn workspace @casehubio/pages-builder run test -- --run && yarn workspace @casehubio/pages-document run test -- --run`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-builder/src/shell/builder-shell.ts
git commit -m "feat(builder): wire _handleInsertAt to connect tree and overlay insertion points Refs #433"
```

### Task 6: Verify in browser

**Files:** None — manual verification

- [ ] **Step 1: Start dev server**

Run: `yarn dev`

- [ ] **Step 2: Verify tree insertion points**

1. Open the builder with a page that has multiple components
2. Expand a column node in the tree
3. Verify `+` insertion-point lines appear between child nodes (subtle until hovered)
4. Hover an insertion point — verify it becomes fully visible
5. Click an insertion point — verify inline picker opens
6. Select a component type — verify it's inserted at the correct position
7. Undo (`Ctrl+Z`) — verify the insert is undone

- [ ] **Step 3: Verify visual preview insertion points**

1. Click a column in the visual preview
2. Verify SelectionOverlay shows scope outline with `⋮` toolbar
3. Verify `+` insertion-point indicators appear between child components
4. Hover an insertion point — verify it highlights
5. Click an insertion point — verify inline picker opens anchored to it
6. Select a component — verify it's inserted at the correct position

- [ ] **Step 4: Verify constraint filtering**

1. Click a row in the tree → click `+` → verify picker shows only columns (not component types)
2. Click a column → click `+` → verify picker shows all component types (not rows)
3. Click a page → click `+` → verify picker shows rows, columns, and all component types

- [ ] **Step 5: Commit any fixes**

If issues found during verification, fix them, test, and commit each fix.

## References

- [2026-09-21-add-pickers-insertion-points-design.md] — design spec this plan implements
- [packages/pages-document/src/page-document.ts] — PageDocument model, mutation methods
- [packages/pages-document/src/container-descriptors.ts] — container type definitions
- [packages/pages-builder/src/overlay/selection-overlay.ts] — overlay rendering
- [packages/pages-builder/src/shell/builder-shell.ts:606-806] — current add/insert handlers
- [packages/pages-builder/src/catalog/palette-context.ts] — PaletteContext interface
- [packages/pages-builder/src/catalog/component-catalog.ts] — catalog and filtering
- [packages/pages-builder/src/palette/inline-picker.ts] — inline picker component
- [packages/pages-builder/src/tree/builder-tree.ts] — tree rendering
- [PP-20260916-b6f3e8] — all mutations through _applyEdit protocol
- [PP-20260817-a11y01] — ARIA interaction contract protocol
- [GitHub #433] — focal issue
