# Scenario-Driven Interactive Tutorials — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #443 — Scenario-driven interactive tutorials with editor automation
**Issue group:** #443

**Goal:** Add editor automation commands to the scenario executor so tutorials can drive CodeMirror editors character-by-character, with highlights and spotlights, using the existing tutorial infrastructure (narrative panel, controller, transport controls).

**Architecture:** ScenarioEditableText SPI defines a DOM-level interface for text editor manipulation. `pages-code-editor` implements it via a CodeMirror 6 bridge. Seven new ARIA actions (`editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`) extend the scenario executor. The sectioned runner delegates to a unified `executeStep` dispatch function. Tutorials use standard `hands-on` format.

**Tech Stack:** TypeScript, Lit 3, CodeMirror 6, vitest

## Global Constraints

- Zero new package dependencies — all work is within existing packages
- SPI uses `Position {line, col}` consistently — no raw character offsets in public API
- SPI discovery via `Symbol.for('scenario-editable-text')` with ancestor walk through shadow DOM hosts
- `typing: progressive` reuses `progressiveFill` algorithm from `scenario-handler.ts`
- `editor-completion` skips gracefully when LSP is not connected (timeout 2s, log warning, continue)
- All new ARIA actions go through the existing `aria` delivery channel — no new delivery channels
- `pages-aria` does NOT depend on `pages-code-editor` or `pages-builder` — SPI is discovered at runtime via the Symbol

---

## Batch 1: SPI Foundation

### Task 1: ScenarioEditableText SPI — interface, discovery, types

**Files:**
- Create: `packages/pages-aria/src/executor/editable-text.ts`
- Test: `packages/pages-aria/src/executor/editable-text.test.ts`

**Interfaces:**
- Consumes: nothing (first task)
- Produces:
  - `Position`: `{ line: number; col: number }`
  - `ScenarioEditableText`: interface with `insertText`, `replaceRange`, `deleteRange`, `setCursor`, `getCursor`, `getText`, `getLineCount`, `setContent`, `highlight`, `clearHighlights`, `triggerCompletion?`, `selectCompletion?`
  - `EDITABLE_TEXT`: `Symbol.for('scenario-editable-text')`
  - `isEditableText(el: Element): boolean`
  - `findEditableText(el: Element): ScenarioEditableText | null`

- [ ] **Step 1: Write failing tests for findEditableText**

```typescript
// packages/pages-aria/src/executor/editable-text.test.ts
import { describe, it, expect } from 'vitest';
import { EDITABLE_TEXT, findEditableText, isEditableText } from './editable-text.js';
import type { ScenarioEditableText } from './editable-text.js';

function mockEditor(): ScenarioEditableText {
  return {
    insertText: () => {},
    replaceRange: () => {},
    deleteRange: () => {},
    setCursor: () => {},
    getCursor: () => ({ line: 1, col: 0 }),
    getText: () => '',
    getLineCount: () => 1,
    setContent: () => {},
    highlight: () => {},
    clearHighlights: () => {},
  };
}

describe('findEditableText', () => {
  it('finds SPI on the element itself', () => {
    const el = document.createElement('div');
    (el as any)[EDITABLE_TEXT] = mockEditor();
    expect(findEditableText(el)).toBeDefined();
  });

  it('returns null when no SPI found', () => {
    const el = document.createElement('div');
    expect(findEditableText(el)).toBeNull();
  });

  it('walks up to parent with SPI', () => {
    const parent = document.createElement('div');
    (parent as any)[EDITABLE_TEXT] = mockEditor();
    const child = document.createElement('span');
    parent.appendChild(child);
    expect(findEditableText(child)).toBeDefined();
  });

  it('isEditableText returns true when symbol present', () => {
    const el = document.createElement('div');
    (el as any)[EDITABLE_TEXT] = mockEditor();
    expect(isEditableText(el)).toBe(true);
  });

  it('isEditableText returns false when symbol absent', () => {
    const el = document.createElement('div');
    expect(isEditableText(el)).toBe(false);
  });
});
```

Run: `yarn workspace @casehubio/pages-aria test editable-text`
Expected: FAIL — module not found

- [ ] **Step 2: Implement SPI types and discovery**

```typescript
// packages/pages-aria/src/executor/editable-text.ts

export interface Position {
  line: number;
  col: number;
}

export interface ScenarioEditableText {
  insertText(text: string): void;
  replaceRange(from: Position, to: Position, text: string): void;
  deleteRange(from: Position, to: Position): void;
  setCursor(line: number, col: number): void;
  getCursor(): Position;
  getText(): string;
  getLineCount(): number;
  setContent(text: string): void;
  highlight(from: Position, to: Position, style?: 'pulse' | 'underline' | 'glow'): void;
  clearHighlights(): void;
  triggerCompletion?(): void;
  selectCompletion?(label: string): boolean;
}

export const EDITABLE_TEXT = Symbol.for('scenario-editable-text');

export function isEditableText(el: Element): el is Element & { [EDITABLE_TEXT]: ScenarioEditableText } {
  return EDITABLE_TEXT in el;
}

export function findEditableText(el: Element): ScenarioEditableText | null {
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

Run: `yarn workspace @casehubio/pages-aria test editable-text`
Expected: PASS

- [ ] **Step 3: Export from executor barrel**

Add to `packages/pages-aria/src/executor/index.ts`:
```typescript
export { EDITABLE_TEXT, findEditableText, isEditableText } from './editable-text.js';
export type { Position, ScenarioEditableText } from './editable-text.js';
```

- [ ] **Step 4: Commit**

```bash
git add packages/pages-aria/src/executor/editable-text.ts packages/pages-aria/src/executor/editable-text.test.ts packages/pages-aria/src/executor/index.ts
git commit -m "feat(#443): ScenarioEditableText SPI — interface, discovery, ancestor walk Refs #443"
```

### Task 2: CodeMirror SPI implementation in pages-code-editor

**Files:**
- Create: `packages/pages-code-editor/src/code-editor-bridge.ts`
- Modify: `packages/pages-code-editor/src/pages-code-editor.ts` (add SPI registration)
- Test: `packages/pages-code-editor/src/code-editor-bridge.test.ts`

**Interfaces:**
- Consumes: `ScenarioEditableText`, `Position`, `EDITABLE_TEXT` (from Task 1 — imported as type only, no package dependency)
- Produces: `CodeEditorBridge` class implementing `ScenarioEditableText`, registered on `pages-code-editor` element via Symbol

**Port reference:** Read `packages/pages-code-editor/src/pages-code-editor.ts` to understand the `EditorView` lifecycle — when it's created, how `_editorView` is accessed, the Compartment pattern for dynamic properties.

- [ ] **Step 1: Write failing tests for CodeEditorBridge**

```typescript
// packages/pages-code-editor/src/code-editor-bridge.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { CodeEditorBridge } from './code-editor-bridge.js';
import { EditorState } from '@codemirror/state';
import { EditorView } from '@codemirror/view';

describe('CodeEditorBridge', () => {
  let view: EditorView;
  let bridge: CodeEditorBridge;

  beforeEach(() => {
    const container = document.createElement('div');
    document.body.appendChild(container);
    view = new EditorView({
      state: EditorState.create({ doc: 'line one\nline two\nline three' }),
      parent: container,
    });
    bridge = new CodeEditorBridge(view);
  });

  it('getText returns full document', () => {
    expect(bridge.getText()).toBe('line one\nline two\nline three');
  });

  it('getLineCount returns correct count', () => {
    expect(bridge.getLineCount()).toBe(3);
  });

  it('insertText inserts at cursor', () => {
    bridge.setCursor(1, 0);
    bridge.insertText('hello ');
    expect(bridge.getText()).toContain('hello line one');
  });

  it('setCursor and getCursor round-trip', () => {
    bridge.setCursor(2, 5);
    const pos = bridge.getCursor();
    expect(pos.line).toBe(2);
    expect(pos.col).toBe(5);
  });

  it('setContent replaces entire buffer', () => {
    bridge.setContent('new content');
    expect(bridge.getText()).toBe('new content');
    expect(bridge.getLineCount()).toBe(1);
  });

  it('replaceRange replaces specified range', () => {
    bridge.replaceRange({ line: 1, col: 0 }, { line: 1, col: 4 }, 'LINE');
    expect(bridge.getText()).toContain('LINE one');
  });

  it('deleteRange removes specified range', () => {
    bridge.deleteRange({ line: 1, col: 0 }, { line: 1, col: 5 });
    expect(bridge.getText().startsWith('one')).toBe(true);
  });
});
```

Run: `yarn workspace @casehubio/pages-code-editor test code-editor-bridge`
Expected: FAIL

- [ ] **Step 2: Implement CodeEditorBridge**

```typescript
// packages/pages-code-editor/src/code-editor-bridge.ts
import type { EditorView } from '@codemirror/view';
import { Decoration, type DecorationSet } from '@codemirror/view';
import { StateEffect, StateField } from '@codemirror/state';

export interface Position {
  line: number;
  col: number;
}

const addHighlight = StateEffect.define<{ from: number; to: number; class: string }>();
const clearHighlights = StateEffect.define<null>();

const highlightField = StateField.define<DecorationSet>({
  create: () => Decoration.none,
  update(decorations, tr) {
    decorations = decorations.map(tr.changes);
    for (const e of tr.effects) {
      if (e.is(addHighlight)) {
        decorations = decorations.update({
          add: [Decoration.mark({ class: e.value.class }).range(e.value.from, e.value.to)],
        });
      } else if (e.is(clearHighlights)) {
        decorations = Decoration.none;
      }
    }
    return decorations;
  },
  provide: (f) => EditorView.decorations.from(f),
});

export class CodeEditorBridge {
  constructor(private readonly view: EditorView) {}

  private toOffset(pos: Position): number {
    const line = this.view.state.doc.line(pos.line);
    return line.from + pos.col;
  }

  private toPosition(offset: number): Position {
    const line = this.view.state.doc.lineAt(offset);
    return { line: line.number, col: offset - line.from };
  }

  insertText(text: string): void {
    const cursor = this.view.state.selection.main.head;
    this.view.dispatch({ changes: { from: cursor, insert: text } });
  }

  replaceRange(from: Position, to: Position, text: string): void {
    this.view.dispatch({
      changes: { from: this.toOffset(from), to: this.toOffset(to), insert: text },
    });
  }

  deleteRange(from: Position, to: Position): void {
    this.view.dispatch({
      changes: { from: this.toOffset(from), to: this.toOffset(to) },
    });
  }

  setCursor(line: number, col: number): void {
    const anchor = this.toOffset({ line, col });
    this.view.dispatch({ selection: { anchor } });
  }

  getCursor(): Position {
    return this.toPosition(this.view.state.selection.main.head);
  }

  getText(): string {
    return this.view.state.doc.toString();
  }

  getLineCount(): number {
    return this.view.state.doc.lines;
  }

  setContent(text: string): void {
    this.view.dispatch({
      changes: { from: 0, to: this.view.state.doc.length, insert: text },
    });
  }

  highlight(from: Position, to: Position, style: string = 'pulse'): void {
    const fromOffset = this.toOffset(from);
    const toOffset = this.toOffset(to);
    if (!this.view.state.field(highlightField, false)) {
      this.view.dispatch({ effects: StateEffect.appendConfig.of([highlightField]) });
    }
    this.view.dispatch({
      effects: addHighlight.of({
        from: fromOffset, to: toOffset,
        class: `scenario-highlight-${style}`,
      }),
    });
  }

  clearHighlights(): void {
    if (this.view.state.field(highlightField, false)) {
      this.view.dispatch({ effects: clearHighlights.of(null) });
    }
  }
}

export { highlightField, addHighlight, clearHighlights };
```

Run: `yarn workspace @casehubio/pages-code-editor test code-editor-bridge`
Expected: PASS

- [ ] **Step 3: Register SPI on pages-code-editor element**

In `packages/pages-code-editor/src/pages-code-editor.ts`, add SPI registration in `connectedCallback` (or after `EditorView` creation):

```typescript
import { CodeEditorBridge } from './code-editor-bridge.js';

const EDITABLE_TEXT = Symbol.for('scenario-editable-text');

// After EditorView is created:
(this as any)[EDITABLE_TEXT] = new CodeEditorBridge(this._editorView);
```

- [ ] **Step 4: Run full test suite**

Run: `yarn workspace @casehubio/pages-code-editor test && yarn workspace @casehubio/pages-aria test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-code-editor/
git commit -m "feat(#443): CodeMirror SPI bridge — insertText, highlight, setContent Refs #443"
```

---

## Batch 2: Executor Integration

### Task 3: Execution path unification — replace executeAriaStep in sectioned-runner

**Files:**
- Modify: `packages/pages-aria/src/executor/command-executor.ts` (add `executeStep` dispatch function)
- Modify: `packages/pages-aria/src/scenario/sectioned-runner.ts` (replace inline `executeAriaStep` with `executeStep`)
- Test: `packages/pages-aria/src/executor/command-executor.test.ts` (new — test dispatch)

**Interfaces:**
- Consumes: `resolveTarget`, `click`, `fill`, `select`, `expand`, `collapse`, `assertState`, `waitFor` (from command-executor.ts), `showSpotlight` (from spotlight.ts)
- Produces: `executeStep(step: ScenarioStep, eventTarget?: EventTarget, speed?: number): Promise<void>`

- [ ] **Step 1: Write failing test for executeStep dispatch**

```typescript
// packages/pages-aria/src/executor/command-executor.test.ts
import { describe, it, expect, vi } from 'vitest';
import { executeStep } from './command-executor.js';

describe('executeStep', () => {
  it('throws on unknown action', async () => {
    await expect(executeStep({
      delivery: 'aria', action: 'nonexistent',
    } as any)).rejects.toThrow('Unknown action');
  });
});
```

Run: `yarn workspace @casehubio/pages-aria test command-executor`
Expected: FAIL — `executeStep` not exported

- [ ] **Step 2: Implement executeStep dispatch**

Add to `packages/pages-aria/src/executor/command-executor.ts`:

```typescript
import type { ScenarioStep } from '../scenario/types.js';
import { showSpotlight } from './spotlight.js';

export async function executeStep(
  step: ScenarioStep, eventTarget?: EventTarget, speed = 1.0,
): Promise<void> {
  if (step.delivery !== 'aria') return;
  const s = step as Record<string, unknown>;
  switch (step.action) {
    case 'click': return click(step.target!);
    case 'fill': return fill(step.target!, s['value'] as string);
    case 'select': return select(step.target!, s['value'] as string);
    case 'expand': return expand(step.target!);
    case 'collapse': return collapse(step.target!);
    case 'assert': return assertState(step.target!, s['state'] as any);
    case 'wait': return waitFor(step.target!, s['state'] as any, (s['timeout'] as number) ?? 5000);
    case 'navigate': window.location.href = s['value'] as string; return;
    default: throw new Error(`Unknown action: ${step.action}`);
  }
}
```

Run: `yarn workspace @casehubio/pages-aria test command-executor`
Expected: PASS

- [ ] **Step 3: Wire sectioned-runner to use executeStep**

In `packages/pages-aria/src/scenario/sectioned-runner.ts`, replace the inline `executeAriaStep` function with a call to `executeStep`:

```typescript
import { executeStep } from '../executor/command-executor.js';

// Replace the inline executeAriaStep call with:
await executeStep(step, options.eventTarget, rs.speed);
```

- [ ] **Step 4: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria test`
Expected: PASS — all 250 tests

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#443): unify execution path — executeStep dispatch in command-executor Refs #443"
```

### Task 4: New ARIA actions — parser + executor handlers + progressiveInsert

**Files:**
- Modify: `packages/pages-aria/src/scenario/parser.ts` (extend ARIA_ACTIONS, catch-all body passthrough, spotlight handler)
- Modify: `packages/pages-aria/src/executor/command-executor.ts` (add editor action handlers, progressiveInsert)
- Test: `packages/pages-aria/src/scenario/parser.test.ts` (add tests for new actions)
- Test: `packages/pages-aria/src/executor/command-executor.test.ts` (add editor action tests)

**Interfaces:**
- Consumes: `findEditableText`, `ScenarioEditableText`, `Position` (from Task 1), `executeStep` (from Task 3)
- Produces: Extended `executeStep` handling `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`, `spotlight`, `show-markdown`

- [ ] **Step 1: Write failing parser tests for new actions**

```typescript
// Add to packages/pages-aria/src/scenario/parser.test.ts
it('parses editor-insert shorthand', () => {
  const yaml = `scenario: test
sections:
  - title: Test
    steps:
      - editor-insert:
          role: textbox
          name: "YAML editor"
          value: "hello"
          typing: progressive`;
  const parsed = parseScenario(yaml);
  const step = (parsed as any).sections[0].steps[0];
  expect(step.action).toBe('editor-insert');
  expect(step.target.role).toBe('textbox');
  expect(step.value).toBe('hello');
  expect(step.typing).toBe('progressive');
});

it('parses spotlight with nested target', () => {
  const yaml = `scenario: test
sections:
  - title: Test
    steps:
      - spotlight:
          target:
            role: tree
            name: "Document outline"
          content: "This is the tree view"`;
  const parsed = parseScenario(yaml);
  const step = (parsed as any).sections[0].steps[0];
  expect(step.action).toBe('spotlight');
  expect(step.target.role).toBe('tree');
  expect(step.content).toBe('This is the tree view');
});

it('parses editor-highlight with from/to positions', () => {
  const yaml = `scenario: test
sections:
  - title: Test
    steps:
      - editor-highlight:
          role: textbox
          name: "YAML editor"
          from: {line: 1, col: 0}
          to: {line: 3, col: 10}
          style: pulse`;
  const parsed = parseScenario(yaml);
  const step = (parsed as any).sections[0].steps[0];
  expect(step.action).toBe('editor-highlight');
  expect(step.from).toEqual({ line: 1, col: 0 });
  expect(step.to).toEqual({ line: 3, col: 10 });
  expect(step.style).toBe('pulse');
});
```

Run: `yarn workspace @casehubio/pages-aria test parser`
Expected: FAIL — new actions not recognized

- [ ] **Step 2: Extend parser — ARIA_ACTIONS set and catch-all body passthrough**

In `packages/pages-aria/src/scenario/parser.ts`:

1. Add to `ARIA_ACTIONS`: `spotlight`, `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`

2. Replace the explicit `value`/`state`/`timeout` extraction in `expandAriaShorthand` with a catch-all passthrough:

```typescript
const targetKeys = new Set(['role', 'name', 'index', 'within']);
const step: ScenarioStep = { delivery: 'aria', name: autoName, action, target };
for (const [key, val] of Object.entries(body)) {
  if (!targetKeys.has(key) && val != null) {
    (step as Record<string, unknown>)[key] = val;
  }
}
```

3. Add spotlight special-case handler (nested target extraction).

Run: `yarn workspace @casehubio/pages-aria test parser`
Expected: PASS

- [ ] **Step 3: Write failing executor tests for editor actions**

```typescript
// Add to packages/pages-aria/src/executor/command-executor.test.ts
import { EDITABLE_TEXT } from './editable-text.js';

describe('editor actions', () => {
  it('editor-insert calls insertText on SPI', async () => {
    const insertText = vi.fn();
    const el = document.createElement('div');
    el.setAttribute('role', 'textbox');
    el.setAttribute('aria-label', 'YAML editor');
    (el as any)[EDITABLE_TEXT] = {
      insertText, setCursor: vi.fn(), getCursor: () => ({ line: 1, col: 0 }),
      getText: () => '', getLineCount: () => 1, setContent: vi.fn(),
      replaceRange: vi.fn(), deleteRange: vi.fn(),
      highlight: vi.fn(), clearHighlights: vi.fn(),
    };
    document.body.appendChild(el);

    await executeStep({
      delivery: 'aria', action: 'editor-insert',
      target: { role: 'textbox', name: 'YAML editor' },
      value: 'hello', typing: 'instant',
    } as any);

    expect(insertText).toHaveBeenCalledWith('hello');
    document.body.removeChild(el);
  });
});
```

Run: `yarn workspace @casehubio/pages-aria test command-executor`
Expected: FAIL — editor-insert not handled

- [ ] **Step 4: Implement editor action handlers in executeStep**

Add handlers for all seven editor actions plus `spotlight` and `show-markdown` to the `executeStep` switch statement in `command-executor.ts`. Implement `progressiveInsert` adapting the phased algorithm from `scenario-handler.ts`.

- [ ] **Step 5: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#443): editor ARIA actions — parser + executor + progressiveInsert Refs #443"
```

---

## Batch 3: Tutorial Delivery

### Task 5: Tutorial content rework — yaml-editor to hands-on scenarios

**Files:**
- Modify: `tutorials/yaml-composition/tutorial.yaml` (rewrite from yaml-editor sections to hands-on sections with editor ARIA steps)
- Modify: `tutorials/yaml-composition/content/*.md` (adjust narrative for demonstration mode)
- Modify: `tutorials/index.html` (set contentType to hands-on)
- Modify: `scripts/build-tutorial-registry.mjs` (no changes needed — hands-on detection already works)

**Interfaces:**
- Consumes: All editor ARIA actions from Task 4, builder-shell ARIA role/name
- Produces: 15-section hands-on tutorial with slide sections (narrative) and editor automation sections

- [ ] **Step 1: Rewrite tutorial.yaml**

Convert each of the 15 sections from `yaml-editor` format (initialYaml, expectedKeys, expectedStructure) to `hands-on` format (content template + editor ARIA steps). For each concept:

- Slide section: narrative markdown (existing content files, with minor adjustments for "watch" rather than "do" framing)
- Editor automation section: `editor-set-content` or `editor-insert` steps with `typing: progressive`, followed by `editor-highlight` and `spotlight` callouts

Example conversion for Step 6 (ForEach Basics):
```yaml
  - title: "ForEach Basics — Explanation"
    content:
      type: template
      path: content/06-foreach-basics.md
    steps: []

  - title: "ForEach Basics — Demo"
    steps:
      - editor-set-content:
          role: textbox
          name: "Page YAML source"
          value: |
            pages:
              - name: Metrics Dashboard
                components:
                  cpu-metric:
                    type: metric
                    properties:
                      field: cpu
                  memory-metric:
                    type: metric
                    properties:
                      field: memory
                  disk-metric:
                    type: metric
                    properties:
                      field: disk
          typing: progressive
      - editor-highlight:
          role: textbox
          name: "Page YAML source"
          from: {line: 4, col: 4}
          to: {line: 15, col: 18}
          style: pulse
      - spotlight:
          target: {role: generic, name: "Page YAML source"}
          content: "Three identical components — only the field value differs"
      - editor-set-content:
          role: textbox
          name: "Page YAML source"
          value: |
            pages:
              - name: Metrics Dashboard
                components:
                  metric:
                    type: metric
                    forEach:
                      as: field
                      in: [cpu, memory, disk]
                    properties:
                      field: ${each.field}
          typing: progressive
      - editor-highlight:
          role: textbox
          name: "Page YAML source"
          from: {line: 6, col: 8}
          to: {line: 8, col: 30}
          style: glow
      - spotlight:
          target: {role: tree, name: "Document outline"}
          content: "forEach generated three components from one template"
```

- [ ] **Step 2: Update narrative markdown files**

Adjust the "Your task:" sections in content files to be observational: "Watch as..." instead of "Add a..." — the user is watching, not editing.

- [ ] **Step 3: Update tutorials/index.html**

Change the yaml-composition entry's `contentType` from `'yaml-editor'` to `'hands-on'`.

- [ ] **Step 4: Build and verify registry**

Run: `node scripts/build-tutorial-registry.mjs`
Verify: yaml-composition has `contentType: 'hands-on'`

- [ ] **Step 5: Commit**

```bash
git add tutorials/
git commit -m "feat(#443): rewrite yaml-composition tutorial as hands-on scenario Refs #443"
```

### Task 6: Tutorial host app — builder-shell as tutorial target

**Files:**
- Modify: `tutorials/index.html` (render builder-shell alongside tutorial host)
- Modify or create: tutorial app layout that renders builder-shell as the target application with scenario narrative and controller

**Interfaces:**
- Consumes: `<pages-builder-shell>` (renders as untyped custom element), `<pages-tutorial-host>` (existing), `<pages-scenario-controller>` (sidebar), `<pages-scenario-narrative>` (narrative panel)
- Produces: Integrated tutorial page where builder-shell is the "app" that the scenario executor drives

- [ ] **Step 1: Update tutorial page layout**

The tutorial page needs to render `<pages-builder-shell>` in the main area alongside the scenario narrative and controller. The builder-shell is the "application" the tutorial drives — similar to how the form-automation tutorial renders a form.

Update `tutorials/index.html` to create a layout with:
- Left: builder-shell (the editor + tree + preview)
- Right: scenario narrative panel (shows slide content between automated sections)
- Floating: scenario controller (existing compact mode — bottom-right pill)

- [ ] **Step 2: Verify tutorial runs end-to-end**

Start dev server: `npx vite --config tutorials/vite.config.ts`
Navigate to the tutorial page, select yaml-composition tutorial, and verify:
- Slide sections show narrative content
- Editor automation sections type YAML into the builder
- Spotlights and highlights appear
- Transport controls pace the automation
- Chapter navigation shows all sections

- [ ] **Step 3: Commit**

```bash
git add tutorials/
git commit -m "feat(#443): tutorial host app — builder-shell as scenario target Refs #443"
```

---

## Out of scope (tracked separately)

- **LSP wiring for workbench editor** — `editor-completion` command ready but LSP not connected to workbench yet
- **User practice mode** — validation infrastructure preserved from #435 for future use
- **New tutorial content beyond yaml-composition** — additional tutorials are separate issues
- **Server-side scenario handler updates** — new actions work in-process via `runSectionedScenario`

## References

- [2026-09-14-scenario-driven-tutorials-design.md] — design spec this plan implements
- [decisions.md] — 7 decisions (D1-D7)
- [packages/pages-aria/src/scenario/parser.ts] — ARIA_ACTIONS set, shorthand expansion
- [packages/pages-aria/src/executor/command-executor.ts] — ARIA command execution
- [packages/pages-aria/src/executor/visual-feedback.ts] — typeText, progressiveFill
- [packages/pages-aria/src/executor/spotlight.ts] — showSpotlight, SpotlightConfig
- [packages/pages-aria/src/scenario/sectioned-runner.ts] — TutorialRunner, step progression
- [packages/pages-aria/src/tutorial/tutorial-host.ts] — tutorial catalog and rendering
- [packages/pages-code-editor/src/pages-code-editor.ts] — CodeMirror 6 component
- [packages/pages-builder/src/shell/builder-shell.ts] — builder workbench
- GE-20260905-3e4256 — drawSelection() required for cursor in shadow DOM
- GE-20260905-5986c1 — Compartment pattern for dynamic CM6 properties in Lit
- casehubio/casehub-pages#435 — yaml-core engine + tutorial content (landed)
- casehubio/casehub-pages#443 — this issue
