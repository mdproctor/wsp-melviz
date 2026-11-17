# Completion Indentation Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #442 — fix(intellij): completion insertText loses YAML indentation context
**Issue group:** #442

**Goal:** Add `textEdit` with explicit ranges to LSP completion items so completions preserve YAML indentation in all clients.

**Architecture:** `completion.ts` already computes `insertText` and has full line/position context. Add a `getWordStart()` helper to find the replacement range start, return `textEdit` alongside existing fields. Both server transports (`server-node.ts`, `server-browser.ts`) map the internal textEdit to LSP `TextEdit` and drop `insertText` when textEdit is present.

**Tech Stack:** TypeScript, vscode-languageserver, vitest

## Global Constraints

- LSP `textEdit.range.end` is always the cursor position
- When `textEdit` is present, omit `insertText` (LSP spec: textEdit takes precedence)
- `textEdit.range` must not span multiple lines

---

## Batch 1: textEdit support in completion pipeline

### Task 1: Add textEdit to CompletionItem and handleCompletion

**Files:**
- Modify: `packages/pages-lsp/src/completion.ts:4-9` (CompletionItem interface)
- Modify: `packages/pages-lsp/src/completion.ts:14-71` (handleCompletion return values)
- Modify: `packages/pages-lsp/src/server.ts:45-49` (ServerHandler.onCompletion return type)
- Test: `packages/pages-lsp/src/completion.test.ts`

**Interfaces:**
- Consumes: `buildYamlContext`, `navigateSchema`, `schemaToCompletions`, `isArrayField` from `schema-navigation.ts`
- Produces: `CompletionItem.textEdit?: { range: { start: Position; end: Position }; newText: string }` — used by server-node.ts and server-browser.ts

- [ ] **Step 1: Write failing tests for textEdit ranges**

Add tests to `completion.test.ts` covering the four key scenarios. Each test verifies the `textEdit` field is present with correct range and newText.

```typescript
it('includes textEdit at root level (col 0)', () => {
  const registry = setup();
  const doc = '';
  const items = handleCompletion(
    'file:///app/test.page.yaml', doc, { line: 0, character: 0 }, registry,
  );
  const pagesItem = items.find(i => i.label === 'pages');
  expect(pagesItem?.textEdit).toEqual({
    range: { start: { line: 0, character: 0 }, end: { line: 0, character: 0 } },
    newText: 'pages',
  });
});

it('includes textEdit at indented position', () => {
  const registry = setup();
  const doc = 'pages:\n  - name: Home\n    components:\n      - type: bar-chart\n        ';
  const lines = doc.split('\n');
  const lastLine = lines[lines.length - 1]!;
  const items = handleCompletion(
    'file:///app/test.page.yaml', doc,
    { line: lines.length - 1, character: lastLine.length },
    registry,
  );
  const widthItem = items.find(i => i.label === '- width');
  expect(widthItem?.textEdit).toBeDefined();
  expect(widthItem!.textEdit!.range.start.line).toBe(lines.length - 1);
  expect(widthItem!.textEdit!.range.start.character).toBe(lastLine.length);
  expect(widthItem!.textEdit!.range.end.character).toBe(lastLine.length);
});

it('textEdit replaces partial text when user has typed a prefix', () => {
  const registry = setup();
  const doc = 'pages:\n  - name: Home\n    components:\n      - type: bar-chart\n        wi';
  const lines = doc.split('\n');
  const lastLine = lines[lines.length - 1]!;
  const items = handleCompletion(
    'file:///app/test.page.yaml', doc,
    { line: lines.length - 1, character: lastLine.length },
    registry,
  );
  const widthItem = items.find(i => i.label === '- width');
  expect(widthItem?.textEdit).toBeDefined();
  expect(widthItem!.textEdit!.range.start.character).toBe(lastLine.length - 2);
  expect(widthItem!.textEdit!.range.end.character).toBe(lastLine.length);
});

it('textEdit for value completion replaces partial value', () => {
  const registry = setup();
  const doc = 'pages:\n  - name: Home\n    components:\n      - type: bar';
  const lines = doc.split('\n');
  const lastLine = lines[lines.length - 1]!;
  const items = handleCompletion(
    'file:///app/test.page.yaml', doc,
    { line: lines.length - 1, character: lastLine.length },
    registry,
  );
  const barItem = items.find(i => i.label === 'bar-chart');
  expect(barItem?.textEdit).toBeDefined();
  expect(barItem!.textEdit!.range.start.character).toBe(lastLine.indexOf('bar'));
  expect(barItem!.textEdit!.newText).toBe('bar-chart');
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-lsp test -- --reporter=verbose 2>&1 | tail -30`
Expected: 4 new tests FAIL (textEdit is undefined)

- [ ] **Step 3: Extend CompletionItem interface**

In `completion.ts`, add `textEdit` to the interface:

```typescript
export interface CompletionItem {
  label: string;
  kind: number;
  insertText?: string;
  detail?: string;
  textEdit?: {
    range: {
      start: { line: number; character: number };
      end: { line: number; character: number };
    };
    newText: string;
  };
}
```

Update `ServerHandler.onCompletion` return type in `server.ts` to match — add the same `textEdit` field to the inline return type at line 45.

- [ ] **Step 4: Add getWordStart helper and wire textEdit into handleCompletion**

Add helper function in `completion.ts`:

```typescript
function getWordStart(textBefore: string): number {
  const match = textBefore.match(/([\w][\w-]*)$/);
  return match ? textBefore.length - match[0].length : textBefore.length;
}
```

For the **value completion** path (lines 34-53 — the `afterValueColon` branch), compute the range from the partial value:

```typescript
if (afterValueColon) {
  const key = afterValueColon[1] ?? '';
  const partialValue = afterValueColon[2] ?? '';
  const resolved = navigateSchema(
    format.documentSchema,
    [...yamlCtx.path, key],
    yamlCtx.siblings,
  );
  if (resolved) {
    const completions = schemaToCompletions(resolved);
    if (completions.length > 0 && completions[0]?.type === 'enum') {
      const rangeStart = position.character - partialValue.length;
      return completions.map(c => ({
        label: c.label,
        kind: ENUM_MEMBER_KIND,
        ...(c.detail ? { detail: c.detail } : {}),
        textEdit: {
          range: {
            start: { line: position.line, character: rangeStart },
            end: { line: position.line, character: position.character },
          },
          newText: c.label,
        },
      }));
    }
  }
  return [];
}
```

For the **property completion** path (lines 55-70), compute the range from the word start:

```typescript
const wordStart = getWordStart(textBefore);

const siblingKeys = new Set(Object.keys(yamlCtx.siblings));
return completions
  .filter(c => c.type !== 'property' || !siblingKeys.has(c.label))
  .map(c => {
    const newText = needsDash ? '- ' + (c.apply || c.label) : c.apply || c.label;
    return {
      label: needsDash ? '- ' + c.label : c.label,
      kind: c.type === 'property' ? PROPERTY_KIND : ENUM_MEMBER_KIND,
      insertText: newText,
      ...(c.detail ? { detail: c.detail } : {}),
      textEdit: {
        range: {
          start: { line: position.line, character: wordStart },
          end: { line: position.line, character: position.character },
        },
        newText,
      },
    };
  });
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-lsp test -- --reporter=verbose 2>&1 | tail -30`
Expected: all tests PASS (including existing tests — no regressions)

- [ ] **Step 6: Commit**

```bash
git -C "$PROJECT" add packages/pages-lsp/src/completion.ts packages/pages-lsp/src/server.ts packages/pages-lsp/src/completion.test.ts
git -C "$PROJECT" commit -m "feat(pages-lsp): add textEdit ranges to completion items

Compute explicit replacement ranges so LSP clients preserve YAML
indentation. Handles property completions, value completions, partial
text replacement, and array-item dash prefixes.

Refs #442"
```

### Task 2: Map textEdit in server transports

**Files:**
- Modify: `packages/pages-lsp/src/server-node.ts:80-88` (onCompletion mapping)
- Modify: `packages/pages-lsp/src/server-browser.ts:61-68` (onCompletion mapping)

**Interfaces:**
- Consumes: `CompletionItem.textEdit` from Task 1
- Produces: LSP-conformant `CompletionItem` with `textEdit` field, no `insertText` when `textEdit` present

- [ ] **Step 1: Update server-node.ts completion mapping**

Replace the `onCompletion` mapping at lines 80-88:

```typescript
connection.onCompletion((params): CompletionItem[] => {
  log(`completion: ${params.textDocument.uri} at ${params.position.line}:${params.position.character}`);
  return handler.onCompletion(params.textDocument.uri, params.position).map(c => ({
    label: c.label,
    kind: c.kind as CompletionItemKind,
    ...(c.textEdit
      ? { textEdit: { range: c.textEdit.range, newText: c.textEdit.newText } }
      : c.insertText ? { insertText: c.insertText } : {}),
    ...(c.detail ? { detail: c.detail } : {}),
  }));
});
```

- [ ] **Step 2: Update server-browser.ts completion mapping**

Replace the `onCompletion` mapping at lines 61-68:

```typescript
connection.onCompletion((params): CompletionItem[] => {
  return handler.onCompletion(params.textDocument.uri, params.position).map(c => ({
    label: c.label,
    kind: c.kind as CompletionItemKind,
    ...(c.textEdit
      ? { textEdit: { range: c.textEdit.range, newText: c.textEdit.newText } }
      : c.insertText ? { insertText: c.insertText } : {}),
    ...(c.detail ? { detail: c.detail } : {}),
  }));
});
```

- [ ] **Step 3: Run full test suite**

Run: `yarn workspace @casehubio/pages-lsp test -- --reporter=verbose 2>&1 | tail -30`
Expected: all tests PASS

- [ ] **Step 4: Commit**

```bash
git -C "$PROJECT" add packages/pages-lsp/src/server-node.ts packages/pages-lsp/src/server-browser.ts
git -C "$PROJECT" commit -m "feat(pages-lsp): map textEdit in server transports

Both node and browser transports now emit LSP textEdit when available,
falling back to insertText for backward compatibility.

Refs #442"
```

## References

- [2026-09-14-completion-indentation-design.md] — design spec this plan implements
- packages/pages-lsp/src/completion.ts — completion provider, primary change target
- packages/pages-lsp/src/server-node.ts:80-88 — node transport LSP mapping
- packages/pages-lsp/src/server-browser.ts:61-68 — browser transport LSP mapping
- packages/pages-lsp/src/server.ts:45-49 — ServerHandler interface
- packages/pages-lsp/src/schema-navigation.ts — buildYamlContext, getIndentLevel
- GitHub #442 — focal issue
