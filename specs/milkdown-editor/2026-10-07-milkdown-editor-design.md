# Milkdown Editor — Design Spec

## Overview

Add a rich markdown editor to the CaseHub Pages platform, powered by Milkdown (ProseMirror-based, markdown-first). The editor serves dual purposes: **content authoring** within pages, and **LLM conversation rendering/editing** where an LLM writes markdown and a human reads/edits it.

The design introduces three new packages and refactors two existing ones:

| Package | Role | Status |
|---------|------|--------|
| `pages-editor-core` | Shared interface, bridge base, MCP adapter | **New** |
| `pages-markdown-editor` | Milkdown LIT wrapper, MarkdownEditorBridge | **New** |
| `pages-document-diff` | Generic diff/split-view extracted from blocks-ui | **New** (extraction) |
| `pages-code-editor` | Existing CodeMirror wrapper, refactored to use shared core | **Refactor** |
| `pages-aria` | Remove duplicated EditableText interface | **Refactor** |

## Architecture

### Package Dependency Graph

```
pages-editor-core
├── EditableText interface
├── EditableTextBridge (abstract base)
├── MCP tool adapter
└── (future: scroll sync, overlay renderer, split view, edit sessions)

pages-code-editor ──depends-on──> pages-editor-core
├── PagesCodeEditor (LIT component)
└── CodeEditorBridge extends EditableTextBridge

pages-markdown-editor ──depends-on──> pages-editor-core, pages-code-editor
├── PagesMarkdownEditor (LIT component)
├── MarkdownEditorBridge extends EditableTextBridge
└── Dual-mode toggle (WYSIWYG ↔ source via pages-code-editor)

pages-document-diff ──depends-on──> pages-editor-core
├── PagesDocumentDiff (LIT component, generic)
├── LCS line diff, word-level highlights
├── Canvas minimap, heading-based scroll sync
└── Split/unified view modes

pages-aria ──depends-on──> pages-editor-core
└── imports EditableText from core (removes local duplicate)
```

### Component Model

```
<pages-markdown-editor>
  ├── Shadow DOM
  │   ├── <div class="toolbar">        ← Custom Lit toolbar
  │   │   └── toolbar buttons → ctx.get(commandsCtx).call(cmd)
  │   ├── <div class="editor-host">    ← Milkdown mounts here (WYSIWYG mode)
  │   │   └── ProseMirror EditorView
  │   ├── <pages-code-editor>          ← Source mode (lazy-loaded)
  │   │   └── CodeMirror EditorView (markdown lang)
  │   └── <div class="overlay-layer">  ← Floating annotations
  │       └── arrows, callouts, markers
  └── Properties
      ├── value: string (markdown text)
      ├── mode: 'wysiwyg' | 'source' | 'split'
      ├── readonly: boolean
      ├── extensions: MilkdownPlugin[]
      └── label: string (ARIA)
```

## 1. pages-editor-core

### EditableText Interface

Renamed from `ScenarioEditableText`. Single canonical location — resolves the current duplication between `pages-aria` and `pages-code-editor`.

```typescript
export interface Position {
  line: number;
  col: number;
}

export type HighlightStyle = 'pulse' | 'underline' | 'glow' | 'box';

export interface AnnotationOptions {
  type: 'arrow' | 'callout' | 'marker' | 'numbered';
  text?: string;
  style?: Record<string, string>;
}

export interface EditSession {
  readonly owner: string;
  readonly mode: 'exclusive';
  cancel(): void;
}

export interface EditableText {
  // Content
  getText(): string;
  getLineCount(): number;
  setContent(text: string): void;

  // Editing
  insertText(text: string): void;
  replaceRange(from: Position, to: Position, text: string): void;
  deleteRange(from: Position, to: Position): void;

  // Cursor
  setCursor(line: number, col: number): void;
  getCursor(): Position;

  // Inline decorations (text-anchored, move with edits)
  highlight(from: Position, to: Position, style?: HighlightStyle): string;
  removeHighlight(id: string): void;
  clearHighlights(): void;

  // Floating annotations (position-relative, reposition on scroll)
  addAnnotation(anchor: Position, options: AnnotationOptions): string;
  removeAnnotation(id: string): void;
  clearAnnotations(): void;

  // Edit sessions (D3)
  beginEditSession(owner: string): EditSession;
  endEditSession(session: EditSession): void;

  // Optional
  triggerCompletion?(): void;
  selectCompletion?(label: string): boolean;
}

export const EDITABLE_TEXT: unique symbol = Symbol.for('editable-text') as any;

export function isEditableText(
  el: Element
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

### EditableTextBridge (Abstract Base)

Shared logic for both `CodeEditorBridge` and `MarkdownEditorBridge`:

- Highlight ID generation and tracking (Map<string, decoration>)
- Annotation ID generation and lifecycle management
- Edit session state (active session, snapshot for rollback)
- Event emission for session state changes

Engine-specific decoration rendering (CodeMirror StateEffect/StateField vs ProseMirror DecorationSet+plugin) stays in each concrete bridge.

### MCP Tool Adapter

Maps MCP tool calls to `EditableText` methods. Provides the MCP tool definitions and handles ambiguity reporting for semantic helpers.

**Core tools (positional):**

| Tool | EditableText method | Returns |
|------|-------------------|---------|
| `editor_get_content` | `getText()` | Document text |
| `editor_set_content` | `setContent(text)` | void |
| `editor_replace_range` | `replaceRange(from, to, text)` | void |
| `editor_insert_text` | `insertText(text)` | void |
| `editor_get_cursor` | `getCursor()` | Position |
| `editor_set_cursor` | `setCursor(line, col)` | void |
| `editor_highlight` | `highlight(from, to, style)` | decoration ID |
| `editor_remove_highlight` | `removeHighlight(id)` | void |
| `editor_annotate` | `addAnnotation(anchor, opts)` | annotation ID |
| `editor_remove_annotation` | `removeAnnotation(id)` | void |
| `editor_clear_overlays` | `clearHighlights()` + `clearAnnotations()` | void |
| `editor_begin_session` | `beginEditSession(owner)` | session handle |
| `editor_end_session` | `endEditSession(session)` | void |

**Resolution helpers (semantic → positional):**

| Tool | Returns | On ambiguity |
|------|---------|-------------|
| `editor_find_text` | `{ matches: Position[] }` | Multiple matches returned; LLM picks |
| `editor_find_heading` | `{ position: Position, endPosition: Position }` | Reports "no match" or "multiple: [list]" |
| `editor_get_line` | `{ text: string }` at line N | Out-of-range error |

## 2. pages-markdown-editor

### PagesMarkdownEditor (LIT Component)

Wraps Milkdown using `@milkdown/kit` (framework-agnostic core). Mounts ProseMirror to a div inside shadow DOM.

**Properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `value` | `string` | `''` | Markdown text content |
| `mode` | `'wysiwyg' \| 'source' \| 'split'` | `'wysiwyg'` | Active view mode |
| `readonly` | `boolean` | `false` | Disables editing |
| `lineNumbers` | `boolean` | `true` | Show line numbers in source mode |
| `tabSize` | `number` | `2` | Tab size in source mode |
| `label` | `string \| undefined` | — | ARIA label |
| `extensions` | `MilkdownPlugin[]` | `[]` | Additional Milkdown plugins |

**Events:**

| Event | Detail | When |
|-------|--------|------|
| `input` | `{ value: string }` | Every document change |
| `change` | `{ value: string }` | On blur |
| `mode-changed` | `{ mode: string }` | View mode toggled |
| `session-started` | `{ owner: string }` | Edit session begins |
| `session-ended` | `{ owner: string, cancelled: boolean }` | Edit session ends |

**Public API:**

- `get editorView()` — underlying ProseMirror EditorView (WYSIWYG) or CodeMirror EditorView (source)
- `applyChangesSilently(changes)` — dispatch changes without triggering `input` event

### MarkdownEditorBridge

Extends `EditableTextBridge`. Translates between `EditableText`'s line/column positions and ProseMirror's internal offset model.

Key implementation details:
- `toOffset(pos: Position): number` — serializes ProseMirror doc to markdown, converts line/col to character offset, maps back to ProseMirror doc offset via `remark`
- `toPosition(offset: number): Position` — reverse mapping
- `highlight()` — creates a ProseMirror `Decoration.inline` via a plugin's DecorationSet
- `addAnnotation()` — creates a positioned DOM element in the overlay layer, repositioned on scroll/resize via `requestAnimationFrame`

### Toolbar

Custom Lit toolbar using `@milkdown/kit` commands:

```
[B] [I] [~~] [H▾] [•] [1.] [☐] [<>] [---] [🔗] [📷] [📊] [∑] [⊞]
 │   │   │    │    │   │    │    │    │     │     │     │    │    │
 │   │   │    │    │   │    │    │    │     │     │     │    │    └─ Table
 │   │   │    │    │   │    │    │    │     │     │     │    └─ Math (lazy KaTeX)
 │   │   │    │    │   │    │    │    │     │     │     └─ Diagram (lazy Mermaid)
 │   │   │    │    │   │    │    │    │     │     └─ Image
 │   │   │    │    │   │    │    │    │     └─ Link
 │   │   │    │    │   │    │    │    └─ Horizontal rule
 │   │   │    │    │   │    │    └─ Code block
 │   │   │    │    │   │    └─ Task list
 │   │   │    │    │   └─ Ordered list
 │   │   │    │    └─ Bullet list
 │   │   │    └─ Heading level (dropdown)
 │   │   └─ Strikethrough
 │   └─ Italic
 └─ Bold
```

Each button: `ctx.get(commandsCtx).call(toggleBoldCommand.key)` etc.

Styled with pages design tokens (`--pages-neutral-*`, `--pages-accent-*`).

### Dual-Mode Toggle

**WYSIWYG → Source:**
1. Serialize ProseMirror doc to markdown via Milkdown's serializer
2. Lazy-load `pages-code-editor` (dynamic import, ~60KB)
3. Create `CodeEditorBridge`, swap `EDITABLE_TEXT` symbol
4. Show CodeMirror with markdown content, line numbers

**Source → WYSIWYG:**
1. Get markdown text from CodeMirror
2. Parse markdown back to ProseMirror doc via Milkdown's parser
3. Swap `EDITABLE_TEXT` symbol back to `MarkdownEditorBridge`
4. Show Milkdown

**Cursor position preservation:** Map cursor through markdown text — the line/column in source corresponds to a character offset in the markdown, which maps to a ProseMirror node position. Approximate, not pixel-perfect.

### Split Mode

Both views visible side-by-side with synchronized scrolling:

- Heading-based anchor pairing (reuse `document-diff` pattern)
- Position-aware interpolation (not linear) — WYSIWYG heights differ from source heights
- Bidirectional sync with `_syncing` guard flag (double-rAF release)
- Edits in either view update the other via serialize/parse

## 3. Overlay System

### Inline Decorations (Text-Anchored)

- Created via `highlight(from, to, style)`, returns ID
- Move with text on edits (tied to document positions)
- Invalidated when underlying text is deleted
- CSS classes: `editor-highlight-pulse`, `editor-highlight-underline`, `editor-highlight-glow`, `editor-highlight-box`

### Floating Annotations (Position-Relative)

- Created via `addAnnotation(anchor, options)`, returns ID
- Rendered as absolutely positioned DOM elements in the overlay layer
- Repositioned on scroll/resize via `requestAnimationFrame`
- Types:
  - **arrow** — SVG arrow from annotation box to anchor position
  - **callout** — text box with pointer to anchor
  - **marker** — numbered circle at anchor position
  - **numbered** — numbered label inline with an arrow to the range

### Overlay Targeting in Dual Mode

When in split mode, overlays target whichever view contains the anchor position. In WYSIWYG-only or source-only mode, overlays target the active view. Inline decorations render in both views when the same content range is visible in both. Floating annotations render once, anchored to the view where the position is most meaningful (WYSIWYG for rendered elements, source for line-level).

## 4. Concurrent Editing

### Edit Sessions (v1 — Exclusive Lock)

```
LLM: beginEditSession("claude")
  → Editor enters read-only for human
  → "AI editing..." indicator shown
  → Document snapshot taken for rollback
  
LLM: replaceRange(...), insertText(...), highlight(...)
  → Changes stream in, visible to human
  
LLM: endEditSession(session)
  → Editor returns to read-write
  → Indicator removed
```

**Cancel semantics:** Human clicks cancel → session rolls back to the snapshot taken at `beginEditSession()`. All partial edits are discarded. The bridge restores ProseMirror doc state from the snapshot.

**Future path:** Collaborative mode (`mode: 'collaborative'`) is a new capability, not a silent upgrade. Callers must explicitly request it and handle conflict resolution. The `EditSession` interface gains a `mode` discriminant.

## 5. Bundle and Loading Strategy

| Dependency | Size (gz) | Loading |
|------------|-----------|---------|
| Milkdown/ProseMirror | ~100KB | Eager (default view) |
| CodeMirror | ~60KB | Lazy (first source toggle) |
| KaTeX | ~130KB | Lazy (math content or toolbar click) |
| Mermaid | ~400KB | Lazy (diagram content or toolbar click) |

Milkdown plugins are grouped by tier:
- **Core** (eager): bold, italic, headings, lists, code, links, images
- **Standard** (loaded with editor): tables, task lists, strikethrough, blockquotes
- **Heavy** (lazy): math (KaTeX), diagrams (Mermaid)

## 6. document-diff Extraction (D7)

Extract generic infrastructure from `blocks-ui/components/document-workbench/src/document-diff.ts` into `pages-document-diff`:

**Moves to pages (~660 lines):**
- LCS line diff engine (`_lineDiff`)
- Word-level diff and highlights (`_wordDiff`, `_applyWordHighlights`, `_annotateWordDiffs`)
- Canvas minimap (`_drawDiffMap`)
- Heading-based scroll sync (`_buildScrollAnchors`, `_setupScrollSync`, `_interp`)
- Divider drag (`_setupDividerDrag`)
- Drop zones (`_setupDropZone`)
- Markdown rendering (`_renderMarkdown`, `_syncPanelContent`)
- Diff navigation (`nextDiff`, `prevDiff`, `scrollToChunk`)
- View modes (`split`/`unified`, `_renderUnified`)
- Heading search (`_findHeading`, `_normalizeLocation`, `scrollToLocation`)
- Section highlight (`highlightSection`, `clearHighlight`)

**Stays in blocks-ui (~130 lines):**
- Thread management (`_threadAnchors`, `thread-created`/`thread-resolved`/`thread-focused` listeners, `_renderThreadGutterMarkers`)
- Timeline snapshot fetching (`timeline-comparison-changed` → `/api/debate/{sessionId}/snapshot/{index}`)
- Selection-to-thread bridge (`selection-changed` events with `side: A/B`)

Blocks-ui keeps a `DrafthouseDocumentDiff` that extends `PagesDocumentDiff` with domain-specific review features.

## 7. Refactoring Existing Code

### ScenarioEditableText → EditableText Rename

1. Move interface to `pages-editor-core`
2. Rename `ScenarioEditableText` → `EditableText`
3. Rename symbol from `scenario-editable-text` → `editable-text`
4. Update `CodeEditorBridge` to extend `EditableTextBridge` from core
5. Update `pages-aria` to import from `pages-editor-core`
6. Update `code-editor-bridge.ts` to remove local interface copy
7. Update `command-executor.ts` and all consumers

### CodeEditorBridge Refactor

- Extract shared logic to `EditableTextBridge` in `pages-editor-core`
- `CodeEditorBridge extends EditableTextBridge`
- Engine-specific: CodeMirror `StateEffect`/`StateField`/`Decoration.mark` stays
- Shared: ID generation, session management, event emission moves to base

## 8. Testing Strategy

### Unit Tests
- `EditableText` interface compliance tests — shared test suite that both `CodeEditorBridge` and `MarkdownEditorBridge` must pass
- MCP tool adapter tests — mock EditableText, verify tool call → method mapping
- Scroll sync engine tests — verify anchor pairing and interpolation
- Diff engine tests — verify LCS, word-level diff (existing `document-diff.test.ts`)

### Integration Tests
- Milkdown mount/unmount lifecycle in LIT shadow DOM
- WYSIWYG ↔ source round-trip (markdown → ProseMirror → markdown)
- Overlay rendering (highlights visible, annotations positioned)
- Edit session lock/unlock/cancel with rollback verification

### Visual Tests (Playwright)
- Toolbar renders all icons
- Dual mode toggle preserves content
- Split mode shows synchronized scrolling
- Overlay annotations visible and correctly positioned
- Edit session indicator shown/hidden

## References

- `packages/pages-code-editor/src/pages-code-editor.ts` — existing CodeMirror LIT wrapper pattern
- `packages/pages-code-editor/src/code-editor-bridge.ts` — existing bridge implementation
- `packages/pages-aria/src/executor/editable-text.ts` — existing interface (to be moved)
- `packages/pages-aria/src/executor/command-executor.ts` — existing MCP-like command dispatching
- `packages/pages-builder/src/shell/diff-patch.ts` — `computeMinimalChanges` utility
- `blocks-ui/components/document-workbench/src/document-diff.ts` — scroll sync, diff engine (to be extracted)
- `milkdown.dev` — Milkdown documentation
- `@milkdown/kit` — framework-agnostic Milkdown core
- `@milkdown-lab/plugin-split-editing` — community reference for ProseMirror↔markdown sync