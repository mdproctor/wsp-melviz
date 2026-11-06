# Decisions — #618 Grid Edit Placeholder Drag

## D1: Layout engine for drag interaction

**Choice:** Keep native CSS Grid — no absolute positioning for items
**Alternatives:**
- Gridstack.js (Option A from issue) — proven UX but uses absolute positioning internally, visual jump on enter/exit, fights existing CSS Grid layout
- Hybrid gridstack engine + native rendering (Option C from issue) — engine isn't cleanly separable from DOM layer
**Rationale:** The existing `{x, y, w, h}` placement model maps directly to `grid-column`/`grid-row` via `applyGridPlacement`. Absolute positioning would discard this model and require a translation layer in both directions. The Grid Spanning showcase proves the CSS Grid model handles all spanning combinations natively.
**Trade-offs:** Must implement compaction algorithm ourselves rather than using gridstack's battle-tested one — but the problem is simpler (integer grid coordinates, not arbitrary pixels)
**Sources:** `packages/pages-component/src/renderer/grid.ts` (applyGridPlacement), `examples/samples/Layout/Grid Spanning.page.yaml`, DashBuilder/Melviz prior art
**Exploration:** deep-analysis
**Status:** captured

## D2: Drag feedback mechanism

**Choice:** Placeholder is a structural CSS Grid item, not an overlay ghost
**Alternatives:**
- Absolutely-positioned ghost overlay (current implementation) — disconnected from grid flow, can't show how other items will reflow
**Rationale:** A placeholder that participates in CSS Grid flow (`grid-column`/`grid-row`) naturally displaces other items and shows the exact landing position. The current ghost overlay floats above the grid with pixel positioning, giving no feedback about final layout.
**Trade-offs:** Placeholder insertion/removal on every snap position change may cause layout thrash — mitigated by CSS transitions on grid items
**Sources:** `packages/pages-component/src/renderer/grid-overlay.ts:45-49` (current ghost), `packages/pages-component/src/renderer/grid-drag.ts:77-91` (cellRectToPixel — eliminated by this approach)
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Collision resolution strategy

**Choice:** Displace is always possible (push to next valid row for the item's w×h). Grid mode determines whether displace is automatic or offered in a popup alongside split/replace/swap.
**Alternatives:**
- Always-popup (every collision prompts) — too disruptive for dashboard-style grids where displacement is the expected behavior
- Always-displace (no popup ever) — destructive for composed grids where panel placement is intentional
**Rationale:** Two valid use cases require different defaults. Dashboard-style "pure grids" expect automatic displacement (like Grafana). Page-builder "composed grids" need user confirmation because each panel's position is intentional. Displace is always geometrically valid — even with spans, you can always find the next row that fits the item's w×h.
**Trade-offs:** Grid mode is a new concept that needs to be surfaced in the placement model
**Sources:** User's DashBuilder/Melviz experience, Grafana/Kibana dashboard patterns
**Exploration:** deep-analysis
**Depends on:** D2
**Status:** captured

## D4: Collision popup options (composed grid mode)

**Choice:** Popup offers: Displace, Split, Replace, Swap
**Alternatives:**
- Drop zones on panel edges (spatial encoding of intent) — better UX but higher visual complexity, deferred to future version
**Rationale:** Popup is simpler to implement and gets the logic right. Future versions can replace the popup with edge-based drop zones that encode intent spatially (drag to left edge = vertical split, drag to top edge = horizontal split).
**Trade-offs:** Popup interrupts the drag flow — acceptable for v1, drop zones are the v2 path
**Sources:** User direction
**Exploration:** quick
**Depends on:** D3
**Status:** captured

## D5: Split mechanics

**Choice:** Split subdivides the target cell to fit both items. Direction options constrained by target cell geometry — horizontal split (stack vertically, valid when h > 1 or grid can grow rows) and vertical split (side by side, valid when w > 1). A 1×1 cell cannot split.
**Alternatives:**
- Free-form split with arbitrary proportions — over-engineered for v1
**Rationale:** Integer grid coordinates mean splits produce clean results. A 4×2 can become two 4×1s (horizontal) or two 2×2s (vertical). Constraining to valid directions prevents impossible layouts.
**Trade-offs:** Only equal splits in v1 — unequal splits (e.g., 2/3 + 1/3) deferred
**Sources:** User direction
**Exploration:** quick
**Depends on:** D4
**Status:** captured

## D6: Drag handle

**Choice:** Only the top ~24px of a panel triggers drag (title bar area). Rest of panel is for content interaction.
**Alternatives:**
- Entire panel is drag target (current behavior) — causes accidental drags when interacting with content
**Rationale:** Issue #618 explicitly identifies "no drag handle" as a UX problem. Standard pattern across Grafana, Kibana, and the original DashBuilder.
**Trade-offs:** Panels without a visible title bar need a subtle drag indicator (⠿ or ≡ icon on hover)
**Sources:** Issue #618 requirements
**Exploration:** quick
**Status:** captured

## D7: Compaction algorithm

**Choice:** Vertical compaction (gravity upward): place dragged item at snap position, push overlapping items down, compact all items upward to fill gaps. All integer arithmetic on 12-column grid.
**Alternatives:**
- No compaction (items stay where dropped, gaps remain) — messy layouts accumulate
- Bidirectional compaction — over-complex, unpredictable
**Rationale:** Single-direction gravity is predictable — items only ever move up (compaction) or down (displaced by dragged item). Users can predict the outcome because there's only one rule: "everything falls to the top."
**Trade-offs:** Horizontal gaps within a row are not compacted — acceptable because column positions are intentional
**Sources:** Issue #618 design section, React-Grid-Layout compaction model
**Exploration:** quick
**Depends on:** D1
**Status:** captured
