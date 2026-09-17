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
