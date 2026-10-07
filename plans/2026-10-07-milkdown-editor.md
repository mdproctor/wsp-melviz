# Milkdown Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — epic must be created before implementation
**Issue group:** TBD — child issues for each package

**Goal:** Add a rich markdown editor (Milkdown) to the Pages platform with MCP-driven LLM collaboration, overlays, edit sessions, and dual-mode editing — unified under a shared `EditableText` interface.

**Architecture:** Three new packages (`pages-editor-core`, `pages-markdown-editor`, `pages-document-diff`) plus refactoring of `pages-code-editor` and `pages-aria`. All editors share a common `EditableText` interface via bridge implementations. MCP tools target this interface for editor-agnostic LLM interaction.

**Tech Stack:** Milkdown (`@milkdown/kit`), ProseMirror, CodeMirror 6, Lit 3.x, TypeScript, Vitest

## Global Constraints

- TypeScript strict mode enabled for all new packages
- Lit ^3.3.3 for all web components
- All packages use `type: "module"` and ESM exports
- Design tokens via CSS custom properties (`--pages-neutral-*`, `--pages-accent-*`)
- Shadow DOM for all LIT components
- Package naming: `@casehubio/pages-*`
- No Vue or React dependencies in new packages
- KaTeX and Mermaid must be lazy-loaded (never eagerly imported)
- Use `ide_refactor_rename` for renames, `ide_move_file` for moves — never bash

---

## Batch 1: Foundation — pages-editor-core + CodeEditorBridge refactor

After this batch: shared `EditableText` interface exists, `CodeEditorBridge` extends the new base class, all existing tests pass, `pages-aria` imports from the new canonical location.

### Task 1: Scaffold pages-editor-core with EditableText interface

**Files:**
- Create: `packages/pages-editor-core/package.json`
- Create: `packages/pages-editor-core/tsconfig.json`
- Create: `packages/pages-editor-core/tsconfig.build.json`
- Create: `packages/pages-editor-core/src/index.ts`
- Create: `packages/pages-editor-core/src/editable-text.ts`
- Create: `packages/pages-editor-core/src/types.ts`
- Create: `packages/pages-editor-core/src/discovery.ts`
- Test: `packages/pages-editor-core/src/editable-text.test.ts`

**Interfaces:**
- Consumes: nothing (foundational)
- Produces: `EditableText`, `Position`, `HighlightStyle`, `AnnotationOptions`, `EditSession`, `EditSessionActiveError`, `EDITABLE_TEXT`, `isEditableText()`, `findEditableText()`

- [ ] **Step 1: Scaffold package**

Create `package.json` following `pages-code-editor`'s structure:

```json
{
  "name": "@casehubio/pages-editor-core",
  "version": "0.1.0",
  "description": "Shared editor interface, bridge base, and MCP tool adapter for CaseHub Pages editors",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {},
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "license": "Apache-2.0"
}
```

Create `tsconfig.json` and `tsconfig.build.json` matching `pages-code-editor`'s pattern (extend `pages-tsconfig`).

- [ ] **Step 2: Write the failing test for EditableText interface and discovery**

```typescript
// packages/pages-editor-core/src/editable-text.test.ts
import { describe, it, expect } from 'vitest';
import type { EditableText, Position, EditSession } from './types.js';
import { EDITABLE_TEXT, isEditableText, findEditableText } from './discovery.js';

describe('EditableText discovery', () => {
  it('EDITABLE_TEXT symbol uses editable-text key', () => {
    expect(EDITABLE_TEXT).toBe(Symbol.for('editable-text'));
  });

  it('isEditableText returns false for plain elements', () => {
    const el = document.createElement('div');
    expect(isEditableText(el)).toBe(false);
  });

  it('isEditableText returns true when symbol is present', () => {
    const el = document.createElement('div');
    const mock: EditableText = {
      getText: () => '',
      getLine: () => '',
      getLineCount: () => 0,
      setContent: () => {},
      findText: () => [],
      insertText: () => {},
      replaceRange: () => {},
      deleteRange: () => {},
      setCursor: () => {},
      getCursor: () => ({ line: 0, col: 0 }),
      highlight: () => '',
      removeHighlight: () => {},
      clearHighlights: () => {},
      addAnnotation: () => '',
      removeAnnotation: () => {},
      clearAnnotations: () => {},
      beginEditSession: () => ({ owner: '', mode: 'exclusive', startedAt: '', cancel: () => {} }),
      endEditSession: () => {},
    };
    (el as any)[EDITABLE_TEXT] = mock;
    expect(isEditableText(el)).toBe(true);
  });

  it('findEditableText walks up DOM tree', () => {
    const parent = document.createElement('div');
    const child = document.createElement('span');
    parent.appendChild(child);
    const mock = { getText: () => 'hello' } as unknown as EditableText;
    (parent as any)[EDITABLE_TEXT] = mock;
    expect(findEditableText(child)).toBe(mock);
  });

  it('findEditableText returns null when not found', () => {
    const el = document.createElement('div');
    expect(findEditableText(el)).toBeNull();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-editor-core test`
Expected: FAIL — modules not found

- [ ] **Step 4: Implement types.ts, discovery.ts, and index.ts**

`types.ts` — all interface definitions from the spec (§1 EditableText Interface):

```typescript
export interface Position { line: number; col: number; }

export type HighlightStyle = 'pulse' | 'underline' | 'glow' | 'box';

export interface AnnotationOptions {
  type: 'arrow' | 'callout' | 'marker' | 'numbered';
  text?: string;
  style?: Record<string, string>;
}

export interface EditSession {
  readonly owner: string;
  readonly mode: 'exclusive';
  readonly startedAt: string;
  cancel(): void;
}

export interface EditableText {
  getText(): string;
  getLine(line: number): string;
  getLineCount(): number;
  setContent(text: string): void;
  findText(query: string): Position[];
  insertText(text: string): void;
  replaceRange(from: Position, to: Position, text: string): void;
  deleteRange(from: Position, to: Position): void;
  setCursor(line: number, col: number): void;
  getCursor(): Position;
  highlight(from: Position, to: Position, style?: HighlightStyle): string;
  removeHighlight(id: string): void;
  clearHighlights(): void;
  addAnnotation(anchor: Position, options: AnnotationOptions): string;
  removeAnnotation(id: string): void;
  clearAnnotations(): void;
  beginEditSession(owner: string): EditSession;
  endEditSession(session: EditSession): void;
  triggerCompletion?(): void;
  selectCompletion?(label: string): boolean;
}

export class EditSessionActiveError extends Error {
  constructor(
    public readonly owner: string,
    public readonly startedAt: string,
  ) {
    super(`Edit session already active (owner: '${owner}', since: ${startedAt})`);
    this.name = 'EditSessionActiveError';
  }
}
```

`discovery.ts` — symbol and discovery functions (copied from spec §1):

```typescript
import type { EditableText } from './types.js';

export const EDITABLE_TEXT: unique symbol = Symbol.for('editable-text') as any;

export function isEditableText(
  el: Element,
): el is Element & { [EDITABLE_TEXT]: EditableText } {
  return EDITABLE_TEXT in el;
}

export function findEditableText(el: Element): EditableText | null {
  let current: Element | null = el;
  while (current) {
    if (isEditableText(current)) return current[EDITABLE_TEXT];
    const root = current.getRootNode();
    if (root instanceof ShadowRoot) {
      current = root.host;
    } else {
      current = current.parentElement;
    }
  }
  return null;
}
```

`index.ts` — re-export everything:

```typescript
export type { Position, HighlightStyle, AnnotationOptions, EditSession, EditableText } from './types.js';
export { EditSessionActiveError } from './types.js';
export { EDITABLE_TEXT, isEditableText, findEditableText } from './discovery.js';
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-editor-core test`
Expected: PASS

- [ ] **Step 6: Add workspace entry and install**

Add `"packages/pages-editor-core"` to root `package.json` workspaces array.

Run: `yarn install`

- [ ] **Step 7: Commit**

```bash
git add packages/pages-editor-core/ package.json yarn.lock
git commit -m "feat: scaffold pages-editor-core with EditableText interface and discovery"
```

---

### Task 2: EditableTextBridge abstract base class

**Files:**
- Create: `packages/pages-editor-core/src/editable-text-bridge.ts`
- Test: `packages/pages-editor-core/src/editable-text-bridge.test.ts`
- Modify: `packages/pages-editor-core/src/index.ts`

**Interfaces:**
- Consumes: `EditableText`, `Position`, `EditSession`, `EditSessionActiveError` from Task 1
- Produces: `EditableTextBridge` abstract class — base for `CodeEditorBridge` and `MarkdownEditorBridge`

- [ ] **Step 1: Write failing test for bridge base class**

Test shared logic: highlight ID tracking, annotation ID tracking, edit session management.

```typescript
// packages/pages-editor-core/src/editable-text-bridge.test.ts
import { describe, it, expect } from 'vitest';
import type { Position } from './types.js';
import { EditableTextBridge } from './editable-text-bridge.js';
import { EditSessionActiveError } from './types.js';

class TestBridge extends EditableTextBridge {
  appliedHighlights: Array<{ id: string; from: Position; to: Position }> = [];
  removedHighlights: string[] = [];
  getText() { return 'line one\nline two\nline three'; }
  getLine(line: number) { return this.getText().split('\n')[line] ?? ''; }
  getLineCount() { return 3; }
  setContent() {}
  findText(query: string) {
    const lines = this.getText().split('\n');
    const results: Position[] = [];
    for (let i = 0; i < lines.length; i++) {
      const col = lines[i].indexOf(query);
      if (col >= 0) results.push({ line: i, col });
    }
    return results;
  }
  insertText() {}
  replaceRange() {}
  deleteRange() {}
  setCursor() {}
  getCursor() { return { line: 0, col: 0 }; }
  protected applyHighlight(id: string, from: Position, to: Position) {
    this.appliedHighlights.push({ id, from, to });
  }
  protected removeHighlightDecoration(id: string) {
    this.removedHighlights.push(id);
  }
  protected clearHighlightDecorations() {
    this.removedHighlights.push('__all__');
  }
  triggerCompletion() {}
  selectCompletion() { return false; }
}

describe('EditableTextBridge', () => {
  it('highlight returns unique IDs', () => {
    const bridge = new TestBridge();
    const id1 = bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
    const id2 = bridge.highlight({ line: 1, col: 0 }, { line: 1, col: 4 });
    expect(id1).not.toBe(id2);
    expect(bridge.appliedHighlights).toHaveLength(2);
  });

  it('removeHighlight removes by ID', () => {
    const bridge = new TestBridge();
    const id = bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
    bridge.removeHighlight(id);
    expect(bridge.removedHighlights).toContain(id);
  });

  it('clearHighlights clears all', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
    bridge.highlight({ line: 1, col: 0 }, { line: 1, col: 4 });
    bridge.clearHighlights();
    expect(bridge.removedHighlights).toContain('__all__');
  });

  it('beginEditSession creates exclusive session', () => {
    const bridge = new TestBridge();
    const session = bridge.beginEditSession('claude');
    expect(session.owner).toBe('claude');
    expect(session.mode).toBe('exclusive');
    expect(session.startedAt).toBeTruthy();
  });

  it('beginEditSession throws when session already active', () => {
    const bridge = new TestBridge();
    bridge.beginEditSession('claude');
    expect(() => bridge.beginEditSession('gpt')).toThrow(EditSessionActiveError);
  });

  it('endEditSession clears active session', () => {
    const bridge = new TestBridge();
    const session = bridge.beginEditSession('claude');
    bridge.endEditSession(session);
    const session2 = bridge.beginEditSession('claude');
    expect(session2).toBeTruthy();
  });

  it('addAnnotation returns unique IDs', () => {
    const bridge = new TestBridge();
    const id1 = bridge.addAnnotation({ line: 0, col: 0 }, { type: 'callout', text: 'hello' });
    const id2 = bridge.addAnnotation({ line: 1, col: 0 }, { type: 'arrow' });
    expect(id1).not.toBe(id2);
  });

  it('removeAnnotation removes by ID', () => {
    const bridge = new TestBridge();
    const id = bridge.addAnnotation({ line: 0, col: 0 }, { type: 'marker' });
    bridge.removeAnnotation(id);
    expect(bridge.getAnnotation(id)).toBeUndefined();
  });

  it('clearAnnotations removes all', () => {
    const bridge = new TestBridge();
    bridge.addAnnotation({ line: 0, col: 0 }, { type: 'callout' });
    bridge.addAnnotation({ line: 1, col: 0 }, { type: 'arrow' });
    bridge.clearAnnotations();
    expect(bridge.annotationCount).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-editor-core test`
Expected: FAIL — `EditableTextBridge` not found

- [ ] **Step 3: Implement EditableTextBridge**

```typescript
// packages/pages-editor-core/src/editable-text-bridge.ts
import type { EditableText, Position, HighlightStyle, AnnotationOptions, EditSession } from './types.js';
import { EditSessionActiveError } from './types.js';

interface StoredAnnotation {
  anchor: Position;
  options: AnnotationOptions;
}

export abstract class EditableTextBridge implements EditableText {
  private _highlights = new Map<string, { from: Position; to: Position; style?: HighlightStyle }>();
  private _annotations = new Map<string, StoredAnnotation>();
  private _activeSession: EditSession | null = null;
  private _nextId = 0;

  private _genId(prefix: string): string {
    return `${prefix}-${++this._nextId}`;
  }

  // Subclasses implement these for engine-specific decoration
  protected abstract applyHighlight(id: string, from: Position, to: Position, style?: HighlightStyle): void;
  protected abstract removeHighlightDecoration(id: string): void;
  protected abstract clearHighlightDecorations(): void;

  // EditableText — content (abstract, engine-specific)
  abstract getText(): string;
  abstract getLine(line: number): string;
  abstract getLineCount(): number;
  abstract setContent(text: string): void;
  abstract findText(query: string): Position[];
  abstract insertText(text: string): void;
  abstract replaceRange(from: Position, to: Position, text: string): void;
  abstract deleteRange(from: Position, to: Position): void;
  abstract setCursor(line: number, col: number): void;
  abstract getCursor(): Position;

  // EditableText — highlights (shared ID management, delegates rendering)
  highlight(from: Position, to: Position, style?: HighlightStyle): string {
    const id = this._genId('hl');
    this._highlights.set(id, { from, to, style });
    this.applyHighlight(id, from, to, style);
    return id;
  }

  removeHighlight(id: string): void {
    if (this._highlights.delete(id)) {
      this.removeHighlightDecoration(id);
    }
  }

  clearHighlights(): void {
    this._highlights.clear();
    this.clearHighlightDecorations();
  }

  // EditableText — annotations (fully managed in base)
  addAnnotation(anchor: Position, options: AnnotationOptions): string {
    const id = this._genId('ann');
    this._annotations.set(id, { anchor, options });
    return id;
  }

  removeAnnotation(id: string): void {
    this._annotations.delete(id);
  }

  clearAnnotations(): void {
    this._annotations.clear();
  }

  getAnnotation(id: string): StoredAnnotation | undefined {
    return this._annotations.get(id);
  }

  get annotationCount(): number {
    return this._annotations.size;
  }

  get activeHighlights(): ReadonlyMap<string, { from: Position; to: Position; style?: HighlightStyle }> {
    return this._highlights;
  }

  get activeAnnotations(): ReadonlyMap<string, StoredAnnotation> {
    return this._annotations;
  }

  // EditableText — edit sessions
  beginEditSession(owner: string): EditSession {
    if (this._activeSession) {
      throw new EditSessionActiveError(this._activeSession.owner, this._activeSession.startedAt);
    }
    const session: EditSession = {
      owner,
      mode: 'exclusive',
      startedAt: new Date().toISOString(),
      cancel: () => { this._activeSession = null; },
    };
    this._activeSession = session;
    return session;
  }

  endEditSession(session: EditSession): void {
    if (this._activeSession === session) {
      this._activeSession = null;
    }
  }

  get isSessionActive(): boolean {
    return this._activeSession !== null;
  }

  // Optional
  triggerCompletion?(): void;
  selectCompletion?(label: string): boolean;
}
```

- [ ] **Step 4: Export from index.ts**

Add to `packages/pages-editor-core/src/index.ts`:

```typescript
export { EditableTextBridge } from './editable-text-bridge.js';
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-editor-core test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-editor-core/
git commit -m "feat: add EditableTextBridge abstract base class with highlight, annotation, and session management"
```

---

### Task 3: Refactor CodeEditorBridge to extend EditableTextBridge

**Files:**
- Modify: `packages/pages-code-editor/package.json` (add `pages-editor-core` dep)
- Modify: `packages/pages-code-editor/src/code-editor-bridge.ts` (extend base, remove duplicated logic)
- Modify: `packages/pages-code-editor/src/pages-code-editor.ts` (update symbol import)
- Modify: `packages/pages-aria/package.json` (add `pages-editor-core` dep)
- Modify: `packages/pages-aria/src/executor/editable-text.ts` (re-export from core)
- Modify: `packages/pages-aria/src/executor/index.ts` (update exports)
- Test: existing tests in `pages-code-editor` and `pages-aria`

**Interfaces:**
- Consumes: `EditableTextBridge`, `EDITABLE_TEXT` from Task 1-2
- Produces: refactored `CodeEditorBridge extends EditableTextBridge`

- [ ] **Step 1: Add `pages-editor-core` dependency to both packages**

Add `"@casehubio/pages-editor-core": "workspace:*"` to `dependencies` in:
- `packages/pages-code-editor/package.json`
- `packages/pages-aria/package.json`

Run: `yarn install`

- [ ] **Step 2: Run existing tests to confirm green baseline**

Run: `yarn workspace @casehubio/pages-code-editor test`
Run: `yarn workspace @casehubio/pages-aria test`
Expected: PASS (baseline)

- [ ] **Step 3: Refactor code-editor-bridge.ts**

Remove the local `Position` and `ScenarioEditableText` interfaces (lines 4-22). Remove the `highlightField` StateField and effects (lines 24-43) — these stay but are now engine-specific decoration logic called from the bridge's `applyHighlight`/`removeHighlightDecoration`/`clearHighlightDecorations` overrides.

Refactor `CodeEditorBridge` to extend `EditableTextBridge`:

```typescript
import { EditableTextBridge } from '@casehubio/pages-editor-core';
import type { Position, HighlightStyle } from '@casehubio/pages-editor-core';
import { EditorView } from '@codemirror/view';
import { StateEffect, StateField } from '@codemirror/state';
import { Decoration, type DecorationSet } from '@codemirror/view';

// CodeMirror-specific decoration machinery (stays here)
const addHighlight = StateEffect.define<{ from: number; to: number; class: string }>();
const removeHighlightById = StateEffect.define<string>();
const clearAllHighlights = StateEffect.define<null>();

// ... highlightField StateField implementation ...

export class CodeEditorBridge extends EditableTextBridge {
  constructor(private readonly view: EditorView) {
    super();
  }

  private toOffset(pos: Position): number { /* existing logic */ }
  private toPosition(offset: number): Position { /* existing logic */ }

  // Content methods (unchanged from existing)
  getText(): string { return this.view.state.doc.toString(); }
  getLine(line: number): string { return this.view.state.doc.line(line + 1).text; }
  getLineCount(): number { return this.view.state.doc.lines; }
  setContent(text: string) { /* existing logic */ }
  findText(query: string): Position[] {
    const text = this.getText();
    const results: Position[] = [];
    const lines = text.split('\n');
    for (let i = 0; i < lines.length; i++) {
      let idx = 0;
      while ((idx = lines[i].indexOf(query, idx)) !== -1) {
        results.push({ line: i, col: idx });
        idx += query.length;
      }
    }
    return results;
  }

  // Editing methods (unchanged from existing)
  insertText(text: string) { /* existing */ }
  replaceRange(from: Position, to: Position, text: string) { /* existing */ }
  deleteRange(from: Position, to: Position) { /* existing */ }
  setCursor(line: number, col: number) { /* existing */ }
  getCursor(): Position { /* existing */ }

  // Engine-specific decoration (called by base class)
  protected applyHighlight(id: string, from: Position, to: Position, style?: HighlightStyle) {
    const cls = `editor-highlight-${style ?? 'pulse'}`;
    this.view.dispatch({
      effects: addHighlight.of({
        from: this.toOffset(from),
        to: this.toOffset(to),
        class: cls,
      }),
    });
  }

  protected removeHighlightDecoration(id: string) {
    this.view.dispatch({ effects: removeHighlightById.of(id) });
  }

  protected clearHighlightDecorations() {
    this.view.dispatch({ effects: clearAllHighlights.of(null) });
  }
}
```

- [ ] **Step 4: Update pages-code-editor.ts to use new symbol**

Replace line 14:
```typescript
// Old:
const EDITABLE_TEXT = Symbol.for('scenario-editable-text');
// New:
import { EDITABLE_TEXT } from '@casehubio/pages-editor-core';
```

- [ ] **Step 5: Update pages-aria editable-text.ts to re-export from core**

Replace the entire file content with re-exports:

```typescript
// packages/pages-aria/src/executor/editable-text.ts
export type { Position, EditableText } from '@casehubio/pages-editor-core';
export { EDITABLE_TEXT, isEditableText, findEditableText } from '@casehubio/pages-editor-core';
```

This preserves the existing public API — all consumers importing from `pages-aria` continue to work. The type name changes from `ScenarioEditableText` to `EditableText`; use `ide_refactor_rename` for any consumers still using the old name.

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @casehubio/pages-code-editor test`
Run: `yarn workspace @casehubio/pages-aria test`
Run: `yarn workspace @casehubio/pages-editor-core test`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-editor-core/ packages/pages-code-editor/ packages/pages-aria/ yarn.lock
git commit -m "refactor: CodeEditorBridge extends EditableTextBridge, canonical EditableText in pages-editor-core"
```

---

## Batch 2: Milkdown Core — basic PagesMarkdownEditor

After this batch: Milkdown renders in a LIT component with a custom toolbar, WYSIWYG editing works, `MarkdownEditorBridge` implements `EditableText`.

### Task 4: Scaffold pages-markdown-editor and mount Milkdown in LIT

**Files:**
- Create: `packages/pages-markdown-editor/package.json`
- Create: `packages/pages-markdown-editor/tsconfig.json`
- Create: `packages/pages-markdown-editor/tsconfig.build.json`
- Create: `packages/pages-markdown-editor/src/index.ts`
- Create: `packages/pages-markdown-editor/src/pages-markdown-editor.ts`
- Test: `packages/pages-markdown-editor/src/pages-markdown-editor.test.ts`

**Interfaces:**
- Consumes: `EDITABLE_TEXT` from `pages-editor-core`
- Produces: `<pages-markdown-editor>` custom element with `value`, `mode`, `readonly` properties

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-markdown-editor",
  "version": "0.1.0",
  "description": "Milkdown-based rich markdown editor — Lit Web Component",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/pages-editor-core": "workspace:*",
    "@milkdown/kit": "^7.6.0",
    "lit": "^3.3.3"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "jsdom": "^26.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Write failing test — component registration and basic rendering**

```typescript
// packages/pages-markdown-editor/src/pages-markdown-editor.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import './pages-markdown-editor.js';

describe('PagesMarkdownEditor', () => {
  it('registers as custom element', () => {
    expect(customElements.get('pages-markdown-editor')).toBeDefined();
  });

  it('has default property values', () => {
    const el = document.createElement('pages-markdown-editor') as any;
    expect(el.value).toBe('');
    expect(el.mode).toBe('wysiwyg');
    expect(el.readonly).toBe(false);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-markdown-editor test`
Expected: FAIL

- [ ] **Step 4: Implement PagesMarkdownEditor shell**

```typescript
// packages/pages-markdown-editor/src/pages-markdown-editor.ts
import { LitElement, css, html } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { Editor, rootCtx, defaultValueCtx } from '@milkdown/kit/core';
import { commonmark } from '@milkdown/kit/preset/commonmark';
import { gfm } from '@milkdown/kit/preset/gfm';
import { history } from '@milkdown/kit/plugin/history';
import { listener, listenerCtx } from '@milkdown/kit/plugin/listener';
import { EDITABLE_TEXT } from '@casehubio/pages-editor-core';

@customElement('pages-markdown-editor')
export class PagesMarkdownEditor extends LitElement {
  @property({ type: String }) value = '';
  @property({ type: String }) mode: 'wysiwyg' | 'source' | 'split' = 'wysiwyg';
  @property({ type: Boolean, reflect: true }) readonly = false;
  @property({ type: String }) label: string | undefined;

  @state() private _editor: Editor | null = null;
  private _suppressUpdate = false;

  static override styles = css`
    :host { display: flex; flex-direction: column; min-height: 200px; }
    .toolbar { display: flex; gap: 2px; padding: 4px 8px; border-bottom: 1px solid var(--pages-neutral-5, #d4d4d4); background: var(--pages-neutral-2, #f5f5f5); flex-shrink: 0; }
    .editor-host { flex: 1; overflow: auto; }
    .editor-host .milkdown { padding: 16px 24px; outline: none; min-height: 100%; }
  `;

  override render() {
    return html`
      <div class="toolbar" role="toolbar" aria-label="Formatting">
        <!-- toolbar buttons added in Task 6 -->
      </div>
      <div class="editor-host"></div>
      <div class="overlay-layer"></div>
    `;
  }

  override async firstUpdated() {
    const host = this.renderRoot.querySelector('.editor-host') as HTMLElement;
    this._editor = await Editor.make()
      .config((ctx) => {
        ctx.set(rootCtx, host);
        ctx.set(defaultValueCtx, this.value);
        ctx.get(listenerCtx).markdownUpdated((_, md) => {
          if (!this._suppressUpdate) {
            this.value = md;
            this.dispatchEvent(new CustomEvent('input', { bubbles: true, composed: true, detail: { value: md } }));
          }
        });
      })
      .use(commonmark)
      .use(gfm)
      .use(history)
      .use(listener)
      .create();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._editor?.destroy();
    this._editor = null;
  }
}
```

- [ ] **Step 5: Add workspace entry, install, run tests**

Add `"packages/pages-markdown-editor"` to root `package.json` workspaces.

Run: `yarn install`
Run: `yarn workspace @casehubio/pages-markdown-editor test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-markdown-editor/ package.json yarn.lock
git commit -m "feat: scaffold pages-markdown-editor with Milkdown mounted in LIT shadow DOM"
```

---

### Task 5: MarkdownEditorBridge with position index

**Files:**
- Create: `packages/pages-markdown-editor/src/markdown-editor-bridge.ts`
- Test: `packages/pages-markdown-editor/src/markdown-editor-bridge.test.ts`
- Modify: `packages/pages-markdown-editor/src/pages-markdown-editor.ts` (attach bridge)

**Interfaces:**
- Consumes: `EditableTextBridge` from `pages-editor-core`
- Produces: `MarkdownEditorBridge extends EditableTextBridge` — implements all `EditableText` methods for ProseMirror

- [ ] **Step 1: Write failing test for bridge content methods**

```typescript
// packages/pages-markdown-editor/src/markdown-editor-bridge.test.ts
import { describe, it, expect } from 'vitest';
// Note: full integration tests require a mounted Milkdown editor
// These unit tests use a mock ProseMirror EditorView

describe('MarkdownEditorBridge', () => {
  it('getText returns markdown serialization of ProseMirror doc');
  it('getLine returns correct line from markdown');
  it('getLineCount returns number of lines');
  it('setContent replaces entire document');
  it('findText returns positions of matches');
  it('insertText inserts at cursor');
  it('replaceRange replaces text between positions');
  it('deleteRange removes text between positions');
  it('setCursor moves selection');
  it('getCursor returns current selection head');
  it('toOffset converts line/col to ProseMirror offset');
  it('toPosition converts ProseMirror offset to line/col');
  it('highlight creates ProseMirror Decoration.inline');
  it('clearHighlights removes all decorations');
});
```

- [ ] **Step 2: Implement MarkdownEditorBridge**

The bridge wraps a Milkdown `Editor` instance. It serializes the ProseMirror doc to markdown for `getText()`, builds a cached position index on construction and after structural edits, and translates `Position` line/col to ProseMirror doc offsets.

Key methods:
- `getText()` — `ctx.get(serializerCtx)(view.state.doc)` to get markdown
- `getLine(n)` — index lookup → block node → extract or serialize
- `toOffset(pos)` — binary search cached index, walk into block
- `insertText(text)` — `view.dispatch(view.state.tr.insertText(text, sel.from))`
- `highlight(from, to, style)` — add `Decoration.inline` via a plugin's DecorationSet

- [ ] **Step 3: Attach bridge to component via EDITABLE_TEXT symbol**

In `pages-markdown-editor.ts`, after Milkdown creates, attach the bridge:

```typescript
import { MarkdownEditorBridge } from './markdown-editor-bridge.js';

// In firstUpdated(), after .create():
this._bridge = new MarkdownEditorBridge(this._editor);
(this as any)[EDITABLE_TEXT] = this._bridge;
```

- [ ] **Step 4: Run tests, commit**

Run: `yarn workspace @casehubio/pages-markdown-editor test`

```bash
git add packages/pages-markdown-editor/
git commit -m "feat: MarkdownEditorBridge with position index and EditableText implementation"
```

---

### Task 6: Custom Lit toolbar

**Files:**
- Create: `packages/pages-markdown-editor/src/toolbar/editor-toolbar.ts`
- Create: `packages/pages-markdown-editor/src/toolbar/toolbar-button.ts`
- Test: `packages/pages-markdown-editor/src/toolbar/editor-toolbar.test.ts`
- Modify: `packages/pages-markdown-editor/src/pages-markdown-editor.ts` (wire toolbar)

**Interfaces:**
- Consumes: Milkdown `Editor` instance, `commandsCtx` from `@milkdown/kit/core`
- Produces: `<editor-toolbar>` and `<toolbar-button>` internal components

- [ ] **Step 1: Write failing test for toolbar button dispatching commands**

```typescript
// packages/pages-markdown-editor/src/toolbar/editor-toolbar.test.ts
import { describe, it, expect } from 'vitest';

describe('EditorToolbar', () => {
  it('renders bold, italic, heading, list, code buttons');
  it('clicking bold dispatches toggleBoldCommand');
  it('buttons display keyboard shortcut in title attribute');
  it('buttons have aria-keyshortcuts attribute');
  it('math button triggers lazy KaTeX import');
  it('diagram button triggers lazy Mermaid import');
});
```

- [ ] **Step 2: Implement ToolbarButton**

```typescript
// packages/pages-markdown-editor/src/toolbar/toolbar-button.ts
import { LitElement, css, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('toolbar-button')
export class ToolbarButton extends LitElement {
  @property({ type: String }) icon = '';
  @property({ type: String }) tooltip = '';
  @property({ type: String }) shortcut = '';
  @property({ type: Boolean }) active = false;

  static override styles = css`
    :host { display: inline-flex; }
    button {
      width: 28px; height: 28px; border: none; border-radius: 4px;
      background: transparent; cursor: pointer; display: flex;
      align-items: center; justify-content: center; font-size: 14px;
      color: var(--pages-neutral-11, #6b7280);
    }
    button:hover { background: var(--pages-neutral-3, #e5e7eb); }
    button[aria-pressed="true"] { background: var(--pages-accent-3, #c7d2fe); color: var(--pages-accent-11, #4338ca); }
  `;

  override render() {
    return html`
      <button
        title="${this.tooltip}${this.shortcut ? ` (${this.shortcut})` : ''}"
        aria-pressed="${this.active}"
        aria-keyshortcuts="${this.shortcut}"
        @click=${() => this.dispatchEvent(new CustomEvent('command', { bubbles: true, composed: true }))}
      >${this.icon}</button>
    `;
  }
}
```

- [ ] **Step 3: Implement EditorToolbar**

The toolbar creates ~15 buttons, each dispatching the corresponding Milkdown command via `ctx.get(commandsCtx).call(commandKey)`. Heavy plugins (KaTeX, Mermaid) use `await import(...)` before dispatching.

- [ ] **Step 4: Wire toolbar into PagesMarkdownEditor**

Replace the empty toolbar div in `render()` with `<editor-toolbar .editor=${this._editor}></editor-toolbar>`.

- [ ] **Step 5: Run tests, commit**

```bash
git add packages/pages-markdown-editor/
git commit -m "feat: custom Lit toolbar dispatching Milkdown commands with keyboard shortcuts"
```

---

## Batch 3: Overlays + Edit Sessions

After this batch: both CodeMirror and Milkdown support highlight decorations, floating annotations, and edit sessions with cancel/rollback.

### Task 7: Highlight system for both bridges

**Files:**
- Modify: `packages/pages-code-editor/src/code-editor-bridge.ts` (add `removeHighlightById` effect handling)
- Modify: `packages/pages-markdown-editor/src/markdown-editor-bridge.ts` (ProseMirror decoration plugin)
- Test: `packages/pages-editor-core/src/compliance.test.ts` (shared compliance suite)

**Interfaces:**
- Consumes: `EditableTextBridge` highlight lifecycle
- Produces: working `highlight()`, `removeHighlight()`, `clearHighlights()` on both bridges

- [ ] **Step 1: Write shared compliance test suite**

```typescript
// packages/pages-editor-core/src/compliance.test.ts
import { describe, it, expect } from 'vitest';
import type { EditableText } from './types.js';

export function editableTextComplianceTests(name: string, factory: () => EditableText) {
  describe(`EditableText compliance: ${name}`, () => {
    it('highlight returns unique string IDs', () => {
      const et = factory();
      const id1 = et.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
      const id2 = et.highlight({ line: 0, col: 5 }, { line: 0, col: 9 });
      expect(typeof id1).toBe('string');
      expect(id1).not.toBe(id2);
    });

    it('removeHighlight does not throw for valid ID', () => {
      const et = factory();
      const id = et.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
      expect(() => et.removeHighlight(id)).not.toThrow();
    });

    it('clearHighlights does not throw', () => {
      const et = factory();
      et.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
      expect(() => et.clearHighlights()).not.toThrow();
    });

    it('addAnnotation returns unique string IDs', () => {
      const et = factory();
      const id1 = et.addAnnotation({ line: 0, col: 0 }, { type: 'callout' });
      const id2 = et.addAnnotation({ line: 1, col: 0 }, { type: 'arrow' });
      expect(typeof id1).toBe('string');
      expect(id1).not.toBe(id2);
    });

    it('beginEditSession returns session with owner', () => {
      const et = factory();
      const session = et.beginEditSession('test');
      expect(session.owner).toBe('test');
      expect(session.mode).toBe('exclusive');
      et.endEditSession(session);
    });
  });
}
```

Export this from index. Both bridge packages import and run it against their implementations.

- [ ] **Step 2: Implement engine-specific decoration for CodeMirror and ProseMirror**

Update `CodeEditorBridge` to track highlight IDs in the StateField so `removeHighlightById` works. Update `MarkdownEditorBridge` to create a ProseMirror plugin with `DecorationSet` that responds to add/remove/clear effects.

- [ ] **Step 3: Run compliance tests on both bridges, commit**

```bash
git commit -m "feat: highlight system with ID-based removal for both CodeMirror and ProseMirror bridges"
```

---

### Task 8: Floating annotations overlay layer

**Files:**
- Create: `packages/pages-markdown-editor/src/overlay/annotation-renderer.ts`
- Create: `packages/pages-markdown-editor/src/overlay/annotation-renderer.test.ts`
- Modify: `packages/pages-markdown-editor/src/pages-markdown-editor.ts` (wire overlay layer)

**Interfaces:**
- Consumes: `activeAnnotations` from `EditableTextBridge`, `coordsAtPos` from ProseMirror EditorView
- Produces: positioned DOM elements in the overlay layer (arrows, callouts, markers)

- [ ] **Step 1: Write failing test**

Test annotation rendering: creating an annotation should produce a positioned DOM element in the overlay div.

- [ ] **Step 2: Implement AnnotationRenderer**

The renderer takes the overlay div and the editor view. On `addAnnotation()`, it creates a positioned DOM element. A `requestAnimationFrame` loop repositions all annotations on scroll/resize using `view.coordsAtPos()`.

- [ ] **Step 3: Run tests, commit**

```bash
git commit -m "feat: floating annotation overlay renderer with scroll-tracking repositioning"
```

---

### Task 9: Edit sessions with lock, cancel, and rollback

**Files:**
- Modify: `packages/pages-editor-core/src/editable-text-bridge.ts` (snapshot/rollback in session)
- Modify: `packages/pages-markdown-editor/src/markdown-editor-bridge.ts` (ProseMirror state snapshot)
- Modify: `packages/pages-markdown-editor/src/pages-markdown-editor.ts` (UI indicator)
- Test: `packages/pages-markdown-editor/src/edit-session.test.ts`

**Interfaces:**
- Consumes: `EditSession` from core
- Produces: working lock/cancel/rollback including document state, decorations, and annotations

- [ ] **Step 1: Write failing test for session rollback**

```typescript
describe('Edit session rollback', () => {
  it('cancel restores document to pre-session state');
  it('cancel clears highlights created during session');
  it('cancel clears annotations created during session');
  it('normal end preserves edits in undo stack');
  it('beginEditSession throws when session already active');
});
```

- [ ] **Step 2: Implement snapshot/restore in EditableTextBridge**

The base class captures a snapshot (document text, highlight map, annotation map) on `beginEditSession()`. `cancel()` calls `setContent()` with the snapshot text and restores highlight/annotation state.

Engine-specific state restoration (ProseMirror EditorState, CodeMirror state) is delegated to a protected `snapshotEngineState()` / `restoreEngineState()` pair that subclasses override.

- [ ] **Step 3: Add "AI editing" indicator to component**

When a session is active, show a banner with the session owner and a cancel button.

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "feat: edit sessions with exclusive lock, cancel rollback, and AI editing indicator"
```

---

## Batch 4: Dual Mode + MCP

After this batch: the markdown editor supports WYSIWYG ↔ source toggling, split mode with scroll sync, and MCP tools can drive the editor.

### Task 10: Dual-mode toggle (WYSIWYG ↔ source)

**Files:**
- Modify: `packages/pages-markdown-editor/src/pages-markdown-editor.ts` (mode switching)
- Test: `packages/pages-markdown-editor/src/dual-mode.test.ts`

**Interfaces:**
- Consumes: `pages-code-editor` via dynamic import, `EDITABLE_TEXT` symbol
- Produces: mode toggle between WYSIWYG and source views with bridge swap

- [ ] **Step 1: Write failing test for mode toggle**

```typescript
describe('Dual mode toggle', () => {
  it('switches from WYSIWYG to source mode');
  it('preserves document content across mode switch');
  it('swaps EDITABLE_TEXT bridge on toggle');
  it('emits mode-changed event');
  it('handles import failure gracefully');
});
```

- [ ] **Step 2: Implement toggle logic**

WYSIWYG→Source: serialize ProseMirror doc, dynamic import `pages-code-editor`, create CodeMirror with markdown content, swap `EDITABLE_TEXT` to `CodeEditorBridge`.

Source→WYSIWYG: get markdown from CodeMirror, re-parse into ProseMirror, swap back to `MarkdownEditorBridge`.

- [ ] **Step 3: Run tests, commit**

```bash
git commit -m "feat: WYSIWYG/source dual-mode toggle with dynamic import and bridge swap"
```

---

### Task 11: Split mode with scroll sync

**Files:**
- Create: `packages/pages-markdown-editor/src/scroll-sync.ts`
- Modify: `packages/pages-markdown-editor/src/pages-markdown-editor.ts` (split layout)
- Test: `packages/pages-markdown-editor/src/scroll-sync.test.ts`

**Interfaces:**
- Consumes: heading-based anchor pairing pattern from `document-diff`
- Produces: synchronized scroll between WYSIWYG and source panes

- [ ] **Step 1: Write failing test for scroll sync engine**

```typescript
describe('ScrollSync', () => {
  it('builds anchors from matching headings');
  it('interpolates position between anchors');
  it('handles bidirectional sync without infinite loop');
  it('uses position-aware interpolation, not linear');
});
```

- [ ] **Step 2: Implement scroll sync**

Extract the heading-based anchor pairing logic. Use position-aware interpolation (query actual element heights in both panels).

- [ ] **Step 3: Wire split layout into component**

When `mode === 'split'`, show both Milkdown and CodeMirror side-by-side. Sync edits with 150ms debounce. Sync transactions dispatched with `addToHistory: false` on the receiving side.

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "feat: split mode with heading-based scroll sync and debounced edit synchronization"
```

---

### Task 12: MCP tool adapter

**Files:**
- Create: `packages/pages-editor-core/src/mcp-tool-adapter.ts`
- Create: `packages/pages-editor-core/src/mcp-tool-definitions.ts`
- Test: `packages/pages-editor-core/src/mcp-tool-adapter.test.ts`
- Modify: `packages/pages-editor-core/src/index.ts` (export)

**Interfaces:**
- Consumes: `EditableText` interface
- Produces: `McpToolAdapter` class with `handleToolCall(name, params)` method, `MCP_TOOL_DEFINITIONS` array

- [ ] **Step 1: Write failing test for MCP tool adapter**

```typescript
describe('McpToolAdapter', () => {
  it('editor_get_content calls getText()', async () => {
    const mock = createMockEditableText('hello world');
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_get_content', {});
    expect(result).toEqual({ content: 'hello world' });
  });

  it('editor_replace_range calls replaceRange()', async () => {
    const mock = createMockEditableText('hello world');
    const adapter = new McpToolAdapter(mock);
    await adapter.handleToolCall('editor_replace_range', {
      from: { line: 0, col: 0 }, to: { line: 0, col: 5 }, text: 'goodbye'
    });
    expect(mock.replaceRange).toHaveBeenCalledWith(
      { line: 0, col: 0 }, { line: 0, col: 5 }, 'goodbye'
    );
  });

  it('editor_find_text returns positions', async () => {
    const mock = createMockEditableText('hello hello');
    mock.findText.mockReturnValue([{ line: 0, col: 0 }, { line: 0, col: 6 }]);
    const result = await adapter.handleToolCall('editor_find_text', { query: 'hello' });
    expect(result.matches).toHaveLength(2);
  });

  it('editor_begin_session handles contention error', async () => {
    const mock = createMockEditableText('');
    mock.beginEditSession.mockImplementation(() => { throw new EditSessionActiveError('claude', '2026-10-07T00:00:00Z'); });
    const result = await adapter.handleToolCall('editor_begin_session', { owner: 'gpt' });
    expect(result.error).toBe('session_active');
    expect(result.owner).toBe('claude');
  });
});
```

- [ ] **Step 2: Implement McpToolAdapter**

A class that takes an `EditableText` and maps tool call names to method calls. Includes position validation, error formatting, and the `editor_find_heading` semantic helper.

- [ ] **Step 3: Define MCP tool schemas**

`mcp-tool-definitions.ts` exports an array of MCP tool definition objects (name, description, input schema) for all 16 tools.

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "feat: MCP tool adapter mapping 16 editor tools to EditableText interface"
```

---

## Batch 5: document-diff Extraction

After this batch: generic diff infrastructure lives in `pages-document-diff`, blocks-ui's `DrafthouseDocumentDiff` extends it with domain-specific review features.

### Task 13: Extract PagesDocumentDiff to pages

**Files:**
- Create: `packages/pages-document-diff/package.json`
- Create: `packages/pages-document-diff/src/pages-document-diff.ts` (generic base, ~1025 lines)
- Create: `packages/pages-document-diff/src/types.ts`
- Create: `packages/pages-document-diff/src/index.ts`
- Test: move/adapt existing tests from blocks-ui

**Interfaces:**
- Consumes: `marked` for markdown rendering
- Produces: `<pages-document-diff>` generic component with LCS diff, word highlights, canvas minimap, scroll sync, split/unified views

- [ ] **Step 1: Create package and extract generic code**

Copy the generic portions (~1025 lines) from `blocks-ui/components/document-workbench/src/document-diff.ts`. Remove thread management, timeline snapshot fetching, and selection-to-thread bridge logic. Refactor `connectedCallback` into a base method that subclasses can extend.

- [ ] **Step 2: Verify extraction compiles and tests pass**

Port existing `document-diff.test.ts` from blocks-ui. Ensure all generic diff, scroll sync, and navigation tests pass.

- [ ] **Step 3: Commit**

```bash
git add packages/pages-document-diff/ package.json yarn.lock
git commit -m "feat: extract PagesDocumentDiff from blocks-ui with generic diff/scroll-sync infrastructure"
```

---

### Task 14: DrafthouseDocumentDiff subclass in blocks-ui

**Files:**
- Modify: `blocks-ui/components/document-workbench/package.json` (add pages-document-diff dep)
- Modify: `blocks-ui/components/document-workbench/src/document-diff.ts` (reduce to subclass)

**Interfaces:**
- Consumes: `PagesDocumentDiff` from `pages-document-diff`
- Produces: `DrafthouseDocumentDiff extends PagesDocumentDiff` with thread/timeline/selection features

- [ ] **Step 1: Replace document-diff.ts with thin subclass**

```typescript
import { PagesDocumentDiff } from '@casehubio/pages-document-diff';
import { customElement } from 'lit/decorators.js';
import { onPagesEvent } from '@casehubio/pages-data';

@customElement('document-diff')
export class DrafthouseDocumentDiff extends PagesDocumentDiff {
  // ~125 lines of domain-specific thread/timeline/selection logic
  // override connectedCallback() to add domain listeners
}
```

- [ ] **Step 2: Run blocks-ui tests to verify nothing breaks**

Run: `yarn workspace @casehubio/blocks-ui-document-workbench test`

- [ ] **Step 3: Commit**

```bash
git commit -m "refactor: DrafthouseDocumentDiff extends PagesDocumentDiff, domain logic stays in blocks-ui"
```

---

## References

- [2026-10-07-milkdown-editor-design.md] — design spec this plan implements
- [packages/pages-code-editor/src/code-editor-bridge.ts] — existing bridge implementation (lines 45-119)
- [packages/pages-code-editor/src/pages-code-editor.ts] — existing LIT wrapper (line 14: EDITABLE_TEXT symbol)
- [packages/pages-aria/src/executor/editable-text.ts] — existing interface (lines 6-19, to be moved)
- [packages/pages-aria/src/executor/command-executor.ts] — existing command dispatching
- [blocks-ui/components/document-workbench/src/document-diff.ts] — scroll sync, diff engine (1217 lines, to be extracted)
- [decisions.md D0-D9] — all design decisions captured during brainstorming
