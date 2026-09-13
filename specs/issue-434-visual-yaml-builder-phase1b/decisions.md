# Visual YAML Builder Phase 1b — Design Decisions

## D1: Spec Scope

**Choice:** One cohesive spec covering all three child issues (#433 structural editing, #432 preview data, #429 dock-workbench Lit)
**Alternatives:**
- #433 only — lower scope risk but misses integration points between structural editing, preview, and the dock that hosts them
**Rationale:** The three issues interact. Preview data depends on structural editing being solid (adding a component should immediately render with sample data). The dock refactor reshapes the shell that hosts all interactions. Designing together ensures coherence.
**Trade-offs:** Larger spec, but phased implementation (#433 → #432 → #429) keeps delivery incremental
**Exploration:** quick

## D2: Add Picker Interaction

**Choice:** Inline popover anchored to `+` button on tree container nodes
**Alternatives:**
- Dock palette with target — clicking `+` sets insertion target, user selects from the existing right-dock palette. Re-uses existing component but disconnects action from location
- Between-node drop zones — thin insertion lines between siblings with `+` buttons. Most explicit about position but clutters the tree
**Rationale:** Popover stays close to the action point — the user's attention is on the tree node. Follows Figma/VS Code pattern. The popover can reuse palette filtering logic without requiring the full palette panel.
**Trade-offs:** New popover component to build; palette filtering logic needs to be shared between the inline popover and the full dock palette
**Exploration:** quick

## D3: Structural Edit Actions — Context Menu

**Choice:** Right-click context menu with Move up/down, Move to…, Wrap in…, Replace with…, Duplicate, Delete
**Alternatives:**
- Selection toolbar — floating mini-toolbar on selection. More discoverable but adds visual noise
- Both — context menu + minimal toolbar. More chrome but covers discovery and power-use
**Rationale:** Standard desktop pattern. Keeps the tree visually clean — actions appear on demand. Keyboard shortcuts (Delete, Ctrl+D) provide power-user shortcuts alongside the menu.
**Trade-offs:** Less discoverable for users unfamiliar with right-click. Mitigated by the `+` button being always visible on containers.
**Exploration:** quick

## D4: Drag-and-Drop Scope

**Choice:** Reorder within container + cross-container moves with drop indicators
**Alternatives:**
- Reorder only — drag within same parent only, cross-container via context menu. Simpler but less fluid
- Defer DnD entirely — all moves via context menu. Keyboard-first approach
**Rationale:** Reorder + cross-container covers the most common move scenarios and matches user expectations for a tree-based builder. Drop indicators show valid targets.
**Trade-offs:** DnD is complex to implement well (hit testing, auto-scroll, visual feedback). The facade's `moveToIndex()` needs robustness improvements for cross-container named-record slots.
**Depends on:** D3 (context menu provides fallback for complex moves DnD can't express)
**Exploration:** quick

## D5: Preview Data Strategy

**Choice:** Strategy registry — each component type registers a data strategy that produces sample data matching its needs
**Alternatives:**
- Catalog defaults only — static sampleData blob per catalog entry. Simple but doesn't adapt to user-configured column mappings
- Hybrid — strategy for data-consuming types, catalog fallback for static content types
**Rationale:** Different component types need fundamentally different data shapes (categories+values for charts, typed columns+rows for tables, single value for metrics). A strategy can read the component's current props (which columns are mapped, which dataset) to produce relevant preview data.
**Trade-offs:** More implementation than static blobs. Each new component type needs a strategy (or falls back to a default empty strategy).
**Exploration:** quick

## D6: Dock-Workbench API

**Choice:** Named slots — standard Lit slot pattern (`<pages-dock-workbench><div slot="left">...</div></pages-dock-workbench>`)
**Alternatives:**
- Config object with render callbacks — more programmatic, closer to current ZoneLayoutEngine. Less declarative
- Zone registry — consumers register content via API (`dock.registerZone()`). More dynamic but less declarative
**Rationale:** Named slots are idiomatic Lit and provide clean separation between the dock's layout concerns (resize, collapse, toggle) and the consumer's content. The dock doesn't need to know what's inside each zone.
**Trade-offs:** Slots can't be dynamically reordered at runtime (fixed zone topology). This is fine — the dock's zone layout (left/centre/right/bottom) is structural, not dynamic.
**Exploration:** quick

## D7: Wrap Container Scope

**Choice:** Row, column, and tabs — the three most common layout containers
**Alternatives:**
- All container types — complete but each has different slot semantics, significant test surface
- Row/column only — minimal but misses the tabbed container use case which is the most common non-layout wrap
**Rationale:** Row and column cover layout restructuring. Tabs covers the most common content grouping. Other containers (sidebar, split, form-scope, accordion) can be added later — the descriptor registry and wrap operation are designed to be extensible.
**Trade-offs:** Users can't wrap in sidebar/split/form-scope initially. Context menu can still show "Add sidebar" to create one and then manually move content into it.
**Exploration:** quick

## D8: Preview Rendering

**Choice:** Live render with strategy-generated sample data using the standard pages-runtime
**Alternatives:**
- Schematic wireframe — labelled boxes with component type. Lighter weight but doesn't show real appearance
- Live render with schematic toggle — best of both but two rendering paths to maintain
**Rationale:** Users can't judge layout with empty boxes or wireframes. Live rendering with sample data shows what the real page will look like. The pages-runtime is already available in the builder context.
**Trade-offs:** Requires the full runtime to be loaded in the builder. Strategy-generated data may not perfectly represent real data distributions. Rendering performance under rapid edits needs debouncing.
**Depends on:** D5 (strategy registry provides the data for live rendering)
**Exploration:** quick
