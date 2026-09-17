# Design: Migrate to Zod v4 Across Pages Packages

**Issue:** casehubio/casehub-pages#451
**Date:** 2026-09-17
**Branch:** issue-451-zod-v4-migration

## Overview

Migrate all 8 pages packages from Zod 3.x to Zod v4. The critical path is
updating 3 files that access Zod internals (`._def`) for schema introspection —
the LSP completion engine, hover provider, and Zod-to-FieldSchema converter.
The remaining work is mechanical API updates across ~30 source files.

## Scope

Phase 1 only — pages packages. Blocks-ui and downstream are separate issues
filed after this lands. This branch unblocks blocks-ui#158 (discriminator
manifests).

## Migration Strategy

### Internal API: `._def` → `._zod.def` with field name adjustments (D1)

The 3 files that access Zod internals for schema introspection will be updated
with the v4 field name mapping. This is not a pure path swap — 5 fields were
renamed and 2 structures changed shape.

**Field mapping:**

| v3 Access | v4 Access | Change Type |
|---|---|---|
| `._def.typeName` = `"ZodString"` | `._zod.def.type` = `"string"` | Renamed + new values |
| `._def.shape` (function or object) | `._zod.def.shape` (plain object) | Shape accessor simplified |
| `._def.innerType` | `._zod.def.innerType` | Unchanged |
| `._def.getter` (ZodLazy) | `._zod.def.getter` | Unchanged |
| `._def.type` (array element) | `._zod.def.element` | **Renamed** |
| `._def.values` (enum string[]) | `._zod.def.entries` | **Renamed** |
| `._def.value` (literal, singular) | `._zod.def.values` (array) | **Renamed + now array** |
| `._def.values` (native enum Record) | `._zod.def.entries` | **Renamed** |
| `._def.options` (union branches) | `._zod.def.options` | Unchanged |
| `._def.left` / `._def.right` | `._zod.def.left` / `._zod.def.right` | Unchanged |
| `._def.valueType` (record) | `._zod.def.valueType` | Unchanged |
| `._def.discriminator` | `._zod.def.discriminator` | Unchanged |
| `._def.optionsMap` (Map) | **Removed** — iterate `options` | **Structural change** |
| `._def.checks` (number constraints) | `._zod.def.checks` | Unchanged |

**Type name value mapping** (used in `typeName()` → switch/if-else chains):

| v3 `typeName` | v4 `def.type` |
|---|---|
| `"ZodString"` | `"string"` |
| `"ZodNumber"` | `"number"` |
| `"ZodBoolean"` | `"boolean"` |
| `"ZodObject"` | `"object"` |
| `"ZodArray"` | `"array"` |
| `"ZodEnum"` | `"enum"` |
| `"ZodNativeEnum"` | `"enum"` (same as ZodEnum — distinguish by `entries` format) |
| `"ZodLiteral"` | `"literal"` |
| `"ZodUnion"` | `"union"` |
| `"ZodDiscriminatedUnion"` | `"union"` (+ `discriminator` field present) |
| `"ZodIntersection"` | `"intersection"` |
| `"ZodRecord"` | `"record"` |
| `"ZodOptional"` | `"optional"` |
| `"ZodDefault"` | `"default"` |
| `"ZodNullable"` | `"nullable"` |
| `"ZodLazy"` | `"lazy"` |
| `"ZodAny"` | `"any"` |
| `"ZodUnknown"` | `"unknown"` |

### Discriminated union resolution (D2)

In v4, discriminated unions share `type: "union"` with regular unions —
distinguished by the presence of a `discriminator` field. The `optionsMap`
(Map from discriminator value → schema) is gone.

The navigation code will:
1. Check `def.discriminator` to detect a discriminated union (instead of matching
   on a separate type name)
2. Iterate `def.options` linearly to find the matching branch (instead of
   `optionsMap.get()`)

Linear scan of ~50 component variants is sub-microsecond — no caching needed.

### Deprecated API cleanup (D3)

All deprecated v3 patterns will be updated to v4 equivalents — no technical debt
left behind. Pre-release project, no backward compat concerns.

| v3 Pattern | v4 Replacement | Locations |
|---|---|---|
| `.passthrough()` | `z.looseObject()` (v4 default is still strip) | Generated schemas, generator script |
| `.merge(other)` | `.extend(other.shape)` | 5 calls in `component-schemas.ts` (dead code, clean up) |
| `.strict()` | `z.strictObject()` | 1 test file |
| `z.record(val)` | `z.record(z.string(), val)` | ~100+ calls across all packages |

### No codemod (D4)

All changes done manually. The codemod doesn't handle the hard parts and could
introduce unexpected transforms.

## Affected Files

### Critical — internal API rewrite (3 files, 31 `._def` accesses)

| File | Accesses | What it does |
|---|---|---|
| `pages-lsp/src/schema-navigation.ts` | 19 | LSP completion — navigates schema tree, produces completion entries |
| `pages-document/src/zod-to-fieldschema.ts` | 10 | Converts Zod schemas to JSON-Schema-like FieldSchema |
| `pages-lsp/src/hover.ts` | 2 | Hover descriptions from schema metadata |

Each file has duplicated helper functions (`typeName()`, `getShape()`, `unwrap()`)
that perform the same `._def` access patterns. All must be updated in lockstep.

### Generator script (1 file)

`pages-schema/scripts/generate-schemas.ts` — emits `z.record(valueType)` and
`.passthrough()`. Update the templates, then regenerate
`component-schemas.generated.ts`.

### Mechanical — `z.record()` two-arg (across all packages)

Every `z.record(x)` call becomes `z.record(z.string(), x)`. Affects:
- `pages-schema/src/document-schema.ts`
- `pages-schema/src/base-schemas.ts`
- `pages-schema/src/component-schemas.ts`
- `pages-data/src/dataset/external/schema.ts`
- `pages-ui/src/parser/page-schema.ts`
- `yaml-core/src/schema.ts`
- Multiple test files

### Package version bumps (8 packages)

All `package.json` files updated from `"zod": "^3.x"` to `"zod": "^4.0.0"`
(or whatever the latest stable v4 is). Single `yarn install` resolves all.

### Type annotations

`z.ZodType` is still available in v4 (simplified generics: `<Output, Input>`
instead of `<Output, Def, Input>`). No bulk rename needed.
`z.ZodTypeAny` → `z.ZodType` if encountered (none found in production code).

## Migration Order

1. **Bump Zod version** — update all 8 `package.json` files, `yarn install`
2. **Update generator script** — fix `z.record()` template and `.passthrough()`
   in `fieldSchemaBlock`, regenerate `component-schemas.generated.ts`
3. **Update schema introspection files** — `schema-navigation.ts`,
   `zod-to-fieldschema.ts`, `hover.ts` using the field mapping table
4. **Fix `z.record()` calls** — all remaining hand-written source files
5. **Clean up deprecated APIs** — `.merge()`, `.strict()`, `.passthrough()`
   in hand-written code
6. **Update test files** — same `z.record()` and type name changes in tests
7. **Verify** — `yarn typecheck && yarn test`

Steps 2-6 can be parallelized since they touch different files. Step 1 must
come first. Step 7 must come last.

## Verification

- `yarn typecheck` — confirms all type annotations are v4-compatible
- `yarn test` — runs all package test suites including:
  - `schema-navigation.test.ts` — completions, DU narrowing, path traversal
  - `zod-to-fieldschema.test.ts` — schema → FieldSchema conversion
  - `hover.test.ts` — hover descriptions
  - `completion.test.ts` — end-to-end completion integration
  - `schema-completion.test.ts` — CodeMirror completion integration
  - `generator.test.ts` — generated schema validation
  - `lookup-parser.test.ts` — data pipeline schema parsing

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| `._zod.def` field names differ from source analysis | Low — verified from v4 source | Console.log sanity check at start of implementation |
| v4 `shape` is lazy (function) not plain object | Low — v4 simplified this | `getShape()` already handles both; test will catch |
| v4 discriminated union has additional internal state beyond `discriminator` + `options` | Low | Tests cover DU narrowing; complete test suite runs |
| `z.ZodType` compat alias missing `.description` at type level | Medium | Check at typecheck step; if broken, use `(schema as any).description` temporarily and file upstream |
| ZodEnum and ZodNativeEnum both map to `type: "enum"` — `entries` format may differ | Low | Console.log both formats at implementation start; adjust accessor accordingly |

## References

- `packages/pages-lsp/src/schema-navigation.ts` — primary introspection file
- `packages/pages-document/src/zod-to-fieldschema.ts` — FieldSchema converter
- `packages/pages-lsp/src/hover.ts` — hover provider
- `packages/pages-schema/scripts/generate-schemas.ts` — schema code generator
- `packages/pages-schema/src/document-schema.ts` — discriminated union usage
- [Zod v4 migration guide](https://zod.dev/v4/changelog)
- [Zod v4 library authors guide](https://zod.dev/library-authors)
- [Zod v4 core docs](https://zod.dev/packages/core)
- [Zod v4 schemas.ts source](https://github.com/colinhacks/zod/blob/main/packages/zod/src/v4/classic/schemas.ts)
- casehubio/casehub-pages#451
