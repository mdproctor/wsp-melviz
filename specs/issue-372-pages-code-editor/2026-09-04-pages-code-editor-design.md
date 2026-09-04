# Design: pages-code-editor with YAML Syntax Highlighting

**Issue:** casehubio/casehub-pages#372
**Date:** 2026-09-04
**Branch:** issue-372-pages-code-editor

## Overview

Three deliverables:

1. **`<pages-code-editor>`** — a reusable Web Component for editing/viewing structured text with syntax highlighting, built on CodeMirror 6
2. **Export promotion** — document `exportDiagram()` availability from `@casehubio/pages-diagram-core` in the graph-renderer consumer guide
3. **Standalone diagram export tool** — an example page composing the code editor with graph canvas and export buttons

The editor is designed for a future where CaseHub YAML and jq expressions have context-aware completion via a Language Server Protocol (LSP) server, and where the same language intelligence powers VS Code and IntelliJ plugins. The LSP server is a separate effort (D8); this branch ships the editor with basic syntax highlighting.

## 1. Component: `<pages-code-editor>`

### Package placement

In `@casehubio/pages-ui-components`, following the established `src/{name}/pages-{name}.ts` pattern. Files:

```
packages/pages-ui-components/src/
  code-editor/
    pages-code-editor.ts       # LitElement wrapping CodeMirror 6
    pages-code-editor.test.ts  # Vitest unit tests
    index.ts                   # Barrel re-export
```

### Dependencies

New npm dependencies for `pages-ui-components`:

- `@codemirror/view` — EditorView, themes, DOM integration
- `@codemirror/state` — EditorState, transactions, extensions
- `@codemirror/language` — language support infrastructure
- `@codemirror/lang-yaml` — YAML Lezer grammar
- `@codemirror/lang-json` — JSON Lezer grammar
- `@codemirror/commands` — standard keybindings (undo, redo, indent)

CodeMirror 6 is modular — these packages total ~40-80KB gzipped. No monolithic bundle.

### Component API

```typescript
class PagesCodeEditor extends LitElement {
  @property({ type: String })
  value = '';

  @property({ type: String })
  language: 'yaml' | 'json' = 'yaml';

  @property({ type: Boolean, reflect: true })
  readonly = false;

  @property({ type: Boolean, attribute: 'line-numbers' })
  lineNumbers = true;

  @property({ type: Number, attribute: 'tab-size' })
  tabSize = 2;

  @property({ attribute: false })
  extensions: Extension[] = [];
}
```

**Registration:** Use the guard-based pattern (`if (!customElements.get(...))`) consistent with the majority of pages-ui-components.

### Properties detail

| Property | Type | Default | Attribute | Description |
|----------|------|---------|-----------|-------------|
| `value` | `string` | `''` | `value` | Document content. Two-way: setting updates the editor; edits update the property. |
| `language` | `'yaml' \| 'json'` | `'yaml'` | `language` | Active Lezer grammar for syntax highlighting. |
| `readonly` | `boolean` | `false` | `readonly` | When true, the editor is not editable. Reflects to attribute for CSS styling (`:host([readonly])`). |
| `lineNumbers` | `boolean` | `true` | `line-numbers` | Show/hide the line number gutter. |
| `tabSize` | `number` | `2` | `tab-size` | Spaces per tab/indent level. |
| `extensions` | `Extension[]` | `[]` | none | CodeMirror extensions for LSP client, custom keybindings, or other plugins. Not settable via HTML attribute. |

### Events

| Event | When | Detail |
|-------|------|--------|
| `input` | Every document change | `{ value: string }` — the new document content |
| `change` | Editor loses focus after content changed | `{ value: string }` — the final document content |

Both events dispatch with `bubbles: true, composed: true` to cross shadow DOM boundaries, consistent with other pages-ui-components.

### CodeMirror integration

**Mounting:** CodeMirror's `EditorView` is created in `firstUpdated()` and mounted into a container `<div>` inside the shadow root. The view is destroyed in `disconnectedCallback()`.

```typescript
override firstUpdated() {
  const container = this.shadowRoot!.querySelector('.cm-host')!;
  this.editorView = new EditorView({
    state: this.createEditorState(),
    parent: container,
  });
}

override disconnectedCallback() {
  this.editorView?.destroy();
  super.disconnectedCallback();
}
```

**State management:** When `value` changes externally (property set), dispatch a transaction to replace the document content — but only if the new value differs from the current document to avoid cursor disruption:

```typescript
override updated(changed: PropertyValues) {
  if (changed.has('value') && this.editorView) {
    const current = this.editorView.state.doc.toString();
    if (current !== this.value) {
      this.editorView.dispatch({
        changes: { from: 0, to: current.length, insert: this.value },
      });
    }
  }
}
```

**Language switching:** When the `language` property changes, reconfigure the editor state with the appropriate Lezer grammar. Use `EditorView.dispatch()` with a `reconfigure` effect.

**Extensions composition:** The editor composes its base extensions with user-provided extensions:

```
base extensions = [
  lineNumbers() | [],           // conditional on lineNumbers property
  languageExtension(language),  // yaml() or json()
  indentUnit.of(' '.repeat(tabSize)),
  pagesTheme,                   // maps --pages-* tokens to CodeMirror theme
  EditorState.readOnly.of(readonly),
  updateListener,               // dispatches input/change events
  ...extensions,                // user-provided (LSP client, etc.)
]
```

### Theming

CodeMirror 6 themes are defined via `EditorView.theme()`. The pages theme maps `--pages-*` CSS custom properties to CodeMirror's theme selectors:

```typescript
const pagesTheme = EditorView.theme({
  '&': {
    fontFamily: 'var(--pages-font-mono, monospace)',
    fontSize: 'var(--pages-font-size-sm, 13px)',
    backgroundColor: 'var(--pages-neutral-1, #fafafa)',
    color: 'var(--pages-neutral-12, #1a1a1a)',
  },
  '.cm-gutters': {
    backgroundColor: 'var(--pages-neutral-2, #f5f5f5)',
    color: 'var(--pages-neutral-8, #8b8b8b)',
    borderRight: '1px solid var(--pages-neutral-4, #e0e0e0)',
  },
  '.cm-activeLine': {
    backgroundColor: 'var(--pages-neutral-3, #eeeeee)',
  },
  // ... syntax highlighting via tags
});
```

Syntax highlighting uses CodeMirror's `HighlightStyle` with Lezer tags, mapped to `--pages-*` semantic colors (accent for keys, success for strings, danger for errors, info for comments).

### Shadow DOM considerations

CodeMirror injects styles into its parent container. Inside shadow DOM, this works naturally — the styles are scoped to the shadow root. Key considerations:

- **`:host { display: block }`** — prevent the LitElement display:inline gotcha (GE-20260810-8df51b)
- **`:host { position: relative }`** — CodeMirror needs a positioned container
- **Height:** `:host` should have a default min-height but be resizable via CSS. Default to `height: 300px` with `resize: vertical; overflow: auto`.
- **Theme token inheritance:** CSS custom properties cascade through shadow DOM (GE-20260712-f5b872). The component does NOT redeclare `--pages-*` tokens internally — it inherits them from the document-level theme (GE-20260706-9335b9).

### Accessibility

CodeMirror 6 provides built-in accessibility:

- `role="textbox"` with `aria-multiline="true"` on the content area
- Screen reader announcements for cursor movement, selection, and content changes
- Keyboard navigation: standard editing keys, Escape to blur
- Tab key: by default CodeMirror traps Tab for indentation. Provide an Escape-then-Tab pattern to allow keyboard users to leave the editor (CodeMirror's default `keymap` includes this).

The component adds:
- `aria-label` via a `label` property or slot
- `aria-readonly` when `readonly` is true

## 2. Export Promotion

`exportDiagram()` remains in `@casehubio/pages-diagram-core`. The function depends on React Flow DOM internals (`.react-flow__viewport`) and `html-to-image@1.11.11` — both belong in diagram-core.

The dependency direction is `pages-diagram-core → graph-renderer` (not the reverse). Re-exporting from graph-renderer would create a circular dependency.

**Action:** Add a section to `docs/guides/consumer-guide.md` documenting that diagram export is available via:

```typescript
import { exportDiagram } from '@casehubio/pages-diagram-core';
```

With usage examples for SVG and PNG export.

## 3. Standalone Diagram Export Tool

A new example page in `examples/` that provides a standalone diagram editing and export environment.

### Layout

```
┌──────────────────────────────────────────────────────┐
│  Diagram Export Tool                    [SVG] [PNG]   │
├─────────────────────────┬────────────────────────────┤
│                         │                            │
│  <pages-code-editor>    │  <pages-graph-canvas>      │
│  language="yaml"        │                            │
│                         │                            │
│  (YAML input)           │  (diagram preview)         │
│                         │                            │
│                         │                            │
└─────────────────────────┴────────────────────────────┘
```

Side-by-side split: code editor on the left, graph canvas on the right. Export buttons (SVG, PNG) in the toolbar.

### Data flow

1. User edits YAML in `<pages-code-editor>`
2. `input` event fires with the new YAML content
3. Parent page parses YAML via `js-yaml` (or the pages YAML parser)
4. Parsed model is fed to `<pages-graph-canvas>` as graph data
5. Canvas renders the diagram
6. Export buttons call `exportDiagram(canvasElement, nodes, format)` from `pages-diagram-core`

### File structure

```
examples/src/pages/
  diagram-export-tool.ts    # The page component
```

Registered as a route in the existing examples router. Accessible at `localhost:8080/diagram-export-tool` (or similar path in the examples dev server).

### Error handling

Invalid YAML displays a diagnostic banner between the toolbar and the editor/canvas split, showing the parse error with line number. The previous valid diagram remains rendered — the canvas is not cleared on parse errors.

## 4. Future: Language Server (D8, separate issue)

Not in scope for #372, but the architecture is designed for it:

- **`extensions` property** on `<pages-code-editor>` accepts a CodeMirror LSP client extension
- **LSP server** (TypeScript, runs in Node for IDEs or web worker for browser) provides:
  - CaseHub YAML schema completion (component types, properties, pipeline stages)
  - jq expression completion and validation
  - Diagnostics (invalid YAML structure, unknown component types, type mismatches)
- **IDE plugins** (VS Code, IntelliJ) use their native editors + the same LSP server
- **Visual form view** — a complementary guided form UI over the same YAML document model, using the LSP schema for field definitions and validation. The `value` property is the shared state between text and form views.

A new GitHub issue will be created for the LSP server effort during implementation planning.

## Testing Strategy

### Unit tests (Vitest + jsdom)

- Property reflection: setting `value`, `language`, `readonly`, `lineNumbers`, `tabSize`
- Event dispatch: `input` fires on content change, `change` fires on blur
- Language switching: reconfigures grammar without losing content
- Readonly mode: editor rejects input when `readonly` is true
- Extensions: custom extensions are composed with base extensions

### Integration tests

- CodeMirror mounts and renders within shadow DOM
- Theme tokens apply correctly (custom properties cascade through shadow boundary)
- The standalone export tool page loads and renders both editor and canvas

### Not tested in #372

- LSP integration (future)
- Context-aware completion (future)
- Visual form view (future)

## References

- [packages/pages-aria/src/controller/yaml-highlighter.ts] — existing YAML tokenizer (stays in pages-aria)
- [packages/pages-aria/src/controller/scenario-yaml-viewer.ts] — existing YAML viewer component
- [packages/pages-diagram-core/src/diagram-export.ts] — exportDiagram() function
- [packages/pages-ui-components/] — target package for the new component
- [GE-20260706-9335b9] — Shadow DOM CSS custom property overrides
- [GE-20260712-f5b872] — CSS custom properties cascade through shadow DOM
- [GE-20260810-8df51b] — LitElement defaults to display:inline
- [GE-20260818-f0257a] — Shadow-aware CSS injection with WeakMap ref-counting
- [GE-20260823-590f19] — Three-point registration for new component types
- [GE-20260813-674be0] — YAML desugarer drops unknown component props silently
- [casehubio/casehub-pages#372] — Issue: pages-code-editor component with YAML syntax highlighting
