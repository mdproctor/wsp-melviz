---
layout: post
title: "Why Your LSP Completions Land at Column Zero"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [intellij, lsp, lsp4ij, completion, yaml]
series: issue-442-completion-indentation
---

# Why Your LSP Completions Land at Column Zero

The LSP server returns correct completions. IntelliJ shows them in the popup. You select one and the text appears — at column 0, not at the cursor. In YAML, that doesn't just look wrong. It breaks the document.

I found this while verifying the LSP4IJ work from the previous session. Completions appeared in the list, which meant the full handshake — initialize, didOpen, textDocument/completion — was working. But selecting an item destroyed the YAML structure. A property that should have appeared at column 8 inside a `components` block was landing at the start of the line.

The LSP specification says a `CompletionItem` can carry either `insertText` (the text to insert) or `textEdit` (the text plus the exact range it should replace). When `textEdit` is absent, the spec says clients should compute a default replacement range from the cursor position. VS Code does this. LSP4IJ does not — it falls back to column 0.

The fix was to compute `textEdit` ranges server-side. The completion handler already knew everything it needed: the line content up to the cursor, the cursor position, whether a dash prefix was required for array items. A four-line helper function extracts the start position of any partially-typed word:

```typescript
function getWordStart(textBefore: string): number {
  const match = textBefore.match(/([\w][\w-]*)$/);
  return match ? textBefore.length - match[0].length : textBefore.length;
}
```

The range end is always the cursor position. The range start is wherever the typed prefix begins — or the cursor position itself when nothing has been typed yet, giving a zero-width "pure insert" range. We added this to both the property completion path (navigating Zod schemas for YAML keys) and the value completion path (enum values after a colon). The two server transports — Node.js stdio for IntelliJ and web worker for the browser editor — both map the internal `textEdit` to the LSP wire format, falling back to `insertText` for any future code path that doesn't set a range.

This is the third LSP4IJ behaviour I've hit that deviates from the spec without any error signal. The stale bundle cache, the TextDocumentSync object-form requirement, and now the missing default edit range. Each one is silent — the server runs, responses arrive, but the client does something unexpected with them. I've been capturing these as garden entries; together they're building a practical survival guide for anyone targeting LSP4IJ from a custom server.
