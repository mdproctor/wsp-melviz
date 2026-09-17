# Decisions — Zod v4 Migration

## D1: Schema introspection migration strategy

**Choice:** Mechanical field-name migration from `._def` to `._zod.def` with adjusted field names — not a redesign around v4 public APIs.

**Alternatives:**
- Redesign around `.toJSONSchema()` — loses live Zod tree, slower, can't handle ZodLazy, would be navigating a serialized representation instead of the live schema graph.
- Use `z.registry` / `.meta()` — these are for registration and annotation, not structural introspection.

**Rationale:** `schema-navigation.ts` traverses the live schema tree to answer "what kind? what children?" — inherently internal access. None of v4's public APIs replace this. The mechanical swap is low-risk, testable, and matches what the code actually needs. However, it's not a pure `._def` → `._zod.def` rename — five fields changed names and two structures changed shape (see field mapping in spec).

**Trade-offs:** We remain coupled to Zod internals. A future Zod 5 could break these again. But this was already true in v3, and structural schema traversal is an accepted library-author pattern.

**Sources:**
- `packages/pages-lsp/src/schema-navigation.ts` — 19 `._def` accesses
- `packages/pages-document/src/zod-to-fieldschema.ts` — 10 `._def` accesses
- `packages/pages-lsp/src/hover.ts` — 2 `._def` accesses
- [Zod v4 library authors guide](https://zod.dev/library-authors)
- [Zod v4 core docs](https://zod.dev/packages/core)
- [Zod v4 migration guide](https://zod.dev/v4/changelog)
- [Zod v4 schemas.ts source](https://github.com/colinhacks/zod/blob/main/packages/zod/src/v4/classic/schemas.ts)

**Exploration:** deep-analysis
**Status:** captured

## D2: Discriminated union branch resolution without optionsMap

**Choice:** Linear scan of `options` array each time — no caching.

**Alternatives:**
- Build a WeakMap-cached `optionsMap` from `options` on first encounter — O(1) lookup but adds caching machinery.

**Rationale:** The component schema has ~50 variants. Linear scan of 50 items is sub-microsecond — well below the threshold where caching adds value. The v4 discriminated union shares `type: "union"` with regular unions, distinguished only by the presence of a `discriminator` field. The code will check for `def.discriminator` to branch into DU-specific logic, then iterate `def.options` to find the matching branch by reading each option's shape and comparing the discriminator field's literal value.

**Trade-offs:** Linear scan is O(n) per completion request. If the variant count grows significantly (hundreds), this could become measurable — but that's not a realistic scenario for component schemas, and we can add caching later if profiling shows a need.

**Depends on:** D1 (schema introspection strategy)
**Sources:**
- `packages/pages-lsp/src/schema-navigation.ts:166-183` — v3 DU resolution via optionsMap
- [Zod v4 schemas.ts source](https://github.com/colinhacks/zod/blob/main/packages/zod/src/v4/classic/schemas.ts) — DU def has `options` + `discriminator`, no `optionsMap`
**Exploration:** quick
**Status:** captured

## D3: Migrate all deprecated APIs, not just removed ones

**Choice:** Update all deprecated v3 patterns to their v4 replacements — no technical debt. Applies to both hand-written code and the generator script.

**Alternatives:**
- Minimal migration (fix only removed APIs) — leaves deprecated `.passthrough()`, `.merge()`, `.strict()` calls in place since they still work in v4. Less churn but accumulates debt.

**Rationale:** Pre-release project — bold changes welcome, no backward compat concerns. Leaving deprecated calls in generated output signals stale tooling. The changes are mechanical:
- `.passthrough()` → remove (v4 objects are passthrough by default) or use `z.looseObject()` where explicit passthrough is needed
- `.merge(other)` → `.extend(other.shape)` (5 calls, all in dead-code `component-schemas.ts` — but clean up anyway)
- `.strict()` → `z.strictObject()` (1 test file)
- `z.record(val)` → `z.record(z.string(), val)` (~100+ calls + generator template)
- Generator script: update both the `z.record()` template and the `fieldSchemaBlock` `.passthrough()` — regenerate output

**Trade-offs:** Slightly larger diff. But all changes are mechanical, independently testable, and leave zero deprecated API usage behind.

**Sources:**
- [Zod v4 migration guide](https://zod.dev/v4/changelog) — .passthrough(), .merge(), .strict() deprecated not removed
- `packages/pages-schema/scripts/generate-schemas.ts:71,157` — generator emits deprecated patterns
**Exploration:** quick
**Status:** captured

## D4: Manual migration, no codemod

**Choice:** All changes done manually — no community codemod.

**Alternatives:**
- Run `zod-v3-to-v4` codemod first for bulk `z.record()` fixes, then manual for the rest — saves typing but requires reviewing every codemod change.

**Rationale:** The mechanical changes are well-scoped and regex-friendly. We understand every file. The codemod doesn't handle the hard parts (`._def`, `optionsMap`, generator, deprecated API cleanup) and could introduce unexpected transforms. Manual gives full control with no surprises.

**Trade-offs:** More typing for ~100 `z.record()` calls. Trivial — the generator script handles ~50 of them automatically, and IDE find-replace covers the rest.

**Sources:**
- [zod-v3-to-v4 codemod](https://github.com/nicoespeon/zod-v3-to-v4) — community-maintained, unofficial
**Exploration:** quick
**Status:** captured
