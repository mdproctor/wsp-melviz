# Dock Workbench — IntelliJ-style Dock Layout Compositor

**Issue:** casehubio/casehub-pages#285
**Date:** 2026-08-04
**Status:** Draft

## Context

Pages has composable workbench primitives — `split`, `dockBar`, `hostPanel`,
`DetachController`, `LayoutState`/`LayoutStore` — all implemented and working
independently. Today a workbench is assembled by manually nesting these in a
Component tree. Issue #285 asks for an orchestrating component that handles
this composition automatically: centre content with collapsible, resizable dock
panels on left, right, and bottom edges.

### Prior Art

- **Workbench primitives spec** (2026-06-30): designed and implemented split,
  dockBar, hostPanel as composable primitives for #64/#65/#66.
- **Layout serialization** (#76): LayoutStore, LayoutState, localStorage and
  REST persistence — closed.
- **Floating/popout panels** (#77): DetachController — closed.
- **Drag-and-drop rearrangement** (#75): separate future epic — open, not in scope.

### Garden Context

- **GE-20260630-b8e2d8**: CSS Grid `fr` tracks don't collapse on `display:none`
  — use flex for toggleable panels. The existing `split` already uses flex.
- **GE-20260522-676291**: ResizeObserver triggers without innerHTML reset cause
  nested annotations in split-pane layouts.

### Protocol Constraints

- **PP-20260705-c7687d**: All Web Components use Lit. New components must extend
  `PagesElement`, `PagesContentElement`, or `LitElement`.
- **PP-20260705-bac842**: `pages-dock-toggle` and `pages-split-resize` are
  reserved framework events. Used for dock bar toggles and split handle drags.

## Architectural Approach

**Three focused primitives + a DSL builder.** The dock workbench is a specific
arrangement of existing things, not a new kind of thing. The missing
capabilities decompose into three small, independently useful primitives. A
builder composes them into the right tree structure.

No new rendering path. No monolithic controller. The standard
`renderComponent()` → activation callback pipeline processes everything.

## New Primitives

### 1. `exclusive` prop on dockBar

The existing dock-bar activation code (activation.ts) creates buttons that
independently toggle panels via `pages-dock-toggle` events. Add an `exclusive`
boolean prop for zone-exclusive behavior.

When `exclusive: true`:

- **Switch panels**: clicking button B while button A is active dispatches two
  synchronous `pages-dock-toggle` events: `{panelId: A, visible: false}` then
  `{panelId: B, visible: true}`. Both events process synchronously within the
  same click handler — the browser batches DOM changes into one repaint.
- **Close zone**: clicking the already-active button dispatches
  `{panelId: A, visible: false}`. The cascading collapse (primitive 3) handles
  collapsing the zone container.
- Non-exclusive dock bars (`exclusive: false` or absent) work exactly as today.

DSL:
```typescript
dockBar("vertical", items, { exclusive: true })
```

Implementation: ~15 lines of change in the dock-bar activation block in
`activation.ts`. The click handler checks `exclusive` prop, tracks
`data-active`, and dispatches the appropriate events.

### 2. `lazy` component type

A new non-layout component type that defers rendering of its children until
explicitly triggered. Children render on first open, then stay in the DOM for
subsequent show/hide — no re-rendering.

**Activation callback** (activation.ts):

```typescript
if (component.type === "lazy") {
  const children = component.slots?.default ?? [];
  el.dataset.lazy = "pending";
  el.addEventListener("pages-lazy-render", () => {
    for (const child of children) {
      renderComponent(el, child, {
        permissions: options?.permissions ?? ALLOW_ALL,
        onNode: callback,
      });
    }
    delete el.dataset.lazy;
  }, { once: true });
  return;
}
```

Since `lazy` is NOT in `LAYOUT_TYPES`, `renderComponent()` creates the
container div but does not recurse into children. The activation callback has
full control — same pattern as `lazy-page` (activation.ts line 569).

**Trigger integration** (site.ts dock-toggle handler): after deciding to show a
panel, check the slot container for `[data-lazy="pending"]` and dispatch
`pages-lazy-render` before setting display. Rendering is synchronous — no flash
of empty content.

```typescript
// In dock-toggle handler, before showing:
const lazyEl = slotContainer.querySelector('[data-lazy="pending"]');
if (lazyEl) {
  lazyEl.dispatchEvent(new Event("pages-lazy-render"));
}
```

**DSL builder:**

```typescript
function lazy(child: Component): Component {
  return { type: "lazy", slots: { default: [child] } };
}
```

The `lazy` primitive is independently useful — it can wrap tab panels,
accordion sections, or any component whose rendering should be deferred.

### 3. Cascading collapse/expand

Replace the existing split-specific collapse logic in site.ts (lines 856-884)
with a general-purpose cascade.

**On hide**: after hiding a slot container, walk up. At each parent, check if
all direct `[data-slot]` children are hidden. If so, hide that parent's own
`[data-slot]` ancestor. Continue until no more collapse is needed. Also
hide/show adjacent split drag handles at each level.

**On show**: before showing a slot container, walk up. If any ancestor slot
container is hidden, show it first (bottom-up). This ensures the zone container
is visible before the panel itself is shown.

This replaces ~30 lines of split-specific code with ~25 lines of general-purpose
cascading. It handles zone containers (`rows`), nested splits, and any future
nesting pattern.

**Why this matters for the workbench**: when the last panel in a zone is hidden
(zone closes), the zone's `rows` container detects all children hidden →
collapses → the parent split redistributes space to the centre. When a panel
reopens, the cascade expands the zone container first, then shows the panel.

## Builder API

### `dockWorkbench()` DSL builder

Lives in `pages-ui` alongside existing builders.

```typescript
interface DockPanelConfig {
  readonly key: string;
  readonly label: string;
  readonly icon: string;
  readonly defaultOpen?: boolean;
  readonly content: Component;
  readonly minSize?: number;
}

interface DockWorkbenchConfig {
  readonly storageKey?: string;
  readonly centre: Component | Component[];
  readonly left?: readonly DockPanelConfig[];
  readonly right?: readonly DockPanelConfig[];
  readonly bottom?: readonly DockPanelConfig[];
}

function dockWorkbench(config: DockWorkbenchConfig): Component
```

No top zone in v1. Adding it later is a non-breaking change — the builder gains
a `top` field.

### Generated tree structure

The builder produces a Component tree of existing types:

```
rows(
  columns(
    dockBar("vertical", leftItems, { exclusive: true }),
    split("horizontal", [
      rows(                                     ← left zone container
        withId("inbox", lazy(hostPanel(...))),
        withId("cases", lazy(hostPanel(...))),
      ),
      split("vertical", [                       ← centre + bottom
        centreContent,                          ← always visible
        rows(                                   ← bottom zone container
          withId("chat", lazy(hostPanel(...))),
        ),
      ]),
      rows(                                     ← right zone container
        withId("family", lazy(hostPanel(...))),
      ),
    ]),
    dockBar("vertical", rightItems, { exclusive: true }),
  ),
  dockBar("horizontal", bottomItems, { exclusive: true }),
)
```

**Layout geometry** (matches IntelliJ):
- Left/right panels span full height between top and bottom bars
- Bottom panel spans the width between left and right panels
- Centre fills all remaining space
- Dock bar strips are outside splits — fixed width, not resizable

**Zone omission**: if a zone has no panels, the builder omits that zone's
`rows()` container, dock bar, and simplifies the split. A config with only
`left` and `centre` produces a simpler tree.

### `defaultOpen` handling

Panels with `defaultOpen: true` are rendered eagerly — the builder does NOT
wrap them in `lazy()`. Only initially-closed panels are lazy-wrapped. This
avoids a timing issue: dock-bar activation fires during `renderComponent()` (via
`onNode`), but the dock-toggle handler in `site.ts` is registered AFTER
rendering. Events dispatched during activation would be missed.

```typescript
// Builder logic per panel:
const wrapped = panel.defaultOpen
  ? withId(panel.key, panel.content)        // render eagerly
  : withId(panel.key, lazy(panel.content))  // render on first open
```

The builder also encodes `defaultOpen` into the dock bar's `DockItem.defaultOpen`
prop so dock-bar activation sets the correct initial `data-active` state.

**State restore (subsequent visits)**: saved state may differ from defaults —
a panel that was not `defaultOpen` may have been opened by the user and saved.
After `loadSite()` restores dock visibility from `LayoutStore`, a post-render
init step walks the restored dock state. For each panel marked visible in saved
state: if its slot container has `[data-lazy="pending"]`, trigger
`pages-lazy-render` and show the slot container. This ensures saved-state
panels render even if they were not `defaultOpen`.

### YAML desugaring

```yaml
type: dock-workbench
storageKey: life-dashboard
centre:
  - type: host-panel
    props: { typeName: main-view }
left:
  - key: inbox
    label: Inbox
    icon: inbox
    defaultOpen: true
    content:
      type: host-panel
      props: { typeName: work-item-inbox, panelProps: { compact: true } }
  - key: cases
    label: Cases
    icon: cases
    content:
      type: host-panel
      props: { typeName: grouped-data-view }
bottom:
  - key: chat
    label: Chat
    icon: chat
    content:
      type: host-panel
      props: { typeName: conversational-pane }
```

The YAML desugarer in `pages-ui` recognizes `type: dock-workbench` and calls
the builder. Same pattern as other DSL types.

## State Persistence

### What's persisted

The workbench state maps onto the existing `LayoutState` shape — no schema
changes:

| State | LayoutState field | Example |
|-------|-------------------|---------|
| Active panel per zone | `docks: Record<string, boolean>` | `{ "inbox": true, "cases": false }` |
| Zone open/closed | Implicit — zone with no `true` panel is closed | |
| Split ratios | `splits: Record<string, number[]>` | `{ "split-0": [25, 50, 25] }` |

The `storageKey` maps to the `LayoutStore` key. `createLocalLayoutStore()`
(localStorage) works out of the box. REST-backed stores work too.

### Save/restore flow

**Save**: already wired. The dock-toggle handler calls `scheduleLayoutSave()`
after every toggle. The split-resize handler does the same.

**Restore**: on `loadSite()`, the runtime loads `LayoutState` from the store and
replays split ratios and dock visibility. Dock replay triggers lazy rendering
for panels that were open in the saved state.

**First visit** (no saved state): `defaultOpen` panels are shown. Subsequent
visits use saved state.

### Zone-exclusive state consistency

With `exclusive` dock bars, at most one panel per zone has `docks[key] = true`.
If persisted state has an inconsistency (two panels in the same zone marked
true), the dock bar's exclusive logic resolves it on first interaction. No
validation at restore time — the invariant is enforced by the UI.

## Package Placement

| Change | Package | Rationale |
|--------|---------|-----------|
| `exclusive` prop on dock-bar activation | `pages-runtime` | Dock-bar activation lives here |
| `lazy` component type activation | `pages-runtime` | All activation callbacks live here |
| Cascading collapse/expand | `pages-runtime` | Dock-toggle handler in `site.ts` |
| `lazy()` DSL builder | `pages-ui` | All DSL builders live here |
| `dockWorkbench()` DSL builder | `pages-ui` | All DSL builders live here |
| YAML desugaring for `dock-workbench` | `pages-ui` | YAML parsing lives here |

No new packages.

## Testing Strategy

### Unit Tests (Vitest)

- **Exclusive dock bar**: click button B while A active → dispatches hide-A
  then show-B events. Click active button → dispatches hide. Non-exclusive dock
  bar unchanged.
- **Lazy wrapper**: renderComponent with lazy type → children NOT rendered,
  container has `data-lazy="pending"`. Dispatch `pages-lazy-render` → children
  render, attribute removed. Second dispatch → no-op (one-shot listener).
- **Cascading collapse**: hide all slot children in a `rows` inside a split →
  rows' slot container collapses → split redistributes. Show one child →
  cascade expand restores all ancestors.
- **dockWorkbench builder**: verify generated tree structure — correct nesting,
  dock bar items match config, lazy wrappers present, IDs assigned.

### Integration Tests

- **Full workbench render**: `loadSite()` with `dockWorkbench()` output → dock
  bars rendered, centre visible, default-open panels rendered (lazy triggered),
  closed panels NOT rendered.
- **Zone switch**: click dock bar button → previous panel hidden, new panel
  shown (lazy rendered on first open).
- **Zone close/open**: click active button → zone collapses, centre absorbs
  space. Click button again → zone expands.
- **State persistence round-trip**: toggle panels, resize splits → dispose →
  reload with same storage key → state restored.
- **All zones closed**: close all panels → only centre and dock bars visible.

## v1 Scope Boundary

**In v1** (this issue):
- Three primitives (exclusive, lazy, cascading collapse)
- Builder with left/right/bottom zones
- State persistence via LayoutStore
- YAML desugaring

**Deferred** (separate issues):
- Top zone — non-breaking addition to builder config
- Keyboard shortcuts (Alt+1, Alt+2) — #285 advanced
- Detach integration — DetachController already works, needs builder wiring
- Drag to reposition panels between zones — #75
- Auto-collapse at breakpoint — #285 advanced
- Badge/notification on dock label — #285 advanced

## Design Constraints Verified

- **Foundation tier**: no casehub upstream dependencies introduced
- **Composable**: each primitive is independently useful beyond dock workbenches
- **No new rendering path**: standard `renderComponent()` pipeline throughout
- **Existing event contract**: uses `pages-dock-toggle` and `pages-split-resize`
  as documented in PP-20260705-bac842
- **Lit protocol**: no new Web Components in v1 — the builder produces a tree
  of existing component types. Future Lit wrapper (v2) would wrap the builder.
- **Backward compatible**: non-exclusive dock bars unchanged, existing split
  collapse behavior preserved (cascading is a superset)
- **Reserved event**: `pages-lazy-render` is a new framework-internal event.
  Update the reserved names table in PP-20260705-bac842 in the same commit
  that adds the lazy primitive (per protocol requirement).
