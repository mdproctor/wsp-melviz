# Decisions — #750 Adopt ViewerController in playbook controller

## D1: How to adopt ViewerController for floating panel state

**Choice:** Create a sibling PanelController in pages-three-way-viewer
**Alternatives:**
- Generalize ViewerController into StateController<M> base — refactors working code for two consumers; premature abstraction
- Use ViewerController directly with mode mapping — semantic mismatch (yaml=closed, split=docked) makes code unreadable
**Rationale:** Same architectural pattern as ViewerController (constructor+options, state getters, transition methods, onChange callback) but with panel-specific semantics. PanelMode ('closed'|'docked'|'floating') is self-documenting. Additive change — no existing code modified.
**Trade-offs:** Two similar-shaped controller classes in the same package. If a third consumer of this pattern appears, consider extracting a generic base then.
**Sources:** packages/pages-three-way-viewer/src/viewer-controller.ts (ViewerController pattern), packages/pages-aria/src/controller/playbook-controller.ts:190-201,661-746 (current ad-hoc state)
**Exploration:** quick
**Status:** captured
