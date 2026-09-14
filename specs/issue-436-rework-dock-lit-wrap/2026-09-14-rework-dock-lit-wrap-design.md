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
| D3 | Package placement | Stays in `pages-primitives`; move pure types (`LayoutStore`, `DockItem`, config interfaces) down to `pages-component` |
| D4 | Content rendering | Render callback injection: `renderContent(container, panelId)` for panels, `renderCentre(container)` for centre |
| D5 | Toggle handling | Lit component owns all dock-toggle logic; `site.ts` removes its handler |
| D6 | Config/standalone separation | Lit component has standalone API only; `activation.ts` converts `DockWorkbenchConfig` to standalone inputs |
| D7 | Builder return type | `dockWorkbench()` returns opaque `{type: "dock-workbench"}` component (matching `floatingWorkspace()` pattern) |
| D8 | CSS delivery | Component injects `<style>` element in `connectedCallback` (light DOM has no `static styles` support) |

Full rationale in `decisions.md`.

## Component API

The Lit component has a single mode: it receives panel descriptors and render callbacks. There is no `config` property — the runtime's activation layer (D6) converts `DockWorkbenchConfig` into these standalone inputs before creating the element.

```typescript
class PagesDockWorkbench extends LitElement {
  createRenderRoot() { return this; }

  // --- Panel descriptors ---
  @property({ attribute: false }) leftPanels?: DockItem[];
  @property({ attribute: false }) rightPanels?: DockItem[];
  @property({ attribute: false }) bottomPanels?: DockItem[];

  // --- Persistence ---
  @property({ attribute: false }) layoutStore?: LayoutStore;
  @property({ attribute: 'persist-key' }) persistKey?: string;

  // --- Content rendering callbacks ---
  @property({ attribute: false })
  renderContent?: (container: HTMLElement, panelId: string) => void;

  @property({ attribute: false })
  renderCentre?: (container: HTMLElement) => void;

  // --- Zone map (set by activation layer for persistence) ---
  @property({ attribute: false }) zoneMap?: ReadonlyMap<string, DockZone>;

  // --- Reactive internal state ---
  @state() private _dockState: Record<string, boolean> = {};
  @state() private _splitSizes: Record<string, number> = {};
  @state() private _resizing: string | null = null;

  // --- Public read-only getters ---
  get dockState(): Readonly<Record<string, boolean>>;

  // --- Public methods ---
  togglePanel(panelId: string): void;
  showPanel(panelId: string): void;
  hidePanel(panelId: string): void;
}
```

All types (`DockItem`, `DockZone`, `LayoutStore`, `LayoutState`) are imported from `@casehubio/pages-component`. No dependency on `pages-ui` or `pages-runtime`.

### Usage

**Runtime YAML path:**
1. `activation.ts` intercepts `component.type === "dock-workbench"`, extracts `DockWorkbenchConfig`, creates `ZoneLayoutEngine`, converts config panels to `DockItem[]` arrays, and creates `<pages-dock-workbench>`.
2. `activation.ts` sets `leftPanels`, `rightPanels`, `bottomPanels`, `renderContent`, `renderCentre`, `layoutStore`, `persistKey`, and `zoneMap`.
3. On first panel open, the Lit component calls `renderContent(container, panelId)`. The activation layer's callback resolves the `panelId` to a `DockPanelConfig.content` and calls `renderComponent()`.
4. `renderCentre(container)` is called during initial layout to populate the centre zone.
5. Runtime queries (`querySelector`) work unchanged — light DOM means all rendered content is in the same DOM tree.

**Standalone (pages-builder and other Lit consumers):**
1. Consumer sets `leftPanels`, `rightPanels`, `bottomPanels` with `DockItem[]` arrays.
2. Consumer provides `renderContent` and `renderCentre` callbacks. `renderContent(container, panelId)` is called on first panel open. `renderCentre(container)` is called during initial layout.
3. `persistKey` enables simple `localStorage` persistence (no `LayoutStore` needed).

The component's API is identical in both paths — the activation layer is a thin adapter.

## Rendering

The Lit component renders the **layout chrome** only:

- **Dock bars** — vertical for left/right sides, horizontal for bottom. Buttons rendered from `DockItem[]` (provided directly via properties). Each button has `data-dock-panel-id` and `data-dock-zone` attributes matching the runtime contract. Buttons are grouped by `DockItem.zone` within each dock bar — exclusivity is enforced per zone group (see §Dock-toggle handling).
- **Resize handles** — between zone containers and the centre area. Pointer-event-based resize (carried forward from the existing Lit component — simpler and self-contained vs. the runtime's `split`-based resize).
- **Zone containers** — empty `<div>` elements with `data-dock-zone` attributes, one per zone. Content is rendered into these by the `renderContent` callback.
- **Centre container** — always visible; populated via `renderCentre` callback during initial layout.

The Lit component does **not** render panel content. Content ownership follows the content-agnostic-workbench protocol.

### CSS delivery (D8)

Light DOM does not support Lit's `static styles`. The component injects a `<style data-pages-dock>` element into the document (or the containing shadow root, if the component is inside one — e.g., builder-shell) during `connectedCallback`. If a `[data-pages-dock]` style element already exists in the same root, the component skips injection to avoid duplicates.

The injected stylesheet covers: dock bar layout (flexbox, gap, padding), resize handle appearance and cursor, zone container sizing, centre container flex, and button states (`[data-active]` highlight). This replaces the ad-hoc `<style data-pages-dock>` injection currently in `site.ts:1275-1285`.

### Generated DOM structure

```
<pages-dock-workbench>                                    ← light DOM root
  <style data-pages-dock>...</style>                      ← injected CSS (if not already present in root)
  <div class="dock-layout">
    <div class="dock-main">
      <div class="dock-bar dock-bar-left"                 ← left dock bar buttons
           role="toolbar" aria-label="Left dock bar"
           aria-orientation="vertical">
        <div data-dock-zone="top">                        ← zone group (exclusivity scoped here)
          <button data-dock-panel-id="inbox"
                  data-dock-zone="top"
                  aria-label="Inbox" aria-pressed="false">
        </div>
      </div>
      <div class="resize-handle resize-left"
           role="separator" aria-orientation="vertical"
           aria-valuenow="260">
      <div class="dock-zone dock-zone-left"               ← left zone container
           role="region" aria-label="Left panel">
        <div data-component-id="inbox" data-deferred="pending">
        <div data-component-id="cases" data-deferred="pending">
      </div>
      <div class="dock-zone dock-zone-centre"             ← centre content (populated via renderCentre)
           role="main">
      <div class="resize-handle resize-right"
           role="separator" aria-orientation="vertical"
           aria-valuenow="320">
      <div class="dock-zone dock-zone-right"              ← right zone container
           role="region" aria-label="Right panel">
      <div class="dock-bar dock-bar-right"                ← right dock bar buttons
           role="toolbar" aria-label="Right dock bar"
           aria-orientation="vertical">
    </div>
    <div class="resize-handle resize-bottom"
         role="separator" aria-orientation="horizontal"
         aria-valuenow="200">
    <div class="dock-zone dock-zone-bottom"               ← bottom zone container
         role="region" aria-label="Bottom panel">
    <div class="dock-bar dock-bar-bottom"                 ← bottom dock bar buttons
         role="toolbar" aria-label="Bottom dock bar"
         aria-orientation="horizontal">
  </div>
</pages-dock-workbench>
```

Zone containers that have no configured panels are omitted entirely (same zone-omission logic as the current `buildTreeFromZones`).

## Dock-toggle handling

The Lit component owns all dock-toggle behavior (D5). When a dock bar button is clicked:

1. **Zone-scoped exclusive enforcement:** Exclusivity is enforced per zone group within a dock bar, not per bar. The component finds the clicked button's `data-dock-zone` attribute and scopes the exclusive check to buttons sharing that zone. This allows panels in different zones of the same bar to be open simultaneously (e.g., builder-shell's Properties in "top" zone and Components in "bottom" zone of the right dock bar). This matches the existing runtime behavior in `site.ts:930-945`.

2. **Toggle logic:** If the clicked panel is already visible, hide it (and cascade collapse). If hidden, show it (and cascade expand).

3. **Deferred render:** If the panel container has `data-deferred="pending"`:
   - Call `renderContent(container, panelId)` if a callback is available.
   - Otherwise dispatch `pages-deferred-render` on the container.
   - Remove `data-deferred` attribute after rendering.

4. **Show/hide:** Set `display` on the panel container. Update the button's `data-active` and `aria-pressed` attributes.

5. **Cascade collapse/expand** (moved from `site.ts:917-1015`):
   - **On hide:** Walk up from the panel element through ancestor containers. If all sibling `[data-component-id]` children in a container are hidden, collapse the container. Continue up through parent containers. Hide adjacent resize handles.
   - **On show:** Walk up from the panel element. If any ancestor container is hidden, show it (bottom-up). Show adjacent resize handles.

6. **State update:** Update `_dockState` record via object spread (creates new reference, triggers Lit re-render). Schedule persistence save.

7. **Event notification (outbound only):** After internal state is updated, dispatch `pages-dock-toggle` with `{ panelId, visible }` detail, `bubbles: true`, `composed: true`. This event is **outbound notification only** — the component does not listen for `pages-dock-toggle` events. External listeners (URL sync, analytics) observe it. No guard mechanism is needed because the event flow is one-directional: component → external listeners.

### Public methods

`togglePanel(panelId)`, `showPanel(panelId)`, `hidePanel(panelId)` — programmatic API executing the same toggle logic as button clicks. The runtime uses these for programmatic panel activation:

- `activateDockPanel()` in site.ts calls `dockEl.showPanel(key)` directly (replacing the current event-dispatch approach)
- URL-hash-driven activation calls `dockEl.showPanel(key)`
- External callers always use methods, never events — events are for observation, not command

## State initialization

On `firstUpdated` (after the initial render creates zone containers):

1. **Render centre content:** Call `renderCentre(centreContainer)` if the callback is set.

2. **Load persisted state:**
   - If `layoutStore` and `persistKey` are set: `await layoutStore.load(persistKey)` → `LayoutState`.
   - Else if `persistKey` is set: read from `localStorage` (sync, same as current behavior).
   - Extract `docks` (panel visibility) and `splits` (sizes) from the loaded state.

3. **Determine active panel per zone group:** For each dock bar zone group:
   - If persisted state has a panel marked `true` → use it (saved state wins).
   - Else if a panel has `defaultOpen: true` → use it.
   - If multiple panels are `true` in a zone → use only the first (enforce zone-scoped exclusivity, prevent broken-on-load).

4. **Activate panels:** For each active panel, run the toggle-show path (cascade expand, deferred render, button state sync).

5. **Seed inactive state:** For each non-active panel, set `_dockState = { ...this._dockState, [panelId]: false }`.

## Resize handling

Carried forward from the existing Lit component with minor adjustments:

- Pointer-event-based resize on drag handles (`pointerdown` → capture, `pointermove` → update size, `pointerup` → release + persist).
- Zone sizes stored as pixel values (`left`, `right`, `bottom`) in `_splitSizes`.
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
      docks: this._dockState,
      splits: { left: this._splitSizes.left ?? 260, right: this._splitSizes.right ?? 320, bottom: this._splitSizes.bottom ?? 200 },
      ...(this.zoneMap ? { zones: Object.fromEntries(this.zoneMap) } : {}),
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
    docks: dockEl ? Object.freeze(dockEl.dockState) : {},
    panels: captureHostPanels(),
    ...(dockEl?.zoneMap ? { zones: Object.freeze(Object.fromEntries(dockEl.zoneMap)) } : {}),
    ...(capturedState ? { containerState: capturedState } : {}),
  });
}
```

The Lit component exposes `dockState` (read-only `Record<string, boolean>`) and `zoneMap` (read-only `Map<string, DockZone>`) as public properties for this purpose.

## Runtime integration — site.ts changes

### Removed from site.ts

| Code | Lines | Reason |
|------|-------|--------|
| `initDockZoneGroup` + post-render dock init | 1289–1326 | Absorbed by Lit component state initialization |
| `findDockConfig` + zone engine auto-creation | 1219–1251 | Activation callback creates engine and Lit component from `dock-workbench` type (D6, D7) |
| `dockState` Map management for dock panels | scattered | Lit component owns dock state |

### Guarded in site.ts (not removed)

| Code | Lines | Change |
|------|-------|--------|
| `pages-dock-toggle` event listener | 917–1015 | **Guard:** `if (e.target.closest('pages-dock-workbench')) return;` — standalone dock-bar components outside a `<pages-dock-workbench>` (e.g., `Split and Dock.page.yaml`) still rely on this handler. The Lit component handles its own events internally; the guard prevents double-handling. |
| `pages-dock-rearrange` handler | 1076–1111 | **Adapted:** handler stays, but instead of clearing the DOM and re-rendering the full tree, it calls `zoneEngine.movePanel()`, then updates the Lit component's `leftPanels`/`rightPanels`/`bottomPanels` and `zoneMap` properties. Lit re-renders reactively. See §Drag rearrange flow below. |

### URL sync fix

`syncUrl("replaceState")` calls `deriveDockState()`, which reads from site.ts's internal `dockState` Map. After this rework, the Map is stale for dock-workbench panels (the Lit component owns their state). Fix: `deriveDockState()` reads from the `<pages-dock-workbench>` element's `dockState` getter — same pattern as `captureLayout()`:

```typescript
function deriveDockState(): Record<string, boolean> {
  const dockEl = target.querySelector<PagesDockWorkbench>("pages-dock-workbench");
  if (dockEl) return { ...dockEl.dockState };
  return Object.fromEntries(dockState); // fallback for standalone dock-bars
}
```

### Drag rearrange flow

When a panel is dragged to a new zone (`pages-dock-rearrange` event):

1. `site.ts` handler receives the event with `{ panelKey, targetZone, insertIndex }`.
2. Handler calls `zoneEngine.movePanel(panelKey, targetZone, insertIndex)` — updates the zone engine's internal map.
3. Handler re-extracts panel lists per side from the engine's updated zone map (same `extractPanels` logic as activation.ts).
4. Handler sets `dockEl.leftPanels`, `dockEl.rightPanels`, `dockEl.bottomPanels`, and `dockEl.zoneMap` on the Lit component.
5. Lit component re-renders reactively — dock bars update, panel containers move to new zone containers.
6. Handler calls `dockEl.showPanel(panelKey)` to ensure the moved panel is visible.
7. `scheduleLayoutSave()` persists the new zone positions.

### Builder change (D7)

`dockWorkbench()` in `pages-ui/src/dsl/builders.ts` changes to return an opaque typed component, matching the `floatingWorkspace()` pattern:

```typescript
export function dockWorkbench(config: DockWorkbenchConfig): TypedComponent<"dock-workbench"> {
  return Object.freeze({ type: "dock-workbench" as const, props: { __dockConfig: config } });
}
```

The tree-building functions (`normalizeConfig`, `buildInitialZoneMap`, `buildTreeFromZones`) remain exported — they are used by `ZoneLayoutEngine.buildTree()` and by activation.ts for panel extraction.

### Added to activation.ts

When the activation callback encounters a `dock-workbench` component type (D6):

```typescript
if (component.type === "dock-workbench" && component.props) {
  const config = component.props.__dockConfig as DockWorkbenchConfig;

  // Create zone engine from config + saved zone positions
  const savedZones = seedLayout?.zones;
  const zoneEngine = createZoneLayoutEngine(config, savedZones);

  // Extract panels per side, converting DockPanelConfig → DockItem
  const panelMap = new Map<string, DockPanelConfig>();
  function extractPanels(side: readonly DockPanelConfig[] | DockSideConfig | undefined): DockItem[] | undefined {
    if (!side) return undefined;
    const panels = "panels" in side ? side.panels : side;
    return panels.map(p => {
      panelMap.set(p.key, p);
      return { icon: p.icon, label: p.label, panelId: p.key, defaultOpen: p.defaultOpen, zone: p.zone, allowedZones: p.allowedZones, fixed: p.fixed };
    });
  }

  const dockEl = document.createElement("pages-dock-workbench") as PagesDockWorkbench;
  dockEl.leftPanels = extractPanels(config.left);
  dockEl.rightPanels = extractPanels(config.right);
  dockEl.bottomPanels = extractPanels(config.bottom);
  dockEl.layoutStore = options?.layoutStore;
  dockEl.persistKey = options?.layoutKey;
  dockEl.zoneMap = zoneEngine.zoneMap;
  dockEl.renderContent = (container, panelId) => {
    const panel = panelMap.get(panelId);
    if (panel) {
      renderComponent(container, panel.content, {
        permissions: options?.permissions ?? ALLOW_ALL,
        onNode: callback,
      });
    }
  };
  dockEl.renderCentre = (container) => {
    const centreComponents = Array.isArray(config.centre) ? config.centre : [config.centre];
    const centreRoot = centreComponents.length === 1
      ? centreComponents[0]!
      : { type: "rows" as const, slots: { default: [...centreComponents] } };
    renderComponent(container, centreRoot, {
      permissions: options?.permissions ?? ALLOW_ALL,
      onNode: callback,
    });
  };
  el.appendChild(dockEl);
  return;
}
```

The activation layer stores the `zoneEngine` reference locally for future drag-and-drop support (out of scope). The `zoneMap` property on the element is sufficient for persistence via `captureLayout()`.

### Changed in site.ts

- `activateDockPanel()` — changes from dispatching `pages-dock-toggle` event to calling `dockEl.showPanel(key)` directly. Requires a reference to the `<pages-dock-workbench>` element (query via `target.querySelector`).

### Unchanged in site.ts

- `pages-split-resize` handler — splits outside dock-workbench still use it.
- `pages-data-request`, `pages-filter`, `pages-sort`, and all data pipeline handlers — panels inside the dock-workbench dispatch these events, which bubble up through light DOM normally.
- `scheduleLayoutSave` — delegates to the Lit component for dock state capture via `captureLayout()`.
- All non-dock event handling.

## pages-builder migration

### Current usage (manual dock rendering)

`builder-shell.ts` renders its own dock panels manually (`_renderDockPanels()`) with independent toggle state (`_propsOpen`, `_compsOpen`). Properties and Components can be open simultaneously in the right dock, stacked vertically.

### New usage (light DOM, callback-driven)

```html
<pages-dock-workbench
  persist-key="pages-builder"
  .leftPanels="${[{ icon: '☰', label: 'Tree', panelId: 'tree', defaultOpen: true }]}"
  .rightPanels="${[
    { icon: '☰', label: 'Props', panelId: 'properties', zone: 'top', defaultOpen: true },
    { icon: '◫', label: 'Comps', panelId: 'components', zone: 'bottom' },
  ]}"
  .renderContent="${this._renderDockContent}"
  .renderCentre="${this._renderEditor}"
>
</pages-dock-workbench>
```

Key migration details:
- **Multi-panel right dock:** Properties and Components are placed in separate zones (`top` and `bottom`) within the right dock bar. Zone-scoped exclusivity means they toggle independently — matching current behavior. Each zone only has one panel, so exclusivity within each zone is a no-op.
- **Centre content:** `renderCentre` callback renders the editor content into the centre zone container.
- **Panel content:** `renderContent(container, panelId)` renders Properties or Components based on `panelId`.
- **No `bottomEnabled`:** If no bottom panels are configured (`bottomPanels` is undefined), the bottom dock bar and zone are omitted entirely.

`builder-shell.ts` provides `renderContent` and `renderCentre` callbacks. Since `builder-shell` uses Shadow DOM, the dock-workbench's light DOM renders into the builder-shell's shadow root — the component's CSS injection targets the containing shadow root, not the document.

The builder's existing toggle state management (`_treeOpen`, `_propsOpen`, `_compsOpen`, `_anyDockOpen`) is replaced by the Lit component's internal `_dockState`. The builder listens for `pages-dock-toggle` events to stay in sync if it needs to react to panel changes.

## Type relocations

| Type | From | To | Notes |
|------|------|----|-------|
| `LayoutStore` interface | `pages-runtime/src/layout-store.ts` | `pages-component/src/model/types.ts` | `createLocalLayoutStore()` stays in pages-runtime, imports interface |
| `DockWorkbenchConfig` | `pages-ui/src/dsl/builders.ts` | `pages-component/src/model/types.ts` | Depends only on `Component` and `DockZone`, both already in pages-component. Follows `FloatingWorkspaceConfig` precedent (already in pages-component) |
| `DockPanelConfig` | `pages-ui/src/dsl/builders.ts` | `pages-component/src/model/types.ts` | Depends only on `Component` and `DockZone` |
| `DockSideConfig` | `pages-ui/src/dsl/builders.ts` | `pages-component/src/model/types.ts` | Depends only on `DockPanelConfig` and `DockZone` |
| `NormalizedSide`, `NormalizedConfig` | `pages-ui/src/dsl/builders.ts` | `pages-component/src/model/types.ts` | Used by `ZoneLayoutEngine` and tree builders |

### Type convergence

`DockBarItem` in `pages-runtime/src/dock-bar-renderer.ts` is a looser duplicate of `DockItem` in `pages-component/src/model/component-props.ts`. Similarly, `DockBarProps` exists in both packages with slightly different strictness.

Resolution:
- **Delete** `DockBarItem` and `DockBarProps` from `dock-bar-renderer.ts`
- **Use** `DockItem` and `DockBarProps` from `pages-component` everywhere
- `dock-bar-renderer.ts` imports from `pages-component` instead of defining its own types
- The Lit component uses `DockItem` (the pages-component name) consistently

No re-exports. Import paths break — callers update. This is a monorepo with no external consumers; the migration is mechanical.

### pages-primitives dependency

`pages-primitives` gains one dependency:
- `@casehubio/pages-component` — for `DockItem`, `DockBarProps`, `DockZone`, `LayoutStore`, `LayoutState` types

No dependency on `@casehubio/pages-ui`. The config-to-standalone conversion lives in `activation.ts` (pages-runtime), not in the Lit component.

## Testing strategy

### Unit tests (Vitest)

Rewrite existing tests for light DOM (no `shadowRoot` queries — query the element directly):

- **Panel configuration:** Set `leftPanels`/`rightPanels`/`bottomPanels`, verify dock bars rendered with correct buttons, zone containers created with `data-dock-zone` attributes.
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

- Create `<pages-dock-workbench>` Lit component in pages-primitives using light DOM
- Change `dockWorkbench()` builder to return opaque `{type: "dock-workbench"}` component (D7)
- Move `LayoutStore`, `DockWorkbenchConfig`, `DockPanelConfig`, `DockSideConfig` to `pages-component`
- Converge `DockBarItem`/`DockItem` and `DockBarProps` types — delete runtime duplicates
- Add activation callback in `activation.ts` for `dock-workbench` component type (D6)
- Remove dock-specific code from `site.ts` (~170 lines)
- Update `builder-shell.ts` to use the new API
- Write tests for light DOM and new API
- `renderDockBar()` becomes unused for the dock-workbench path — the Lit component IS the dock bar renderer. `renderDockBar()` is retained only if non-dock-workbench dock bars exist (currently they don't — all dock bars are produced by `dockWorkbench()` desugaring)

### Out of scope (separate issues)

- Drag-and-drop panel rearrangement (#75) — `ZoneLayoutEngine` is wired but drag UI is not part of this rework
- Keyboard shortcuts (Alt+1, Alt+2) — file issue during implementation
- Animation (slide in/out) — file issue during implementation
- `renderDockBar()` removal — file issue during implementation to evaluate and remove if confirmed unused

### Protocol evolution

The workbench-integration-pattern protocol (PP-20260810-72779a) says complex interactive components extend pages-ui, pages-component, and pages-runtime — "No new packages." This spec places the Lit component in pages-primitives, which predates the protocol. The protocol was written before Lit components existed in this architecture; putting a LitElement in pages-runtime (pure TypeScript, no Lit dependency) doesn't make sense. This spec should be accompanied by a protocol amendment that acknowledges pages-primitives as the Lit component layer, extending the three-package pattern to four.

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
