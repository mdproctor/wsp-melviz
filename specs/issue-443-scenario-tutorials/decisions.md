# Decisions — issue-443-scenario-tutorials

## D1: Editor SPI for scenario automation

**Choice:** Define a `ScenarioEditableText` interface that any text editing surface can implement. The scenario executor targets elements by ARIA role/name, then calls through the SPI. `pages-code-editor` is the first implementation, backed by CodeMirror 6. The SPI is registered on the component that owns the editor instance — `pages-code-editor` owns the `EditorView`, so it owns the SPI. Discovery uses an ancestor walk (`findEditableText`) that traverses shadow DOM host boundaries from the resolved ARIA element upward.
**Alternatives:**
- ARIA-only keyboard simulation — pure but fragile; CM6 uses its own input pipeline and may not respond to synthetic `KeyboardEvent`s reliably
- Direct CodeMirror API from executor — works but couples the executor to one editor implementation
**Rationale:** The SPI keeps the executor editor-agnostic. Any future editor (Monaco, plain textarea wrapper) implements the same interface. The executor's job is to find the element and call methods — it never imports CodeMirror.
**Trade-offs:** One level of indirection. Implementations must be registered or discoverable via a property/symbol on the DOM element.
**Sources:** Existing `fill` command pattern (executor calls `el.value`), `pages-builder-shell` Lit component API
**Exploration:** quick
**Status:** captured

## D2: New ARIA actions for editor automation

**Choice:** Add `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion` to the existing `aria` delivery channel, plus `spotlight` to the parser's `ARIA_ACTIONS` set. Same YAML shorthand pattern as `fill`/`click`/`select`, with a catch-all body passthrough in the parser (replaces the old three-field extraction). `spotlight` gets a special-case handler like `navigate` and `show-markdown` because its YAML body uses nested `target` rather than flat `role`/`name`.
**Alternatives:**
- New delivery channel `editor` alongside `aria`/`graphql`/`simulated` — cleaner separation but fragments the command namespace; ARIA targeting is still the discovery mechanism
- Extend `fill` to detect editor elements — overloads one command with two very different behaviors
**Rationale:** Editor commands use the same ARIA targeting (`role` + `name` to find the element) and the same dispatch flow. They're actions within the `aria` channel, not a separate channel. The parser's `ARIA_ACTIONS` set grows naturally. A central `executeStep` dispatch function in `command-executor.ts` routes all actions — existing and new — to their handlers.
**Trade-offs:** The `aria` action set grows. If many more non-standard actions accumulate, a sub-namespace may be needed later.
**Sources:** `packages/pages-aria/src/scenario/parser.ts` (ARIA_ACTIONS set), `packages/pages-aria/src/executor/command-executor.ts`
**Exploration:** quick
**Status:** captured

## D3: Tutorial format — standard hands-on sectioned scenarios

**Choice:** Yaml-composition tutorials use the standard `hands-on` sectioned scenario format. Slides for narrative (sections with empty `steps: []`), ARIA steps for editor automation. No custom `yaml-editor` content type for this use case.
**Alternatives:**
- Keep `yaml-editor` content type with custom rendering — already built but bypasses the entire scenario infrastructure (no narrative panel, no controller sidebar, no chapter navigation, no callouts)
- New content type `editor-driven` — unnecessary; `hands-on` already handles the mix of slides and automated steps
**Rationale:** The existing infrastructure provides everything: `pages-scenario-narrative` for slides, `pages-scenario-controller` for chapter navigation and transport, `runSectionedScenario` for step progression with pause/resume. The new editor commands plug into this without any tutorial host changes.
**Trade-offs:** The `yaml-editor` infrastructure from #435 (types, runner, host dispatch) becomes unused for tutorials. It may still have value for a future user-practice mode.
**Sources:** `packages/pages-aria/src/scenario/sectioned-runner.ts`, `packages/pages-aria/src/tutorial/tutorial-host.ts`
**Exploration:** quick
**Depends on:** D2
**Status:** captured

## D4: Typing animation via existing progressiveFill through SPI

**Choice:** Reuse the existing `progressiveFill` typing animation. The executor calls `ScenarioEditableText.insertText()` character-by-character (or word-by-word in later phases), with the same accelerating speed phases as the current `progressiveFill`.
**Alternatives:**
- New typing animation specific to editors — unnecessary duplication; the character-at-a-time pattern is the same regardless of target element type
- No animation, instant insert — loses the "watching someone type" experience that makes tutorials engaging
**Rationale:** `progressiveFill` already has the right behavior: 5 phases from char-by-char to 5-words-at-a-time, speed-controllable, skip-forward support. The only change is routing through the SPI instead of `el.value`.
**Trade-offs:** None significant. The SPI's `insertText` may need to accept partial inserts (character-by-character) efficiently.
**Sources:** `packages/pages-aria/src/executor/visual-feedback.ts` (typeText), `packages/pages-aria/src/server/scenario-handler.ts` (progressiveFill)
**Exploration:** quick
**Status:** captured

## D5: Editor region callouts via SPI highlight method

**Choice:** Add `highlight(from, to, style)` and `clearHighlights()` to the `ScenarioEditableText` SPI. The CodeMirror implementation uses CM6's `Decoration` API. This creates three callout layers: element spotlight (existing), editor region highlight (new), tree node spotlight (existing).
**Alternatives:**
- Use spotlight overlay on computed pixel positions — fragile; text reflows, editor scrolls
- CSS overlay on line number gutter only — can't highlight specific columns or ranges within a line
**Rationale:** CodeMirror's `Decoration` API is designed for exactly this — highlighting ranges with custom CSS classes. The SPI abstracts it so other editors can implement with their own decoration systems.
**Trade-offs:** Highlight coordinates are line/column based, not character-offset based. The tutorial author needs to know the line numbers, which depend on the editor's current content state.
**Sources:** CodeMirror 6 `Decoration` API, existing `highlightElement` in visual-feedback.ts
**Exploration:** quick
**Status:** captured

## D6: Design for real LSP autocomplete, gate on availability

**Choice:** `editor-completion` triggers autocomplete via the SPI's `triggerCompletion()`. Designed for real LSP completions. Steps using autocomplete are skipped gracefully when the LSP isn't connected to the workbench editor.
**Alternatives:**
- Simulated/mocked completions — deterministic but fake; doesn't demonstrate the real product
- Block on LSP availability — would prevent tutorials from working until LSP integration lands
**Rationale:** The tutorials should demonstrate the real product experience. The LSP is coming to the workbench editor. Designing for it now means the tutorial content works as soon as the LSP is wired in. Graceful skip means the tutorials still work without it — the editor-insert steps still type the YAML, you just don't see the autocomplete popup.
**Trade-offs:** Tutorial steps that depend on autocomplete won't demonstrate that feature until the LSP is integrated. The content works either way — autocomplete is enhancement, not gate.
**Sources:** `packages/pages-lsp/` (LSP server), `packages/pages-code-editor/` (CodeMirror component)
**Exploration:** quick
**Status:** captured

## D7: User practice mode deferred

**Choice:** User-edits-themselves with validation is out of scope for #443. The existing `yaml-editor` validation infrastructure (validateYamlStep, expectedKeys, expectedStructure) is preserved for future use.
**Alternatives:**
- Include practice mode now — increases scope significantly; the scenario executor needs a "wait for user" command type
**Rationale:** The primary goal is demonstrating yaml-core through live automation. User practice is a separate interaction model that can layer on top later.
**Trade-offs:** Tutorials are watch-only. Users learn by observation, not by doing. Practice can be added as a follow-up.
**Sources:** `packages/pages-aria/src/tutorial/yaml-editor-runner.ts` (existing validation)
**Exploration:** quick
**Status:** captured
