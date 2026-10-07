# Milkdown Editor — Design Spec

## Overview

Add a rich markdown editor to the CaseHub Pages platform, powered by Milkdown (ProseMirror-based, markdown-first). The editor serves dual purposes: **content authoring** within pages, and **LLM conversation rendering/editing** where an LLM writes markdown and a human reads/edits it.

**Tracking:** GitHub issues must be created before implementation begins — an epic for the overall milkdown editor work, with child issues for each package (`pages-editor-core`, `pages-markdown-editor`, `pages-document-diff`), the refactoring work, and ARC42STORIES.MD chapter update. The prior code-editor spec (issue #372) and scenario tutorials spec (issue #443) established the `ScenarioEditableText` SPI and `CodeEditorBridge` pattern that this spec evolves.

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
├── Overlay renderer (highlight + annotation lifecycle)
└── Edit session management (lock, snapshot, rollback)

pages-code-editor ──depends-on──> pages-editor-core
├── PagesCodeEditor (LIT component)
└── CodeEditorBridge extends EditableTextBridge

pages-markdown-editor ──depends-on──> pages-editor-core
├── PagesMarkdownEditor (LIT component)
├── MarkdownEditorBridge extends EditableTextBridge
└── Dual-mode toggle (WYSIWYG ↔ source via dynamic import of pages-code-editor)

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
  getLine(line: number): string;
  getLineCount(): number;
  setContent(text: string): void;

  // Search
  findText(query: string): Position[];

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

**Relationship to command-executor:** The MCP tool adapter and the scenario command-executor (`command-executor.ts`) are two separate entry points to the same `EditableText` interface, serving different audiences:
- **Command-executor:** dispatches pre-authored scenario steps via the `aria` delivery channel. Supports progressive typing animation, spotlight integration, and speed control. Used for tutorial/demo playback.
- **MCP tool adapter:** direct, immediate LLM agent interaction. No animation — all operations execute instantly. Used for interactive editing sessions.

Both call through `EditableText`. They should NOT be consolidated — their behavioral requirements differ fundamentally (animated vs. instant). Shared logic (position validation, error formatting) lives in `EditableTextBridge`.

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

| Tool | EditableText method | Returns | On ambiguity |
|------|-------------------|---------|-------------|
| `editor_find_text` | `findText(query)` | `{ matches: Position[] }` | Multiple matches returned; LLM picks |
| `editor_find_heading` | _(adapter utility)_ | `{ position, endPosition }` | Reports "no match" or "multiple: [list]" |
| `editor_get_line` | `getLine(line)` | `{ text: string }` at line N | Out-of-range error |

`editor_find_text` and `editor_get_line` map directly to `EditableText` methods — each engine provides an efficient implementation (CodeMirror: built-in search/line access; ProseMirror: document tree walk, no serialization needed). `editor_find_heading` is an adapter-level utility that calls `getText()` and parses heading structure — heading semantics are markdown-level, not a core text operation.

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
- `applyChangesSilently(tr: Transaction)` — dispatch a ProseMirror `Transaction` without triggering `input` event. In source mode, the method serializes the transaction's effect to a CodeMirror `ChangeSpec` and applies it to the CodeMirror view. The type is `Transaction` (ProseMirror), not `ChangeSpec` (CodeMirror) — ProseMirror is the canonical editor engine; source mode is a secondary view.

### MarkdownEditorBridge

Extends `EditableTextBridge`. Translates between `EditableText`'s line/column positions and ProseMirror's internal offset model.

#### Position Mapping — Cached Position Index

The bridge maintains a **bidirectional position index** between markdown line/col coordinates and ProseMirror document offsets. This avoids full document serialization on every position operation — block-level serialization may occur for structural edits and multi-line block access.

**Index construction:** When Milkdown parses markdown into a ProseMirror document (on initial load and on source→WYSIWYG toggle), the parser emits source-position metadata via remark's `sourcePositions` plugin. The bridge captures this into a sorted array mapping `{line, col}` → `docOffset` for each block-level node boundary.

**Index update — two-sided:**
ProseMirror's `Mapping` shifts the `docOffset` side of the index (right side) on each transaction. But `Mapping` has no concept of markdown line numbers — it operates on document positions, not serialized text coordinates. The `{line, col}` side (left side) requires knowing how many markdown lines the inserted/deleted content spans.

For inline edits within a block (typing text, adding bold markers), line counts don't change — only `docOffset` shifts via `Mapping`. For structural edits (adding a paragraph, inserting a table, splitting a heading), the bridge serializes the affected ProseMirror node(s) to markdown, counts lines, and adjusts all subsequent index entries' line numbers. This is partial serialization — one or a few blocks, not the full document.

**Lookup:**
- `toOffset(pos: Position): number` — binary search the cached index for the nearest block boundary at or before `pos.line`, then walk ProseMirror nodes within that block to resolve `pos.col` to a precise doc offset.
- `toPosition(offset: number): Position` — reverse binary search: find the block boundary, compute line/col from the node's source position metadata.
- `getLine(line)` — locates the containing block via the index. For single-line blocks (paragraphs, headings), extracts text directly from ProseMirror nodes. For multi-line blocks (tables, code blocks), serializes that block to markdown and splits by newline to access the requested line. This is block-level serialization, not document-level.

**Fidelity constraints:** For inline content within simple paragraphs, headings, and list items, the mapping is exact. For complex structures (tables, deeply nested lists), the mapping resolves to the nearest valid position within the containing cell/item. The bridge logs a warning when a position falls in an ambiguous region. MCP tool operations on ambiguous positions return an error with the nearest valid alternatives.

**Key distinction:** "Approximate" cursor preservation during WYSIWYG↔source toggle is a UX convenience (best-effort cursor placement after a structural re-parse). MCP tool position operations use the cached index and are precise — an LLM edit will land at the correct position or the tool returns an error.

Key implementation details:
- `highlight()` — creates a ProseMirror `Decoration.inline` via a plugin's DecorationSet
- `addAnnotation()` — creates a positioned DOM element in the overlay layer, repositioned on scroll/resize via `requestAnimationFrame`
- `findText(query)` — walks ProseMirror document tree node-by-node, matching text content without serialization
- `getLine(line)` — uses the position index to locate the containing block; single-line blocks return text directly, multi-line blocks serialize the block and split by newline

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

**Keyboard shortcuts:** Milkdown's `@milkdown/kit` provides standard ProseMirror keybindings. The editor uses these defaults:

| Shortcut | Action |
|----------|--------|
| `Mod+B` | Toggle bold |
| `Mod+I` | Toggle italic |
| `Mod+Shift+X` | Toggle strikethrough |
| `Mod+E` | Toggle inline code |
| `Mod+Shift+7` | Toggle ordered list |
| `Mod+Shift+8` | Toggle bullet list |
| `Mod+K` | Insert/edit link |

(`Mod` = Ctrl on Windows/Linux, Cmd on macOS.) Toolbar buttons display their shortcut in the tooltip via `title` attribute. The keyboard shortcuts are discoverable via ARIA — each toolbar button has `aria-keyshortcuts` set to the binding.

Styled with pages design tokens (`--pages-neutral-*`, `--pages-accent-*`).

### Dual-Mode Toggle

Source mode uses a **dynamic import** of `@casehubio/pages-code-editor` — no compile-time TypeScript dependency. The `pages-markdown-editor` package depends only on `pages-editor-core`. The `<pages-code-editor>` element is discovered by tag name after dynamic import, and the bridge swap uses the `EDITABLE_TEXT` symbol (already runtime-discovered via `Symbol.for`). This keeps the packages independent and the CodeMirror dependency truly lazy.

**WYSIWYG → Source:**
1. Serialize ProseMirror doc to markdown via Milkdown's serializer
2. `await import('@casehubio/pages-code-editor')` (first toggle only, ~60KB)
3. Create `<pages-code-editor>` element, discover `CodeEditorBridge` via `EDITABLE_TEXT` symbol, swap
4. Show CodeMirror with markdown content, line numbers

**Import failure:** If the dynamic import in step 2 fails (network error, module not found), the toggle reverts to WYSIWYG mode — the ProseMirror view remains active, no bridge swap occurs. The component emits a `mode-changed` event with `{ mode: 'wysiwyg', error: 'source-unavailable' }` so the host can surface a notification. The toggle button is not disabled — the user can retry.

**Source → WYSIWYG:**
1. Get markdown text from CodeMirror
2. Parse markdown back to ProseMirror doc via Milkdown's parser, rebuild position index
3. Swap `EDITABLE_TEXT` symbol back to `MarkdownEditorBridge`
4. Show Milkdown

**Cursor position preservation:** Best-effort UX convenience during mode toggle. The bridge maps cursor position through the markdown text — the line/column in source corresponds to a character offset, which maps to the nearest ProseMirror node position via the cached position index. This is approximate for complex structures (tables, nested lists) but exact for simple content.

### Split Mode

Both views visible side-by-side with synchronized scrolling:

- Heading-based anchor pairing (reuse `document-diff` pattern)
- Position-aware interpolation (not linear) — WYSIWYG heights differ from source heights
- Bidirectional sync with `_syncing` guard flag (double-rAF release)

**Edit synchronization (not O(n) per keystroke):**

Content sync between views is **debounced at 150ms** — keystrokes within the debounce window are batched into a single sync.

- **WYSIWYG → Source:** The ProseMirror transaction's `changes` describe what was modified. The bridge serializes only the affected block(s) and applies a targeted text replacement in CodeMirror via `applyChangesSilently`, rather than re-serializing the entire document.
- **Source → WYSIWYG:** On debounce fire, diff old markdown against new using `computeMinimalChanges` (existing utility in `pages-builder/src/shell/diff-patch.ts`). Apply only the changed blocks as ProseMirror transactions.
- **Fallback:** If the targeted sync fails (e.g., structural change that crosses block boundaries like converting a paragraph to a table), fall back to a full re-parse of the changed view. This is the slow path and expected to be rare.

## 3. Overlay System

### Inline Decorations (Text-Anchored)

- Created via `highlight(from, to, style)`, returns ID
- Move with text on edits (tied to document positions)
- Invalidated when underlying text is deleted
- CSS classes: `editor-highlight-pulse`, `editor-highlight-underline`, `editor-highlight-glow`, `editor-highlight-box`

Note: These CSS classes are internal to the editor shadow DOM and replace the `scenario-highlight-*` classes used in `CodeEditorBridge` (R1-13). The `visual-feedback.ts` system in `pages-aria` is a separate concern — it highlights DOM elements (buttons, inputs, panels) during scenario playback using `outline` CSS. The editor decoration system highlights text ranges within the editor using engine-native decoration APIs. These are orthogonal: element-level spotlight vs. text-range decoration.

### Floating Annotations (Position-Relative)

- Created via `addAnnotation(anchor, options)`, returns ID
- Rendered as absolutely positioned DOM elements in the overlay layer
- Repositioned on scroll/resize via `requestAnimationFrame`
- Types:
  - **arrow** — SVG arrow from annotation box to anchor position
  - **callout** — text box with pointer to anchor
  - **marker** — numbered circle at anchor position
  - **numbered** — numbered label inline with an arrow to the range

#### Coordinate System

Annotations are positioned using a two-step coordinate transform:

1. **Position → pixel coords:** `EditorView.coordsAtPos(docOffset)` (ProseMirror) or `view.coordsAtPos(offset)` (CodeMirror) returns `{left, top, right, bottom}` in viewport-relative pixels. These APIs account for scroll position internally.
2. **Viewport → overlay-relative:** Subtract the overlay div's `getBoundingClientRect()` origin to get CSS `left`/`top` values for the annotation element.

On scroll/resize, the `requestAnimationFrame` loop re-executes step 1 (the editor's `coordsAtPos` result changes as content scrolls) and step 2 (the overlay div may have moved). Shadow DOM is transparent to this — `coordsAtPos` returns viewport-relative coordinates regardless of shadow boundary.

### Overlay Targeting in Dual Mode

When in split mode, overlays target whichever view contains the anchor position. In WYSIWYG-only or source-only mode, overlays target the active view. Inline decorations render in both views when the same content range is visible in both. Floating annotations render once, anchored to the WYSIWYG view by default (where rendered elements have meaningful visual positions), unless the annotation's `Position` falls within a code block or raw HTML region, in which case it anchors to the source view.

### Annotation Lifecycle Across Mode Switches

On mode switch (WYSIWYG↔source), annotations are **re-anchored**:

1. All annotations store their anchor as a canonical `Position` (line/col in markdown text). The markdown text is the same in both modes — it is the canonical representation.
2. The overlay layer is cleared (all annotation DOM elements removed).
3. Each annotation's position is recomputed against the new editor view using the active bridge's `toOffset()` + the new view's `coordsAtPos()`.
4. Annotations are re-rendered in the new coordinate space.
5. If an annotation's anchor falls in a region that has no visual representation in the new mode (e.g., a collapsed section in WYSIWYG), the annotation is hidden until the anchor becomes visible.

Inline decorations (highlights) are similarly re-created: the `EditableTextBridge` base class tracks active highlight ranges by ID. On bridge swap, all highlights are re-applied to the new editor's decoration system.

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

**Session contention:** `beginEditSession()` throws if a session is already active. The error includes the current lock holder and start time for diagnostics:

```typescript
throw new EditSessionActiveError(activeSession.owner, activeSession.startedAt);
// → "Edit session already active (owner: 'claude', since: 2026-10-07T02:46:33Z)"
```

The MCP tool `editor_begin_session` translates this to a structured tool error response: `{ error: "session_active", owner: "claude", since: "2026-10-07T02:46:33Z" }`. The calling LLM can inspect the error and decide whether to wait and retry or proceed differently. No queue — v1 is a simple exclusive lock.

**Cancel semantics:** Human clicks cancel → the session performs a **comprehensive rollback** of all side effects:

1. **Document state:** Restore ProseMirror `EditorState` (or CodeMirror state if in source mode) from the snapshot taken at `beginEditSession()`.
2. **Decorations:** Clear all highlights created during the session. The `EditSession` tracks highlight IDs created after session start; rollback calls `removeHighlight(id)` for each, then restores the pre-session DecorationSet.
3. **Annotations:** Remove all annotations created during the session. The `EditSession` tracks annotation IDs; rollback calls `removeAnnotation(id)` for each, removing the DOM elements from the overlay layer.
4. **Bridge state:** If the user switched editor modes during the session (WYSIWYG↔source), restore the `EDITABLE_TEXT` symbol to the pre-session bridge and re-activate the pre-session view mode.
5. **Position index:** Rebuild the cached position index from the restored document state.

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

Extract generic infrastructure from `blocks-ui/components/document-workbench/src/document-diff.ts` (1217 lines total) into `pages-document-diff`:

**Moves to pages (~1025 lines):**
- CSS styles (~240 lines, all non-thread styles)
- HTML template (`createRenderRoot`, ~35 lines)
- Interfaces/types (`DiffChunk`, `PanelState`, `ScrollAnchor`, ~18 lines)
- Class infrastructure, generic fields, `configure()` (~40 lines)
- LCS line diff engine (`_lineDiff`, ~37 lines)
- Word-level diff and highlights (`_wordDiff`, `_applyWordHighlights`, `_annotateWordDiffs`, ~85 lines)
- Canvas minimap (`_drawDiffMap`, ~25 lines)
- Diff annotation and orchestration (`_annotateRendered`, `_updateDiffMap`, ~56 lines)
- Heading-based scroll sync (`_buildScrollAnchors`, `_setupScrollSync`, `_interp`, `_scrollPercent`, ~69 lines)
- Divider drag (`_setupDividerDrag`, ~19 lines)
- Drop zones (`_setupDropZone`, ~20 lines)
- Diff map click handler (`_onDiffMapClick`, ~15 lines)
- Markdown rendering (`_renderMarkdown`, `_syncPanelContent`, `_syncPanelMeta`, `_fetchFile`, ~40 lines)
- File operations (`selectFile`, `currentPath`, `loadFile`, `loadContent`, ~60 lines)
- Diff navigation (`nextDiff`, `prevDiff`, `_scrollToChunk`, `_nonEqIndices`, `_chunkOutOfView`, ~72 lines)
- View modes (`setViewMode`, `_renderUnified`, `swapPanels`, `getDiffSummary`, ~65 lines)
- Heading search (`_findHeading`, `_normalizeLocation`, `_normHead`, `scrollToLocation`, ~80 lines)
- Section highlight (`highlightSection`, `clearHighlight`, ~25 lines)
- Lifecycle (`connectedCallback` generic wiring, `disconnectedCallback`, `toggleSync`, ~45 lines)

**Stays in blocks-ui (~125 lines of logic, as `DrafthouseDocumentDiff extends PagesDocumentDiff`):**
- `_threadAnchors` field and thread-specific CSS (~13 lines)
- Thread event wiring: `thread-created`, `thread-resolved`, `thread-focused` listeners (~37 lines)
- `_renderThreadGutterMarkers()` (~26 lines)
- Timeline snapshot fetching: `timeline-comparison-changed` → `/api/debate/{sessionId}/snapshot/{index}` (~15 lines)
- Selection-to-thread bridge: `mouseup` → `selection-changed` events with `side: A/B` (~34 lines)

Note: The thread/timeline/selection logic is currently interleaved in `connectedCallback`. The extraction requires refactoring `connectedCallback` into a base method (generic setup) with a subclass override (DraftHouse-specific event wiring). The subclass calls `super.connectedCallback()` and adds its domain-specific listeners.

## 7. Refactoring Existing Code

### ScenarioEditableText → EditableText Rename

1. Move interface to `pages-editor-core`
2. Rename `ScenarioEditableText` → `EditableText`
3. Rename symbol from `scenario-editable-text` → `editable-text`
4. Update `CodeEditorBridge` to extend `EditableTextBridge` from core
5. Update `pages-aria` to import from `pages-editor-core`
6. Update `code-editor-bridge.ts` to remove local interface copy
7. Update `command-executor.ts` and all consumers

**Cross-repo impact analysis for `Symbol.for` key change:**
The string key `'scenario-editable-text'` appears in exactly two source files:
- `pages-code-editor/src/pages-code-editor.ts` line 14 (local `const`)
- `pages-aria/src/executor/editable-text.ts` line 21 (exported `const`)

In `blocks-ui`, the key appears only in vendored copies under `.casehub-packages/` — these are synced from the pages packages on publish and update automatically. No blocks-ui application code uses the raw string key directly; all consumers use the imported `EDITABLE_TEXT` constant. The rename is safe: update both source files atomically, publish the packages, and vendored copies sync on the next `casehub-packages` update.

Prior spec docs and a blog post reference the old key (`2026-09-14-scenario-driven-tutorials-design.md`, `2026-09-14-mdp04-teaching-by-typing.md`) — these are documentation, not runtime code. No update needed, but a note can be added for clarity.

### Breaking Interface Changes

The following changes to `EditableText` are **intentionally breaking**:

| Change | Impact | Migration |
|--------|--------|-----------|
| `highlight()` return `void` → `string` | Callers that type-narrow on `void` return will get a TypeScript error. Existing callers (`command-executor.ts` line 193) ignore the return value — no runtime breakage. | Mechanical: fix any type errors at call sites. |
| New methods: `removeHighlight(id)`, `addAnnotation(...)`, `removeAnnotation(id)`, `clearAnnotations()`, `beginEditSession(owner)`, `endEditSession(session)`, `findText(query)`, `getLine(line)` | All `EditableText` implementors must provide these. Currently one implementor: `CodeEditorBridge`. | Implement the new methods in `CodeEditorBridge`. |
| Style type `'pulse' \| 'underline' \| 'glow'` → `HighlightStyle` (adds `'box'`) | Union expansion — existing callers are unaffected. | None. |
| CSS class `scenario-highlight-*` → `editor-highlight-*` | Internal to shadow DOM. No external CSS can target these. | None — rename is transparent to consumers. |

### CodeEditorBridge Refactor

- Extract shared logic to `EditableTextBridge` in `pages-editor-core`
- `CodeEditorBridge extends EditableTextBridge`
- Engine-specific: CodeMirror `StateEffect`/`StateField`/`Decoration.mark` stays
- Shared: ID generation, highlight/annotation tracking, session management, event emission moves to base

## 8. Testing Strategy

### Unit Tests
- `EditableText` interface compliance tests — shared test suite that both `CodeEditorBridge` and `MarkdownEditorBridge` must pass
- Position index tests — verify `toOffset`/`toPosition` round-trip for paragraphs, headings, lists, tables, code blocks
- MCP tool adapter tests — mock EditableText, verify tool call → method mapping
- Scroll sync engine tests — verify anchor pairing and interpolation
- Diff engine tests — verify LCS, word-level diff (existing `document-diff.test.ts`)

### Integration Tests
- Milkdown mount/unmount lifecycle in LIT shadow DOM
- WYSIWYG ↔ source round-trip (markdown → ProseMirror → markdown):
  - **Acceptance criterion:** semantic equivalence (same remark AST structure after normalizing whitespace). Not string equality — known accepted divergences: trailing whitespace stripping, list indentation normalization (2-space vs 4-space), ATX heading whitespace, table column alignment padding. These are cosmetic and do not affect document meaning.
  - Position mapping fidelity: after round-trip, `toOffset({line: N, col: M})` must resolve to the same document content (not necessarily the same byte offset — structural changes from normalization are allowed).
- Overlay rendering (highlights visible, annotations positioned)
- Edit session lock/unlock/cancel with comprehensive rollback verification (document, decorations, annotations, bridge state)

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