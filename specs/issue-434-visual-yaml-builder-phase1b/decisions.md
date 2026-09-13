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

**Choice:** Hybrid — strategy registry for data-consuming component types, render-as-is for everything else
**Alternatives:**
- Pure strategy registry — every component type registers a strategy. Forces 33+ no-op strategies for types that don't consume data (layout, content, form, workbench types)
- Catalog defaults only — static sampleData blob per catalog entry. Simple but doesn't adapt to user-configured column mappings
**Rationale:** Only ~20 of 56 catalog entries consume datasets (charts, tables, metrics, data components). Layout, content, form, and workbench types render meaningfully without sample data. Strategies are only needed where a component would show an empty box without data. The registry checks for a registered strategy; if absent, the component renders with its default props — no empty fallback needed.
**Trade-offs:** Two paths (strategy vs render-as-is), but the "render-as-is" path is the absence of a strategy, not a separate system.
**Status:** revised (from R1-07 — original dismissed hybrid without analysing the component distribution)
**Exploration:** quick

## D6: Dock-Workbench API

**Choice:** Extract the existing dock-workbench's layout logic into a `<pages-dock-workbench>` Lit component using named slots. Refactor, not duplicate.
**Alternatives:**
- Config object with render callbacks — more programmatic, closer to current ZoneLayoutEngine. Less declarative
- Zone registry — consumers register content via API (`dock.registerZone()`). More dynamic but less declarative
- Leave existing runtime pipeline untouched, only refactor the builder shell — misses the reuse opportunity
**Rationale:** The codebase already has a dock-workbench with zone layout, panel visibility, and split preservation (via ZoneLayoutEngine in pages-runtime). The builder shell has a bespoke hardcoded dock doing the same job. Extracting the existing logic into a Lit component serves both consumers: the runtime pipeline renders a `<pages-dock-workbench>` instead of raw divs, and the builder shell replaces its bespoke dock with the same component. Named slots are idiomatic Lit — the dock manages resize/collapse/toggle, consumers provide content.
**Trade-offs:** Refactoring the runtime pipeline to emit the Lit component is a migration — existing dock-workbench YAML must continue working. The `workbench-integration-pattern` protocol says no new packages, so this component should live in an existing package (pages-primitives or pages-ui).
**Status:** revised (from R1-01 — clarified as refactor of existing infrastructure, not parallel creation)
**Exploration:** quick

## D7: Wrap Operations — Two Distinct Categories

**Choice:** Separate "Wrap in row/column" (page-level restructuring) from "Wrap in container" (descriptor-driven container insertion). Phase 1b supports both categories but implements container wrap for tabs initially.
**Alternatives:**
- Treat all wraps uniformly — conflates two fundamentally different operations (page layout restructuring vs container insertion). The facade APIs differ: row/column wrap calls page-level addRow/addColumn with migration; container wrap uses the descriptor registry
- Row/column only — misses the primary container use case (tabs)
- All container types — significant test surface, but the mechanism is registry-driven so extensibility is inherent
**Rationale:** "Wrap in row" means restructuring the page layout mode (flat → rows) and moving components into a row/column hierarchy. "Wrap in tabs" means inserting a new `type: tabs` component with the target as first child. Different operations at different model levels. The context menu shows both: "Layout" submenu (Row, Column) and "Container" submenu (Tabs, with other descriptor-registered types added later).
**Trade-offs:** Two wrap code paths. Justified because the model-level operations genuinely differ. Container wrap is extensible via the descriptor registry — adding accordion/sidebar/split later requires no new code paths, just adding entries to the wrap menu filter.
**Status:** revised (from R1-04 — separated page restructuring from container wrapping)
**Depends on:** D9 (facade operations), D11 (undo transactions for compound wraps)
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

## D9: Facade Structural Operations API

**Choice:** Extend existing node interfaces — `ComponentNode` gets `addChild()`, `wrapIn()`, `replaceWith()`; `PageNode`/`RowNode`/`ColumnNode` get `insertChildAt()`. Compound operations use D11's transaction API for atomic undo.
**Alternatives:**
- StructuralEditor service — separate class that takes PageDocument and performs compound operations. Solves parent-access and undo-transaction concerns in one design. But: `moveToIndex()` and `duplicate()` already live on `ComponentNode`; splitting some operations to a service and leaving others on nodes creates an inconsistent API surface
**Rationale:** Operations feel natural on nodes — `node.wrapIn('tabs')`. Node paths are valid within a single render cycle (facade re-creates node objects on every document change via re-parse). For compound operations, D11's transaction API ensures atomic undo — the compound snapshots once at transaction start, then internal mutations skip individual undo pushes. Parent path derivation via `this.path.slice(0, -1)` is correct within a transaction because no external mutations intervene.
**Trade-offs:** Node interfaces grow. Parent-access correctness depends on the transaction model (D11) preventing interleaved mutations.
**Depends on:** D11 (undo transaction model)
**Sources:** `packages/pages-document/src/page-document.ts`, `packages/pages-document/src/container-descriptors.ts`
**Status:** revised (from R1-02, R1-03 — addressed undo transaction gap and parent-access concern)
**Exploration:** quick

## D10: Preview Data Strategy Location

**Choice:** In `pages-builder` package under `src/data/strategies/`
**Alternatives:**
- In pages-document — co-located with facade but conflates model with rendering concerns
- New pages-preview package — clean separation but unnecessary package overhead for a small surface
**Rationale:** Preview data is a builder concern — it serves the builder's preview mode. Strategies need to read component properties (from the facade) and produce data shapes (from pages-data types). Both are available as imports from pages-builder.
**Trade-offs:** If preview data is ever needed outside the builder (e.g. showcase, documentation), it would need to be extracted later. Acceptable risk — no current consumer outside the builder.
**Exploration:** quick

## D11: Undo Transaction Model

**Choice:** `doc.beginTransaction()` / `doc.commitTransaction()` API that suppresses individual undo pushes and captures a single snapshot at transaction start
**Alternatives:**
- Low-level mutation methods — internal variants of operations that skip undo, called only by compound operations. Doubles the API surface and risks callers using the wrong variant
- Snapshot-restore pattern — compound operations snapshot manually then use raw YAML AST manipulation. Bypasses the facade's typed API, loses validation and format guarantees
**Rationale:** Transaction/commit is the standard pattern for compound operations with atomic undo. `beginTransaction()` saves the current document state and sets a flag. All mutations during the transaction skip `_pushUndoInternal()`. `commitTransaction()` clears the flag — the pre-transaction snapshot is already on the undo stack. Nested transactions are not needed (compound operations don't compose into deeper compounds).
**Trade-offs:** Adds 2 methods to `PageDocument`. Every compound operation must be wrapped in try/finally to ensure `commitTransaction()` is called even on error — otherwise the undo stack is corrupted.
**Sources:** Review R1-02, R1-10 — identified as the single most impactful unstated assumption
**Exploration:** quick

## D12: Structural Editing Surface Scope

**Choice:** Tree-panel-only for Phase 1b. All structural operations are invoked from the tree (context menu, `+` picker, DnD). The facade API (D9) is surface-agnostic — tree, preview, and YAML editor can invoke the same operations.
**Alternatives:**
- Multi-surface — support structural operations from the visual preview (direct manipulation) and YAML editor (LSP refactoring). Complete but significantly more scope
**Rationale:** The tree is the natural structural editing surface — it mirrors the document hierarchy. Visual preview supports click-to-select (existing), which feeds the tree selection. YAML editor structural operations (via LSP) are a separate concern for a later phase. Keeping the API surface-agnostic means future surfaces don't require facade changes — only UI wiring.
**Trade-offs:** Users can't drag components in the visual preview. Acceptable for Phase 1b — the preview is primarily for viewing, not editing.
**Sources:** Review R1-11 — made implicit assumption explicit
**Exploration:** quick

## D13: Named-Record Slot Targeting for DnD

**Choice:** New `moveToSlot(target: { path; slotName: string }, index: number)` operation for moving into named-record container slots (tabs, pills, accordion, etc.). DnD into named-record containers prompts for slot name.
**Alternatives:**
- Scope DnD to array-based containers only — simpler but 7 of 10 container types use named-record slots, making DnD mostly useless for containers
- Auto-generate slot names — "Tab N" / "Section N". Avoids the prompt but produces generic names the user would rename anyway
**Rationale:** `moveToIndex()` operates on array-based sequences. Named-record slots (tabs, pills, accordion, carousel, stack, menu, tree) require a string key for the map entry. When a user drags a component into a tabs container, the drop target identifies the slot name (existing tab name for insertion, or triggers a "New tab" prompt for a new slot). The `moveToSlot()` operation handles the YAML structure: create/navigate the named entry, insert the component into the child array.
**Trade-offs:** DnD into containers with named slots requires an extra interaction (slot name). Existing tabs show as drop targets directly — dropping onto "Tab 1" moves into that tab without prompting. Only creating a new slot requires a name prompt.
**Depends on:** D4 (DnD scope), D11 (transaction for atomic move)
**Sources:** Review R1-05 — moveToIndex cannot express named-record targeting
**Exploration:** quick

## D14: Property Migration on Replace

**Choice:** Preserve common properties by key match when replacing a component type. Properties that exist in both the source and target schemas transfer; properties unique to the source are dropped; properties unique to the target get schema defaults.
**Alternatives:**
- Schema-aware migration with type coercion — correct but complex, overkill for Phase 1b
- Drop all properties — simple but destructive, bad UX when replacing similar components (bar-chart → line-chart)
**Rationale:** Most replace operations swap between similar types within a category (chart → chart, form → form). Key-matching preserves `lookup`, `dataset`, `filter`, `title` and other shared properties. The `componentSchemaRegistry` provides the target schema for defaults. This is the 80/20 solution — handles the common case correctly without schema graph traversal.
**Trade-offs:** False matches possible (same key name, different semantics across types). In practice, the Pages component property namespace is consistent enough that key-matching is correct.
**Depends on:** D9 (replaceWith operation)
**Sources:** Review R1-12 — replace without property migration is destructive
**Exploration:** quick
