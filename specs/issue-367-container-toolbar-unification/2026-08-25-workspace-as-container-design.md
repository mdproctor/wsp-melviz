# Workspace Centre as Root Container

**Issue:** #369 (with #367, #368 prerequisites landed)
**Goal:** Make the floating workspace centre a regular Container with `layout: "free"` so the recursive container model applies uniformly from root to leaf. Eliminate dual-state bookkeeping between engine, backend, and strategy.

## Architecture — Before and After

### Before (current)

```
wireFloatingWorkspace (orchestrator, 362 lines)
  ├── FloatingFrameEngine (data model — positions, sizes, z-order, tabs, 334 lines)
  ├── FloatingFrameBackend (DOM + DnD, 1095 lines)
  │    └── per-frame Container → LayoutStrategy
  └── ContainerToolbar (workspace-level, separate from containers)
```

Three systems managing overlapping state. Mode transitions destroy one rendering system and rebuild another.

Note: The #345 unified architecture spec proposed decomposing `group-organiser-backend.ts` into `frame-renderer.ts`, `container-tree-ops.ts`, and `dnd-coordinator.ts`. Only `container-tree-ops.ts` (141 lines, tree traversal + serialization) and `frame-state.ts` (10 lines, type definition) were extracted. The migration path is from the current monolithic backend (1095 lines), not from the decomposed modules that #345 envisioned.

### After

```
wireFloatingWorkspace (thin initializer, ~80 lines)
  └── rootContainer (Container, layout: "free")
       ├── entry A → childContainer (frame A's container tree)
       ├── entry B → childContainer (frame B's container tree)
       └── ...
```

One system. Mode transitions are `rootContainer.setLayout()`. DnD lives in the free-layout strategy. The toolbar is the container's standard `ContainerToolbar`, configured with workspace-appropriate callbacks:
- `onAdd` → creates a new entry in the free-layout strategy at a default position
- `onArrange` → dispatches to `layout-math.ts` presets via the strategy's `arrange(preset)` method
- `onLayoutChange` → switches the root container's layout

## Layer Model

| Layer | Responsibility | State |
|-------|---------------|-------|
| **Container** | Entry lifecycle, layout delegation, toolbar, state capture | entries array |
| **LayoutStrategy** | Rendering, interaction, layout-specific behavior | strategy closure |
| **Shared primitives** | Stateless DOM creation, pure math | none |

SETTLED: No facade layer — consumers migrate to the Container API directly (revised from D2)

### New module: `layout-math.ts`

Pure functions extracted from `frame-organisers.ts` (73 lines) and the `FloatingFrameEngine` resize logic:
- `computeZonePreset(preset, entryCount, canvasSize)` — dispatches to side-by-side, stacked, grid, main+sidebar, focus (replaces `applyPreset()` from `frame-organisers.ts`)
- `scaleProportionally(entries, oldSize, newSize)` — proportional resize on container resize (extracted from `handleResize` logic duplicated in both `group-organiser-backend.ts` and `free-layout-strategy.ts`)
- `equalGrid(count, canvasSize)` — tile arrangement (grid-only convenience, wraps `computeZonePreset("grid", ...)`)

All functions operate on `FreeLayoutEntry` records (position + size) — same type the strategy already uses internally. No DOM, no state. Testable without JSDOM.

The free-layout strategy exposes `arrange(preset: Preset, canvasSize?: { width; height })` which delegates to `computeZonePreset` and applies the resulting positions/sizes to all entries. This replaces both `tileArrange()` (current grid-only method) and the engine's `applyOrganiser()`.

The 58 engine tests migrate to pure function tests here plus strategy-level DOM tests.

## Free-Layout Strategy Changes

The free-layout strategy grows to absorb all frame interaction:

### Current capabilities (keep)
- Positioned frame rendering via `injectFrameChrome`
- Entry state (position, size, pin, z-order)
- Resize via ResizeObserver
- Zone picker (snap to grid)
- `tileArrange()` (grid only — extended to full `arrange(preset)` covering all 5 presets)

### New capabilities (absorb from backend)
- **Cross-entry DnD** — drag state machine detecting targets across sibling entries
- **Edge splits** — drag to edge creates split container via `refreshEntry`
- **Tab drag between entries** — move tabs from one entry's child container to another
- **Cross-entry drop** — drop tab onto another entry's tab strip
- **Edge split preview** — visual overlay showing where the split will land
- **`detachEntry(key): Entry | null`** — removes an entry from the strategy WITHOUT disposing its content. Unmounts `childContainer` (detaches from DOM) but does not dispose it. Does not call `entry.contentDispose`. Returns the live entry for transfer to another container. This is the safe primitive for DnD entry transfer — `removeEntry` calls `contentDispose` which destroys child containers (verified: `wrapContentFactory` sets `dispose: () => { child.dispose(); }` for entries with `childContainer`).
- **`hideEntry(key)` / `showEntry(key)`** — entry visibility for detach support. Added as optional methods on the `LayoutStrategy` interface so the detach handler can call `rootContainer.organiser.hideEntry(key)` without type-casting. Each strategy implements meaningfully: free-layout removes/re-adds the frame element from DOM (preserving entryState); tabbed removes/re-adds the tab button; accordion collapses and locks the section.

### DnD Architecture

```
Child tabbed strategy
  → fires pages-tab-drag-start (DOM event, bubbles)        [D6]

Parent free-layout strategy
  → listens on host element
  → event.stopPropagation() — claims ownership              [scoping]
  → enters drag mode (ghost element, pointer capture)
  → pointermove: detect target entry from entryState positions
    → over tab strip: show cross-entry drop preview
    → over edge zone: show edge split preview
    → outside all entries but inside host: show new-entry preview
    → outside host bounds entirely: fire pages-tab-escaped   [depth escape]
  → pointerup: execute drop
    → cross-entry: detachEntry from source, addEntry to target
    → edge split: detachEntry + childContainerFactory + refreshEntry
    → new entry: detachEntry + childContainerFactory at drop position
```

All spatial detection uses `entryState` (positions, sizes) already in the strategy. No external coordinator needed.

#### Scoping mechanism for nested free-layout containers

When free-layout containers nest (D4), each strategy only handles drags from its own scope. The mechanism:

1. **Claim on intercept:** The innermost free-layout strategy receives `pages-tab-drag-start`, calls `event.stopPropagation()`, and enters drag mode. No ancestor free-layout strategy sees the event.

2. **Depth escape:** If the pointer exits the inner strategy's host bounds entirely during drag, the inner strategy:
   - Calls `detachEntry(tabKey)` on the source container — removes the entry WITHOUT disposing its content (child containers survive unmounted; leaf entries retain their `component` descriptor)
   - Fires `pages-tab-escaped` (DOM event, bubbles) with `{ entry }` carrying the live detached entry on its host element
   - Exits drag mode

3. **Parent pickup:** The parent free-layout strategy listens for `pages-tab-escaped` on its host. On receipt, it calls `event.stopPropagation()` (prevents further ancestors from also claiming the escaped tab at 3+ nesting levels), then enters drag mode for the escaped tab — the user continues dragging seamlessly into the parent's scope. If the tab escapes the parent too, the parent fires its own `pages-tab-escaped`, and the grandparent picks it up via the same chain.

This gives clean ownership transfer without race conditions. Each strategy handles exactly one drag at a time. Both `pages-tab-drag-start` and `pages-tab-escaped` handlers call `stopPropagation()` to ensure exactly one strategy claims each event. The `pages-tab-escaped` event is analogous to the existing `pages-tab-drag-out` mechanism in `wireFloatingWorkspace` — both handle "tab leaves its container's scope."

Note: In the depth escape scenario, the dragged item is always a leaf tab entry from a child tabbed strategy — it has `component` but not `childContainer`. The `detachEntry` primitive is used for safety and correctness (the #345 surgical replant invariant: "the container tree is never destroyed and recreated"), even though the disposal bug in `removeEntry` would only manifest if an entry with `childContainer` were transferred.

### Tab drag out (entry creation)

When a tab is dragged outside all entries but inside the host, the free-layout strategy creates a new entry at the drop position. Creating a leaf Container requires configuration (ContentFactory, LayoutCallbacks, ContainerPolicy, depth) that the strategy doesn't own — this is provided via a `childContainerFactory` callback:

```
childContainerFactory: (entries: Entry[]) => Container
```

Provided by `wireFloatingWorkspace` during root Container configuration, capturing the correct content factory, policy, and callbacks in its closure. The free-layout strategy's DnD handler calls it during tab drag out and edge splits:

```
const detached = sourceContainer.detachEntry(tabKey);
const leafContainer = childContainerFactory([detached]);
const newEntry = { key: generateKey(), label: detached.label, childContainer: leafContainer };
container.addEntry(newEntry);
```

This replaces the engine's `createFrame` + backend's `renderFrame`. The `childContainerFactory` corresponds to the backend's current `createLeafContainer(frameKey, entries)` function, which captures the same configuration in its closure.

## Persistence

### Format: `ContainerState` (recursive)

```typescript
{
  layout: "free",
  tabs: [
    { key: "preview", label: "Preview", content: null,
      children: { layout: "tabbed", tabs: [...] } },
    { key: "outline", label: "Outline", content: null,
      children: { layout: "tabbed", tabs: [...] } },
  ],
  layoutState: {
    entries: {
      preview: { position: { x: 50, y: 50 }, size: { width: 400, height: 280 } },
      outline: { position: { x: 500, y: 80 }, size: { width: 320, height: 240 } },
    },
    zOrder: ["preview", "outline"],
    pinned: ["outline"],
  }
}
```

The `layoutState` field uses the existing `FreeLayoutState` type (`entries: Record<string, FreeLayoutEntry>`, `zOrder: string[]`) extended with `pinned: string[]`. Position and size are bundled per-entry in `FreeLayoutEntry` — matching the current `free-layout-strategy.ts` `getState()` format. The `pinned` field surfaces the strategy's internal `pinnedKeys` set, requiring `getState()` and `restoreState()` updates.

### Migration

`migrateFrameLayout(frames: FrameLayout[]): ContainerState` — one-time conversion at load time in wireFloatingWorkspace. Pure function, testable. Called when saved layout is `FrameLayout[]` format (detected by array shape). New saves always write `ContainerState`.

## wireFloatingWorkspace (~80 lines)

```
function wireFloatingWorkspace(container, savedLayout?, options?):
  1. Convert savedLayout if FrameLayout[] format (migration)
  2. Create root Container with layout "free", restored from ContainerState
  3. Mount root Container into container element
  4. Wire external custom events (pages-frame-move, etc.) via DOM listener on container
  5. Wire detach handler (see §Detach handler migration below)
  6. Return WireHandle { rootContainer, dispose }
```

### Detach handler migration

The detach handler currently uses 4 engine methods. Migration mapping:

| Engine method | Container equivalent |
|---|---|
| `engine.frames.get(key)` | `rootContainer.entries.find(e => e.key === key)` + strategy `entryState` for position/size |
| `engine.hideFrame(key)` | `rootContainer.organiser.hideEntry(key)` — removes frame element from DOM, preserves entryState |
| `engine.showFrame(key)` | `rootContainer.organiser.showEntry(key)` — re-renders frame element from preserved entryState |
| `engine.setDetached(key, bool)` | Entry metadata flag on the entry object |

`hideEntry` and `showEntry` are optional methods on the `LayoutStrategy` interface (see §Free-Layout Strategy Changes). The detach handler calls them through `rootContainer.organiser` — no type-cast needed because the methods are on the base interface.

The detach handler signature changes from `createFrameDetachHandler(engine, container, contentFactory, signal)` to `createFrameDetachHandler(rootContainer, container, contentFactory, signal)`.

### Nested workspace arrangement

In the current model, `getNestedEngine` bridges nested workspace arrangement — when a frame contains a nested workspace, the parent's arrange button dispatches to the nested engine. This engine-specific plumbing is unnecessary in the new model.

With the recursive Container model, nested workspaces are just nested Containers with `layout: "free"`. Each Container's own toolbar dispatches arrange to its own strategy's `arrange(preset)` method. The `getNestedEngine` option and `workspace-content-lifecycle.ts` `ContentManager` are eliminated — their responsibilities dissolve with the recursive Container model.

### activation.ts migration

With the facade removed, `activation.ts` migrates from `handle.engine` to `handle.rootContainer`:

| Current (engine) | New (rootContainer) |
|---|---|
| `handle.engine.frames.get(key)` | `handle.rootContainer.entries.find(e => e.key === key)` |
| `handle.engine.removeTab(frame, tab, ...)` | `findContainerWithTab(rootContainer, tab).removeEntry(tab)` |
| `handle.engine.addTab(frame, tab, ...)` | target container `.addEntry(entry)` |
| `handle.engine.frames.values()` | `handle.rootContainer.entries` |
| `createFrameKeyboardHandler(handle.engine, ...)` | `createFrameKeyboardHandler(handle.rootContainer, ...)` |

The cross-frame drop handler in `activation.ts` (`backend.onCrossFrameDrop(...)`) becomes unnecessary — cross-entry DnD is handled internally by the free-layout strategy. The `ContentManager` (`workspace-content-lifecycle.ts`) is eliminated.

## What Gets Deleted

| Module | Lines | Reason |
|--------|-------|--------|
| `FloatingFrameEngine` | 334 | Replaced by Container + layout-math.ts |
| `frame-organisers.ts` | 73 | Absorbed into layout-math.ts |
| `group-organiser-backend.ts` DnD code | ~220 | Absorbed into free-layout strategy |
| `group-organiser-backend.ts` frame rendering | ~200 | Already shared via frame-shell.ts |
| `wire-floating-workspace.ts` mode transitions | ~150 | `setLayout()` handles this |
| `wire-floating-workspace.ts` event wiring | ~100 | Simplified to external event dispatch |
| `workspace-content-lifecycle.ts` | 80 | Responsibilities dissolve with recursive Container model |

Estimated net: ~1160 lines deleted, ~350 lines added (layout-math + strategy DnD + migration + activation.ts migration).

## Risk Mitigation

- **Regression risk:** 1003 existing tests. Add integration tests for DnD in free-layout strategy. Migration function gets round-trip tests.
- **Strategy size:** free-layout-strategy.ts grows from 361 to ~600 lines. If too large, DnD handler extracts as `free-layout-dnd.ts` (internal module, not a public boundary).
- **Nested free-layout DnD:** Scoped via `stopPropagation()` + `pages-tab-escaped` event for depth escape. Each strategy handles exactly one drag at a time — no race conditions.

## References

- D1-D8 decisions: `specs/issue-367-container-toolbar-unification/decisions.md` (D2 revised: facade removed in favor of direct migration)
- #345 recursive container model (Entry.childContainer, refreshEntry, surgical replant)
- #366 split divider resize
- #367 arrange button at all depths
- #368 toolbar position
- `container-tree-ops.ts` — tree traversal via Entry.childContainer
- `frame-shell.ts` — shared frame rendering primitives
