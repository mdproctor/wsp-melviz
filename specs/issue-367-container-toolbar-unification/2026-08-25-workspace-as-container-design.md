# Workspace Centre as Root Container

**Issue:** #369 (with #367, #368 prerequisites landed)
**Goal:** Make the floating workspace centre a regular Container with `layout: "free"` so the recursive container model applies uniformly from root to leaf. Eliminate dual-state bookkeeping between engine, backend, and strategy.

## Architecture — Before and After

### Before (current)

```
wireFloatingWorkspace (orchestrator, 360 lines)
  ├── FloatingFrameEngine (data model — positions, sizes, z-order, tabs)
  ├── FloatingFrameBackend (DOM + DnD, 1094 lines)
  │    └── per-frame Container → LayoutStrategy
  └── ContainerToolbar (workspace-level, separate from containers)
```

Three systems managing overlapping state. Mode transitions destroy one rendering system and rebuild another.

### After

```
wireFloatingWorkspace (thin initializer, ~80 lines)
  └── rootContainer (Container, layout: "free")
       ├── entry A → childContainer (frame A's container tree)
       ├── entry B → childContainer (frame B's container tree)
       └── ...
```

One system. Mode transitions are `rootContainer.setLayout()`. The toolbar is the container's standard toolbar. DnD lives in the free-layout strategy.

## Layer Model

| Layer | Responsibility | State |
|-------|---------------|-------|
| **Container** | Entry lifecycle, layout delegation, toolbar, state capture | entries array |
| **LayoutStrategy** | Rendering, interaction, layout-specific behavior | strategy closure |
| **Shared primitives** | Stateless DOM creation, pure math | none |
| **Facade** (temporary) | Backward compat for FloatingFrameBackend API | delegates to root Container |

### New module: `layout-math.ts`

Pure functions extracted from FloatingFrameEngine:
- `computeZonePreset(preset, entryCount, canvasSize)` — side-by-side, stacked, grid, main+sidebar, focus
- `scaleProportionally(entries, oldSize, newSize)` — proportional resize on container resize
- `equalGrid(count, canvasSize)` — tile arrangement

No DOM, no state. Testable without JSDOM. The 58 engine tests migrate to pure function tests here plus strategy-level DOM tests.

## Free-Layout Strategy Changes

The free-layout strategy grows to absorb all frame interaction:

### Current capabilities (keep)
- Positioned frame rendering via `injectFrameChrome`
- Entry state (position, size, pin, z-order)
- Resize via ResizeObserver
- Zone picker (snap to grid)
- `arrange()` method (added in #367)

### New capabilities (absorb from backend)
- **Cross-entry DnD** — drag state machine detecting targets across sibling entries
- **Edge splits** — drag to edge creates split container via `refreshEntry`
- **Tab drag between entries** — move tabs from one entry's child container to another
- **Cross-entry drop** — drop tab onto another entry's tab strip
- **Edge split preview** — visual overlay showing where the split will land

### DnD Architecture

```
Child tabbed strategy
  → fires pages-tab-drag-start (DOM event, bubbles)        [D6]

Parent free-layout strategy
  → listens on host element
  → enters drag mode (ghost element, pointer capture)
  → pointermove: detect target entry from entryState positions
    → over tab strip: show cross-entry drop preview
    → over edge zone: show edge split preview
  → pointerup: execute drop
    → cross-entry: remove from source, add to target
    → edge split: refreshEntry with new split container
```

All spatial detection uses `entryState` (positions, sizes) already in the strategy. No external coordinator needed.

### Tab drag out (entry creation)

When a tab is dragged outside all entries, the free-layout strategy creates a new entry at the drop position:
```
const newEntry = { key: generateKey(), label: tab.label, childContainer: newLeafContainer };
container.addEntry(newEntry);
```

This replaces the engine's `createFrame` + backend's `renderFrame`.

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
    positions: { preview: { x: 50, y: 50 }, outline: { x: 500, y: 80 } },
    sizes: { preview: { width: 400, height: 280 }, outline: { width: 320, height: 240 } },
    zOrder: ["preview", "outline"],
    pinned: ["outline"],
  }
}
```

### Migration

`migrateFrameLayout(frames: FrameLayout[]): ContainerState` — one-time conversion at load time in wireFloatingWorkspace. Pure function, testable. Called when saved layout is `FrameLayout[]` format (detected by array shape). New saves always write `ContainerState`.

## FloatingFrameBackend Facade

Thin adapter implementing the existing interface by delegating to the root Container:

| Backend method | Delegates to |
|---------------|-------------|
| `renderFrame(layout)` | `rootContainer.addEntry(...)` with restored child container |
| `removeFrame(key)` | `rootContainer.removeEntry(key)` |
| `addTab(frameKey, tab)` | find child container, `container.addEntry(...)` |
| `removeTab(frameKey, tabKey)` | find child container, `container.removeEntry(tabKey)` |
| `getRootContainer(key)` | `rootContainer.entries.find(e => e.key === key)?.childContainer` |
| `captureContainerTree(key)` | `captureContainerState(childContainer)` |
| `getFrameElement(key)` | query DOM by `data-frame-key` |
| `dispose()` | `rootContainer.dispose()` |

Deprecated via #370. Consumers migrate to Container API directly.

## wireFloatingWorkspace (~80 lines)

```
function wireFloatingWorkspace(container, savedLayout?, options?):
  1. Convert savedLayout if FrameLayout[] format (migration)
  2. Create root Container with layout "free", restored from ContainerState
  3. Mount root Container into container element
  4. Wire external custom events (pages-frame-move, etc.) via DOM listener on container
  5. Wire detach handler
  6. Return WireHandle { rootContainer, dispose }
```

## What Gets Deleted

| Module | Lines | Reason |
|--------|-------|--------|
| `FloatingFrameEngine` | ~400 | Replaced by Container + layout-math.ts |
| `ZoneLayoutEngine` | ~200 | Absorbed into layout-math.ts |
| `group-organiser-backend.ts` DnD code | ~220 | Absorbed into free-layout strategy |
| `group-organiser-backend.ts` frame rendering | ~200 | Already shared via frame-shell.ts |
| `wire-floating-workspace.ts` mode transitions | ~150 | `setLayout()` handles this |
| `wire-floating-workspace.ts` event wiring | ~100 | Simplified to external event dispatch |

Estimated net: ~1200 lines deleted, ~400 lines added (layout-math + strategy DnD + facade + migration).

## Risk Mitigation

- **Regression risk:** 1003 existing tests. Add integration tests for DnD in free-layout strategy. Migration function gets round-trip tests.
- **Strategy size:** free-layout-strategy.ts grows from 364 to ~600 lines. If too large, DnD handler extracts as `free-layout-dnd.ts` (internal module, not a public boundary).
- **Edge cases:** Drag from nested free-layout into parent free-layout — the DOM event bubbles to the nearest free-layout ancestor. Each strategy only handles drags within its own scope.

## References

- D1-D8 decisions: `specs/issue-367-container-toolbar-unification/decisions.md`
- #345 recursive container model (Entry.childContainer, refreshEntry, surgical replant)
- #366 split divider resize
- #367 arrange button at all depths
- #368 toolbar position
- #370 facade deprecation
- `container-tree-ops.ts` — tree traversal via Entry.childContainer
- `frame-shell.ts` — shared frame rendering primitives
