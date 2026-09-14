# Fix: Completion insertText loses YAML indentation context

**Issue:** casehubio/casehub-pages#442
**Date:** 2026-09-14
**Branch:** issue-442-completion-indentation

## Problem

The LSP server returns `CompletionItem` objects with `insertText` but no `textEdit`. LSP4IJ inserts the text at column 0 instead of the cursor position, breaking YAML structure. This is critical — YAML indentation is structural, not cosmetic.

## Fix

Add `textEdit` with explicit `Range` to every `CompletionItem` returned by `handleCompletion()`. The `textEdit.range` specifies exactly what text to replace; `textEdit.newText` is the completion text.

### Range computation

`handleCompletion()` already has `textBefore` (the line content up to the cursor) and `position`. The range end is always the cursor position. The range start depends on the completion context:

**Property completions** (the main code path, lines 55-70):

Extract the word prefix the user has typed by scanning `textBefore` backwards from the cursor. The prefix is any `\w[\w-]*` characters (plus optional leading `- `) at the end of `textBefore` after whitespace.

| Scenario | textBefore | Range start char | newText |
|----------|-----------|-----------------|---------|
| Empty after indent | `"    "` | 4 (cursor) | `title:` |
| Partial typed | `"    ti"` | 4 | `title:` |
| After dash-space | `"    - "` | 6 (cursor) | `type:` |
| Partial after dash | `"    - ty"` | 6 | `type:` |
| needsDash, no dash typed | `"    "` | 4 (cursor) | `- type:` |

When `needsDash` is true and the user hasn't typed a dash, the newText includes `- ` and the range starts at the cursor (no existing text to replace).

When `needsDash` is true and the user has typed a partial word without a dash, the range replaces the partial word and newText includes `- `.

**Value completions** (the `afterValueColon` path, lines 34-53):

The partial value is `afterValueColon[2]`. Range start = `position.character - partialValue.length`. newText = the enum label.

### Interface changes

Extend the internal `CompletionItem` in `completion.ts`:

```typescript
export interface CompletionItem {
  label: string;
  kind: number;
  insertText?: string;
  detail?: string;
  textEdit?: {
    range: { start: { line: number; character: number }; end: { line: number; character: number } };
    newText: string;
  };
}
```

Update `ServerHandler.onCompletion` return type in `server.ts` to include `textEdit`.

### Server mapping

In `server-node.ts`, map the internal `textEdit` to the LSP `TextEdit`:

```typescript
...(c.textEdit ? { textEdit: { range: c.textEdit.range, newText: c.textEdit.newText } } : {}),
```

When `textEdit` is present, omit `insertText` — the LSP spec says `textEdit` takes precedence, and sending both may confuse some clients.

In `server-browser.ts`, apply the same mapping for the browser CodeMirror transport.

### Computing the word prefix

Add a helper to `completion.ts`:

```typescript
function getWordStart(textBefore: string): number {
  const match = textBefore.match(/([\w][\w-]*)$/);
  return match ? textBefore.length - match[0].length : textBefore.length;
}
```

This returns the character offset where the typed word begins. If no word is being typed (cursor after whitespace or dash-space), it returns the cursor position (zero-width range = pure insert).

## Files changed

| File | Change |
|------|--------|
| `packages/pages-lsp/src/completion.ts` | Add `textEdit` to return values, add `getWordStart` helper |
| `packages/pages-lsp/src/server.ts` | Extend `ServerHandler.onCompletion` return type |
| `packages/pages-lsp/src/server-node.ts` | Map `textEdit` to LSP TextEdit, drop `insertText` when `textEdit` present |
| `packages/pages-lsp/src/server-browser.ts` | Same mapping for browser transport |
| `packages/pages-lsp/src/completion.test.ts` | Add tests for textEdit ranges at various indentation levels |

## Testing

- Property completion at root level (col 0) — range start = 0
- Property completion at 2-space indent — range start = 2
- Property completion at 4-space indent with partial text — range replaces partial
- Value completion after `type: ` with partial value — range replaces partial value
- Array item with needsDash — newText includes `- `, range doesn't include existing whitespace
- Regression: existing label/kind/detail fields unchanged

## References

- packages/pages-lsp/src/completion.ts — completion provider, where fix lands
- packages/pages-lsp/src/server-node.ts:80-88 — LSP mapping (no textEdit today)
- packages/pages-lsp/src/schema-navigation.ts — buildYamlContext, getIndentLevel
- LSP spec textDocument/completion — textEdit takes precedence over insertText
- HANDOFF.md — indentation bug noted as follow-up from #437
