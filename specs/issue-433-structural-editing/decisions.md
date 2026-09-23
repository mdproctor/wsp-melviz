# Design Decisions — #433 Structural Editing

## D1: Toolbar trigger model

**Choice:** Selection-based with single trigger icon. A `⋮` icon appears on the selected scope outline's top-right corner. Hovering the icon expands to the full button set. No hover-based per-component toolbars.
**Alternatives:**
- Hover-based (current) — toolbar appears on every component mouseover. Rejected: visual noise, no selection precision.
- Always-visible buttons on selection — all buttons shown at once on the scope outline. Rejected: clutters the selection outline.
**Rationale:** Single icon keeps the visual clean. Hover-expand is discoverable without noise. Selection-based means buttons follow the user's intent (what they selected), not cursor position.
**Trade-offs:** One extra hover interaction to reach buttons. Acceptable because the trigger is always visible and expansion is instant.
**Sources:** HANDOFF.md design decision, ComponentOverlayManager (overlay/component-overlay.ts), builder-shell.ts:354-405 (scope overlay)
**Exploration:** quick
**Status:** captured

## D2: Button set

**Choice:** Full 5-button set: Add (+), Insert (↓), Cut (✂), Copy (⎘), Delete (✕). Buttons adapt to the selected node type.
**Alternatives:**
- Match tree (4 buttons, no delete) — less discoverable for delete action in visual mode.
- Minimal (cut/copy/delete only) — forces add/insert through tree. Breaks visual-mode workflow.
**Rationale:** Visual mode should be self-contained — users shouldn't need to switch to tree for common operations. Delete is especially important in visual mode where you can see what you're removing.
**Trade-offs:** Denser expanded toolbar. Mitigated by the single-icon trigger — buttons only appear on deliberate hover.
**Sources:** builder-tree.ts button set, component-overlay.ts:50 (current overlay buttons)
**Exploration:** quick
**Status:** captured

## D3: Architecture — SelectionOverlay class

**Choice:** New `SelectionOverlay` class that owns both the scope outline (blue border) and toolbar as one visual unit. Replaces both the inline scope overlay in `_applyPreviewHighlight` and `ComponentOverlayManager`. CSS-driven hover expansion (no JS mousemove tracking).
**Alternatives:**
- Inline in builder-shell — extend `_applyPreviewHighlight`. No new abstractions but method grows to 120+ lines mixing positioning, element creation, event wiring, and hover management.
- Lit web component — shadow DOM encapsulation. Overkill for an imperatively-positioned internal element nested inside builder-shell's own shadow DOM.
- Repurpose ComponentOverlayManager — fundamentally wrong abstraction (N overlays for N components vs. 1 overlay for 1 selection). Would be a full rewrite wearing the old name.
**Rationale:** The scope outline and toolbar share lifecycle, trigger, and DOM parent — they're one visual unit. A dedicated class gives clean separation: builder-shell owns selection state and bounds computation, SelectionOverlay owns rendering and interaction. Testable in isolation. Replaces two incomplete systems with one focused class.
**Trade-offs:** New file, but replaces component-overlay.ts which gets deleted. Net reduction in complexity.
**Sources:** component-overlay.ts (to be deleted — overlay events are unwired in production), builder-shell.ts:354-405 (_applyPreviewHighlight), scope-collector.ts
**Exploration:** deep-analysis
**Status:** captured

## D4: Per-component hover overlays

**Choice:** Remove entirely. No per-component hover overlays. Selection-based toolbar is the only visual interaction layer.
**Alternatives:**
- Keep as subtle highlight (no toolbar) — faint outline on hover to preview click target. Adds visual hint but also adds DOM noise.
- Keep with toolbar — both hover and selection toolbars coexist. Confusing — two toolbar sources for the same actions.
**Rationale:** Click already selects the component and shows the scope outline. Adding a hover preview adds marginal value while keeping the DOM-heavy per-component overlay system alive.
**Trade-offs:** No hover preview of click target. Acceptable because the scope outline appears immediately on click.
**Sources:** ComponentOverlayManager (overlay/component-overlay.ts — entire class deleted)
**Exploration:** quick
**Status:** captured

---

# Phase 2 — Contextual Add Pickers + Position-Complete Insertion

## D5: Design scope

**Choice:** Combined spec covering contextual add pickers (schema-constrained type filtering) and position-complete insertion (N+1 insertion points). These share constraint logic, insertion API, and UX.
**Alternatives:**
- Add pickers only — defers insertion points. Artificial split since pickers need position awareness anyway.
- All three priorities (including move/wrap/replace) — too large; move has distinct UX (move mode, drop targets, action chooser).
**Rationale:** The `+` insertion points between siblings ARE the contextual add pickers — they combine "where" and "what" into one interaction.
**Trade-offs:** Larger spec than pickers alone. Acceptable because the constraint logic is shared.
**Sources:** Issue #433 priorities 1-2, HANDOFF.md What's Next
**Exploration:** quick
**Status:** captured

## D6: Constraint model

**Choice:** Structural hierarchy only. Enforce the document schema's structural rules — rows only under pages, columns only under rows, components only in columns/containers. All 55 component types are valid inside any component container slot. Keep existing `contextRelevance` for soft promotion/demotion (form types promoted in form-scope, data types need-prereq without datasets).
**Alternatives:**
- Per-type restrictions — add `allowedChildTypes` to container descriptors (e.g. form-scope only accepts form components). More precise but requires defining the constraint matrix for all 10 containers. Over-constrains: users may want unusual combinations.
- Soft promotion only — no hard constraints, just reorder. Too permissive: lets users create structurally invalid documents.
**Rationale:** The structural rules are already implicit in the document schema. Hard constraints prevent invalid structure; soft promotion handles editorial guidance. Clean separation of concerns.
**Trade-offs:** Users can add any component type inside any container — a chart inside form-scope is allowed. Acceptable because contextRelevance already demotes irrelevant types.
**Sources:** document-schema.ts (Zod hierarchy), container-descriptors.ts, palette-context.ts (contextRelevance), PP-20260907-951cbe (generated schemas protocol)
**Exploration:** quick
**Status:** captured

## D7: Insertion point surfaces

**Choice:** Both tree view and visual preview show inter-sibling insertion points.
**Alternatives:**
- Tree only — visual preview keeps only SelectionOverlay `+` (add as child). Misses the visual-first editing workflow.
- Visual preview only — tree loses precision. Users already expect tree to be the structural editing tool.
**Rationale:** Tree is the precision tool for structural editing; visual preview is for contextual editing. Both surfaces need insertion points for position-complete insertion.
**Trade-offs:** DOM complexity in visual preview overlay layer. Mitigated by showing insertion points only when parent is selected (D8).
**Sources:** builder-tree.ts (existing tree `+` buttons), selection-overlay.ts (overlay positioning)
**Exploration:** quick
**Status:** captured

## D8: Visual insertion point visibility

**Choice:** Insertion points appear in visual preview when a container is selected. Selecting a column shows `+` between its children; selecting a page shows `+` between its rows/components.
**Alternatives:**
- On hover near gap — insertion point appears when mouse hovers the gap between components. Hard to target precisely.
- Always in edit mode — all insertion points visible everywhere. Visual noise with 20+ components.
**Rationale:** Ties into existing selection model. Select parent → see insertion points → click one → picker opens. Natural flow, no visual noise until the user signals intent.
**Trade-offs:** Users must select the parent first — can't click insertion points directly without a selection. Acceptable because the selection model is already established (SelectionOverlay D1-D4).
**Sources:** selection-overlay.ts (selection model), builder-shell.ts (_applyPreviewHighlight)
**Exploration:** quick
**Status:** captured

## D9: Constraint function location and API

**Choice:** New `insertion-constraints.ts` in `pages-document` with two functions: `allowedTypesAt(parentNodeType)` returns valid child types for a parent; `allowedTypesForSiblingOf(nodeType)` derives the parent context for sibling insertion. Returns an `InsertionConstraint` object with `structuralTypes` (row, column) and `componentTypes` (the 55 component types, or empty).
**Alternatives:**
- In `pages-builder` alongside palette-context — couples constraints to UI. Other consumers (validation, programmatic API) can't use it.
- Extend `contextRelevance` functions on catalog entries — scatters logic across 55 entries instead of centralizing rules.
**Rationale:** Constraints are document-level rules, not UI concerns. `pages-document` already owns `container-descriptors.ts` and `PageDocument`. The builder consumes the function to filter the picker; the model layer consumes it for validation.
**Trade-offs:** New dependency from `pages-builder` on `pages-document` for constraints. This dependency already exists (builder uses `PageDocument`).
**Sources:** container-descriptors.ts, palette-context.ts, document-schema.ts (structural hierarchy)
**Exploration:** quick
**Status:** captured

## D10: Visual insertion point rendering

**Choice:** Extend `SelectionOverlay` with `showInsertionPoints(childBounds, parentPath, parentNodeType)` and `hideInsertionPoints()`. Insertion-point DOM elements are rendered in the overlay-root (same layer as the scope outline), positioned vertically between child bounding rects. Each `+` element serves as the picker anchor and dispatches `selection-insert-at` with `{ parentPath, index, parentNodeType, target }`.
**Alternatives:**
- Inject into iframe DOM — tighter visual integration but breaks iframe isolation and complicates preview rendering.
- Separate InsertionOverlay class — cleaner separation but adds a class for something tightly coupled to selection state.
**Rationale:** Same pattern as the existing scope outline — pure overlay, no iframe DOM changes. SelectionOverlay already owns the scope outline lifecycle; insertion points share the same lifecycle (appear on selection, disappear on deselection).
**Trade-offs:** SelectionOverlay grows in responsibility. Acceptable because insertion points are fundamentally part of the selection interaction.
**Sources:** selection-overlay.ts (overlay-root pattern), scope-collector.ts (child bounds computation)
**Exploration:** quick
**Status:** captured

## D11: Tree insertion point interaction

**Choice:** Tree renders insertion-point indicators (thin line + centered `+`) between sibling nodes at all N+1 positions when a parent is expanded. Low-opacity until hovered. Clicking fires `tree-insert-at` with `{ parentPath, index, parentNodeType }`, bypassing the position picker.
**Alternatives:**
- No tree insertion points — rely on existing `↓` insert button + position picker. Two-step flow (choose before/after, then choose type) is slower than one-click.
- Always-visible insertion points — visual noise in deep trees.
**Rationale:** Explicit insertion points give one-click position + type selection. The existing `↓` button stays as a keyboard-accessible alternative. Subtle-until-hovered keeps the tree clean.
**Trade-offs:** More DOM elements in the tree. Mitigated by only rendering insertion points for expanded parents.
**Sources:** builder-tree.ts (tree rendering, existing inline buttons)
**Exploration:** quick
**Status:** captured

## D12: Mutation layer completeness

**Choice:** Add missing `insertAt` methods to `PageDocument` model classes: `page.insertComponentAt(index, type, props)`, `page.insertColumnAt(index)`, `row.insertColumnAt(index)`. All mutation methods (`addChild`, `insertAt` variants) validate against `allowedTypesAt()` and throw on invalid types.
**Alternatives:**
- UI-only filtering without model validation — programmatic errors can create invalid documents.
- Validation as a separate pass (like graph-core's `validateConstraints`) — allows invalid intermediate states. Unnecessary for the page builder where mutations are atomic.
**Rationale:** Defense in depth. UI filters prevent user errors; model validation catches programmatic errors. The constraint rules are simple enough that per-mutation validation adds negligible cost.
**Trade-offs:** Existing code that calls `addComponent()` with structurally invalid types will break. Acceptable — those cases are bugs (adding a row inside a component, for example) that should fail explicitly.
**Sources:** page-document.ts (addComponent, insertComponentAt, addRow, insertRowAt), PP-20260916-b6f3e8 (all-mutations-through-applyEdit)
**Exploration:** quick
**Status:** captured
