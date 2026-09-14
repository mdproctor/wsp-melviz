# Design Journal — issue-436-rework-dock-lit-wrap

## 2026-09-14 — Session 1: Design + Batches 1-2

### Key design insight
The user challenged the initial assumption that the Lit component and runtime dock-workbench are two separate use cases. First-principles analysis revealed that a light DOM Lit component can serve both paths — Shadow DOM was the only real barrier, and it isn't needed (CSS variables handle styling; pages-builder gets isolation from its own shadow root).

### Decisions (D1-D8)
- D1: Light DOM — no shadow boundary, querySelector works unchanged
- D2: Lit component owns persistence, accepts optional LayoutStore
- D3: Types move to pages-component, Lit component stays in pages-primitives
- D4: Render callback injection for content rendering
- D5: Lit component owns all dock-toggle handling
- D6: Config/standalone separation — activation.ts converts DockWorkbenchConfig
- D7: Builder returns opaque `{type: "dock-workbench"}`
- D8: CSS injection via `<style data-pages-dock>`

### Review findings (3 gaps fixed)
1. Standalone dock-bars break if handler fully removed → guard with `closest()` check
2. `syncUrl` loses dock state → `deriveDockState()` reads from Lit element
3. Drag rearrange path underspecified → handler stays in site.ts, updates Lit properties reactively

### Implementation progress
- **Batch 1 (Type foundation):** Complete. LayoutStore + dock config types moved to pages-component. DockBarItem/DockItem converged.
- **Batch 2 (Lit component):** Complete. Light DOM rewrite with dock bars, zone containers, resize handles, CSS injection. Toggle handling with zone-scoped exclusivity, cascade collapse/expand, deferred render. State initialization from LayoutStore/localStorage, debounced persistence.
- **Batch 3 (Runtime wiring):** Not started. 3 tasks remain: builder change, activation+site.ts, builder-shell migration.

### Hazard: IntelliJ MCP buffer ≠ disk
`ide_replace_text_in_file` modifies IntelliJ's buffer but may not flush to disk. Git commits read disk. Verify with Read tool after IntelliJ edits. IntelliJ also lost the project mid-session (switched to main), requiring manual branch recovery.
