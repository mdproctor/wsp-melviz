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

## D2: Tokenizer extraction — move to pages-primitives

**Choice:** Extract `tokenizeYamlLine` and `YamlToken` from pages-aria into pages-primitives as a new `syntax/` module. Leave `buildStepLineMap` and `LineRange` in pages-aria (scenario-specific, depends on the `yaml` package). Update pages-aria's internal imports to source the tokenizer from pages-primitives.
**Alternatives:**
- Move to pages-ui-components — wrong dependency direction; pages-aria re-exporting from pages-ui-components would pull in a massive transitive tree (pages-component, pages-data, pages-filter-bar, pages-table)
- New standalone pages-syntax package — cleanest architecturally (zero dependencies) but adds package.json/tsconfig overhead for ~70 lines
- Copy the tokenizer — two copies to maintain, drift risk
- Import from pages-aria — odd dependency direction (ui-components → aria/controller)
**Rationale:** Both pages-aria and pages-ui-components already depend on pages-primitives. The tokenizer is a pure function with zero runtime dependencies (not even Lit). pages-primitives is explicitly for shared Lit-level utilities and infrastructure. No new dependency edges are created. The tokenizer is not publicly exported from pages-aria, so there are no external consumers to preserve backward compatibility for.
**Trade-offs:** pages-primitives gains a non-Lit module (the tokenizer is pure TypeScript). This is architecturally acceptable since primitives already hosts type-only modules (AriaContract). The module split means pages-aria imports the tokenizer from pages-primitives and keeps buildStepLineMap locally.
**Sources:** packages/pages-aria/src/controller/yaml-highlighter.ts, packages/pages-aria/src/controller/yaml-highlighter.test.ts, packages/pages-primitives/package.json, decision review R1-03
**Exploration:** quick
**Status:** revised
**Depends on:** D5 (package placement — pages-ui-components consumes the tokenizer from pages-primitives)

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

## D4: Language extensibility — pluggable stateful tokenizer

**Choice:** Accept a tokenizer function as a component property using a stateful interface: `(line: string, state: S) => { tokens: Token[]; endState: S }`. Ship YAML and JSON tokenizers. Define a language-agnostic `SyntaxToken` type with a `type` string field (not a fixed union) so each language can define its own token types, and a `text` string field for the token content.
**Alternatives:**
- Stateless `(line: string) => Token[]` — simpler but cannot handle YAML multi-line constructs (block scalars `|`/`>`, multi-line quoted strings). The existing `tokenizeYamlLine` already has this bug. Adding state later would be a breaking API change.
- YAML only, extend later — simpler initial API but requires breaking change to add languages
- Fixed token type union (`key | string | comment | literal | punct | plain`) — works for YAML but JSON has different semantic distinctions (brackets, braces, null). A string-typed `type` field lets each language define its own vocabulary while the renderer maps types to CSS classes.
**Rationale:** A stateful interface costs nothing to define — tokenizers that don't need state use `undefined` as `S`. It prevents a breaking change when adding correct multi-line support. A string-typed token type allows language-specific vocabularies without coupling the component to any particular language's grammar.
**Trade-offs:** The YAML tokenizer MVP can still be line-stateless (pass `undefined` through), but the interface accommodates future correctness. The renderer needs a generic token-type-to-CSS-class mapping rather than a hardcoded set.
**Sources:** casehubio/casehub-pages#372 issue body (stretch: JSON highlighting), decision review R1-04 (multi-line constructs), decision review R1-10 (token type union)
**Exploration:** quick
**Status:** revised

## D5: Package placement — pages-ui-components

**Choice:** Add pages-code-editor as a new component in the existing pages-ui-components package, following the `src/{name}/pages-{name}.ts` pattern.
**Alternatives:**
- New standalone package — more isolation but adds build config, package.json, tsconfig overhead for no benefit (no heavy unique dependencies)
**Rationale:** Follows established conventions. The component uses Lit 3, pages design tokens, and shadow DOM — identical stack to existing components. No new package infrastructure needed.
**Trade-offs:** pages-ui-components grows slightly larger, but it's already the component collection package.
**Sources:** packages/pages-ui-components/package.json, packages/pages-ui-components/src/ directory structure
**Exploration:** quick
**Status:** captured

## D6: Export promotion — keep in pages-diagram-core, document

**Choice:** Keep exportDiagram() in pages-diagram-core. Do not re-export from graph-renderer. Instead, document in graph-renderer's README that export functionality is available via `@casehubio/pages-diagram-core`.
**Alternatives:**
- Re-export from graph-renderer — creates a circular dependency (pages-diagram-core depends on graph-renderer, not the other way around)
- Move to graph-renderer — adds html-to-image dependency to a pure React Flow/ELK rendering bridge
- New pages-export-utils package — over-engineered for one function
**Rationale:** The actual dependency direction is pages-diagram-core → graph-renderer. Re-exporting from graph-renderer would require adding pages-diagram-core as a dependency of graph-renderer, creating a cycle. The function depends on React Flow DOM internals (.react-flow__viewport) and html-to-image — both belong in diagram-core.
**Trade-offs:** Consumers must know to import from pages-diagram-core for export. Documentation bridges the discoverability gap.
**Sources:** packages/pages-diagram-core/package.json (depends on graph-renderer), packages/graph-renderer/package.json (no diagram-core dependency), decision review R1-02
**Exploration:** quick
**Status:** revised

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
