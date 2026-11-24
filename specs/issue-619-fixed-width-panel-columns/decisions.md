## D1: Auto-sizing computation location

**Choice:** Render-time dynamic computation — measure the grid container's width before placing items, compute `effectiveW` from `fixedWidth` and `cellWidth`.
**Alternatives:**
- Editor-computed, stored in YAML — compute optimal `w` in the editor and persist it. No runtime overhead but stale if container width differs from editor.
**Rationale:** Correct separation of concerns: YAML declares intent (`fixedWidth: 160`), renderer computes layout. Responsive — adapts to any container width without editor dependency.
**Trade-offs:** Synchronous DOM measure in the render path; potential forced reflow if container hasn't been laid out. Mitigated by measuring before adding children.
**Sources:** `render.ts:116-165` (render loop), `grid-drag.ts:21-36` (computeGridGeometry), issue #619
**Exploration:** quick
**Status:** captured

## D2: fixedWidth type location

**Choice:** Add `fixedWidth?: number` to `GridPlacement` type, alongside `x`, `y`, `w`, `h`.
**Alternatives:**
- On `GridItem` — separates sizing from placement, but `fixedWidth` directly constrains column allocation (`w`), so co-locating them is more natural.
**Rationale:** `fixedWidth` is a constraint on `w` (column span). Keeping them in the same type makes the relationship explicit and simplifies the render-time computation.
**Trade-offs:** `GridPlacement` grows slightly; both `positioned` and `auto` variants gain the optional field.
**Sources:** `types.ts:20-27` (GridPlacement/GridItem types)
**Exploration:** quick
**Status:** captured

## D3: effectiveW clamping

**Choice:** `effectiveW = Math.min(Math.ceil(fixedWidth / cellWidth), w)` — clamp to the declared `w` as upper bound.
**Alternatives:**
- No clamping — `effectiveW = Math.ceil(fixedWidth / cellWidth)` regardless of `w`. Risky: on very narrow screens, `effectiveW` could exceed `w`, meaning the panel grows beyond its YAML-declared span.
**Rationale:** The declared `w` is the author's maximum intent. On narrow screens where `fixedWidth` requires all `w` columns, the panel behaves like a normal panel (no `justify-self`, full span). On wider screens, it shrinks. This is safe and predictable.
**Trade-offs:** None significant — clamping only prevents unexpected growth.
**Depends on:** D1 (render-time computation)
**Sources:** Issue #619 constraint: "column allocation must be integer"
**Exploration:** quick
**Status:** captured
