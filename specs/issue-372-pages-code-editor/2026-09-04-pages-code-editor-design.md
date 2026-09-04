# Design: pages-code-editor with YAML Syntax Highlighting

**Issue:** casehubio/casehub-pages#372
**Date:** 2026-09-04
**Branch:** issue-372-pages-code-editor

## Overview

Three deliverables:

1. **`<pages-code-editor>`** — a reusable Web Component for editing/viewing structured text with syntax highlighting, built on CodeMirror 6
2. **Export promotion** — move `exportDiagram()` from `@casehubio/pages-diagram-core` to `@casehubio/graph-renderer`, consolidating React Flow DOM coupling in the rendering layer
3. **Standalone diagram export tool** — a standalone HTML entry point composing the code editor with graph canvas and export buttons

The editor is designed for a future where CaseHub YAML and jq expressions have context-aware completion via a Language Server Protocol (LSP) server, and where the same language intelligence powers VS Code and IntelliJ plugins. The LSP server is a separate effort (D8, tracked as casehubio/casehub-pages#407); this branch ships the editor with basic syntax highlighting.

## 1. Component: `<pages-code-editor>`

### Package placement

Standalone package `@casehubio/pages-code-editor`, following the `@casehubio/pages-table` precedent — heavyweight components with their own dependency profile and architectural complexity live in their own packages. `pages-ui-components` is a collection of lightweight form primitives (`PagesInput`, `PagesSelect`, `PagesTextarea`, etc.) whose only third-party dependency is `lit`. Adding six `@codemirror/*` packages (40-80KB gzipped) would fundamentally change its character.

```
packages/pages-code-editor/
  package.json
  tsconfig.json
  tsconfig.build.json
  src/
    pages-code-editor.ts       # LitElement wrapping CodeMirror 6
    pages-code-editor.test.ts  # Vitest unit tests
    index.ts                   # Barrel re-export
```

Package structure:

```json
{
  "name": "@casehubio/pages-code-editor",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "sideEffects": ["./dist/index.js", "./src/index.ts"],
  "dependencies": {
    "@codemirror/view": "^6.x",
    "@codemirror/state": "^6.x",
    "@codemirror/language": "^6.x",
    "@codemirror/lang-yaml": "^6.x",
    "@codemirror/lang-json": "^6.x",
    "@codemirror/commands": "^6.x",
    "lit": "^3.3.3"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "jsdom": "^26.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  }
}
```

The code editor is for programmatic use — it is not registered as a component type in the YAML dashboard component model (no three-point registration needed).

### Dependencies

New npm dependencies for `pages-code-editor`:

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

  @property({ type: String })
  label: string | undefined;

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
| `label` | `string \| undefined` | `undefined` | `label` | Accessible label for the editor. Sets `aria-label` on the CodeMirror content area via `EditorView.contentAttributes`. |
| `extensions` | `Extension[]` | `[]` | none | CodeMirror extensions for LSP client, custom keybindings, or other plugins. Not settable via HTML attribute. |

### Events

| Event | When | Detail |
|-------|------|--------|
| `input` | Every document change | None — read `event.target.value` |
| `change` | Editor loses focus after content changed | None — read `event.target.value` |

Both events dispatch as plain `Event` with `bubbles: true, composed: true`, consistent with the `pages-ui-components` convention (`PagesInput`, `PagesTextarea`). Consumers read the value from `event.target.value`, where `event.target` is the `<pages-code-editor>` element (shadow DOM re-targets the event). The component's `value` property is updated before the event fires.

### CodeMirror integration

**Mounting:** CodeMirror's `EditorView` is created when the component connects to the DOM and the shadow root is available. To handle the Lit lifecycle correctly — where `disconnectedCallback()` fires on every DOM removal but `firstUpdated()` fires only once — creation uses a guard pattern:

```typescript
private _editorView: EditorView | null = null;
private _pendingCreate = false;

override connectedCallback() {
  super.connectedCallback();
  if (!this._editorView) {
    this._pendingCreate = true;
  }
}

override firstUpdated() {
  this._createEditor();
}

override updated(changed: PropertyValues) {
  if (this._pendingCreate && !this._editorView) {
    this._createEditor();
  }
  if (this._editorView) {
    this._syncProperties(changed);
  }
}

override disconnectedCallback() {
  this._editorView?.destroy();
  this._editorView = null;
  super.disconnectedCallback();
}

private _createEditor() {
  this._pendingCreate = false;
  const container = this.shadowRoot!.querySelector('.cm-host')!;
  this._editorView = new EditorView({
    state: this.createEditorState(),
    parent: container,
  });
}
```

This ensures:
- First creation happens after `firstUpdated()` when the shadow DOM is ready
- Re-insertion after removal (conditional rendering, tab switching) recreates the editor via `updated()`
- No double-creation (guard on `_editorView` nullness)

**Dynamic property reconfiguration:** When properties change externally, `_syncProperties()` dispatches the appropriate reconfiguration transactions. Each reconfigurable property uses a CodeMirror `Compartment` — the mechanism for dynamically replacing extensions at runtime:

```typescript
private _languageCompartment = new Compartment();
private _readonlyCompartment = new Compartment();
private _lineNumbersCompartment = new Compartment();
private _tabSizeCompartment = new Compartment();
private _labelCompartment = new Compartment();
private _extensionsCompartment = new Compartment();

private _syncProperties(changed: PropertyValues) {
  if (!this._editorView) return;

  if (changed.has('value')) {
    const current = this._editorView.state.doc.toString();
    if (current !== this.value) {
      this._editorView.dispatch({
        changes: { from: 0, to: current.length, insert: this.value },
      });
    }
  }

  if (changed.has('language')) {
    this._editorView.dispatch({
      effects: this._languageCompartment.reconfigure(
        this.language === 'json' ? json() : yaml()
      ),
    });
  }

  if (changed.has('readonly')) {
    this._editorView.dispatch({
      effects: this._readonlyCompartment.reconfigure(
        EditorState.readOnly.of(this.readonly)
      ),
    });
  }

  if (changed.has('lineNumbers')) {
    this._editorView.dispatch({
      effects: this._lineNumbersCompartment.reconfigure(
        this.lineNumbers ? lineNumbers() : []
      ),
    });
  }

  if (changed.has('tabSize')) {
    this._editorView.dispatch({
      effects: this._tabSizeCompartment.reconfigure(
        indentUnit.of(' '.repeat(this.tabSize))
      ),
    });
  }

  if (changed.has('label')) {
    this._editorView.dispatch({
      effects: this._labelCompartment.reconfigure(
        EditorView.contentAttributes.of(
          this.label ? { 'aria-label': this.label } : {}
        )
      ),
    });
  }

  if (changed.has('extensions')) {
    this._editorView.dispatch({
      effects: this._extensionsCompartment.reconfigure(this.extensions),
    });
  }
}
```

**Extensions composition:** The editor composes its base extensions with user-provided extensions:

```
base extensions = [
  lineNumbersCompartment.of(lineNumbers() | []),
  languageCompartment.of(yaml() | json()),
  tabSizeCompartment.of(indentUnit.of('  ')),
  readonlyCompartment.of(EditorState.readOnly.of(false)),
  labelCompartment.of(EditorView.contentAttributes.of(label ? { 'aria-label': label } : {})),
  pagesTheme,
  updateListener,
  extensionsCompartment.of(extensions),
]
```

### Theming

CodeMirror 6 themes are defined via `EditorView.theme()`. The pages theme maps `--pages-*` CSS custom properties to CodeMirror's theme selectors:

```typescript
const pagesTheme = EditorView.theme({
  '&': {
    height: '100%',
    fontFamily: 'var(--pages-font-mono, monospace)',
    fontSize: 'var(--pages-font-size-sm, 13px)',
    backgroundColor: 'var(--pages-neutral-1, #fafafa)',
    color: 'var(--pages-neutral-12, #1a1a1a)',
  },
  '.cm-scroller': {
    overflow: 'auto',
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
- **Height:** `:host` provides outer dimensions: `height: 300px; overflow: hidden; resize: vertical;` (`overflow: hidden` enables the resize handle). The CodeMirror theme sets `'&': { height: '100%' }` and `'.cm-scroller': { overflow: 'auto' }` so CM6's internal scroller handles scrolling — this preserves virtual rendering, scroll-to-cursor, and sticky line number gutter. Do NOT put `overflow: auto` on `:host` — it breaks CM6's scroll management.
- **Theme token inheritance:** CSS custom properties cascade through shadow DOM (GE-20260712-f5b872). The component does NOT redeclare `--pages-*` tokens internally — it inherits them from the document-level theme (GE-20260706-9335b9).

### Accessibility

CodeMirror 6 provides built-in accessibility:

- `role="textbox"` with `aria-multiline="true"` on the content area
- Screen reader announcements for cursor movement, selection, and content changes
- Keyboard navigation: standard editing keys, Escape to blur
- Tab key: by default CodeMirror traps Tab for indentation. Provide an Escape-then-Tab pattern to allow keyboard users to leave the editor (CodeMirror's default `keymap` includes this).

The component adds:
- `aria-label` via the `label` property, applied to the CodeMirror content area via `EditorView.contentAttributes`
- `aria-readonly` when `readonly` is true

### Relationship to existing editors

The pages monorepo has three existing editor/viewer components:

| Component | Package | Role | Relationship to pages-code-editor |
|-----------|---------|------|-----------------------------------|
| `PagesPromptEditor` | `pages-diagram-core` | Lightweight textarea for diagram property editing | Stays as-is. Zero third-party deps, serves its purpose. The code editor is an upgrade path for cases needing syntax highlighting, not a replacement. |
| `PagesJsonViewer` | `pages-diagram-core` | Read-only `<pre>` JSON display | Stays as-is. Trivially simple, no CodeMirror needed. |
| `PagesScenarioYamlViewer` | `pages-aria` | Scenario-aware YAML viewer with step highlighting, drag/resize, guide tab | Could use `pages-code-editor` as its rendering engine in a future refactor, but this is a separate effort — the viewer is deeply coupled to the scenario system (`ScenarioConnectionController`, step line mapping, guide tab). |

`yaml-highlighter.ts` (in `pages-aria`) provides the custom YAML tokenizer used by `PagesScenarioYamlViewer`. It remains until any future migration of that component to CodeMirror.

No consolidation or breaking changes to existing components in #372 scope.

## 2. Export Promotion

Move `exportDiagram()` from `@casehubio/pages-diagram-core` to `@casehubio/graph-renderer`, as requested by issue #372.

**Rationale:** `exportDiagram()` queries `.react-flow__viewport` — this is React Flow DOM coupling that belongs in the package that owns React Flow rendering (`graph-renderer`). The `html-to-image` dependency moves with it. This consolidates all React Flow DOM access in one package and removes `html-to-image` from `pages-diagram-core`'s dependency list.

**Dependency safety:** The dependency direction is `pages-diagram-core → graph-renderer` (diagram-core depends on graph-renderer, not the reverse). Moving `exportDiagram()` INTO `graph-renderer` means diagram-core imports it from there — the dependency direction is unchanged, no cycle is created.

**What moves:** `exportDiagram()`, `computeNodeBounds()`, `computeExportViewport()`, `ExportBounds`, `ExportViewport`, `ExportFormat` types, and the `html-to-image` dependency.

**Diagram-core update:** Remove `diagram-export.ts` and `diagram-export.test.ts` (58 lines, 7 test cases for `computeNodeBounds` and `computeExportViewport`). Remove `html-to-image` from dependencies. Update the existing caller in `diagram-base-mixin.ts` (line ~9: `import { exportDiagram } from './diagram-export.js'` → `import { exportDiagram } from '@casehubio/graph-renderer'`).

**Version pin:** Maintain `html-to-image@1.11.11` (pinned — later versions have rendering bugs, per issue #372).

**Build script:** Add `@casehubio/pages-code-editor` to the root `package.json` `build:packages` script. The existing `graph-renderer` already builds before `diagram-core` in the chain, so the export move does not affect build order.

**Consumer guide update:** Add export documentation to `docs/guides/consumer-guide.md` referencing `@casehubio/graph-renderer`:

```typescript
import { exportDiagram } from '@casehubio/graph-renderer';
```

Fix the stale component name reference in the consumer guide: currently says `<casehub-diagram-canvas>` (`CasehubDiagramCanvas`) but the actual component is `<pages-graph-canvas>` (`GraphCanvas`).

## 3. Standalone Diagram Export Tool

A standalone HTML entry point in `examples/` that provides a diagram editing and export environment. Per issue #372: "A standalone HTML entry point (not embedded in a shell app)."

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
2. `input` event fires — parent reads `event.target.value`
3. Parent page parses YAML via the `yaml` package (v2.x — the YAML 1.2 parser used throughout the monorepo)
4. Parsed model is fed to `<pages-graph-canvas>` as graph data
5. Canvas renders the diagram
6. Export buttons call `exportDiagram(canvasElement, nodes, format)` from `@casehubio/graph-renderer`

### File structure

```
examples/
  src/
    diagram-export-tool.ts    # Entry point and page component
  diagram-export-tool.html    # Standalone HTML page
```

The tool is a separate webpack entry point, independent of the examples gallery:

```javascript
// webpack.config.js additions
entry: {
  "casehub-bundle": path.resolve(__dirname, "src/casehub-entry.ts"),
  "diagram-export-tool": path.resolve(__dirname, "src/diagram-export-tool.ts"),
}
```

Accessible at `localhost:8080/diagram-export-tool.html` during development.

The examples gallery (`app.js`) is a vanilla JS YAML dashboard gallery driven by `samples.json` and hash-based navigation — it has no TypeScript SPA router and no `pages/` directory structure. The diagram export tool is a separate standalone page, not a gallery sample.

### Error handling

Invalid YAML displays a diagnostic banner between the toolbar and the editor/canvas split, showing the parse error with line number. The previous valid diagram remains rendered — the canvas is not cleared on parse errors.

## 4. Future: Language Server (D8)

Not in scope for #372, but the architecture is designed for it:

- **`extensions` property** on `<pages-code-editor>` accepts a CodeMirror LSP client extension
- **LSP server** (TypeScript, runs in Node for IDEs or web worker for browser) provides:
  - CaseHub YAML schema completion (component types, properties, pipeline stages)
  - jq expression completion and validation
  - Diagnostics (invalid YAML structure, unknown component types, type mismatches)
- **IDE plugins** (VS Code, IntelliJ) use their native editors + the same LSP server
- **Visual form view** — a complementary guided form UI over the same YAML document model, using the LSP schema for field definitions and validation. The `value` property is the shared state between text and form views.

Tracked as casehubio/casehub-pages#407.

## Testing Strategy

### Unit tests (Vitest + jsdom)

- Property reflection: setting `value`, `language`, `readonly`, `lineNumbers`, `tabSize`, `label`
- Extension composition: custom extensions are composed with base extensions
- Compartment reconfiguration: property changes dispatch correct effects

### Integration tests (Playwright)

CodeMirror 6 uses `contenteditable`, `Selection`, `Range`, `getComputedStyle`, and `MutationObserver` — several of which jsdom doesn't fully support. Tests involving actual editor rendering require a real browser:

- CodeMirror mounts and renders within shadow DOM
- Event dispatch: `input` fires on content change, `change` fires on blur
- Language switching: reconfigures grammar without losing content
- Readonly mode: editor rejects input when `readonly` is true
- Theme tokens apply correctly (custom properties cascade through shadow boundary)
- Re-insertion after DOM removal: editor recreates correctly
- The standalone export tool page loads and renders both editor and canvas

The existing Playwright config (`examples/playwright.config.ts`) provides the infrastructure.

### Not tested in #372

- LSP integration (future, #407)
- Context-aware completion (future, #407)
- Visual form view (future)

## References

- [packages/pages-aria/src/controller/yaml-highlighter.ts] — existing YAML tokenizer (stays in pages-aria)
- [packages/pages-aria/src/controller/scenario-yaml-viewer.ts] — existing YAML viewer component
- [packages/pages-diagram-core/src/diagram-export.ts] — exportDiagram() function (moving to graph-renderer)
- [GE-20260706-9335b9] — Shadow DOM CSS custom property overrides
- [GE-20260712-f5b872] — CSS custom properties cascade through shadow DOM
- [GE-20260810-8df51b] — LitElement defaults to display:inline
- [GE-20260818-f0257a] — Shadow-aware CSS injection with WeakMap ref-counting
- [GE-20260813-674be0] — YAML desugarer drops unknown component props silently
- [casehubio/casehub-pages#372] — Issue: pages-code-editor component with YAML syntax highlighting
- [casehubio/casehub-pages#407] — Issue: LSP server for CaseHub YAML and jq (D8)
