## D1: Controller Architecture

**Choice:** Controller Tree with Coordinator
**Alternatives:**
- Single Unified Controller — one controller manages all grid levels; requires full rewrite of existing 877-line controller, high risk
- Event Delegation Only — independent controllers communicate via DOM events; cross-grid state is split with no shared state, hard to keep consistent at arbitrary depth
**Rationale:** Preserves the existing well-tested single-grid controller. Adds cross-grid awareness as a separate concern (coordinator). Naturally supports arbitrary depth via tree traversal. Clean bridge routing via `gridPath` property on events.
**Trade-offs:** Additional module (coordinator) to maintain. Controllers need lifecycle hooks for registration/deregistration.
**Sources:** grid-edit-controller.ts (existing controller architecture), activation.ts:948-951 (attachment callsite), grid-edit-bridge.ts (bridge routing)
**Exploration:** quick
**Status:** captured

## D2: Controller Registration

**Choice:** DOM-based discovery
**Alternatives:**
- Path-based registration — YAML path passed to attachGridEditController, path prefix matching; requires threading path through render pipeline into activation.ts
- Data attribute markup — grid elements get data-grid-path attribute during rendering; same render pipeline change, string-encoded
**Rationale:** Zero changes to the render pipeline. DOM structure is the correct representation of visual nesting. Path for bridge routing derived from coordinator tree + document structure.
**Trade-offs:** DOM walks slightly more expensive than path lookup, but only on controller add/remove (not during drag).
**Sources:** activation.ts:948-951 (current callsite has Component but not path), render.ts (rendering pipeline)
**Exploration:** quick
**Status:** captured

## D3: Bridge Routing

**Choice:** Grid path on the event
**Alternatives:**
- PageNode target on event — event carries PageNode reference; creates cross-package dependency (component → document), wrong dependency direction
- Bridge walks DOM itself — bridge receives grid element and resolves path; duplicates coordinator responsibility
**Rationale:** Coordinator resolves DOM → YAML path once. Event carries `gridPath: (string|number)[]`. Bridge navigates document directly via existing path-based access. Clean separation of concerns.
**Trade-offs:** Coordinator needs a reliable DOM → YAML path mapping. If rendering changes how elements map to YAML nodes, the mapping must be updated.
**Depends on:** D1 (coordinator owns the path resolution), D2 (DOM-based discovery provides the element-to-path mapping)
**Sources:** grid-edit-bridge.ts:43-51 (current root-only routing), page-document.ts:372-381 (path-based getItems), builder-shell.ts:205-209 (event handler)
**Exploration:** quick
**Status:** captured

## D4: Cross-Grid Drag Handoff

**Choice:** Coordinator intercepts pointer, delegates to controllers
**Alternatives:**
- Drag overlay — lifts item into fixed-position overlay; requires visual clone, loses FLIP animation during transit
- Source controller manages — source reaches into target controller internals; violates encapsulation
**Rationale:** Coordinator already knows the tree and resolves deepest grid. Individual controllers gain minimal new API (releaseItem/acceptForeignItem). Cross-grid state stays coherent in one place. No visual clone needed — source controller hides item, target controller shows placeholder.
**Trade-offs:** Document-level pointermove listener during drag. Controllers need new release/accept API surface (thin methods).
**Depends on:** D1 (coordinator exists), D2 (tree is built)
**Sources:** grid-edit-controller.ts:285-341 (handlePointerDown), grid-edit-controller.ts:343-441 (handlePointerMove), grid-drag.ts (geometry/snapping)
**Exploration:** quick
**Status:** captured

## D5: Empty Nested Grid Cleanup

**Choice:** Keep as empty container
**Alternatives:**
- Auto-remove — remove empty grid container automatically; could surprise users who intended to drag items back
- Collapse to placeholder — replace grid component with empty placeholder item; preserves space without grid semantics
**Rationale:** Consistent with existing behavior — empty grid items stay until explicitly removed. Users can drag items back or delete manually.
**Trade-offs:** Empty grid containers persist in YAML until manually removed. Minor clutter.
**Sources:** grid-edit-bridge.ts:184-188 (_removeItem), page-document.ts:460 (removeItem)
**Exploration:** quick
**Status:** captured
