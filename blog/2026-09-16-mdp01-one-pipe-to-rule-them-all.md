---
title: "One Pipe to Rule Them All"
date: 2026-09-16
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
series: issue-449-verify-schema-completions
tags: [edit-pipeline, architecture, coordinator-pattern, undo, typescript]
---

The workbench had six entry points that mutated document state, three independent write-back paths, and no coordination between them. Every editor keystroke created a new PageDocument, killing undo history. Every model change replaced all editor content, destroying cursor position. Tree context menu actions fired events nobody was listening for.

I wanted one path in, one path out. Every mutation flows through `_applyEdit(origin, fn)`, every view sync through `_syncViews(origin)`. The origin tag — `'editor'`, `'tree'`, `'properties'`, `'palette'`, `'toolbar'` — tells the sync fan-out what to skip. Editor-originated changes skip pushing YAML back to the editor (because the editor already has the text). Everything else syncs everywhere.

The first problem was dual undo. PageDocument has its own undo stack and change notification, designed for standalone use. When the shell takes over coordination, both the shell's undo stack and PageDocument's internal stack record every mutation. Both fire notifications. We solved this with a coordinated-mode flag — a construction-time boolean that makes `_pushUndo()` and `_notify()` no-ops. The shell creates documents via `PageDocument.parseCoordinated()`, and the internal mechanisms go quiet. The ordering matters: enable coordinated mode *before* adding the coordinator, or you get a window where both fire.

The second problem was cursor destruction. The old `_pushYamlToEditor` replaced all editor content on every model change — full `view.dispatch({ changes: { from: 0, to: length, insert: newText } })`. We replaced this with `computeMinimalChanges`, a character-level prefix/suffix diff that produces a single CodeMirror `ChangeSpec` covering only the changed region. I started with a line-based diff (the design spec specified it), but the `\n` separator accounting for insertions and deletions was subtly wrong. Character-level matching is simpler and always correct.

The third problem was YamlSync — an intermediate class that listened for editor input events, debounced them, parsed the text, and swapped the document. It was doing the right thing, but it was a parallel path that bypassed the coordinator. We replaced it with `_handleEditorInput` (300ms debounce, diagnostics check for invalid YAML) and `_flushEditorSync` (immediate flush when a structured edit arrives during active typing). The flush-before-structured-edit pattern is the subtle part: if the user is typing in the editor and clicks a palette tile, the pipeline flushes pending text changes first, then applies the palette insert. Two undo snapshots — one for the flush, one for the insert. No recursion.

Property sources had a stale-reference bug hiding in them. When you select a tree node, the property panel creates `get data()` and `onChange()` closures that capture the resolved node. These work fine for in-place mutations — the node reference stays valid. But undo swaps the entire document instance. The captured reference now points to the old document. The fix: lazy `resolve()` closures that look up nodes from `this._document` at execution time. `_resolvePropertySource()` runs from `_syncViews` on every sync pass, so the property panel always reflects the current document.

The tree-add inline picker was the last piece. The `+` button on tree nodes was dispatching events into the void. We rendered `<pages-builder-inline-picker>` alongside the tree, connected it to `_applyEdit('tree', ...)`, and the three tests that had been RED since last session went green.

Along the way, Claude spotted a pre-existing bug in the property palette — every field label was rendering twice. The `_renderTagEditor` method was setting `el.label = label` on the input element AND wrapping it in a `<div class="field-label">${label}</div>`. Checkboxes were correct (they skip the wrapper div), but every other field type showed the name twice. A one-line fix: only set `el.label` for checkboxes.

The net result is a 200-line reduction (561 deleted, 361 added in production code) despite adding tree actions, inline picker wiring, and cursor-preserving diff-patch. YamlSync and its test file are gone. The architecture diagram from the design spec is now the actual code: edit sources → `_applyEdit` → PageDocument → `_syncViews` → views.

What's left on the workbench is polish: LSP integration with the tree-add picker, textarea fields for HTML template/javascript properties, and four tree actions that need picker UI (wrap-column, wrap-tabs, replace-with, add-child). The pipeline infrastructure is done — those are features that plug into it.
