# Milkdown Editor — Design Decisions

## D0: Editor Engine Selection

**Choice:** Milkdown as the WYSIWYG markdown engine
**Alternatives:**
- Tiptap — most widely adopted ProseMirror wrapper (MIT since v2). Large plugin ecosystem (~500K weekly npm downloads), first-party collaboration support. Has React, Vue, and vanilla JS bindings — no Lit binding (same as Milkdown). Ecosystem advantage is real but driven by React projects. Tiptap is a general-purpose rich text editor that CAN handle markdown but isn't designed around it — markdown support requires extension configuration.
- BlockNote — purpose-built Notion-like block editor with built-in slash commands and drag handles. Too opinionated for our needs: we require deep extensibility for scenario engine integration, MCP tool overlays, and custom annotation layers.
- ProseMirror directly — maximum control, no framework overhead. But we'd reimplement markdown↔ProseMirror conversion, plugin lifecycle, and toolbar infrastructure that Milkdown provides. "Thin wrapper" means thin over Milkdown's infrastructure, not thin over raw ProseMirror.
- Lexical (Meta) — lighter weight, framework-agnostic. Younger ecosystem with fewer markdown-oriented plugins. ProseMirror's markdown ecosystem is substantially more mature.
**Rationale:** Milkdown was selected as a user requirement for this spec. The rationale is sound: Milkdown is **markdown-first** — its ProseMirror schema is derived from the markdown spec via remark/mdast, meaning the editor's document model IS the markdown structure. For a markdown editor with MCP tool integration, this architectural alignment is the decisive factor. Milkdown's `@milkdown/kit` package provides a framework-agnostic core that can be mounted to any DOM element (including inside a Lit component's shadow DOM) without requiring React, Vue, or any framework binding. Neither Milkdown nor Tiptap offers a Lit binding — both require custom integration with a Lit component. With equal integration effort, Milkdown's markdown-first design wins for a markdown editor. Its smaller community (vs Tiptap) is offset by direct ProseMirror access when Milkdown's abstractions don't reach — we're not locked in.
**Framework integration note:** Milkdown's official framework integrations are `@milkdown/react`, `@milkdown/vue`, and `@milkdown/integrations/solidjs`. There is no `@milkdown/lit` package. Milkdown's UI component layer (`@milkdown/components`) and pre-configured editor (`@milkdown/crepe`) use Vue 3 internally. Our integration uses `@milkdown/kit` (framework-agnostic core) directly, with a custom Lit toolbar — see D6.
**Trade-offs:** Smaller community than Tiptap means less ecosystem support for edge cases. Mitigated by ProseMirror escape hatch. Custom Lit toolbar required (no pre-built Lit-compatible toolbar from either Milkdown or Tiptap).
**Sources:** milkdown.dev, npmjs.com/@milkdown/kit, user requirement for this spec
**Exploration:** user-requirement (surfaced by R1-02 review)
**Status:** revised (R2-01: corrected factual error — removed non-existent @milkdown/lit reference, revised rationale from framework binding to markdown-first design as key differentiator)

## D1: Component Architecture

**Choice:** Thin Milkdown wrapper + shared EditableText bridge with line/column position model
**Alternatives:**
- Editor host abstraction — single EditorHost manages both editors, handles mode switching and overlay state. Adds indirection between MCP and the editor.
- Reactive controller — DocumentController manages state, both editors are thin views. Clean separation but fights ProseMirror's own document model with a third representation layer.
- Offset-based position model — use ProseMirror's flat integer offsets instead of line/column. Natural for ProseMirror but alien to MCP tools and scenario engine (which work with markdown text) and requires CodeMirror to translate to offsets.
- Dual addressing (both line/col and offset) — exposes two parallel APIs for the same operations, forcing consumers to choose and creating surface area for inconsistency.
**Rationale:** Follows the proven `pages-code-editor` pattern. `PagesMarkdownEditor` wraps Milkdown and implements `EditableText` (renamed from `ScenarioEditableText`) via a `MarkdownEditorBridge`. MCP tools target the `EditableText` interface, so they work with either editor without knowing the underlying engine. The line/column position model is correct because the canonical content is markdown text — a line-oriented format. MCP tools and the scenario engine address markdown positions, not ProseMirror tree positions. The bridge's job is to translate between line/col and ProseMirror's internal offset model, exactly as `CodeEditorBridge` already does with `toOffset()`/`toPosition()` methods. For complex constructs (tables, nested lists), the bridge serializes the ProseMirror doc model to markdown and computes positions against the serialized text.
**Interface duplication:** The `ScenarioEditableText` interface is currently declared identically in both `pages-aria/src/executor/editable-text.ts` (lines 6–19) and `pages-code-editor/src/code-editor-bridge.ts` (lines 9–22). `CodeEditorBridge` redeclares locally because `pages-code-editor` cannot depend on `pages-aria` — the ARIA executor is a higher-level concern than the editor component. This duplication is exactly what motivates extracting the interface to `pages-editor-core` (D8), which both packages can import from.
**Trade-offs:** Some overlay implementation logic may be duplicated between `CodeEditorBridge` and `MarkdownEditorBridge`, though the shared interface keeps consumers uniform. ProseMirror bridge translation adds complexity for WYSIWYG content with non-linear line mapping, but this is encapsulated in the bridge implementation.
**Sources:** packages/pages-code-editor/src/code-editor-bridge.ts, packages/pages-aria/src/executor/editable-text.ts
**Exploration:** quick
**Status:** revised (R1-02: added D0 for engine selection; R1-03: explicit rationale for line/col model with offset alternative rejected; R1-07: acknowledged interface duplication and dependency direction constraint)

## D2: Overlay API Location

**Choice:** Extend EditableText interface with overlay methods — highlight IDs for targeted removal, annotation methods with separate lifecycle semantics
**Alternatives:**
- Separate OverlayManager interface — cleaner separation but two interfaces to discover and manage, breaks the "one API" goal.
- Extension/plugin pattern — overlays injected as extensions. Minimal core but MCP tools need to know about extensions, not just the interface.
- DecorationProvider pattern (ProseMirror-style) — separates "what to decorate" from "how to render." Leaks ProseMirror internals into a cross-engine interface.
**Rationale:** The whole point is one API for editing and overlays in any line/column-based document. The interface includes two distinct method families: (1) **Text decorations** (highlights) — anchored to document positions, moved with text on edits, invalidated when underlying text is deleted. `highlight()` returns a string ID; `removeHighlight(id)` removes a specific one; `clearHighlights()` removes all. (2) **Floating annotations** (arrows, callouts, markers) — positioned relative to document elements or positions, repositioned by the bridge on scroll/resize, not invalidated by text edits unless their anchor element is removed. `addAnnotation()` returns an ID; `removeAnnotation(id)` removes it. Both share the same interface but the bridge manages their lifecycles differently. This is invisible to consumers — they add and remove by ID.
**Breaking change:** Adding a return value to `highlight()` (void → string) is a TypeScript-compatible change — existing callers that discard the return still compile. The existing `editorHighlight` function in `command-executor.ts` (line ~173) does not capture the return and will continue to work unchanged.
**Trade-offs:** Interface grows, but all methods are editor-related. The two lifecycle models (text-anchored vs element-anchored) are managed by the bridge, not exposed to consumers.
**Sources:** packages/pages-aria/src/executor/editable-text.ts
**Exploration:** quick
**Status:** revised (R1-08: separated highlight and annotation lifecycle semantics, acknowledged breaking change, added DecorationProvider alternative)

## D3: Concurrent Editing Strategy

**Choice:** Edit sessions with lock indicator and explicit cancel semantics. Future collaborative editing is a new capability, not an implicit upgrade.
**Alternatives:**
- Optimistic regions — LLM locks a region, human edits elsewhere. Complex region management, half a CRDT without consistency guarantees.
- Full CRDT (Yjs) from day one — truly collaborative but significant complexity, ~30KB added, overkill for one LLM + one human.
- Implicit upgrade from lock to merge — claiming the session contract can silently evolve from exclusive to collaborative. This is architecturally unsound because a lock session and a merge session have opposite guarantees (exclusive access vs concurrent reconciliation), and consumers that assume exclusive access would silently break.
**Rationale:** `beginEditSession(owner, mode?)` / `endEditSession(session)` on EditableText. v1 mode is `exclusive` (default) — LLM takes a session, editor shows "AI editing" indicator, human can watch but not edit. Cancel semantics: human clicks cancel → session is rolled back to the document state at `beginEditSession()`. The bridge snapshots the ProseMirror doc state at session start and restores on cancel. Partial edits are discarded entirely. Future collaborative mode (`mode: 'collaborative'`) is opt-in — existing callers get exclusive by default. Upgrading to collaborative requires callers to explicitly request it and handle conflict resolution. This is a new capability, not contract evolution.
**Trade-offs:** Human must wait during LLM edits (can watch changes stream in, can cancel with full rollback). Acceptable for v1 since LLM edits are typically fast bursts. Snapshot/restore has memory cost proportional to document size — negligible for markdown documents.
**Sources:** None — greenfield, no existing concurrency code in the project.
**Exploration:** quick
**Status:** revised (R1-09: honest framing of lock→merge path as new capability, added explicit cancel semantics with rollback)

## D4: MCP Tool Surface

**Choice:** Positional core tools + semantic resolution helpers with ambiguity reporting
**Alternatives:**
- Pure positional only — simpler but forces LLMs to do all position arithmetic themselves.
- Semantic helpers that also edit — fewer round-trips but less composable and introduces ambiguity in the edit tools themselves.
- Compound semantic operations (e.g. `replace_heading("Introduction", newContent)`) — eliminates the resolve-then-edit round-trip but creates a larger, less composable tool surface. Each compound operation is a special case; the combinatorial explosion of "resolve X then do Y" operations is unsustainable.
**Rationale:** Core tools (get/set content, replace_range, insert_text, highlight, annotate, edit sessions) use line/column positions. Resolution helpers (find_text, find_heading) return positions. If a helper finds ambiguity (multiple matches, no match), it reports that to the LLM, which drops to positional tools for precision. Helpers resolve, core tools act — clean layering. The extra round-trip when helpers succeed is negligible in practice — MCP calls within the browser are sub-millisecond. For a typical 5-section document revision, 10 MCP calls vs 5 adds microseconds of overhead, not meaningful latency.
**Trade-offs:** Extra round-trip when helpers succeed (resolve then edit), but composability and clarity outweigh the negligible latency cost.
**Sources:** pages-aria/src/executor/command-executor.ts (resolveEditor → findEditableText → method calls)
**Exploration:** quick
**Status:** revised (R1-10: fixed source attribution from yaml-core McpBinding to pages-aria command-executor, added compound operations alternative, defended composability over latency)

## D5: Dual Mode (WYSIWYG ↔ Source)

**Choice:** Own toggle using pages-code-editor for source mode, with position-aware scroll sync
**Alternatives:**
- @milkdown-lab/plugin-split-editing community plugin — demonstrated in the official playground but already shows limitations (e.g. no sync scrolling) that we've solved in our own editors. Low-activity (1 maintainer, year-old release).
- WYSIWYG only for v1 — simpler but users expect source access.
**Rationale:** Build the split/toggle ourselves, swapping between Milkdown (WYSIWYG) and pages-code-editor (source). We control both sides and can implement sync scrolling, cursor position preservation on toggle, and other polish the community plugin lacks. The EditableText bridge swaps from MarkdownEditorBridge to CodeEditorBridge on toggle — MCP tools continue working through the same interface. Can reference the community plugin's sync logic as prior art for the bidirectional ProseMirror↔markdown serialization.
**Serialization caveat:** The toggle involves serializing ProseMirror's doc model to markdown (for source mode) and parsing markdown back to ProseMirror's doc model (for WYSIWYG mode). This round-trip is lossy for some edge-case constructs (e.g. non-standard HTML in markdown, certain whitespace patterns). Cursor position is preserved by mapping through the markdown text — approximate, not pixel-perfect. "Seamless" refers to the MCP tool contract (same interface, same operations), not to visual continuity.
**Trade-offs:** Two editor engines loaded in the component (see D9 for loading strategy). Sync logic between ProseMirror doc model and raw markdown on toggle (serialize/deserialize). Acceptable given we already own both components and have solved similar problems.
**Scroll sync:** Reuse the heading-based anchor pairing from `document-diff`, but replace the linear interpolation with position-aware interpolation. Document-diff's linear interpolation works when both panels have similar content topology (both rendered markdown). For WYSIWYG↔source, the height ratio between anchors differs dramatically — a heading section with a large image takes 300px in WYSIWYG but one line in source. The scroll sync engine must query actual element positions in both panels and interpolate based on viewport-relative positions, not linear fraction. The heading-based anchor pairing is still correct (structural headings match between views); only the interpolation strategy changes.
**Depends on:** D1 (EditableText bridge makes the swap transparent to MCP tools)
**Sources:** @milkdown-lab/plugin-split-editing (reference only), packages/pages-code-editor/, blocks-ui/components/document-workbench/src/document-diff.ts (scroll sync anchor pairing — not interpolation — as prior art)
**Exploration:** deep-analysis
**Status:** revised (R1-11: replaced linear interpolation with position-aware interpolation for WYSIWYG↔source height differences; R1-17: qualified "seamless" claim to MCP contract, not visual continuity)

## D6: Toolbar and Formatting Icons

**Choice:** Custom Lit toolbar dispatching Milkdown commands via `@milkdown/kit`
**Alternatives:**
- Use `@milkdown/crepe` (pre-built toolbar with Vue 3 internally) — batteries-included, minimal setup. But `@milkdown/crepe` bundles Vue 3 as an internal dependency (~30–40KB gz additional). Adding Vue as a third framework (alongside Lit and React/React Flow) increases hidden complexity. The pre-built Vue toolbar cannot be styled with pages design tokens without overriding Vue component internals.
- Wrap `@milkdown/components` Vue toolbar in Lit — `@milkdown/components` is Vue 3-based. This requires Vue as a runtime dependency and creates an awkward Vue-inside-Lit layering. Same design token integration problem as Crepe.
- Use ProseMirror's `prosemirror-menu` directly — bypasses Milkdown's command system, simpler Lit integration but loses Milkdown plugin coordination (plugin commands are registered with Milkdown's context, not ProseMirror's menu directly).
**Rationale:** Milkdown's core (`@milkdown/kit`) is framework-agnostic — it exposes ProseMirror commands for every formatting action (`toggleBoldCommand`, `wrapInHeadingCommand`, `insertTableCommand`, etc.) through its plugin context. The editor view mounts to any DOM element. A custom Lit toolbar builds on this foundation: each toolbar button is a Lit component styled with pages design tokens that calls the corresponding Milkdown command via `ctx.get(commandsCtx).call(commandName)`. This approach gives native design token integration (toolbar buttons are pages Lit components), zero framework overhead (no Vue/React runtime), and full control over toolbar layout and interaction patterns. The toolbar renders: bold, italic, headings, lists, code blocks, links, tables, math (KaTeX), diagrams (Mermaid), task lists, strikethrough, images.
**Implementation effort:** Building the Lit toolbar requires creating ~15–20 button components and a toolbar container. Each button is a thin wrapper: icon + click handler that dispatches a Milkdown command. The commands already exist in Milkdown's plugin system — we're building the UI, not the logic. Comparable effort to other pages-ui-components (badge, button, checkbox, etc.).
**Fallback:** If custom toolbar proves too costly, `@milkdown/crepe` can be used as a self-contained editor with Vue 3 bundled internally. Crepe works with vanilla JS (`new Crepe({ root: element })`), so it mounts inside Lit components without a framework binding. The trade-off is ~30–40KB additional gzipped weight and reduced design token control.
**Lazy loading:** KaTeX (~130KB gz) and Mermaid (~400KB gz) must NOT be eagerly loaded. They are loaded on demand — KaTeX when content contains `$$` or `$` math delimiters, Mermaid when content contains ` ```mermaid ` fenced code blocks. Toolbar buttons for math/diagram trigger the respective lazy import before inserting the block.
**Trade-offs:** More upfront implementation work than using Crepe's pre-built toolbar. Offset by native design token integration, zero hidden framework dependencies, and full toolbar customization for MCP tool integration (e.g. "AI editing" indicator from D3).
**Sources:** milkdown.dev, npmjs.com/@milkdown/kit, npmjs.com/@milkdown/crepe
**Exploration:** quick
**Status:** revised (R1-12: added lazy loading strategy for KaTeX/Mermaid; R2-01: corrected factual error — removed non-existent @milkdown/lit, replaced with custom Lit toolbar + @milkdown/kit commands, documented @milkdown/crepe as fallback)

## D7: Extract generic diff infrastructure from document-diff

**Choice:** Extract generic diff/split-view infrastructure from `document-diff` into pages; leave domain-specific review features in blocks-ui
**Alternatives:**
- Move document-diff as-is into pages — imports Drafthouse-specific review concepts (thread management, timeline snapshots, debate session endpoints) into the platform layer, violating the three-layer model ("pages knows nothing about domain concepts").
- Leave document-diff in blocks-ui, duplicate the generic infrastructure — duplicates ~660 lines of generic diff/scroll/split-view code.
**Rationale:** Reading the actual `document-diff` source (blocks-ui/components/document-workbench/src/document-diff.ts, 1217 lines) reveals domain-specific logic that was not acknowledged in the original decision:
- **Thread management** (lines 51–57, 367–408, 626–651): `_threadAnchors` array with `threadId`/`side`/`startLine`/`endLine`/`status`, listeners for `thread-created`/`thread-resolved`/`thread-focused` events, `_renderThreadGutterMarkers()` — Drafthouse document-review concepts.
- **Timeline snapshot fetching** (lines 464–478): `timeline-comparison-changed` event handler fetching from `/api/debate/{sessionId}/snapshot/{index}` — Drafthouse debate session endpoints.
- **Selection-to-thread bridge** (lines 429–462): `selection-changed` events with `side: A/B` semantics for the review panel.
The **generic** parts (~660 lines) — LCS diff engine, word-level highlights, canvas minimap, heading-based scroll sync, divider drag, drop zones, markdown rendering, diff navigation, view modes — are cleanly separable. These move to pages as reusable infrastructure. The **domain** parts (~130 lines) remain in blocks-ui, either as event listeners on a Drafthouse-specific wrapper component or as a subclass that extends the pages infrastructure.
**Trade-offs:** More work than a wholesale move. Drafthouse needs to update to use the extracted infrastructure + its own review layer. Worth it — the alternative imports domain logic into the platform layer.
**Sources:** blocks-ui/components/document-workbench/src/document-diff.ts, platform docs/platform/ui-architecture.md (three-layer model: "pages knows nothing about domain concepts")
**Exploration:** quick
**Status:** revised (R1-13: acknowledged domain-specific logic in document-diff, changed from wholesale move to clean extraction)

## D8: Shared editor infrastructure in pages-editor-core

**Choice:** New `pages-editor-core` package, starting with the interface contract and growing iteratively as implementations emerge
**Alternatives:**
- Extend pages-primitives — keeps package count down but widens primitives' scope beyond UI primitives into editor-specific domain logic.
- Expand pages-code-editor — name no longer matches scope, creates confusing dependency where Milkdown depends on a package called "code-editor."
- Start with `pages-editor-api` (interface only) — too narrow; the MCP tool adapter and bridge base class are tightly coupled to the interface and should ship together.
- Pre-design all seven responsibilities before implementation — risks building abstractions before having two concrete implementations to abstract over.
**Rationale:** The package provides an iterative foundation:
**Initial scope (ships with D1 implementation):**
- `EditableText` interface (renamed from `ScenarioEditableText`) + `EDITABLE_TEXT` symbol + discovery functions — resolves the current duplication between pages-aria and pages-code-editor
- `EditableTextBridge` abstract base class — shared highlight ID management, edit session state. `CodeEditorBridge` and `MarkdownEditorBridge` extend it. Engine-specific decoration logic stays in each bridge implementation (CodeMirror StateEffect/StateField vs ProseMirror DecorationSet+plugin).
- MCP tool adapter — maps MCP tool calls to EditableText methods, handles ambiguity reporting for semantic helpers. Replaces the pattern in pages-aria's command-executor without creating a dependency on pages-aria.

**Extracted during implementation (grows as D2, D3, D5, D7 are built):**
- Scroll sync engine — extracted from document-diff during D5/D7 implementation, once the position-aware interpolation is proven
- Overlay renderer — floating annotations extracted when D2 is implemented and both bridges have annotation support
- Split view container — extracted from document-diff during D7 extraction, as a composable Lit component in pages-editor-core (Lit dependency is acceptable — pages-primitives already establishes Lit at the component level)
- Edit session manager — extracted when D3 is implemented

`pages-code-editor`, `pages-markdown-editor`, and the extracted `document-diff` all depend on `pages-editor-core`. Each editor package is a thin wrapper providing the engine-specific bridge implementation.
**Trade-offs:** Adds a new package to the monorepo. Requires refactoring `CodeEditorBridge` to extend the new base class. Initial package is smaller than originally specified — additional responsibilities are extracted when proven in implementation rather than pre-designed.
**Depends on:** D1 (bridge pattern), D2 (overlay in EditableText), D3 (edit sessions), D5 (scroll sync extraction), D7 (document-diff extraction)
**Sources:** packages/pages-code-editor/src/code-editor-bridge.ts, packages/pages-aria/src/executor/editable-text.ts, blocks-ui/components/document-workbench/src/document-diff.ts
**Exploration:** quick
**Status:** revised (R1-14: narrowed initial scope to interface+bridge+MCP adapter; deferred scroll sync, overlay renderer, split view, edit session manager to iterative extraction during implementation; R1-16: D8→D7 dependency updated to reflect D7's clean extraction)

## D9: Bundle and Loading Strategy

**Choice:** Lazy-load inactive editor engine and heavy dependencies on demand
**Alternatives:**
- Eager loading of both engines — simplest implementation but ~700KB+ gzipped payload for the editor component alone (ProseMirror ~100KB + CodeMirror ~60KB + KaTeX ~130KB + Mermaid ~400KB).
- Single engine only (WYSIWYG-only or source-only) — eliminates the dual-engine weight but removes core functionality.
**Rationale:** The dual-mode editor (D5) loads two editor engines. Consumers embedding the editor (Drafthouse, Claudony) already have their own bundle weight. Loading strategy:
- **Primary engine (Milkdown/ProseMirror):** loaded eagerly — this is the default view.
- **Source engine (CodeMirror):** lazy-loaded on first toggle to source mode. The `pages-code-editor` package is dynamically imported.
- **KaTeX:** lazy-loaded when content contains math delimiters (`$$`, `$`) or when user clicks the math toolbar button.
- **Mermaid:** lazy-loaded when content contains ` ```mermaid ` blocks or when user clicks the diagram toolbar button.
- **Milkdown plugins:** grouped by feature tier. Core formatting (bold, italic, headings, lists) is eager. Advanced features (tables, task lists, strikethrough) are loaded with the editor. Heavy features (math, diagrams) are lazy per above.
**Trade-offs:** First toggle to source mode incurs a load delay (~60KB fetch + parse). First math/diagram insertion incurs load delay. Both are one-time costs with browser caching. The UX trade-off (sub-second delay on first use) is acceptable compared to 700KB+ upfront.
**Sources:** No prior art in the project — greenfield loading strategy.
**Exploration:** surfaced-by-review (R1-04)
**Status:** captured
