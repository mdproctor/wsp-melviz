---
layout: post
title: "Migrating to Zod v4 — the undocumented field mapping"
date: 2026-09-17
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [zod, migration, typescript, schema-introspection]
series: issue-451-zod-v4-migration
---

Zod v4 has been stable for fourteen months. Every package in the ecosystem has
moved. We were still on 3.x, and the longer we waited the more migration debt
we accumulated — particularly since the pages-lsp completion engine
introspects Zod internals to walk schema trees on every keystroke.

The migration guide says `._def` moved to `._zod.def` and that "the structure
of all internal defs is subject to change." What it doesn't say is which fields
changed. We had to read the v4 source to build the mapping.

Five fields were renamed: `typeName` became `def.type` with lowercase values
(`"ZodString"` to `"string"`), the array element accessor moved from `._def.type`
to `def.element`, enum values from `._def.values` to `def.entries`, and literal
values from a singular `._def.value` to `def.values` as an array. Two structures
changed shape entirely: `ZodDiscriminatedUnion` lost its `optionsMap` and now
shares `type: "union"` with regular unions, distinguished only by a
`discriminator` field. `ZodEnum` and `ZodNativeEnum` both became `type: "enum"`,
distinguished by whether `entries` is an array or an object.

A pure find-and-replace of `._def` to `._zod.def` would have compiled, run, and
produced silently wrong completions. The LSP would have returned empty results
where it used to return property lists, because `def.type` on an array schema
returns `"array"` (the type discriminator), not the array element schema that
`._def.type` used to return. That's the kind of failure that doesn't show up
in a smoke test.

The other surprise was a behavioural change in `.default().optional()`. In v3,
`.optional()` short-circuits: a missing field returns `undefined`, and the
`.default()` inside never fires. In v4, `.default()` fires first: a missing
field returns the default value. This broke our lookup parser's alias
resolution — `col.order ?? col.sortOrder` was picking up `"ASCENDING"` from
the default instead of `undefined`, so the alias `sortOrder: "DESCENDING"`
was silently ignored. The fix was to remove `.default()` from schemas used in
alias patterns and let the transform's `??` chain handle the fallback, which
it was already doing — the schema-level default was redundant in v3 and
actively harmful in v4.

The actual migration was mechanical once we had the field mapping. Three files
hold all the `._def` access: `schema-navigation.ts` (the completion engine),
`zod-to-fieldschema.ts` (converts Zod schemas to JSON Schema-like FieldSchema
for the property palette), and `hover.ts` (hover descriptions). Each has
duplicated helper functions — `typeName()`, `getShape()`, `unwrap()` — that
perform the same access patterns. Updating all three in lockstep was
straightforward with the mapping table as a reference.

The generator script that produces `component-schemas.generated.ts` from
TypeScript interfaces also needed updating: `z.record(val)` now requires two
arguments, and `.passthrough()` is deprecated in favour of `z.looseObject()`.
One `yarn generate` after fixing the templates and a hundred schema calls
updated themselves.

The `dist/` staleness issue cost an hour of debugging. Several packages in the
monorepo have `"main": "dist/index.js"` — vitest in downstream packages loads
the compiled output, not the source. After updating the TypeScript source but
before rebuilding, the integration test was loading stale v3-compiled schemas
from `dist/`. The `forEach` and `when` mixin fields were missing from the
component schema shape — not because the spread was broken in v4, but because
the test was running against old compiled code that didn't have the v4 changes.
Rebuilding `pages-schema` and `pages-data` dist directories fixed it
immediately.

This unblocks blocks-ui's discriminator manifest work, which needs v4's
automatic discriminator detection. More immediately, the 6.5x faster object
parsing and 57% smaller bundle should be measurable in the LSP's completion
latency — the completion engine parses schema objects on every keystroke.
