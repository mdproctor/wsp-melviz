# Design Journal — issue-530-milkdown-editor

## 2026-10-07 — Design + Batch 1 foundation

**Branch:** `issue-530-milkdown-editor` | **Issue:** #530

### Design phase

Brainstormed the Milkdown editor design through 8 decisions (D0-D9 after review).
Key choices: Milkdown for its markdown-first architecture (ProseMirror schema derived
from remark/mdast), one `EditableText` interface for all editors, MCP tools with
positional core + semantic helpers, edit sessions with exclusive lock.

Decision review (standard, 3 rounds) caught a factual error — `@milkdown/lit` doesn't
exist. Corrected D0 rationale to markdown-first design as the differentiator, rewrote
D6 toolbar strategy as custom Lit buttons dispatching `@milkdown/kit` commands.

Spec review (standard, 3 rounds) added: position index caching strategy, undo/redo
behavior across modes, session contention error contract, import failure fallback.

### Batch 1: Foundation

Shipped `pages-editor-core` with:
- `EditableText` interface (renamed from `ScenarioEditableText`, 0-based positions)
- `EditableTextBridge` abstract base (highlight ID tracking, annotation lifecycle, edit sessions)
- `EDITABLE_TEXT` symbol changed from `scenario-editable-text` to `editable-text`
- Discovery functions (`isEditableText`, `findEditableText`)

Refactored `CodeEditorBridge` to extend `EditableTextBridge`. Added `getLine()`,
`findText()`, `removeHighlight()`. Highlight StateField now tracks IDs for targeted
removal. CSS class prefix changed from `scenario-highlight-` to `editor-highlight-`.

`pages-aria` re-exports from core with `ScenarioEditableText` as deprecated alias.

**89 tests pass** (19 core + 70 code-editor). Pre-existing failures in pages-aria
(playbook parser, tutorial host) are unrelated.

### Next session

Batch 2: Scaffold `pages-markdown-editor`, mount Milkdown in LIT, implement
`MarkdownEditorBridge` with cached position index, build custom Lit toolbar.
