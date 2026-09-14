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
interface Position { line: number; col: number; }

interface ScenarioEditableText {
  insertText(text: string): void;
  replaceRange(from: Position, to: Position, text: string): void;
  deleteRange(from: Position, to: Position): void;
  setCursor(line: number, col: number): void;
  getCursor(): Position;
  getText(): string;
  getLineCount(): number;
  setContent(text: string): void;
  highlight(from: Position, to: Position, style?: 'pulse' | 'underline' | 'glow'): void;
  clearHighlights(): void;
  triggerCompletion?(): void;
  selectCompletion?(label: string): boolean;
}
```

All range-based methods use `Position` (`{line, col}`) consistently — matching the YAML authoring format, `setCursor`/`getCursor`, and `highlight`. The CodeMirror implementation converts internally via `view.state.doc.line(pos.line).from + pos.col`.

**Discovery:** The executor checks for a well-known Symbol property on the resolved DOM element. Because ARIA tree walker resolution finds the deepest matching element (e.g. CodeMirror's `div.cm-content[role="textbox"]` inside a code editor's shadow DOM), the executor walks **up** through shadow DOM host boundaries to find the nearest ancestor bearing the SPI symbol:

```typescript
const EDITABLE_TEXT = Symbol.for('scenario-editable-text');

function isEditableText(el: Element): el is Element & { [EDITABLE_TEXT]: ScenarioEditableText } {
  return EDITABLE_TEXT in el;
}

function findEditableText(el: Element): ScenarioEditableText | null {
  let current: Element | null = el;
  while (current) {
    if (isEditableText(current)) return current[EDITABLE_TEXT];
    // Walk up: parent element in light DOM, then shadow host
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

**Registration pattern:** Each implementation sets the symbol property on its host element. The SPI is owned by the component that owns the editor instance — `pages-code-editor` owns the `EditorView`, so it owns the SPI:

```typescript
// In pages-code-editor.ts
connectedCallback() {
  super.connectedCallback();
  (this as any)[Symbol.for('scenario-editable-text')] = new CodeEditorBridge(this);
}
```

### 2.2 CodeMirror implementation

`pages-code-editor` implements `ScenarioEditableText` by bridging to its internal `EditorView`. The bridge is a private class inside `pages-code-editor` — it accesses the `_editorView` directly (no `as any` casts, no cross-component private field access). No CodeMirror types are exported.

| SPI method | CodeMirror implementation |
|---|---|
| `insertText(text)` | `view.dispatch({changes: {from: cursor, insert: text}})` |
| `replaceRange(from, to, text)` | Convert positions via `toOffset(pos)` → `view.dispatch({changes: {from: offset1, to: offset2, insert: text}})` |
| `deleteRange(from, to)` | Convert positions via `toOffset(pos)` → `view.dispatch({changes: {from: offset1, to: offset2}})` |
| `setCursor(line, col)` | `view.dispatch({selection: {anchor: toOffset({line, col})}})` |
| `getCursor()` | Convert `view.state.selection.main.head` back to `{line, col}` via `doc.lineAt(offset)` |
| `getText()` | `view.state.doc.toString()` |
| `getLineCount()` | `view.state.doc.lines` |
| `setContent(text)` | `view.dispatch({changes: {from: 0, to: doc.length, insert: text}})` |
| `highlight(from, to, style)` | Convert positions → offsets → `Decoration.mark({class})` via `StateEffect` |
| `clearHighlights()` | Clear the decoration set via `StateEffect` |
| `triggerCompletion()` | `startCompletion(view)` from `@codemirror/autocomplete` |
| `selectCompletion(label)` | Find completion in active tooltip, apply it |

`toOffset(pos: Position)` is a private helper: `view.state.doc.line(pos.line).from + pos.col`. The offset-based CodeMirror API is an implementation detail — the SPI's public surface is consistently `{line, col}`.

This resolves the existing fragile access pattern where `builder-shell.ts` reaches into `pages-code-editor`'s private `_editorView` via `(editorEl as any)._editorView`. The SPI becomes the public API for editor manipulation — `builder-shell` can adopt it for its own needs over time.

### 2.3 New ARIA actions

Seven new actions added to the `aria` delivery channel. They follow the same YAML shorthand pattern as existing commands.

| Action | Purpose | YAML shape |
|---|---|---|
| `editor-insert` | Insert text at cursor | `{role, name, value, typing?, line?, col?}` |
| `editor-replace` | Replace a text range | `{role, name, from: {line, col}, to: {line, col}, value, typing?}` |
| `editor-delete` | Delete a text range | `{role, name, from: {line, col}, to: {line, col}}` |
| `editor-set-content` | Atomically replace entire editor content | `{role, name, value, typing?}` |
| `editor-cursor` | Move cursor to position | `{role, name, line, col}` |
| `editor-highlight` | Highlight a code region | `{role, name, from: {line, col}, to: {line, col}, style?}` |
| `editor-completion` | Trigger and select autocomplete | `{role, name, label?}` |

`editor-set-content` replaces the full editor buffer atomically via `setContent()` on the SPI. This is essential for non-incremental section transitions where the next section's YAML is unrelated to the previous section's content. With `typing: instant` (recommended for transitions), it acts as a clean reset. With `typing: progressive`, it types the new content from scratch after clearing.

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

**Parser integration:** Add all seven editor actions plus `spotlight` to the `ARIA_ACTIONS` set in `parser.ts`. The current set (`navigate`, `click`, `fill`, `select`, `expand`, `collapse`, `assert`, `wait`, `show-markdown`) is extended to include: `spotlight`, `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`.

**Shorthand expander changes:** The existing `expandAriaShorthand` only passes through `value`, `state`, and `timeout` from the YAML body — all other fields (`typing`, `from`, `to`, `line`, `col`, `style`, `label`) are silently dropped. Two changes fix this:

1. **Catch-all passthrough for generic actions.** After extracting `role`/`name`/`index`/`within` for the target, ALL remaining body properties are spread onto the step:

```typescript
const targetKeys = new Set(['role', 'name', 'index', 'within']);
const step: ScenarioStep = { delivery: 'aria', name: autoName, action, target };
for (const [key, val] of Object.entries(body)) {
  if (!targetKeys.has(key) && val != null) {
    (step as Record<string, unknown>)[key] = val;
  }
}
```

This replaces the three explicit `value`/`state`/`timeout` lines, naturally passing through `typing`, `from`, `to`, `line`, `col`, `style`, `label`, and any future fields without parser changes.

2. **Special-case handler for `spotlight`.** Like `navigate` and `show-markdown`, `spotlight` has a non-standard YAML shape — its target is nested as `body.target` (not flat `body.role`/`body.name`), and it carries unique fields (`content`, `position`, `duration`, `also`). The generic path would produce `{role: 'unknown', name: 'unknown'}`. A dedicated handler extracts the nested target and passes through all properties:

```typescript
if (action === 'spotlight') {
  const body = raw[action] as Record<string, unknown>;
  const tgt = body.target as Record<string, unknown> | undefined;
  const target: AriaTarget | undefined = tgt
    ? { role: tgt.role as string, name: tgt.name as string,
        ...(tgt.index != null ? { index: tgt.index as string } : {}),
        ...(tgt.within != null ? { within: tgt.within as AriaTarget } : {}) }
    : undefined;
  const step: ScenarioStep = {
    delivery: 'aria',
    name: `spotlight-${target?.role ?? 'unknown'}-${target?.name ?? 'unknown'}`,
    action: 'spotlight',
    ...(target ? { target } : {}),
  };
  for (const [key, val] of Object.entries(body)) {
    if (key !== 'target' && val != null) {
      (step as Record<string, unknown>)[key] = val;
    }
  }
  return step;
}
```

This preserves `content`, `position`, `duration`, and `also` on the step object for the executor to consume.

**Executor integration:** Add a handler for each new action in `command-executor.ts`, plus a central dispatch function. The executor uses `findEditableText` (§2.1) to discover the SPI from the resolved target element, walking up through shadow DOM hosts:

```typescript
async function editorInsert(target: AriaTarget, value: string, typing: string, speed: number, line?: number, col?: number): Promise<void> {
  const el = resolveTarget(target);
  const editor = findEditableText(el);
  if (!editor) throw new Error(`Target is not an editable text element`);
  if (line !== undefined && col !== undefined) {
    editor.setCursor(line, col);
  }
  if (typing === 'instant') {
    editor.insertText(value);
  } else {
    await progressiveInsert(editor, value, speed, (remaining) => editor.insertText(remaining));
  }
}
```

**`progressiveInsert` adaptation:** The existing `progressiveFill` in `scenario-handler.ts` takes `speed: number` and computes delays from it (`charDelay = Math.max(10, 40 / speed)`, `wordDelay = Math.max(20, 60 / speed)`). `progressiveInsert` preserves this: it takes `speed` and uses the same delay formulas. Instead of `el.value = revealed`, it calls `editor.insertText(char)` for each character (or chunk) in the progressive phases. The phased acceleration (character → word chunks → larger chunks) is preserved. The adapted function lives in `command-executor.ts` alongside the other editor action handlers.

**`editor-replace` strategy:** `editor-replace` uses a delete-then-insert approach: first `deleteRange(from, to)` removes the original content (leaving the cursor at `from`), then progressive insert types the replacement text. This means all three progressive-typing actions share the same insertion codepath — they differ only in their pre-operation:

| Action | Pre-operation | Progressive phase |
|---|---|---|
| `editor-insert` | Optional `setCursor(line, col)` | `progressiveInsert` at cursor |
| `editor-replace` | `deleteRange(from, to)` | `progressiveInsert` at cursor (now at `from`) |
| `editor-set-content` | `setContent('')` (clear buffer) | `progressiveInsert` at cursor (now at 0,0) |

**Action-aware skip mechanism:** The skip shortcut must use the correct SPI method for each action, since `setContent(fullValue)` would destroy existing content during an `editor-insert`. `progressiveInsert` takes a `finishFn` callback that completes the operation when typing is skipped:

| Action | `finishFn` (called on skip) | Rationale |
|---|---|---|
| `editor-insert` | `editor.insertText(remainingText)` | Inserts only what hasn't been typed yet — preserves existing content |
| `editor-set-content` | `editor.setContent(fullValue)` | Replaces entire buffer — matches the action's intent |
| `editor-replace` | `editor.insertText(remainingText)` | Range already deleted in pre-operation; insert the remainder at cursor |

**Dispatch function:** `command-executor.ts` gains an `executeStep` function that dispatches by action name. This is the single entry point used by `sectioned-runner.ts`. It takes `speed` so that intra-step timing (progressive typing delays, spotlight duration) respects the transport control:

```typescript
export async function executeStep(step: ScenarioStep, eventTarget?: EventTarget, speed = 1.0): Promise<void> {
  const s = step as Record<string, unknown>;
  switch (step.action) {
    case 'click': return click(step.target!);
    case 'fill': return fill(step.target!, s.value as string);
    case 'select': return select(step.target!, s.value as string);
    case 'expand': return expand(step.target!);
    case 'collapse': return collapse(step.target!);
    case 'assert': return assertState(step.target!, s.state as Partial<AriaState>);
    case 'wait': return waitFor(step.target!, s.state as Partial<AriaState>, s.timeout as number ?? 5000);
    case 'navigate': window.location.href = s.value as string; return;
    case 'show-markdown': return showMarkdownStep(s, eventTarget);
    case 'spotlight': return spotlightStep(s, speed);
    case 'editor-insert': return editorInsert(step.target!, s.value as string, s.typing as string ?? 'progressive', speed, s.line as number, s.col as number);
    case 'editor-set-content': return editorSetContent(step.target!, s.value as string, s.typing as string ?? 'progressive', speed);
    case 'editor-replace': return editorReplace(step.target!, s.from as Position, s.to as Position, s.value as string, s.typing as string ?? 'progressive', speed);
    case 'editor-delete': return editorDelete(step.target!, s.from as Position, s.to as Position);
    case 'editor-cursor': return editorCursor(step.target!, s.line as number, s.col as number);
    case 'editor-highlight': return editorHighlight(step.target!, s.from as Position, s.to as Position, s.style as string);
    case 'editor-completion': return editorCompletion(step.target!, s.label as string);
    default: throw new Error(`Unknown action: ${step.action}`);
  }
}
```

The sectioned runner passes `rs.speed` to `executeStep(step, eventTarget, rs.speed)`.

**Speed propagation:** `speed` flows from the transport control through to intra-step timing:
- `progressiveInsert(editor, value, speed, finishFn)` — computes `charDelay = Math.max(10, 40 / speed)` and `wordDelay = Math.max(20, 60 / speed)`, matching `progressiveFill`'s formula
- `spotlightStep(step, speed)` — computes `duration = Math.max(2000, wordCount * 250 / speed)`, matching `scenario-handler.ts` line 690

At speed 0.5×, typing is slow and spotlights linger. At speed 10×, typing is near-instant and spotlights flash. Both intra-step and inter-step timing scale together.

**`show-markdown` in the sectioned runner context:** The push-wire path (`scenario-handler.ts`) dispatches a `scenario-narrative` event to a narrative target. In the sectioned runner, narrative content is managed via `fireState` events. `showMarkdownStep` dispatches the same `scenario-narrative` custom event on the provided `eventTarget`, which `pages-scenario-narrative` already listens for. The sectioned runner passes its `eventTarget` through to `executeStep`.

**`spotlight` in the executor:** `spotlightStep` delegates to the existing `showSpotlight()` from `executor/spotlight.ts`, constructing a `SpotlightConfig` from the step's `target`, `content`, `position`, `duration`, and `also` fields. Duration is computed from `speed` when not explicitly specified. The resolved ARIA target is converted to a DOM element via `resolveTarget`.

**Execution path unification:** The existing `sectioned-runner.ts` has an inline `executeAriaStep` function that only handles `click`, `fill`, and `select` via `document.querySelector` (which cannot cross shadow DOM boundaries). This function is replaced with a call to `executeStep(step, eventTarget, rs.speed)`, which uses the full ARIA tree walker (`findAllByRole`) that correctly traverses shadow DOM. This unification ensures all ARIA actions — existing and new — are available to tutorials via `runSectionedScenario`.

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
          name: "Page YAML source"
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

The `contentType` field is NOT set in the YAML meta — it is a registration-time concern on `TutorialDescriptor` (in `tutorial/types.ts`), set when tutorials are registered in the catalog. `TutorialMeta` (the interface backing the YAML `meta` block) does not include `contentType`. The catalog registration sets `contentType: 'hands-on'` for the reworked tutorial.

**Content data transformation:** The existing `yaml-editor` format sections carry `initialYaml`, `expectedKeys`, `expectedStructure`, `hint`, `solutionYaml`, and `buildOnPrevious` fields. In the `hands-on` rework, these are intentionally dropped — the YAML examples become `editor-insert`/`editor-set-content` step values, the narrative markdown is reused as section content, and the validation metadata (`expectedKeys`, `expectedStructure`) has no equivalent in the demonstration-mode tutorial. The validation infrastructure (`validateYamlStep`, `deepSubsetMatch` in `yaml-editor-runner.ts`) is preserved for the deferred user-practice mode.

The existing tutorial content (15 narrative markdown files, YAML examples) is reused — the markdown files become slide section content, the YAML examples become `editor-insert` step values. Non-incremental section transitions use `editor-set-content` with `typing: instant` to atomically replace the editor buffer.

### 2.6 Tutorial host changes

The tutorial host requires two changes:

**1. Execution path unification.** The existing `_renderTutorial()` method starts `runSectionedScenario`, which uses an inline `executeAriaStep` function limited to `click`/`fill`/`select` via `document.querySelector`. This is replaced with delegation to `command-executor.ts` (see §2.3), making all ARIA actions available to tutorials through the standard tree walker.

**2. Builder-shell rendering.** The existing `_renderTutorial()` renders `pages-scenario-narrative` and `pages-scenario-controller` but does NOT render `<pages-builder-shell>` — only `_renderYamlEditorTutorial()` does. A dedicated tutorial app component (§3 task 6) renders builder-shell as the tutorial's target application alongside the scenario narrative and controller. This is structurally similar to how `_renderYamlEditorTutorial` renders builder-shell today, but integrated into the `hands-on` path rather than the `yaml-editor` path.

The `yaml-editor` code path in the tutorial host remains for the deferred user-practice mode but is not used by the reworked tutorials.

---

## 3. Issue decomposition

| Order | Issue | Depends on | Scale |
|---|---|---|---|
| 1 | ScenarioEditableText SPI — interface definition, Symbol-based discovery, `findEditableText` ancestor walk, type exports | — | S |
| 2 | CodeMirror SPI implementation in pages-code-editor — bridge class, highlight decorations, SPI registration | 1 | M |
| 3 | Execution path unification — replace `executeAriaStep` in `sectioned-runner.ts` with delegation to `command-executor.ts` | — | S |
| 4 | New ARIA actions — parser expansion (`spotlight` + 7 editor actions), executor handlers, `progressiveInsert` adapter | 1, 3 | M |
| 5 | Tutorial content rework — convert 15-step content from yaml-editor sections to hands-on scenarios with editor commands | 2, 4 | L |
| 6 | Tutorial host app — dedicated page/component that renders builder-shell as the tutorial target application with scenario narrative and controller | 2 | S |
| 7 | Follow-up (TBD-1): LSP wiring for workbench editor — connect pages-lsp to the CodeMirror editor in builder-shell | — | M |
| 8 | Follow-up (TBD-2): User practice mode — validation-gated editing using existing yaml-editor infrastructure | 5 | M |
| 9 | Follow-up (TBD-3): New tutorial content beyond yaml-composition | 5 | M |
| 10 | Follow-up (TBD-4): Scenario server-side changes — add editor actions to `scenario-handler.ts` if needed | 4 | S |

`TBD-N` placeholders are replaced with GitHub issue numbers during implementation (filed against `casehubio/casehub-pages`).

---

## 4. Out of scope

Each deferred item is tracked as a GitHub issue (filed during implementation):

- **LSP integration in workbench editor** (TBD-1) — designed for but not implemented; `editor-completion` skips gracefully when LSP is absent
- **User practice mode** (TBD-2) — deferred per D7; existing yaml-editor validation infrastructure (`validateYamlStep`, `deepSubsetMatch`) is preserved
- **New tutorial content beyond yaml-composition** (TBD-3) — this issue reworks the existing 15 steps; additional tutorials are separate issues
- **Scenario server-side changes** (TBD-4) — the new commands work through the in-process `runSectionedScenario` path; server-side `scenario-handler.ts` updates are a follow-up if needed

---

## References

- `packages/pages-aria/src/scenario/parser.ts` — ARIA_ACTIONS set, shorthand expansion
- `packages/pages-aria/src/executor/command-executor.ts` — ARIA command execution
- `packages/pages-aria/src/executor/visual-feedback.ts` — typeText, highlightElement, isTypingSkipped, resetTypingSkip
- `packages/pages-aria/src/server/scenario-handler.ts` — progressiveFill (source for progressiveInsert adaptation)
- `packages/pages-aria/src/executor/spotlight.ts` — showSpotlight, SpotlightConfig
- `packages/pages-aria/src/scenario/sectioned-runner.ts` — TutorialRunner, step progression
- `packages/pages-aria/src/tutorial/tutorial-host.ts` — tutorial catalog and rendering
- `packages/pages-aria/src/controller/scenario-controller.ts` — outline, transport controls
- `packages/pages-aria/src/controller/scenario-narrative.ts` — markdown rendering
- `packages/pages-builder/src/shell/builder-shell.ts` — builder-change event, tutorial target host
- `packages/pages-code-editor/src/pages-code-editor.ts` — CodeMirror 6 component, SPI host (owns EditorView)
- GE-20260905-3e4256 — drawSelection() required for cursor in shadow DOM
- GE-20260907-6fdc04 — tooltip override cascade in shadow DOM
- GE-20260905-5986c1 — Compartment pattern for dynamic CM6 properties in Lit
- casehubio/casehub-pages#435 — yaml-core engine + tutorial content (landed)
