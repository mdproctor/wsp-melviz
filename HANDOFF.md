# HANDOFF — casehub-pages

Branch: `issue-345-recursive-container-model`
Issue: #345 — Recursive Container model — entries as nested Containers

## Last Session

Implemented the core recursive Container model (Entry.childContainer, containerizeEntry/flattenEntry, tree-walking refactor, persistence bridge). Then spent significant time fixing pre-existing floating workspace UX bugs exposed by testing the feature in the browser: frame resize on viewport change, toolbar position stability across mode cycles, workspace mode switching rendering real content instead of placeholder headings, and eliminating split-brain state between free-mode frames and workspace-mode containers. 20+ commits on the branch.

## Immediate Next Step

The workspace mode switching still has issues — after cycling through modes (free→tabbed→accordion→free), the Preview frame loses its tab strip toolbar actions (☰ +). The root cause: `injectStripActions` in `renderFrame` runs once at initial render but isn't re-invoked when frames are hidden/shown by `applyWorkspaceMode`. The fix needs `injectStripActions` to run after frames are shown again (in the `showFrames` path), or the strip actions need to be part of the Container's own lifecycle rather than externally injected DOM.

Broader remaining work on #345:
- Robust frame toolbar lifecycle across workspace mode switches
- TDD for mode cycling, state preservation, toolbar persistence
- `restoreLayout` path for containerTree (capture works, restore from persistence not wired)
- Nest button re-injection after mode cycle round-trips
- Frame Sandbox example fix (`free-layout` YAML alias not desugared to `free`)

## References

- Design spec: `docs/specs/issue-345-recursive-container-model/`
- Decisions: `docs/specs/issue-345-recursive-container-model/decisions.md`
- Plan: `plans/2026-08-23-recursive-container-model.md` (workspace)
- Key files changed: `packages/pages-runtime/src/frame-sandbox/container.ts`, `types.ts`, `group-organiser-backend.ts`, `wire-floating-workspace.ts`, `container-toolbar.ts`, `floating-frame-engine.ts`, `floating-frame-backend.ts`, `activation.ts`
