# Decisions — Diagram Editing Infrastructure

## D1: Scope relationship to existing Phase 4

**Choice:** Evolution of Phase 4 (Structural Editing) in the visual diagram editor spec
**Alternatives:**
- New phase after Phase 4 — Phase 4 lands basic structural editing first, then this adds interactions later. Delays the full editing UX.
- Standalone infrastructure spec — diagram-agnostic primitives only. Loses the integration context.
**Rationale:** The interactions are complementary to Phase 4's existing add/remove/replace scope. Auto-layout means these drag interactions are tractable — every mutation ends with re-layout, not free-position reconciliation.
**Trade-offs:** Larger Phase 4 scope. But the interactions share infrastructure (constraint checking, re-layout, undo) so building them together avoids duplication.
**Sources:** `docs/specs/2026-08-01-visual-diagram-editor-design.md` Phase 4
**Exploration:** quick
**Status:** captured

## D2: Palette home

**Choice:** Palette component lives in `graph-renderer` alongside `GraphCanvas`
**Alternatives:**
- New `pages-diagram-palette` package — more isolation but another package; palette is tightly coupled to stencil registry anyway
- `graph-core` — splits concern but graph-core is currently rendering-free and should stay that way
**Rationale:** Palette consumes `StencilDescriptor[]` from the stencil registry and `WorkStencil[]` from work-registry. Both registries are in or consumed by graph-renderer. Keeps all rendering infrastructure together. Domain stencil packages just register — they don't own palette UI.
**Trade-offs:** graph-renderer grows. But palette is a rendering concern, not a data model concern.
**Sources:** `packages/graph-renderer/src/registry/stencil-registry.ts`, `packages/graph-work-registry/`
**Exploration:** quick
**Status:** captured

## D3: Constraint SPI — extend StencilGrammar

**Choice:** Extend `StencilGrammar` with an optional `validateConnection(source, target, model): boolean` callback
**Alternatives:**
- Separate `ConnectionConstraint` SPI — cleaner separation but doubles registration surface; stencil authors must register both grammar and constraint provider
- Static rules only — all constraint logic in the domain adapter. Simpler but loses standardised drop-zone feedback in graph-renderer
**Rationale:** Static rules (allowedTo/From, cardinality) remain the fast path for most stencils. The dynamic validator only runs for stencils that register one, enabling domain-specific logic (e.g., "this Binding already has a capability target, so no more outbound connections") without leaving graph-core.
**Trade-offs:** graph-core gains a callback type. But it's optional — stencils without dynamic constraints are unaffected.
**Sources:** `packages/graph-core/src/grammar.ts`, `packages/graph-core/src/validator.ts`
**Exploration:** quick
**Status:** captured

## D4: Drag interaction model — pointer events with ghost

**Choice:** Pointer events (pointerdown/pointermove/pointerup) with ghost element
**Alternatives:**
- HTML5 Drag API — browser handles ghost but no custom styling during drag, drop coordinates don't account for React Flow's viewport transform, and pan-on-drag conflicts
- React Flow's built-in connection system — native for node-to-node connections but only covers that one interaction; palette drag, edge insertion, and empty-space drop need a separate system
**Rationale:** Full control over visual feedback (valid/invalid indicators, snap previews). Works inside React Flow's canvas coordinate system with viewport transform compensation. Consistent with the existing dock-drag system in pages-runtime. All 11 interactions use the same pointer event foundation.
**Trade-offs:** More code than HTML5 drag. But the ghost element, hit-testing, and visual feedback are all custom anyway.
**Sources:** `packages/pages-runtime/src/dock-drag.ts`, GE-20260826-ee71b5 (DOM event scoping), GE-20260825-309197 (coordinator pattern)
**Exploration:** quick
**Status:** captured

## D5: Node chooser — inline popover at interaction point

**Choice:** Inline popover appearing at the click/drop point with filtered search
**Alternatives:**
- Palette highlights valid types — reuses palette UI but user must look away from insertion point
- Modal dialog — most discoverable but heaviest; breaks flow for a quick selection
**Rationale:** Stays in context — user sees where the node will go. Small floating panel shows valid node types grouped by category. Dismisses on selection or Escape. For diagrams with many stencil types (case + marketplace work stencils), filtered search prevents overwhelming the user.
**Trade-offs:** New component to build. But it's a simple filtered list — not a complex widget.
**Sources:** None (standard UX pattern)
**Exploration:** quick
**Status:** captured

## D6: Delete UX — context-dependent auto-decision with undo

**Choice:** Auto-join when node has exactly 1 inbound + 1 outbound edge; auto-disconnect otherwise with dangling edge cleanup. Undo reverses any wrong choice. Popover with options only for ambiguous multi-edge cases.
**Alternatives:**
- Always show popover with options — explicit but adds a click to every delete, even for terminal nodes
- Keyboard modifier (Delete = join, Shift+Delete = disconnect) — fastest for power users but undiscoverable
**Rationale:** The common case (linear chain: A → B → C, delete B, auto-join A → C) is unambiguous and should be instant. Terminal nodes (no outbound) just delete with inbound edge cleanup. Multi-edge nodes (2+ inbound or 2+ outbound) present a brief popover because the join target is ambiguous.
**Trade-offs:** Auto-decision might surprise users who expected to disconnect. Undo is the safety net.
**Sources:** `packages/graph-core/src/edit.ts` (removeNode with cascade)
**Exploration:** quick
**Status:** captured

## D7: Interaction coordinator — single with modes

**Choice:** Single `InteractionCoordinator` managing all drag interactions via a state machine: idle → connecting / inserting / moving / palette-drag
**Alternatives:**
- Separate coordinator per interaction — self-contained but shared concerns (ghost element, hit-testing, drop zone highlighting) get duplicated
- No coordinator, event handlers on canvas — simpler for few interactions but hard to extract later
**Rationale:** One coordinator = one place for pointer capture, ghost management, and cleanup. Each mode implements a common interface with its own visual feedback and hit-test logic. Follows the garden entry's advice: design the coordinator interface before the state machine grows.
**Trade-offs:** Single coordinator is larger. But mode implementations are separate — the coordinator is a dispatcher, not a monolith.
**Sources:** GE-20260825-309197 (coordinator pattern), GE-20260826-ee71b5 (DOM event scoping)
**Exploration:** quick
**Status:** captured

## D8: Full EditPolicy SPI

**Choice:** A single `EditPolicy` SPI interface that the domain adapter implements: `canConnect()`, `getInsertableTypes()`, `getCreatableTypes()`, `canDelete()`, `getDeleteStrategy()`. graph-renderer calls these during interactions.
**Alternatives:**
- Grammar-only inference — derive everything from StencilGrammar rules. Simpler but can't express domain rules beyond grammar (e.g., "Goals are always terminal" is in grammar, but "this worker already has 3 bindings and the team convention is max 3" is not)
- Callbacks on palette items — each stencil knows where it can go. Distributed, hard to compose cross-stencil rules
**Rationale:** Centralises all domain-specific editing logic in one interface. graph-renderer is the consumer — it calls EditPolicy during drag interactions to determine valid targets, insertable types, and delete strategies. The domain adapter (e.g., graph-stencil-case) implements EditPolicy with full knowledge of the domain model.
**Trade-offs:** New SPI to implement per domain. But the alternative is scattered constraint logic across stencils and grammar.
**Depends on:** D3 (StencilGrammar extension provides the static layer; EditPolicy provides the dynamic layer)
**Sources:** `packages/graph-core/src/grammar.ts`, `packages/graph-core/src/validator.ts`, `packages/graph-core/src/edit.ts`
**Exploration:** quick
**Status:** captured
