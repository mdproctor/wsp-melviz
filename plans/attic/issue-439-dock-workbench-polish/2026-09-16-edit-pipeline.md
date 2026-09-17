# Edit Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #449 — verify schema completions for all CaseHub YAML formats
**Issue group:** #449

**Goal:** Replace the fragmented sync architecture in pages-builder with a
normalized edit pipeline where all mutations flow through a single coordinator.

**Architecture:** Single `_applyEdit(origin, fn)` coordinator wraps every
document mutation. `_syncViews(origin)` fans out to all views, skipping the
origin. PageDocument gains a `_coordinated` flag to suppress internal
undo/notification when the shell coordinates. Editor sync uses diff-patch
instead of full replacement.

**Tech Stack:** TypeScript, Lit, CodeMirror 6, yaml (npm), vitest, pages-document, pages-builder

## Global Constraints

- Pre-release stage — backward compat not required
- PageDocument's public undo API remains for standalone consumers
- No bash file operations on .ts source files — use IntelliJ MCP
- TDD: every task starts with a failing test
- Build pages-code-editor after source changes (`yarn workspace @casehubio/pages-code-editor run build`)

---

## Batch 1: Foundation — Coordinated Mode

### Task 1: Add coordinated mode to PageDocument

**Files:**
- Modify: `packages/pages-document/src/page-document.ts:83-126` (undo/notify/transaction methods)
- Test: `packages/pages-document/src/page-document.test.ts` (new test file or append to existing)

**Interfaces:**
- Produces: `PageDocument.parseCoordinated(yaml: string): PageDocument` — factory that returns `_coordinated = true`
- Produces: `PageDocument._coordinated: boolean` — private flag, default false

- [ ] **Step 1: Write failing tests**

```typescript
describe('coordinated mode', () => {
  it('parseCoordinated returns instance with coordinated flag', () => {
    const doc = PageDocument.parseCoordinated('pages:\n- name: P\n');
    expect(doc.getPages()).toHaveLength(1);
    // Mutations should not fire onChange
    const calls: string[] = [];
    doc.onChange(() => calls.push('notified'));
    doc.getPages()[0]!.name = 'Changed';
    expect(calls).toHaveLength(0);
  });

  it('coordinated mode suppresses _pushUndo', () => {
    const doc = PageDocument.parseCoordinated('pages:\n- name: P\n');
    doc.getPages()[0]!.name = 'Changed';
    expect(doc.canUndo()).toBe(false);
  });

  it('standalone mode still fires notifications', () => {
    const doc = PageDocument.parse('pages:\n- name: P\n');
    const calls: string[] = [];
    doc.onChange(() => calls.push('notified'));
    doc.getPages()[0]!.name = 'Changed';
    expect(calls).toHaveLength(1);
  });

  it('coordinated transaction preserves atomicity', () => {
    const doc = PageDocument.parseCoordinated('pages:\n- name: P\n  components:\n  - type: title\n');
    const page = doc.getPages()[0]!;
    const comp = page.getComponents()[0]!;
    // wrapInRow uses beginTransaction/abortTransaction internally
    page.wrapInRow([0]);
    expect(page.getLayoutMode()).toBe('rows');
    expect(doc.canUndo()).toBe(false); // undo suppressed
  });

  it('coordinated abortTransaction does not pop undo stack', () => {
    const doc = PageDocument.parseCoordinated('pages:\n- name: P\n');
    doc.beginTransaction();
    doc.getPages()[0]!.name = 'temp';
    doc.abortTransaction();
    expect(doc.getPages()[0]!.name).toBe('P');
    expect(doc.canUndo()).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests, verify they fail**

Run: `yarn workspace @casehubio/pages-document run test -- -t 'coordinated'`
Expected: FAIL — `parseCoordinated` does not exist

- [ ] **Step 3: Implement coordinated mode**

Add to `page-document.ts`:

1. Add private field: `private _coordinated = false;`
2. Add factory: `static parseCoordinated(yaml: string): PageDocument`
3. Guard `_pushUndo()` with `if (this._coordinated) return;`
4. Guard `_notify()` with `if (this._coordinated) return;`
5. Guard `beginTransaction()` undo push with `if (!this._coordinated)`
6. Guard `abortTransaction()` undo pop with `if (!this._coordinated)`

- [ ] **Step 4: Run tests, verify they pass**

Run: `yarn workspace @casehubio/pages-document run test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(pages-document): add coordinated mode for shell-managed undo/sync

Refs #449
```

---

## Batch 2: Coordinator + Diff-Patch

### Task 2: Add _applyEdit coordinator and _syncViews

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Consumes: `PageDocument.parseCoordinated()` from Task 1
- Produces: `_applyEdit(origin: EditOrigin, fn: () => void): void`
- Produces: `_syncViews(origin: EditOrigin): void`
- Produces: `_parseDocument(yaml: string): PageDocument`

- [ ] **Step 1: Write failing test**

```typescript
it('_applyEdit wraps mutations and syncs views', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const before = el.document.getPages().length;
  // Use the toolbar + Page button which should go through _applyEdit
  const addPageBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === '+ Page') as HTMLElement;
  addPageBtn.click();
  await el.updateComplete;

  expect(el.document.getPages()).toHaveLength(before + 1);
  // Tree should show new page
  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as any;
  expect(tree.document.getPages()).toHaveLength(before + 1);
  // Editor should contain new page YAML
  const codeEditor = el.shadowRoot!.querySelector('pages-code-editor') as any;
  const editorText = codeEditor?.value ?? '';
  expect(editorText).toContain('New Page');
});
```

- [ ] **Step 2: Run test, verify current behavior**

Run: `yarn workspace @casehubio/pages-builder run test -- -t '_applyEdit wraps'`
This test may pass with existing code. If so, add a more specific test for coordinated mode (no internal onChange fired).

- [ ] **Step 3: Implement _applyEdit and _syncViews**

Add to `builder-shell.ts`:

```typescript
type EditOrigin = 'editor' | 'tree' | 'properties' | 'palette' | 'toolbar';

private _undoStack: string[] = [];
private _redoStack: string[] = [];

private _parseDocument(yaml: string): PageDocument {
  return PageDocument.parseCoordinated(yaml);
}

private _applyEdit(origin: EditOrigin, fn: () => void): void {
  if (origin !== 'editor' && this._pendingEditorSync) {
    this._flushEditorSync();
  }
  this._undoStack.push(this._document.toString());
  if (this._undoStack.length > 50) this._undoStack.shift();
  this._redoStack.length = 0;
  fn();
  this._syncViews(origin);
}

private _syncViews(origin: EditOrigin): void {
  if (origin !== 'editor') {
    this._diffPatchEditor();
  }
  this._syncTree();
  this._resolvePropertySource();
  this._syncPreview();
  this._emitChange();
  this.requestUpdate();
}
```

Wrap all 16 mutation sites in `_applyEdit`:
- `_addPage` (L452): `this._applyEdit('toolbar', () => this._document.addPage('New Page'))`
- `_addDataset` (L456): `this._applyEdit('toolbar', () => this._document.addDataset('new_dataset'))`
- `_updatePropertySource` onChange closures (L531, L561, L576, L595, L614, L615): wrap each in `this._applyEdit('properties', () => { ... })`
- `_handleComponentSelect` (L740-771): wrap in `this._applyEdit('palette', () => { ... })`

Switch all `PageDocument.parse()` calls to `this._parseDocument()`:
- `connectedCallback` (L130): `this._document = this._parseDocument(this.yaml)`
- `willUpdate` yaml change handler

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehubio/pages-builder run test`
Expected: ALL PASS (except tree-add inline picker tests — still RED)

- [ ] **Step 5: Commit**

```
feat(pages-builder): add _applyEdit coordinator and _syncViews fan-out

Refs #449
```

### Task 3: computeMinimalChanges and _diffPatchEditor

**Files:**
- Create: `packages/pages-builder/src/shell/diff-patch.ts`
- Create: `packages/pages-builder/src/shell/diff-patch.test.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`

**Interfaces:**
- Produces: `computeMinimalChanges(before: string, after: string): ChangeSpec[]`
- Produces: `_diffPatchEditor(): void` (replaces `_pushYamlToEditor`)

- [ ] **Step 1: Write failing tests for computeMinimalChanges**

```typescript
import { computeMinimalChanges } from './diff-patch.js';

describe('computeMinimalChanges', () => {
  it('returns empty array for identical strings', () => {
    expect(computeMinimalChanges('abc', 'abc')).toEqual([]);
  });

  it('replaces changed middle region', () => {
    const before = 'line1\nline2\nline3\n';
    const after = 'line1\nchanged\nline3\n';
    const changes = computeMinimalChanges(before, after);
    expect(changes).toHaveLength(1);
    expect(changes[0]).toMatchObject({ from: 6, to: 11, insert: 'changed' });
  });

  it('handles appended lines', () => {
    const before = 'line1\n';
    const after = 'line1\nline2\n';
    const changes = computeMinimalChanges(before, after);
    expect(changes).toHaveLength(1);
  });

  it('handles full replacement', () => {
    const changes = computeMinimalChanges('old', 'new');
    expect(changes).toHaveLength(1);
    expect(changes[0]).toMatchObject({ from: 0, to: 3, insert: 'new' });
  });
});
```

- [ ] **Step 2: Run tests, verify fail**

Run: `yarn workspace @casehubio/pages-builder run test -- -t 'computeMinimalChanges'`
Expected: FAIL — module not found

- [ ] **Step 3: Implement computeMinimalChanges**

```typescript
import type { ChangeSpec } from '@codemirror/state';

export function computeMinimalChanges(before: string, after: string): ChangeSpec[] {
  if (before === after) return [];
  const beforeLines = before.split('\n');
  const afterLines = after.split('\n');

  let prefixLen = 0;
  while (prefixLen < beforeLines.length && prefixLen < afterLines.length
    && beforeLines[prefixLen] === afterLines[prefixLen]) {
    prefixLen++;
  }

  let suffixLen = 0;
  while (suffixLen < beforeLines.length - prefixLen
    && suffixLen < afterLines.length - prefixLen
    && beforeLines[beforeLines.length - 1 - suffixLen] === afterLines[afterLines.length - 1 - suffixLen]) {
    suffixLen++;
  }

  let fromChar = 0;
  for (let i = 0; i < prefixLen; i++) fromChar += beforeLines[i]!.length + 1;

  let toChar = before.length;
  for (let i = 0; i < suffixLen; i++) toChar -= beforeLines[beforeLines.length - 1 - i]!.length + 1;

  const insertLines = afterLines.slice(prefixLen, afterLines.length - suffixLen);
  const insert = insertLines.join('\n');

  return [{ from: fromChar, to: Math.max(fromChar, toChar), insert }];
}
```

- [ ] **Step 4: Run tests, verify pass**

Run: `yarn workspace @casehubio/pages-builder run test -- -t 'computeMinimalChanges'`
Expected: ALL PASS

- [ ] **Step 5: Wire _diffPatchEditor in builder-shell**

Replace `_pushYamlToEditor` calls with `_diffPatchEditor`:

```typescript
private _diffPatchEditor(): void {
  const editorEl = this.shadowRoot?.querySelector('pages-code-editor') as any;
  if (!editorEl) return;
  const editorText = editorEl.value ?? '';
  const modelText = this._document.toString();
  if (editorText === modelText) return;
  const view = editorEl._editorView;
  if (!view) return;
  const changes = computeMinimalChanges(editorText, modelText);
  if (changes.length > 0) {
    view.dispatch({ changes });
  }
}
```

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @casehubio/pages-builder run test`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```
feat(pages-builder): cursor-preserving diff-patch for editor sync

Refs #449
```

---

## Batch 3: Remove Old Sync + Inline Editor Handler

### Task 4: Remove _subscribeToDocument and replace YamlSync

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` (remove _subscribeToDocument, _pushYamlToEditor, _connectYamlSync, _docUnsub; add _handleEditorInput, _flushEditorSync)
- Delete: `packages/pages-builder/src/shell/yaml-sync.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/pages-builder/src/shell/yaml-sync.test.ts`
- Test: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Removes: `YamlSync` class, `_subscribeToDocument()`, `_pushYamlToEditor()`, `_connectYamlSync()`, `_docUnsub`
- Produces: `_handleEditorInput(): void`, `_flushEditorSync(): void`

- [ ] **Step 1: Write failing test for editor→model sync**

```typescript
it('editor text change flows through _applyEdit after debounce', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  // Simulate editor input — change page name in YAML
  const newYaml = MINIMAL_PAGE.replace('Overview', 'Renamed');
  const codeEditor = el.shadowRoot!.querySelector('pages-code-editor') as any;
  codeEditor.value = newYaml;
  codeEditor.dispatchEvent(new Event('input', { bubbles: true, composed: true }));

  // Before debounce: model unchanged
  expect(el.document.getPages()[0]!.name).toBe('Overview');

  // After debounce (300ms)
  await new Promise(r => setTimeout(r, 400));
  await el.updateComplete;

  expect(el.document.getPages()[0]!.name).toBe('Renamed');
});

it('invalid YAML does not update model', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const codeEditor = el.shadowRoot!.querySelector('pages-code-editor') as any;
  codeEditor.value = 'invalid: [yaml: {broken';
  codeEditor.dispatchEvent(new Event('input', { bubbles: true, composed: true }));

  await new Promise(r => setTimeout(r, 400));
  await el.updateComplete;

  // Model stays at last valid state
  expect(el.document.getPages()[0]!.name).toBe('Overview');
});
```

- [ ] **Step 2: Run tests, verify they fail**

Run: `yarn workspace @casehubio/pages-builder run test -- -t 'editor text change'`
Expected: May pass or fail depending on current YamlSync behavior

- [ ] **Step 3: Implement inline editor handler**

Add to builder-shell.ts:

```typescript
private _pendingEditorSync: number | undefined;
private _editorDirty = false;

private _handleEditorInput(): void {
  this._editorDirty = true;
  clearTimeout(this._pendingEditorSync);
  this._pendingEditorSync = window.setTimeout(() => {
    this._pendingEditorSync = undefined;
    const editorEl = this.shadowRoot?.querySelector('pages-code-editor') as any;
    const text = editorEl?.value ?? '';
    const newDoc = this._parseDocument(text);
    if (newDoc.diagnostics?.some((d: any) => d.severity === 'error')) return;
    this._applyEdit('editor', () => { this._document = newDoc; });
    this._editorDirty = false;
  }, 300);
}

private _flushEditorSync(): void {
  clearTimeout(this._pendingEditorSync);
  this._pendingEditorSync = undefined;
  if (!this._editorDirty) return;
  const editorEl = this.shadowRoot?.querySelector('pages-code-editor') as any;
  const text = editorEl?.value ?? '';
  const newDoc = this._parseDocument(text);
  if (newDoc.diagnostics?.some((d: any) => d.severity === 'error')) return;
  this._undoStack.push(this._document.toString());
  if (this._undoStack.length > 50) this._undoStack.shift();
  this._redoStack.length = 0;
  this._document = newDoc;
  this._editorDirty = false;
}
```

Wire the editor input handler in `_syncCentre`:
```html
<pages-code-editor
  .extensions="${[...builderHighlightExtension, ...this._schemaExtensions]}"
  language="yaml"
  label="Page YAML source"
  @input="${() => this._handleEditorInput()}"
></pages-code-editor>
```

- [ ] **Step 4: Remove old sync infrastructure**

Remove from builder-shell.ts:
- `_subscribeToDocument()` method and all call sites
- `_pushYamlToEditor()` method
- `_connectYamlSync()` method
- `_docUnsub` field
- `YamlSync` import
- `YamlSync` usage in connectedCallback/willUpdate

Delete files:
- `packages/pages-builder/src/shell/yaml-sync.ts` (use ide_refactor_safe_delete)
- `packages/pages-builder/src/shell/yaml-sync.test.ts`

- [ ] **Step 5: Run all tests**

Run: `yarn workspace @casehubio/pages-builder run test`
Expected: ALL PASS (yaml-sync tests gone, shell tests pass with new handler)

- [ ] **Step 6: Commit**

```
refactor(pages-builder): replace YamlSync with inline editor handler

Remove _subscribeToDocument, _pushYamlToEditor, _connectYamlSync, and
the YamlSync class. Editor→model sync uses debounced _handleEditorInput
with diagnostics check. Model→editor sync uses _diffPatchEditor.

Refs #449
```

---

## Batch 4: Tree Actions + Shell-Managed Undo

### Task 5: Wire tree-action and tree-drop events

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` (_syncTree template, new handlers)
- Test: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Consumes: `_applyEdit` from Task 2
- Produces: `_handleTreeAction(e: CustomEvent)`, `_handleTreeDrop(e: CustomEvent)`

- [ ] **Step 1: Write failing tests**

```typescript
it('tree-action delete removes component', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = `pages:\n- name: P\n  components:\n  - type: title\n  - type: metric\n`;
  document.body.appendChild(el);
  await awaitReady(el);

  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-action', {
    bubbles: true, composed: true,
    detail: { action: 'delete', path: ['pages', 0, 'components', 1], nodeType: 'component' },
  }));
  await el.updateComplete;

  expect(el.document.getPages()[0]!.getComponents()).toHaveLength(1);
  expect(el.document.getPages()[0]!.getComponents()[0]!.type).toBe('title');
});

it('tree-action duplicate copies component', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = `pages:\n- name: P\n  components:\n  - type: title\n`;
  document.body.appendChild(el);
  await awaitReady(el);

  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-action', {
    bubbles: true, composed: true,
    detail: { action: 'duplicate', path: ['pages', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  expect(el.document.getPages()[0]!.getComponents()).toHaveLength(2);
});

it('tree-action add-row adds row to page', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = ROWS_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const rowsBefore = el.document.getPages()[0]!.getRows().length;
  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('tree-action', {
    bubbles: true, composed: true,
    detail: { action: 'add-row', path: ['pages', 0], nodeType: 'page' },
  }));
  await el.updateComplete;

  expect(el.document.getPages()[0]!.getRows()).toHaveLength(rowsBefore + 1);
});
```

- [ ] **Step 2: Run tests, verify fail**

Run: `yarn workspace @casehubio/pages-builder run test -- -t 'tree-action'`
Expected: FAIL — handler not wired

- [ ] **Step 3: Implement _handleTreeAction**

Add handler and wire in `_syncTree` template:

```typescript
private _handleTreeAction(e: CustomEvent<{ action: string; path: readonly (string | number)[]; nodeType: TreeNodeType }>): void {
  const { action, path, nodeType } = e.detail;
  this._applyEdit('tree', () => {
    switch (action) {
      case 'delete': this._deleteAtPath(path, nodeType); break;
      case 'duplicate': this._duplicateAtPath(path, nodeType); break;
      case 'add-row': this._findPageAtPath(path)?.addRow(); break;
      case 'add-column': this._findRowAtPath(path)?.addColumn(); break;
      case 'move-up': this._moveAtPath(path, nodeType, -1); break;
      case 'move-down': this._moveAtPath(path, nodeType, 1); break;
    }
  });
}
```

Wire in `_syncTree`:
```html
@tree-action="${(e: CustomEvent) => this._handleTreeAction(e)}"
@tree-drop="${(e: CustomEvent) => this._handleTreeDrop(e)}"
```

- [ ] **Step 4: Run tests, verify pass**

Run: `yarn workspace @casehubio/pages-builder run test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(pages-builder): wire tree-action and tree-drop through edit pipeline

Refs #449
```

### Task 6: Shell-managed undo/redo

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Replaces: `_document.undo()` / `_document.redo()` with shell stack ops

- [ ] **Step 1: Write failing tests**

```typescript
it('undo reverts toolbar add-page', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const addPageBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === '+ Page') as HTMLElement;
  addPageBtn.click();
  expect(el.document.getPages()).toHaveLength(2);

  const undoBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === 'Undo') as HTMLElement;
  undoBtn.click();
  await el.updateComplete;

  expect(el.document.getPages()).toHaveLength(1);
});

it('undo reverts property change', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await el.updateComplete;

  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('node-select', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  const source = (el.shadowRoot!.querySelector('pages-property-palette') as any).source;
  source.onChange(['text'], 'Changed');
  await el.updateComplete;
  expect(el.document.toString()).toContain('Changed');

  const undoBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === 'Undo') as HTMLElement;
  undoBtn.click();
  await el.updateComplete;

  expect(el.document.toString()).toContain('Hello');
  expect(el.document.toString()).not.toContain('Changed');
});

it('redo after undo restores state', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await awaitReady(el);

  const addPageBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === '+ Page') as HTMLElement;
  addPageBtn.click();

  const undoBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === 'Undo') as HTMLElement;
  undoBtn.click();
  expect(el.document.getPages()).toHaveLength(1);

  const redoBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === 'Redo') as HTMLElement;
  redoBtn.click();
  await el.updateComplete;

  expect(el.document.getPages()).toHaveLength(2);
});
```

- [ ] **Step 2: Run tests, verify behavior**

Run: `yarn workspace @casehubio/pages-builder run test -- -t 'undo reverts'`

- [ ] **Step 3: Replace shell undo/redo**

Replace `_undo` and `_redo` methods:

```typescript
private _undo(): void {
  if (this._undoStack.length === 0) return;
  this._redoStack.push(this._document.toString());
  const prev = this._undoStack.pop()!;
  this._document = this._parseDocument(prev);
  this._syncViews('toolbar');
}

private _redo(): void {
  if (this._redoStack.length === 0) return;
  this._undoStack.push(this._document.toString());
  const next = this._redoStack.pop()!;
  this._document = this._parseDocument(next);
  this._syncViews('toolbar');
}
```

Update toolbar button bindings:
```html
?disabled="${this._undoStack.length === 0}"  <!-- was: !this._document.canUndo() -->
?disabled="${this._redoStack.length === 0}"  <!-- was: !this._document.canRedo() -->
```

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehubio/pages-builder run test`
Expected: ALL PASS

- [ ] **Step 5: Rebuild and verify in Playwright**

```bash
yarn workspace @casehubio/pages-code-editor run build
```

Open http://localhost:5173/ — verify:
- Type in editor, wait, undo reverts
- Click +Page, undo removes it
- Change property, undo reverts

- [ ] **Step 6: Commit**

```
feat(pages-builder): shell-managed undo/redo across all edit origins

Refs #449
```

---

## Batch 5: Property Source Resolution (Lazy)

### Task 7: Lazy property source with resolve closures

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-builder/src/shell/builder-shell.test.ts`

**Interfaces:**
- Produces: `_resolvePropertySource(): void` — called from `_syncViews`
- Modifies: `_updatePropertySource()` — calls `_resolvePropertySource()` plus selection side-effects

- [ ] **Step 1: Write failing test**

```typescript
it('property source stays fresh after document swap via undo', async () => {
  el = document.createElement('pages-builder-shell') as PagesBuilderShell;
  el.yaml = MINIMAL_PAGE;
  document.body.appendChild(el);
  await el.updateComplete;

  // Select component
  const tree = el.shadowRoot!.querySelector('pages-builder-tree') as HTMLElement;
  tree.dispatchEvent(new CustomEvent('node-select', {
    bubbles: true, composed: true,
    detail: { path: ['pages', 0, 'components', 0], nodeType: 'component' },
  }));
  await el.updateComplete;

  // Change property
  const source1 = (el.shadowRoot!.querySelector('pages-property-palette') as any).source;
  source1.onChange(['text'], 'Modified');
  await el.updateComplete;
  expect(source1.data['text']).toBe('Modified');

  // Undo
  const undoBtn = Array.from(el.shadowRoot!.querySelectorAll('.toolbar-btn'))
    .find(b => b.textContent?.trim() === 'Undo') as HTMLElement;
  undoBtn.click();
  await el.updateComplete;

  // Property source should show original value, not stale
  const source2 = (el.shadowRoot!.querySelector('pages-property-palette') as any).source;
  expect(source2.data['text']).toBe('Hello');
});
```

- [ ] **Step 2: Implement _resolvePropertySource with lazy closures**

Extract node resolution from `_updatePropertySource` into `_resolvePropertySource`.
Use lazy `resolve()` closures in `onChange` and `get data()`.

- [ ] **Step 3: Run all tests**

Run: `yarn workspace @casehubio/pages-builder run test`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```
feat(pages-builder): lazy property source resolution via resolve closures

Refs #449
```

---

## References

- [2026-09-16-edit-pipeline-design.md] — design spec this plan implements
- [packages/pages-document/src/page-document.ts:83-126] — undo/notify/transaction internals
- [packages/pages-builder/src/shell/builder-shell.ts:158-181] — current _subscribeToDocument + _pushYamlToEditor
- [packages/pages-builder/src/shell/builder-shell.ts:452-771] — all 16 mutation sites
- [packages/pages-builder/src/shell/builder-shell.ts:882-907] — current YamlSync wiring
- [packages/pages-builder/src/shell/yaml-sync.ts] — current bidirectional sync (to be removed)
- [packages/pages-builder/src/tree/tree-context-menu.ts] — 11 tree action types
- [GitHub #449] — focal issue
