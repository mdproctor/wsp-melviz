# Design Decisions — Visual Toolbar Redesign

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
