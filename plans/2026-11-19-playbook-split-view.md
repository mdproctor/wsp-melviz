# Playbook Split-View Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #720 — Split-view playbook editor
**Issue group:** #720

**Goal:** Add a split-view playbook editing experience with bidirectional click-sync between YAML editor and step viewer, consumable by both the web gallery and the IntelliJ plugin via JCEF.

**Architecture:** Five layers — (1) enhance StepPositionScanner for hierarchical nesting, (2) add cursor-change callback to EditableTextBridge, (3) add click events to step viewer cards, (4) wire them together in a split-view container, (5) IntelliJ FileEditorProvider with JCEF preview.

**Tech Stack:** TypeScript, Lit web components, yaml-core, Monaco (via pages-code-editor), Kotlin/JCEF (IntelliJ plugin)

## Global Constraints

- All new components use Lit 3.x `@customElement` pattern
- Follow existing test patterns: vitest for TS, gallery.html for visual verification
- No new package — split-view lives in `pages-step-viewer`
- IntelliJ plugin requires JDK 21 and LSP4IJ dependency
- Step addresses use dot-separated hierarchical format: `"1.then.0"`

---

## Batch 1: Position Scanner + Editor Callback

### Task 1: Enhance StepPositionScanner for hierarchical step addresses

**Files:**
- Modify: `packages/yaml-core/src/annotation/step-position-scanner.ts`
- Modify: `packages/yaml-core/src/annotation/step-position-scanner.test.ts`

**Interfaces:**
- Consumes: nothing (foundation layer)
- Produces:
  ```typescript
  interface StepLineInfo {
    line: number;       // 0-indexed start line
    endLine: number;    // 0-indexed exclusive end line
    address: string;    // hierarchical: "0", "1.then.0", "2.catch.1"
  }
  // Static method signature unchanged:
  StepPositionScanner.scan(rawYaml: string): StepLineInfo[]
  // New helper:
  StepPositionScanner.findByLine(infos: StepLineInfo[], line: number): StepLineInfo | undefined
  StepPositionScanner.findByAddress(infos: StepLineInfo[], address: string): StepLineInfo | undefined
  ```

- [ ] **Step 1: Write failing tests for flat steps with endLine**

```typescript
import { StepPositionScanner } from "./step-position-scanner.js";

describe("StepPositionScanner hierarchical", () => {
  it("scans flat steps with endLine", () => {
    const yaml = [
      "- action: step-a",
      "  with:",
      "    key: val",
      "- action: step-b",
      "  with:",
      "    other: val",
    ].join("\n");
    const result = StepPositionScanner.scan(yaml);
    expect(result).toEqual([
      { line: 0, endLine: 3, address: "0" },
      { line: 3, endLine: 6, address: "1" },
    ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/yaml-core/src/annotation/step-position-scanner.test.ts`
Expected: FAIL — endLine not in current return type

- [ ] **Step 3: Write failing tests for nested if-else**

```typescript
it("scans if-else with nested steps", () => {
  const yaml = [
    "- action: step-0",
    "- if: condition",
    "  then:",
    "    - action: then-step-0",
    "    - action: then-step-1",
    "  else:",
    "    - action: else-step-0",
    "- action: step-2",
  ].join("\n");
  const result = StepPositionScanner.scan(yaml);
  expect(result).toEqual([
    { line: 0, endLine: 1, address: "0" },
    { line: 1, endLine: 7, address: "1" },
    { line: 3, endLine: 4, address: "1.then.0" },
    { line: 4, endLine: 5, address: "1.then.1" },
    { line: 6, endLine: 7, address: "1.else.0" },
    { line: 7, endLine: 8, address: "2" },
  ]);
});
```

- [ ] **Step 4: Write failing tests for try-catch, match, swim-lane, race**

```typescript
it("scans try-catch with body and catch blocks", () => {
  const yaml = [
    "- try:",
    "    - action: body-0",
    "    - action: body-1",
    "  catch:",
    "    - action: catch-0",
  ].join("\n");
  const result = StepPositionScanner.scan(yaml);
  expect(result).toEqual([
    { line: 0, endLine: 5, address: "0" },
    { line: 1, endLine: 2, address: "0.body.0" },
    { line: 2, endLine: 3, address: "0.body.1" },
    { line: 4, endLine: 5, address: "0.catch.0" },
  ]);
});

it("scans match with named arms", () => {
  const yaml = [
    "- match: status",
    "  arms:",
    "    open:",
    "      - action: handle-open",
    "    closed:",
    "      - action: handle-closed",
  ].join("\n");
  const result = StepPositionScanner.scan(yaml);
  expect(result).toEqual([
    { line: 0, endLine: 6, address: "0" },
    { line: 3, endLine: 4, address: "0.match.open.0" },
    { line: 5, endLine: 6, address: "0.match.closed.0" },
  ]);
});

it("scans parallel/swim-lane steps", () => {
  const yaml = [
    "- parallel:",
    "    agent:",
    "      - action: agent-step",
    "    reviewer:",
    "      - action: review-step",
  ].join("\n");
  const result = StepPositionScanner.scan(yaml);
  expect(result).toEqual([
    { line: 0, endLine: 5, address: "0" },
    { line: 2, endLine: 3, address: "0.lane.agent.0" },
    { line: 4, endLine: 5, address: "0.lane.reviewer.0" },
  ]);
});
```

- [ ] **Step 5: Write failing tests for lookup helpers**

```typescript
it("findByLine returns deepest matching step", () => {
  const infos: StepLineInfo[] = [
    { line: 0, endLine: 1, address: "0" },
    { line: 1, endLine: 7, address: "1" },
    { line: 3, endLine: 5, address: "1.then.0" },
  ];
  expect(StepPositionScanner.findByLine(infos, 0)).toEqual(infos[0]);
  expect(StepPositionScanner.findByLine(infos, 3)).toEqual(infos[2]);
  expect(StepPositionScanner.findByLine(infos, 1)).toEqual(infos[1]);
  expect(StepPositionScanner.findByLine(infos, 10)).toBeUndefined();
});

it("findByAddress returns matching step", () => {
  const infos: StepLineInfo[] = [
    { line: 0, endLine: 1, address: "0" },
    { line: 3, endLine: 5, address: "1.then.0" },
  ];
  expect(StepPositionScanner.findByAddress(infos, "1.then.0")).toEqual(infos[1]);
  expect(StepPositionScanner.findByAddress(infos, "missing")).toBeUndefined();
});
```

- [ ] **Step 6: Implement hierarchical scanner**

Replace the existing `scan` method body with an indentation-aware walker. Track nesting by indentation level. When encountering a list item (`- `), check the first key for control-flow types (`if`, `try`, `match`, `parallel`, `race`, `select`). For control-flow steps, recursively scan sub-blocks (`then:`, `else:`, `catch:`, `arms:`, lanes) using indentation boundaries to delimit each block.

Add `endLine` to `StepLineInfo`. Add static `findByLine` (returns deepest step whose `[line, endLine)` contains the query line) and `findByAddress` (simple filter by address string).

- [ ] **Step 7: Run all tests**

Run: `npx vitest run packages/yaml-core/src/annotation/step-position-scanner.test.ts`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/annotation/step-position-scanner.ts packages/yaml-core/src/annotation/step-position-scanner.test.ts
git commit -m "feat(#720): hierarchical step position scanner with nesting support"
```

### Task 2: Add onCursorChange to EditableText and EditableTextBridge

**Files:**
- Modify: `packages/pages-editor-core/src/types.ts`
- Modify: `packages/pages-editor-core/src/editable-text-bridge.ts`
- Modify: `packages/pages-editor-core/src/index.ts` (export Disposable if not already)
- Create: `packages/pages-editor-core/src/editable-text-bridge.test.ts`

**Interfaces:**
- Consumes: existing `EditableText` interface, `Position` type
- Produces:
  ```typescript
  interface EditableText {
    // ... existing methods ...
    onCursorChange?(callback: (position: Position) => void): Disposable;
  }
  interface Disposable { dispose(): void; }
  ```

- [ ] **Step 1: Write failing test for cursor change callback**

```typescript
import { describe, it, expect, vi } from "vitest";
import { EditableTextBridge } from "./editable-text-bridge.js";
import type { Position } from "./types.js";

class TestBridge extends EditableTextBridge {
  private _cursor: Position = { line: 0, col: 0 };
  getText() { return ""; }
  getLine() { return ""; }
  getLineCount() { return 1; }
  setContent() {}
  findText() { return []; }
  insertText() {}
  replaceRange() {}
  deleteRange() {}
  setCursor(line: number, col: number) { this._cursor = { line, col }; }
  getCursor() { return this._cursor; }
  protected applyHighlight() { return "h1"; }
  protected removeHighlightDecoration() {}
  protected clearHighlightDecorations() {}
  createReader() { return { nextLine: () => null, peek: () => null, hasNext: () => false, position: () => ({ line: 0, col: 0 }) }; }
  beginEditSession(owner: string) { return { id: "s1", owner, changes: [] }; }
  endEditSession() {}
}

describe("EditableTextBridge cursor change", () => {
  it("fires callback on notifyCursorChange", () => {
    const bridge = new TestBridge();
    const callback = vi.fn();
    bridge.onCursorChange(callback);
    bridge.notifyCursorChange({ line: 5, col: 0 });
    expect(callback).toHaveBeenCalledWith({ line: 5, col: 0 });
  });

  it("disposable unsubscribes", () => {
    const bridge = new TestBridge();
    const callback = vi.fn();
    const disposable = bridge.onCursorChange(callback);
    disposable.dispose();
    bridge.notifyCursorChange({ line: 5, col: 0 });
    expect(callback).not.toHaveBeenCalled();
  });

  it("supports multiple listeners", () => {
    const bridge = new TestBridge();
    const cb1 = vi.fn();
    const cb2 = vi.fn();
    bridge.onCursorChange(cb1);
    bridge.onCursorChange(cb2);
    bridge.notifyCursorChange({ line: 3, col: 0 });
    expect(cb1).toHaveBeenCalledOnce();
    expect(cb2).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: FAIL — `onCursorChange` and `notifyCursorChange` don't exist

- [ ] **Step 3: Implement onCursorChange in EditableTextBridge**

Add `onCursorChange` as an optional method to the `EditableText` interface in `types.ts`. Implement it in `EditableTextBridge` as a listener list with a `notifyCursorChange(pos: Position)` protected method that subclasses call.

```typescript
// In EditableTextBridge:
private _cursorListeners: Array<(pos: Position) => void> = [];

onCursorChange(callback: (position: Position) => void): Disposable {
  this._cursorListeners.push(callback);
  return {
    dispose: () => {
      const idx = this._cursorListeners.indexOf(callback);
      if (idx >= 0) this._cursorListeners.splice(idx, 1);
    },
  };
}

protected notifyCursorChange(position: Position): void {
  for (const cb of this._cursorListeners) cb(position);
}
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-editor-core/src/editable-text-bridge.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-editor-core/src/types.ts packages/pages-editor-core/src/editable-text-bridge.ts packages/pages-editor-core/src/editable-text-bridge.test.ts packages/pages-editor-core/src/index.ts
git commit -m "feat(#720): add onCursorChange callback to EditableTextBridge"
```

---

## Batch 2: Step Viewer Click Events

### Task 3: Add click events and step-address attributes to step viewer

**Files:**
- Modify: `packages/pages-step-viewer/src/step-dispatch.ts`
- Modify: `packages/pages-step-viewer/src/pages-step-viewer.ts`
- Create: `packages/pages-step-viewer/src/step-click.ts`
- Create: `packages/pages-step-viewer/src/step-click.test.ts`

**Interfaces:**
- Consumes: `ResolvedStep` from yaml-core, `renderStepContent` function
- Produces:
  ```typescript
  // step-click.ts
  interface StepClickDetail {
    address: string;
    originalEvent: MouseEvent;
  }
  function assignStepAddresses(steps: ResolvedStep[], prefix?: string): Map<ResolvedStep, string>
  // Event name: "step-click" — CustomEvent<StepClickDetail>
  ```

- [ ] **Step 1: Write failing test for address assignment**

```typescript
import { describe, it, expect } from "vitest";
import { assignStepAddresses } from "./step-click.js";
import type { ResolvedStep } from "@casehubio/yaml-core";

function step(kind: string, children?: Record<string, ResolvedStep[]>): ResolvedStep {
  return { kind, qualifiedName: kind, action: {}, ...children } as unknown as ResolvedStep;
}

describe("assignStepAddresses", () => {
  it("assigns flat sequential addresses", () => {
    const steps = [step("plugin"), step("plugin"), step("plugin")];
    const map = assignStepAddresses(steps);
    expect(map.get(steps[0])).toBe("0");
    expect(map.get(steps[1])).toBe("1");
    expect(map.get(steps[2])).toBe("2");
  });

  it("assigns nested addresses for if-else", () => {
    const thenStep = step("plugin");
    const elseStep = step("plugin");
    const ifStep = {
      kind: "if-else",
      qualifiedName: "if-else",
      action: {},
      thenSteps: [thenStep],
      elseSteps: [elseStep],
    } as unknown as ResolvedStep;
    const map = assignStepAddresses([ifStep]);
    expect(map.get(ifStep)).toBe("0");
    expect(map.get(thenStep)).toBe("0.then.0");
    expect(map.get(elseStep)).toBe("0.else.0");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-step-viewer/src/step-click.test.ts`
Expected: FAIL — module doesn't exist

- [ ] **Step 3: Implement assignStepAddresses**

Walk the step tree recursively. For each step kind that has children (`if-else` → `thenSteps`/`elseSteps`, `try-catch-finally` → `bodySteps`/`catchSteps`, `match` → `arms`, `parallel` → `lanes`, `race` → `branches`), recurse with the appropriate prefix. Build a `Map<ResolvedStep, string>` mapping each step object to its address.

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-step-viewer/src/step-click.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Add click handlers to renderStepContent**

Modify `step-dispatch.ts`: wrap each `renderStepContent` call in an outer `div` with `data-step-address` and a `@click` handler:

```typescript
export function renderStepContentWithAddress(
  step: ResolvedStep,
  address: string,
): TemplateResult {
  return html`<div
    class="step-clickable"
    data-step-address=${address}
    @click=${(e: MouseEvent) => {
      e.currentTarget?.dispatchEvent(new CustomEvent('step-click', {
        bubbles: true, composed: true,
        detail: { address, originalEvent: e },
      }));
    }}
  >${renderStepContent(step)}</div>`;
}
```

Update `pages-step-viewer.ts` render method to use `assignStepAddresses` and `renderStepContentWithAddress`.

- [ ] **Step 6: Add step-active CSS class support**

In `pages-step-viewer.ts`, add an `activeAddress` reactive property. In render, apply `step-active` class to the div whose address matches. Add CSS for the highlight:

```css
.step-clickable.step-active {
  outline: 2px solid var(--pages-primary, #3b82f6);
  outline-offset: 2px;
  border-radius: 4px;
}
```

Add a public method `setActiveStep(address: string | null)`.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-step-viewer/src/step-click.ts packages/pages-step-viewer/src/step-click.test.ts packages/pages-step-viewer/src/step-dispatch.ts packages/pages-step-viewer/src/pages-step-viewer.ts
git commit -m "feat(#720): step viewer click events with hierarchical addresses"
```

---

## Batch 3: Split-View Container

### Task 4: Create playbook-split-view component

**Files:**
- Create: `packages/pages-step-viewer/src/playbook-split-view.ts`
- Create: `packages/pages-step-viewer/src/playbook-split-view.test.ts`
- Modify: `packages/pages-step-viewer/package.json` (add `@casehubio/pages-editor-core` dependency)
- Modify: `packages/pages-step-viewer/src/index.ts` (export new component)
- Modify: `packages/pages-step-viewer/gallery.html` (add split-view demo section)

**Interfaces:**
- Consumes:
  - `StepPositionScanner.scan()`, `StepPositionScanner.findByLine()`, `StepPositionScanner.findByAddress()` from yaml-core
  - `EditableText.onCursorChange()`, `EditableText.setCursor()`, `EditableText.highlightRange()` from editor-core
  - `PagesStepViewer.setActiveStep(address)`, `step-click` event from step-viewer
  - `Walker.resolve(steps, catalog)` from yaml-core
- Produces:
  ```typescript
  @customElement('playbook-split-view')
  class PlaybookSplitView extends LitElement {
    @property() yaml: string;
    @property({ type: Number }) splitRatio: number; // 0.0-1.0, default 0.5
    editor: EditableText;  // bound after first render
  }
  ```

- [ ] **Step 1: Write failing test for YAML→viewer pipeline**

```typescript
import { describe, it, expect } from "vitest";
import { PlaybookSplitView } from "./playbook-split-view.js";

describe("PlaybookSplitView", () => {
  it("is defined as custom element", () => {
    const el = document.createElement("playbook-split-view");
    expect(el).toBeInstanceOf(PlaybookSplitView);
  });
});
```

- [ ] **Step 2: Implement PlaybookSplitView skeleton**

Lit component with two panels: left panel hosts a `<pages-code-editor>` slot, right panel hosts a `<pages-step-viewer>`. CSS grid layout with resizable splitter. On `yaml` property change (debounced 300ms), parse → resolve → update step viewer.

```typescript
@customElement('playbook-split-view')
export class PlaybookSplitView extends LitElement {
  @property() yaml = '';
  @property({ type: Number }) splitRatio = 0.5;
  @state() private _steps: ResolvedStep[] = [];
  @state() private _parseError: string | null = null;
  @state() private _activeAddress: string | null = null;

  private _positionInfos: StepLineInfo[] = [];
  private _debounceTimer?: ReturnType<typeof setTimeout>;
  private _cursorDisposable?: Disposable;

  // ... render with left/right panels, error display, sync wiring
}
```

- [ ] **Step 3: Wire click-to-editor sync**

Listen for `step-click` on the viewer panel. On click:
1. `StepPositionScanner.findByAddress(_positionInfos, detail.address)`
2. If found: `editor.setCursor(info.line, 0)` + `editor.highlightRange(info.line, info.endLine, 'step-highlight')`

- [ ] **Step 4: Wire cursor-to-viewer sync**

Call `editor.onCursorChange(pos => ...)`. On cursor change:
1. `StepPositionScanner.findByLine(_positionInfos, pos.line)`
2. If found: `viewer.setActiveStep(info.address)` + scroll step into view

- [ ] **Step 5: Add to gallery**

Add a "Split View" section to `gallery.html` with a sample playbook YAML string and a `<playbook-split-view>` element. Verify visual rendering.

- [ ] **Step 6: Run all step-viewer tests**

Run: `npx vitest run packages/pages-step-viewer/`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-step-viewer/src/playbook-split-view.ts packages/pages-step-viewer/src/playbook-split-view.test.ts packages/pages-step-viewer/src/index.ts packages/pages-step-viewer/package.json packages/pages-step-viewer/gallery.html
git commit -m "feat(#720): playbook-split-view component with bidirectional sync"
```

---

## Batch 4: IntelliJ Plugin

### Task 5: Add FileEditorProvider with JCEF step viewer preview

**Files:**
- Create: `plugins/intellij/src/main/kotlin/io/casehub/intellij/PlaybookEditorProvider.kt`
- Create: `plugins/intellij/src/main/kotlin/io/casehub/intellij/PlaybookPreviewPanel.kt`
- Modify: `plugins/intellij/src/main/resources/META-INF/plugin.xml` (register provider)
- Create: `plugins/intellij/src/main/resources/playbook-preview.html` (minimal HTML host)
- Modify: `plugins/intellij/build.gradle.kts` (add JCEF dependency if needed)

**Interfaces:**
- Consumes: `pages-step-viewer` bundle (step viewer + scanner + sync adapter), IntelliJ `TextEditorWithPreview` API, `JBCefBrowser`, `JBCefJSQuery`
- Produces: FileEditorProvider registered for `*.playbook.yaml`

- [ ] **Step 1: Create playbook-preview.html**

Minimal HTML page that imports the step-viewer bundle and exposes `window.syncCursorLine(line)` and `window.onStepClick(callback)`:

```html
<!DOCTYPE html>
<html>
<head>
  <script src="step-viewer-bundle.js"></script>
  <style>
    body { margin: 0; background: var(--pages-surface-0, #0f172a); }
    pages-step-viewer { display: block; padding: 16px; }
  </style>
</head>
<body>
  <pages-step-viewer id="viewer"></pages-step-viewer>
  <script>
    const viewer = document.getElementById('viewer');
    let positionInfos = [];

    window.updateYaml = function(yaml) {
      try {
        const steps = resolveYaml(yaml);
        viewer.steps = steps;
        positionInfos = StepPositionScanner.scan(yaml);
      } catch (e) {
        // show error
      }
    };

    window.syncCursorLine = function(line) {
      const info = StepPositionScanner.findByLine(positionInfos, line);
      if (info) viewer.setActiveStep(info.address);
    };

    viewer.addEventListener('step-click', (e) => {
      const info = StepPositionScanner.findByAddress(positionInfos, e.detail.address);
      if (info && window._stepClickQuery) {
        window._stepClickQuery(String(info.line));
      }
    });
  </script>
</body>
</html>
```

- [ ] **Step 2: Create PlaybookPreviewPanel**

```kotlin
class PlaybookPreviewPanel(project: Project) : Disposable {
    val browser: JBCefBrowser = JBCefBrowserBuilder().build()
    private val stepClickQuery = JBCefJSQuery.create(browser as JBCefBrowserBase)

    var onStepClick: ((line: Int) -> Unit)? = null

    init {
        stepClickQuery.addHandler { lineStr ->
            val line = lineStr.toIntOrNull() ?: return@addHandler null
            onStepClick?.invoke(line)
            null
        }
        val html = loadResource("/playbook-preview.html")
        browser.loadHTML(html)
        browser.jbCefClient.addLoadHandler(/* inject stepClickQuery JS */)
    }

    fun updateYaml(yaml: String) {
        val escaped = yaml.replace("\\", "\\\\").replace("`", "\\`")
        browser.cefBrowser.executeJavaScript("window.updateYaml(`$escaped`)", "", 0)
    }

    fun syncCursorLine(line: Int) {
        browser.cefBrowser.executeJavaScript("window.syncCursorLine($line)", "", 0)
    }

    override fun dispose() { stepClickQuery.dispose() }
}
```

- [ ] **Step 3: Create PlaybookEditorProvider**

```kotlin
class PlaybookEditorProvider : TextEditorWithPreview.Provider() {
    override fun accept(project: Project, file: VirtualFile): Boolean =
        file.name.endsWith(".playbook.yaml")

    override fun createEditor(project: Project, file: VirtualFile): FileEditor {
        val textEditor = TextEditorProvider.getInstance().createEditor(project, file) as TextEditor
        val preview = PlaybookPreviewPanel(project)

        // Sync editor → preview
        textEditor.editor.caretModel.addCaretListener(object : CaretListener {
            override fun caretPositionChanged(e: CaretEvent) {
                preview.syncCursorLine(e.newPosition.line)
            }
        })

        // Sync content changes
        textEditor.editor.document.addDocumentListener(object : DocumentListener {
            override fun documentChanged(e: DocumentEvent) {
                preview.updateYaml(e.document.text)
            }
        })

        // Sync preview → editor
        preview.onStepClick = { line ->
            val offset = textEditor.editor.document.getLineStartOffset(line)
            textEditor.editor.caretModel.moveToOffset(offset)
            textEditor.editor.scrollingModel.scrollToCaret(ScrollType.CENTER)
        }

        // Initial content
        preview.updateYaml(file.contentsToByteArray().toString(Charsets.UTF_8))

        return TextEditorWithPreview(textEditor, preview.browser.component, "Playbook Preview")
    }
}
```

- [ ] **Step 4: Register in plugin.xml**

```xml
<extensions defaultExtensionNs="com.intellij">
    <fileEditorProvider implementation="io.casehub.intellij.PlaybookEditorProvider"/>
</extensions>
```

- [ ] **Step 5: Create step-viewer bundle entry point**

Add a webpack/vite entry that bundles: `pages-step-viewer` components, `StepPositionScanner`, `Walker.resolve`, and the `resolveYaml` helper. Output as a single JS file for JCEF consumption.

- [ ] **Step 6: Build and verify**

Run: `JAVA_HOME=/path/to/jdk21 plugins/intellij/gradlew -p plugins/intellij buildPlugin`
Expected: BUILD SUCCESSFUL

- [ ] **Step 7: Commit**

```bash
git add plugins/intellij/
git commit -m "feat(#720): IntelliJ playbook preview with JCEF step viewer"
```

---

## References

- `specs/wip-helpdesk-form-pages/2026-11-19-playbook-split-view-design.md` — design spec
- `packages/yaml-core/src/annotation/step-position-scanner.ts` — current flat scanner
- `packages/pages-editor-core/src/types.ts` — EditableText interface
- `packages/pages-editor-core/src/editable-text-bridge.ts` — bridge base class
- `packages/pages-step-viewer/src/step-dispatch.ts` — step rendering dispatch
- `packages/pages-step-viewer/src/pages-step-viewer.ts` — viewer component
- `packages/pages-step-viewer/gallery.html` — gallery with resolveYaml pattern
- `packages/pages-markdown-editor/src/pages-markdown-editor.ts` — split-mode reference
- `plugins/intellij/` — current LSP plugin
- GitHub #720 — focal issue
