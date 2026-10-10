# Rich Highlight API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #535 — Rich highlight API — styling, semantic targeting, animated overlays, LLM sync
**Issue group:** #535

**Goal:** Extend the highlight system to support configurable styling, semantic text targeting, animated overlay rendering, read-back queries, and a line reader for LLM-synchronized document tracking.

**Architecture:** Replace the `HighlightStyle` string union with a composable `HighlightOptions` interface that supports custom background, border, text-decoration, labels, and groups. Named presets (`pulse`, `glow`, etc.) become aliases. New semantic targeting methods (`highlightSentence`, `highlightText`, extended `highlightLine`) find ranges by content. A `HighlightRenderer` replaces `Decoration.inline` with absolutely-positioned overlay `<div>`s using CSS transitions for smooth animation. A `LineReader` provides a tracked highlight that advances through the document. All new methods get MCP tool definitions.

**Tech Stack:** TypeScript, ProseMirror (via Milkdown), LIT (overlay DOM), Vitest

## Global Constraints

- All new types go in `packages/pages-editor-core/src/types.ts`
- All new abstract/concrete methods follow the existing `EditableTextBridge` → `MarkdownEditorBridge` split
- MCP tool names use `editor_` prefix (existing convention)
- `HighlightStyle` string union remains as an alias mapping to `HighlightOptions` for backward compat — existing callers passing `'pulse'` must keep working
- No new package dependencies — use ProseMirror/Milkdown APIs already in the dependency tree
- Tests use Vitest with the existing `TestBridge` pattern from `editable-text-bridge.test.ts`

---

## Batch 1: Configurable Styling Foundation

### Task 1: HighlightOptions type and preset resolution

**Files:**
- Modify: `packages/pages-editor-core/src/types.ts:6` (replace `HighlightStyle` union)
- Create: `packages/pages-editor-core/src/highlight-options.ts` (preset map + resolver)
- Test: `packages/pages-editor-core/src/highlight-options.test.ts`
- Modify: `packages/pages-editor-core/src/index.ts` (export new module)

**Interfaces:**
- Consumes: nothing (foundation)
- Produces: `HighlightOptions` interface, `HighlightStyle` type alias (backward compat), `resolveHighlightStyle(style?: HighlightStyle | HighlightOptions): HighlightOptions`, `highlightOptionsToCSS(opts: HighlightOptions): string`

- [ ] **Step 1: Write failing tests for HighlightOptions resolution**

```typescript
// highlight-options.test.ts
import { describe, it, expect } from 'vitest';
import { resolveHighlightStyle, highlightOptionsToCSS } from './highlight-options.js';

describe('resolveHighlightStyle', () => {
  it('returns HighlightOptions unchanged', () => {
    const opts = { background: 'rgba(255,0,0,0.2)', border: '1px solid red' };
    expect(resolveHighlightStyle(opts)).toEqual(opts);
  });

  it('resolves "pulse" preset to HighlightOptions', () => {
    const result = resolveHighlightStyle('pulse');
    expect(result.background).toBeDefined();
  });

  it('resolves "underline" preset', () => {
    const result = resolveHighlightStyle('underline');
    expect(result.textDecoration).toBeDefined();
  });

  it('resolves "glow" preset', () => {
    const result = resolveHighlightStyle('glow');
    expect(result.background).toBeDefined();
  });

  it('resolves "box" preset', () => {
    const result = resolveHighlightStyle('box');
    expect(result.border).toBeDefined();
  });

  it('resolves "error" preset', () => {
    const result = resolveHighlightStyle('error');
    expect(result.background).toBeDefined();
  });

  it('resolves "suggestion" preset', () => {
    const result = resolveHighlightStyle('suggestion');
    expect(result.background).toBeDefined();
  });

  it('defaults to pulse when undefined', () => {
    const result = resolveHighlightStyle(undefined);
    expect(result).toEqual(resolveHighlightStyle('pulse'));
  });
});

describe('highlightOptionsToCSS', () => {
  it('converts background', () => {
    const css = highlightOptionsToCSS({ background: 'red' });
    expect(css).toContain('background: red');
  });

  it('converts border and borderRadius', () => {
    const css = highlightOptionsToCSS({ border: '1px solid red', borderRadius: '3px' });
    expect(css).toContain('border: 1px solid red');
    expect(css).toContain('border-radius: 3px');
  });

  it('converts textDecoration', () => {
    const css = highlightOptionsToCSS({ textDecoration: 'underline wavy red' });
    expect(css).toContain('text-decoration: underline wavy red');
  });

  it('returns empty string for empty options', () => {
    expect(highlightOptionsToCSS({})).toBe('');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/highlight-options.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement HighlightOptions type and resolver**

Add to `types.ts`:
```typescript
export interface HighlightOptions {
  background?: string;
  border?: string;
  borderRadius?: string;
  textDecoration?: string;
  label?: string;
  group?: string;
}

export type HighlightStyle = 'pulse' | 'underline' | 'glow' | 'box' | 'error' | 'suggestion';
```

Create `highlight-options.ts`:
```typescript
import type { HighlightOptions, HighlightStyle } from './types.js';

const PRESETS: Record<HighlightStyle, HighlightOptions> = {
  pulse: { background: 'rgba(99, 102, 241, 0.3)', borderRadius: '2px' },
  underline: { textDecoration: 'underline wavy rgba(99, 102, 241, 0.7)' },
  glow: { background: 'rgba(99, 102, 241, 0.2)' },
  box: { border: '1px solid rgba(99, 102, 241, 0.5)' },
  error: { background: 'rgba(239, 68, 68, 0.2)', border: '1px solid rgba(239, 68, 68, 0.4)' },
  suggestion: { background: 'rgba(34, 197, 94, 0.15)', border: '1px dashed rgba(34, 197, 94, 0.5)' },
};

export function resolveHighlightStyle(style?: HighlightStyle | HighlightOptions): HighlightOptions {
  if (style === undefined) return { ...PRESETS.pulse };
  if (typeof style === 'string') return { ...PRESETS[style] };
  return style;
}

export function highlightOptionsToCSS(opts: HighlightOptions): string {
  const parts: string[] = [];
  if (opts.background) parts.push(`background: ${opts.background}`);
  if (opts.border) parts.push(`border: ${opts.border}`);
  if (opts.borderRadius) parts.push(`border-radius: ${opts.borderRadius}`);
  if (opts.textDecoration) parts.push(`text-decoration: ${opts.textDecoration}`);
  return parts.join('; ');
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-editor-core/src/highlight-options.test.ts`
Expected: PASS

- [ ] **Step 5: Add exports to index.ts**

Add to `packages/pages-editor-core/src/index.ts`:
```typescript
export type { HighlightOptions } from './types.js';
export { resolveHighlightStyle, highlightOptionsToCSS } from './highlight-options.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-editor-core/src/types.ts packages/pages-editor-core/src/highlight-options.ts packages/pages-editor-core/src/highlight-options.test.ts packages/pages-editor-core/src/index.ts
git commit -m "feat(#535): add HighlightOptions type with preset resolution and CSS conversion"
```

### Task 2: Update EditableTextBridge and MarkdownEditorBridge to accept HighlightOptions

**Files:**
- Modify: `packages/pages-editor-core/src/editable-text-bridge.ts:16,26,41-46,59-61,63-66,68-83,107`
- Modify: `packages/pages-editor-core/src/types.ts:36` (EditableText interface)
- Modify: `packages/pages-markdown-editor/src/markdown-editor-bridge.ts:13,23-28,189-199,280-291`
- Modify: `packages/pages-editor-core/src/editable-text-bridge.test.ts`
- Modify: `packages/pages-editor-core/src/mcp-tool-adapter.test.ts:33` (mock highlight return)

**Interfaces:**
- Consumes: `HighlightOptions`, `HighlightStyle`, `resolveHighlightStyle`, `highlightOptionsToCSS` from Task 1
- Produces: Updated `highlight(from, to, style?)` accepting `HighlightStyle | HighlightOptions`, stored highlights now carry resolved `HighlightOptions`

- [ ] **Step 1: Write failing tests for HighlightOptions in bridge**

Add to `editable-text-bridge.test.ts`:
```typescript
describe('EditableTextBridge — HighlightOptions', () => {
  it('highlight accepts HighlightOptions object', () => {
    const bridge = new TestBridge();
    const id = bridge.highlight(
      { line: 0, col: 0 },
      { line: 0, col: 4 },
      { background: 'red', group: 'errors' },
    );
    expect(typeof id).toBe('string');
    expect(bridge.appliedHighlights[0]!.style).toEqual({ background: 'red', group: 'errors' });
  });

  it('highlight still accepts string style for backward compat', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 }, 'glow');
    expect(bridge.appliedHighlights[0]!.style).toEqual({ background: 'rgba(99, 102, 241, 0.2)' });
  });

  it('highlight with no style defaults to pulse preset', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
    expect(bridge.appliedHighlights[0]!.style!.background).toBeDefined();
  });

  it('activeHighlights stores resolved HighlightOptions', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 }, { border: '2px solid blue' });
    const hl = [...bridge.activeHighlights.values()][0]!;
    expect(hl.style).toEqual({ border: '2px solid blue' });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: FAIL — type mismatch or wrong stored value

- [ ] **Step 3: Update types and bridge to use HighlightOptions**

Update `EditableText` interface in `types.ts`:
```typescript
highlight(from: Position, to: Position, style?: HighlightStyle | HighlightOptions): string;
```

Update `EditableTextBridge`:
- Change `_highlights` map value type from `{ from: Position; to: Position; style: HighlightStyle | undefined }` to `{ from: Position; to: Position; style: HighlightOptions }`
- Change `applyHighlight` abstract signature to accept `HighlightOptions`
- In `highlight()`, call `resolveHighlightStyle(style)` before storing and passing to `applyHighlight`
- Update `activeHighlights` getter return type
- Update `highlightLine`, `highlightBlock`, `highlightRange` signatures to accept `HighlightStyle | HighlightOptions`

Update `MarkdownEditorBridge`:
- Change `_decorations` map value type from `{ from: number; to: number; style: string }` to `{ from: number; to: number; style: HighlightOptions }`
- Remove `_styleForHighlight` method — replace with `highlightOptionsToCSS` call in `_applyDecorations`
- Update `applyHighlight` to accept `HighlightOptions`
- Update `_highlightDocRange` to accept and pass `HighlightOptions`

Update `TestBridge` in test file to match new abstract signature.

- [ ] **Step 4: Run all editor-core and markdown-editor tests**

Run: `yarn vitest run packages/pages-editor-core/ packages/pages-markdown-editor/`
Expected: PASS (all existing tests + new ones)

- [ ] **Step 5: Update MCP tool adapter to accept HighlightOptions**

In `mcp-tool-adapter.ts`, the `editor_highlight` case already passes `params.style` through. Since `HighlightOptions` is a superset, this works without changes to the adapter code. Update the `MCP_TOOL_DEFINITIONS` for `editor_highlight` to document the new `style` parameter as accepting either a preset name or a `HighlightOptions` object:

```typescript
style: {
  oneOf: [
    { type: 'string', enum: ['pulse', 'underline', 'glow', 'box', 'error', 'suggestion'] },
    {
      type: 'object',
      properties: {
        background: { type: 'string' },
        border: { type: 'string' },
        borderRadius: { type: 'string' },
        textDecoration: { type: 'string' },
        label: { type: 'string' },
        group: { type: 'string' },
      },
    },
  ],
  description: 'Highlight style — a preset name or a HighlightOptions object',
},
```

Update mock in `mcp-tool-adapter.test.ts` so `highlight` mock accepts the new type.

- [ ] **Step 6: Run full test suite**

Run: `yarn vitest run packages/pages-editor-core/ packages/pages-markdown-editor/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-editor-core/src/ packages/pages-markdown-editor/src/markdown-editor-bridge.ts
git commit -m "feat(#535): update highlight API to accept configurable HighlightOptions"
```

---

## Batch 2: Semantic Targeting and Read-back API

### Task 3: highlightSentence, highlightText, and extended highlightLine

**Files:**
- Modify: `packages/pages-editor-core/src/editable-text-bridge.ts` (add `highlightSentence`, `highlightText`, extend `highlightLine`)
- Modify: `packages/pages-editor-core/src/types.ts` (add methods to `EditableText` interface)
- Modify: `packages/pages-editor-core/src/editable-text-bridge.test.ts`
- Modify: `packages/pages-editor-core/src/compliance.ts` (add compliance tests for new methods)

**Interfaces:**
- Consumes: `highlight()` from Tasks 1-2, `HighlightStyle | HighlightOptions`
- Produces: `highlightSentence(pos?: Position, style?: HighlightStyle | HighlightOptions): string`, `highlightText(query: string, style?: HighlightStyle | HighlightOptions): string[]`, `highlightLine(line: number, count?: number, style?: HighlightStyle | HighlightOptions): string`

- [ ] **Step 1: Write failing tests for highlightSentence**

Add to `editable-text-bridge.test.ts`:
```typescript
describe('EditableTextBridge — highlightSentence', () => {
  const TEXT = 'Hello world. This is a test. Another sentence here.';

  class SentenceTestBridge extends TestBridge {
    override getText() { return TEXT; }
    override getLine(line: number) { return TEXT.split('\n')[line] ?? ''; }
    override getLineCount() { return 1; }
    override getCursor() { return { line: 0, col: 14 }; }
  }

  it('highlights sentence at cursor position', () => {
    const bridge = new SentenceTestBridge();
    const id = bridge.highlightSentence();
    expect(typeof id).toBe('string');
    const hl = bridge.appliedHighlights[0]!;
    expect(hl.from).toEqual({ line: 0, col: 13 });
    expect(hl.to).toEqual({ line: 0, col: 28 });
  });

  it('highlights sentence at explicit position', () => {
    const bridge = new SentenceTestBridge();
    bridge.highlightSentence({ line: 0, col: 0 });
    const hl = bridge.appliedHighlights[0]!;
    expect(hl.from).toEqual({ line: 0, col: 0 });
    expect(hl.to).toEqual({ line: 0, col: 12 });
  });

  it('highlights last sentence when pos is near end', () => {
    const bridge = new SentenceTestBridge();
    bridge.highlightSentence({ line: 0, col: 35 });
    const hl = bridge.appliedHighlights[0]!;
    expect(hl.from).toEqual({ line: 0, col: 29 });
    expect(hl.to).toEqual({ line: 0, col: 51 });
  });

  it('accepts style parameter', () => {
    const bridge = new SentenceTestBridge();
    bridge.highlightSentence(undefined, 'error');
    expect(bridge.appliedHighlights[0]!.style).toBeDefined();
  });
});
```

- [ ] **Step 2: Write failing tests for highlightText**

```typescript
describe('EditableTextBridge — highlightText', () => {
  class MultiMatchBridge extends TestBridge {
    override getText() { return 'foo bar foo baz foo'; }
    override getLine(line: number) { return this.getText().split('\n')[line] ?? ''; }
    override getLineCount() { return 1; }
  }

  it('highlights all occurrences of a string', () => {
    const bridge = new MultiMatchBridge();
    const ids = bridge.highlightText('foo');
    expect(ids).toHaveLength(3);
    expect(bridge.appliedHighlights).toHaveLength(3);
    expect(bridge.appliedHighlights[0]!.from).toEqual({ line: 0, col: 0 });
    expect(bridge.appliedHighlights[0]!.to).toEqual({ line: 0, col: 3 });
    expect(bridge.appliedHighlights[1]!.from).toEqual({ line: 0, col: 8 });
    expect(bridge.appliedHighlights[2]!.from).toEqual({ line: 0, col: 16 });
  });

  it('returns empty array when no matches', () => {
    const bridge = new MultiMatchBridge();
    const ids = bridge.highlightText('zzz');
    expect(ids).toHaveLength(0);
  });

  it('accepts style parameter', () => {
    const bridge = new MultiMatchBridge();
    bridge.highlightText('foo', { background: 'yellow', group: 'search' });
    expect(bridge.appliedHighlights[0]!.style).toEqual({ background: 'yellow', group: 'search' });
  });
});
```

- [ ] **Step 3: Write failing tests for extended highlightLine with count**

```typescript
describe('EditableTextBridge — highlightLine with count', () => {
  it('highlightLine with count=3 highlights 3 consecutive lines', () => {
    const bridge = new BlockTestBridge();
    const id = bridge.highlightLine(4, 3);
    expect(typeof id).toBe('string');
    const hl = bridge.appliedHighlights[0]!;
    expect(hl.from).toEqual({ line: 4, col: 0 });
    expect(hl.to.line).toBe(6);
  });

  it('highlightLine with count=1 behaves like original', () => {
    const bridge = new BlockTestBridge();
    bridge.highlightLine(0, 1);
    const hl = bridge.appliedHighlights[0]!;
    expect(hl.from).toEqual({ line: 0, col: 0 });
    expect(hl.to).toEqual({ line: 0, col: 7 });
  });

  it('highlightLine clamps count to available lines', () => {
    const bridge = new BlockTestBridge();
    bridge.highlightLine(10, 100);
    const hl = bridge.appliedHighlights[0]!;
    expect(hl.from).toEqual({ line: 10, col: 0 });
    expect(hl.to.line).toBe(11);
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: FAIL — methods don't exist

- [ ] **Step 5: Implement the three methods**

Add to `EditableText` interface in `types.ts`:
```typescript
highlightSentence(pos?: Position, style?: HighlightStyle | HighlightOptions): string;
highlightText(query: string, style?: HighlightStyle | HighlightOptions): string[];
```

Update `highlightLine` signature in `EditableText`:
```typescript
highlightLine(line: number, count?: number, style?: HighlightStyle | HighlightOptions): string;
```

Implement in `EditableTextBridge`:

```typescript
highlightSentence(pos?: Position, style?: HighlightStyle | HighlightOptions): string {
  const cursor = pos ?? this.getCursor();
  const lineText = this.getLine(cursor.line);
  const sentenceEnds = /[.!?]\s+(?=[A-Z])|[.!?]$/g;
  const boundaries = [0];
  let match;
  while ((match = sentenceEnds.exec(lineText)) !== null) {
    boundaries.push(match.index + match[0].length);
  }
  if (boundaries[boundaries.length - 1] !== lineText.length) {
    boundaries.push(lineText.length);
  }
  let sentenceStart = 0;
  let sentenceEnd = lineText.length;
  for (let i = 0; i < boundaries.length - 1; i++) {
    if (cursor.col >= boundaries[i]! && cursor.col < boundaries[i + 1]!) {
      sentenceStart = boundaries[i]!;
      sentenceEnd = boundaries[i + 1]!;
      break;
    }
  }
  return this.highlight(
    { line: cursor.line, col: sentenceStart },
    { line: cursor.line, col: sentenceEnd },
    style,
  );
}

highlightText(query: string, style?: HighlightStyle | HighlightOptions): string[] {
  const positions = this.findText(query);
  return positions.map(pos =>
    this.highlight(pos, { line: pos.line, col: pos.col + query.length }, style),
  );
}

highlightLine(line: number, count?: number, style?: HighlightStyle | HighlightOptions): string {
  const n = count ?? 1;
  const endLine = Math.min(line + n - 1, this.getLineCount() - 1);
  const endText = this.getLine(endLine);
  return this.highlight(
    { line, col: 0 },
    { line: endLine, col: endText.length },
    style,
  );
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: PASS

- [ ] **Step 7: Add compliance tests for new methods**

Add to `compliance.ts`:
```typescript
it('highlightSentence returns a string ID', () => {
  const et = factory();
  const id = et.highlightSentence();
  expect(typeof id).toBe('string');
});

it('highlightText returns an array of string IDs', () => {
  const et = factory();
  const text = et.getText();
  if (text.length > 0) {
    const ids = et.highlightText(text.substring(0, 1));
    expect(Array.isArray(ids)).toBe(true);
    for (const id of ids) {
      expect(typeof id).toBe('string');
    }
  }
});
```

- [ ] **Step 8: Run full test suite**

Run: `yarn vitest run packages/pages-editor-core/`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-editor-core/src/
git commit -m "feat(#535): add highlightSentence, highlightText, and multi-line highlightLine"
```

### Task 4: Read-back API — getHighlightText, listHighlights, clearHighlightGroup

**Files:**
- Modify: `packages/pages-editor-core/src/editable-text-bridge.ts` (add methods)
- Modify: `packages/pages-editor-core/src/types.ts` (add to `EditableText` interface)
- Modify: `packages/pages-editor-core/src/editable-text-bridge.test.ts`

**Interfaces:**
- Consumes: `_highlights` Map, `getText()`, `HighlightOptions` (with `group` field) from Tasks 1-2
- Produces: `getHighlightText(id: string): string | undefined`, `listHighlights(): Array<{ id: string; from: Position; to: Position; text: string; group?: string }>`, `clearHighlightGroup(group: string): void`

- [ ] **Step 1: Write failing tests**

Add to `editable-text-bridge.test.ts`:
```typescript
describe('EditableTextBridge — read-back API', () => {
  it('getHighlightText returns text under a highlight', () => {
    const bridge = new TestBridge();
    const id = bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 });
    expect(bridge.getHighlightText(id)).toBe('line');
  });

  it('getHighlightText returns undefined for unknown ID', () => {
    const bridge = new TestBridge();
    expect(bridge.getHighlightText('nonexistent')).toBeUndefined();
  });

  it('getHighlightText works across lines', () => {
    const bridge = new TestBridge();
    const id = bridge.highlight({ line: 0, col: 5 }, { line: 1, col: 4 });
    const text = bridge.getHighlightText(id);
    expect(text).toBe('one\nline');
  });

  it('listHighlights returns all highlights with metadata', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 }, { group: 'errors' });
    bridge.highlight({ line: 1, col: 0 }, { line: 1, col: 4 });
    const list = bridge.listHighlights();
    expect(list).toHaveLength(2);
    expect(list[0]!.text).toBe('line');
    expect(list[0]!.group).toBe('errors');
    expect(list[1]!.group).toBeUndefined();
  });

  it('clearHighlightGroup removes only matching group', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 }, { group: 'errors' });
    bridge.highlight({ line: 1, col: 0 }, { line: 1, col: 4 }, { group: 'search' });
    bridge.highlight({ line: 2, col: 0 }, { line: 2, col: 4 });
    bridge.clearHighlightGroup('errors');
    expect(bridge.activeHighlights.size).toBe(2);
    expect(bridge.listHighlights().every(h => h.group !== 'errors')).toBe(true);
  });

  it('clearHighlightGroup calls removeHighlightDecoration for each', () => {
    const bridge = new TestBridge();
    bridge.highlight({ line: 0, col: 0 }, { line: 0, col: 4 }, { group: 'g' });
    bridge.highlight({ line: 1, col: 0 }, { line: 1, col: 4 }, { group: 'g' });
    bridge.clearHighlightGroup('g');
    expect(bridge.removedHighlights).toHaveLength(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: FAIL — methods don't exist

- [ ] **Step 3: Implement read-back methods**

Add to `EditableText` interface in `types.ts`:
```typescript
getHighlightText(id: string): string | undefined;
listHighlights(): Array<{ id: string; from: Position; to: Position; text: string; group?: string }>;
clearHighlightGroup(group: string): void;
```

Implement in `EditableTextBridge`:
```typescript
getHighlightText(id: string): string | undefined {
  const hl = this._highlights.get(id);
  if (!hl) return undefined;
  const text = this.getText();
  const lines = text.split('\n');
  if (hl.from.line === hl.to.line) {
    return lines[hl.from.line]?.substring(hl.from.col, hl.to.col);
  }
  const parts: string[] = [];
  parts.push(lines[hl.from.line]?.substring(hl.from.col) ?? '');
  for (let i = hl.from.line + 1; i < hl.to.line; i++) {
    parts.push(lines[i] ?? '');
  }
  parts.push(lines[hl.to.line]?.substring(0, hl.to.col) ?? '');
  return parts.join('\n');
}

listHighlights(): Array<{ id: string; from: Position; to: Position; text: string; group?: string }> {
  return [...this._highlights.entries()].map(([id, hl]) => ({
    id,
    from: hl.from,
    to: hl.to,
    text: this.getHighlightText(id) ?? '',
    group: hl.style.group,
  }));
}

clearHighlightGroup(group: string): void {
  for (const [id, hl] of [...this._highlights.entries()]) {
    if (hl.style.group === group) {
      this._highlights.delete(id);
      this.removeHighlightDecoration(id);
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-editor-core/src/
git commit -m "feat(#535): add read-back API — getHighlightText, listHighlights, clearHighlightGroup"
```

### Task 5: MCP tools for new methods

**Files:**
- Modify: `packages/pages-editor-core/src/mcp-tool-adapter.ts` (add cases)
- Modify: `packages/pages-editor-core/src/mcp-tool-definitions.ts` (add tool schemas)
- Modify: `packages/pages-editor-core/src/mcp-tool-adapter.test.ts` (add tests)

**Interfaces:**
- Consumes: `highlightSentence`, `highlightText`, `highlightLine` (with count), `getHighlightText`, `listHighlights`, `clearHighlightGroup` from Tasks 3-4
- Produces: MCP tools: `editor_highlight_sentence`, `editor_highlight_text`, `editor_highlight_line`, `editor_get_highlight_text`, `editor_list_highlights`, `editor_clear_highlight_group`

- [ ] **Step 1: Write failing tests for new MCP tools**

Add to `mcp-tool-adapter.test.ts`:
```typescript
describe('semantic highlight tools', () => {
  it('editor_highlight_sentence calls highlightSentence()', async () => {
    const mock = createMockEditableText('Hello world. Another sentence.');
    mock.highlightSentence = vi.fn(() => 'hl-s1');
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_highlight_sentence', {});
    expect(result).toEqual({ id: 'hl-s1' });
    expect(mock.highlightSentence).toHaveBeenCalled();
  });

  it('editor_highlight_text calls highlightText()', async () => {
    const mock = createMockEditableText('hello hello');
    mock.highlightText = vi.fn(() => ['hl-1', 'hl-2']);
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_highlight_text', { query: 'hello', style: 'error' });
    expect(result).toEqual({ ids: ['hl-1', 'hl-2'] });
    expect(mock.highlightText).toHaveBeenCalledWith('hello', 'error');
  });

  it('editor_highlight_line calls highlightLine() with count', async () => {
    const mock = createMockEditableText('a\nb\nc');
    mock.highlightLine = vi.fn(() => 'hl-l1');
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_highlight_line', { line: 0, count: 2 });
    expect(result).toEqual({ id: 'hl-l1' });
    expect(mock.highlightLine).toHaveBeenCalledWith(0, 2, undefined);
  });
});

describe('read-back tools', () => {
  it('editor_get_highlight_text returns text', async () => {
    const mock = createMockEditableText('hello world');
    mock.getHighlightText = vi.fn(() => 'hello');
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_get_highlight_text', { id: 'hl-1' });
    expect(result).toEqual({ text: 'hello' });
  });

  it('editor_list_highlights returns highlight list', async () => {
    const mock = createMockEditableText('hello');
    const list = [{ id: 'hl-1', from: { line: 0, col: 0 }, to: { line: 0, col: 5 }, text: 'hello', group: 'g' }];
    mock.listHighlights = vi.fn(() => list);
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_list_highlights', {});
    expect(result).toEqual({ highlights: list });
  });

  it('editor_clear_highlight_group calls clearHighlightGroup()', async () => {
    const mock = createMockEditableText('');
    mock.clearHighlightGroup = vi.fn();
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_clear_highlight_group', { group: 'errors' });
    expect(result).toEqual({ success: true });
    expect(mock.clearHighlightGroup).toHaveBeenCalledWith('errors');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/mcp-tool-adapter.test.ts`
Expected: FAIL — unknown_tool errors

- [ ] **Step 3: Add cases to McpToolAdapter.handleToolCall**

Add to the switch in `mcp-tool-adapter.ts`:
```typescript
case 'editor_highlight_sentence': {
  const id = this.editor.highlightSentence(
    params.pos as Position | undefined,
    params.style,
  );
  return { id };
}

case 'editor_highlight_text': {
  const ids = this.editor.highlightText(params.query, params.style);
  return { ids };
}

case 'editor_highlight_line': {
  const id = this.editor.highlightLine(params.line, params.count, params.style);
  return { id };
}

case 'editor_get_highlight_text': {
  const text = this.editor.getHighlightText(params.id);
  if (text === undefined) return { error: 'not_found', id: params.id };
  return { text };
}

case 'editor_list_highlights':
  return { highlights: this.editor.listHighlights() };

case 'editor_clear_highlight_group':
  this.editor.clearHighlightGroup(params.group);
  return { success: true };
```

- [ ] **Step 4: Add MCP tool definitions**

Add to `MCP_TOOL_DEFINITIONS` array in `mcp-tool-definitions.ts`:
```typescript
{
  name: 'editor_highlight_sentence',
  description: 'Highlight the sentence at the cursor or a given position. Returns a highlight ID.',
  inputSchema: {
    type: 'object',
    properties: {
      pos: { ...positionSchema, description: 'Position within the sentence (default: cursor)' },
      style: highlightStyleSchema,
    },
  },
},
{
  name: 'editor_highlight_text',
  description: 'Find and highlight ALL occurrences of a string. Returns an array of highlight IDs.',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Text to find and highlight' },
      style: highlightStyleSchema,
    },
    required: ['query'],
  },
},
{
  name: 'editor_highlight_line',
  description: 'Highlight one or more consecutive lines. Returns a highlight ID.',
  inputSchema: {
    type: 'object',
    properties: {
      line: { type: 'number', description: 'Zero-based starting line number' },
      count: { type: 'number', description: 'Number of lines to highlight (default: 1)' },
      style: highlightStyleSchema,
    },
    required: ['line'],
  },
},
{
  name: 'editor_get_highlight_text',
  description: 'Get the text content under a highlight by ID.',
  inputSchema: {
    type: 'object',
    properties: { id: { type: 'string', description: 'Highlight ID' } },
    required: ['id'],
  },
},
{
  name: 'editor_list_highlights',
  description: 'List all active highlights with IDs, ranges, text content, and group.',
  inputSchema: { type: 'object', properties: {} },
},
{
  name: 'editor_clear_highlight_group',
  description: 'Remove all highlights belonging to a named group.',
  inputSchema: {
    type: 'object',
    properties: { group: { type: 'string', description: 'Group name to clear' } },
    required: ['group'],
  },
},
```

Extract the shared `highlightStyleSchema` at the top of the file:
```typescript
const highlightStyleSchema = {
  oneOf: [
    { type: 'string', enum: ['pulse', 'underline', 'glow', 'box', 'error', 'suggestion'] },
    {
      type: 'object',
      properties: {
        background: { type: 'string' },
        border: { type: 'string' },
        borderRadius: { type: 'string' },
        textDecoration: { type: 'string' },
        label: { type: 'string' },
        group: { type: 'string' },
      },
    },
  ],
  description: 'Highlight style — a preset name or custom options',
};
```

Also update the existing `editor_highlight` definition to use `highlightStyleSchema`.

- [ ] **Step 5: Update mock in test to include new methods**

Add to `createMockEditableText()` return:
```typescript
highlightSentence: vi.fn(() => 'hl-s1'),
highlightText: vi.fn(() => []),
highlightLine: vi.fn(() => 'hl-l1'),
getHighlightText: vi.fn(() => undefined),
listHighlights: vi.fn(() => []),
clearHighlightGroup: vi.fn(),
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-editor-core/src/mcp-tool-adapter.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-editor-core/src/
git commit -m "feat(#535): add MCP tools for semantic targeting and read-back API"
```

---

## Batch 3: Animated Overlay Rendering and Line Reader

### Task 6: HighlightRenderer — overlay-based highlight rendering

**Files:**
- Create: `packages/pages-markdown-editor/src/overlay/highlight-renderer.ts`
- Create: `packages/pages-markdown-editor/src/overlay/highlight-renderer.test.ts`
- Modify: `packages/pages-markdown-editor/src/markdown-editor-bridge.ts` (switch from Decoration.inline to HighlightRenderer)

**Interfaces:**
- Consumes: `HighlightOptions`, `highlightOptionsToCSS` from Task 1; `coordsAtPos` pattern from ProseMirror `EditorView`
- Produces: `HighlightRenderer` class with `add(id, from, to, options)`, `remove(id)`, `clear()`, `reposition()`, `startTracking()`, `stopTracking()`, `destroy()`

- [ ] **Step 1: Write failing tests for HighlightRenderer**

```typescript
// highlight-renderer.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { HighlightRenderer } from './highlight-renderer.js';

function createMockOverlay(): HTMLElement {
  const overlay = document.createElement('div');
  overlay.getBoundingClientRect = vi.fn(() => ({
    left: 0, top: 0, right: 800, bottom: 600, width: 800, height: 600, x: 0, y: 0, toJSON: () => {},
  }));
  return overlay;
}

type RectProvider = (from: number, to: number) => Array<{ left: number; top: number; right: number; bottom: number }>;

function createMockRectProvider(): RectProvider {
  return vi.fn((from: number, _to: number) => [{
    left: from * 8, top: 0, right: (from + 1) * 8, bottom: 20,
  }]);
}

describe('HighlightRenderer', () => {
  let overlay: HTMLElement;
  let rectProvider: RectProvider;
  let renderer: HighlightRenderer;

  beforeEach(() => {
    overlay = createMockOverlay();
    rectProvider = createMockRectProvider();
    renderer = new HighlightRenderer(overlay, rectProvider);
  });

  it('add creates an overlay element', () => {
    renderer.add('hl-1', 0, 10, { background: 'red' });
    expect(overlay.children).toHaveLength(1);
    const el = overlay.children[0] as HTMLElement;
    expect(el.style.position).toBe('absolute');
    expect(el.style.background).toBe('red');
  });

  it('add sets CSS transition for animation', () => {
    renderer.add('hl-1', 0, 10, { background: 'red' });
    const el = overlay.children[0] as HTMLElement;
    expect(el.style.transition).toContain('top');
    expect(el.style.transition).toContain('left');
  });

  it('add sets label as title attribute', () => {
    renderer.add('hl-1', 0, 10, { background: 'red', label: 'Error here' });
    const el = overlay.children[0] as HTMLElement;
    expect(el.title).toBe('Error here');
  });

  it('remove removes the overlay element', () => {
    renderer.add('hl-1', 0, 10, { background: 'red' });
    renderer.remove('hl-1');
    expect(overlay.children).toHaveLength(0);
  });

  it('clear removes all overlay elements', () => {
    renderer.add('hl-1', 0, 10, { background: 'red' });
    renderer.add('hl-2', 20, 30, { background: 'blue' });
    renderer.clear();
    expect(overlay.children).toHaveLength(0);
  });

  it('reposition updates element positions', () => {
    renderer.add('hl-1', 0, 10, { background: 'red' });
    const el = overlay.children[0] as HTMLElement;
    const initialLeft = el.style.left;
    (rectProvider as any).mockReturnValue([{ left: 100, top: 50, right: 180, bottom: 70 }]);
    renderer.reposition();
    expect(el.style.left).not.toBe(initialLeft);
  });

  it('destroy cleans up everything', () => {
    renderer.add('hl-1', 0, 10, { background: 'red' });
    renderer.startTracking();
    renderer.destroy();
    expect(overlay.children).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-markdown-editor/src/overlay/highlight-renderer.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement HighlightRenderer**

Create `packages/pages-markdown-editor/src/overlay/highlight-renderer.ts`:
```typescript
import type { HighlightOptions } from '@casehubio/pages-editor-core';
import { highlightOptionsToCSS } from '@casehubio/pages-editor-core';

type RectProvider = (from: number, to: number) => Array<{ left: number; top: number; right: number; bottom: number }>;

interface RenderedHighlight {
  id: string;
  from: number;
  to: number;
  options: HighlightOptions;
  elements: HTMLElement[];
}

export class HighlightRenderer {
  private _highlights = new Map<string, RenderedHighlight>();
  private _rafId: number | null = null;

  constructor(
    private readonly overlay: HTMLElement,
    private readonly rectsFor: RectProvider,
  ) {}

  add(id: string, from: number, to: number, options: HighlightOptions): void {
    this.remove(id);
    const rects = this.rectsFor(from, to);
    const overlayRect = this.overlay.getBoundingClientRect();
    const elements = rects.map(rect => {
      const el = document.createElement('div');
      el.className = 'highlight-overlay';
      el.dataset['highlightId'] = id;
      el.style.position = 'absolute';
      el.style.pointerEvents = 'none';
      el.style.transition = 'top 0.3s ease, left 0.3s ease, width 0.3s ease, height 0.3s ease';
      el.style.left = `${rect.left - overlayRect.left}px`;
      el.style.top = `${rect.top - overlayRect.top}px`;
      el.style.width = `${rect.right - rect.left}px`;
      el.style.height = `${rect.bottom - rect.top}px`;
      const css = highlightOptionsToCSS(options);
      if (css) el.style.cssText += `; ${css}`;
      el.style.position = 'absolute';
      el.style.transition = 'top 0.3s ease, left 0.3s ease, width 0.3s ease, height 0.3s ease';
      if (options.label) el.title = options.label;
      this.overlay.appendChild(el);
      return el;
    });
    this._highlights.set(id, { id, from, to, options, elements });
  }

  remove(id: string): void {
    const hl = this._highlights.get(id);
    if (hl) {
      for (const el of hl.elements) el.remove();
      this._highlights.delete(id);
    }
  }

  clear(): void {
    for (const hl of this._highlights.values()) {
      for (const el of hl.elements) el.remove();
    }
    this._highlights.clear();
  }

  reposition(): void {
    const overlayRect = this.overlay.getBoundingClientRect();
    for (const hl of this._highlights.values()) {
      const rects = this.rectsFor(hl.from, hl.to);
      for (let i = 0; i < hl.elements.length; i++) {
        const rect = rects[i];
        const el = hl.elements[i];
        if (rect && el) {
          el.style.left = `${rect.left - overlayRect.left}px`;
          el.style.top = `${rect.top - overlayRect.top}px`;
          el.style.width = `${rect.right - rect.left}px`;
          el.style.height = `${rect.bottom - rect.top}px`;
        }
      }
    }
  }

  startTracking(): void {
    if (this._rafId !== null) return;
    const tick = () => {
      this.reposition();
      this._rafId = requestAnimationFrame(tick);
    };
    this._rafId = requestAnimationFrame(tick);
  }

  stopTracking(): void {
    if (this._rafId !== null) {
      cancelAnimationFrame(this._rafId);
      this._rafId = null;
    }
  }

  destroy(): void {
    this.stopTracking();
    this.clear();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-markdown-editor/src/overlay/highlight-renderer.test.ts`
Expected: PASS

- [ ] **Step 5: Wire HighlightRenderer into MarkdownEditorBridge**

Replace `_decorations` map and `_applyDecorations()` / `Decoration.inline` approach with `HighlightRenderer`:
- Add `private _highlightRenderer: HighlightRenderer | null = null` field
- Add `initOverlay(overlay: HTMLElement): void` method that creates the renderer using `(from, to) => this.view.coordsAtPos(...)` as the rect provider
- In `applyHighlight`, delegate to `_highlightRenderer.add()` if initialized, fall back to `_applyDecorations()` if no overlay
- In `removeHighlightDecoration`, delegate to `_highlightRenderer.remove()`
- In `clearHighlightDecorations`, delegate to `_highlightRenderer.clear()`
- Keep `_applyDecorations()` as fallback for when no overlay element is provided (e.g., in tests)

- [ ] **Step 6: Run full markdown-editor test suite**

Run: `yarn vitest run packages/pages-markdown-editor/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-markdown-editor/src/
git commit -m "feat(#535): add HighlightRenderer with CSS-transition animated overlays"
```

### Task 7: LineReader — tracked highlight with advance/moveTo

**Files:**
- Create: `packages/pages-editor-core/src/line-reader.ts`
- Create: `packages/pages-editor-core/src/line-reader.test.ts`
- Modify: `packages/pages-editor-core/src/editable-text-bridge.ts` (add `createReader`)
- Modify: `packages/pages-editor-core/src/types.ts` (add `LineReader` interface and `createReader` to `EditableText`)
- Modify: `packages/pages-editor-core/src/index.ts` (export)

**Interfaces:**
- Consumes: `EditableText.highlight()`, `EditableText.removeHighlight()`, `EditableText.highlightSentence()` from Tasks 2-3
- Produces: `LineReader` interface with `moveTo(pos: Position): void`, `advance(): void`, `position(): Position`, `dispose(): void`; `EditableText.createReader(style?: HighlightStyle | HighlightOptions): LineReader`

- [ ] **Step 1: Write failing tests for LineReader**

```typescript
// line-reader.test.ts
import { describe, it, expect } from 'vitest';
import type { Position, HighlightStyle, HighlightOptions } from './types.js';
import { EditableTextBridge } from './editable-text-bridge.js';

const TEXT = 'Hello world. This is a test. Another sentence here.';

class ReaderTestBridge extends EditableTextBridge {
  appliedHighlights: Array<{ id: string; from: Position; to: Position; style: HighlightOptions }> = [];
  removedHighlights: string[] = [];

  getText() { return TEXT; }
  getLine(line: number) { return TEXT.split('\n')[line] ?? ''; }
  getLineCount() { return 1; }
  setContent() {}
  findText(query: string) {
    const results: Position[] = [];
    let idx = 0;
    while ((idx = TEXT.indexOf(query, idx)) !== -1) {
      results.push({ line: 0, col: idx });
      idx += query.length;
    }
    return results;
  }
  insertText() {}
  replaceRange() {}
  deleteRange() {}
  setCursor() {}
  getCursor() { return { line: 0, col: 0 }; }

  protected override applyHighlight(id: string, from: Position, to: Position, style: HighlightOptions) {
    this.appliedHighlights.push({ id, from, to, style });
  }
  protected override removeHighlightDecoration(id: string) {
    this.removedHighlights.push(id);
  }
  protected override clearHighlightDecorations() {}
}

describe('LineReader', () => {
  it('createReader returns a reader with initial position', () => {
    const bridge = new ReaderTestBridge();
    const reader = bridge.createReader();
    expect(reader.position()).toEqual({ line: 0, col: 0 });
  });

  it('createReader creates a highlight', () => {
    const bridge = new ReaderTestBridge();
    bridge.createReader();
    expect(bridge.appliedHighlights.length).toBeGreaterThan(0);
  });

  it('moveTo updates highlight position', () => {
    const bridge = new ReaderTestBridge();
    const reader = bridge.createReader();
    const initialCount = bridge.appliedHighlights.length;
    reader.moveTo({ line: 0, col: 13 });
    expect(reader.position()).toEqual({ line: 0, col: 13 });
    expect(bridge.appliedHighlights.length).toBeGreaterThan(initialCount);
  });

  it('advance moves to next sentence', () => {
    const bridge = new ReaderTestBridge();
    const reader = bridge.createReader();
    reader.advance();
    const pos = reader.position();
    expect(pos.col).toBeGreaterThan(0);
  });

  it('dispose removes the highlight', () => {
    const bridge = new ReaderTestBridge();
    const reader = bridge.createReader();
    reader.dispose();
    expect(bridge.removedHighlights.length).toBeGreaterThan(0);
  });

  it('createReader accepts custom style', () => {
    const bridge = new ReaderTestBridge();
    bridge.createReader({ background: 'rgba(0,255,0,0.2)' });
    expect(bridge.appliedHighlights[0]!.style.background).toBe('rgba(0,255,0,0.2)');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/line-reader.test.ts`
Expected: FAIL — method doesn't exist

- [ ] **Step 3: Implement LineReader**

Add to `types.ts`:
```typescript
export interface LineReader {
  moveTo(pos: Position): void;
  advance(): void;
  position(): Position;
  dispose(): void;
}
```

Add `createReader` to `EditableText` interface:
```typescript
createReader(style?: HighlightStyle | HighlightOptions): LineReader;
```

Create `line-reader.ts`:
```typescript
import type { EditableText, LineReader, Position, HighlightStyle, HighlightOptions } from './types.js';

export function createLineReader(
  editor: EditableText,
  style?: HighlightStyle | HighlightOptions,
): LineReader {
  let currentPos: Position = { line: 0, col: 0 };
  let highlightId: string | null = null;

  function updateHighlight(): void {
    if (highlightId) editor.removeHighlight(highlightId);
    highlightId = editor.highlightSentence(currentPos, style);
  }

  updateHighlight();

  return {
    moveTo(pos: Position): void {
      currentPos = pos;
      updateHighlight();
    },

    advance(): void {
      const text = editor.getLine(currentPos.line);
      const sentenceEnd = /[.!?]\s+/g;
      sentenceEnd.lastIndex = currentPos.col;
      const match = sentenceEnd.exec(text);
      if (match) {
        currentPos = { line: currentPos.line, col: match.index + match[0].length };
      } else if (currentPos.line < editor.getLineCount() - 1) {
        currentPos = { line: currentPos.line + 1, col: 0 };
      }
      updateHighlight();
    },

    position(): Position {
      return { ...currentPos };
    },

    dispose(): void {
      if (highlightId) {
        editor.removeHighlight(highlightId);
        highlightId = null;
      }
    },
  };
}
```

Add `createReader` to `EditableTextBridge`:
```typescript
createReader(style?: HighlightStyle | HighlightOptions): LineReader {
  return createLineReader(this, style);
}
```

Import `createLineReader` and `LineReader` in `editable-text-bridge.ts`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-editor-core/src/line-reader.test.ts`
Expected: PASS

- [ ] **Step 5: Add exports to index.ts**

```typescript
export type { LineReader } from './types.js';
export { createLineReader } from './line-reader.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-editor-core/src/
git commit -m "feat(#535): add LineReader with sentence-level advance and moveTo"
```

### Task 8: MCP tools for LineReader

**Files:**
- Modify: `packages/pages-editor-core/src/mcp-tool-adapter.ts` (add reader lifecycle)
- Modify: `packages/pages-editor-core/src/mcp-tool-definitions.ts` (add tool schemas)
- Modify: `packages/pages-editor-core/src/mcp-tool-adapter.test.ts`

**Interfaces:**
- Consumes: `EditableText.createReader()`, `LineReader` from Task 7
- Produces: MCP tools: `editor_reader_start`, `editor_reader_move`, `editor_reader_advance`, `editor_reader_stop`

- [ ] **Step 1: Write failing tests for reader MCP tools**

Add to `mcp-tool-adapter.test.ts`:
```typescript
describe('reader tools', () => {
  it('editor_reader_start creates a reader', async () => {
    const mock = createMockEditableText('Hello world. Second sentence.');
    const mockReader = {
      moveTo: vi.fn(),
      advance: vi.fn(),
      position: vi.fn(() => ({ line: 0, col: 0 })),
      dispose: vi.fn(),
    };
    mock.createReader = vi.fn(() => mockReader);
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_reader_start', { style: 'pulse' });
    expect(result.position).toEqual({ line: 0, col: 0 });
    expect(mock.createReader).toHaveBeenCalledWith('pulse');
  });

  it('editor_reader_move moves the reader', async () => {
    const mock = createMockEditableText('Hello world.');
    const mockReader = {
      moveTo: vi.fn(),
      advance: vi.fn(),
      position: vi.fn(() => ({ line: 0, col: 5 })),
      dispose: vi.fn(),
    };
    mock.createReader = vi.fn(() => mockReader);
    const adapter = new McpToolAdapter(mock);
    await adapter.handleToolCall('editor_reader_start', {});
    const result = await adapter.handleToolCall('editor_reader_move', { pos: { line: 0, col: 5 } });
    expect(result.position).toEqual({ line: 0, col: 5 });
    expect(mockReader.moveTo).toHaveBeenCalledWith({ line: 0, col: 5 });
  });

  it('editor_reader_advance advances the reader', async () => {
    const mock = createMockEditableText('Hello. World.');
    const mockReader = {
      moveTo: vi.fn(),
      advance: vi.fn(),
      position: vi.fn(() => ({ line: 0, col: 7 })),
      dispose: vi.fn(),
    };
    mock.createReader = vi.fn(() => mockReader);
    const adapter = new McpToolAdapter(mock);
    await adapter.handleToolCall('editor_reader_start', {});
    const result = await adapter.handleToolCall('editor_reader_advance', {});
    expect(result.position).toEqual({ line: 0, col: 7 });
    expect(mockReader.advance).toHaveBeenCalled();
  });

  it('editor_reader_stop disposes the reader', async () => {
    const mock = createMockEditableText('Hello.');
    const mockReader = {
      moveTo: vi.fn(),
      advance: vi.fn(),
      position: vi.fn(() => ({ line: 0, col: 0 })),
      dispose: vi.fn(),
    };
    mock.createReader = vi.fn(() => mockReader);
    const adapter = new McpToolAdapter(mock);
    await adapter.handleToolCall('editor_reader_start', {});
    const result = await adapter.handleToolCall('editor_reader_stop', {});
    expect(result).toEqual({ success: true });
    expect(mockReader.dispose).toHaveBeenCalled();
  });

  it('editor_reader_move returns error when no reader active', async () => {
    const mock = createMockEditableText('');
    const adapter = new McpToolAdapter(mock);
    const result = await adapter.handleToolCall('editor_reader_move', { pos: { line: 0, col: 0 } });
    expect(result.error).toBe('no_reader');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-editor-core/src/mcp-tool-adapter.test.ts`
Expected: FAIL — unknown_tool

- [ ] **Step 3: Add reader state and cases to McpToolAdapter**

Add field: `private _reader: LineReader | null = null;`

Add cases:
```typescript
case 'editor_reader_start': {
  if (this._reader) this._reader.dispose();
  this._reader = this.editor.createReader(params.style);
  return { position: this._reader.position() };
}

case 'editor_reader_move': {
  if (!this._reader) return { error: 'no_reader' };
  this._reader.moveTo(params.pos as Position);
  return { position: this._reader.position() };
}

case 'editor_reader_advance': {
  if (!this._reader) return { error: 'no_reader' };
  this._reader.advance();
  return { position: this._reader.position() };
}

case 'editor_reader_stop': {
  if (this._reader) {
    this._reader.dispose();
    this._reader = null;
  }
  return { success: true };
}
```

- [ ] **Step 4: Add MCP tool definitions for reader**

Add to `MCP_TOOL_DEFINITIONS`:
```typescript
{
  name: 'editor_reader_start',
  description: 'Start a tracked line reader that highlights the current sentence. Animates between positions.',
  inputSchema: {
    type: 'object',
    properties: { style: highlightStyleSchema },
  },
},
{
  name: 'editor_reader_move',
  description: 'Move the line reader to a specific position.',
  inputSchema: {
    type: 'object',
    properties: { pos: { ...positionSchema, description: 'Target position' } },
    required: ['pos'],
  },
},
{
  name: 'editor_reader_advance',
  description: 'Advance the line reader to the next sentence.',
  inputSchema: { type: 'object', properties: {} },
},
{
  name: 'editor_reader_stop',
  description: 'Stop and dispose the line reader, removing its highlight.',
  inputSchema: { type: 'object', properties: {} },
},
```

- [ ] **Step 5: Update mock to include createReader**

Add to `createMockEditableText()`:
```typescript
createReader: vi.fn(() => ({
  moveTo: vi.fn(),
  advance: vi.fn(),
  position: vi.fn(() => ({ line: 0, col: 0 })),
  dispose: vi.fn(),
})),
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-editor-core/src/mcp-tool-adapter.test.ts`
Expected: PASS

- [ ] **Step 7: Run full test suite for both packages**

Run: `yarn vitest run packages/pages-editor-core/ packages/pages-markdown-editor/`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-editor-core/src/
git commit -m "feat(#535): add MCP tools for LineReader — start, move, advance, stop"
```

---

## References

- [casehubio/casehub-pages#535] — focal issue with full spec
- [packages/pages-editor-core/src/types.ts] — EditableText interface, HighlightStyle type
- [packages/pages-editor-core/src/editable-text-bridge.ts] — abstract bridge class
- [packages/pages-markdown-editor/src/markdown-editor-bridge.ts] — ProseMirror bridge implementation
- [packages/pages-editor-core/src/mcp-tool-adapter.ts] — MCP tool dispatch
- [packages/pages-editor-core/src/mcp-tool-definitions.ts] — MCP tool schemas
- [packages/pages-markdown-editor/src/overlay/annotation-renderer.ts] — overlay positioning pattern (AnnotationRenderer)
- [packages/pages-document-diff/src/pages-document-diff.ts:473-497] — section-highlight-bar pattern
