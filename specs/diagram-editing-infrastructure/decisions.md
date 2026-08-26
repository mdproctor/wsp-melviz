# Decisions — Diagram Editing Infrastructure

## D1: Scope relationship to existing Phase 4

**Choice:** Evolution of Phase 4 (Structural Editing) in the visual diagram editor spec
**Alternatives:**
- New phase after Phase 4 — Phase 4 lands basic structural editing first, then this adds interactions later. Delays the full editing UX.
- Standalone infrastructure spec — diagram-agnostic primitives only. Loses the integration context.
**Rationale:** The interactions are complementary to Phase 4's existing add/remove/replace scope. Auto-layout means these drag interactions are tractable — every mutation ends with re-layout, not free-position reconciliation.
**Trade-offs:** Larger Phase 4 scope. But the interactions share infrastructure (constraint checking, re-layout, undo) so building them together avoids duplication. Implementation can be phased (4a: basic structural editing via toolbar/context menu, 4b: drag interactions) without splitting the design.
**Sources:** `docs/specs/2026-08-01-visual-diagram-editor-design.md` Phase 4
**Exploration:** quick
**Status:** captured

## D2: Palette home

**Choice:** Palette component lives in `casehub-diagram` (blocks-ui) as a Lit component with Shadow DOM, consuming stencil registry data from `graph-renderer` via imports
**Alternatives:**
- `graph-renderer` — collocates palette with registry but puts a Lit component in a React-centric package, creating an architectural misfit
- New `pages-diagram-palette` package — more isolation; follows the `pages-property-palette` precedent but adds a package for a component that is inherently tied to the diagram shell
- `graph-core` — splits concern but graph-core is rendering-free and should stay that way
**Rationale:** The spec's §3.1 architecture places `<casehub-diagram-palette>` under `casehub-diagram` (blocks-ui), not under graph-renderer. graph-renderer is React-centric (React 18, ReactDOM, JSX) — a Lit component with Shadow DOM doesn't belong there. The palette is a UI shell component that composes data from `getAllStencils()` and work-registry — it's a consumer of graph-renderer, not part of it. This aligns with the established pattern: `pages-property-palette` is a standalone Lit component that consumes schema data from external sources.
**Trade-offs:** Palette and canvas are in different packages. But data dependency ≠ package co-location — the palette imports from `@casehubio/graph-renderer`, same as any other consumer.
**Sources:** `packages/graph-renderer/src/registry/stencil-registry.ts`, `packages/pages-property-palette/`, spec §3.1
**Exploration:** quick
**Status:** revised (R1-03: moved from graph-renderer to casehub-diagram to align with spec §3.1 architecture and avoid Lit/React package misfit)

## D3: Constraint SPI — static grammar only

**Choice:** Keep `StencilGrammar` as a pure data interface — static connection rules and containment only. No runtime callbacks.
**Alternatives:**
- Add optional `validateConnection(source, target, model): boolean` callback to StencilGrammar — enables domain-specific validation in graph-core but breaks the pure-data contract and creates two-layer dynamic validation confusion with EditPolicy (D8)
- Separate `ConnectionConstraint` SPI in graph-core — cleaner separation but still adds behavioral logic to the data model tier
**Rationale:** `StencilGrammar` (grammar.ts) is a pure data interface: `type`, `connections`, `containment`. The validator (validator.ts) does purely structural validation using these static rules. This purity is architecturally valuable — pure data models are trivially testable, serializable, and have zero framework dependencies. All dynamic validation (context-dependent, runtime-aware) belongs in `EditPolicy.canConnect()` (D8), which has full access to the model and domain knowledge. EditPolicy can consult StencilGrammar's static rules as a first pass before applying domain logic.
**Trade-offs:** Stencil authors can't register dynamic constraints at the grammar level. But dynamic constraints are inherently domain-specific — they belong in the domain adapter's EditPolicy implementation, not in the type-level grammar.
**Sources:** `packages/graph-core/src/grammar.ts`, `packages/graph-core/src/validator.ts`
**Exploration:** quick
**Status:** revised (R1-02: removed validateConnection callback to preserve graph-core's pure data contract; dynamic validation consolidated in EditPolicy D8)

## D4: Drag interaction model — hybrid React Flow + custom pointer events

**Choice:** Use React Flow's built-in connection system for node-to-node connections; custom pointer events with ghost element for interactions React Flow doesn't handle (palette drag, edge insertion, empty-space creation)
**Alternatives:**
- Custom pointer events for ALL interactions — full control but reimplements viewport transform compensation, connection line rendering, and handle integration that React Flow already provides
- HTML5 Drag API — browser handles ghost but no custom styling during drag, drop coordinates don't account for React Flow's viewport transform, and pan-on-drag conflicts
- React Flow's connection system for everything — only covers node-to-node connections; palette drag, edge insertion, and empty-space drop need a separate system regardless
**Rationale:** React Flow Handle components are already rendered on every node (stencil-wrapper.tsx) — source handles and target handles at all four cardinal positions, gated by grammar rules. React Flow v12's connection system provides viewport transform compensation, connection line rendering during drag, `isValidConnection` callback for constraint checking, and integration with these existing Handle components. Using it for connections avoids reimplementing this infrastructure. Custom pointer events remain necessary for interactions that cross DOM boundaries (palette drag starts outside the canvas) or don't involve node handles (edge insertion, empty-space creation).
**Trade-offs:** Two interaction mechanisms on the same canvas. But they don't conflict — React Flow's system owns handle-initiated connections, custom events own everything else. The boundary is clean: if a drag starts on a Handle, React Flow owns it; otherwise, custom events own it.
**Sources:** `packages/graph-renderer/src/stencil-wrapper.tsx` (Handle rendering), `packages/graph-renderer/src/bridge/ReactFlowApp.tsx` (connection callbacks available but unwired)
**Exploration:** quick
**Status:** revised (R1-04: adopted hybrid approach leveraging existing Handle infrastructure and React Flow's connection system for node-to-node connections)

## D5: Node chooser — inline popover at interaction point

**Choice:** Inline popover appearing at the click/drop point with filtered search, rendered in DOM space with viewport-compensated positioning
**Alternatives:**
- Palette highlights valid types — reuses palette UI but user must look away from insertion point
- Modal dialog — most discoverable but heaviest; breaks flow for a quick selection
- React Flow coordinate space (moves with pan/zoom) — technically possible but popovers should stay fixed on screen for usability
**Rationale:** Stays in context — user sees where the node will go. Small floating panel shows valid node types grouped by category. Dismisses on selection or Escape. Renders in DOM space (outside React Flow's viewport transform) with position computed from `reactFlowInstance.flowToScreenPosition()` at the interaction point. Does NOT move with pan/zoom — it's a transient UI overlay, not a graph element. Shares type data source with palette: both consume `EditPolicy.getCreatableTypes()` and `EditPolicy.getInsertableTypes()` (D8) for context-appropriate filtering. The palette shows all types; the node chooser filters based on interaction context (edge insertion → insertable types at that edge, empty-space → creatable types).
**Trade-offs:** New component to build. But it's a simple filtered list — not a complex widget.
**Sources:** None (standard UX pattern)
**Exploration:** quick
**Status:** revised (R1-09: specified DOM-space rendering with viewport compensation; R1-14: clarified shared data source with palette via EditPolicy)

## D6: Delete UX — context-dependent auto-decision with undo

**Choice:** Auto-join when leaf node has exactly 1 inbound + 1 outbound edge; auto-disconnect otherwise with dangling edge cleanup. Undo reverses any wrong choice. Popover with options only for ambiguous multi-edge cases. EditPolicy.getDeleteStrategy() can override the default heuristic per domain.
**Alternatives:**
- Always show popover with options — explicit but adds a click to every delete, even for terminal nodes
- Keyboard modifier (Delete = join, Shift+Delete = disconnect) — fastest for power users but undiscoverable
**Rationale:** The common case (linear chain: A → B → C, delete B, auto-join A → C) is unambiguous and should be instant. Terminal nodes (no outbound) just delete with inbound edge cleanup. Multi-edge nodes (2+ inbound or 2+ outbound) present a brief popover because the join target is ambiguous. For containment nodes (nodes with children), deletion cascades the subtree (existing `removeNode` behavior in edit.ts) — auto-join only fires on leaf nodes without children. `EditPolicy.getDeleteStrategy()` (D8) allows domain adapters to override the default heuristic when the auto-join topology rarely applies (e.g., Case definitions where Workers typically have multiple inbound Binding edges). Domain adapters can opt into "always popover" if preferred.
**Trade-offs:** Auto-decision might surprise users who expected to disconnect. Undo is the safety net. Domain adapters can override via EditPolicy if the topology makes auto-join rare.
**Sources:** `packages/graph-core/src/edit.ts` (removeNode with subtree cascade)
**Exploration:** quick
**Status:** revised (R1-06: added containment cascade handling, EditPolicy override path, and domain topology consideration)

## D7: Interaction coordination — per-handler architecture

**Choice:** One custom drag handler (palette drag) with its own utilities (ghost element, hit-testing, viewport transform). Two click-to-popover flows (edge insertion via `onEdgeClick`, empty-space creation via `onPaneClick`) that trigger the node chooser (D5). React Flow's event system handles native interactions (connections, node drag, pan/zoom, selection, edge reconnection).
**Alternatives:**
- Single InteractionCoordinator managing ALL interactions via state machine — creates dual-control problem with React Flow's internal interaction management; forces negotiation with React Flow's pointer capture for every native interaction
- Per-handler with shared utilities across 3 "custom interactions" — overstates the shared infrastructure; edge insertion and empty-space creation are click handlers, not drag interactions, and don't need ghost elements or hit-testing
- No coordinator, event handlers on canvas — simplest but no structured handler for palette drag
**Rationale:** React Flow IS already a coordinator for its native interactions: node selection, node drag, pan/zoom, box selection, connections, and edge reconnection (per revised D4). Only palette drag is a true custom drag interaction — it starts outside the canvas (in the Lit palette component), crosses the DOM boundary into the React Flow viewport, needs ghost element rendering, hit-testing against canvas nodes, and viewport transform compensation. Edge insertion and empty-space creation are click→popover flows: React Flow fires the event callback (`onEdgeClick`/`onPaneClick`), the handler opens the node chooser popover (D5), the user selects a type, and the mutation commits. No drag state, no ghost, no hit-testing. Escape cancels the palette drag (cleans up ghost) or dismisses the node chooser popover. A boolean flag (`paletteDragActive`) prevents palette drag from starting during an open popover or vice versa.
**Trade-offs:** Ghost element, hit-testing, and viewport transform utilities are palette-drag-specific rather than shared. This is simpler — no premature abstraction across unlike interaction shapes.
**Sources:** GE-20260825-309197 (coordinator pattern), GE-20260826-ee71b5 (DOM event scoping)
**Exploration:** quick
**Status:** revised (R1-05: scoped to custom interactions only; R1-15: escape handling; R2-02: clarified palette drag as the only true custom drag interaction)

## D8: EditPolicy SPI

**Choice:** An `EditPolicy` SPI interface defined in `graph-renderer`, implemented by the domain adapter (e.g., graph-stencil-case). Methods: `canConnect()`, `getInsertableTypes()`, `getCreatableTypes()`, `canDelete()`, `getDeleteStrategy()`. graph-renderer calls these during interactions to determine valid targets, insertable types, and delete strategies. Registered via `setEditPolicy(policy)` — a simple setter on graph-renderer's module scope.
**Alternatives:**
- Grammar-only inference — derive everything from StencilGrammar rules. Simpler but can't express domain rules beyond grammar (e.g., "this worker already has 3 bindings and the team convention is max 3")
- Callbacks on palette items — each stencil knows where it can go. Distributed, hard to compose cross-stencil rules
- Define EditPolicy in graph-core — would add behavioral dependency to the pure data tier
**Rationale:** Centralises all domain-specific editing logic in one interface. EditPolicy is DEFINED in graph-renderer (the framework tier that owns the editing interactions) and IMPLEMENTED by the domain adapter (graph-stencil-case, which has full domain knowledge). `canConnect()` subsumes all dynamic validation — it can consult `getGrammar()` for static rules, then apply domain logic. This consolidates the validation that was previously split across D3 and D8. Signatures may be refined during implementation as interactions are built; the interface shape (what questions interactions ask of the domain) is stable even if parameter types evolve.
**Trade-offs:** New SPI to implement per domain. But the alternative is scattered constraint logic across stencils and grammar.
**Sources:** `packages/graph-core/src/grammar.ts`, `packages/graph-core/src/validator.ts`, `packages/graph-core/src/edit.ts`
**Exploration:** quick
**Status:** revised (R1-02: consolidated dynamic validation from D3; R1-07: specified DI mechanism via setEditPolicy; R1-12: EditPolicy defined in graph-renderer, not graph-core — preserves graph-core purity)

## D9: React Flow interaction layer usage

**Choice:** Use React Flow's native interaction APIs where they apply; custom interactions only for what React Flow doesn't handle
**Alternatives:**
- Bypass React Flow's interaction layer entirely — reimplement connections, node drag, pan/zoom with custom pointer events
- Use React Flow for everything — not possible since palette drag, edge insertion, and empty-space creation aren't React Flow concepts
**Rationale:** React Flow provides well-tested, viewport-aware interaction handling for connections (`onConnect`/`onConnectStart`/`onConnectEnd`/`isValidConnection`), node movement (`onNodeDrag`), pan/zoom, and selection. Reimplementing these creates maintenance burden and viewport transform bugs. The Handle components rendered on every node (stencil-wrapper.tsx) are the entry point for React Flow's connection system — they should be connected, not bypassed.

Per-interaction analysis:

| Interaction | Category | Owner | Mechanism |
|---|---|---|---|
| Connection drawing | Drag | React Flow | `onConnect`/`isValidConnection` + existing Handles |
| Edge reconnection | Drag | React Flow | `reconnectEdges` prop + `onReconnect` callbacks |
| Node moving | Drag | React Flow | `onNodeDrag` → re-layout on drop |
| Node selection | Click | React Flow | `onNodeClick` (already wired) |
| Pan/zoom | Gesture | React Flow | Built-in (already working) |
| Box selection | Drag | React Flow | `selectionOnDrag` (already enabled) |
| Palette drag | Drag | Custom | Pointer events + ghost (crosses DOM boundary) |
| Edge insertion | Click→popover | React Flow event | `onEdgeClick` → node chooser (D5) |
| Empty-space creation | Click→popover | React Flow event | `onPaneClick` → node chooser (D5) |

Note: Edge insertion and empty-space creation use React Flow's event routing (`onEdgeClick`, `onPaneClick`) but are not React Flow "native" interactions — they trigger application-level popover flows. They are not drag interactions and do not need ghost elements, hit-testing, or viewport transform utilities. Only palette drag is a true custom drag interaction.

**Trade-offs:** Custom drag interactions (palette drag only) must respect React Flow's pointer capture for native interactions. Clean boundary: Handle-initiated drags are React Flow's; palette-initiated drags are custom; click events route through React Flow callbacks to application logic.
**Sources:** `packages/graph-renderer/src/bridge/ReactFlowApp.tsx`, `packages/graph-renderer/src/stencil-wrapper.tsx`
**Exploration:** N/A (implicit decision surfaced by review)
**Status:** revised (R2-01: added edge reconnection; R2-02: clarified interaction categories — edge insertion and empty-space creation are click→popover flows, not drag interactions)

## D10: graph-core architectural boundary — pure data

**Choice:** graph-core remains a pure data model package — no runtime callbacks, no framework dependencies, no behavioral logic
**Alternatives:**
- Allow callbacks in graph-core (e.g., `validateConnection` on StencilGrammar) — more convenient for stencil authors but breaks testability, serializability, and graph-core's independence
- Move EditPolicy SPI to graph-core — centralises all SPIs but adds a behavioral dependency to the data tier
**Rationale:** graph-core today is pure data and functions: `grammar.ts` (data interface), `model.ts` (data types), `validator.ts` (pure function over data), `edit.ts` (pure functions producing new models). This purity makes it trivially testable (no mocks needed), serializable (grammars can be stored/transmitted), and independent (zero framework dependencies). Behavioral SPIs (EditPolicy) belong in graph-renderer, which owns the interactions that need them. Static structural validation stays in graph-core; dynamic domain validation goes through EditPolicy in graph-renderer.
**Sources:** `packages/graph-core/src/grammar.ts`, `packages/graph-core/src/model.ts`, `packages/graph-core/src/validator.ts`, `packages/graph-core/src/edit.ts`
**Exploration:** N/A (implicit decision surfaced by review)
**Status:** captured

## D11: Undo integration for editing interactions

**Choice:** YAML snapshot pushed to undo stack before each mutation. Compound operations (drag → create → connect → re-layout) are a single undo unit. Cancelled interactions push nothing.
**Alternatives:**
- Per-step undo entries for compound operations — fine-grained but can leave the graph in intermediate states on partial undo
- Coordinator-managed undo — coordinator pushes/pops undo entries around interaction lifecycle. Adds coupling between coordinator and undo stack
**Rationale:** The spec (§2.6) defines YAML-snapshot undo: each edit pushes the previous YAML string onto the stack. The undo unit is one logical edit operation. For drag interactions, the "logical edit" is the final mutation (create node, add edge, delete node), not the drag gesture itself. Snapshot is taken before the mutation; if the mutation fails or is cancelled, no snapshot is pushed. For compound operations like palette drag (create node + auto-connect + re-layout), only one snapshot is pushed covering the entire compound operation — undoing it restores the graph to the state before the drag started. The `casehub-diagram` component owns the undo stack (per spec §2.6). Interaction handlers call a `pushUndo()` callback before committing their mutations.
**Sources:** `docs/specs/2026-08-01-visual-diagram-editor-design.md` §2.6
**Exploration:** N/A (implicit decision surfaced by review)
**Status:** captured
