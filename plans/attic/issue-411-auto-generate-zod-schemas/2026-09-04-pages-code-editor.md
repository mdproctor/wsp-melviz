# pages-code-editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #372 — pages-code-editor component with YAML syntax highlighting
**Issue group:** #372

**Goal:** Build a reusable CodeMirror 6 Web Component for editing structured text with syntax highlighting, promote exportDiagram to graph-renderer, and create a standalone diagram export tool.

**Architecture:** Standalone `@casehubio/pages-code-editor` package wrapping CodeMirror 6 in a LitElement with shadow DOM. The component uses Compartment-based dynamic reconfiguration for all properties. Export promotion moves `exportDiagram()` from pages-diagram-core to graph-renderer. A standalone HTML page in examples/ composes both for diagram editing and export.

**Tech Stack:** Lit 3.x, CodeMirror 6 (@codemirror/view, state, language, lang-yaml, lang-json, commands), html-to-image 1.11.11, Vitest, Playwright

## Global Constraints

- CodeMirror packages: `@codemirror/view`, `@codemirror/state`, `@codemirror/language`, `@codemirror/lang-yaml`, `@codemirror/lang-json`, `@codemirror/commands` — all `^6.x`
- `html-to-image` pinned at `1.11.11` — later versions have rendering bugs
- `lit` version: `^3.3.3` — matches all other packages
- Component registration: guard-based pattern (`if (!customElements.get(...))`)
- Events: plain `Event` with `bubbles: true, composed: true` — consumers read `event.target.value`
- No `--pages-*` token redeclaration inside shadow DOM — inherit from document-level theme
- `:host { display: block }` always — prevent inline display gotcha
- CM6 scroll management: `.cm-scroller { overflow: auto }`, `:host { overflow: hidden }` — never `overflow: auto` on `:host`

---

## Batch 1: Code editor package — scaffold and core component

### Task 1: Scaffold @casehubio/pages-code-editor package

**Files:**
- Create: `packages/pages-code-editor/package.json`
- Create: `packages/pages-code-editor/tsconfig.json`
- Create: `packages/pages-code-editor/tsconfig.build.json`
- Create: `packages/pages-code-editor/src/index.ts`
- Modify: `package.json` (root — add to `build:packages` chain)

**Interfaces:**
- Consumes: nothing
- Produces: Package infrastructure that Task 2 builds on. `src/index.ts` will re-export `PagesCodeEditor` once it exists.

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-code-editor",
  "version": "0.1.0",
  "description": "CaseHub Pages code editor — Lit Web Component wrapping CodeMirror 6 with syntax highlighting",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "sideEffects": ["./dist/index.js", "./src/index.ts"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "clean": "rimraf dist .typecheck",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@codemirror/commands": "^6.8.0",
    "@codemirror/lang-json": "^6.0.0",
    "@codemirror/lang-yaml": "^6.1.0",
    "@codemirror/language": "^6.10.0",
    "@codemirror/state": "^6.5.0",
    "@codemirror/view": "^6.36.0",
    "lit": "^3.3.3"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "jsdom": "^26.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": []
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "emitDeclarationOnly": false,
    "composite": false
  },
  "exclude": ["**/*.test.ts"]
}
```

- [ ] **Step 4: Create src/index.ts (placeholder)**

```typescript
export {};
```

- [ ] **Step 5: Add to root build:packages chain**

In root `package.json`, add `npm run --workspace=packages/pages-code-editor build &&` after the `pages-table` build step (position 14 in the chain). The package has no workspace dependencies, so it can build early.

- [ ] **Step 6: Install dependencies**

Run: `npm install` from the repo root to resolve the new workspace package and install CodeMirror dependencies.

- [ ] **Step 7: Verify the package builds**

Run: `npm run --workspace=packages/pages-code-editor build`
Expected: Success with empty `dist/index.js` and `dist/index.d.ts`

- [ ] **Step 8: Commit**

```bash
git add packages/pages-code-editor/ package.json package-lock.json
git commit -m "feat(#372): scaffold @casehubio/pages-code-editor package

Standalone package for CodeMirror 6 Lit web component, following
the pages-table precedent for heavyweight standalone components.

Refs casehubio/casehub-pages#372"
```

### Task 2: Implement PagesCodeEditor component

**Files:**
- Create: `packages/pages-code-editor/src/pages-code-editor.ts`
- Modify: `packages/pages-code-editor/src/index.ts`

**Interfaces:**
- Consumes: CodeMirror 6 APIs (`EditorView`, `EditorState`, `Compartment`, `lineNumbers`, `yaml`, `json`, `indentUnit`, `defaultKeymap`, `history`, `historyKeymap`)
- Produces: `PagesCodeEditor` class (LitElement), registered as `<pages-code-editor>`. Properties: `value`, `language`, `readonly`, `lineNumbers`, `tabSize`, `label`, `extensions`. Events: `input`, `change`.

- [ ] **Step 1: Write the failing test — component instantiation**

Create `packages/pages-code-editor/src/pages-code-editor.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';

describe('PagesCodeEditor', () => {
  it('should export PagesCodeEditor class', async () => {
    const { PagesCodeEditor } = await import('./pages-code-editor.js');
    expect(PagesCodeEditor).toBeDefined();
    expect(typeof PagesCodeEditor).toBe('function');
  });

  it('should have correct default property values', async () => {
    const { PagesCodeEditor } = await import('./pages-code-editor.js');
    const el = new PagesCodeEditor();
    expect(el.value).toBe('');
    expect(el.language).toBe('yaml');
    expect(el.readonly).toBe(false);
    expect(el.lineNumbers).toBe(true);
    expect(el.tabSize).toBe(2);
    expect(el.label).toBeUndefined();
    expect(el.extensions).toEqual([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm run --workspace=packages/pages-code-editor test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesCodeEditor**

Create `packages/pages-code-editor/src/pages-code-editor.ts`:

```typescript
import { LitElement, html, css, type PropertyValues } from 'lit';
import { property } from 'lit/decorators.js';
import { EditorView, lineNumbers as cmLineNumbers, keymap } from '@codemirror/view';
import { EditorState, Compartment, type Extension } from '@codemirror/state';
import { indentUnit } from '@codemirror/language';
import { yaml } from '@codemirror/lang-yaml';
import { json } from '@codemirror/lang-json';
import { defaultKeymap, history, historyKeymap, indentWithTab } from '@codemirror/commands';
import { syntaxHighlighting, HighlightStyle } from '@codemirror/language';
import { tags } from '@lezer/highlight';

const pagesHighlightStyle = HighlightStyle.define([
  { tag: tags.propertyName, color: 'var(--pages-accent-11, #3451b2)' },
  { tag: tags.string, color: 'var(--pages-success-11, #18794e)' },
  { tag: tags.number, color: 'var(--pages-warning-11, #ad5700)' },
  { tag: tags.bool, color: 'var(--pages-warning-11, #ad5700)' },
  { tag: tags.null, color: 'var(--pages-neutral-8, #8b8b8b)' },
  { tag: tags.comment, color: 'var(--pages-info-9, #0091ff)', fontStyle: 'italic' },
  { tag: tags.punctuation, color: 'var(--pages-neutral-9, #6f6f6f)' },
  { tag: tags.keyword, color: 'var(--pages-danger-11, #cd2b31)' },
]);

const pagesTheme = EditorView.theme({
  '&': {
    height: '100%',
    fontFamily: 'var(--pages-font-mono, ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, monospace)',
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
  '.cm-activeLineGutter': {
    backgroundColor: 'var(--pages-neutral-3, #eeeeee)',
  },
  '.cm-activeLine': {
    backgroundColor: 'var(--pages-neutral-3, #eeeeee)',
  },
  '&.cm-focused': {
    outline: '2px solid var(--pages-accent-8, #adc8ff)',
    outlineOffset: '-2px',
  },
  '.cm-cursor': {
    borderLeftColor: 'var(--pages-neutral-12, #1a1a1a)',
  },
  '.cm-selectionBackground': {
    backgroundColor: 'var(--pages-accent-4, #e1ecff) !important',
  },
});

function languageExtension(lang: string): Extension {
  return lang === 'json' ? json() : yaml();
}

export class PagesCodeEditor extends LitElement {
  static override styles = css`
    :host {
      display: block;
      position: relative;
      height: 300px;
      overflow: hidden;
      resize: vertical;
      border: 1px solid var(--pages-neutral-6, #d0d0d0);
      border-radius: var(--pages-radius-md, 6px);
    }
    :host([readonly]) {
      resize: none;
    }
    .cm-host {
      height: 100%;
    }
  `;

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

  private _editorView: EditorView | null = null;
  private _pendingCreate = false;
  private _suppressUpdate = false;

  private _languageCompartment = new Compartment();
  private _readonlyCompartment = new Compartment();
  private _lineNumbersCompartment = new Compartment();
  private _tabSizeCompartment = new Compartment();
  private _labelCompartment = new Compartment();
  private _extensionsCompartment = new Compartment();

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
    if (this._editorView && !this._suppressUpdate) {
      this._syncProperties(changed);
    }
  }

  override disconnectedCallback() {
    this._editorView?.destroy();
    this._editorView = null;
    super.disconnectedCallback();
  }

  override render() {
    return html`<div class="cm-host"></div>`;
  }

  private _createEditor() {
    this._pendingCreate = false;
    const container = this.shadowRoot!.querySelector('.cm-host')!;
    this._editorView = new EditorView({
      state: EditorState.create({
        doc: this.value,
        extensions: [
          this._lineNumbersCompartment.of(
            this.lineNumbers ? cmLineNumbers() : []
          ),
          this._languageCompartment.of(languageExtension(this.language)),
          this._tabSizeCompartment.of(indentUnit.of(' '.repeat(this.tabSize))),
          this._readonlyCompartment.of(EditorState.readOnly.of(this.readonly)),
          this._labelCompartment.of(
            EditorView.contentAttributes.of(
              this.label ? { 'aria-label': this.label } : {}
            )
          ),
          keymap.of([...defaultKeymap, ...historyKeymap, indentWithTab]),
          history(),
          pagesTheme,
          syntaxHighlighting(pagesHighlightStyle),
          EditorView.updateListener.of((update) => {
            if (update.docChanged) {
              this._suppressUpdate = true;
              this.value = update.state.doc.toString();
              this._suppressUpdate = false;
              this.dispatchEvent(
                new Event('input', { bubbles: true, composed: true })
              );
            }
            if (update.focusChanged && !update.view.hasFocus) {
              this.dispatchEvent(
                new Event('change', { bubbles: true, composed: true })
              );
            }
          }),
          this._extensionsCompartment.of(this.extensions),
        ],
      }),
      parent: container,
    });
  }

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
          languageExtension(this.language)
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
          this.lineNumbers ? cmLineNumbers() : []
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
}

if (!customElements.get('pages-code-editor')) {
  customElements.define('pages-code-editor', PagesCodeEditor);
}
```

- [ ] **Step 4: Update src/index.ts barrel export**

```typescript
export { PagesCodeEditor } from './pages-code-editor.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npm run --workspace=packages/pages-code-editor test`
Expected: 2 tests PASS

- [ ] **Step 6: Verify the package builds**

Run: `npm run --workspace=packages/pages-code-editor build`
Expected: Success — `dist/index.js`, `dist/index.d.ts`, `dist/pages-code-editor.js`, `dist/pages-code-editor.d.ts`

- [ ] **Step 7: Commit**

```bash
git add packages/pages-code-editor/src/
git commit -m "feat(#372): implement PagesCodeEditor — CodeMirror 6 Lit web component

LitElement wrapping CodeMirror 6 with shadow DOM, Compartment-based
dynamic reconfiguration for all properties, pages theme tokens,
YAML/JSON language support, and guard-based custom element registration.

Refs casehubio/casehub-pages#372"
```

---

## Batch 2: Export promotion — move exportDiagram to graph-renderer

### Task 3: Move exportDiagram to graph-renderer

**Files:**
- Move: `packages/pages-diagram-core/src/diagram-export.ts` → `packages/graph-renderer/src/diagram-export.ts` (use `ide_move_file`)
- Move: `packages/pages-diagram-core/src/diagram-export.test.ts` → `packages/graph-renderer/src/diagram-export.test.ts` (use `ide_move_file`)
- Modify: `packages/graph-renderer/package.json` (add `html-to-image` dependency)
- Modify: `packages/graph-renderer/src/index.ts` (add exports)
- Modify: `packages/pages-diagram-core/src/index.ts` (remove exports, re-import from graph-renderer)
- Modify: `packages/pages-diagram-core/package.json` (remove `html-to-image` dependency)
- Modify: `packages/pages-diagram-core/src/diagram-base-mixin.ts` (update import)

**Interfaces:**
- Consumes: `html-to-image` (`toSvg`, `toPng`)
- Produces: `exportDiagram`, `computeNodeBounds`, `computeExportViewport`, `ExportFormat`, `ExportBounds`, `ExportViewport` — all exported from `@casehubio/graph-renderer`

- [ ] **Step 1: Run existing diagram-export tests to confirm green baseline**

Run: `npm run --workspace=packages/pages-diagram-core test -- --run src/diagram-export.test.ts`
Expected: PASS — 7 test cases for `computeNodeBounds` and `computeExportViewport`

- [ ] **Step 2: Add html-to-image dependency to graph-renderer**

In `packages/graph-renderer/package.json`, add to `dependencies`:

```json
"html-to-image": "1.11.11"
```

Run: `npm install` from repo root.

- [ ] **Step 3: Move diagram-export.ts to graph-renderer**

Use `ide_move_file` to move:
- `packages/pages-diagram-core/src/diagram-export.ts` → `packages/graph-renderer/src/diagram-export.ts`
- `packages/pages-diagram-core/src/diagram-export.test.ts` → `packages/graph-renderer/src/diagram-export.test.ts`

- [ ] **Step 4: Add exports to graph-renderer index.ts**

Add to `packages/graph-renderer/src/index.ts`:

```typescript
export {
  exportDiagram,
  computeNodeBounds,
  computeExportViewport,
  type ExportFormat,
  type ExportBounds,
  type ExportViewport,
} from './diagram-export.js';
```

- [ ] **Step 5: Update diagram-core index.ts — re-export from graph-renderer**

In `packages/pages-diagram-core/src/index.ts`, replace the diagram-export imports:

Change:
```typescript
export { exportDiagram, computeNodeBounds, computeExportViewport } from './diagram-export.js';
export type { ExportFormat, ExportBounds, ExportViewport } from './diagram-export.js';
```

To:
```typescript
export { exportDiagram, computeNodeBounds, computeExportViewport } from '@casehubio/graph-renderer';
export type { ExportFormat, ExportBounds, ExportViewport } from '@casehubio/graph-renderer';
```

- [ ] **Step 6: Update diagram-base-mixin.ts import**

In `packages/pages-diagram-core/src/diagram-base-mixin.ts`, change:

```typescript
import { exportDiagram } from './diagram-export.js';
```

To:

```typescript
import { exportDiagram } from '@casehubio/graph-renderer';
```

- [ ] **Step 7: Remove html-to-image from diagram-core dependencies**

In `packages/pages-diagram-core/package.json`, remove `"html-to-image": "1.11.11"` from `dependencies`.

Run: `npm install` from repo root.

- [ ] **Step 8: Run tests in both packages**

Run: `npm run --workspace=packages/graph-renderer test -- --run src/diagram-export.test.ts`
Expected: PASS — same 7 test cases now in graph-renderer

Run: `npm run --workspace=packages/pages-diagram-core test`
Expected: PASS — diagram-core tests still pass with re-exported functions

- [ ] **Step 9: Build both packages**

Run: `npm run build:packages`
Expected: Full build chain succeeds. `graph-renderer` builds at position 11 with the new export. `pages-diagram-core` at position 18 imports from `graph-renderer`.

- [ ] **Step 10: Commit**

```bash
git add packages/graph-renderer/ packages/pages-diagram-core/ package-lock.json
git commit -m "feat(#372): move exportDiagram from pages-diagram-core to graph-renderer

Consolidates React Flow DOM coupling in the rendering layer.
exportDiagram queries .react-flow__viewport — this belongs in the
package that owns React Flow rendering. html-to-image@1.11.11 moves
with it. pages-diagram-core re-exports for backward compatibility.

Refs casehubio/casehub-pages#372"
```

### Task 4: Update consumer guide documentation

**Files:**
- Modify: `docs/guides/consumer-guide.md`

**Interfaces:**
- Consumes: nothing
- Produces: Updated documentation showing `exportDiagram` import from `@casehubio/graph-renderer`, fixed stale `casehub-diagram-canvas` reference

- [ ] **Step 1: Add export documentation section**

In `docs/guides/consumer-guide.md`, add a section documenting diagram export:

```markdown
### Diagram Export

Export diagrams as SVG or PNG using `exportDiagram()` from `@casehubio/graph-renderer`:

\`\`\`typescript
import { exportDiagram } from '@casehubio/graph-renderer';

// Export the canvas as PNG
const canvasElement = document.querySelector('pages-graph-canvas');
exportDiagram(canvasElement, nodes, 'png');

// Export as SVG
exportDiagram(canvasElement, nodes, 'svg', 'my-diagram.svg');
\`\`\`

The export captures the React Flow viewport at 2x pixel ratio (PNG) or as a cleaned SVG.
Uses `html-to-image@1.11.11` (pinned for stability).
```

- [ ] **Step 2: Fix stale component name reference**

Search `docs/guides/consumer-guide.md` for `casehub-diagram-canvas` or `CasehubDiagramCanvas` and replace with `pages-graph-canvas` / `GraphCanvas`.

- [ ] **Step 3: Commit**

```bash
git add docs/guides/consumer-guide.md
git commit -m "docs(#372): add exportDiagram docs and fix stale component name in consumer guide

Refs casehubio/casehub-pages#372"
```

---

## Batch 3: Standalone diagram export tool and build integration

### Task 5: Create standalone diagram export tool

**Files:**
- Create: `examples/diagram-export-tool.html`
- Create: `examples/src/diagram-export-tool.ts`
- Modify: `examples/webpack.config.js` (add entry point and aliases)

**Interfaces:**
- Consumes: `PagesCodeEditor` from `@casehubio/pages-code-editor`, `GraphCanvas` from `@casehubio/graph-renderer`, `exportDiagram` from `@casehubio/graph-renderer`, `yaml` package for parsing
- Produces: Standalone HTML page at `localhost:8080/diagram-export-tool.html`

- [ ] **Step 1: Create diagram-export-tool.html**

Create `examples/diagram-export-tool.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Diagram Export Tool</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { height: 100%; font-family: system-ui, -apple-system, sans-serif; }
  </style>
</head>
<body>
  <diagram-export-tool></diagram-export-tool>
  <script src="diagram-export-tool.js"></script>
</body>
</html>
```

- [ ] **Step 2: Create diagram-export-tool.ts entry point**

Create `examples/src/diagram-export-tool.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { parse } from 'yaml';
import '@casehubio/pages-code-editor';
import { exportDiagram } from '@casehubio/graph-renderer';
import '@casehubio/graph-renderer';

const DEFAULT_YAML = `nodes:
  - id: start
    label: Start
    position: { x: 0, y: 0 }
  - id: process
    label: Process
    position: { x: 200, y: 0 }
  - id: end
    label: End
    position: { x: 400, y: 0 }
edges:
  - source: start
    target: process
  - source: process
    target: end
`;

@customElement('diagram-export-tool')
class DiagramExportTool extends LitElement {
  static override styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }
    .toolbar {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 8px 16px;
      background: var(--pages-neutral-2, #f5f5f5);
      border-bottom: 1px solid var(--pages-neutral-4, #e0e0e0);
    }
    .toolbar h1 {
      font-size: 16px;
      font-weight: 600;
      flex: 1;
    }
    .toolbar button {
      padding: 6px 12px;
      border: 1px solid var(--pages-neutral-6, #d0d0d0);
      border-radius: 4px;
      background: white;
      cursor: pointer;
      font-size: 13px;
    }
    .toolbar button:hover {
      background: var(--pages-neutral-3, #eeeeee);
    }
    .error-banner {
      padding: 8px 16px;
      background: var(--pages-danger-3, #ffe5e5);
      color: var(--pages-danger-11, #cd2b31);
      font-size: 13px;
      font-family: monospace;
    }
    .content {
      display: flex;
      flex: 1;
      min-height: 0;
    }
    pages-code-editor {
      flex: 1;
      height: auto;
      border: none;
      border-radius: 0;
      border-right: 1px solid var(--pages-neutral-4, #e0e0e0);
      resize: none;
    }
    .canvas-container {
      flex: 1;
      position: relative;
    }
    pages-graph-canvas {
      width: 100%;
      height: 100%;
    }
  `;

  @state() private _yamlContent = DEFAULT_YAML;
  @state() private _graphData: unknown = null;
  @state() private _parseError: string | null = null;

  override connectedCallback() {
    super.connectedCallback();
    this._parseYaml(this._yamlContent);
  }

  private _parseYaml(yamlStr: string) {
    try {
      this._graphData = parse(yamlStr);
      this._parseError = null;
    } catch (e: unknown) {
      this._parseError = e instanceof Error ? e.message : String(e);
    }
  }

  private _onInput(e: Event) {
    const editor = e.target as HTMLElement & { value: string };
    this._yamlContent = editor.value;
    this._parseYaml(this._yamlContent);
  }

  private _export(format: 'svg' | 'png') {
    const canvas = this.shadowRoot?.querySelector('pages-graph-canvas');
    if (canvas && this._graphData) {
      const data = this._graphData as { nodes?: Array<{ id: string }> };
      exportDiagram(canvas as HTMLElement, data.nodes ?? [], format);
    }
  }

  override render() {
    return html`
      <div class="toolbar">
        <h1>Diagram Export Tool</h1>
        <button @click=${() => this._export('svg')}>SVG</button>
        <button @click=${() => this._export('png')}>PNG</button>
      </div>
      ${this._parseError
        ? html`<div class="error-banner">${this._parseError}</div>`
        : ''}
      <div class="content">
        <pages-code-editor
          .value=${this._yamlContent}
          language="yaml"
          label="YAML diagram source"
          @input=${this._onInput}
        ></pages-code-editor>
        <div class="canvas-container">
          <pages-graph-canvas
            .graphData=${this._graphData}
          ></pages-graph-canvas>
        </div>
      </div>
    `;
  }
}
```

Note: The exact `graphData` property name and shape for `pages-graph-canvas` should be verified via `ide_find_class` on `GraphCanvas` — the property may be named differently. Adjust accordingly.

- [ ] **Step 3: Add webpack entry point and alias**

In `examples/webpack.config.js`, add to the `entry` object:

```javascript
"diagram-export-tool": path.resolve(__dirname, "src/diagram-export-tool.ts"),
```

Add to the `resolve.alias` map:

```javascript
"@casehubio/pages-code-editor": path.resolve(__dirname, "../packages/pages-code-editor/src"),
```

- [ ] **Step 4: Start dev server and verify**

Run: `npm run --workspace=examples dev` (or equivalent)
Navigate to: `http://localhost:8080/diagram-export-tool.html`
Expected: Side-by-side editor and canvas. YAML editing updates the diagram preview. SVG/PNG buttons trigger download.

- [ ] **Step 5: Commit**

```bash
git add examples/
git commit -m "feat(#372): add standalone diagram export tool

Side-by-side YAML editor (pages-code-editor) and graph canvas with
SVG/PNG export buttons. Separate webpack entry point, independent
of the examples gallery.

Refs casehubio/casehub-pages#372"
```

### Task 6: Playwright integration tests

**Files:**
- Create: `examples/tests/code-editor.spec.ts`

**Interfaces:**
- Consumes: `PagesCodeEditor` via the examples dev server, diagram export tool page
- Produces: Playwright test suite covering CodeMirror rendering, events, property changes, and the standalone tool

- [ ] **Step 1: Write Playwright test for code editor rendering**

Create `examples/tests/code-editor.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('PagesCodeEditor', () => {
  test('renders CodeMirror inside shadow DOM', async ({ page }) => {
    await page.goto('/diagram-export-tool.html');
    const editor = page.locator('diagram-export-tool')
      .locator('pages-code-editor');
    await expect(editor).toBeVisible();

    const cmEditor = editor.locator('.cm-editor');
    await expect(cmEditor).toBeVisible();

    const cmContent = editor.locator('.cm-content');
    await expect(cmContent).toHaveAttribute('contenteditable', 'true');
  });

  test('displays YAML with syntax highlighting', async ({ page }) => {
    await page.goto('/diagram-export-tool.html');
    const editor = page.locator('diagram-export-tool')
      .locator('pages-code-editor');
    await expect(editor).toBeVisible();

    const content = editor.locator('.cm-content');
    await expect(content).toContainText('nodes');
  });

  test('readonly mode prevents editing', async ({ page }) => {
    await page.goto('/diagram-export-tool.html');

    await page.evaluate(() => {
      const tool = document.querySelector('diagram-export-tool');
      const editor = tool?.shadowRoot?.querySelector('pages-code-editor');
      if (editor) (editor as any).readonly = true;
    });

    const editor = page.locator('diagram-export-tool')
      .locator('pages-code-editor');
    const cmContent = editor.locator('.cm-content');
    await expect(cmContent).toHaveAttribute('contenteditable', 'false');
  });
});

test.describe('Diagram Export Tool', () => {
  test('page loads with editor and canvas', async ({ page }) => {
    await page.goto('/diagram-export-tool.html');
    await expect(page.locator('diagram-export-tool')).toBeVisible();

    const editor = page.locator('diagram-export-tool')
      .locator('pages-code-editor');
    await expect(editor).toBeVisible();
  });

  test('shows error banner for invalid YAML', async ({ page }) => {
    await page.goto('/diagram-export-tool.html');

    const editor = page.locator('diagram-export-tool')
      .locator('pages-code-editor');
    await editor.locator('.cm-content').click();
    await page.keyboard.press('Control+A');
    await page.keyboard.type('invalid: yaml: : :');

    const errorBanner = page.locator('diagram-export-tool')
      .locator('.error-banner');
    await expect(errorBanner).toBeVisible({ timeout: 5000 });
  });
});
```

- [ ] **Step 2: Run Playwright tests**

Run: `npx playwright test examples/tests/code-editor.spec.ts`
Expected: All tests PASS

- [ ] **Step 3: Commit**

```bash
git add examples/tests/code-editor.spec.ts
git commit -m "test(#372): Playwright integration tests for code editor and export tool

Refs casehubio/casehub-pages#372"
```

---

## References

- [specs/issue-372-pages-code-editor/2026-09-04-pages-code-editor-design.md] — design spec this plan implements
- [packages/pages-table/package.json] — precedent for standalone heavyweight package
- [packages/graph-renderer/src/index.ts] — target for exportDiagram re-export
- [packages/pages-diagram-core/src/diagram-export.ts] — source of exportDiagram
- [packages/pages-diagram-core/src/diagram-base-mixin.ts:~9] — existing caller of exportDiagram
- [examples/webpack.config.js] — examples build configuration
- [GE-20260810-8df51b] — LitElement display:inline gotcha
- [GE-20260706-9335b9] — Shadow DOM CSS custom property overrides
- [GE-20260712-f5b872] — CSS custom properties cascade through shadow DOM
- [casehubio/casehub-pages#372] — focal issue
- [casehubio/casehub-pages#407] — LSP server (future, out of scope)
