---
title: "The editor that understands YAML"
date: 2026-09-05
author: mdp
entry_type: note
subtype: diary
series: casehub-pages
tags: [code-editor, codemirror, yaml, completion, lsp]
refs: ["casehubio/casehub-pages#372", "casehubio/casehub-pages#407", "casehubio/casehub-pages#408"]
---

Built `<pages-code-editor>` — a CodeMirror 6 web component that wraps a
full editing experience inside a Lit element with shadow DOM. The starting
point was simple: extract the inline YAML highlighter from blocks-ui into
a reusable component. The ending point was different.

The conversation shifted when Mark asked about context-aware completion
for jq expressions and CaseHub YAML. That changed the architecture from
a textarea overlay to CodeMirror 6, and from a utility component to the
front end of a Language Server Protocol pipeline. The editor ships now
with syntax highlighting and YAML parse validation. Completion is
deferred to a schema-driven approach (#408) backed by the case
definition's existing JSON Schema.

Three things that weren't obvious going in:

**stripTs is a trap.** The gallery's companion script executor strips
TypeScript syntax via regex before eval. Any YAML value starting with a
capital letter — `Revenue`, `Orders`, `Customers` — gets silently eaten
because the regex thinks `: Revenue` is a type annotation. The fix:
embed YAML content in hidden `<pre>` elements in the dashboard YAML, or
fetch from a separate file.

**CodeMirror needs drawSelection() in shadow DOM.** Without it, the
browser's native caret doesn't render inside the shadow root. No error,
no warning — the cursor is simply invisible. Adding `drawSelection()`
and bumping the cursor width to 2px made it reliably visible.

**Completion without a schema is worse than no completion.** The first
attempt hardcoded CaseHub dashboard YAML keys. It worked for dashboard
YAML — until someone opened a case definition YAML and got `datasets:
inline datasources` as a suggestion inside a `body:` block. Hardcoded
completion that's wrong for one domain poisons trust in all domains.
The right path is schema-driven: the case definition already has a
1421-line JSON Schema, and the dashboard DSL has Zod schemas for
parsing. Completion should derive from those, not from hand-typed lists.

The branch also moved `exportDiagram()` from pages-diagram-core to
graph-renderer — consolidating React Flow DOM coupling in the package
that owns it. The decision review caught the original plan to re-export
(would have created a circular dependency) and proposed the move instead.

Two gallery samples demonstrate the editor: a CaseHub dashboard YAML
with live preview (the bar chart and pie chart render as you type), and
a case definition YAML showing the Employee Onboarding sequential
pattern. Both highlight the gap that #408 will close — schema-driven
completion that knows which YAML it's looking at.
