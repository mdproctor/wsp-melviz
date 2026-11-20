---
title: "Teaching by typing"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
series: issue-443-scenario-tutorials
projects:
  - casehubio/casehub-pages
tags: [tutorials, scenario-executor, spi, editor-automation, hands-on]
---

# Teaching by typing

The yaml-composition tutorial from #435 worked — the content was solid, the validation logic caught errors, all fifteen steps passed against the expand engine. But when I ran it in the browser, it was obvious the delivery was wrong. A tiny editor with a disabled Next button and no instructions. No narrative panel, no chapter navigation, no transport controls. It had reinvented tutorial infrastructure that already existed, and done it badly.

The existing scenario executor already types text character by character, highlights targeted elements with pulsing animations, and shows spotlight callouts with positioned text bubbles. The scenario controller already provides play/pause/step with a speed slider and a chapter outline. The narrative panel already renders markdown between automated sections. All of this was bypassed by the custom `yaml-editor` content type.

The fix was to make the scenario executor understand text editors.

## The SPI

A `ScenarioEditableText` interface — `insertText`, `replaceRange`, `setCursor`, `highlight`, `clearHighlights`, `triggerCompletion`. Any text editing surface implements it. The executor discovers it via `Symbol.for('scenario-editable-text')` on the DOM element, walking up through shadow DOM host boundaries to find the nearest ancestor bearing the symbol.

`pages-code-editor` is the first implementation. A `CodeEditorBridge` class translates between the SPI's `{line, col}` positions and CodeMirror 6's character offsets internally. The executor never imports CodeMirror — it calls `editor.insertText('forEach:\n  as: field')` and the bridge handles the dispatch.

Seven new ARIA actions plug into the existing parser and executor: `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`. Same YAML shorthand as `fill` and `click`. The parser's catch-all body passthrough means future fields don't need parser changes.

## The content rework

With the executor ready, the fifteen tutorial sections converted from the old `yaml-editor` format — `initialYaml`, `expectedKeys`, `solutionYaml` — into standard `hands-on` scenarios. Each section now has a narrative slide (the existing markdown, reframed from "Your task:" to "Watch:") and ARIA steps that drive the editor.

The strongest sections are the ones with contrast. ForEach Basics types out three identical components, highlights the repetition, spotlights a callout — "Three identical components, only the field value differs" — then clears and types the `forEach` version. The tree view updates and a second spotlight confirms: "forEach generated three components from one template." The Modules section does the same: two copy-pasted dashboards, then a module definition with two imports. The before-and-after pattern turns a code diff into a live demonstration.

Simpler sections just type the solution. Variables types out a `variables:` block and `${app.name}` references. Conditionals types `when:` clauses. The progressive typing algorithm accelerates through five phases — character by character, then word chunks, then larger batches — so short YAML is watchable and long YAML doesn't drag.

## The host

The tutorial host needed builder-shell in the DOM for the ARIA tree walker to find the code editor. The existing `_renderTutorial()` rendered narrative and controller but no application surface — only the `yaml-editor` path rendered builder-shell.

Rather than hardcoding builder-shell into every `hands-on` tutorial, I added a `target` field to `TutorialDescriptor`. When `target: 'builder-shell'` is set, the tutorial layout renders builder-shell on the left (flex 3) and narrative plus controller on the right (flex 2). Other hands-on tutorials — the form-automation tutorial, for instance — are unaffected. The ARIA tree walker searches from `document.body` through all shadow roots, so the editor's textbox is findable regardless of where builder-shell sits in the DOM hierarchy.

## What's still out

`editor-completion` is wired but gracefully skips — LSP isn't connected to the workbench editor yet. When it is, the tutorial content can add autocomplete steps that trigger the popup and select completions. The existing skip-during-progressive-typing mechanism (the `finishFn` callback) is declared but not yet invoked — transport skip during character-by-character typing won't fast-forward. Both are follow-up items that don't block the core tutorial experience.

The `yaml-editor` validation infrastructure stays intact for a future user-practice mode where the viewer edits and gets feedback. The hands-on tutorials are watch-only — the user learns by observation, then practices independently.
