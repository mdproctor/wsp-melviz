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
