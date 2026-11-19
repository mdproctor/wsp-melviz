# Split-View Playbook Editor — Design Spec

**Issue:** #720
**Date:** 2026-11-19
**Branch:** wip/helpdesk-form-pages

## Overview

Add a split-view playbook editing experience: YAML code editor on the left, step viewer visualization on the right (read-only), with bidirectional click-sync. The same step viewer component is consumed by the IntelliJ plugin via JCEF for IDE-native preview.

## Architecture

Five layers, each independently testable:

```
                    ┌─────────────────────────────────────────┐
                    │  <playbook-split-view>  (step-viewer)   │
                    │  ┌──────────────┐  ┌──────────────────┐ │
                    │  │ code-editor  │  │ step-viewer      │ │
                    │  │ (editable)   │  │ (read-only)      │ │
                    │  └──────┬───────┘  └────────┬─────────┘ │
                    │         │  StepPositionScanner  │        │
                    │         └──────────┬───────────┘        │
                    └────────────────────┼────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │  IntelliJ Plugin   │                    │
                    │  FileEditorProvider │                    │
                    │  ┌──────────────┐  ┌──────────────────┐ │
                    │  │ IntelliJ     │  │ JCEF panel       │ │
                    │  │ text editor  │  │ <step-viewer>    │ │
                    │  └──────────────┘  └──────────────────┘ │
                    └─────────────────────────────────────────┘
```

**Web split-view:** Both panels are web components — `pages-code-editor` (left) and `pages-step-viewer` (right) — wired together by the `<playbook-split-view>` container.

**IntelliJ split-view:** IntelliJ's native text editor (left) + JCEF panel rendering `<pages-step-viewer>` only (right). IntelliJ's `CaretListener` drives sync via `executeJavaScript`; click events return via `JBCefJSQuery`.

## Layer 1: StepPositionScanner Enhancement

**Package:** `yaml-core`
**File:** `src/annotation/step-position-scanner.ts`

### Current state

`StepPositionScanner.scan(rawYaml)` returns flat `{line, address}[]` where address is a sequential index ("0", "1", "2"). No nesting awareness.

### Enhancement

Produce hierarchical addresses that reflect the step structure:

| YAML structure | Address |
|---|---|
| Top-level step 0 | `"0"` |
| Top-level step 1 (if-else) | `"1"` |
| If-else → then branch → step 0 | `"1.then.0"` |
| If-else → else branch → step 1 | `"1.else.1"` |
| Try-catch → body → step 0 | `"2.body.0"` |
| Try-catch → catch → step 0 | `"2.catch.0"` |
| Match → arm "foo" → step 0 | `"3.match.foo.0"` |
| Swim lane → lane "agent" → step 0 | `"4.lane.agent.0"` |
| Race → branch 0 → step 0 | `"5.race.0.0"` |

### Return type

```typescript
interface StepLineInfo {
  line: number;      // 0-indexed start line in YAML
  endLine: number;   // 0-indexed end line (exclusive)
  address: string;   // hierarchical address
}
```

The `endLine` enables range-based matching: given a cursor at any line within a step's YAML block, find the step whose `[line, endLine)` range contains it.

### Implementation approach

Walk the YAML line by line tracking indentation depth. When a `- ` list item is found, determine whether it's a control-flow step (if, try, match, race, select, parallel) by checking the first key. For control-flow steps, recursively scan their sub-blocks using indentation to determine boundaries.

### Tests

- Flat steps (sequential)
- Nested if-else with then/else branches
- Try-catch with body and catch blocks
- Match with named arms
- Swim lanes with named lanes
- Deeply nested (if inside try inside lane)
- Empty blocks
- Address → line and line → address lookup helpers

## Layer 2: EditableTextBridge Cursor Callback

**Package:** `pages-editor-core`
**File:** `src/editable-text-bridge.ts`

### Addition

```typescript
interface EditableText {
  // ... existing methods ...
  onCursorChange(callback: (position: Position) => void): Disposable;
}
```

Returns a `Disposable` for cleanup. The `pages-code-editor` implementation wires this to Monaco's `onDidChangeCursorPosition` event.

### Tests

- Callback fires when cursor moves
- Disposable unsubscribes
- Multiple listeners
- No callback during programmatic `setCursor()` (prevent loops)

## Layer 3: Step Viewer Click Events

**Package:** `pages-step-viewer`

### DOM changes

Every step card rendered by `renderStepContent()` gets:
- `data-step-address="1.then.0"` attribute
- `@click` handler that dispatches `step-click` CustomEvent

### Event

```typescript
interface StepClickDetail {
  address: string;    // hierarchical step address
  originalEvent: MouseEvent;
}
```

The event bubbles so the split-view container can listen on the viewer root.

### Visual feedback

Clicked/active step card gets a `step-active` CSS class (highlight border). Only one step is active at a time.

### Tests

- Click fires event with correct address
- Nested step click has nested address
- Only one step highlighted at a time
- Event bubbles to parent

## Layer 4: Split-View Container

**Package:** `pages-step-viewer`
**File:** `src/playbook-split-view.ts`

### Component

```typescript
@customElement('playbook-split-view')
export class PlaybookSplitView extends LitElement {
  @property() yaml: string = '';
  @property() splitRatio: number = 0.5;
}
```

### Behavior

1. **YAML change → viewer update:** Debounce 300ms, re-parse and re-resolve steps, update viewer
2. **Click step → editor sync:** Listen for `step-click` event → look up `StepLineInfo` by address → `setCursor(line, 0)` + `highlightRange(line, endLine)`
3. **Editor cursor → viewer sync:** `onCursorChange` → find step containing cursor line (binary search on `StepLineInfo[]`) → scroll step card into view + set `step-active`
4. **Resizable splitter:** CSS `resize` or drag handle between panels

### Error handling

If YAML is invalid (parse error), the viewer panel shows the error message inline. The editor continues working — no blocking.

## Layer 5: IntelliJ Plugin

**Package:** `plugins/intellij`

### FileEditorProvider

Register for `*.playbook.yaml` file pattern. Create a `TextEditorWithPreview` (IntelliJ's built-in split editor API, same as Markdown plugin):
- Left: standard `TextEditor`
- Right: `JCEFHtmlPanel` loading a minimal HTML page that imports the step-viewer bundle

### Sync protocol

**Editor → preview:** `CaretListener` on the IntelliJ editor fires on cursor move. Send cursor line to JCEF:
```kotlin
browser.executeJavaScript("window.syncCursorLine($line)")
```

The page's JS receives this and uses `StepPositionScanner` (bundled in the step-viewer JS) to find and highlight the matching step.

**Preview → editor:** Click on a step card triggers a `JBCefJSQuery` callback:
```kotlin
val query = JBCefJSQuery.create(browser)
query.addHandler { address ->
    // Look up line from StepPositionScanner, move IntelliJ caret
}
```

### Bundle

The step-viewer package needs a standalone bundle (no code editor — IntelliJ provides its own editor). Add a webpack/vite entry point that exports `pages-step-viewer` + `StepPositionScanner` + a thin sync adapter for the `syncCursorLine` / click-callback protocol.

### File pattern

Register for `*.playbook.yaml` — the LSP server already handles `*.page.yaml`. Playbook files are a distinct pattern.

## Implementation Order

1. **StepPositionScanner enhancement** — foundation, no UI dependencies
2. **EditableTextBridge cursor callback** — small interface addition
3. **Step viewer click events** — DOM attributes + event handlers
4. **Split-view container** — wires 1-3 together
5. **IntelliJ FileEditorProvider** — consumes the step-viewer bundle via JCEF

Each layer is independently testable. Layers 1-3 can be built in parallel.

## Testing Strategy

- **Unit tests:** Scanner address generation, cursor callback lifecycle, event dispatch
- **Integration tests:** Split-view sync (programmatic — set cursor, verify active step; click step, verify cursor position)
- **Visual verification:** Gallery page for the split-view component with sample playbook YAML
- **IntelliJ:** Manual verification in the IDE (JCEF testing is complex to automate)

## References

- `yaml-core/src/annotation/step-position-scanner.ts` — current flat scanner
- `pages-editor-core/src/editable-text-bridge.ts` — editor bridge interface
- `pages-step-viewer/src/pages-step-viewer.ts` — viewer component
- `pages-step-viewer/src/step-dispatch.ts` — 16 component type dispatch
- `pages-markdown-editor/src/pages-markdown-editor.ts` — split-mode reference pattern
- `pages-markdown-editor/src/scroll-sync.ts` — anchor-based scroll synchronization
- `plugins/intellij/` — current LSP-only plugin
- IntelliJ `TextEditorWithPreview` API — standard split editor pattern
