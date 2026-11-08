# Decisions — #625 Unified Grid Layout Model

Decisions D1-D13 were captured during the #618 POC and carry forward
into this epic. D14-D19 are new decisions for the production implementation.

## Inherited from #618 POC

### D1: Layout engine for drag interaction
**Choice:** Keep native CSS Grid — no absolute positioning for items
**Exploration:** deep-analysis
**Status:** inherited

### D2: Drag feedback mechanism
**Choice:** Placeholder is a structural CSS Grid item, not an overlay ghost
**Exploration:** quick
**Depends on:** D1
**Status:** inherited

### D3: Collision resolution strategy
**Choice:** Grid mode determines whether displace is automatic or offered alongside split/replace/swap
**Exploration:** deep-analysis
**Depends on:** D2
**Status:** inherited

### D4: Collision popup options (composed grid mode)
**Choice:** Popup offers: Displace, Split, Replace, Swap
**Exploration:** quick
**Depends on:** D3
**Status:** inherited

### D5: Split mechanics
**Choice:** Equal splits only. Direction constrained by cell geometry.
**Exploration:** quick
**Depends on:** D4
**Status:** inherited

### D6: Drag handle
**Choice:** Only top ~24px (title bar area) triggers drag
**Exploration:** quick
**Status:** inherited

### D7: Compaction algorithm
**Choice:** Vertical compaction with upward gravity (place/push/compact)
**Exploration:** quick
**Depends on:** D1
**Status:** inherited

### D8: Grid mode declaration
**Choice:** Explicit `collision` YAML property — `displace` or `prompt`
**Exploration:** quick
**Depends on:** D3
**Status:** inherited

### D9: Unified layout model
**Choice:** Unify on `items`, remove `rows`/`columns`. Rewrite existing YAML files with migration script; parser drops rows/columns support entirely.
**Exploration:** deep-analysis
**Status:** inherited — clarified: clean break with file rewrite, no parser sugar

### D10: Two-mode editor — layout mode and content mode
**Choice:** Layout mode = structural ops (split/merge/resize/add-remove rows) + swap drag. Content mode = spatial ops (displace/push/replace drag). Toolbar toggle switches modes.
**Exploration:** quick
**Depends on:** D9
**Status:** inherited — clarified: layout mode includes drag-to-swap, not drag-free

### D11: Empty containers as first-class items
**Choice:** Item with no `component` key is a valid empty container
**Exploration:** quick
**Depends on:** D10
**Status:** inherited

### D12: Layout mode interactions
**Choice:** Split, merge, resize edges, add/remove rows, drag-to-swap. Output is a grid of containers (empty or populated).
**Exploration:** quick
**Depends on:** D10, D11
**Status:** inherited — clarified: drag-to-swap included

### D13: Content mode — drag into containers with replace
**Choice:** Drag onto empty container fills it. Occupied containers use dual-handle UX (displace/swap/replace).
**Exploration:** quick
**Depends on:** D4, D11
**Status:** inherited

## New decisions for #625

### D14: Grid model type redesign

**Choice:** Optional component — `component` becomes optional on `GridItem`. Absent = empty container. Nested grid = component of type `grid` with its own `items` array.
**Alternatives:**
- Union type (`EmptyContainer | ComponentItem | NestedGrid`) — proliferates types, requires match/switch everywhere
- Content slot (`content: Component | GridLayout | null`) — introduces new vocabulary alongside existing `component`
**Rationale:** Smallest change to the type system. `Component.items` field already exists for nesting. One vocabulary, three states expressed by presence/absence of `component` and whether it has `items`.
**Trade-offs:** Code must handle `item.component` being undefined — every access site needs a guard. But this is a natural consequence of empty containers being first-class.
**Sources:** types.ts:24 (`GridItem`), types.ts:1 (`Component.items`), D11 (empty containers)
**Exploration:** quick
**Depends on:** D11
**Status:** captured

### D15: Editor architecture — single controller with mode flag

**Choice:** One `GridEditController` class with `mode: 'layout' | 'content'` property. Pointer handlers branch on mode for collision semantics (swap vs displace). Structural operations (split/merge) enabled only in layout mode.
**Alternatives:**
- Two controller classes (`LayoutModeController` / `ContentModeController`) — clean separation but requires shared context extraction and teardown/setup on mode switch
- Strategy pattern (controller delegates to `ModeStrategy` interface) — extension point for future modes, but indirection overhead for just two strategies
**Rationale:** Two modes share ~80% of code (pointer tracking, snap-to-cell, geometry, overlay, undo). The difference is collision resolution on drop and whether structural ops are enabled. A mode flag is the right abstraction level.
**Trade-offs:** Mode branching inside handlers can grow messy if modes diverge further — but current scope is well-bounded.
**Sources:** grid-edit-controller.ts (current controller), POC grid-drag-poc.html
**Exploration:** quick
**Depends on:** D10
**Status:** captured

### D16: Compaction with recursive nesting

**Choice:** Independent compaction per grid level. Each grid (root or nested) runs its own compaction. Parent treats nested grid item as a fixed-size box. Optional auto-shrink of container height to match compacted content.
**Alternatives:**
- Coordinated upward propagation — nested compaction triggers parent re-layout. More responsive but couples compaction across levels.
- Manual resize only — nested containers never auto-resize. Simplest but worst UX.
**Rationale:** Independent compaction keeps each level self-contained. The compaction algorithm doesn't change — it's the same place/push/compact on each level's items. Auto-shrink is a small post-compaction step.
**Trade-offs:** A nested grid that compacts to fewer rows won't automatically close the gap in the parent unless auto-shrink is implemented. Acceptable — auto-shrink is straightforward.
**Sources:** grid-collision.ts (current compaction), D7 (compaction algorithm), D14 (nesting model)
**Exploration:** quick
**Depends on:** D7, D14
**Status:** captured

### D17: YAML serialization for nested grids

**Choice:** Nested grid = component of type `grid` with `items` array. No new YAML keys. Parser handles `items` recursively.
**Alternatives:**
- Inline `grid` property on item (peer to `component`) — adds new YAML key
- Separate named grid definitions at top level — adds reference/lookup layer
**Rationale:** `Component.items` already exists in the type system. A grid is just a component with children. No new concepts in YAML or the parser.
**Trade-offs:** Deep nesting produces deeply indented YAML. Acceptable — recursive nesting beyond 2-3 levels is uncommon, and the editor hides YAML complexity.
**Sources:** types.ts:12 (`Component.items`), page-parser.ts:281-310 (items parsing)
**Exploration:** quick
**Depends on:** D9, D14
**Status:** captured

### D18: Selection model for layout mode

**Choice:** Click-select + Shift-extend. Click selects one cell. Shift+click extends to rectangular region (bounding rect). Drag across empty cells creates spanning container (#628). Merge operates on selection; split on single selected cell.
**Alternatives:**
- Lasso/marquee selection — visual but conflicts with swap-drag gesture
- Explicit select/drag mode toggle — avoids conflict but adds another mode layer
**Rationale:** Standard spreadsheet/table pattern. No gesture conflict: drag handle initiates swap-drag, Shift+click initiates selection. Empty cells have no drag handle, so drag across them is unambiguous.
**Trade-offs:** Users must know Shift+click for multi-select — but this is universally understood.
**Sources:** Excel/Google Sheets merge workflow, Figma multi-select
**Exploration:** quick
**Depends on:** D12
**Status:** captured

### D19: Undo/redo via PageDocument

**Choice:** All grid operations go through `_applyEdit('grid', ...)`. No grid-local undo stack. Undo/redo is document-level via PageDocument.
**Alternatives:**
- Grid-local undo stack with document sync — faster visual undo during drag but two sources of truth, violates protocol
**Rationale:** The `all-mutations-through-applyedit` protocol (PP-20260916-b6f3e8) is critical. The POC needed its own undo because it has no PageDocument. Production code uses the document's existing mechanism.
**Trade-offs:** None — this is the only correct approach per protocol.
**Sources:** all-mutations-through-applyedit.md (protocol), builder-shell.ts (`_applyEdit`), page-document.ts (`_pushUndoInternal`)
**Exploration:** quick
**Depends on:** D10
**Status:** captured

### Additional clarifications (not decisions — no alternatives considered)

- **Migration strategy:** Rewrite all .page.yaml files with a migration script. Parser drops rows/columns support. Clean break.
- **Merge shape:** Rectangular only (CSS Grid constraint — grid-column/grid-row can only express rectangular spans).
- **Split ratio:** Equal halves only. Resize edges afterward for unequal proportions.
- **YAML output for auto-flow:** Items with no `placement` key default to auto-flow (w:1, h:1). Positioned items get explicit `{x, y, w, h}`.
- **Nested column count:** Independent per sub-grid (default 12). Parent geometry doesn't constrain nested grids.
- **Existing infrastructure reuse:** Palettes, catalog, SPI registry, APIs all reused as-is. Epic scope is grid model and editor only.
- **Full rewrite:** All grid-* files and types rewritten from scratch. POC and existing code serve as behavioural reference, not porting source.
