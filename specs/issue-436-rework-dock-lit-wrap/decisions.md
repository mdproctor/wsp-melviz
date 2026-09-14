## D1: Light DOM vs Shadow DOM

**Choice:** Light DOM (`createRenderRoot() { return this; }`)
**Alternatives:**
- Shadow DOM + slots — style encapsulation for dock chrome, but adds deferred-render coordination protocol between Lit component and runtime with no concrete benefit (runtime already uses light DOM; pages-builder gets isolation from its own shadow root)
**Rationale:** Runtime querySelector works unchanged, deferred rendering is trivial (runtime renders into zone containers directly), one DOM tree with one event propagation path. No style encapsulation need — dock styles use CSS variables and inline styles.
**Trade-offs:** No style encapsulation for the dock component itself. Consumers who need isolation must provide it (pages-builder already does via its own shadow root).
**Sources:** packages/pages-primitives/src/dock/pages-dock-workbench.ts, packages/pages-runtime/src/site.ts:917-1015 (dock-toggle handler queries), packages/pages-builder/src/shell/builder-shell.ts:778-823 (consumer wraps in shadow root)
**Exploration:** quick
**Status:** captured

## D2: Persistence ownership

**Choice:** Lit component accepts optional `LayoutStore`, falls back to direct `localStorage`
**Alternatives:**
- Always delegate to runtime via events — fires `pages-dock-state-change`, runtime captures and persists. Breaks standalone use (no listener).
- Runtime remains sole persistence owner — Lit component is stateless layout, runtime pushes state down. Doesn't match Lit's reactive model.
**Rationale:** The Lit component is always the state owner for dock layout. Runtime provides a LayoutStore; standalone consumers don't. No dual state tracking — site.ts removes its dock-specific dockState Map for panels inside a `<pages-dock-workbench>`.
**Trade-offs:** The Lit component must understand LayoutStore's interface (save/load), creating a dependency on that type. Mitigated: LayoutStore is a simple interface in pages-component, not a runtime concern.
**Sources:** packages/pages-runtime/src/site.ts:1153-1188 (scheduleLayoutSave, captureLayout), packages/pages-primitives/src/dock/pages-dock-workbench.ts:34-62 (current localStorage persistence)
**Exploration:** quick
**Depends on:** D1 (light DOM — runtime can read dock state from DOM if needed)
**Status:** captured

## D3: Package placement and type dependencies

**Choice:** Move pure types down to pages-component, keep Lit component in pages-primitives
**Alternatives:**
- Move Lit component up to pages-runtime — no type moves needed, but runtime becomes a grab-bag of orchestration and UI components, violating the current separation
**Rationale:** `LayoutStore` is a pure interface over `LayoutState` (already in pages-component) — it belongs there, not in pages-runtime. `DockBarItem` is a pure data type. Moving them down keeps package boundaries clean. pages-primitives adds dependencies on pages-component (types) and pages-ui (DockWorkbenchConfig type).
**Trade-offs:** Two type relocations create import churn in pages-runtime (LayoutStore) and dock-bar-renderer.ts (DockBarItem). One-time cost, mechanical change.
**Sources:** packages/pages-component/src/model/types.ts:54 (LayoutState already here), packages/pages-runtime/src/layout-store.ts:3 (LayoutStore — pure interface), packages/pages-runtime/src/dock-bar-renderer.ts:4-9 (DockBarItem — pure data type), packages/pages-primitives/package.json (currently depends on pages-tsconfig only)
**Exploration:** quick
**Depends on:** D2 (Lit component needs LayoutStore interface)
**Status:** captured

## D4: Runtime ↔ Lit component integration for content rendering

**Choice:** Render callback injection — Lit component accepts optional `renderContent: (container: HTMLElement, panelConfig: DockPanelConfig) => void`
**Alternatives:**
- Event-based protocol — Lit component dispatches `pages-panel-render-request`, runtime listens and renders into the container. More decoupled but adds async coordination and a new event type to the contract.
**Rationale:** Direct function call is synchronous, no new event type, same pattern as activation callbacks and content factories (floating-workspace). Lit component doesn't import from pages-runtime — it calls whatever function it's given. For standalone use, no callback needed.
**Trade-offs:** Tighter coupling at the call site (runtime must set the property before the component renders). Acceptable — the activation callback already runs at element creation time, which is the natural place to set this.
**Sources:** packages/pages-runtime/src/activation.ts:563-587 (deferred activation pattern), content-agnostic-workbench protocol (content factory pattern)
**Exploration:** quick
**Depends on:** D1 (light DOM — rendered content is queryable), D3 (DockPanelConfig type available in pages-primitives)
**Status:** captured

## D5: Dock-toggle handling ownership

**Choice:** Lit component owns all dock-toggle handling — exclusive zone logic, cascade collapse/expand, deferred render trigger, button state sync
**Alternatives:**
- Lit component delegates toggle to runtime — fires `pages-dock-toggle` upward, runtime's existing handler does all work. Breaks standalone use (no handler, nothing happens).
**Rationale:** The Lit component encapsulates the behavior it renders. site.ts gets simpler (~100 lines of dock-specific code removed). Standalone use works without runtime. The component re-dispatches `pages-dock-toggle` with `composed: true` after handling so external listeners (URL sync, analytics) still observe the event.
**Trade-offs:** Duplicates cascade collapse/expand logic from site.ts into the Lit component during transition — but since site.ts's version is being removed, this is a move not a duplication. The Lit component becomes the single source of truth.
**Sources:** packages/pages-runtime/src/site.ts:917-1015 (dock-toggle handler being absorbed), packages/pages-runtime/src/site.ts:1289-1326 (initDockZoneGroup being absorbed)
**Exploration:** quick
**Depends on:** D1 (light DOM — cascade collapse walks DOM normally), D4 (render callback — deferred render trigger invokes callback)
**Status:** captured
