---
layout: post
title: "Recursive Forms"
date: 2026-09-04
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [schema-form, nested-objects, arrays, oneOf, recursive-rendering, FormValueProvider, mixin]
---

# Recursive Forms

JSON Schema is recursive. The form renderer should be too.

That's the design principle behind #222 — nested object and array schema
support for `PagesSchemaForm`. The previous implementation handled flat
schemas only: each property mapped to one leaf input. Nested `properties`,
`items`, and `oneOf` were in the `FieldSchema` type but ignored by the
renderer.

## The Architecture

Every JSON Schema node maps to exactly one component type. Three new Lit
components handle the structural cases:

- **`pages-object-group`** — a fieldset that renders sub-properties
  recursively. Objects within objects just nest deeper.
- **`pages-array-group`** — a list with add/remove/reorder controls.
  Uses synthetic monotonic keys for DOM reconciliation via Lit's
  `repeat()` directive.
- **`pages-variant-group`** — a discriminated union selector. Auto-detects
  the discriminator from `const` values across `oneOf` variants.

The recursion terminates at leaf inputs — the same `pages-input`,
`pages-select`, `pages-checkbox` that flat forms use. No new leaf types.

## FormValueProvider Protocol

The unifying contract: every form component — leaf and composite —
implements `currentValue`, `value`, `error`, and `validate()`. A shared
`FormValueMixin(LitElement)` provides the template method structure with
four abstract hooks: `collectValue()`, `propagateValue()`,
`validateSelf()`, `validateChildren()`.

Value collection is recursive: `currentValue` on a composite calls
`currentValue` on each child. The result is a structured JSON record
matching the schema shape — `{ address: { street: "...", city: "..." },
tags: ["a", "b"] }`.

## The Dirty Flag

One gotcha surfaced during code review: distributing parent values to
children during Lit's `render()` overwrites user edits when the component
re-renders for unrelated reasons (toggling `editable`, collapsing a
section). The fix is a `_valuePending` dirty flag — set by the value
setter, cleared after distribution. Only propagate when value actually
changed.

## What Landed

14 design decisions. 3 rounds of decision review (7 revisions, 3 new
decisions surfaced). 2 rounds of spec review. 8 implementation tasks
across 4 batches. 21 files changed, ~1900 lines added. All tests green.
