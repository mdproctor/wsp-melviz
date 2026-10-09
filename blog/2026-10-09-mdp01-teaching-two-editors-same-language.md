---
layout: post
title: "Teaching Two Editors the Same Language"
date: 2026-10-09
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [typescript, prosemirror, milkdown, codemirror, editor, mcp]
---

# Teaching Two Editors the Same Language

CaseHub needed a rich markdown editor — not just for content authoring, but as a surface that LLM agents can read from, write to, and point at. The existing `CodeEditorBridge` already gave CodeMirror a programmatic API for scenario playback. The challenge was making a second editor engine — ProseMirror via Milkdown — speak the same interface, so the MCP tool layer doesn't care which editor is underneath.

I chose Milkdown because it's markdown-first. Most ProseMirror wrappers treat markdown as an import/export format. Milkdown treats it as the canonical representation — the schema maps directly to markdown constructs, and the serializer round-trips cleanly. That matters when an LLM agent is editing: it thinks in markdown, and position mismatches between the agent's model and the editor's internal state produce silent corruption.

The core design decision was an `EditableText` interface backed by an abstract `EditableTextBridge`. Both `CodeEditorBridge` (CodeMirror) and `MarkdownEditorBridge` (ProseMirror) extend it. The interface covers content access, search, editing, cursor, highlights, annotations, and edit sessions. Sixteen MCP tools map to these methods through a single `McpToolAdapter` that takes `EditableText` — the adapter doesn't know which engine is running.

The trickiest part was highlights. ProseMirror's `Decoration.inline` and CodeMirror's `StateEffect`/`StateField` are completely different mechanisms for the same visual result. The bridge pattern handles this cleanly — the base class tracks highlight state by ID, and each subclass implements `applyHighlight`/`removeHighlightDecoration`/`clearHighlightDecorations` using its engine's native API. Four styles — pulse, underline, glow, box — all resolved to inline CSS on ProseMirror and class-based decorations on CodeMirror.

Visual line highlighting was harder than expected. ProseMirror has no concept of a "visual line" — its document model is block-level (paragraphs, headings, list items), and a long paragraph that wraps across three rendered lines is a single node. I needed to highlight exactly the rendered line at the cursor position, adapting to window width. Five commits went into this before landing on `Range.getClientRects()` — which returns one rect per inline fragment, groupable by Y-band into visual lines — combined with `posAtCoords` at the left and right edges of the cursor's visual line band to map pixel boundaries back to document positions. The earlier attempts using `coordsAtPos` Y-walking failed because coordinates shift with scroll state and inline code marks.

The third new package, `pages-document-diff`, was an extraction from blocks-ui. The original `DocumentDiff` component (1217 lines) had domain-generic infrastructure tangled with DraftHouse-specific thread, timeline, and selection logic. I pulled the generic core out — LCS diff, word-level highlights, canvas minimap, heading-based scroll sync, split/unified views — into a standalone base class. The blocks-ui component became a 162-line subclass keeping only its domain logic. The scroll sync pattern from the extraction also drove the split-mode implementation in the markdown editor.

The edit session model is deliberately simple: exclusive lock with snapshot rollback. An LLM agent calls `beginEditSession`, gets exclusive write access, and the editor takes a full state snapshot. Cancel restores the snapshot — including undo history, so the user can't redo cancelled changes. This is the right v1. Collaborative editing is a future capability that requires explicit opt-in and conflict resolution; pretending it's a simple upgrade would have been a mistake.

Forty-five commits, three new packages, 225 tests. The highlight system is functional but basic — fixed styles, no animation, no read-back. Issue #535 extends it with configurable styling, semantic targeting, animated overlays, and a line reader for LLM-synchronized document navigation.
