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
