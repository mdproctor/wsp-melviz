## D1: Scope — all three deliverables

**Choice:** Implement all three deliverables in this branch: (1) pages-code-editor component, (2) document exportDiagram availability from pages-diagram-core (revised from re-export per D6), (3) standalone diagram export tool in examples/
**Alternatives:**
- Code editor only — smaller scope, but leaves the issue partially done
- Editor + standalone tool — skips export promotion, misses the graph-renderer public API improvement
**Rationale:** The standalone tool depends on the editor, and the export re-export is trivial. Completing all three closes #372 fully.
**Trade-offs:** Larger branch, but deliverables 2 and 3 are small increments once the editor exists.
**Sources:** casehubio/casehub-pages#372 issue body
**Exploration:** quick
**Status:** captured

## D2: Tokenizer stays in pages-aria (no extraction needed)

**Choice:** Leave the existing `tokenizeYamlLine` and `YamlToken` in pages-aria. The editor uses CodeMirror's Lezer grammars for syntax highlighting (per revised D3/D4), so the custom tokenizer is not needed by the editor. It continues to serve the scenario YAML viewer, which has different requirements (step-level position tracking, active-step highlighting).
**Alternatives:**
- Extract to pages-primitives — was the plan when the editor needed the custom tokenizer, but CodeMirror makes this unnecessary
- Extract to pages-ui-components — wrong dependency direction (see decision review R1-03)
**Rationale:** No code movement needed. The custom tokenizer and the Lezer grammar serve different purposes and coexist cleanly. Extraction would be premature — if a future consumer needs the custom tokenizer, it can be extracted then.
**Trade-offs:** Two highlighting implementations exist in the codebase (Lezer for editor, custom for scenario viewer). Acceptable — they target different use cases.
**Sources:** packages/pages-aria/src/controller/yaml-highlighter.ts, decision review R1-03
**Exploration:** quick
**Status:** revised
**Supersedes:** Previous D2 (extract to pages-primitives)

## D3: Edit mechanism — CodeMirror 6

**Choice:** Use CodeMirror 6 as the editing engine, wrapped in a LitElement Web Component. CodeMirror provides syntax highlighting via Lezer grammars, cursor management, selection, undo/redo, and — critically — a completion framework and LSP client extension point for future language server integration.
**Alternatives:**
- Textarea overlay — simpler but cannot support completion popups, cursor-aware language intelligence, or IDE plugin parity. The requirement for context-aware completion and future VS Code/IntelliJ plugin sharing makes this insufficient.
- Contenteditable — browser inconsistencies, worse a11y, hard to maintain caret position
- Monaco — ~2MB+, overkill; CodeMirror 6 is modular (~40-80KB for core + language)
**Rationale:** The long-term vision is IDE plugins for VS Code and IntelliJ sharing a Language Server with the web editor. CodeMirror 6 is the standard web-based editor that supports LSP integration. It has built-in YAML and JSON language modes, a completion framework, and lint/diagnostics support. The textarea overlay would require rebuilding all of this poorly.
**Trade-offs:** Adds CodeMirror 6 as a dependency (~40-80KB gzipped for core + YAML/JSON). More complex integration with shadow DOM (styles must be injected into the shadow root). But this cost is justified by the completion requirement and IDE plugin vision.
**Sources:** casehubio/casehub-pages#372 issue body, user clarification (IDE plugin target, context-aware completion requirement)
**Exploration:** quick
**Status:** revised

## D4: Language support — CodeMirror Lezer grammars + future LSP

**Choice:** Use CodeMirror's built-in language support: `@codemirror/lang-yaml` and `@codemirror/lang-json` for syntax highlighting via Lezer grammars. Expose a `language` property on the component that selects the active language mode. For context-aware completion (CaseHub YAML schema, jq expressions), design the component to accept a CodeMirror `Extension[]` property so a language server client extension can be plugged in later.
**Alternatives:**
- Custom tokenizer interface `(line, state) => { tokens, endState }` — reinvents what Lezer grammars already do, and wouldn't integrate with CodeMirror's completion/lint framework
- Custom tokenizer from pages-aria — production-quality for line-level highlighting but stateless, cannot handle multi-line constructs, and incompatible with CodeMirror's architecture
**Rationale:** CodeMirror's Lezer grammars handle multi-line constructs, bracket matching, and code folding correctly out of the box. The `Extension[]` property provides a clean seam for future LSP integration without API changes. The existing `tokenizeYamlLine` from pages-aria stays in pages-aria for the scenario viewer — it serves a different purpose (step-level highlighting with position tracking).
**Trade-offs:** The existing custom tokenizer is not reused in the editor. Two highlighting implementations coexist: Lezer for the editor, custom tokenizer for the scenario viewer. This is acceptable — they serve different use cases and the Lezer grammar is strictly more capable.
**Sources:** casehubio/casehub-pages#372, user clarification (context-aware completion, IDE plugin target)
**Exploration:** quick
**Status:** revised
**Supersedes:** Previous D4 (custom stateful tokenizer interface)

## D5: Package placement — standalone @casehubio/pages-code-editor

**Choice:** Create a standalone `@casehubio/pages-code-editor` package, following the `@casehubio/pages-table` precedent — heavyweight components with their own dependency profile live in their own packages.
**Alternatives:**
- In pages-ui-components — pages-ui-components is lightweight form primitives (only `lit` as third-party dep). Adding 6 `@codemirror/*` packages (40-80KB gzipped) fundamentally changes its character and bloats every consumer's install.
**Rationale:** pages-table already established the pattern: complex standalone components with unique dependencies get their own package. The code editor has 6 CodeMirror packages — a distinct dependency profile from the lightweight form primitives in pages-ui-components.
**Trade-offs:** Adds a new package to the monorepo (package.json, tsconfig, build script entry). Justified by dependency isolation.
**Sources:** packages/pages-table/ (precedent), spec review R1-02
**Exploration:** quick
**Status:** revised
**Supersedes:** Previous D5 (pages-ui-components)

## D6: Export promotion — move to graph-renderer

**Choice:** Move `exportDiagram()`, `computeNodeBounds()`, `computeExportViewport()`, types, and `html-to-image@1.11.11` from pages-diagram-core to graph-renderer, as issue #372 requests.
**Alternatives:**
- Keep in pages-diagram-core with documentation — was the previous D6 decision, but spec review R1-05 showed the move is safe: the dependency direction is diagram-core → graph-renderer, so moving export INTO graph-renderer means diagram-core imports from there with no cycle
- Re-export from graph-renderer — would create a cycle if graph-renderer depended on diagram-core, but the actual direction makes a move (not re-export) the right approach
**Rationale:** `exportDiagram()` queries `.react-flow__viewport` — this is React Flow DOM coupling that belongs in the package that owns React Flow rendering. Moving it consolidates all React Flow DOM access in graph-renderer and removes `html-to-image` from diagram-core.
**Trade-offs:** graph-renderer gains the `html-to-image` dependency (pinned at 1.11.11). Acceptable — export is a rendering concern.
**Sources:** casehubio/casehub-pages#372, spec review R1-05
**Exploration:** quick
**Status:** revised
**Supersedes:** Previous D6 (keep in diagram-core)

## D7: Standalone tool location — examples/

**Choice:** Add the standalone diagram export tool as a new example page in the examples/ directory.
**Alternatives:**
- In pages-diagram-core — diagram-core is a library, not an application entry point
- New top-level tool/ directory — adds build configuration for a single HTML page
**Rationale:** It's a tool built from existing components, not a library. The examples/ directory already has a dev server and build configuration for serving component demos.
**Trade-offs:** Discoverable as an example but may need separate documentation to surface as a tool for Claude sessions.
**Sources:** casehubio/casehub-pages#372 issue body (standalone HTML entry point)
**Exploration:** quick
**Status:** captured

## D8: Language intelligence architecture — LSP, separate issue

**Choice:** Context-aware completion for CaseHub YAML and jq expressions will be delivered via a Language Server Protocol (LSP) server, tracked as a separate issue. The `pages-code-editor` component ships now with basic YAML/JSON syntax highlighting. The component's `extensions` property provides the seam for plugging in an LSP client extension later.
**Alternatives:**
- Build completion directly into the web component — works for web but doesn't share with VS Code/IntelliJ plugins
- Ship everything together — delays the editor for months while the language server is built
**Rationale:** The long-term vision is IDE plugins for VS Code and IntelliJ that share language intelligence with the web editor. LSP is the industry standard for this. The web editor ships immediately with highlighting; completion arrives when the language server lands. VS Code and IntelliJ plugins use their native editors + the same LSP server.
**Trade-offs:** Context-aware completion is deferred from #372. The editor is useful without it (syntax highlighting, editing, read-only display), but the full vision requires a separate effort.
**Sources:** User requirement (VS Code/IntelliJ plugin target, context-aware jq and YAML completion)
**Exploration:** quick
**Status:** captured
