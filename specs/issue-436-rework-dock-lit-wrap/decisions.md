## D1: Light DOM vs Shadow DOM

**Choice:** Light DOM (`createRenderRoot() { return this; }`)
**Alternatives:**
- Shadow DOM + slots — style encapsulation for dock chrome, but adds deferred-render coordination protocol between Lit component and runtime with no concrete benefit (runtime already uses light DOM; pages-builder gets isolation from its own shadow root)
**Rationale:** Runtime querySelector works unchanged, deferred rendering is trivial (runtime renders into zone containers directly), one DOM tree with one event propagation path. No style encapsulation need — dock styles use CSS variables and inline styles.
**Trade-offs:** No style encapsulation for the dock component itself. Consumers who need isolation must provide it (pages-builder already does via its own shadow root).
**Sources:** packages/pages-primitives/src/dock/pages-dock-workbench.ts, packages/pages-runtime/src/site.ts:917-1015 (dock-toggle handler queries), packages/pages-builder/src/shell/builder-shell.ts:778-823 (consumer wraps in shadow root)
**Exploration:** quick
**Status:** captured
