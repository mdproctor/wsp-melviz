---
title: "Teaching by typing"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
series: issue-443-scenario-tutorials
projects:
  - casehubio/casehub-pages
tags: [tutorials, scenario-executor, spi, editor-automation]
---

# Teaching by typing

The yaml-composition tutorial from #435 worked — the content was solid, the validation logic caught errors, all fifteen steps passed against the expand engine. But when I ran it in the browser, it was obvious the delivery was wrong. A tiny editor with a disabled Next button and no instructions. No narrative panel, no chapter navigation, no transport controls. It had reinvented tutorial infrastructure that already existed, and done it badly.

The existing scenario executor already types text character by character, highlights targeted elements with pulsing animations, and shows spotlight callouts with positioned text bubbles. The scenario controller already provides play/pause/step with a speed slider and a chapter outline. The narrative panel already renders markdown between automated sections. All of this was bypassed by the custom `yaml-editor` content type.

The fix was to make the scenario executor understand text editors.

## The SPI

A `ScenarioEditableText` interface — `insertText`, `replaceRange`, `setCursor`, `highlight`, `clearHighlights`, `triggerCompletion`. Any text editing surface implements it. The executor discovers it via `Symbol.for('scenario-editable-text')` on the DOM element, walking up through shadow DOM host boundaries to find the nearest ancestor bearing the symbol.

`pages-code-editor` is the first implementation. A `CodeEditorBridge` class translates between the SPI's `{line, col}` positions and CodeMirror 6's character offsets internally. The executor never imports CodeMirror — it calls `editor.insertText('forEach:\n  as: field')` and the bridge handles the dispatch.

Seven new ARIA actions plug into the existing parser and executor: `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`. Same YAML shorthand as `fill` and `click`. The parser's catch-all body passthrough means future fields don't need parser changes.

## What's next

The SPI and executor are done. What remains is content — rewriting the fifteen tutorial sections from the old yaml-editor format into standard hands-on scenarios with `editor-insert` steps, spotlight callouts, and slide sections for narrative. The tutorial will use the full scenario infrastructure: narrative panel on one side, builder workbench on the other, controller floating in the corner with play/pause and chapter progress. The user watches yaml-core build pages live, character by character.
