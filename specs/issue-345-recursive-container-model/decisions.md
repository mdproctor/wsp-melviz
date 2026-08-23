## D1: Nesting model — Entry owns children directly

**Choice:** `Entry` gains an optional `childContainer?: Container`. When set, the entry is a non-leaf that renders its child Container instead of raw content. The external `FrameState.childContainers: Map<string, Container>` is eliminated. The Container tree is navigable through Entry alone.
**Alternatives:**
- Dual model: Entry declares, external map backs — avoids touching split creation/collapse but creates two sources of truth
- Keep external map, extend to arbitrary entry keys — minimal type changes but doesn't achieve the "Entry as Container" model
**Rationale:** Matches the i3 spec's "everything is a Container" principle. Single source of truth. Tree structure is self-describing — no external registry needed to understand parent-child relationships. The existing `childContainers` map was a stopgap for the split-only case (#312).
**Trade-offs:** Requires refactoring all split creation/collapse logic in `group-organiser-backend.ts` to use `Entry.childContainer` instead of the map. Moderate diff but the split logic already creates child Containers — the change is where they're stored, not how they're created.
**Sources:** `packages/pages-runtime/src/frame-sandbox/types.ts` (Entry interface), `packages/pages-runtime/src/frame-sandbox/container.ts` (Container interface), `packages/pages-runtime/src/group-organiser-backend.ts` (FrameState.childContainers)
**Exploration:** quick
**Status:** captured

## D2: Nesting trigger — content-area + button

**Choice:** A `+` button inside the active tab's content area converts a leaf entry into a non-leaf. The tab-strip `+` remains "add sibling tab." Clicking the content-area `+` wraps the existing content into the first child of a new child Container, then adds a second empty child — the user sees their original content plus a new tab inside the nested Container.
**Alternatives:**
- Right-click context menu on tab header — more discoverable options but requires context menu infrastructure not yet built
- Tab-strip dropdown splitting "add tab" vs "add nested tab" — adds a click to the common case (sibling add)
**Rationale:** Direct, single-click gesture. Content-area placement makes the spatial relationship clear: "I'm adding inside this tab." Consistent with the issue's design direction. Tab-strip + remains the fast path for the common case (sibling).
**Trade-offs:** Requires injecting a + button into each leaf tab's content area. The button must hide when the entry is already non-leaf (it already has a nested Container with its own tab strip). Must respect `maxDepth` — hidden when depth limit reached.
**Depends on:** D1 (Entry owns children — the + creates a childContainer on the active entry)
**Sources:** Issue #345 body (design direction section)
**Exploration:** quick
**Status:** captured

## D3: Content wrapping — existing content becomes first child

**Choice:** When a leaf entry converts to non-leaf, the existing `contentElement` is detached and re-mounted as the first entry in the new child Container. The user's view is preserved — their content is now inside a tab of the nested Container. A second empty tab is added so the nesting is immediately useful.
**Alternatives:**
- Replace with empty Container — simpler but destructive, user loses current content
- Keep as background, overlay Container — complex and confusing
**Rationale:** Non-destructive. The user clicked + to add something alongside their content, not to replace it. Content factory can re-render if needed, but preserving the existing DOM element avoids unnecessary re-renders.
**Trade-offs:** The content element's parent changes, which may affect CSS scoping or event listeners that assume a specific DOM ancestry. Mitigated by the content factory pattern — content is already designed to be mountable in any container.
**Depends on:** D1, D2
**Sources:** `packages/pages-runtime/src/frame-sandbox/container.ts` (contentFactory pattern, Entry.contentElement lifecycle)
**Exploration:** quick
**Status:** captured

## D4: Depth semantics — full tree depth, unified

**Choice:** `ContainerPolicy.maxDepth` counts from root Container to deepest leaf Container, regardless of whether nesting was created by splits or by entry nesting. The existing `depth` parameter on `createContainer()` already tracks this. Child Containers created via `Entry.childContainer` pass `parentContainer.depth + 1` as their depth.
**Alternatives:**
- Separate depth counters for split nesting vs entry nesting — more flexible but two concepts to reason about, harder to enforce a global limit
**Rationale:** Uniform depth model. The user doesn't care whether nesting came from a split or a tab — depth is depth. The existing `depth` field on Container and the `maxDepth` check in `createContainer()` already implement this correctly for splits; entry nesting plugs into the same mechanism.
**Trade-offs:** A maxDepth of 3 may feel restrictive when combining splits and entry nesting (root → split → entry-nest = depth 3, already at limit). May need to increase DEFAULT_POLICY.maxDepth. But this is a policy value, not an architectural constraint.
**Sources:** `packages/pages-runtime/src/frame-sandbox/types.ts:3-6` (ContainerPolicy), `packages/pages-runtime/src/frame-sandbox/container.ts:148-153` (depth check)
**Exploration:** quick
**Status:** captured

## D5: Persistence — recursive FrameTabConfig

**Choice:** `FrameTabConfig` gains an optional `children?: ContainerState` field. `ContainerState` contains `layout: Layout` and `tabs: FrameTabConfig[]` — the same recursive shape as the runtime Container tree. When `children` is present, the tab is non-leaf and its content is the serialized child Container. When absent, the tab is a leaf (backward compatible). `FrameLayout.tabs` stays as the top-level field.
**Alternatives:**
- Separate `containerTree?` field on FrameLayout replacing `tabs` — bigger migration, cleaner separation but more work for backward compat
- Rename `tabs` to `entries` with inline recursion — breaks backward compat for field name
**Rationale:** Minimal, backward-compatible extension. Existing layouts without `children` load correctly as flat tab lists. New layouts serialize the full tree. The recursion is natural — a tab's children are just more tabs in a Container.
**Trade-offs:** `FrameTabConfig` gains responsibility for tree structure. The type name ("TabConfig") is slightly misleading for non-leaf entries, but renaming it would break the existing API surface.
**Depends on:** D1 (Entry.childContainer defines the runtime tree that must be serialized)
**Sources:** `packages/pages-component/src/model/types.ts:91-111` (FrameLayout), workspace-compositor spec §8 (CompositorState pattern)
**Exploration:** quick
**Status:** captured

## D6: Collapse — auto-flatten on single child

**Choice:** When a nested Container's last sibling is closed and only one child remains, the remaining child's content replaces the Container — the entry reverts to a leaf. This is the same behavior as split collapse (where the sibling leaf becomes root when the other is emptied).
**Alternatives:**
- Stay nested, require manual flatten — more predictable but creates unnecessary single-child depth
- Auto-flatten only when empty (zero children) — inconsistent with split behavior
**Rationale:** Consistent with the existing split collapse pattern in `group-organiser-backend.ts:297-311`. Prevents orphaned single-child nesting that adds depth without utility. The user can always re-nest.
**Trade-offs:** Auto-flatten may surprise users who intentionally want a single-tab nested Container (e.g., for layout isolation). But this is an edge case — the common case is that single-child nesting is accidental cleanup residue.
**Depends on:** D1 (collapse operates on Entry.childContainer)
**Sources:** `packages/pages-runtime/src/group-organiser-backend.ts:297-311` (existing split collapse pattern)
**Exploration:** quick
**Status:** captured

## D7: Tree walking — unified traversal via Entry.childContainer

**Choice:** Refactor all tree-walking helpers (`findLeafContainer`, `findContainerWithTab`, `forEachLeafContainer`, `findParentSplitEntry`) to walk via `Entry.childContainer`. The `childMap: Map<string, Container>` parameter is removed from all signatures. Both split children and entry-nested children are reached through the same mechanism: iterate `container.entries`, check `entry.childContainer`, recurse.
**Alternatives:**
- Adapter pattern: build temporary childMap from Entry.childContainer fields — smaller diff but maintains two mental models and the adapter rebuilds on every mutation
- New helpers alongside old with gradual deprecation — safest migration but code duplication
**Rationale:** One traversal pattern for the entire tree. The helpers are already recursive — the change is parameter removal, not logic rewrite. With Entry owning children (D1), there's no reason for an external map parameter.
**Trade-offs:** All call sites in `group-organiser-backend.ts` must be updated (approximately 15 call sites). The refactor touches a large file but each change is mechanical — drop the map argument, change the child lookup from `childMap.get(entry.key)` to `entry.childContainer`.
**Depends on:** D1 (Entry.childContainer is the traversal mechanism)
**Sources:** `packages/pages-runtime/src/group-organiser-backend.ts:55-128` (existing helpers)
**Exploration:** quick
**Status:** captured
