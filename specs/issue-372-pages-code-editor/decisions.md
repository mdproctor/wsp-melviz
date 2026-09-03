## D1: Scope — all three deliverables

**Choice:** Implement all three deliverables in this branch: (1) pages-code-editor component, (2) re-export exportDiagram from graph-renderer, (3) standalone diagram export tool in examples/
**Alternatives:**
- Code editor only — smaller scope, but leaves the issue partially done
- Editor + standalone tool — skips export promotion, misses the graph-renderer public API improvement
**Rationale:** The standalone tool depends on the editor, and the export re-export is trivial. Completing all three closes #372 fully.
**Trade-offs:** Larger branch, but deliverables 2 and 3 are small increments once the editor exists.
**Sources:** casehubio/casehub-pages#372 issue body
**Exploration:** quick
**Status:** captured

## D2: Tokenizer extraction — move to pages-ui-components

**Choice:** Extract the YAML tokenizer (`tokenizeYamlLine`, `YamlToken`) from pages-aria into pages-ui-components as an internal module. Re-export from pages-aria to avoid breaking existing consumers.
**Alternatives:**
- Import from pages-aria — creates odd dependency direction (ui-components → aria/controller)
- Copy the tokenizer — two copies to maintain, drift risk
**Rationale:** The tokenizer is a pure function with no scenario-controller dependencies. It belongs with the editor component. Re-export preserves backward compatibility.
**Trade-offs:** Requires updating pages-aria's imports and adding a workspace dependency on pages-ui-components (or extracting to a shared location).
**Sources:** packages/pages-aria/src/controller/yaml-highlighter.ts, packages/pages-aria/src/controller/yaml-highlighter.test.ts
**Exploration:** quick
**Status:** captured

## D3: Edit mechanism — textarea overlay

**Choice:** Transparent textarea overlaid on a syntax-highlighted `<pre>` element. Same proven pattern used in blocks-ui.
**Alternatives:**
- Contenteditable — richer editing but worse a11y, browser inconsistencies, hard to maintain caret position during re-highlighting
- CodeMirror 6 — full-featured but ~150KB dependency, overkill for a syntax-highlighted textarea
**Rationale:** Native textarea gives free IME support, screen reader compatibility, and standard keyboard behavior. Scroll sync is manageable for typical use cases (< 200 lines).
**Trade-offs:** No advanced features (bracket matching, auto-indent, multi-cursor). Scroll sync requires careful CSS alignment.
**Sources:** casehubio/casehub-pages#372 issue body (describes the textarea overlay pattern)
**Exploration:** quick
**Status:** captured

## D4: Language extensibility — pluggable tokenizer

**Choice:** Accept a tokenizer function as a component property. Ship YAML and JSON tokenizers. The tokenizer interface is `(line: string) => Token[]`.
**Alternatives:**
- YAML only, extend later — simpler initial API but requires breaking change to add languages
**Rationale:** The interface is trivial to define (`tokenizer` property accepting a function). JSON is listed as a stretch goal in the issue. Designing it in now avoids an API-breaking change.
**Trade-offs:** Slightly more complex API surface (property instead of hardcoded behavior), but the default can be YAML.
**Sources:** casehubio/casehub-pages#372 issue body (stretch: JSON highlighting)
**Exploration:** quick
**Status:** captured

## D5: Package placement — pages-ui-components

**Choice:** Add pages-code-editor as a new component in the existing pages-ui-components package, following the `src/{name}/pages-{name}.ts` pattern.
**Alternatives:**
- New standalone package — more isolation but adds build config, package.json, tsconfig overhead for no benefit (no heavy unique dependencies)
**Rationale:** Follows established conventions. The component uses Lit 3, pages design tokens, and shadow DOM — identical stack to existing components. No new package infrastructure needed.
**Trade-offs:** pages-ui-components grows slightly larger, but it's already the component collection package.
**Sources:** packages/pages-ui-components/package.json, packages/pages-ui-components/src/ directory structure
**Exploration:** quick
**Status:** captured

## D6: Export promotion — re-export from graph-renderer

**Choice:** Keep exportDiagram() implementation in pages-diagram-core, re-export from graph-renderer's public API.
**Alternatives:**
- Move to graph-renderer — mixes html-to-image dependency into the Lit wrapper layer
- New pages-export-utils package — over-engineered for one function
**Rationale:** graph-renderer already depends on pages-diagram-core, so no new dependency edge. Consumers get export via the same package they use for rendering.
**Trade-offs:** Implementation stays in diagram-core, which is React-level code. But that's where the React Flow DOM knowledge lives anyway.
**Sources:** packages/pages-diagram-core/src/diagram-export.ts, packages/graph-renderer/
**Exploration:** quick
**Status:** captured

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
