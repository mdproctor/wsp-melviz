# Decisions — issue-420-lsp-schemas

## D1: Generator location

**Choice:** Generator lives in `lsp-schemas/scripts/` in blocks-ui
**Alternatives:**
- pages-schema in casehub-pages — centralises generation but requires cross-repo path mappings to reach blocks-ui domain types
- New pages-lsp subpath — makes pages-lsp domain-aware, which contradicts its role as a generic LSP framework
**Rationale:** The source types (graph-stencil-case, graph-stencil-htn, graph-stencil-swf) are all in blocks-ui. Keeping the generator near its sources avoids cross-repo type resolution issues. The existing hand-written schemas in lsp-schemas get replaced in-place.
**Trade-offs:** Work happens in blocks-ui repo rather than casehub-pages (where the issue is filed). The pages-schema component generator can serve as a template but won't share code with the domain generator.
**Sources:** lsp-schemas/package.json, pages-schema/scripts/generate-schemas.ts, GE-20260907-156200 (stale dist gotcha)
**Exploration:** quick
**Status:** captured

## D2: Index signature handling

**Choice:** Strip index signatures at the property walk
**Alternatives:**
- Emit .passthrough() — preserves unknown keys but weakens LSP diagnostics
- Configurable per-format — more flexible but premature; can be added later if needed
**Rationale:** CaseDefinition types have `[k: string]: unknown` on most sub-types. Stripping these produces tight Zod schemas that give accurate LSP completion and diagnostics. The `stripIndexSignatures` option mentioned in the issue.
**Trade-offs:** If a domain type genuinely needs open-ended keys, the generator won't emit them. This can be addressed with per-format config later.
**Sources:** graph-stencil-case/src/types/generated/case-definition.ts:59 (`[k: string]: unknown | undefined`)
**Exploration:** quick
**Status:** captured

## D3: Key-presence union recovery

**Choice:** JSON discriminator manifest files per format
**Alternatives:**
- Inline annotations in generator — simpler but rules are buried in code
- Skip union recovery — defers the problem, LSP suggests all keys instead of context-appropriate ones
**Rationale:** Declarative JSON manifests (`discriminators/case-definition.json`) are inspectable, maintainable, and separate concern (which properties form exclusive unions) from mechanism (how to generate Zod). The LSP's existing `VariantDispatch` type already has a `key-presence` strategy.
**Trade-offs:** Requires maintaining a manifest file per format alongside the TypeScript types. Manifests can go stale if upstream types change union semantics.
**Sources:** pages-lsp/src/types.ts:9-13 (VariantDispatch), issue #420 body ("discriminator manifests")
**Exploration:** quick
**Status:** captured

## D4: SWF source treatment

**Choice:** Same ts-morph pipeline for all four formats
**Alternatives:**
- Keep SWF hand-written — pragmatic but inconsistent staleness tracking
- Separate SWF generator — clean separation but unnecessary infrastructure for a simple type
**Rationale:** ts-morph reads .d.ts from npm packages just as well as source .ts files. One pipeline produces consistent output and one staleness test pattern for all formats.
**Trade-offs:** SWF types from `@openworkflowspec/sdk` may have different type idioms than workspace packages. The generator must handle both source and declaration file inputs.
**Sources:** graph-stencil-swf/src/types.ts, @openworkflowspec/sdk
**Exploration:** quick
**Status:** captured

## D5: Staleness test approach

**Choice:** Vitest in-memory diff
**Alternatives:**
- Script-level --check flag — more explicit but requires dual-mode generator
**Rationale:** Same pattern as the component generator in pages-schema. A vitest test runs the generator in dry-run mode, compares output strings against committed .generated.ts files. Fast, no filesystem side effects, familiar pattern.
**Trade-offs:** Test needs access to the same ts-morph project the generator uses, duplicating some setup.
**Sources:** pages-schema/src/generator.test.ts
**Exploration:** quick
**Status:** captured

## D6: Output file layout

**Choice:** One .generated.ts file per domain format
**Alternatives:**
- Single combined file — matches component generator but bigger diffs
**Rationale:** Each generated file replaces its hand-written counterpart (e.g. `schemas/case-definition.ts` → `schemas/case-definition.generated.ts`). Parallels the existing file structure, enables per-format staleness, and keeps diffs reviewable.
**Trade-offs:** Generator must handle multiple output files. Slightly more complex script than a single-output generator.
**Sources:** lsp-schemas/src/schemas/ (existing per-format file structure)
**Exploration:** quick
**Status:** captured
