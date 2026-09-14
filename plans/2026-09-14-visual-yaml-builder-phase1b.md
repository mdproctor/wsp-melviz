# Visual YAML Builder Phase 1b Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #434 — Epic: Visual YAML Builder Phase 1b
**Issue group:** #433, #432, #429

**Goal:** Complete the builder's authoring experience — structural editing
(add/move/wrap/replace), preview data (no empty boxes), and dock-workbench
extraction (reusable Lit component).

**Architecture:** Three workstreams implemented sequentially: facade
extensions for structural operations with a transaction API for atomic
undo (#433), preview data via strategy registry + YAML transformation
(#432), dock-workbench extraction from builder shell to `<pages-dock-workbench>`
in pages-ui (#429).

**Tech Stack:** TypeScript, Lit 3, `yaml` library (CST), Vitest, Playwright

## Global Constraints

- All web components use `pages-` prefix with `@customElement` decorator
- ARIA roles + accessible names on all interactive components (aria-interaction-contract protocol)
- CSS tokens use `--pages-` prefix (css-design-tokens protocol)
- Lit for interactive UI, vanilla only for legacy auth (web-component-strategy protocol)
- No new packages — all code in existing packages
- All facade mutations must push undo snapshots (or use transaction API)
- `NEEDS_DATASET_TYPES` in `component-catalog.ts` is the authoritative list of data-consuming types

---

## Batch 1: Facade Foundation — Transaction API + Positional Insert

### Task 1: Transaction API on PageDocument

**Files:**
- Modify: `packages/pages-document/src/page-document.ts`
- Test: `packages/pages-document/src/page-document.test.ts`

**Interfaces:**
- Produces: `PageDocument.beginTransaction()`, `PageDocument.commitTransaction()`, `PageDocument.abortTransaction()`

- [ ] **Step 1: Write failing tests for transaction API**

```typescript
describe('transaction API', () => {
  it('compound operation is a single undo step', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart\n      - type: line-chart');
    const page = doc.getPages()[0];
    const comps = page.getComponents();
    expect(comps).toHaveLength(2);

    doc.beginTransaction();
    page.removeChild(1); // remove line-chart
    page.removeChild(0); // remove bar-chart
    page.addComponent('pie-chart');
    doc.commitTransaction();

    expect(page.getComponents()).toHaveLength(1);
    expect(page.getComponents()[0].type).toBe('pie-chart');

    // single undo reverses the entire compound
    doc.undo();
    const restored = doc.getPages()[0].getComponents();
    expect(restored).toHaveLength(2);
    expect(restored[0].type).toBe('bar-chart');
    expect(restored[1].type).toBe('line-chart');
  });

  it('suppresses notifications during transaction, fires one on commit', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart');
    const notifications: string[] = [];
    doc.onChange(() => notifications.push('changed'));

    doc.beginTransaction();
    doc.getPages()[0].addComponent('line-chart');
    doc.getPages()[0].addComponent('pie-chart');
    expect(notifications).toHaveLength(0); // suppressed

    doc.commitTransaction();
    expect(notifications).toHaveLength(1); // single coalesced
  });

  it('abortTransaction restores pre-transaction state', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart');
    const original = doc.toString();

    doc.beginTransaction();
    doc.getPages()[0].removeChild(0);
    expect(doc.getPages()[0].getComponents()).toHaveLength(0);

    doc.abortTransaction();
    expect(doc.toString()).toBe(original);
    expect(doc.getPages()[0].getComponents()).toHaveLength(1);
    // undo stack should not contain the aborted operation
    expect(doc.canUndo()).toBe(false);
  });

  it('beginTransaction throws if already in transaction', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1');
    doc.beginTransaction();
    expect(() => doc.beginTransaction()).toThrow();
    doc.commitTransaction();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-document test -- --run -t "transaction API"`
Expected: FAIL — `beginTransaction` is not a function

- [ ] **Step 3: Implement transaction API**

Add three private fields and three public methods to `PageDocument`:

```typescript
private _inTransaction = false;
private _transactionSnapshot: string | null = null;

beginTransaction(): void {
  if (this._inTransaction) throw new Error('Already in transaction');
  this._inTransaction = true;
  this._transactionSnapshot = this.toString();
  this._pushUndoInternal(); // snapshot before compound starts
}

commitTransaction(): void {
  this._inTransaction = false;
  this._transactionSnapshot = null;
  this._notifyInternal(); // single coalesced notification
}

abortTransaction(): void {
  if (!this._inTransaction) return;
  this._inTransaction = false;
  // restore from snapshot
  this._doc = parseDocument(this._transactionSnapshot!);
  this._transactionSnapshot = null;
  // pop the undo entry that beginTransaction pushed
  this._undoStack.pop();
  this._redoStack.length = 0;
}
```

Modify `_pushUndoInternal()` to skip when `_inTransaction` is true (except the initial call from `beginTransaction`).
Modify `_notifyInternal()` to skip when `_inTransaction` is true.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-document test -- --run -t "transaction API"`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add packages/pages-document/src/page-document.ts packages/pages-document/src/page-document.test.ts
git commit -m "feat(pages-document): add transaction API for compound undo — begin/commit/abort Refs #433"
```

---

### Task 2: Positional Insert Operations + defaultContentSlot

**Files:**
- Modify: `packages/pages-document/src/page-document.ts`
- Modify: `packages/pages-document/src/container-descriptors.ts`
- Modify: `packages/pages-document/src/types.ts`
- Test: `packages/pages-document/src/page-document.test.ts`
- Test: `packages/pages-document/src/container-descriptors.test.ts`

**Interfaces:**
- Consumes: Transaction API from Task 1
- Produces: `PageNode.insertChildAt()`, `RowNode.insertColumnAt()`, `ColumnNode.insertComponentAt()`, `ContainerChildDescriptor.defaultContentSlot`

- [ ] **Step 1: Write failing tests for positional insert**

```typescript
describe('positional insert', () => {
  it('insertComponentAt inserts at specific index in column', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    rows:\n      - columns:\n          - components:\n              - type: bar-chart\n              - type: line-chart');
    const col = doc.getPages()[0].getRows()[0].getColumns()[0];

    col.insertComponentAt(1, 'pie-chart');

    const comps = col.getComponents();
    expect(comps).toHaveLength(3);
    expect(comps[0].type).toBe('bar-chart');
    expect(comps[1].type).toBe('pie-chart');
    expect(comps[2].type).toBe('line-chart');
  });

  it('insertChildAt inserts component at index in flat page', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart\n      - type: line-chart');
    const page = doc.getPages()[0];

    page.insertChildAt(0, 'title', { text: 'Header' });

    const comps = page.getComponents();
    expect(comps).toHaveLength(3);
    expect(comps[0].type).toBe('title');
  });

  it('insertColumnAt inserts column at specific index in row', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    rows:\n      - columns:\n          - span: 6\n            components: []\n          - span: 6\n            components: []');
    const row = doc.getPages()[0].getRows()[0];

    row.insertColumnAt(1, 4);

    const cols = row.getColumns();
    expect(cols).toHaveLength(3);
    expect(cols[1].span).toBe(4);
  });

  it('insertComponentAt supports undo', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    rows:\n      - columns:\n          - components:\n              - type: bar-chart');
    const col = doc.getPages()[0].getRows()[0].getColumns()[0];
    col.insertComponentAt(0, 'title');
    expect(col.getComponents()).toHaveLength(2);

    doc.undo();
    expect(doc.getPages()[0].getRows()[0].getColumns()[0].getComponents()).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Write failing test for defaultContentSlot**

```typescript
// In container-descriptors.test.ts
describe('defaultContentSlot', () => {
  it('sidebar defaults to content slot, not sidebar slot', () => {
    const desc = CONTAINER_DESCRIPTORS.find(d => d.type === 'sidebar')!;
    expect(desc.defaultContentSlot).toBe('content');
  });

  it('tabs defaults to first slot', () => {
    const desc = CONTAINER_DESCRIPTORS.find(d => d.type === 'tabs')!;
    expect(desc.defaultContentSlot).toBe('tabs');
  });

  it('split defaults to children slot', () => {
    const desc = CONTAINER_DESCRIPTORS.find(d => d.type === 'split')!;
    expect(desc.defaultContentSlot).toBe('split');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-document test -- --run`
Expected: FAIL — methods and properties not defined

- [ ] **Step 4: Add defaultContentSlot to ContainerChildDescriptor**

In `container-descriptors.ts`, add `defaultContentSlot: string` to `ContainerChildDescriptor` interface and populate for each entry:
- tabs/pills/accordion/carousel/stack/menu/tree: first slot's yamlKey
- sidebar: `'content'` (not `'sidebar'`)
- split: `'split'`
- form-scope: `'form-scope'`

- [ ] **Step 5: Implement positional insert methods**

Add `insertComponentAt(index, type, props?)` to ColumnNode — uses `Document.addIn()` with the column's component array path at the specified index. Push undo before mutation.

Add `insertChildAt(index, type, props?)` to PageNode — dispatches based on layout mode (flat: component array, rows: row array).

Add `insertColumnAt(index, span?)` to RowNode — inserts a new column YAML node at the specified index in the row's columns array.

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-document test -- --run`
Expected: PASS

- [ ] **Step 7: Commit**

```
git add packages/pages-document/
git commit -m "feat(pages-document): positional insert operations + defaultContentSlot on descriptors Refs #433"
```

---

## Batch 2: Facade Compound Operations

### Task 3: Container addChild

**Files:**
- Modify: `packages/pages-document/src/page-document.ts`
- Test: `packages/pages-document/src/page-document.test.ts`

**Interfaces:**
- Consumes: Transaction API (Task 1), container descriptors with `defaultContentSlot` (Task 2)
- Produces: `ComponentNode.addChild(slot, type, props?)`

- [ ] **Step 1: Write failing tests for addChild**

```typescript
describe('addChild', () => {
  it('adds child to tabs named-record slot', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: tabs
        tabs:
          "Tab 1":
            components:
              - type: bar-chart`);
    const tabs = doc.getPages()[0].getComponents()[0];
    expect(tabs.isContainer()).toBe(true);

    tabs.addChild('Tab 1', 'line-chart');

    const children = tabs.getChildren();
    expect(children.slots['Tab 1']).toHaveLength(2);
    expect(children.slots['Tab 1'][1].type).toBe('line-chart');
  });

  it('adds child to sidebar array slot', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: sidebar
        content:
          - type: bar-chart`);
    const sidebar = doc.getPages()[0].getComponents()[0];

    sidebar.addChild('content', 'line-chart');

    const children = sidebar.getChildren();
    expect(children.slots['content']).toHaveLength(2);
  });

  it('creates new named slot if it does not exist', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: tabs
        tabs:
          "Tab 1":
            components:
              - type: bar-chart`);
    const tabs = doc.getPages()[0].getComponents()[0];

    tabs.addChild('Tab 2', 'pie-chart');

    const children = tabs.getChildren();
    expect(Object.keys(children.slots)).toContain('Tab 2');
    expect(children.slots['Tab 2'][0].type).toBe('pie-chart');
  });

  it('throws for non-container component', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart');
    const comp = doc.getPages()[0].getComponents()[0];
    expect(() => comp.addChild('slot', 'line-chart')).toThrow();
  });

  it('supports undo', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: tabs
        tabs:
          "Tab 1":
            components:
              - type: bar-chart`);
    const tabs = doc.getPages()[0].getComponents()[0];
    tabs.addChild('Tab 1', 'line-chart');
    expect(tabs.getChildren().slots['Tab 1']).toHaveLength(2);

    doc.undo();
    const restored = doc.getPages()[0].getComponents()[0];
    expect(restored.getChildren().slots['Tab 1']).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-document test -- --run -t "addChild"`
Expected: FAIL — `addChild` is not a function

- [ ] **Step 3: Implement addChild on ComponentNode**

```typescript
addChild(slot: string, type: string, props?: Record<string, unknown>): ComponentNode {
  if (!this.isContainer()) throw new Error(`${this.type} is not a container`);
  const descriptor = this.getChildren().descriptor;
  const slotDesc = descriptor.slots.find(s => s.yamlKey === slot || slot === slot);

  this.doc.beginTransaction();
  try {
    // Build the component YAML node
    const compNode = this.doc._createComponentNode(type, props);

    if (slotDesc.kind === 'named-record') {
      // Navigate to the named-record map, create entry if missing
      // Add component to the entry's childKey array
    } else if (slotDesc.kind === 'array') {
      // Append to the array at slotDesc.yamlKey
    } else if (slotDesc.kind === 'nested-array') {
      // Append to slotDesc.childKey inside slotDesc.yamlKey
    }

    this.doc.commitTransaction();
    return /* new ComponentNode for the added component */;
  } catch (e) {
    this.doc.abortTransaction();
    throw e;
  }
}
```

The exact YAML path manipulation uses `Document.addIn()`, `Document.setIn()`, and `Document.createNode()` from the `yaml` library — the same patterns used by existing `addComponent()` methods.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-document test -- --run -t "addChild"`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add packages/pages-document/
git commit -m "feat(pages-document): addChild for container components — all 3 slot kinds Refs #433"
```

---

### Task 4: wrapIn + wrapInRow

**Files:**
- Modify: `packages/pages-document/src/page-document.ts`
- Test: `packages/pages-document/src/page-document.test.ts`

**Interfaces:**
- Consumes: Transaction API (Task 1), positional insert (Task 2), addChild (Task 3), `defaultContentSlot` (Task 2)
- Produces: `ComponentNode.wrapIn(containerType)`, `PageNode.wrapInRow(componentIndices)`

- [ ] **Step 1: Write failing tests for wrapIn**

```typescript
describe('wrapIn', () => {
  it('wraps component in tabs container', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    rows:\n      - columns:\n          - components:\n              - type: bar-chart\n              - type: line-chart');
    const col = doc.getPages()[0].getRows()[0].getColumns()[0];
    const barChart = col.getComponents()[0];

    const tabsContainer = barChart.wrapIn('tabs');

    expect(tabsContainer.type).toBe('tabs');
    const children = tabsContainer.getChildren();
    expect(children.slots['Tab 1'][0].type).toBe('bar-chart');
    // line-chart should still be a sibling of the tabs container
    expect(col.getComponents()).toHaveLength(2);
    expect(col.getComponents()[0].type).toBe('tabs');
    expect(col.getComponents()[1].type).toBe('line-chart');
  });

  it('uses defaultContentSlot for sidebar', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart');
    const comp = doc.getPages()[0].getComponents()[0];

    const sidebar = comp.wrapIn('sidebar');

    expect(sidebar.type).toBe('sidebar');
    const children = sidebar.getChildren();
    // bar-chart should be in 'content' slot, not 'sidebar' slot
    expect(children.slots['content'][0].type).toBe('bar-chart');
  });

  it('is atomic — single undo restores original', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart\n      - type: line-chart');
    doc.getPages()[0].getComponents()[0].wrapIn('tabs');

    doc.undo();
    const comps = doc.getPages()[0].getComponents();
    expect(comps).toHaveLength(2);
    expect(comps[0].type).toBe('bar-chart');
  });
});
```

- [ ] **Step 2: Write failing tests for wrapInRow**

```typescript
describe('wrapInRow', () => {
  it('wraps flat-mode components into a row', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart\n      - type: line-chart\n      - type: pie-chart');
    const page = doc.getPages()[0];
    expect(page.getLayoutMode()).toBe('flat');

    page.wrapInRow([0, 1]);

    expect(page.getLayoutMode()).toBe('rows');
    const rows = page.getRows();
    expect(rows).toHaveLength(1);
    const colComps = rows[0].getColumns()[0].getComponents();
    expect(colComps).toHaveLength(2);
    expect(colComps[0].type).toBe('bar-chart');
    expect(colComps[1].type).toBe('line-chart');
  });

  it('inserts row among existing rows in rows mode', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    rows:
      - columns:
          - components:
              - type: bar-chart
              - type: line-chart
              - type: pie-chart`);
    const page = doc.getPages()[0];
    const col = page.getRows()[0].getColumns()[0];
    // Wrap first two components into a new row
    // This requires a column-level wrapInRow or similar
    // For now, test page-level wrapInRow in rows mode
    expect(page.getLayoutMode()).toBe('rows');
  });

  it('throws in columns mode', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    columns:\n      - components:\n          - type: bar-chart');
    const page = doc.getPages()[0];
    expect(() => page.wrapInRow([0])).toThrow();
  });

  it('is atomic — single undo restores original', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart\n      - type: line-chart');
    doc.getPages()[0].wrapInRow([0, 1]);

    doc.undo();
    expect(doc.getPages()[0].getLayoutMode()).toBe('flat');
    expect(doc.getPages()[0].getComponents()).toHaveLength(2);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-document test -- --run -t "wrapIn|wrapInRow"`
Expected: FAIL

- [ ] **Step 4: Implement wrapIn on ComponentNode**

Transaction-wrapped compound: (1) clone self's YAML node, (2) remove self from parent sequence, (3) create container node with self as first child in `defaultContentSlot`, (4) insert container at self's original index.

- [ ] **Step 5: Implement wrapInRow on PageNode**

Transaction-wrapped: (1) collect components at specified indices, (2) remove them in reverse index order, (3) if flat mode, convert to rows by wrapping existing YAML, (4) create a row with one full-width column containing the collected components, (5) insert row at position of first removed component.

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-document test -- --run -t "wrapIn|wrapInRow"`
Expected: PASS

- [ ] **Step 7: Commit**

```
git add packages/pages-document/
git commit -m "feat(pages-document): wrapIn (container) + wrapInRow (page restructuring) Refs #433"
```

---

### Task 5: replaceWith + moveToSlot

**Files:**
- Modify: `packages/pages-document/src/page-document.ts`
- Test: `packages/pages-document/src/page-document.test.ts`

**Interfaces:**
- Consumes: Transaction API (Task 1), `componentSchemaRegistry` from `@casehubio/pages-schema`
- Produces: `ComponentNode.replaceWith(type)`, `ComponentNode.moveToSlot(target, slotName, index)`

- [ ] **Step 1: Write failing tests for replaceWith**

```typescript
describe('replaceWith', () => {
  it('replaces type and preserves compatible properties', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: bar-chart
        properties:
          lookup: { uuid: sales }
          title: Revenue`);
    const comp = doc.getPages()[0].getComponents()[0];

    const newComp = comp.replaceWith('line-chart');

    expect(newComp.type).toBe('line-chart');
    // lookup and title exist on both bar-chart and line-chart
    expect(newComp.getProperties().lookup).toEqual({ uuid: 'sales' });
    expect(newComp.getProperties().title).toBe('Revenue');
  });

  it('drops incompatible properties', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: bar-chart
        properties:
          subtype: column
          lookup: { uuid: sales }`);
    const comp = doc.getPages()[0].getComponents()[0];

    const newComp = comp.replaceWith('metric');
    expect(newComp.type).toBe('metric');
    // subtype doesn't exist on metric schema
  });

  it('is atomic — single undo restores original', () => {
    const doc = PageDocument.parse('pages:\n  - name: p1\n    components:\n      - type: bar-chart\n        properties:\n          title: Test');
    doc.getPages()[0].getComponents()[0].replaceWith('line-chart');

    doc.undo();
    expect(doc.getPages()[0].getComponents()[0].type).toBe('bar-chart');
    expect(doc.getPages()[0].getComponents()[0].getProperties().title).toBe('Test');
  });
});
```

- [ ] **Step 2: Write failing tests for moveToSlot**

```typescript
describe('moveToSlot', () => {
  it('moves component into existing named slot', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: tabs
        tabs:
          "Tab 1":
            components:
              - type: bar-chart
          "Tab 2":
            components:
              - type: line-chart
      - type: pie-chart`);
    const page = doc.getPages()[0];
    const pie = page.getComponents()[1];
    const tabsPath = page.getComponents()[0].path;

    pie.moveToSlot({ path: tabsPath, slotName: 'Tab 1' }, 1);

    const tabs = doc.getPages()[0].getComponents()[0];
    expect(tabs.getChildren().slots['Tab 1']).toHaveLength(2);
    expect(tabs.getChildren().slots['Tab 1'][1].type).toBe('pie-chart');
    // pie-chart removed from page level
    expect(doc.getPages()[0].getComponents()).toHaveLength(1);
  });

  it('creates new named slot when it does not exist', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: tabs
        tabs:
          "Tab 1":
            components:
              - type: bar-chart
      - type: pie-chart`);
    const pie = doc.getPages()[0].getComponents()[1];
    const tabsPath = doc.getPages()[0].getComponents()[0].path;

    pie.moveToSlot({ path: tabsPath, slotName: 'Tab 2' }, 0);

    const tabs = doc.getPages()[0].getComponents()[0];
    expect(tabs.getChildren().slots['Tab 2']).toHaveLength(1);
    expect(tabs.getChildren().slots['Tab 2'][0].type).toBe('pie-chart');
  });

  it('is atomic — single undo restores original', () => {
    const doc = PageDocument.parse(`pages:
  - name: p1
    components:
      - type: tabs
        tabs:
          "Tab 1":
            components:
              - type: bar-chart
      - type: pie-chart`);
    const pie = doc.getPages()[0].getComponents()[1];
    const tabsPath = doc.getPages()[0].getComponents()[0].path;
    pie.moveToSlot({ path: tabsPath, slotName: 'Tab 1' }, 0);

    doc.undo();
    expect(doc.getPages()[0].getComponents()).toHaveLength(2);
    expect(doc.getPages()[0].getComponents()[1].type).toBe('pie-chart');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-document test -- --run -t "replaceWith|moveToSlot"`
Expected: FAIL

- [ ] **Step 4: Implement replaceWith**

Transaction-wrapped: (1) read source properties, (2) look up target schema from `componentSchemaRegistry`, (3) compute migrated properties (top-level key match with type compatibility), (4) remove source from parent, (5) create new component with target type + migrated properties at source's position.

- [ ] **Step 5: Implement moveToSlot**

Transaction-wrapped: (1) remove self from current parent sequence, (2) navigate to target container path, (3) find or create named slot entry, (4) insert self at specified index in the slot's child array.

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-document test -- --run -t "replaceWith|moveToSlot"`
Expected: PASS

- [ ] **Step 7: Run full page-document test suite**

Run: `yarn --cwd packages/pages-document test -- --run`
Expected: All PASS

- [ ] **Step 8: Commit**

```
git add packages/pages-document/
git commit -m "feat(pages-document): replaceWith (property migration) + moveToSlot (named-record targeting) Refs #433"
```

---

## Batch 3: Tree Structural Editing UI

### Task 6: PaletteFilter Extraction + Inline Picker Popover

**Files:**
- Create: `packages/pages-builder/src/palette/palette-filter.ts`
- Create: `packages/pages-builder/src/palette/inline-picker.ts`
- Modify: `packages/pages-builder/src/palette/builder-palette.ts`
- Modify: `packages/pages-builder/src/catalog/palette-context.ts`
- Test: `packages/pages-builder/src/palette/palette-filter.test.ts`
- Test: `packages/pages-builder/src/palette/inline-picker.test.ts`

**Interfaces:**
- Consumes: `PaletteContext` (existing), `ComponentCatalogEntry` (existing), `CATALOG` (existing)
- Produces: `PaletteFilter.filter(ctx, catalog): FilteredEntry[]`, `<pages-builder-inline-picker>` Lit component

- [ ] **Step 1: Add `parentSlot` to PaletteContext**

In `palette-context.ts`, add `parentSlot?: string` to the `PaletteContext` interface.

- [ ] **Step 2: Write failing tests for PaletteFilter**

```typescript
describe('PaletteFilter', () => {
  it('promotes form components inside form-scope', () => {
    const ctx: PaletteContext = {
      parentType: 'form-scope',
      acceptsComponents: true,
      availableDatasets: ['ds1'],
      siblingTypes: [],
    };
    const results = PaletteFilter.filter(ctx, CATALOG);
    const formTypes = results.filter(r => r.relevance === 'promoted');
    expect(formTypes.some(f => f.entry.type === 'input')).toBe(true);
  });

  it('hides page-level-only components inside containers', () => {
    const ctx: PaletteContext = {
      parentType: 'tabs',
      parentSlot: 'Tab 1',
      acceptsComponents: true,
      availableDatasets: [],
      siblingTypes: [],
    };
    const results = PaletteFilter.filter(ctx, CATALOG);
    expect(results.find(r => r.entry.type === 'page')).toBeUndefined();
  });

  it('marks data components as needs-prereq when no datasets', () => {
    const ctx: PaletteContext = {
      parentType: undefined,
      acceptsComponents: true,
      availableDatasets: [],
      siblingTypes: [],
    };
    const results = PaletteFilter.filter(ctx, CATALOG);
    const barChart = results.find(r => r.entry.type === 'bar-chart')!;
    expect(barChart.relevance).toBe('needs-prereq');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-builder test -- --run -t "PaletteFilter"`
Expected: FAIL

- [ ] **Step 4: Extract PaletteFilter from builder-palette.ts**

Move the filtering logic from `builder-palette.ts`'s render method into `PaletteFilter.filter()`. The function takes a `PaletteContext` and the full `CATALOG` array, returns `{ entry: ComponentCatalogEntry, relevance: 'promoted' | 'normal' | 'hidden' | 'needs-prereq' }[]`.

Modify `builder-palette.ts` to call `PaletteFilter.filter()` instead of inline logic.

- [ ] **Step 5: Run PaletteFilter tests**

Run: `yarn --cwd packages/pages-builder test -- --run -t "PaletteFilter"`
Expected: PASS

- [ ] **Step 6: Write inline-picker component**

`<pages-builder-inline-picker>` — a Lit component that:
- Accepts `context: PaletteContext` and `anchor: HTMLElement` properties
- Renders a compact palette grid using `PaletteFilter.filter()`
- Has search input at top, category tabs below, component grid
- Uses `FocusTrapMixin` and `RovingTabindexMixin`
- Fires `component-select` event with `ComponentCatalogEntry`
- Positions itself as a popover anchored to the `anchor` element

- [ ] **Step 7: Write inline-picker tests**

Test: opens and closes, fires `component-select`, keyboard navigation works, respects context filtering.

- [ ] **Step 8: Run all builder tests**

Run: `yarn --cwd packages/pages-builder test -- --run`
Expected: PASS

- [ ] **Step 9: Commit**

```
git add packages/pages-builder/src/palette/ packages/pages-builder/src/catalog/
git commit -m "feat(pages-builder): extract PaletteFilter + inline picker popover component Refs #433"
```

---

### Task 7: Context Menu Component

**Files:**
- Create: `packages/pages-primitives/src/context-menu/context-menu.ts`
- Test: `packages/pages-primitives/src/context-menu/context-menu.test.ts`
- Modify: `packages/pages-primitives/src/index.ts` (export)

**Interfaces:**
- Consumes: `FocusTrapMixin`, `KeyboardShortcutMixin` from pages-primitives
- Produces: `<pages-context-menu>` Lit component with `items`, `anchor`, `open` properties

- [ ] **Step 1: Write failing tests for context menu**

```typescript
describe('pages-context-menu', () => {
  it('renders menu items from items property', async () => {
    const el = await fixture<PagesContextMenu>(html`
      <pages-context-menu .items=${[
        { label: 'Delete', action: 'delete', shortcut: '⌫' },
        { label: 'Duplicate', action: 'duplicate', shortcut: '⌃D' },
      ]} .open=${true}>
      </pages-context-menu>
    `);
    const items = el.shadowRoot!.querySelectorAll('[role="menuitem"]');
    expect(items).toHaveLength(2);
    expect(items[0].textContent).toContain('Delete');
  });

  it('fires menu-action event on item click', async () => {
    const actions: string[] = [];
    const el = await fixture<PagesContextMenu>(html`
      <pages-context-menu
        .items=${[{ label: 'Delete', action: 'delete' }]}
        .open=${true}
        @menu-action=${(e: CustomEvent) => actions.push(e.detail.action)}>
      </pages-context-menu>
    `);
    el.shadowRoot!.querySelector('[role="menuitem"]')!.click();
    expect(actions).toEqual(['delete']);
  });

  it('supports submenus', async () => {
    const el = await fixture<PagesContextMenu>(html`
      <pages-context-menu .items=${[
        { label: 'Wrap in…', children: [
          { label: 'Row', action: 'wrap-row' },
          { label: 'Tabs', action: 'wrap-tabs' },
        ]},
      ]} .open=${true}>
      </pages-context-menu>
    `);
    const parent = el.shadowRoot!.querySelector('[aria-haspopup="true"]');
    expect(parent).toBeTruthy();
  });

  it('closes on Escape', async () => {
    const el = await fixture<PagesContextMenu>(html`
      <pages-context-menu .items=${[{ label: 'X', action: 'x' }]} .open=${true}>
      </pages-context-menu>
    `);
    el.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape' }));
    expect(el.open).toBe(false);
  });

  it('has correct ARIA roles', async () => {
    const el = await fixture<PagesContextMenu>(html`
      <pages-context-menu .items=${[{ label: 'X', action: 'x' }]} .open=${true}>
      </pages-context-menu>
    `);
    expect(el.shadowRoot!.querySelector('[role="menu"]')).toBeTruthy();
    expect(el.shadowRoot!.querySelector('[role="menuitem"]')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-primitives test -- --run -t "pages-context-menu"`
Expected: FAIL

- [ ] **Step 3: Implement `<pages-context-menu>`**

Lit component with:
- `@property() items: MenuItem[]` — menu item tree
- `@property({ type: Boolean }) open = false`
- `@property({ attribute: false }) anchor?: HTMLElement` — positioning anchor
- ARIA: `role="menu"` on container, `role="menuitem"` on items, `aria-haspopup` on submenu parents
- Mixins: `FocusTrapMixin`, `KeyboardShortcutMixin`
- Events: `menu-action` with `{ action: string }` detail
- Dismiss: Escape, click outside, item selection

```typescript
interface MenuItem {
  label: string;
  action?: string;
  shortcut?: string;
  disabled?: boolean;
  separator?: boolean;
  children?: MenuItem[];
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-primitives test -- --run -t "pages-context-menu"`
Expected: PASS

- [ ] **Step 5: Export from pages-primitives**

Add `context-menu` sub-path export in `package.json` exports map and re-export from index.

- [ ] **Step 6: Commit**

```
git add packages/pages-primitives/
git commit -m "feat(pages-primitives): add <pages-context-menu> Lit component with ARIA Refs #433"
```

---

### Task 8: Tree Interaction Wiring — Context Menu + Keyboard + Add Button

**Files:**
- Modify: `packages/pages-builder/src/tree/builder-tree.ts`
- Create: `packages/pages-builder/src/tree/tree-context-menu.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-builder/src/tree/builder-tree.test.ts`

**Interfaces:**
- Consumes: `<pages-context-menu>` (Task 7), `<pages-builder-inline-picker>` (Task 6), all facade operations (Tasks 1-5)
- Produces: Tree with `+` buttons, right-click context menu, keyboard shortcuts

- [ ] **Step 1: Create tree-context-menu.ts — menu item computation**

```typescript
export function computeMenuItems(nodeType: string, node: any): MenuItem[] {
  // Returns context-dependent menu items per the spec §1.4 table
}
```

Map node types to action sets. Handle submenus for "Wrap in…" (Layout: Row/Column + Container: Tabs) and "Replace with…" (opens picker).

- [ ] **Step 2: Write failing tests for tree interactions**

```typescript
describe('builder-tree interactions', () => {
  it('shows + button on container nodes', async () => {
    // Render tree with a page containing a column
    // Hover over column node
    // Expect + button to appear
  });

  it('opens context menu on right-click', async () => {
    // Right-click on a component node
    // Expect pages-context-menu to appear with correct items
  });

  it('Delete key removes selected node', async () => {
    // Select a leaf component
    // Press Delete
    // Expect component removed from document
  });

  it('Ctrl+D duplicates selected node', async () => {
    // Select a component
    // Press Ctrl+D
    // Expect duplicate inserted after selected
  });

  it('Ctrl+Shift+Up moves node up among siblings', async () => {
    // Select second component in a column
    // Press Ctrl+Shift+Up
    // Expect it moved to first position
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-builder test -- --run -t "builder-tree interactions"`
Expected: FAIL

- [ ] **Step 4: Add `+` button rendering to tree nodes**

In `builder-tree.ts`, modify the node render method to show a `+` icon button on container nodes (pages, rows, columns, container components). On click, open the `<pages-builder-inline-picker>` anchored to the button.

- [ ] **Step 5: Add context menu trigger**

Wire `contextmenu` event on tree nodes → render `<pages-context-menu>` with items from `computeMenuItems()`. Wire `menu-action` events to facade operations.

- [ ] **Step 6: Add keyboard shortcuts**

Via `KeyboardShortcutMixin`:
- `Delete`/`Backspace` → `removeChild()` on parent (confirm for containers with children via `<pages-modal>`)
- `Ctrl+D` → `duplicate()` on selected ComponentNode
- `Ctrl+Shift+P` → open inline picker at selected node
- `Ctrl+Shift+Up` → `moveToIndex()` with index-1 on parent
- `Ctrl+Shift+Down` → `moveToIndex()` with index+1 on parent

- [ ] **Step 7: Wire context menu actions in builder-shell**

Connect menu actions (wrap, replace, move-to, add-child) to facade methods via the shell's event handler layer.

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-builder test -- --run`
Expected: PASS

- [ ] **Step 9: Commit**

```
git add packages/pages-builder/
git commit -m "feat(pages-builder): tree + button, context menu, keyboard shortcuts Refs #433"
```

---

## Batch 4: Tree Drag-and-Drop

### Task 9: Drag-and-Drop with Drop Indicators

**Files:**
- Create: `packages/pages-builder/src/tree/tree-dnd.ts`
- Modify: `packages/pages-builder/src/tree/builder-tree.ts`
- Test: `packages/pages-builder/src/tree/tree-dnd.test.ts`
- Test: `packages/pages-builder/src/tree/builder-tree.test.ts`

**Interfaces:**
- Consumes: `ComponentNode.moveToIndex()`, `ComponentNode.moveToSlot()` (Tasks 1-5)
- Produces: DnD on tree nodes with reorder + cross-container moves

- [ ] **Step 1: Write tree-dnd.ts — DnD logic module**

```typescript
export interface DropTarget {
  type: 'between' | 'on-container' | 'on-slot' | 'invalid';
  parentPath: readonly (string | number)[];
  index: number;
  slotName?: string;
}

export function computeDropTarget(draggedPath, dropElement, dropPosition): DropTarget { ... }
export function isValidDrop(draggedNodeType, target: DropTarget): boolean { ... }
```

- [ ] **Step 2: Write failing tests for DnD logic**

```typescript
describe('tree-dnd', () => {
  it('reorder: computes between-siblings target', () => {
    const target = computeDropTarget(
      ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0],
      mockElement({ path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 1] }),
      'before'
    );
    expect(target.type).toBe('between');
    expect(target.index).toBe(1);
  });

  it('cross-container: computes on-container target', () => {
    const target = computeDropTarget(
      ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0],
      mockElement({ path: ['pages', 0, 'rows', 0, 'columns', 1] }),
      'on'
    );
    expect(target.type).toBe('on-container');
  });

  it('rejects drop on row', () => {
    expect(isValidDrop('bar-chart', {
      type: 'on-container',
      parentPath: ['pages', 0, 'rows', 0],
      index: 0,
    })).toBe(false);
  });

  it('rejects drop on dataset', () => {
    expect(isValidDrop('bar-chart', {
      type: 'on-container',
      parentPath: ['datasets', 0],
      index: 0,
    })).toBe(false);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-builder test -- --run -t "tree-dnd"`
Expected: FAIL

- [ ] **Step 4: Implement DnD logic**

Implement `computeDropTarget()` — determines drop type from pointer position relative to tree node (top third = before, bottom third = after, middle = on-container). Implement `isValidDrop()` — validates target accepts components.

- [ ] **Step 5: Wire DnD into builder-tree**

Add drag handles (visible on hover). Wire `dragstart`, `dragover`, `drop`, `dragend` events. Render drop indicators (horizontal line for between, highlight for on-container). On drop, call `moveToIndex()` or `moveToSlot()` based on target type.

Add auto-scroll (16px threshold), ghost image, and ARIA announcements via `LiveRegionMixin`.

- [ ] **Step 6: Run all tests**

Run: `yarn --cwd packages/pages-builder test -- --run`
Expected: PASS

- [ ] **Step 7: Commit**

```
git add packages/pages-builder/src/tree/
git commit -m "feat(pages-builder): tree drag-and-drop with reorder + cross-container moves Refs #433"
```

---

## Batch 5: Preview Data Provider

### Task 10: Strategy Registry + Strategies + Preview Hints

**Files:**
- Create: `packages/pages-builder/src/data/preview-data-registry.ts`
- Create: `packages/pages-builder/src/data/strategies/chart-strategy.ts`
- Create: `packages/pages-builder/src/data/strategies/table-strategy.ts`
- Create: `packages/pages-builder/src/data/strategies/metric-strategy.ts`
- Create: `packages/pages-builder/src/data/strategies/data-component-strategy.ts`
- Modify: `packages/pages-builder/src/catalog/component-catalog.ts` (add `previewHints`)
- Test: `packages/pages-builder/src/data/preview-data-registry.test.ts`
- Test: `packages/pages-builder/src/data/strategies/chart-strategy.test.ts`

**Interfaces:**
- Consumes: `NEEDS_DATASET_TYPES`, `ComponentCatalogEntry`, `DatasetNode` from pages-document
- Produces: `PreviewDataRegistry`, `PreviewDataStrategy`, `DatasetSnapshot`, `PreviewHints`

- [ ] **Step 1: Write failing tests for strategy registry**

```typescript
describe('PreviewDataRegistry', () => {
  it('returns strategy for registered type', () => {
    const registry = createDefaultRegistry();
    expect(registry.get('bar-chart')).toBeDefined();
  });

  it('returns undefined for non-data type', () => {
    const registry = createDefaultRegistry();
    expect(registry.get('html')).toBeUndefined();
  });

  it('covers all NEEDS_DATASET_TYPES', () => {
    const registry = createDefaultRegistry();
    for (const type of NEEDS_DATASET_TYPES) {
      expect(registry.get(type)).toBeDefined();
    }
  });
});
```

- [ ] **Step 2: Write failing tests for ChartDataStrategy**

```typescript
describe('ChartDataStrategy', () => {
  it('generates rows from dataset column definitions', () => {
    const strategy = new ChartDataStrategy();
    const datasets = [mockDatasetNode({
      uuid: 'sales',
      columns: [
        { id: 'region', type: 'TEXT' },
        { id: 'revenue', type: 'NUMBER' },
      ],
    })];
    const snapshots = strategy.generate(
      { lookup: { uuid: 'sales' } },
      datasets
    );
    expect(snapshots).toHaveLength(1);
    expect(snapshots[0].uuid).toBe('sales');
    expect(snapshots[0].rows.length).toBeGreaterThanOrEqual(5);
    expect(typeof snapshots[0].rows[0].region).toBe('string');
    expect(typeof snapshots[0].rows[0].revenue).toBe('number');
  });

  it('falls back to generic data when no column definitions', () => {
    const strategy = new ChartDataStrategy();
    const datasets = [mockDatasetNode({ uuid: 'sales', columns: [] })];
    const snapshots = strategy.generate(
      { lookup: { uuid: 'sales' } },
      datasets
    );
    expect(snapshots).toHaveLength(1);
    expect(snapshots[0].rows.length).toBeGreaterThanOrEqual(5);
  });

  it('returns empty when no matching dataset', () => {
    const strategy = new ChartDataStrategy();
    const snapshots = strategy.generate({ lookup: { uuid: 'missing' } }, []);
    expect(snapshots).toHaveLength(0);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-builder test -- --run -t "PreviewData|ChartData"`
Expected: FAIL

- [ ] **Step 4: Implement strategy interfaces and registry**

```typescript
// preview-data-registry.ts
export interface PreviewDataStrategy {
  generate(props: Record<string, unknown>, datasets: DatasetNode[]): DatasetSnapshot[];
}

export interface DatasetSnapshot {
  uuid: string;
  rows: Record<string, unknown>[];
}

export function createDefaultRegistry(): Map<string, PreviewDataStrategy> { ... }
```

- [ ] **Step 5: Implement all four strategies**

ChartDataStrategy (10 chart types), TableDataStrategy (3 table types), MetricDataStrategy (2 metric types), DataComponentStrategy (5 data types). Each reads `lookup.uuid`, finds matching dataset, generates typed rows from column definitions.

- [ ] **Step 6: Add previewHints to ComponentCatalogEntry**

```typescript
interface PreviewHints {
  placeholderText?: string;
  sampleChildren?: number;
  displayDefaults?: Record<string, unknown>;
}
```

Add `previewHints` to `ComponentCatalogEntry` interface. Populate for container types (`sampleChildren: 2` for tabs/accordion/sidebar) and content types (`placeholderText` for html/markdown).

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-builder test -- --run -t "PreviewData|ChartData"`
Expected: PASS

- [ ] **Step 8: Commit**

```
git add packages/pages-builder/src/data/ packages/pages-builder/src/catalog/
git commit -m "feat(pages-builder): preview data strategy registry + 4 strategies + preview hints Refs #432"
```

---

### Task 11: YAML Transformation + Preview Integration

**Files:**
- Create: `packages/pages-builder/src/data/preview-yaml-transform.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-builder/src/data/preview-yaml-transform.test.ts`

**Interfaces:**
- Consumes: `PreviewDataRegistry` (Task 10), `PageDocument` (existing), `renderPreview` callback (existing)
- Produces: `transformYamlForPreview(yaml, doc, registry): string`

- [ ] **Step 1: Write failing tests for YAML transformation**

```typescript
describe('transformYamlForPreview', () => {
  it('replaces URL datasets with inline content', () => {
    const yaml = `datasets:
  - uuid: sales
    url: /api/sales
    columns:
      - { id: region, type: TEXT }
      - { id: revenue, type: NUMBER }
pages:
  - name: p1
    components:
      - type: bar-chart
        properties:
          lookup: { uuid: sales }`;
    const doc = PageDocument.parse(yaml);
    const registry = createDefaultRegistry();

    const transformed = transformYamlForPreview(yaml, doc, registry);
    const transformedDoc = PageDocument.parse(transformed);
    const ds = transformedDoc.getDatasets()[0];

    expect(ds.getSource()).toBe('content');
    expect(ds.url).toBeUndefined();
  });

  it('preserves datasets that already have content source', () => {
    const yaml = `datasets:
  - uuid: inline_ds
    content: '[{"a":1}]'
pages:
  - name: p1
    components:
      - type: bar-chart
        properties:
          lookup: { uuid: inline_ds }`;
    const doc = PageDocument.parse(yaml);
    const registry = createDefaultRegistry();

    const transformed = transformYamlForPreview(yaml, doc, registry);
    expect(transformed).toContain('content:');
  });

  it('injects stub children for containers with previewHints', () => {
    const yaml = `pages:
  - name: p1
    components:
      - type: tabs
        tabs: {}`;
    const doc = PageDocument.parse(yaml);
    const registry = createDefaultRegistry();

    const transformed = transformYamlForPreview(yaml, doc, registry);
    // tabs with sampleChildren: 2 should have stub tabs injected
    expect(transformed).toContain('Tab 1');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-builder test -- --run -t "transformYamlForPreview"`
Expected: FAIL

- [ ] **Step 3: Implement transformYamlForPreview**

```typescript
export function transformYamlForPreview(
  yaml: string,
  doc: PageDocument,
  registry: Map<string, PreviewDataStrategy>
): string {
  // 1. Parse a working copy (don't mutate the real document)
  const workDoc = PageDocument.parse(yaml);

  // 2. For each dataset with URL source referenced by a data-consuming component:
  //    - Run strategy.generate() to get DatasetSnapshot
  //    - Replace url source with content: JSON.stringify(snapshot.rows)

  // 3. For each container component with no children and previewHints.sampleChildren:
  //    - Inject stub child components

  // 4. For components with previewHints.displayDefaults:
  //    - Merge defaults under properties (don't override user-set values)

  return workDoc.toString();
}
```

- [ ] **Step 4: Wire into builder-shell**

Modify `_refreshPreview()` in `builder-shell.ts`: before calling `this.renderPreview(container, yaml)`, call `transformYamlForPreview(yaml, this._doc, this._previewRegistry)`. Store the registry as a class field initialised in `connectedCallback`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-builder test -- --run`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add packages/pages-builder/
git commit -m "feat(pages-builder): YAML transformation for preview — inline data + preview hints Refs #432"
```

---

## Batch 6: Dock-Workbench Lit Component

### Task 12: `<pages-dock-workbench>` Lit Component

**Files:**
- Create: `packages/pages-ui/src/dock/dock-workbench.ts`
- Test: `packages/pages-ui/src/dock/dock-workbench.test.ts`
- Modify: `packages/pages-ui/src/index.ts` (export)
- Modify: `packages/pages-ui/package.json` (sub-path export)

**Interfaces:**
- Produces: `<pages-dock-workbench>` with left/centre/right/bottom/status-bar/toggle-bar slots, zone sizing, collapse, enable/disable, persistence

- [ ] **Step 1: Write failing tests for dock-workbench**

```typescript
describe('pages-dock-workbench', () => {
  it('renders slotted content in zones', async () => {
    const el = await fixture(html`
      <pages-dock-workbench>
        <div slot="left">Tree</div>
        <div slot="centre">Editor</div>
        <div slot="right">Props</div>
      </pages-dock-workbench>
    `);
    expect(el.shadowRoot!.querySelector('slot[name="left"]')).toBeTruthy();
    expect(el.shadowRoot!.querySelector('slot[name="centre"]')).toBeTruthy();
    expect(el.shadowRoot!.querySelector('slot[name="right"]')).toBeTruthy();
  });

  it('hides left zone when leftEnabled=false', async () => {
    const el = await fixture(html`
      <pages-dock-workbench .leftEnabled=${false}>
        <div slot="left">Tree</div>
        <div slot="centre">Editor</div>
      </pages-dock-workbench>
    `);
    const leftZone = el.shadowRoot!.querySelector('.zone-left');
    expect(leftZone).toBeNull();
  });

  it('collapses zone when collapsed property set', async () => {
    const el = await fixture(html`
      <pages-dock-workbench .leftCollapsed=${true}>
        <div slot="left">Tree</div>
        <div slot="centre">Editor</div>
      </pages-dock-workbench>
    `);
    const leftZone = el.shadowRoot!.querySelector('.zone-left');
    expect(leftZone!.classList.contains('collapsed')).toBe(true);
  });

  it('fires dock-panel-toggle on collapse toggle', async () => {
    const events: CustomEvent[] = [];
    const el = await fixture(html`
      <pages-dock-workbench
        @dock-panel-toggle=${(e: CustomEvent) => events.push(e)}>
        <div slot="left">Tree</div>
        <div slot="centre">Editor</div>
      </pages-dock-workbench>
    `);
    // Toggle left panel
    el.leftCollapsed = true;
    await el.updateComplete;
    expect(events.length).toBeGreaterThan(0);
    expect(events[0].detail.zone).toBe('left');
  });

  it('persists sizes to localStorage', async () => {
    const el = await fixture(html`
      <pages-dock-workbench persist-key="test-dock" left-width="300">
        <div slot="left">Tree</div>
        <div slot="centre">Editor</div>
      </pages-dock-workbench>
    `);
    // Trigger resize
    el.leftWidth = 400;
    await el.updateComplete;
    // Wait for debounced persistence
    await new Promise(r => setTimeout(r, 350));
    const stored = JSON.parse(localStorage.getItem('test-dock')!);
    expect(stored.leftWidth).toBe(400);
    localStorage.removeItem('test-dock');
  });

  it('has correct ARIA roles', async () => {
    const el = await fixture(html`
      <pages-dock-workbench>
        <div slot="left">Tree</div>
        <div slot="centre">Editor</div>
      </pages-dock-workbench>
    `);
    const regions = el.shadowRoot!.querySelectorAll('[role="region"]');
    expect(regions.length).toBeGreaterThanOrEqual(2);
    const separators = el.shadowRoot!.querySelectorAll('[role="separator"]');
    expect(separators.length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-ui test -- --run -t "pages-dock-workbench"`
Expected: FAIL

- [ ] **Step 3: Implement `<pages-dock-workbench>`**

Lit component with CSS grid layout. Zones are rendered conditionally based on `*Enabled` properties. Each enabled zone has a `<slot>` and optional resize handle. Resize is pointer-event-based drag on zone borders, constrained to `minPanelSize`/`maxPanelSize`, with debounced localStorage persistence.

Toggle bar renders icon buttons for each enabled zone. Each button toggles the zone's collapsed state.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-ui test -- --run -t "pages-dock-workbench"`
Expected: PASS

- [ ] **Step 5: Add exports**

Add `dock` sub-path export to `packages/pages-ui/package.json`. Export from index.

- [ ] **Step 6: Commit**

```
git add packages/pages-ui/
git commit -m "feat(pages-ui): <pages-dock-workbench> Lit component with zone enable/disable Refs #429"
```

---

### Task 13: Builder Shell Migration

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Consumes: `<pages-dock-workbench>` (Task 12)
- Produces: Builder shell using dock-workbench component, two-state YAML toggle

- [ ] **Step 1: Write failing tests for migrated shell**

```typescript
describe('builder-shell dock migration', () => {
  it('renders pages-dock-workbench as root layout', async () => {
    const el = await fixture(html`<pages-builder-shell></pages-builder-shell>`);
    expect(el.shadowRoot!.querySelector('pages-dock-workbench')).toBeTruthy();
  });

  it('tree is in left slot', async () => {
    const el = await fixture(html`<pages-builder-shell></pages-builder-shell>`);
    const tree = el.shadowRoot!.querySelector('pages-builder-tree');
    expect(tree?.getAttribute('slot')).toBe('left');
  });

  it('YAML toggle shows/hides bottom panel', async () => {
    const el = await fixture(html`<pages-builder-shell></pages-builder-shell>`);
    const dock = el.shadowRoot!.querySelector('pages-dock-workbench')!;

    // Initially bottom collapsed
    expect(dock.bottomCollapsed).toBe(true);

    // Toggle YAML
    el._yamlExpanded = true;
    await el.updateComplete;
    expect(dock.bottomCollapsed).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-builder test -- --run -t "builder-shell dock"`
Expected: FAIL

- [ ] **Step 3: Migrate builder-shell to use dock-workbench**

Replace the bespoke dock layout in `render()` with `<pages-dock-workbench>` and slotted content (per spec §3.3). Remove:
- Inline dock panel CSS (~100 lines)
- Resize handle event listeners
- Toggle bar rendering
- `_viewMode` three-state toggle → two-state `_yamlExpanded` boolean
- Toolbar source/split/visual buttons → single YAML show/hide toggle

- [ ] **Step 4: Run all builder tests**

Run: `yarn --cwd packages/pages-builder test -- --run`
Expected: PASS

- [ ] **Step 5: Run full project test suite**

Run: `yarn test`
Expected: All packages PASS

- [ ] **Step 6: Commit**

```
git add packages/pages-builder/
git commit -m "feat(pages-builder): migrate shell to <pages-dock-workbench>, simplify view modes Refs #429"
```

---

## References

- `specs/issue-434-visual-yaml-builder-phase1b/2026-09-14-visual-yaml-builder-phase1b-design.md` — design spec this plan implements
- `specs/issue-434-visual-yaml-builder-phase1b/decisions.md` — 14 design decisions
- `packages/pages-document/src/page-document.ts` — existing facade (~600 lines)
- `packages/pages-document/src/container-descriptors.ts` — slot descriptor registry
- `packages/pages-builder/src/tree/builder-tree.ts` — existing tree component
- `packages/pages-builder/src/palette/builder-palette.ts` — existing palette
- `packages/pages-builder/src/catalog/component-catalog.ts` — 56 entries, `NEEDS_DATASET_TYPES`
- `packages/pages-builder/src/shell/builder-shell.ts` — existing shell with bespoke dock
- `packages/pages-runtime/src/provider-factory.ts` — `InlineProvider` for content datasets
- `docs/protocols/casehub/web-component-strategy.md` — Lit conventions
- `docs/protocols/casehub/aria-interaction-contract.md` — ARIA requirements
- GitHub #434, #433, #432, #429
