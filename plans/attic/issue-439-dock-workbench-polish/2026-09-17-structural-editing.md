# Structural Editing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #433 — Tree/visual structural editing
**Issue group:** #433, #439

**Goal:** Add clipboard-driven cut/copy/paste structural editing to the YAML builder tree, with guided valid-target highlighting and cross-file support.

**Architecture:** Global `BuilderClipboard` singleton holds YAML fragment metadata and insert-mode state. Tree items get four inline buttons (Add, Insert, Cut, Copy). When clipboard has content, Add/Insert paste instead of opening the picker. Insert mode highlights valid targets; Esc exits mode without clearing clipboard.

**Tech Stack:** Lit, CodeMirror, YAML library, Clipboard API

## Global Constraints

- All new modules in `packages/pages-builder/src/clipboard/`
- TDD: every test written before implementation
- Use `ide_insert_member` / `ide_replace_member` for code edits
- Use `ide_refactor_rename` / `ide_move_file` for renames/moves
- Reuse existing `PaletteContext` / `contextRelevance` for validation
- System clipboard via `navigator.clipboard.writeText()` / `readText()`

---

## Batch 1: Clipboard Foundation

### Task 1: BuilderClipboard singleton

**Files:**
- Create: `packages/pages-builder/src/clipboard/builder-clipboard.ts`
- Test: `packages/pages-builder/src/clipboard/builder-clipboard.test.ts`

**Interfaces:**
- Produces: `BuilderClipboard` class with `fragment: string | null`, `fragmentType: string | null`, `operation: 'cut' | 'copy' | null`, `insertMode: boolean`, `set(fragment, type, op)`, `clear()`, `setInsertMode(boolean)`, `subscribe(cb): () => void`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi } from 'vitest';
import { BuilderClipboard } from './builder-clipboard.js';

describe('BuilderClipboard', () => {
  it('starts empty', () => {
    const cb = new BuilderClipboard();
    expect(cb.fragment).toBeNull();
    expect(cb.fragmentType).toBeNull();
    expect(cb.operation).toBeNull();
    expect(cb.insertMode).toBe(false);
  });

  it('set populates state and enters insert mode', () => {
    const cb = new BuilderClipboard();
    cb.set('type: metric', 'component', 'copy');
    expect(cb.fragment).toBe('type: metric');
    expect(cb.fragmentType).toBe('component');
    expect(cb.operation).toBe('copy');
    expect(cb.insertMode).toBe(true);
  });

  it('clear resets all state', () => {
    const cb = new BuilderClipboard();
    cb.set('type: metric', 'component', 'cut');
    cb.clear();
    expect(cb.fragment).toBeNull();
    expect(cb.insertMode).toBe(false);
  });

  it('setInsertMode toggles mode without clearing fragment', () => {
    const cb = new BuilderClipboard();
    cb.set('type: metric', 'component', 'copy');
    cb.setInsertMode(false);
    expect(cb.insertMode).toBe(false);
    expect(cb.fragment).toBe('type: metric');
  });

  it('notifies subscribers on change', () => {
    const cb = new BuilderClipboard();
    const spy = vi.fn();
    cb.subscribe(spy);
    cb.set('type: metric', 'component', 'copy');
    expect(spy).toHaveBeenCalledTimes(1);
  });

  it('unsubscribe stops notifications', () => {
    const cb = new BuilderClipboard();
    const spy = vi.fn();
    const unsub = cb.subscribe(spy);
    unsub();
    cb.set('type: metric', 'component', 'copy');
    expect(spy).not.toHaveBeenCalled();
  });

  it('is a singleton', () => {
    const { getClipboard } = await import('./builder-clipboard.js');
    expect(getClipboard()).toBe(getClipboard());
  });
});
```

- [ ] **Step 2: Run tests — verify failure**

Run: `yarn workspace @casehubio/pages-builder vitest run src/clipboard/builder-clipboard.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement BuilderClipboard**

```typescript
export class BuilderClipboard {
  fragment: string | null = null;
  fragmentType: string | null = null;
  operation: 'cut' | 'copy' | null = null;
  insertMode = false;

  private _listeners: (() => void)[] = [];

  set(fragment: string, fragmentType: string, operation: 'cut' | 'copy'): void {
    this.fragment = fragment;
    this.fragmentType = fragmentType;
    this.operation = operation;
    this.insertMode = true;
    this._notify();
  }

  clear(): void {
    this.fragment = null;
    this.fragmentType = null;
    this.operation = null;
    this.insertMode = false;
    this._notify();
  }

  setInsertMode(active: boolean): void {
    this.insertMode = active;
    this._notify();
  }

  subscribe(cb: () => void): () => void {
    this._listeners.push(cb);
    return () => {
      const idx = this._listeners.indexOf(cb);
      if (idx >= 0) this._listeners.splice(idx, 1);
    };
  }

  private _notify(): void {
    for (const cb of this._listeners) cb();
  }
}

let _instance: BuilderClipboard | undefined;
export function getClipboard(): BuilderClipboard {
  if (!_instance) _instance = new BuilderClipboard();
  return _instance;
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `yarn workspace @casehubio/pages-builder vitest run src/clipboard/builder-clipboard.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```
wip: add BuilderClipboard singleton with subscribe/notify  Refs #433
```

### Task 2: YAML fragment serialization and deserialization

**Files:**
- Create: `packages/pages-builder/src/clipboard/yaml-fragment.ts`
- Test: `packages/pages-builder/src/clipboard/yaml-fragment.test.ts`

**Interfaces:**
- Consumes: YAML library (`stringify`, `parseDocument`)
- Produces: `serializeNode(doc: PageDocument, path: readonly (string|number)[]): string`, `parseFragment(yaml: string): { type: string } | null`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { PageDocument } from '@casehubio/pages-document';
import { serializeNode, parseFragment } from './yaml-fragment.js';

const ROWS_PAGE = `pages:
- name: Dashboard
  rows:
  - columns:
    - span: 6
      components:
      - type: bar-chart
        properties:
          subtype: column
    - span: 6
      components:
      - type: metric
        properties:
          title: Users
`;

describe('serializeNode', () => {
  it('serializes a component node to YAML', () => {
    const doc = PageDocument.parse(ROWS_PAGE);
    const yaml = serializeNode(doc, ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0]);
    expect(yaml).toContain('type: bar-chart');
    expect(yaml).toContain('subtype: column');
    expect(yaml).not.toContain('metric');
  });

  it('serializes a column node', () => {
    const doc = PageDocument.parse(ROWS_PAGE);
    const yaml = serializeNode(doc, ['pages', 0, 'rows', 0, 'columns', 0]);
    expect(yaml).toContain('span: 6');
    expect(yaml).toContain('bar-chart');
  });

  it('serializes a row node', () => {
    const doc = PageDocument.parse(ROWS_PAGE);
    const yaml = serializeNode(doc, ['pages', 0, 'rows', 0]);
    expect(yaml).toContain('columns:');
    expect(yaml).toContain('bar-chart');
    expect(yaml).toContain('metric');
  });
});

describe('parseFragment', () => {
  it('detects component fragment', () => {
    const result = parseFragment('type: bar-chart\nproperties:\n  subtype: column');
    expect(result).toEqual({ type: 'component' });
  });

  it('detects column fragment', () => {
    const result = parseFragment('span: 6\ncomponents:\n- type: metric');
    expect(result).toEqual({ type: 'column' });
  });

  it('detects row fragment', () => {
    const result = parseFragment('columns:\n- span: 12\n  components: []');
    expect(result).toEqual({ type: 'row' });
  });

  it('returns null for non-YAML', () => {
    expect(parseFragment('not yaml {')).toBeNull();
  });

  it('returns null for empty string', () => {
    expect(parseFragment('')).toBeNull();
  });

  it('returns null for unrecognized YAML structure', () => {
    expect(parseFragment('foo: bar')).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests — verify failure**

Run: `yarn workspace @casehubio/pages-builder vitest run src/clipboard/yaml-fragment.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement serialize and parse**

```typescript
import { PageDocument } from '@casehubio/pages-document';
import { parseDocument, stringify } from 'yaml';

export function serializeNode(doc: PageDocument, path: readonly (string | number)[]): string {
  const fresh = parseDocument(doc.toString(), { keepSourceTokens: true });
  const node = fresh.getIn(path as (string | number)[], true);
  if (!node) return '';
  return stringify(node);
}

export function parseFragment(yaml: string): { type: string } | null {
  if (!yaml || !yaml.trim()) return null;
  try {
    const doc = parseDocument(yaml);
    if (doc.errors.length > 0) return null;
    const contents = doc.toJSON();
    if (typeof contents !== 'object' || contents === null) return null;
    if ('type' in contents) return { type: 'component' };
    if ('span' in contents) return { type: 'column' };
    if ('columns' in contents) return { type: 'row' };
    if ('rows' in contents || 'components' in contents || 'name' in contents) return { type: 'page' };
    return null;
  } catch {
    return null;
  }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `yarn workspace @casehubio/pages-builder vitest run src/clipboard/yaml-fragment.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```
wip: add YAML fragment serialization and type detection  Refs #433
```

---

## Batch 2: Tree Buttons and Clipboard Integration

### Task 3: Add Insert, Cut, Copy buttons to tree items

**Files:**
- Modify: `packages/pages-builder/src/tree/builder-tree.ts` — add buttons, events, remove DnD
- Modify: `packages/pages-builder/src/tree/builder-tree.test.ts` — new tests
- Delete: `packages/pages-builder/src/tree/tree-dnd.ts` (use `ide_refactor_safe_delete`)

**Interfaces:**
- Consumes: `BuilderClipboard` from Task 1
- Produces: `tree-cut`, `tree-copy`, `tree-insert` events with `{ path, nodeType, target: HTMLElement }`

- [ ] **Step 1: Write failing tests for new buttons and events**

```typescript
it('renders cut and copy buttons on every non-section node', async () => {
  const doc = PageDocument.parse(ROWS_PAGE);
  el = document.createElement('pages-builder-tree') as PagesBuilderTree;
  el.document = doc;
  document.body.appendChild(el);
  await el.updateComplete;

  const cutBtns = el.shadowRoot!.querySelectorAll('.cut-btn');
  const copyBtns = el.shadowRoot!.querySelectorAll('.copy-btn');
  const nonSectionItems = el.shadowRoot!.querySelectorAll('[data-node-type]:not([data-node-type="section"])');
  expect(cutBtns.length).toBe(nonSectionItems.length);
  expect(copyBtns.length).toBe(nonSectionItems.length);
});

it('renders insert button on every non-section node', async () => {
  const doc = PageDocument.parse(ROWS_PAGE);
  el = document.createElement('pages-builder-tree') as PagesBuilderTree;
  el.document = doc;
  document.body.appendChild(el);
  await el.updateComplete;

  const insertBtns = el.shadowRoot!.querySelectorAll('.insert-btn');
  const nonSectionItems = el.shadowRoot!.querySelectorAll('[data-node-type]:not([data-node-type="section"])');
  expect(insertBtns.length).toBe(nonSectionItems.length);
});

it('fires tree-cut on cut button click', async () => {
  const doc = PageDocument.parse(ROWS_PAGE);
  el = document.createElement('pages-builder-tree') as PagesBuilderTree;
  el.document = doc;
  document.body.appendChild(el);
  await el.updateComplete;

  const events: CustomEvent[] = [];
  el.addEventListener('tree-cut', ((e: CustomEvent) => events.push(e)) as EventListener);

  const cutBtn = el.shadowRoot!.querySelector<HTMLButtonElement>('.cut-btn')!;
  cutBtn.click();

  expect(events).toHaveLength(1);
  expect(events[0]!.detail.path).toBeDefined();
  expect(events[0]!.detail.nodeType).toBeDefined();
});

it('fires tree-copy on copy button click', async () => {
  const doc = PageDocument.parse(ROWS_PAGE);
  el = document.createElement('pages-builder-tree') as PagesBuilderTree;
  el.document = doc;
  document.body.appendChild(el);
  await el.updateComplete;

  const events: CustomEvent[] = [];
  el.addEventListener('tree-copy', ((e: CustomEvent) => events.push(e)) as EventListener);

  const copyBtn = el.shadowRoot!.querySelector<HTMLButtonElement>('.copy-btn')!;
  copyBtn.click();

  expect(events).toHaveLength(1);
});
```

- [ ] **Step 2: Run tests — verify failure**

Run: `yarn workspace @casehubio/pages-builder vitest run src/tree/builder-tree.test.ts`
Expected: FAIL — no `.cut-btn` elements

- [ ] **Step 3: Add buttons to `_renderNode` and new event handlers**

In `builder-tree.ts`, update `_renderNode` to add Insert, Cut, Copy buttons alongside the existing Add button. Add `_handleInsertClick`, `_handleCutClick`, `_handleCopyClick` methods that dispatch `tree-insert`, `tree-cut`, `tree-copy` events. Remove DnD event handlers (`_handleDragStart`, `_handleDragOver`, `_handleDragLeave`, `_handleDrop`, `_handleDragEnd`) and `draggable` attribute. Remove the `tree-dnd.ts` import. Add button CSS (same opacity pattern as existing add-btn).

Template addition for each non-section node (after the existing add-btn):
```html
<button class="insert-btn" aria-label="Insert near ${node.label}"
  @click="${(e: Event) => this._handleInsertClick(node, e)}">↓</button>
<button class="cut-btn" aria-label="Cut ${node.label}"
  @click="${(e: Event) => this._handleCutClick(node, e)}">✂</button>
<button class="copy-btn" aria-label="Copy ${node.label}"
  @click="${(e: Event) => this._handleCopyClick(node, e)}">⎘</button>
```

- [ ] **Step 4: Run tests — verify pass**

Run: `yarn workspace @casehubio/pages-builder vitest run src/tree/builder-tree.test.ts`
Expected: PASS

- [ ] **Step 5: Remove `tree-dnd.ts`**

Use `ide_refactor_safe_delete` on `packages/pages-builder/src/tree/tree-dnd.ts`. Remove the corresponding test file `tree-dnd.test.ts`.

- [ ] **Step 6: Run full builder test suite**

Run: `yarn workspace @casehubio/pages-builder vitest run`
Expected: All tests PASS (some existing DnD tests will need removal)

- [ ] **Step 7: Commit**

```
wip: add insert/cut/copy buttons to tree, remove DnD  Refs #433
```

### Task 4: Wire clipboard to builder-shell — cut, copy, paste execution

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — handle tree-cut, tree-copy, clipboard-aware add/insert
- Modify: `packages/pages-builder/src/shell/builder-shell.test.ts` — new tests

**Interfaces:**
- Consumes: `BuilderClipboard` (Task 1), `serializeNode` / `parseFragment` (Task 2), `tree-cut` / `tree-copy` events (Task 3)
- Produces: clipboard-aware `_handleTreeAdd` and `_handleTreeInsert` methods

- [ ] **Step 1: Write failing tests**

```typescript
it('tree-cut serializes node and removes it from document', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const compsBefore = el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length;

  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-cut', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  const compsAfter = el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length;
  expect(compsAfter).toBe(compsBefore - 1);
});

it('tree-copy serializes node without removing it', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const compsBefore = el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length;

  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-copy', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  const compsAfter = el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length;
  expect(compsAfter).toBe(compsBefore);
});

it('add pastes clipboard content as child when clipboard has fragment', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  // Copy a component first
  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-copy', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  // Add (paste) into second column
  const compsBefore = el.document.getPages()[0]!.getRows()[0]!.getColumns()[1]!.getComponents().length;
  tree.dispatchEvent(new CustomEvent('tree-add', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 1], nodeType: 'column' },
  }));
  await el.updateComplete;

  const compsAfter = el.document.getPages()[0]!.getRows()[0]!.getColumns()[1]!.getComponents().length;
  expect(compsAfter).toBe(compsBefore + 1);
});
```

- [ ] **Step 2: Run tests — verify failure**

Run: `yarn workspace @casehubio/pages-builder vitest run src/shell/builder-shell.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement handlers**

In `builder-shell.ts`:
- Import `getClipboard`, `serializeNode`, `parseFragment`
- Add `_handleTreeCut(e)`: serialize node, set clipboard, delete node from document
- Add `_handleTreeCopy(e)`: serialize node, set clipboard (no delete)
- Modify `_handleTreeAdd`: check `getClipboard()` — if fragment present and insertMode, paste as child; otherwise open picker
- Add `_handleTreeInsert(e)`: if clipboard active, paste before/after; otherwise show position picker then component picker
- Wire `@tree-cut` and `@tree-copy` events in `_syncTree`
- Add Esc handler to exit insert mode: `getClipboard().setInsertMode(false)`

- [ ] **Step 4: Run tests — verify pass**

Run: `yarn workspace @casehubio/pages-builder vitest run src/shell/builder-shell.test.ts`
Expected: PASS

- [ ] **Step 5: Run full builder tests**

Run: `yarn workspace @casehubio/pages-builder vitest run`
Expected: All PASS

- [ ] **Step 6: Commit**

```
wip: wire cut/copy/paste through builder-shell  Refs #433
```

---

## Batch 3: Insert Mode UX

### Task 5: Clipboard indicator and insert-mode target highlighting

**Files:**
- Modify: `packages/pages-builder/src/tree/builder-tree.ts` — subscribe to clipboard, highlight valid targets
- Modify: `packages/pages-builder/src/tree/builder-tree.test.ts` — new tests
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — render clipboard indicator, Esc key

**Interfaces:**
- Consumes: `BuilderClipboard` (Task 1), validation rules from `component-catalog.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// builder-tree.test.ts
it('highlights valid paste targets when insert mode active', async () => {
  const doc = PageDocument.parse(ROWS_PAGE);
  el = document.createElement('pages-builder-tree') as PagesBuilderTree;
  el.document = doc;
  el.clipboardFragmentType = 'component';
  el.insertMode = true;
  document.body.appendChild(el);
  await el.updateComplete;

  const validTargets = el.shadowRoot!.querySelectorAll('.paste-target');
  expect(validTargets.length).toBeGreaterThan(0);
  // Columns and pages should be valid targets for components
  const columnItems = el.shadowRoot!.querySelectorAll('[data-node-type="column"].paste-target');
  expect(columnItems.length).toBeGreaterThan(0);
});

it('does not highlight invalid paste targets', async () => {
  const doc = PageDocument.parse(ROWS_PAGE);
  el = document.createElement('pages-builder-tree') as PagesBuilderTree;
  el.document = doc;
  el.clipboardFragmentType = 'component';
  el.insertMode = true;
  document.body.appendChild(el);
  await el.updateComplete;

  // Sections should never be valid paste targets
  const sectionTargets = el.shadowRoot!.querySelectorAll('[data-node-type="section"].paste-target');
  expect(sectionTargets.length).toBe(0);
});

// builder-shell.test.ts
it('Esc exits insert mode without clearing clipboard', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  // Put something on clipboard
  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-copy', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  // Press Esc
  el.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
  await el.updateComplete;

  const { getClipboard } = await import('../clipboard/builder-clipboard.js');
  expect(getClipboard().insertMode).toBe(false);
  expect(getClipboard().fragment).not.toBeNull();
});
```

- [ ] **Step 2: Run tests — verify failure**

Expected: FAIL — no `clipboardFragmentType` property, no `.paste-target` class

- [ ] **Step 3: Implement insert-mode highlighting in tree**

Add reactive properties `clipboardFragmentType` and `insertMode` to `PagesBuilderTree`. In `_renderNode`, add `paste-target` class to nodes that are valid targets for the fragment type. Add `paste-target` CSS with accent-colored Add/Insert buttons.

In `builder-shell.ts`:
- Subscribe to `getClipboard()` and pass `clipboardFragmentType` / `insertMode` to the tree
- Render clipboard indicator banner in `_syncTree` when clipboard has content
- Add global Esc keydown listener that calls `getClipboard().setInsertMode(false)`

- [ ] **Step 4: Run tests — verify pass**

Run: `yarn workspace @casehubio/pages-builder vitest run`
Expected: All PASS

- [ ] **Step 5: Verify visually in browser**

Start dev server, copy a component, verify targets highlight. Press Esc, verify targets un-highlight but clipboard indicator remains.

- [ ] **Step 6: Commit**

```
wip: add insert-mode target highlighting and clipboard indicator  Refs #433
```

### Task 6: Position strategy popup for Insert

**Files:**
- Create: `packages/pages-builder/src/palette/position-picker.ts`
- Test: `packages/pages-builder/src/palette/position-picker.test.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — wire position picker

**Interfaces:**
- Produces: `<pages-position-picker>` component with `position-select` event (`{ position: 'before' | 'after' }`)

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './position-picker.js';
import type { PagesPositionPicker } from './position-picker.js';

describe('PagesPositionPicker', () => {
  let el: PagesPositionPicker;
  afterEach(() => { el?.remove(); });

  it('renders before and after buttons when open', async () => {
    el = document.createElement('pages-position-picker') as PagesPositionPicker;
    el.open = true;
    document.body.appendChild(el);
    await el.updateComplete;
    const btns = el.shadowRoot!.querySelectorAll('button');
    expect(btns.length).toBe(2);
    expect(btns[0]!.textContent?.trim()).toBe('Before');
    expect(btns[1]!.textContent?.trim()).toBe('After');
  });

  it('fires position-select with before', async () => {
    el = document.createElement('pages-position-picker') as PagesPositionPicker;
    el.open = true;
    document.body.appendChild(el);
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('position-select', ((e: CustomEvent) => events.push(e)) as EventListener);
    el.shadowRoot!.querySelector<HTMLButtonElement>('button')!.click();
    expect(events[0]!.detail.position).toBe('before');
  });

  it('closes on Escape', async () => {
    el = document.createElement('pages-position-picker') as PagesPositionPicker;
    el.open = true;
    document.body.appendChild(el);
    await el.updateComplete;

    const events: Event[] = [];
    el.addEventListener('picker-close', (e) => events.push(e));
    el.shadowRoot!.querySelector('.position-popover')!
      .dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
    expect(el.open).toBe(false);
    expect(events).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests — verify failure**

- [ ] **Step 3: Implement position-picker component**

Small Lit component: two buttons ("Before" / "After"), positioned absolutely near its anchor (same pattern as inline-picker). Fires `position-select` event.

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Wire into builder-shell**

When Insert is clicked (without clipboard): show position picker → on selection show component picker → insert at chosen position. When Insert is clicked with clipboard: show position picker → paste at chosen position.

- [ ] **Step 6: Run full test suite, verify visually**

- [ ] **Step 7: Commit**

```
wip: add position picker for insert before/after  Refs #433
```

---

## Batch 4: Keyboard Shortcuts and Cleanup

### Task 7: Keyboard shortcuts for cut/copy/paste and Esc

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — keyboard bindings
- Modify: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Consumes: all previous tasks

- [ ] **Step 1: Write failing tests**

```typescript
it('Ctrl+X on selected node fires cut', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  // Select a component
  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('node-select', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  const compsBefore = el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length;
  el.dispatchEvent(new KeyboardEvent('keydown', { key: 'x', ctrlKey: true, bubbles: true }));
  await el.updateComplete;

  expect(el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length).toBe(compsBefore - 1);
});

it('Ctrl+C on selected node fires copy without removing', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('node-select', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'rows', 0, 'columns', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  const compsBefore = el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length;
  el.dispatchEvent(new KeyboardEvent('keydown', { key: 'c', ctrlKey: true, bubbles: true }));
  await el.updateComplete;

  expect(el.document.getPages()[0]!.getRows()[0]!.getColumns()[0]!.getComponents().length).toBe(compsBefore);
});
```

- [ ] **Step 2: Run tests — verify failure**

- [ ] **Step 3: Add keyboard shortcut handlers**

Register `Ctrl+X`, `Ctrl+C` in the shell's keyboard shortcut bindings. Each calls the same logic as the tree-cut/tree-copy handlers using `this._selectedPath` and `this._selectedNodeType`.

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Run full test suite**

Run: `yarn workspace @casehubio/pages-builder vitest run`
Expected: All PASS

- [ ] **Step 6: Visual verification in browser**

Start dev server. Select a component in tree. Ctrl+X — node disappears, targets highlight. Esc — targets un-highlight. Ctrl+C on another node. Click Insert on a column — position picker appears. Select "After" — node pasted.

- [ ] **Step 7: Commit**

```
feat: structural editing — cut/copy/paste with guided targets  Refs #433
```

---

## Batch 5: Visual Mode — Overlay Clipboard Operations

### Task 8: Component overlay positioning engine

**Files:**
- Create: `packages/pages-builder/src/overlay/component-overlay.ts`
- Test: `packages/pages-builder/src/overlay/component-overlay.test.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — wire overlay after preview render

**Interfaces:**
- Consumes: `BuilderClipboard` (Task 1), preview container DOM
- Produces: `ComponentOverlayManager` class — creates/positions/removes overlays tracking `[data-component-type]` elements

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import { ComponentOverlayManager } from './component-overlay.js';

describe('ComponentOverlayManager', () => {
  let container: HTMLDivElement;
  let overlayRoot: HTMLDivElement;

  afterEach(() => {
    container?.remove();
    overlayRoot?.remove();
  });

  it('creates overlay divs for each data-component-type element', () => {
    container = document.createElement('div');
    container.innerHTML = `
      <div data-component-type="metric" style="width:100px;height:50px;"></div>
      <div data-component-type="bar-chart" style="width:200px;height:100px;"></div>
    `;
    document.body.appendChild(container);
    overlayRoot = document.createElement('div');
    document.body.appendChild(overlayRoot);

    const mgr = new ComponentOverlayManager(container, overlayRoot);
    mgr.update();

    const overlays = overlayRoot.querySelectorAll('.component-overlay');
    expect(overlays.length).toBe(2);
  });

  it('each overlay has add, insert, cut, copy buttons', () => {
    container = document.createElement('div');
    container.innerHTML = `<div data-component-type="metric" style="width:100px;height:50px;"></div>`;
    document.body.appendChild(container);
    overlayRoot = document.createElement('div');
    document.body.appendChild(overlayRoot);

    const mgr = new ComponentOverlayManager(container, overlayRoot);
    mgr.update();

    const overlay = overlayRoot.querySelector('.component-overlay')!;
    expect(overlay.querySelector('.overlay-add-btn')).toBeTruthy();
    expect(overlay.querySelector('.overlay-insert-btn')).toBeTruthy();
    expect(overlay.querySelector('.overlay-cut-btn')).toBeTruthy();
    expect(overlay.querySelector('.overlay-copy-btn')).toBeTruthy();
  });

  it('buttons are hidden by default, visible on hover', () => {
    container = document.createElement('div');
    container.innerHTML = `<div data-component-type="metric" style="width:100px;height:50px;"></div>`;
    document.body.appendChild(container);
    overlayRoot = document.createElement('div');
    document.body.appendChild(overlayRoot);

    const mgr = new ComponentOverlayManager(container, overlayRoot);
    mgr.update();

    const toolbar = overlayRoot.querySelector('.overlay-toolbar') as HTMLElement;
    expect(toolbar).toBeTruthy();
  });

  it('highlights valid targets when insert mode active', () => {
    container = document.createElement('div');
    container.innerHTML = `
      <div data-component-type="metric"></div>
      <div data-component-type="bar-chart"></div>
    `;
    document.body.appendChild(container);
    overlayRoot = document.createElement('div');
    document.body.appendChild(overlayRoot);

    const mgr = new ComponentOverlayManager(container, overlayRoot);
    mgr.update();
    mgr.setInsertMode('component');

    const targets = overlayRoot.querySelectorAll('.paste-target');
    expect(targets.length).toBe(2);
  });

  it('dispose removes all overlays', () => {
    container = document.createElement('div');
    container.innerHTML = `<div data-component-type="metric"></div>`;
    document.body.appendChild(container);
    overlayRoot = document.createElement('div');
    document.body.appendChild(overlayRoot);

    const mgr = new ComponentOverlayManager(container, overlayRoot);
    mgr.update();
    mgr.dispose();

    expect(overlayRoot.querySelectorAll('.component-overlay').length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests — verify failure**

- [ ] **Step 3: Implement ComponentOverlayManager**

The manager:
1. Queries `container` for all `[data-component-type]` elements
2. For each, creates an absolutely-positioned overlay `div` in `overlayRoot`
3. Positions each overlay using `getBoundingClientRect()` relative to `overlayRoot`
4. Each overlay contains a toolbar (top-right) with Add/Insert/Cut/Copy buttons
5. Toolbar uses `opacity: 0` → `opacity: 1` on overlay hover
6. `setInsertMode(fragmentType)` adds `paste-target` class to valid overlays
7. `update()` recalculates positions (call on scroll, resize, re-render)
8. `dispose()` removes all overlays

Overlay button clicks fire custom events (`overlay-add`, `overlay-insert`, `overlay-cut`, `overlay-copy`) with the component's index/path info.

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

```
wip: add component overlay manager for visual mode  Refs #433
```

### Task 9: Wire overlays into builder-shell preview

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — create overlay root, wire manager lifecycle
- Modify: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Consumes: `ComponentOverlayManager` (Task 8), `BuilderClipboard` (Task 1)
- Produces: visual mode cut/copy/paste working end-to-end

- [ ] **Step 1: Write failing tests**

```typescript
it('visual mode shows overlays on rendered components', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  el.renderPreview = (container, yaml) => {
    container.innerHTML = '<div data-component-type="title" style="width:100px;height:50px;">Hello</div>';
  };
  document.body.appendChild(el);
  await awaitReady(el);

  const overlayRoot = el.shadowRoot!.querySelector('.overlay-root');
  expect(overlayRoot).toBeTruthy();
  const overlays = overlayRoot!.querySelectorAll('.component-overlay');
  expect(overlays.length).toBeGreaterThan(0);
});
```

- [ ] **Step 2: Run tests — verify failure**

- [ ] **Step 3: Implement overlay wiring**

In `builder-shell.ts`:
- Add an overlay root `div` as a sibling of the preview container (positioned relative to the preview, `pointer-events: none` except on overlay buttons)
- After `_refreshPreview()` completes, call `overlayManager.update()`
- Subscribe to `BuilderClipboard` — when insert mode changes, call `overlayManager.setInsertMode(fragmentType)`
- Handle overlay button events (`overlay-add`, etc.) — same logic as tree event handlers
- On preview scroll/resize, call `overlayManager.update()` to reposition

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Run full test suite**

- [ ] **Step 6: Visual verification in browser**

Start dev server. Switch to Split/Visual mode. Hover over a rendered component — toolbar appears top-right. Click Copy. Valid target overlays highlight. Click Insert on a target — position picker → paste.

- [ ] **Step 7: Commit**

```
wip: wire visual overlay clipboard operations  Refs #433
```

---

## Batch 6: YAML Text View — Guided Line Indicators

### Task 10: CodeMirror paste-target line decorations

**Files:**
- Create: `packages/pages-builder/src/shell/yaml-paste-indicators.ts`
- Test: `packages/pages-builder/src/shell/yaml-paste-indicators.test.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — add extension to editor

**Interfaces:**
- Consumes: `BuilderClipboard` (Task 1), `PageDocument`, `findPathAtOffset` / `classifyPath` from `yaml-path.ts`
- Produces: CodeMirror extension that shows gutter markers at valid paste lines

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { computeValidPasteLines } from './yaml-paste-indicators.js';
import { PageDocument } from '@casehubio/pages-document';

const ROWS_PAGE = `pages:
- name: Dashboard
  rows:
  - columns:
    - span: 6
      components:
      - type: metric
    - span: 6
      components:
      - type: bar-chart
`;

describe('computeValidPasteLines', () => {
  it('returns line numbers where a component can be pasted', () => {
    const doc = PageDocument.parse(ROWS_PAGE);
    const lines = computeValidPasteLines(doc, 'component');
    // Components can be siblings of existing components (before/after)
    // and children of columns/pages
    expect(lines.length).toBeGreaterThan(0);
    // Should include lines near existing components
    const yaml = doc.toString();
    const metricLine = yaml.split('\n').findIndex(l => l.includes('type: metric')) + 1;
    expect(lines.some(l => Math.abs(l.line - metricLine) <= 1)).toBe(true);
  });

  it('returns empty for invalid fragment type at all positions', () => {
    const doc = PageDocument.parse(ROWS_PAGE);
    // 'dataset' fragments can't be inserted among components
    const lines = computeValidPasteLines(doc, 'dataset');
    expect(lines).toHaveLength(0);
  });

  it('each result includes position strategy', () => {
    const doc = PageDocument.parse(ROWS_PAGE);
    const lines = computeValidPasteLines(doc, 'component');
    for (const entry of lines) {
      expect(['before', 'after', 'child']).toContain(entry.position);
      expect(entry.line).toBeGreaterThan(0);
    }
  });
});
```

- [ ] **Step 2: Run tests — verify failure**

- [ ] **Step 3: Implement paste-line computation and CodeMirror extension**

`computeValidPasteLines(doc, fragmentType)`:
1. Walk the document tree (pages → rows → columns → components)
2. For each node, check if the fragment type is valid at that position
3. For valid positions, compute the YAML line number where a before/after/child indicator should appear
4. Return `{ line: number, position: 'before' | 'after' | 'child', path: (string|number)[] }[]`

CodeMirror extension:
- State field holding valid paste lines (updated when clipboard changes)
- Gutter markers at valid lines — small colored indicators (▸ for before, ▾ for after, + for child)
- Line decorations with subtle background on valid lines
- Click on a gutter marker → execute paste at that position
- Extension is inert when `insertMode === false`

```typescript
import { StateEffect, StateField, type Extension } from '@codemirror/state';
import { EditorView, gutter, GutterMarker } from '@codemirror/view';

export const setPasteTargets = StateEffect.define<PasteLineEntry[] | null>();

// ... gutter + decoration setup similar to yaml-gutter.ts
```

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Wire into builder-shell**

In `builder-shell.ts`:
- Add `yamlPasteIndicatorExtension` to the code editor's extensions
- When clipboard state changes (subscribe), compute valid paste lines and dispatch `setPasteTargets` effect
- When a gutter marker is clicked, execute the paste using the stored path and position

- [ ] **Step 6: Run full test suite**

- [ ] **Step 7: Visual verification in browser**

Start dev server. Copy a component from the tree. Switch focus to the YAML editor. Gutter shows colored markers at valid insert points. Click a marker — component pasted at that line.

- [ ] **Step 8: Commit**

```
feat: YAML text view guided paste indicators  Refs #433
```

## References

- `specs/issue-439-dock-workbench-polish/2026-09-17-structural-editing-design.md` — design spec
- `packages/pages-builder/src/tree/builder-tree.ts` — tree component
- `packages/pages-builder/src/tree/tree-dnd.ts` — existing DnD (to be removed)
- `packages/pages-document/src/page-document.ts:828` — `ComponentNode.moveToIndex()`
- `packages/pages-builder/src/clipboard/` — new clipboard module
- `packages/pages-builder/src/catalog/component-catalog.ts` — type validation
- `packages/pages-builder/src/palette/inline-picker.ts` — existing picker
- `packages/pages-builder/src/shell/yaml-gutter.ts` — existing gutter extension pattern
- `packages/pages-builder/src/shell/yaml-path.ts` — path/range computation
- GitHub #433 — Tree/visual structural editing
- GitHub #439 — Dock workbench polish

- `specs/issue-439-dock-workbench-polish/2026-09-17-structural-editing-design.md` — design spec
- `packages/pages-builder/src/tree/builder-tree.ts` — tree component
- `packages/pages-builder/src/tree/tree-dnd.ts` — existing DnD (to be removed)
- `packages/pages-document/src/page-document.ts:828` — `ComponentNode.moveToIndex()`
- `packages/pages-builder/src/clipboard/` — new clipboard module
- `packages/pages-builder/src/catalog/component-catalog.ts` — type validation
- `packages/pages-builder/src/palette/inline-picker.ts` — existing picker
- GitHub #433 — Tree/visual structural editing
- GitHub #439 — Dock workbench polish
