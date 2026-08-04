# Dock Workbench — IntelliJ-style Dock Layout Compositor

**Issue:** casehubio/casehub-pages#285
**Date:** 2026-08-04
**Status:** Revised after light design review

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

- `pages-dock-toggle` and `pages-split-resize` are reserved framework events
  (pages-event-contract protocol). Used for dock bar toggles and split handle
  drags.
- All Web Components use Lit (web-component-strategy protocol). No new Web
  Components in v1 — the builder produces a tree of existing component types.

## Architectural Approach

**Three focused primitives + a DSL builder.** The dock workbench is a specific
arrangement of existing things, not a new kind of thing. The missing
capabilities decompose into three small, independently useful primitives. A
builder composes them into the right tree structure.

No new rendering path. No monolithic controller. The standard
`renderComponent()` → activation callback pipeline processes everything.

## New Primitives

### 1. `exclusive` prop on dockBar

The existing dock-bar activation code (activation.ts:450-495) creates buttons
that independently toggle panels via `pages-dock-toggle` events. Add an
`exclusive` boolean prop for zone-exclusive behavior.

When `exclusive: true`:

- **Switch panels**: clicking button B while button A is active dispatches two
  synchronous `pages-dock-toggle` events: `{panelId: A, visible: false}` then
  `{panelId: B, visible: true}`. Both events process synchronously within the
  same click handler — the browser batches DOM changes into one repaint.
- **Close zone**: clicking the already-active button dispatches
  `{panelId: A, visible: false}`. The cascading collapse (primitive 3) handles
  collapsing the zone container.
- Non-exclusive dock bars (`exclusive: false` or absent) work exactly as today.

**Builder signature change.** The existing `dockBar()` builder accepts
`(orientation, items)`. Add an optional third argument for options:

```typescript
function dockBar(
  orientation: "vertical" | "horizontal",
  items: DockItem[],
  options?: { exclusive?: boolean },
): Component<"dock-bar">
```

The `exclusive` prop is stored in `component.props.exclusive` alongside the
existing `orientation` and `items` props. The dock-bar activation code reads
it from `component.props`.

**UX model.** The issue describes "multiple panels per zone" as "tabbed within
a dock zone." The exclusive dock-bar buttons ARE the tabs — clicking a button
shows that panel and hides the previous one, same as IntelliJ. There is no
separate tab bar inside the zone. This is a deliberate simplification: the dock
bar serves as both the zone toggle and the panel selector. If a future need
arises for a visible tab bar within a zone, it can be layered without changing
this primitive.

### 2. `deferred` component type

A new component type that defers rendering of its children until explicitly
triggered. Children render on first open, then stay in the DOM for subsequent
show/hide — no re-rendering.

**Naming.** Named `deferred`, not `lazy`, to avoid collision with the existing
`LAZY_TYPES` set in render.ts (line 12). `LAZY_TYPES` controls swap-based lazy
rendering for interactive types (tabs, carousel, etc.) where content is torn
down and re-rendered on swap. `deferred` has render-once-keep-forever semantics
— a distinct lifecycle category.

**Rendering mechanism.** Add `"deferred"` to the `LAZY_TYPES` set in render.ts.
This is the gating mechanism: `renderNode` (render.ts:107) checks
`LAZY_TYPES.has(component.type)` before recursing into children. When true, it
creates the slot container divs but does NOT render children into them. Instead,
it passes children to `wireInteractivity` via a `LazyConfig`.

For `deferred`, `wireInteractivity` has no matching case in its switch
statement — it returns without doing anything. The empty slot containers remain
in the DOM. The activation callback handles everything.

**Activation callback** (activation.ts):

```typescript
if (component.type === "deferred") {
  const children = component.slots?.default ?? [];
  el.dataset.deferred = "pending";
  el.addEventListener("pages-deferred-render", () => {
    for (const child of children) {
      renderComponent(el, child, {
        permissions: options?.permissions ?? ALLOW_ALL,
        onNode: callback,
      });
    }
    delete el.dataset.deferred;
  }, { once: true });
  return;
}
```

Note: `renderComponent` calls `target.innerHTML = ""` before rendering. For
single-child `deferred` wrappers, this clears the empty slot container created
by `renderNode` and replaces it with the child content. **Single-child
constraint**: the `deferred()` builder wraps exactly one child. Multi-child
deferred is not supported in v1 (each child would need separate
`renderNode` calls to avoid the innerHTML clear, which requires exporting
`renderNode` — a larger change).

**Trigger integration** (site.ts dock-toggle handler): when showing a panel,
check the element for `[data-deferred="pending"]` and dispatch
`pages-deferred-render` before setting display.

```typescript
// In dock-toggle handler, before showing:
if (panelEl.dataset.deferred === "pending") {
  panelEl.dispatchEvent(new Event("pages-deferred-render"));
}
```

**Data pipeline notification.** After deferred rendering, the activation
callback for the child component (e.g., `host-panel`) registers it in the
`ComponentRegistry` and dispatches `pages-data-request` if it has a lookup.
The data pipeline handles this normally — same as any component rendered via
`renderComponent`. No additional pipeline notification needed.

**DSL builder:**

```typescript
function deferred(child: Component): Component {
  return { type: "deferred", slots: { default: [child] } };
}
```

The `deferred` primitive is independently useful — it can wrap tab panels,
accordion sections, or any component whose rendering should be deferred. For
non-dock-toggle visibility changes (tabs, accordion), the visibility handler
in `wireInteractivity` would also need to check for `data-deferred` — this is
a follow-up enhancement, not required for the dock workbench where all deferred
panels are toggled via `pages-dock-toggle`.

**Reserved event.** `pages-deferred-render` is a new framework-internal event.
Update the reserved names table in the pages-event-contract protocol in the
same commit (per protocol requirement).

### 3. Dock-toggle handler: component-level targeting + cascading collapse

**Problem with current handler.** The existing dock-toggle handler (site.ts:839)
finds a panel by `[data-component-id]`, then walks up to the nearest
`[data-slot]` and hides/shows that slot container. This works for `split`
(each child has its own numbered slot) but fails for `rows` where all children
share a single `[data-slot="default"]` — hiding the slot hides ALL children.

**Fix: operate on component elements directly.** The handler hides/shows the
`[data-component-id]` element itself, then cascades collapse/expand through
parent containers.

**On hide:**

1. Hide the component element: `panelEl.style.display = "none"`
2. Walk up to the parent slot container
3. Check if all `[data-component-id]` children in the slot are hidden
4. If all hidden: hide the slot container, hide adjacent split drag handles
5. Walk up to the slot's parent container, check if all its `[data-slot]`
   children are hidden. If all hidden: hide the container's own parent slot.
6. Continue cascading until a visible sibling is found or the root is reached.

**On show:**

1. Walk up from the component element — if any ancestor slot container is
   hidden, show it first (bottom-up). Show adjacent split drag handles.
2. Check for `[data-deferred="pending"]` on the component — trigger render.
3. Show the component element.

**Backward compatibility.** For existing `split` layouts where each child has
its own slot, hiding the component (the only child in the slot) cascades to
hiding the slot. This produces the same result as the current handler. Existing
tests pass without modification.

## Builder API

### `dockWorkbench()` DSL builder

Lives in `pages-ui` alongside existing builders.

```typescript
interface DockPanelConfig {
  readonly key: string;         // unique ID — becomes data-component-id
  readonly label: string;       // dock bar button text
  readonly icon: string;        // dock bar button icon
  readonly defaultOpen?: boolean;
  readonly content: Component;  // panel content (any Component type)
  readonly minSize?: number;    // min width (left/right) or height (bottom) in px
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

**Key → panelId mapping.** `DockPanelConfig.key` serves as both
`DockItem.panelId` (on the dock bar button) and `component.id` (via
`withId(key, ...)`). The dock-toggle handler finds `[data-component-id="key"]`
— the same string the dock bar dispatches as `panelId`. No mapping layer.

**`minSize` mapping.** `DockPanelConfig.minSize` maps to the `split`'s
`minSizes` array. The builder positions each zone's minSize at the correct
index in the outer horizontal split's `minSizes`. Centre has no minSize
constraint (it fills remaining space). `maxSize` is deferred — not needed for
v1 (the split drag handles already constrain via adjacent panel minSizes).

No top zone in v1. Adding it later is a non-breaking change — the builder
gains a `top` field.

### Generated tree structure

The builder generates a `Component` tree of existing types. ALL dock panels
start with `style: { display: "none" }` — the post-render init step
(`applyDockState`, see State Initialization) determines which panels to show.

```
rows(
  columns(
    dockBar("vertical", leftItems, { exclusive: true }),
    split("horizontal", [
      rows(                                     ← left zone container
        withId("inbox", deferred(hostPanel(...))),  ← style: { display: "none" }
        withId("cases", deferred(hostPanel(...))),  ← style: { display: "none" }
      ),
      split("vertical", [                       ← centre + bottom
        centreContent,                          ← always visible
        rows(                                   ← bottom zone container
          withId("chat", deferred(hostPanel(...))),
        ),
      ]),
      rows(                                     ← right zone container
        withId("family", deferred(hostPanel(...))),
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

**Zone omission.** If a zone has no panels, the builder omits that zone's
`rows()` container, dock bar, and simplifies the split:

- **Left + right + bottom**: full tree as above (3-child outer split + inner
  vertical split)
- **Left + bottom only**: 2-child outer split (left zone + inner vertical),
  no right dock bar
- **Left only**: 2-child outer split (left zone + centre), no inner vertical
  split, no bottom dock bar
- **Centre only**: no outer split, no dock bars — just the centre content
- **Bottom only**: inner vertical split (centre + bottom zone), bottom dock bar

### State Initialization (`applyDockState`)

**Problem.** Three conditions must be established after rendering: (a) non-active
panels must be hidden, (b) active panels must be deferred-rendered and shown,
(c) `dockState` must be seeded for layout persistence.

The builder sets `style: { display: "none" }` on all panels, solving (a) at
render time. Conditions (b) and (c) require a post-render step because panel
elements must exist in the DOM before they can be shown.

**Timing.** In `loadSite()`, the dock-toggle listener is registered at line 834
and `renderComponent` runs at line 1017. Events dispatched after rendering ARE
received by the handler. Layout state is loaded at line 984-1001 (before
rendering), so `dockState` is pre-seeded from saved state by the time the
post-render step runs.

**`applyDockState` step** — runs after `renderComponent` and
`applySavedSplitRatios`, analogous to the existing post-render init pattern:

1. **Collect zone membership**: scan dock-bar elements for their panel IDs
   and `data-active` (defaultOpen) state. Group panels by their parent dock
   bar (each dock bar = one zone).

2. **Determine active panel per zone**: for each zone, check if `dockState`
   (pre-seeded from saved LayoutState) has a visible entry. If so, use it
   (saved state wins). If not, use the defaultOpen panel. If multiple panels
   in a zone are marked visible in saved state, use only the first
   (enforce exclusivity — prevents broken-on-load from corrupted state).

3. **Dispatch dock-toggle events**: for each active panel, dispatch
   `pages-dock-toggle { panelId, visible: true }` on the target container.
   The existing handler processes each event: triggers deferred render, shows
   the panel, cascades expand (ensures zone container is visible), seeds
   `dockState`.

4. **Sync dock-bar buttons**: after processing events, update dock-bar button
   `data-active` state to match actual visibility. This handles the case
   where saved state differs from `defaultOpen` — the buttons match the
   restored state, not the initial defaults.

5. **Seed inactive panels**: for each non-active panel, set
   `dockState.set(panelId, false)` so that `scheduleLayoutSave()` captures
   the full state on first save.

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

**Restore**: `loadSite()` loads `LayoutState` from the store (line 984-1001)
before rendering, pre-seeding `dockState` and `splitRatios`. After rendering,
`applySavedSplitRatios` applies split ratios to the DOM. Then `applyDockState`
(see State Initialization) uses the pre-seeded `dockState` to determine which
panels to show.

**First visit** (no saved state): `dockState` is empty. `applyDockState` falls
back to `defaultOpen`. Subsequent visits use saved state.

### Zone-exclusive state validation

With `exclusive` dock bars, at most one panel per zone has `docks[key] = true`.
`applyDockState` enforces this invariant at restore time: if persisted state has
multiple panels marked true in a zone, only the first is shown. This prevents
broken-on-load from corrupted state — users never see a visually broken layout.

## Package Placement

| Change | Package | Rationale |
|--------|---------|-----------|
| `exclusive` prop on dock-bar activation | `pages-runtime` | Dock-bar activation lives here |
| `deferred` type in `LAZY_TYPES` | `pages-component` | render.ts lives here |
| `deferred` activation callback | `pages-runtime` | All activation callbacks live here |
| Component-level dock-toggle + cascade | `pages-runtime` | Dock-toggle handler in `site.ts` |
| `applyDockState` post-render step | `pages-runtime` | `loadSite()` in `site.ts` |
| `deferred()` DSL builder | `pages-ui` | All DSL builders live here |
| `dockWorkbench()` DSL builder | `pages-ui` | All DSL builders live here |
| YAML desugaring for `dock-workbench` | `pages-ui` | YAML parsing lives here |

No new packages.

## Testing Strategy

### Unit Tests (Vitest)

- **Exclusive dock bar**: click button B while A active → dispatches hide-A
  then show-B events. Click active button → dispatches hide. Non-exclusive dock
  bar unchanged.
- **Deferred wrapper**: renderComponent with deferred type → children NOT
  rendered (LAZY_TYPES gate), container has `data-deferred="pending"`. Dispatch
  `pages-deferred-render` → children render, attribute removed. Second dispatch
  → no-op (one-shot listener).
- **Component-level dock-toggle**: hide a component in a shared slot (rows) →
  only that component hidden, siblings visible. Hide all siblings → slot
  collapses. Show one → slot expands.
- **Cascading collapse**: hide all components in a zone container inside a
  split → zone slot collapses → split redistributes. Show one component →
  cascade expand restores all ancestors.
- **dockWorkbench builder**: verify generated tree structure — correct nesting,
  dock bar items match config, `deferred` wrappers present, IDs assigned,
  all panels have `style.display = "none"`.
- **applyDockState**: defaultOpen panels shown, non-defaultOpen hidden. Saved
  state overrides defaults. Multiple panels marked true in same zone → only
  first shown (exclusivity enforcement).

### Integration Tests

- **Full workbench render**: `loadSite()` with `dockWorkbench()` output → dock
  bars rendered, centre visible, default-open panels rendered (deferred
  triggered by applyDockState), closed panels NOT rendered.
- **Zone switch**: click dock bar button → previous panel hidden, new panel
  shown (deferred rendered on first open).
- **Zone close/open**: click active button → zone collapses, centre absorbs
  space. Click button again → zone expands.
- **State persistence round-trip**: toggle panels, resize splits → dispose →
  reload with same storage key → state restored correctly.
- **All zones closed**: close all panels → only centre and dock bars visible.
- **Corrupted state recovery**: load with two panels marked true in same zone
  → only one shown, no visual glitch.

## v1 Scope Boundary

**In v1** (this issue):
- Three primitives (exclusive, deferred, component-level dock-toggle + cascade)
- Builder with left/right/bottom zones
- `applyDockState` post-render initialization
- State persistence via LayoutStore with exclusivity validation
- YAML desugaring

**Deferred** (separate issues):
- Top zone — non-breaking addition to builder config
- Keyboard shortcuts (Alt+1, Alt+2) — #285 advanced
- Detach integration — DetachController already works, needs builder wiring
- Drag to reposition panels between zones — #75
- Auto-collapse at breakpoint — #285 advanced
- Badge/notification on dock label — #285 advanced
- Animation (slide in/out) — #285 core requirement, deferred to follow-up
  (functional layout first, polish second)
- `deferred` trigger in `wireInteractivity` for tabs/accordion — follow-up
  enhancement for general-purpose deferred rendering

## Design Constraints Verified

- **Foundation tier**: no casehub upstream dependencies introduced
- **Composable**: each primitive is independently useful beyond dock workbenches
- **No new rendering path**: standard `renderComponent()` pipeline throughout
- **Existing event contract**: uses `pages-dock-toggle` and `pages-split-resize`
  as reserved framework events
- **Backward compatible**: non-exclusive dock bars unchanged, existing split
  collapse behavior preserved (component-level targeting cascades to slot-level
  for single-child slots, matching existing behavior)
- **Reserved event**: `pages-deferred-render` is a new framework-internal event.
  Update the reserved names table in the pages-event-contract protocol in the
  same commit that adds the deferred primitive.
