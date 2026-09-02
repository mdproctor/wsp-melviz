## D1: Pill type shape

**Choice:** Inline literal type inside NodeDecoration
**Alternatives:**
- Named exported interface (`NodePill`) — enables independent typing but breaks consistency with existing badge/border/overlay pattern
**Rationale:** All existing decoration fields use inline readonly object literals. Consistency over ceremony.
**Trade-offs:** Consumers who need to type pill arrays independently must use `NodeDecoration['pills']`
**Sources:** packages/graph-core/src/model.ts:22 — existing NodeDecoration interface
**Exploration:** quick
**Status:** captured

## D2: Pill visual placement

**Choice:** Bottom edge of the node, inside the decoration wrapper
**Alternatives:**
- Below the badge (top-right area) — collides with badge, cluttered
- Below the node entirely (floating) — disconnected from node boundary, layout complications
**Rationale:** Bottom edge avoids badge collision, reads naturally as supplementary metadata, stays within node bounds
**Trade-offs:** Tall pill lists could extend beyond the node's visual boundary
**Sources:** packages/graph-renderer/src/stencil-wrapper.tsx:50-76 — existing DecorationBadge placement
**Exploration:** quick
**Status:** captured
