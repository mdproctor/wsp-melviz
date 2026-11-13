## D1: Merge interaction state machine location

**Choice:** Extend GridEditController with `mergeActive` flag and merge-specific pointer logic
**Alternatives:**
- Separate MergeController — cleaner separation but pointer handoff complexity not worth it for ~100 lines of merge logic
- Overlay-driven — fights existing architecture by intercepting pointer events the controller owns
**Rationale:** Controller already owns all pointer interactions (drag, resize, selection). Adding merge as another interaction type keeps event ownership in one place with no handoff coordination.
**Trade-offs:** Controller grows larger (~1220 lines → ~1350), but the alternative is two-class coordination for a single boolean flag
**Sources:** grid-edit-controller.ts (1120 lines, EditorMode, pointer handlers), grid-overlay.ts (overlay rendering)
**Exploration:** quick
**Status:** captured

## D2: Merge overlay visual approach

**Choice:** CSS-only overlay — class toggles on grid container and items (`[data-merge-active]`, root highlight, absorption target classes)
**Alternatives:**
- DOM overlay panels — positioned divs over grid cells, more control but fragile during drag and requires measurement
- Canvas overlay — overkill for colored rectangles, fights DOM-based overlay approach
**Rationale:** CSS class toggles are lightest-weight and align with existing overlay patterns (selection outlines use DOM classes). Dim, highlight root, highlight targets all map to CSS selectors. No layout recalc, easy to animate.
**Trade-offs:** Less control over advanced visuals (merge arrows, cell labels) but those aren't needed for the interaction
**Sources:** grid-overlay.ts (DOM-based overlay functions), selection-overlay.ts (class-based toolbar)
**Exploration:** quick
**Status:** captured

## D3: Drag-to-absorb targeting model

**Choice:** Cardinal direction from root — drag direction determines axis, contiguous cells along that axis from root edge are candidates, pointer distance controls how many cells highlight
**Alternatives:**
- Pointer-under targeting — hit-test whichever cell pointer is over, more flexible but unpredictable and prone to accidental absorption
- Rectangular lasso — drag defines a rectangle, powerful but hard to control precisely on small grids
**Rationale:** Cardinal direction makes drags intentional and predictable. Combined with L-shape support (second drag in perpendicular direction), covers rectangular and L-shaped merges cleanly.
**Trade-offs:** Can't merge diagonally-adjacent cells in a single drag, but that's not a real grid operation anyway
**Sources:** grid-split-merge.ts (mergeItems computes bounding rect), grid-edit-controller.ts (pointer event handling)
**Exploration:** quick
**Status:** captured
