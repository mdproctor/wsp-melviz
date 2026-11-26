# Decisions — #749 Migrate playbook-split-view to pages-three-way-viewer

## D1: Where does the line↔step-address translation live?

**Choice:** PagesStepViewer implements ViewerSyncTarget directly
**Alternatives:**
- Wrapper element in slot — adds an extra custom element just for a 3-line translation; over-engineering for purity
- External translation in PlaybookSplitView — cursor sync only works when PlaybookSplitView is the consumer; every other consumer must reimplement the same wiring
**Rationale:** `positionInfos` is metadata about the steps the viewer is already rendering — cohesive, not a foreign concern. PagesStepViewer already imports from yaml-core. Adding `StepLineInfo[]` and a sync method makes the component self-sufficient in any viewer context, fulfilling the ViewerSyncTarget protocol that ThreeWayViewer's slot mechanism expects.
**Trade-offs:** PagesStepViewer gains a dependency on StepPositionScanner (already transitively available via yaml-core). Components that don't need cursor sync simply don't set positionInfos.
**Sources:** packages/pages-three-way-viewer/src/sync-protocol.ts, packages/pages-three-way-viewer/src/three-way-viewer.ts:149-153, packages/pages-step-viewer/src/playbook-split-view.ts:111-119
**Exploration:** quick
**Status:** captured
