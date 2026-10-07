# Milkdown Editor — Design Decisions

## D1: Component Architecture

**Choice:** Thin Milkdown wrapper + shared EditableText bridge
**Alternatives:**
- Editor host abstraction — single EditorHost manages both editors, handles mode switching and overlay state. Adds indirection between MCP and the editor.
- Reactive controller — DocumentController manages state, both editors are thin views. Clean separation but fights ProseMirror's own document model with a third representation layer.
**Rationale:** Follows the proven `pages-code-editor` pattern. `PagesMarkdownEditor` wraps Milkdown and implements `EditableText` (renamed from `ScenarioEditableText`) via a `MarkdownEditorBridge`. MCP tools target the `EditableText` interface, so they work with either editor without knowing the underlying engine. One API for editing and overlays in any line/column-based document.
**Trade-offs:** Some overlay implementation logic may be duplicated between `CodeEditorBridge` and `MarkdownEditorBridge`, though the shared interface keeps consumers uniform.
**Sources:** packages/pages-code-editor/src/code-editor-bridge.ts, packages/pages-aria/src/executor/editable-text.ts
**Exploration:** quick
**Status:** captured

## D2: Overlay API Location

**Choice:** Extend EditableText interface directly with overlay methods
**Alternatives:**
- Separate OverlayManager interface — cleaner separation but two interfaces to discover and manage, breaks the "one API" goal.
- Extension/plugin pattern — overlays injected as extensions. Minimal core but MCP tools need to know about extensions, not just the interface.
**Rationale:** The whole point is one API for editing and overlays in any line/column-based document. Highlights return IDs for targeted removal. Floating annotations (arrows, callouts, markers) are added/removed by ID.
**Trade-offs:** Interface grows, but all methods are editor-related and the alternative (splitting) creates friction for MCP tool consumers.
**Sources:** packages/pages-aria/src/executor/editable-text.ts
**Exploration:** quick
**Status:** captured

## D3: Concurrent Editing Strategy

**Choice:** Edit sessions with lock indicator, designed for future merge upgrade
**Alternatives:**
- Optimistic regions — LLM locks a region, human edits elsewhere. Complex region management, half a CRDT without consistency guarantees.
- Full CRDT (Yjs) from day one — truly collaborative but significant complexity, ~30KB added, overkill for one LLM + one human.
**Rationale:** `beginEditSession(owner)` / `endEditSession(session)` on EditableText. LLM takes a session, editor shows "AI editing" indicator, human can cancel. The `EditSession` abstraction is the upgrade path to merge — its contract can evolve from "lock" to "merge-capable session" without changing the MCP tool API.
**Trade-offs:** Human must wait during LLM edits (can watch changes stream in, can cancel). Acceptable for v1 since LLM edits are typically fast bursts.
**Sources:** None — greenfield, no existing concurrency code in the project.
**Exploration:** quick
**Status:** captured

## D4: MCP Tool Surface

**Choice:** Positional core tools + semantic resolution helpers with ambiguity reporting
**Alternatives:**
- Pure positional only — simpler but forces LLMs to do all position arithmetic themselves.
- Semantic helpers that also edit — fewer round-trips but less composable and introduces ambiguity in the edit tools themselves.
**Rationale:** Core tools (get/set content, replace_range, insert_text, highlight, annotate, edit sessions) use line/column positions. Resolution helpers (find_text, find_heading) return positions. If a helper finds ambiguity (multiple matches, no match), it reports that to the LLM, which drops to positional tools for precision. Helpers resolve, core tools act — clean layering.
**Trade-offs:** Extra round-trip when helpers succeed (resolve then edit), but composability and clarity outweigh the latency cost.
**Sources:** Existing McpBinding pattern in packages/yaml-core/src/step/types.ts
**Exploration:** quick
**Status:** captured
