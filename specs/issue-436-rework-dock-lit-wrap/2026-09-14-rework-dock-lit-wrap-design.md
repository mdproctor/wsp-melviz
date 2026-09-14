# Rework `<pages-dock-workbench>` Lit Component to Wrap Runtime Dock Infrastructure

**Issue:** casehubio/casehub-pages#436
**Date:** 2026-09-14
**Status:** Draft

## Context

The `<pages-dock-workbench>` Lit component created in #429 builds its own toggle bar, resize handling, persistence, and event contract instead of wrapping the existing runtime dock infrastructure (`ZoneLayoutEngine`, `renderDockBar()`, `DockBarProps`, `DockBarItem`, `pages-dock-toggle` events).

This creates a dual architecture:

- **Runtime path** (pages-runtime + pages-ui): `DockWorkbenchConfig` → `dockWorkbench()` builder → Component tree of primitives (`split`, `rows`, `columns`, `dockBar`, `deferred`) → `renderComponent()` → activation callbacks → `site.ts` event handlers. `ZoneLayoutEngine` manages a 6-zone model with panel drag support. State persistence via `LayoutStore`.
- **Lit component** (pages-primitives): Standalone `LitElement` with Shadow DOM. Own resize handling (pointer events), toggle logic, persistence (direct `localStorage`), custom events (`dock-panel-toggle`, `dock-panel-resize`). No awareness of `DockBarItem`, `DockBarProps`, `ZoneLayoutEngine`, or `pages-dock-toggle`. Only consumer: `pages-builder` (`builder-shell.ts`).

Two independent systems doing the same job with different APIs.

## Architectural Approach

**Unified Lit component** serving both the runtime YAML path and standalone Lit consumers. The `<pages-dock-workbench>` Lit component becomes the single dock-workbench implementation. The runtime targets it for YAML-declared dock workbenches; `pages-builder` uses it directly.

The Lit component is a **layout engine**, not a content renderer. It renders dock bars, resize handles, and zone containers. Content is provided by the runtime (via a render callback) or by the consumer (standalone mode). This follows the content-agnostic-workbench protocol — the workbench manages layout, never specific content types.

### Key decisions

| # | Decision | Choice |
|---|----------|--------|
| D1 | DOM mode | Light DOM (`createRenderRoot() { return this; }`) — no shadow boundary |
| D2 | Persistence | Lit component owns state; accepts optional `LayoutStore`, falls back to `localStorage` |
| D3 | Package placement | Stays in `pages-primitives`; move pure types (`LayoutStore`, `DockBarItem`) down to `pages-component` |
| D4 | Content rendering | Render callback injection: `renderContent(container, panelConfig)` |
| D5 | Toggle handling | Lit component owns all dock-toggle logic; `site.ts` removes its handler |

Full rationale in `decisions.md`.

## Component API

```typescript
class PagesDockWorkbench extends LitElement {
  createRenderRoot() { return this; }

  // --- Config mode (runtime YAML path) ---
  @property({ attribute: false }) config?: DockWorkbenchConfig;

  // --- Standalone mode (pages-builder) ---
  @property({ attribute: false }) leftPanels?: DockBarItem[];
  @property({ attribute: false }) rightPanels?: DockBarItem[];
  @property({ attribute: false }) bottomPanels?: DockBarItem[];

  // --- Persistence ---
  @property({ attribute: false }) layoutStore?: LayoutStore;
  @property({ attribute: 'persist-key' }) persistKey?: string;

  // --- Content rendering callback ---
  @property({ attribute: false })
  renderContent?: (container: HTMLElement, panel: DockPanelConfig) => void;

  // --- Zone engine ---
  @property({ attribute: false }) zoneEngine?: ZoneLayoutEngine;

  // --- Reactive internal state ---
  @state() private _dockState = new Map<string, boolean>();
  @state() private _splitRatios = new Map<string, number[]>();
  @state() private _resizing: string | null = null;

  // --- Public methods ---
  togglePanel(panelId: string): void;
  showPanel(panelId: string): void;
  hidePanel(panelId: string): void;
}
```

### Two usage modes

**Config mode** — runtime YAML path:
1. Runtime's activation callback creates `<pages-dock-workbench>`, sets `config`, `renderContent`, `layoutStore`, and `persistKey`.
2. Lit component creates `ZoneLayoutEngine` internally from config, builds the tree, renders dock bars and zone containers.
3. On first panel open, calls `renderContent(container, panelConfig)` for deferred content rendering.
4. Runtime queries (`querySelector`) work unchanged — light DOM means all rendered content is in the same DOM tree.

**Standalone mode** — `pages-builder` and other Lit consumers:
1. Consumer sets `leftPanels`, `rightPanels`, `bottomPanels` with `DockBarItem[]` arrays.
2. Consumer provides content as child elements or via `renderContent` callback.
3. `persistKey` enables simple `localStorage` persistence (no `LayoutStore` needed).

When `config` is set, it takes precedence — `leftPanels`/`rightPanels`/`bottomPanels` are ignored.

## Rendering

The Lit component renders the **layout chrome** only:

- **Dock bars** — vertical for left/right sides, horizontal for bottom. Buttons rendered from `DockBarItem[]` (extracted from config or provided directly). Each button has `data-dock-panel-id` and `data-dock-zone` attributes matching the runtime contract.
- **Resize handles** — between zone containers and the centre area. Pointer-event-based resize (carried forward from the existing Lit component — simpler and self-contained vs. the runtime's `split`-based resize).
- **Zone containers** — empty `<div>` elements with `data-dock-zone` attributes, one per zone. Content is rendered into these by the `renderContent` callback or by the consumer.
- **Centre container** — always visible, holds the centre content.

The Lit component does **not** render panel content. Content ownership follows the content-agnostic-workbench protocol.

### Generated DOM structure

```
<pages-dock-workbench>          ← light DOM root
  <div class="dock-layout">
    <div class="dock-main">
      <div class="dock-bar dock-bar-left">     ← left dock bar buttons
      <div class="resize-handle resize-left">
      <div class="dock-zone dock-zone-left">   ← left zone container (content injected here)
        <div data-component-id="inbox" data-deferred="pending">
        <div data-component-id="cases" data-deferred="pending">
      </div>
      <div class="resize-handle resize-centre-right">
      <div class="dock-zone dock-zone-centre"> ← centre content
      <div class="resize-handle resize-right">
      <div class="dock-zone dock-zone-right">  ← right zone container
      <div class="dock-bar dock-bar-right">    ← right dock bar buttons
    </div>
    <div class="resize-handle resize-bottom">
    <div class="dock-zone dock-zone-bottom">   ← bottom zone container
    <div class="dock-bar dock-bar-bottom">     ← bottom dock bar buttons
  </div>
</pages-dock-workbench>
```

Zone containers that have no configured panels are omitted entirely (same zone-omission logic as the current `buildTreeFromZones`).

## Dock-toggle handling

The Lit component owns all dock-toggle behavior (D5). When a dock bar button is clicked:

1. **Exclusive enforcement:** If the button's dock bar is exclusive (default for dock-workbench zones), find the currently active panel in the same zone. If one exists and it's different from the clicked panel, hide it — set `display: none`, remove `data-active` from its button, update `_dockState`.

2. **Toggle logic:** If the clicked panel is already visible, hide it (and cascade collapse). If hidden, show it (and cascade expand).

3. **Deferred render:** If the panel container has `data-deferred="pending"`:
   - Call `renderContent(container, panelConfig)` if a callback is available.
   - Otherwise dispatch `pages-deferred-render` on the container.
   - Remove `data-deferred` attribute after rendering.

4. **Show/hide:** Set `display` on the panel container. Update the button's `data-active` attribute.

5. **Cascade collapse/expand** (moved from `site.ts:917-1015`):
   - **On hide:** Walk up from the panel element through ancestor containers. If all sibling `[data-component-id]` children in a container are hidden, collapse the container. Continue up through parent containers. Hide adjacent split drag handles.
   - **On show:** Walk up from the panel element. If any ancestor container is hidden, show it (bottom-up). Show adjacent split drag handles.

6. **State update:** Update `_dockState` map. Schedule persistence save.

7. **Event re-dispatch:** After internal handling, dispatch `pages-dock-toggle` with `{ panelId, visible }` detail, `bubbles: true`, `composed: true`. External listeners (URL sync, analytics) observe the event. The internal handler does **not** re-process this event (guard via a flag or by dispatching on `this` rather than a child).

### Public methods

`togglePanel(panelId)`, `showPanel(panelId)`, `hidePanel(panelId)` — programmatic API for the same toggle logic. The runtime uses these for URL-hash-driven panel activation and keyboard shortcuts.

## State initialization

On `firstUpdated` (after the initial render creates zone containers):

1. **Load persisted state:**
   - If `layoutStore` and `persistKey` are set: `await layoutStore.load(persistKey)` → `LayoutState`.
   - Else if `persistKey` is set: read from `localStorage` (sync, same as current behavior).
   - Extract `docks` (panel visibility), `splits` (ratios), and `zones` (panel-to-zone mapping) from the loaded state.

2. **Apply zone mapping:** If saved zone positions exist and `config` is set, pass them to `createZoneLayoutEngine(config, savedZones)`. The engine incorporates saved positions, respecting `allowedZones` constraints.

3. **Determine active panel per zone group:** For each dock bar zone group:
   - If persisted state has a panel marked `true` → use it (saved state wins).
   - Else if a panel has `defaultOpen: true` → use it.
   - If multiple panels are `true` in a zone → use only the first (enforce exclusivity, prevent broken-on-load).

4. **Activate panels:** For each active panel, run the toggle-show path (cascade expand, deferred render, button state sync).

5. **Seed inactive state:** For each non-active panel, set `_dockState.set(panelId, false)`.

## Resize handling

Carried forward from the existing Lit component with minor adjustments:

- Pointer-event-based resize on drag handles (`pointerdown` → capture, `pointermove` → update size, `pointerup` → release + persist).
- Zone sizes stored as pixel values (`leftWidth`, `rightWidth`, `bottomHeight`) in `_splitRatios` or dedicated `@state()` properties.
- Min/max constraints via configurable properties (default: `minPanelSize = 120`, `maxPanelSize = 600`).
- On resize end, schedule persistence save.

## Persistence

The Lit component is always the dock state owner (D2).

### Save

Debounced (300ms) save after any toggle or resize:

```typescript
private _scheduleSave(): void {
  clearTimeout(this._saveTimer);
  this._saveTimer = setTimeout(() => {
    const state: Partial<LayoutState> = {
      docks: Object.fromEntries(this._dockState),
      splits: Object.fromEntries(this._splitRatios),
      ...(this.zoneEngine ? { zones: Object.fromEntries(this.zoneEngine.zoneMap) } : {}),
    };
    if (this.layoutStore && this.persistKey) {
      this.layoutStore.save(this.persistKey, state as LayoutState);
    } else if (this.persistKey) {
      localStorage.setItem(this.persistKey, JSON.stringify(state));
    }
  }, 300);
}
```

### Load

See State initialization above. `LayoutStore.load()` is async; `localStorage` is sync. The component handles both paths.

### Runtime integration with captureLayout

`site.ts`'s `captureLayout()` currently reads from its internal `dockState` Map. After this rework, it reads from the `<pages-dock-workbench>` element:

```typescript
function captureLayout(): LayoutState {
  const dockEl = target.querySelector<PagesDockWorkbench>("pages-dock-workbench");
  return Object.freeze({
    splits: Object.freeze(Object.fromEntries(splitRatios)),
    docks: dockEl ? Object.freeze(Object.fromEntries(dockEl.dockState)) : {},
    panels: captureHostPanels(),
    ...(dockEl?.zoneEngine ? { zones: Object.freeze(Object.fromEntries(dockEl.zoneEngine.zoneMap)) } : {}),
    ...(capturedState ? { containerState: capturedState } : {}),
  });
}
```

The Lit component exposes `dockState` (read-only `Map<string, boolean>`) and `zoneEngine` as public properties for this purpose.

## Runtime integration — site.ts changes

### Removed from site.ts

| Code | Lines | Reason |
|------|-------|--------|
| `pages-dock-toggle` event listener | 917–1015 | Absorbed by Lit component (D5) |
| `initDockZoneGroup` + post-render dock init | 1289–1326 | Absorbed by Lit component state initialization |
| `findDockConfig` + zone engine auto-creation | 1219–1251 | Lit component creates its own engine from `config` |
| Dock state restoration in `pages-dock-rearrange` handler | 1076–1111 | Lit component handles re-render via reactive properties |
| `dockState` Map management for dock panels | scattered | Lit component owns dock state |

### Added to activation.ts

When the activation callback encounters a `dock-workbench` component type:

```typescript
if (component.type === "dock-workbench" && component.props) {
  const config = component.props.__dockConfig as DockWorkbenchConfig;
  const dockEl = document.createElement("pages-dock-workbench") as PagesDockWorkbench;
  dockEl.config = config;
  dockEl.layoutStore = options?.layoutStore;
  dockEl.persistKey = options?.layoutKey;
  dockEl.renderContent = (container, panel) => {
    renderComponent(container, panel.content, {
      permissions: options?.permissions ?? ALLOW_ALL,
      onNode: callback,
    });
  };
  el.appendChild(dockEl);
  return;
}
```

The existing `dockWorkbench()` builder in `pages-ui` continues to produce a `Component` with `__dockConfig`. The activation callback now creates a `<pages-dock-workbench>` element instead of letting `renderComponent` expand the tree of primitives. The builder's tree output becomes unused for the runtime path but remains available for `ZoneLayoutEngine.buildTree()` inside the Lit component.

### Unchanged in site.ts

- `pages-split-resize` handler — splits outside dock-workbench still use it.
- `pages-data-request`, `pages-filter`, `pages-sort`, and all data pipeline handlers — panels inside the dock-workbench dispatch these events, which bubble up through light DOM normally.
- `scheduleLayoutSave` — delegates to the Lit component for dock state capture via `captureLayout()`.
- All non-dock event handling.

## pages-builder migration

### Current usage (Shadow DOM, slot-based)

```html
<pages-dock-workbench
  left-width="260"
  right-width="${this._dockWidth}"
  persist-key="pages-builder"
  .leftCollapsed="${!this._treeOpen}"
  .rightCollapsed="${!this._anyDockOpen}"
  .bottomEnabled="${false}"
>
  <div slot="left">...</div>
  <div slot="centre">...</div>
  <div slot="right">...</div>
  <div slot="toggle-bar-right">...</div>
</pages-dock-workbench>
```

### New usage (light DOM, config-driven)

```html
<pages-dock-workbench
  persist-key="pages-builder"
  .leftPanels="${[{ icon: '☰', label: 'Tree', panelId: 'tree', defaultOpen: true }]}"
  .rightPanels="${[
    { icon: '☰', label: 'Props', panelId: 'properties' },
    { icon: '◫', label: 'Comps', panelId: 'components' },
  ]}"
  .bottomEnabled="${false}"
>
</pages-dock-workbench>
```

`builder-shell.ts` provides a `renderContent` callback to render its own panel content into the zone containers. Since `builder-shell` uses Shadow DOM, the dock-workbench's light DOM renders into the builder-shell's shadow root — no style leakage to the page.

The builder's existing toggle state management (`_treeOpen`, `_propsOpen`, `_compsOpen`, `_anyDockOpen`) is replaced by the Lit component's internal `_dockState`. The builder listens for `pages-dock-toggle` events to stay in sync if it needs to react to panel changes.

## Type relocations

| Type | From | To |
|------|------|----|
| `LayoutStore` interface | `pages-runtime/src/layout-store.ts` | `pages-component/src/model/types.ts` |
| `createLocalLayoutStore()` | stays in `pages-runtime/src/layout-store.ts` | imports interface from `pages-component` |
| `DockBarItem` interface | `pages-runtime/src/dock-bar-renderer.ts` | `pages-component/src/model/types.ts` |
| `DockBarProps` interface | `pages-runtime/src/dock-bar-renderer.ts` | `pages-component/src/model/types.ts` |

`pages-runtime` re-exports the moved types from `pages-component` for backward compatibility (one release cycle, then remove re-exports).

`pages-primitives` gains dependencies:
- `@casehubio/pages-component` — `LayoutStore`, `LayoutState`, `DockBarItem`, `DockBarProps`, `DockZone` types
- `@casehubio/pages-ui` — `DockWorkbenchConfig`, `DockPanelConfig` types

## Testing strategy

### Unit tests (Vitest)

Rewrite existing tests for light DOM (no `shadowRoot` queries — query the element directly):

- **Config mode:** Set `config`, verify dock bars rendered with correct buttons, zone containers created with `data-dock-zone` attributes.
- **Standalone mode:** Set `leftPanels`/`rightPanels`, verify dock bars and containers.
- **Toggle:** Click dock bar button → panel shown, button gets `data-active`; click again → panel hidden, attribute removed.
- **Exclusive:** Click button B while A is active → A hidden, B shown. Only one panel visible per zone.
- **Deferred render:** `renderContent` callback called on first open only. Subsequent show/hide toggles `display` without re-calling.
- **Cascade collapse:** Hide all panels in a zone → zone container hidden. Show one → container visible.
- **Persistence (localStorage):** Toggle panels, verify `localStorage` updated. Construct new element with same `persistKey`, verify state restored.
- **Persistence (LayoutStore):** Provide mock `LayoutStore`, verify `save`/`load` called with correct `LayoutState` shape.
- **Resize:** Pointer events on handle → panel size updates within min/max constraints.
- **ARIA:** Regions, separators, labels present and correct.
- **Public methods:** `showPanel`, `hidePanel`, `togglePanel` produce correct state changes.

### Integration tests

- **Runtime integration:** `loadSite()` with `type: dock-workbench` YAML → `<pages-dock-workbench>` element created in DOM, panels rendered via callback, `pages-data-request` events from panels reach the data pipeline.
- **State round-trip:** Toggle panels, resize zones → persist → dispose → reload with same storage key → state restored correctly.
- **pages-builder:** `builder-shell` renders dock-workbench correctly with light DOM inside its own shadow root; panel toggle and resize functional.

## Scope boundary

### In scope

- Rewrite `<pages-dock-workbench>` to use light DOM and wrap runtime infrastructure
- Move `LayoutStore`, `DockBarItem`, `DockBarProps` types to `pages-component`
- Update `activation.ts` to create Lit component for `dock-workbench` component type
- Remove dock-specific code from `site.ts` (~170 lines)
- Update `builder-shell.ts` to use the new API
- Rewrite tests for light DOM and new API

### Out of scope (separate issues)

- Drag-and-drop panel rearrangement (#75) — `ZoneLayoutEngine` is wired but drag UI is not part of this rework
- Keyboard shortcuts (Alt+1, Alt+2) — future enhancement
- Animation (slide in/out) — future polish
- `renderDockBar()` function removal — may still be useful for non-Lit contexts; evaluate after this lands

## References

- `packages/pages-primitives/src/dock/pages-dock-workbench.ts` — current Lit component being reworked
- `packages/pages-runtime/src/site.ts:917-1015` — dock-toggle handler being absorbed
- `packages/pages-runtime/src/site.ts:1219-1326` — zone engine creation and dock init being absorbed
- `packages/pages-runtime/src/dock-bar-renderer.ts` — `DockBarItem`/`DockBarProps` types being relocated
- `packages/pages-runtime/src/zone-layout-engine.ts` — `ZoneLayoutEngine` interface used by Lit component
- `packages/pages-runtime/src/layout-store.ts` — `LayoutStore` interface being relocated
- `packages/pages-ui/src/dsl/builders.ts:604-629` — `DockPanelConfig`/`DockWorkbenchConfig` types
- `packages/pages-builder/src/shell/builder-shell.ts:778-823` — consumer being migrated
- `docs/specs/issue-285-dock-workbench/2026-08-04-dock-workbench-design.md` — original dock-workbench design spec
- `docs/protocols/casehub/workbench-integration-pattern.md` — extend existing packages, no new packages
- `docs/protocols/casehub/web-component-strategy.md` — all Web Components use Lit
- `docs/protocols/casehub/content-agnostic-workbench.md` — workbench manages layout, not content
- `docs/protocols/casehub/pages-event-contract.md` — `pages-dock-toggle` is a reserved framework event
