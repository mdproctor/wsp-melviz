---
layout: post
title: "The migration that found three bugs it wasn't looking for"
date: 2026-09-06
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [pages-viz, pages-property-palette, lit, web-components, refactoring]
series: issue-375-migrate-schemaform-palette
---

# The migration that found three bugs it wasn't looking for

PagesSchemaForm had its own rendering loop — imperative DOM management,
element caching in a Map, a `mapFieldToComponentType` resolver that
duplicated most of what the palette's `resolveEditor` already does. Issue
#375 said: embed `<pages-property-palette>` internally, keep the public
API unchanged, eliminate the duplication.

The design was straightforward. Build a `PropertyPaletteSource` from the
TypedDataSet, provide a custom `EditorResolver` for the composite types
(object-group, array-group, variant-group), maintain a data mirror for
`validate()` and `submit()`. The palette renders; PagesSchemaForm is the
controller.

Then the decision review caught three things I'd missed.

The first was a factual error in my own rationale. I'd written that the
composite types "need TypedDataSet, DataSetLookup, cell extraction" — 
justifying why they couldn't move to the palette. Claude read the
imports. PagesObjectGroup extends `FormValueMixin(LitElement)`. No
TypedDataSet. No pipeline dependency at all. The composites are already
palette-compatible. I'd assumed coupling that doesn't exist. The custom
resolver approach still made sense for scope — but for a different reason
than I'd written.

The second was a rendering bug in the palette itself. `_renderTagEditor`
calls `document.createElement(tag)` on every render cycle. No element
reuse. In practice: text inputs lose focus mid-typing, dropdowns close,
partially entered values vanish on any re-render. This had to be fixed
before migration could work. We added an element cache keyed by field
path, active-key tracking for stale pruning, and two public accessors
(`getFieldElement`, `setFieldErrors`) that PagesSchemaForm needs for
validation and `fieldsOnly` mode.

The third surfaced during implementation: the schema enrichment logic
was injecting `enum` arrays from dataset distinct values into every
string field without an explicit enum. Intended to populate select
dropdowns. Actual effect: every text input became a select. The old code
avoided this because `extractDistinctValues` only ran for fields already
mapped as selects. The new code ran it during schema preparation, before
the resolver ever saw the field. The fix was to remove the injection
entirely — the auto-derive path (`deriveSchemaFromDataSet`) already
handles LABEL columns correctly.

A fourth issue appeared in the palette's `_renderField`: it intercepts
`type: "object"` schemas and routes them to its own nested-object
renderer, even when a custom resolver has already returned a descriptor
for that schema. The composites never rendered because the palette
overrode the resolver's decision. A `!customResult` guard fixed it.

The migration landed as three commits: palette caching, PagesSchemaForm
rewrite, fieldsOnly wiring. All 620 tests pass. The public API is
unchanged — consumers don't know the palette exists.

What this opens up: the composites are pipeline-agnostic. Moving them
into the palette itself is now a natural follow-up — every palette
consumer would get native array add/remove/reorder and oneOf variant
support without needing a custom resolver. That's not this branch, but
this branch proved it's feasible.
