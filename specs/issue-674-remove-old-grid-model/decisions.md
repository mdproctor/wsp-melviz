## D1: Clean break — remove all old model code

**Choice:** Remove all rows/columns support from the TS runtime in one pass. No backward compatibility, no deprecation period.
**Alternatives:**
- Deprecation-then-removal — adds complexity for zero consumers (pre-release repo)
- Runtime migration (auto-convert at parse time) — keeps dead code paths, masks undetected old-format files
**Rationale:** Pre-release repo with no external consumers. D9 from #625 already decided this: "Unify on items, remove rows/columns. Clean break."
**Trade-offs:** Any unmigrated YAML renders as empty grid. Acceptable because the parser already doesn't handle `rows:`.
**Sources:** docs/specs/issue-625-unified-grid-layout-model/decisions.md D9
**Exploration:** quick
**Status:** captured

## D2: Remove layout components (type: columns, type: rows) alongside page-level model

**Choice:** Remove `type: "columns"` and `type: "rows"` layout components, their type guards, CSS handlers, and schema entries. Grid items replace all layout needs.
**Alternatives:**
- Keep layout components as convenience wrappers — adds two code paths for the same thing
**Rationale:** Pre-release, no technical debt. The items model with `at: {x, y, w, h}` handles all layouts the components did. Keeping them is dead weight.
**Trade-offs:** Any YAML using `type: columns` or `type: rows` in component slots needs migration to grid items.
**Sources:** User directive ("pre-release, no technical debt. migrate everything")
**Exploration:** quick
**Status:** captured

## D3: Ordering — YAML first, then TS bottom-up

**Choice:** Migrate all 7 YAML source files first (using existing migration script), then remove TS code bottom-up: pages-document types → pages-schema validators → pages-ui parser/desugarer → pages-component (migration script, layout types) → pages-builder → tests.
**Alternatives:**
- TS first, let compiler errors guide — risks runtime breakage if YAML files reference removed types during intermediate states
- All at once — too large to review, harder to bisect if something breaks
**Rationale:** Once old-format YAML is gone, TS removal is safe — nothing references the dead code paths. Bottom-up means consumers see compile errors immediately when their imports disappear.
**Trade-offs:** Slightly more commits, but each is independently correct.
**Sources:** packages/pages-component/src/renderer/grid-migration.ts (existing migration script)
**Exploration:** quick
**Status:** captured
