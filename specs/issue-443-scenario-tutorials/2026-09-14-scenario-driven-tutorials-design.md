# Scenario-Driven Interactive Tutorials — Design Spec

**Issue:** casehubio/casehub-pages#443
**Branch:** issue-443-scenario-tutorials
**Date:** 2026-09-14
**Scope:** Editor automation SPI, new scenario commands, tutorial content rework

---

## 1. Vision

CaseHub tutorials should teach by demonstration — the scenario executor types YAML into the editor character by character, triggers autocomplete, selects completions, and highlights code regions, while the user watches with full transport controls (play/pause/step/speed). The existing tutorial infrastructure (narrative panel, controller sidebar, chapter navigation, spotlight callouts) provides the UX.

Issue #435 landed the yaml-core engine (variables, forEach, modules, conditionals), schema composition, builder integration, and tutorial content. This issue reworks the tutorial delivery from a custom `yaml-editor` content type to standard `hands-on` scenarios that drive the editor through ARIA commands.

---

## 2. Architecture

### 2.1 ScenarioEditableText SPI

A DOM-level interface that any text editing surface can implement. The scenario executor discovers elements by ARIA role/name, then calls through this interface for all text manipulation.

```typescript
interface ScenarioEditableText {
  insertText(text: string): void;
  replaceRange(from: number, to: number, text: string): void;
  deleteRange(from: number, to: number): void;
  setCursor(line: number, col: number): void;
  getCursor(): { line: number; col: number };
  getText(): string;
  getLineCount(): number;
  highlight(from: {line: number; col: number},
            to: {line: number; col: number},
            style?: 'pulse' | 'underline' | 'glow'): void;
  clearHighlights(): void;
  triggerCompletion?(): void;
  selectCompletion?(label: string): boolean;
}
```

**Discovery:** The executor checks for a well-known property on the resolved DOM element:

```typescript
const EDITABLE_TEXT = Symbol.for('scenario-editable-text');

function isEditableText(el: Element): el is Element & { [EDITABLE_TEXT]: ScenarioEditableText } {
  return EDITABLE_TEXT in el;
}
```

**Registration pattern:** Each implementation sets the symbol property on its host element:

```typescript
// In pages-builder-shell.ts
connectedCallback() {
  super.connectedCallback();
  (this as any)[Symbol.for('scenario-editable-text')] = this._editorBridge;
}
```

### 2.2 CodeMirror implementation

`pages-builder-shell` implements `ScenarioEditableText` by bridging to its internal `EditorView`:

| SPI method | CodeMirror implementation |
|---|---|
| `insertText(text)` | `view.dispatch({changes: {from: cursor, insert: text}})` |
| `replaceRange(from, to, text)` | `view.dispatch({changes: {from, to, insert: text}})` |
| `deleteRange(from, to)` | `view.dispatch({changes: {from, to}})` |
| `setCursor(line, col)` | `view.dispatch({selection: {anchor: lineOffset + col}})` |
| `getCursor()` | Read from `view.state.selection.main.head` |
| `getText()` | `view.state.doc.toString()` |
| `getLineCount()` | `view.state.doc.lines` |
| `highlight(from, to, style)` | Create `Decoration.mark({class})` via `StateEffect` |
| `clearHighlights()` | Clear the decoration set via `StateEffect` |
| `triggerCompletion()` | `startCompletion(view)` from `@codemirror/autocomplete` |
| `selectCompletion(label)` | Find completion in active tooltip, apply it |

The bridge is a private class inside `pages-builder-shell` — it holds a reference to the `EditorView` and implements each method. No CodeMirror types are exported.

### 2.3 New ARIA actions

Six new actions added to the `aria` delivery channel. They follow the same YAML shorthand pattern as existing commands.

| Action | Purpose | YAML shape |
|---|---|---|
| `editor-insert` | Insert text at cursor | `{role, name, value, typing?, line?, col?}` |
| `editor-replace` | Replace a text range | `{role, name, from: {line, col}, to: {line, col}, value, typing?}` |
| `editor-delete` | Delete a text range | `{role, name, from: {line, col}, to: {line, col}}` |
| `editor-cursor` | Move cursor to position | `{role, name, line, col}` |
| `editor-highlight` | Highlight a code region | `{role, name, from: {line, col}, to: {line, col}, style?}` |
| `editor-completion` | Trigger and select autocomplete | `{role, name, label?}` |

**`typing` field:** Controls text insertion animation.

| Value | Behavior |
|---|---|
| `progressive` (default for insert/replace) | Character-at-a-time with accelerating phases (reuses `progressiveFill` algorithm) |
| `instant` | Full text inserted immediately, no animation |
| `character` | Pure character-at-a-time, no acceleration |

**`editor-completion` flow:**

1. Call `triggerCompletion()` on the SPI — opens the autocomplete popup
2. If `label` is specified, wait for the completion list to appear (poll at 50ms, timeout 2s)
3. Call `selectCompletion(label)` — finds the item and applies it
4. If LSP is not connected or completion doesn't appear within timeout, skip gracefully (log a warning, continue to next step)

**Parser integration:** Add all six to the `ARIA_ACTIONS` set in `parser.ts`. The shorthand expander handles `from`/`to` nested objects and `typing`/`style`/`label` fields.

**Executor integration:** Add a handler for each action in `command-executor.ts`:

```typescript
async function editorInsert(target: AriaTarget, value: string, typing: string, line?: number, col?: number): Promise<void> {
  const el = resolveTarget(target);
  if (!isEditableText(el)) throw new Error(`Target is not an editable text element`);
  const editor = el[EDITABLE_TEXT];
  if (line !== undefined && col !== undefined) {
    editor.setCursor(line, col);
  }
  if (typing === 'instant') {
    editor.insertText(value);
  } else {
    await progressiveInsert(editor, value, typing);
  }
}
```

`progressiveInsert` adapts the existing `progressiveFill` algorithm to work through the SPI's `insertText` method instead of `el.value`.

### 2.4 Editor region callouts

Three callout layers compose for rich tutorial demonstrations:

| Layer | Mechanism | Use case |
|---|---|---|
| Element spotlight (existing) | `showSpotlight()` — dim backdrop + clip-path cutout + callout bubble | "This is the tree view" |
| Editor region highlight (new) | SPI `highlight()` → CM6 `Decoration` | "This forEach directive generates copies" |
| Tree node spotlight (existing) | `showSpotlight()` targeting a treeitem by role/name | "A new Modules section appeared" |

A tutorial step sequence might:
1. `editor-insert` — types `forEach:\n  as: field\n  in: [cpu, memory, disk]` character by character
2. `editor-highlight` — highlights the just-typed text with `pulse` style
3. `spotlight` — spotlights the tree node that appeared (`role: treeitem, name: "forEach indicator"`)
4. `show-markdown` — shows a callout explaining what happened

### 2.5 Tutorial content structure

Tutorials use the standard `hands-on` sectioned scenario format:

```yaml
scenario: yaml-composition
meta:
  title: "CaseHub YAML Composition"
  description: "Watch the composition language build pages live"
  area: yaml-composition
  contentType: hands-on

sections:
  - title: "YAML Basics"
    content:
      type: template
      path: content/01-yaml-basics.md
    steps: []  # slide only — auto-pauses

  - title: "Building a Page"
    content:
      type: template
      path: content/01b-building.md
    steps:
      - editor-insert:
          role: textbox
          name: "YAML editor"
          value: |
            pages:
              - name: My First Page
                components:
                  - type: title
                    properties:
                      text: Hello World
          typing: progressive
      - spotlight:
          target: {role: tree, name: "Document outline"}
          content: "The tree view shows your page structure"
```

The existing tutorial content (15 narrative markdown files, YAML examples) is reused — the markdown files become slide section content, the YAML examples become `editor-insert` step values.

### 2.6 Tutorial host changes

Minimal. The tutorial host already handles `hands-on` content type correctly via `runSectionedScenario`. The `yaml-editor` code path in the tutorial host remains for future user-practice mode but is not used by the reworked tutorials.

The only change: the tutorial host needs to render `<pages-builder-shell>` as the target application (instead of a form or other app). This is a tutorial-level concern — the YAML file specifies a section that loads the builder shell, and the scenario steps target it by ARIA role/name.

**Builder shell as tutorial target:** The tutorial's HTML host page (or a dedicated tutorial app component) must render `<pages-builder-shell>` somewhere in the DOM so the executor can find it. This is the same pattern as the form-automation tutorial rendering a form for the executor to target.

---

## 3. Issue decomposition

| Order | Issue | Depends on | Scale |
|---|---|---|---|
| 1 | ScenarioEditableText SPI — interface definition, Symbol-based discovery, type exports | — | S |
| 2 | CodeMirror SPI implementation in pages-builder-shell — bridge class, highlight decorations | 1 | M |
| 3 | New ARIA actions — parser expansion, executor handlers, progressive insert adapter | 1 | M |
| 4 | Tutorial content rework — convert 15-step content from yaml-editor sections to hands-on scenarios with editor commands | 2, 3 | L |
| 5 | Tutorial host app — dedicated page/component that renders builder-shell as the tutorial target application | 2 | S |
| 6 | Follow-up: LSP wiring for workbench editor — connect pages-lsp to the CodeMirror editor in builder-shell | — | M |
| 7 | Follow-up: User practice mode — validation-gated editing using existing yaml-editor infrastructure | 4 | M |

---

## 4. Out of scope

- **LSP integration in workbench editor** — designed for but not implemented; `editor-completion` skips gracefully when LSP is absent
- **User practice mode** — deferred per D7; existing yaml-editor validation infrastructure preserved
- **New tutorial content beyond yaml-composition** — this issue reworks the existing 15 steps; additional tutorials are separate issues
- **Scenario server-side changes** — the new commands work through the in-process `runSectionedScenario` path; server-side `scenario-handler.ts` updates are a follow-up if needed

---

## References

- `packages/pages-aria/src/scenario/parser.ts` — ARIA_ACTIONS set, shorthand expansion
- `packages/pages-aria/src/executor/command-executor.ts` — ARIA command execution
- `packages/pages-aria/src/executor/visual-feedback.ts` — typeText, progressiveFill, highlightElement
- `packages/pages-aria/src/executor/spotlight.ts` — showSpotlight, SpotlightConfig
- `packages/pages-aria/src/scenario/sectioned-runner.ts` — TutorialRunner, step progression
- `packages/pages-aria/src/tutorial/tutorial-host.ts` — tutorial catalog and rendering
- `packages/pages-aria/src/controller/scenario-controller.ts` — outline, transport controls
- `packages/pages-aria/src/controller/scenario-narrative.ts` — markdown rendering
- `packages/pages-builder/src/shell/builder-shell.ts` — CodeMirror wrapper, builder-change event
- `packages/pages-code-editor/src/pages-code-editor.ts` — CodeMirror 6 component
- GE-20260905-3e4256 — drawSelection() required for cursor in shadow DOM
- GE-20260907-6fdc04 — tooltip override cascade in shadow DOM
- GE-20260905-5986c1 — Compartment pattern for dynamic CM6 properties in Lit
- casehubio/casehub-pages#435 — yaml-core engine + tutorial content (landed)
