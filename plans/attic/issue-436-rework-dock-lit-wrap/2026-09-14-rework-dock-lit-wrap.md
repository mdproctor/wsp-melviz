# Rework `<pages-dock-workbench>` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #436 — Rework pages-dock-workbench Lit component to wrap existing runtime dock infrastructure
**Issue group:** #436

**Goal:** Replace the standalone `<pages-dock-workbench>` Lit component with a unified light-DOM implementation that wraps the runtime dock infrastructure, serving both the runtime YAML path and standalone Lit consumers.

**Architecture:** The Lit component becomes a layout engine using light DOM (`createRenderRoot() { return this; }`). It renders dock bars, resize handles, and zone containers — never content. Content arrives via `renderContent`/`renderCentre` callbacks (runtime provides these; standalone consumers provide their own). The runtime's activation callback converts `DockWorkbenchConfig` into the component's standalone API. Toggle handling, cascade collapse/expand, and persistence move from `site.ts` into the Lit component.

**Tech Stack:** TypeScript, Lit 3.x, Vitest

## Global Constraints

- All types used by the Lit component come from `@casehubio/pages-component` — no dependency on `pages-ui` or `pages-runtime`
- Light DOM only — no Shadow DOM, no `static styles`
- CSS delivered via `<style data-pages-dock>` injection in `connectedCallback`
- `pages-dock-toggle` is a reserved framework event (pages-event-contract protocol)
- Workbench-integration-pattern protocol: extend existing packages, no new packages
- Web-component-strategy protocol: all Web Components use Lit, `pages-` prefix

---

## Batch 1: Type foundation

### Task 1: Move config types to pages-component

**Files:**
- Modify: `packages/pages-component/src/model/types.ts` — add `LayoutStore`, `DockWorkbenchConfig`, `DockPanelConfig`, `DockSideConfig`, `NormalizedSide`, `NormalizedConfig`
- Modify: `packages/pages-component/src/model/index.ts` — export new types
- Modify: `packages/pages-runtime/src/layout-store.ts` — import `LayoutStore` from `@casehubio/pages-component`, keep `createLocalLayoutStore()`
- Modify: `packages/pages-ui/src/dsl/builders.ts` — import types from `@casehubio/pages-component` instead of defining locally
- Modify: `packages/pages-runtime/src/zone-layout-engine.ts` — import from `@casehubio/pages-component`
- Modify: `packages/pages-runtime/src/site.ts` — update `LayoutStore` import

**Interfaces:**
- Consumes: existing `Component`, `DockZone`, `DockSide` from `@casehubio/pages-component`
- Produces: `LayoutStore`, `DockWorkbenchConfig`, `DockPanelConfig`, `DockSideConfig`, `NormalizedSide`, `NormalizedConfig` exported from `@casehubio/pages-component`

- [ ] **Step 1: Add types to pages-component/src/model/types.ts**

Use `ide_insert_member` to add after the `LayoutState` interface (line ~61):

```typescript
export interface LayoutStore {
  load(key: string): Promise<LayoutState | null>;
  save(key: string, state: LayoutState): Promise<void>;
  delete(key: string): Promise<void>;
}

export interface DockPanelConfig {
  readonly key: string;
  readonly label: string;
  readonly icon: string;
  readonly defaultOpen?: boolean;
  readonly content: Component;
  readonly minSize?: number;
  readonly zone?: "top" | "bottom" | "left" | "right";
  readonly allowedZones?: readonly DockZone[];
  readonly fixed?: boolean;
}

export interface DockSideConfig {
  readonly zones?: 1 | 2;
  readonly buttonPosition?: "start" | "end";
  readonly panels: readonly DockPanelConfig[];
}

export interface DockWorkbenchConfig {
  readonly storageKey?: string;
  readonly centre: Component | Component[];
  readonly left?: readonly DockPanelConfig[] | DockSideConfig;
  readonly right?: readonly DockPanelConfig[] | DockSideConfig;
  readonly bottom?: readonly DockPanelConfig[] | DockSideConfig;
  readonly statusBar?: Component;
}

export interface NormalizedSide {
  readonly zones: 1 | 2;
  readonly buttonPosition: "start" | "end";
  readonly panels: readonly DockPanelConfig[];
  readonly side: DockSide;
}

export interface NormalizedConfig {
  readonly centre: Component | Component[];
  readonly storageKey?: string | undefined;
  readonly left?: NormalizedSide | undefined;
  readonly right?: NormalizedSide | undefined;
  readonly bottom?: NormalizedSide | undefined;
  readonly statusBar?: Component | undefined;
}
```

- [ ] **Step 2: Export new types from model/index.ts**

Add to the `export type { ... } from "./types.js"` block in `packages/pages-component/src/model/index.ts`:

```typescript
  LayoutStore,
  DockPanelConfig,
  DockSideConfig,
  DockWorkbenchConfig,
  NormalizedSide,
  NormalizedConfig,
```

- [ ] **Step 3: Update pages-runtime/src/layout-store.ts**

Replace the `LayoutStore` interface definition with an import. Keep `createLocalLayoutStore()`:

```typescript
import type { LayoutState, LayoutStore } from "@casehubio/pages-component";

export type { LayoutStore };

export function createLocalLayoutStore(prefix = "pages-layout:"): LayoutStore {
  // ... existing implementation unchanged
}
```

- [ ] **Step 4: Update pages-ui/src/dsl/builders.ts**

Replace the local type definitions (lines 604-680) with imports from `@casehubio/pages-component`:

```typescript
import type { DockPanelConfig, DockSideConfig, DockWorkbenchConfig, NormalizedSide, NormalizedConfig, DockZone, DockSide, Component } from "@casehubio/pages-component";
```

Remove the local interface definitions for `DockPanelConfig`, `DockSideConfig`, `DockWorkbenchConfig`, `NormalizedSide`, `NormalizedConfig`. Keep the re-exports so downstream consumers still find these types via `@casehubio/pages-ui`:

```typescript
export type { DockPanelConfig, DockSideConfig, DockWorkbenchConfig, NormalizedSide, NormalizedConfig };
```

- [ ] **Step 5: Update pages-runtime/src/zone-layout-engine.ts**

Change line 2:

```typescript
import type { DockPanelConfig, DockWorkbenchConfig } from "@casehubio/pages-component";
```

Keep the import of builder functions (`normalizeConfig`, `buildInitialZoneMap`, `buildTreeFromZones`) from `@casehubio/pages-ui`.

- [ ] **Step 6: Update pages-runtime/src/site.ts LayoutStore import**

Find the existing `LayoutStore` import from `"./layout-store.js"` and ensure it still resolves (the re-export in layout-store.ts handles this). No change needed if the re-export is in place.

- [ ] **Step 7: Run typecheck**

Run: `yarn typecheck` from project root
Expected: PASS — all packages compile with types in their new locations

- [ ] **Step 8: Run tests**

Run: `yarn vitest run` from project root
Expected: PASS — no behavioral changes, only type relocations

- [ ] **Step 9: Commit**

```bash
git add packages/pages-component/src/model/types.ts packages/pages-component/src/model/index.ts packages/pages-runtime/src/layout-store.ts packages/pages-ui/src/dsl/builders.ts packages/pages-runtime/src/zone-layout-engine.ts
git commit -m "refactor: move LayoutStore and dock config types to pages-component Refs #436"
```

---

### Task 2: Converge DockBarItem/DockItem types

**Files:**
- Modify: `packages/pages-runtime/src/dock-bar-renderer.ts` — delete `DockBarItem` and `DockBarProps` interfaces, import `DockItem` and `DockBarProps` from `@casehubio/pages-component`
- Modify: `packages/pages-runtime/src/activation.ts` — update imports if needed

**Interfaces:**
- Consumes: `DockItem`, `DockBarProps` from `@casehubio/pages-component`
- Produces: `renderDockBar()` now uses `DockItem` (was `DockBarItem`) — same shape, stricter types

- [ ] **Step 1: Compare types**

`DockBarItem` (pages-runtime, being deleted):
```typescript
export interface DockBarItem {
  readonly icon: string;
  readonly label: string;
  readonly panelId: string;
  readonly defaultOpen?: boolean | undefined;
  readonly zone?: string | undefined;
}
```

`DockItem` (pages-component, canonical):
```typescript
export interface DockItem {
  readonly icon: string;
  readonly label: string;
  readonly panelId: string;
  readonly defaultOpen?: boolean;
  readonly zone?: string;
  readonly allowedZones?: readonly DockZone[];
  readonly fixed?: boolean;
}
```

`DockItem` is a superset — `DockBarItem` is a loose duplicate. Safe to replace.

`DockBarProps` in pages-runtime uses `readonly items?: readonly DockBarItem[] | undefined` — update to `readonly items?: readonly DockItem[] | undefined`. The pages-component version already has `readonly items: readonly DockItem[]` (non-optional, stricter). The runtime renderer must handle the optional case, so keep the renderer's local `DockBarProps` for now or adjust.

Actually — pages-component already exports `DockBarProps` at `component-props.ts:39-44` with stricter types. The runtime's `DockBarProps` at `dock-bar-renderer.ts:12-17` is a looser version. Delete the runtime's and use pages-component's. The renderer function `renderDockBar` already handles undefined items with an early return.

- [ ] **Step 2: Update dock-bar-renderer.ts**

Delete the `DockBarItem` and `DockBarProps` interface definitions (lines 4-17). Replace with imports:

```typescript
import type { DockItem, DockBarProps } from "@casehubio/pages-component";
```

Update `DockBarOptions` to stay (it depends on `ZoneLayoutEngine` which is local):

```typescript
export interface DockBarOptions {
  readonly zoneEngine?: ZoneLayoutEngine | undefined;
  readonly siteTarget?: HTMLElement | undefined;
}
```

Update internal references from `DockBarItem` → `DockItem` in `renderDockButtons` parameter (line 26):

```typescript
function renderDockButtons(
  container: HTMLElement,
  items: readonly DockItem[],
  eventTarget: HTMLElement,
  exclusive: boolean,
  zoneName: string | undefined,
): void {
```

And in `renderDockBar` (line 60), adjust the destructure — `DockBarProps` from pages-component has `items` as non-optional `readonly DockItem[]`, but the function currently checks `if (!items) return;`. Change the function signature to accept the pages-component `DockBarProps`:

```typescript
export function renderDockBar(el: HTMLElement, props: DockBarProps, options?: DockBarOptions): void {
  const { orientation, items, exclusive } = props;
  if (!items || items.length === 0) return;
  // ... rest unchanged
}
```

- [ ] **Step 3: Update activation.ts imports if needed**

Check line 52 of `activation.ts`:
```typescript
import {renderDockBar} from "./dock-bar-renderer.js";
```

The import of `DockBarProps` type — search for `DockBarProps` in activation.ts. If it imports from dock-bar-renderer, update to import from `@casehubio/pages-component`. If it uses inline cast (line 559: `component.props as DockBarProps`), the import source just needs updating.

- [ ] **Step 4: Run typecheck**

Run: `yarn typecheck` from project root
Expected: PASS

- [ ] **Step 5: Run tests**

Run: `yarn vitest run` from project root
Expected: PASS — `DockItem` is a superset of `DockBarItem`, all call sites are compatible

- [ ] **Step 6: Commit**

```bash
git add packages/pages-runtime/src/dock-bar-renderer.ts packages/pages-runtime/src/activation.ts
git commit -m "refactor: converge DockBarItem into DockItem from pages-component Refs #436"
```

---

## Batch 2: Lit component rewrite

### Task 3: Rewrite PagesDockWorkbench with light DOM

**Files:**
- Modify: `packages/pages-primitives/package.json` — add `@casehubio/pages-component` dependency
- Modify: `packages/pages-primitives/src/dock/pages-dock-workbench.ts` — full rewrite
- Modify: `packages/pages-primitives/src/dock/pages-dock-workbench.test.ts` — rewrite for light DOM
- Test: `packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`

**Interfaces:**
- Consumes: `DockItem`, `DockZone`, `LayoutStore`, `LayoutState` from `@casehubio/pages-component`
- Produces: `PagesDockWorkbench` class with properties `leftPanels`, `rightPanels`, `bottomPanels`, `renderContent`, `renderCentre`, `layoutStore`, `persistKey`, `zoneMap`, `minPanelSize`, `maxPanelSize`; methods `togglePanel(panelId)`, `showPanel(panelId)`, `hidePanel(panelId)`; getter `dockState`

- [ ] **Step 1: Add pages-component dependency**

Edit `packages/pages-primitives/package.json` — add to `dependencies`:

```json
"@casehubio/pages-component": "workspace:*"
```

- [ ] **Step 2: Write the rendering tests**

Rewrite `packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import './pages-dock-workbench.js';
import type { PagesDockWorkbench } from './pages-dock-workbench.js';
import type { DockItem } from '@casehubio/pages-component';

function createDock(props?: Partial<PagesDockWorkbench>): PagesDockWorkbench {
  const el = document.createElement('pages-dock-workbench') as PagesDockWorkbench;
  if (props) {
    for (const [key, value] of Object.entries(props)) {
      (el as any)[key] = value;
    }
  }
  return el;
}

const leftItems: DockItem[] = [
  { icon: '📁', label: 'Explorer', panelId: 'explorer', defaultOpen: true },
  { icon: '🔍', label: 'Search', panelId: 'search' },
];

const rightItems: DockItem[] = [
  { icon: '⚙', label: 'Props', panelId: 'properties', zone: 'top', defaultOpen: true },
  { icon: '🧩', label: 'Comps', panelId: 'components', zone: 'bottom' },
];

describe('PagesDockWorkbench', () => {
  let el: PagesDockWorkbench;

  afterEach(() => {
    el?.remove();
  });

  it('uses light DOM (no shadow root)', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.shadowRoot).toBeNull();
    expect(el.querySelector('.dock-layout')).toBeTruthy();
  });

  it('renders left dock bar with buttons', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    const bar = el.querySelector('.dock-bar-left');
    expect(bar).toBeTruthy();
    const buttons = bar!.querySelectorAll('button[data-dock-panel-id]');
    expect(buttons.length).toBe(2);
    expect(buttons[0]!.dataset.dockPanelId).toBe('explorer');
    expect(buttons[1]!.dataset.dockPanelId).toBe('search');
  });

  it('renders right dock bar with buttons', async () => {
    el = createDock({ rightPanels: rightItems });
    document.body.appendChild(el);
    await el.updateComplete;

    const bar = el.querySelector('.dock-bar-right');
    expect(bar).toBeTruthy();
    const buttons = bar!.querySelectorAll('button[data-dock-panel-id]');
    expect(buttons.length).toBe(2);
  });

  it('renders centre zone container', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.querySelector('.dock-zone-centre')).toBeTruthy();
  });

  it('omits zones with no panels', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.querySelector('.dock-bar-right')).toBeNull();
    expect(el.querySelector('.dock-zone-right')).toBeNull();
    expect(el.querySelector('.dock-bar-bottom')).toBeNull();
    expect(el.querySelector('.dock-zone-bottom')).toBeNull();
  });

  it('renders resize handles between zones', async () => {
    el = createDock({ leftPanels: leftItems, rightPanels: rightItems });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.querySelector('.resize-handle.resize-left')).toBeTruthy();
    expect(el.querySelector('.resize-handle.resize-right')).toBeTruthy();
  });

  it('creates panel containers with data-component-id and data-deferred', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    const panels = el.querySelectorAll('[data-component-id]');
    expect(panels.length).toBe(2);
    expect(panels[0]!.getAttribute('data-component-id')).toBe('explorer');
    expect(panels[0]!.getAttribute('data-deferred')).toBe('pending');
  });

  it('injects CSS style element', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    const style = document.querySelector('style[data-pages-dock]') ?? el.querySelector('style[data-pages-dock]');
    expect(style).toBeTruthy();
  });

  it('has ARIA roles', async () => {
    el = createDock({ leftPanels: leftItems, rightPanels: rightItems });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.querySelector('[role="toolbar"]')).toBeTruthy();
    expect(el.querySelector('[role="separator"]')).toBeTruthy();
    expect(el.querySelector('[role="region"]')).toBeTruthy();
    expect(el.querySelector('[role="main"]')).toBeTruthy();
  });

  it('calls renderCentre on firstUpdated', async () => {
    const renderCentre = vi.fn();
    el = createDock({ leftPanels: leftItems, renderCentre });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(renderCentre).toHaveBeenCalledTimes(1);
    const container = renderCentre.mock.calls[0]![0] as HTMLElement;
    expect(container.classList.contains('dock-zone-centre')).toBe(true);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`
Expected: FAIL — the current component uses Shadow DOM and different API

- [ ] **Step 4: Write the component implementation**

Rewrite `packages/pages-primitives/src/dock/pages-dock-workbench.ts`:

```typescript
import { LitElement, html, nothing, type TemplateResult } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { DockItem, DockZone, LayoutStore, LayoutState } from '@casehubio/pages-component';

const DOCK_STYLES = `
  pages-dock-workbench { display: flex; flex-direction: column; height: 100%; overflow: hidden; }
  pages-dock-workbench .dock-layout { display: flex; flex-direction: column; flex: 1; overflow: hidden; }
  pages-dock-workbench .dock-main { display: flex; flex: 1; overflow: hidden; }
  pages-dock-workbench .dock-zone { overflow: auto; }
  pages-dock-workbench .dock-zone-left, pages-dock-workbench .dock-zone-right { flex-shrink: 0; }
  pages-dock-workbench .dock-zone-centre { flex: 1; min-width: 0; }
  pages-dock-workbench .dock-zone-bottom { flex-shrink: 0; border-top: 1px solid var(--pages-border-color, #dadce0); }
  pages-dock-workbench .resize-handle { flex-shrink: 0; background: transparent; transition: background 0.15s; touch-action: none; }
  pages-dock-workbench .resize-handle:hover, pages-dock-workbench .resize-handle:active { background: var(--pages-primary, #1967d2); }
  pages-dock-workbench .resize-left, pages-dock-workbench .resize-right { width: 4px; cursor: col-resize; }
  pages-dock-workbench .resize-bottom { height: 4px; cursor: row-resize; }
  pages-dock-workbench .dock-bar { display: flex; gap: 0; padding: 4px; flex-shrink: 0; }
  pages-dock-workbench .dock-bar-left, pages-dock-workbench .dock-bar-right { flex-direction: column; border-left: 1px solid var(--pages-border-color, #dadce0); }
  pages-dock-workbench .dock-bar-bottom { flex-direction: row; border-top: 1px solid var(--pages-border-color, #dadce0); }
  pages-dock-workbench .dock-bar [data-dock-zone] { display: flex; flex-direction: inherit; gap: 2px; min-width: 24px; min-height: 24px; }
  pages-dock-workbench .dock-bar [data-dock-spacer] { flex: 1; }
  pages-dock-workbench .dock-bar button { border: none; background: transparent; cursor: pointer; padding: 6px; border-radius: var(--pages-radius-sm, 4px); font-size: 16px; color: var(--pages-text-secondary, #5f6368); }
  pages-dock-workbench .dock-bar button:hover { background: var(--pages-hover-bg, rgba(0, 0, 0, 0.04)); }
  pages-dock-workbench .dock-bar button[data-active] { color: var(--pages-primary, #1967d2); background: rgba(25, 103, 210, 0.08); }
`;

export class PagesDockWorkbench extends LitElement {
  override createRenderRoot() { return this; }

  @property({ attribute: false }) leftPanels?: DockItem[];
  @property({ attribute: false }) rightPanels?: DockItem[];
  @property({ attribute: false }) bottomPanels?: DockItem[];

  @property({ attribute: false }) layoutStore?: LayoutStore;
  @property({ attribute: 'persist-key' }) persistKey?: string;

  @property({ attribute: false }) renderContent?: (container: HTMLElement, panelId: string) => void;
  @property({ attribute: false }) renderCentre?: (container: HTMLElement) => void;

  @property({ attribute: false }) zoneMap?: ReadonlyMap<string, DockZone>;

  @property({ type: Number, attribute: 'min-panel-size' }) minPanelSize = 120;
  @property({ type: Number, attribute: 'max-panel-size' }) maxPanelSize = 600;

  @state() private _dockState: Record<string, boolean> = {};
  @state() private _splitSizes: Record<string, number> = { left: 260, right: 320, bottom: 200 };
  @state() private _resizing: string | null = null;
  private _saveTimer: ReturnType<typeof setTimeout> | undefined;
  private _centreRendered = false;

  get dockState(): Readonly<Record<string, boolean>> {
    return this._dockState;
  }

  override connectedCallback(): void {
    super.connectedCallback();
    const root = this.getRootNode() as Document | ShadowRoot;
    if (!root.querySelector('style[data-pages-dock]')) {
      const style = document.createElement('style');
      style.setAttribute('data-pages-dock', '');
      style.textContent = DOCK_STYLES;
      if (root === document) {
        document.head.appendChild(style);
      } else {
        (root as ShadowRoot).prepend(style);
      }
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    clearTimeout(this._saveTimer);
  }

  override async firstUpdated(): Promise<void> {
    await this._loadState();
    if (this.renderCentre && !this._centreRendered) {
      const centreEl = this.querySelector<HTMLElement>('.dock-zone-centre');
      if (centreEl) {
        this.renderCentre(centreEl);
        this._centreRendered = true;
      }
    }
    this._initPanelVisibility();
  }

  private async _loadState(): Promise<void> {
    if (this.layoutStore && this.persistKey) {
      const saved = await this.layoutStore.load(this.persistKey);
      if (saved) {
        if (saved.docks) this._dockState = { ...saved.docks };
        if (saved.splits) {
          const s = saved.splits as unknown as Record<string, number>;
          this._splitSizes = { ...this._splitSizes, ...s };
        }
      }
    } else if (this.persistKey && typeof localStorage !== 'undefined') {
      try {
        const raw = localStorage.getItem(this.persistKey);
        if (raw) {
          const data = JSON.parse(raw);
          if (data.docks) this._dockState = { ...data.docks };
          if (data.splits) this._splitSizes = { ...this._splitSizes, ...data.splits };
        }
      } catch { /* ignore */ }
    }
  }

  private _initPanelVisibility(): void {
    // Implemented in Task 5 — state initialization
  }

  togglePanel(panelId: string): void {
    // Implemented in Task 4 — toggle handling
  }

  showPanel(panelId: string): void {
    // Implemented in Task 4
  }

  hidePanel(panelId: string): void {
    // Implemented in Task 4
  }

  private _scheduleSave(): void {
    // Implemented in Task 5 — persistence
  }

  private _renderDockBar(items: DockItem[], orientation: 'vertical' | 'horizontal', side: string): TemplateResult {
    const groups = new Map<string, DockItem[]>();
    for (const item of items) {
      const zone = item.zone ?? 'top';
      const list = groups.get(zone) ?? [];
      list.push(item);
      groups.set(zone, list);
    }

    const zoneEntries = [...groups.entries()];
    const topZones = zoneEntries.filter(([z]) => z === 'top' || z === 'top-second');
    const bottomZones = zoneEntries.filter(([z]) => z === 'bottom');
    const renderGroup = ([zone, zoneItems]: [string, DockItem[]]) => html`
      <div data-dock-zone="${zone}" style="display: flex; flex-direction: ${orientation === 'horizontal' ? 'row' : 'column'}; gap: 2px; min-width: 24px; min-height: 24px;">
        ${zoneItems.map(item => html`
          <button
            data-dock-panel-id="${item.panelId}"
            data-dock-zone="${zone}"
            title="${item.label}"
            aria-label="${item.label}"
            aria-pressed="${this._dockState[item.panelId] ? 'true' : 'false'}"
            ?data-active="${this._dockState[item.panelId]}"
            @click="${() => this.togglePanel(item.panelId)}"
          >${item.icon}</button>
        `)}
      </div>
    `;

    return html`
      <div class="dock-bar dock-bar-${side}"
           role="toolbar"
           aria-label="${side.charAt(0).toUpperCase() + side.slice(1)} dock bar"
           aria-orientation="${orientation}">
        ${topZones.map(renderGroup)}
        ${topZones.length > 0 && bottomZones.length > 0 ? html`<div data-dock-spacer style="flex: 1;"></div>` : nothing}
        ${bottomZones.map(renderGroup)}
      </div>
    `;
  }

  private _renderPanelContainers(items: DockItem[]): TemplateResult {
    return html`${items.map(item => html`
      <div data-component-id="${item.panelId}" data-deferred="pending" style="display: none; flex: 1; min-height: 0;"></div>
    `)}`;
  }

  private _handleResizeStart(zone: string, e: PointerEvent): void {
    e.preventDefault();
    this._resizing = zone;
    (e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
  }

  private _handleResizeMove(zone: string, e: PointerEvent): void {
    if (this._resizing !== zone) return;
    const rect = this.getBoundingClientRect();
    let size: number;
    if (zone === 'left') size = e.clientX - rect.left;
    else if (zone === 'right') size = rect.right - e.clientX;
    else size = rect.bottom - e.clientY;
    size = Math.max(this.minPanelSize, Math.min(this.maxPanelSize, size));
    this._splitSizes = { ...this._splitSizes, [zone]: size };
  }

  private _handleResizeEnd(e: PointerEvent): void {
    if (!this._resizing) return;
    this._resizing = null;
    (e.currentTarget as HTMLElement).releasePointerCapture(e.pointerId);
    this._scheduleSave();
  }

  override render(): TemplateResult {
    const hasLeft = this.leftPanels && this.leftPanels.length > 0;
    const hasRight = this.rightPanels && this.rightPanels.length > 0;
    const hasBottom = this.bottomPanels && this.bottomPanels.length > 0;

    return html`
      <div class="dock-layout">
        <div class="dock-main">
          ${hasLeft ? html`
            ${this._renderDockBar(this.leftPanels!, 'vertical', 'left')}
            <div class="resize-handle resize-left"
                 role="separator" aria-orientation="vertical"
                 aria-valuenow="${this._splitSizes.left}"
                 @pointerdown="${(e: PointerEvent) => this._handleResizeStart('left', e)}"
                 @pointermove="${(e: PointerEvent) => this._handleResizeMove('left', e)}"
                 @pointerup="${(e: PointerEvent) => this._handleResizeEnd(e)}"></div>
            <div class="dock-zone dock-zone-left" role="region" aria-label="Left panel"
                 style="width: ${this._splitSizes.left}px">
              ${this._renderPanelContainers(this.leftPanels!)}
            </div>
          ` : nothing}

          <div class="dock-zone dock-zone-centre" role="main"></div>

          ${hasRight ? html`
            <div class="resize-handle resize-right"
                 role="separator" aria-orientation="vertical"
                 aria-valuenow="${this._splitSizes.right}"
                 @pointerdown="${(e: PointerEvent) => this._handleResizeStart('right', e)}"
                 @pointermove="${(e: PointerEvent) => this._handleResizeMove('right', e)}"
                 @pointerup="${(e: PointerEvent) => this._handleResizeEnd(e)}"></div>
            <div class="dock-zone dock-zone-right" role="region" aria-label="Right panel"
                 style="width: ${this._splitSizes.right}px">
              ${this._renderPanelContainers(this.rightPanels!)}
            </div>
            ${this._renderDockBar(this.rightPanels!, 'vertical', 'right')}
          ` : nothing}
        </div>

        ${hasBottom ? html`
          <div class="resize-handle resize-bottom"
               role="separator" aria-orientation="horizontal"
               aria-valuenow="${this._splitSizes.bottom}"
               @pointerdown="${(e: PointerEvent) => this._handleResizeStart('bottom', e)}"
               @pointermove="${(e: PointerEvent) => this._handleResizeMove('bottom', e)}"
               @pointerup="${(e: PointerEvent) => this._handleResizeEnd(e)}"></div>
          <div class="dock-zone dock-zone-bottom" role="region" aria-label="Bottom panel"
               style="height: ${this._splitSizes.bottom}px">
            ${this._renderPanelContainers(this.bottomPanels!)}
          </div>
          ${this._renderDockBar(this.bottomPanels!, 'horizontal', 'bottom')}
        ` : nothing}
      </div>
    `;
  }
}

if (!customElements.get('pages-dock-workbench')) {
  customElements.define('pages-dock-workbench', PagesDockWorkbench);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-primitives/
git commit -m "feat: rewrite PagesDockWorkbench with light DOM rendering Refs #436"
```

---

### Task 4: Toggle handling and cascade

**Files:**
- Modify: `packages/pages-primitives/src/dock/pages-dock-workbench.ts` — implement `togglePanel`, `showPanel`, `hidePanel`, cascade logic
- Modify: `packages/pages-primitives/src/dock/pages-dock-workbench.test.ts` — add toggle tests

**Interfaces:**
- Consumes: `renderContent` callback, `_dockState` internal state
- Produces: `togglePanel(panelId)`, `showPanel(panelId)`, `hidePanel(panelId)` methods; dispatches `pages-dock-toggle` events

- [ ] **Step 1: Write toggle tests**

Append to the test file:

```typescript
describe('Toggle handling', () => {
  let el: PagesDockWorkbench;

  afterEach(() => {
    el?.remove();
  });

  it('showPanel makes panel visible and sets button active', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('explorer');
    await el.updateComplete;

    const panel = el.querySelector('[data-component-id="explorer"]') as HTMLElement;
    expect(panel.style.display).not.toBe('none');
    const btn = el.querySelector('button[data-dock-panel-id="explorer"]') as HTMLElement;
    expect(btn.dataset.active).toBeDefined();
    expect(el.dockState['explorer']).toBe(true);
  });

  it('hidePanel hides panel and removes button active', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('explorer');
    await el.updateComplete;
    el.hidePanel('explorer');
    await el.updateComplete;

    const panel = el.querySelector('[data-component-id="explorer"]') as HTMLElement;
    expect(panel.style.display).toBe('none');
    const btn = el.querySelector('button[data-dock-panel-id="explorer"]') as HTMLElement;
    expect(btn.dataset.active).toBeUndefined();
    expect(el.dockState['explorer']).toBe(false);
  });

  it('togglePanel toggles visibility', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    el.togglePanel('explorer');
    expect(el.dockState['explorer']).toBe(true);
    el.togglePanel('explorer');
    expect(el.dockState['explorer']).toBe(false);
  });

  it('exclusive zone: showing B hides A in same zone', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('explorer');
    el.showPanel('search');

    expect(el.dockState['explorer']).toBe(false);
    expect(el.dockState['search']).toBe(true);
  });

  it('different zones are independent', async () => {
    el = createDock({ rightPanels: rightItems });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('properties');
    el.showPanel('components');

    expect(el.dockState['properties']).toBe(true);
    expect(el.dockState['components']).toBe(true);
  });

  it('calls renderContent on first open only', async () => {
    const renderContent = vi.fn();
    el = createDock({ leftPanels: leftItems, renderContent });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('explorer');
    expect(renderContent).toHaveBeenCalledTimes(1);
    expect(renderContent).toHaveBeenCalledWith(expect.any(HTMLElement), 'explorer');

    el.hidePanel('explorer');
    el.showPanel('explorer');
    expect(renderContent).toHaveBeenCalledTimes(1);
  });

  it('dispatches pages-dock-toggle event', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('pages-dock-toggle', ((e: CustomEvent) => events.push(e)) as EventListener);

    el.showPanel('explorer');
    expect(events).toHaveLength(1);
    expect(events[0]!.detail).toEqual({ panelId: 'explorer', visible: true });
    expect(events[0]!.bubbles).toBe(true);
    expect(events[0]!.composed).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`
Expected: FAIL — toggle methods are stubs

- [ ] **Step 3: Implement toggle logic**

Replace the stub methods in `pages-dock-workbench.ts`:

```typescript
  private _allPanels(): DockItem[] {
    return [...(this.leftPanels ?? []), ...(this.rightPanels ?? []), ...(this.bottomPanels ?? [])];
  }

  private _findPanelZone(panelId: string): string | undefined {
    for (const item of this._allPanels()) {
      if (item.panelId === panelId) return item.zone ?? 'top';
    }
    return undefined;
  }

  private _findPanelSide(panelId: string): 'left' | 'right' | 'bottom' | undefined {
    if (this.leftPanels?.some(p => p.panelId === panelId)) return 'left';
    if (this.rightPanels?.some(p => p.panelId === panelId)) return 'right';
    if (this.bottomPanels?.some(p => p.panelId === panelId)) return 'bottom';
    return undefined;
  }

  private _sameSidePanels(panelId: string): DockItem[] {
    const side = this._findPanelSide(panelId);
    if (side === 'left') return this.leftPanels ?? [];
    if (side === 'right') return this.rightPanels ?? [];
    if (side === 'bottom') return this.bottomPanels ?? [];
    return [];
  }

  togglePanel(panelId: string): void {
    if (this._dockState[panelId]) {
      this.hidePanel(panelId);
    } else {
      this.showPanel(panelId);
    }
  }

  showPanel(panelId: string): void {
    const zone = this._findPanelZone(panelId);
    const newState = { ...this._dockState };

    // Exclusive: hide other panels in same zone on same side
    if (zone) {
      for (const item of this._sameSidePanels(panelId)) {
        if (item.panelId !== panelId && (item.zone ?? 'top') === zone && newState[item.panelId]) {
          newState[item.panelId] = false;
          this._hidePanelDom(item.panelId);
        }
      }
    }

    newState[panelId] = true;
    this._dockState = newState;
    this._showPanelDom(panelId);
    this._scheduleSave();

    this.dispatchEvent(new CustomEvent('pages-dock-toggle', {
      bubbles: true, composed: true,
      detail: { panelId, visible: true },
    }));
  }

  hidePanel(panelId: string): void {
    this._dockState = { ...this._dockState, [panelId]: false };
    this._hidePanelDom(panelId);
    this._scheduleSave();

    this.dispatchEvent(new CustomEvent('pages-dock-toggle', {
      bubbles: true, composed: true,
      detail: { panelId, visible: false },
    }));
  }

  private _showPanelDom(panelId: string): void {
    const panelEl = this.querySelector<HTMLElement>(`[data-component-id="${panelId}"]`);
    if (!panelEl) return;

    // Deferred render
    if (panelEl.dataset.deferred === 'pending') {
      if (this.renderContent) {
        this.renderContent(panelEl, panelId);
      } else {
        panelEl.dispatchEvent(new Event('pages-deferred-render'));
      }
      delete panelEl.dataset.deferred;
    }

    panelEl.style.display = '';

    // Cascade expand: ensure zone container is visible
    const zoneContainer = panelEl.parentElement;
    if (zoneContainer && zoneContainer.style.display === 'none') {
      zoneContainer.style.display = '';
      // Show adjacent resize handle
      const prev = zoneContainer.previousElementSibling as HTMLElement | null;
      if (prev?.classList.contains('resize-handle')) prev.style.display = '';
      const next = zoneContainer.nextElementSibling as HTMLElement | null;
      if (next?.classList.contains('resize-handle')) next.style.display = '';
    }

    // Update button
    const btn = this.querySelector<HTMLElement>(`button[data-dock-panel-id="${panelId}"]`);
    if (btn) {
      btn.dataset.active = '';
      btn.setAttribute('aria-pressed', 'true');
    }
  }

  private _hidePanelDom(panelId: string): void {
    const panelEl = this.querySelector<HTMLElement>(`[data-component-id="${panelId}"]`);
    if (!panelEl) return;

    panelEl.style.display = 'none';

    // Update button
    const btn = this.querySelector<HTMLElement>(`button[data-dock-panel-id="${panelId}"]`);
    if (btn) {
      delete btn.dataset.active;
      btn.setAttribute('aria-pressed', 'false');
    }

    // Cascade collapse: if all panels in zone container are hidden, hide the container
    const zoneContainer = panelEl.parentElement;
    if (zoneContainer) {
      const siblings = zoneContainer.querySelectorAll<HTMLElement>('[data-component-id]');
      const allHidden = siblings.length > 0 && Array.from(siblings).every(s => s.style.display === 'none');
      if (allHidden) {
        zoneContainer.style.display = 'none';
        const prev = zoneContainer.previousElementSibling as HTMLElement | null;
        if (prev?.classList.contains('resize-handle')) prev.style.display = 'none';
        const next = zoneContainer.nextElementSibling as HTMLElement | null;
        if (next?.classList.contains('resize-handle')) next.style.display = 'none';
      }
    }
  }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-primitives/src/dock/
git commit -m "feat: implement dock-toggle handling with exclusive zones and cascade Refs #436"
```

---

### Task 5: State initialization and persistence

**Files:**
- Modify: `packages/pages-primitives/src/dock/pages-dock-workbench.ts` — implement `_initPanelVisibility`, `_scheduleSave`
- Modify: `packages/pages-primitives/src/dock/pages-dock-workbench.test.ts` — add persistence and init tests

**Interfaces:**
- Consumes: `LayoutStore.load()`, `LayoutStore.save()`, `localStorage`
- Produces: restored panel visibility on load; persisted `LayoutState` shape on save

- [ ] **Step 1: Write persistence tests**

Append to the test file:

```typescript
describe('Persistence', () => {
  let el: PagesDockWorkbench;

  afterEach(() => {
    el?.remove();
    localStorage.clear();
  });

  it('saves dock state to localStorage', async () => {
    el = createDock({ leftPanels: leftItems, persistKey: 'test-dock' });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('explorer');

    // Wait for debounce
    await new Promise(r => setTimeout(r, 350));

    const saved = JSON.parse(localStorage.getItem('test-dock')!);
    expect(saved.docks.explorer).toBe(true);
  });

  it('restores dock state from localStorage', async () => {
    localStorage.setItem('test-dock', JSON.stringify({
      docks: { explorer: true, search: false },
      splits: { left: 300 },
    }));

    el = createDock({ leftPanels: leftItems, persistKey: 'test-dock' });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.dockState['explorer']).toBe(true);
    expect(el.dockState['search']).toBe(false);
  });

  it('saves via LayoutStore when provided', async () => {
    const mockStore: LayoutStore = {
      load: vi.fn().mockResolvedValue(null),
      save: vi.fn().mockResolvedValue(undefined),
      delete: vi.fn().mockResolvedValue(undefined),
    };

    el = createDock({ leftPanels: leftItems, layoutStore: mockStore, persistKey: 'test-dock' });
    document.body.appendChild(el);
    await el.updateComplete;

    el.showPanel('explorer');
    await new Promise(r => setTimeout(r, 350));

    expect(mockStore.save).toHaveBeenCalledWith('test-dock', expect.objectContaining({
      docks: expect.objectContaining({ explorer: true }),
    }));
  });

  it('loads from LayoutStore when provided', async () => {
    const mockStore: LayoutStore = {
      load: vi.fn().mockResolvedValue({ docks: { search: true }, splits: {}, panels: {} }),
      save: vi.fn().mockResolvedValue(undefined),
      delete: vi.fn().mockResolvedValue(undefined),
    };

    el = createDock({ leftPanels: leftItems, layoutStore: mockStore, persistKey: 'test-dock' });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.dockState['search']).toBe(true);
  });
});

describe('State initialization', () => {
  let el: PagesDockWorkbench;

  afterEach(() => {
    el?.remove();
    localStorage.clear();
  });

  it('activates defaultOpen panel on init', async () => {
    const renderContent = vi.fn();
    el = createDock({ leftPanels: leftItems, renderContent });
    document.body.appendChild(el);
    await el.updateComplete;

    // explorer has defaultOpen: true
    expect(el.dockState['explorer']).toBe(true);
    expect(renderContent).toHaveBeenCalledWith(expect.any(HTMLElement), 'explorer');
  });

  it('saved state overrides defaultOpen', async () => {
    localStorage.setItem('test-dock', JSON.stringify({
      docks: { explorer: false, search: true },
      splits: {},
    }));

    const renderContent = vi.fn();
    el = createDock({ leftPanels: leftItems, persistKey: 'test-dock', renderContent });
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.dockState['explorer']).toBe(false);
    expect(el.dockState['search']).toBe(true);
  });

  it('enforces exclusivity on load — only first true panel in zone', async () => {
    localStorage.setItem('test-dock', JSON.stringify({
      docks: { explorer: true, search: true },
      splits: {},
    }));

    el = createDock({ leftPanels: leftItems, persistKey: 'test-dock' });
    document.body.appendChild(el);
    await el.updateComplete;

    // Both are in 'top' zone (default) — only first should be active
    expect(el.dockState['explorer']).toBe(true);
    expect(el.dockState['search']).toBe(false);
  });
});

describe('Resize', () => {
  let el: PagesDockWorkbench;

  afterEach(() => {
    el?.remove();
  });

  it('resize handle updates zone size', async () => {
    el = createDock({ leftPanels: leftItems });
    document.body.appendChild(el);
    await el.updateComplete;

    const handle = el.querySelector<HTMLElement>('.resize-handle.resize-left')!;
    expect(handle).toBeTruthy();
    expect(handle.getAttribute('role')).toBe('separator');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`
Expected: FAIL — `_initPanelVisibility` and `_scheduleSave` are stubs

- [ ] **Step 3: Implement _initPanelVisibility**

Replace the stub in `pages-dock-workbench.ts`:

```typescript
  private _initPanelVisibility(): void {
    const allPanels = this._allPanels();
    const newState = { ...this._dockState };
    const activatedPerZoneKey = new Set<string>();

    for (const item of allPanels) {
      const side = this._findPanelSide(item.panelId);
      const zone = item.zone ?? 'top';
      const zoneKey = `${side}:${zone}`;
      const panelId = item.panelId;

      if (newState[panelId] === true && !activatedPerZoneKey.has(zoneKey)) {
        activatedPerZoneKey.add(zoneKey);
        this._showPanelDom(panelId);
      } else if (newState[panelId] === true && activatedPerZoneKey.has(zoneKey)) {
        // Enforce exclusivity — only first true in zone wins
        newState[panelId] = false;
      } else if (newState[panelId] === undefined && item.defaultOpen && !activatedPerZoneKey.has(zoneKey)) {
        newState[panelId] = true;
        activatedPerZoneKey.add(zoneKey);
        this._showPanelDom(panelId);
      } else {
        newState[panelId] = newState[panelId] ?? false;
      }
    }

    this._dockState = newState;
  }
```

- [ ] **Step 4: Implement _scheduleSave**

Replace the stub:

```typescript
  private _scheduleSave(): void {
    if (!this.persistKey) return;
    clearTimeout(this._saveTimer);
    this._saveTimer = setTimeout(() => {
      const state = {
        docks: this._dockState,
        splits: this._splitSizes,
        ...(this.zoneMap ? { zones: Object.fromEntries(this.zoneMap) } : {}),
      };
      if (this.layoutStore) {
        this.layoutStore.save(this.persistKey!, state as unknown as LayoutState);
      } else if (typeof localStorage !== 'undefined') {
        localStorage.setItem(this.persistKey!, JSON.stringify(state));
      }
    }, 300);
  }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-primitives/src/dock/pages-dock-workbench.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-primitives/src/dock/
git commit -m "feat: implement state initialization, persistence, and resize Refs #436"
```

---

## Batch 3: Runtime wiring

### Task 6: Builder return type change

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts` — change `dockWorkbench()` return type
- Modify: `packages/pages-ui/src/dsl/builders.test.ts` — update affected tests

**Interfaces:**
- Consumes: `DockWorkbenchConfig` from `@casehubio/pages-component`
- Produces: `dockWorkbench()` returns `TypedComponent<"dock-workbench">` with `props.__dockConfig`

- [ ] **Step 1: Update the builder function**

Replace the `dockWorkbench` function body in `packages/pages-ui/src/dsl/builders.ts` (line ~892):

```typescript
export function dockWorkbench(config: DockWorkbenchConfig): TypedComponent<"dock-workbench"> {
  return Object.freeze({ type: "dock-workbench" as const, props: { __dockConfig: config } });
}
```

The tree-building functions (`normalizeConfig`, `buildInitialZoneMap`, `buildTreeFromZones`, `buildSideContent`, `buildSideStripe`) remain exported — they are used by `ZoneLayoutEngine.buildTree()`.

- [ ] **Step 2: Update builder tests**

The existing `describe("dockWorkbench builder"...)` tests verify tree structure. Update them to verify the opaque return:

```typescript
describe("dockWorkbench builder", () => {
  it("returns opaque dock-workbench component with config", () => {
    const config: DockWorkbenchConfig = {
      centre: { type: "html", props: { content: "centre" } },
      left: [{ key: "nav", label: "Nav", icon: "☰", content: { type: "html", props: { content: "nav" } } }],
    };
    const result = dockWorkbench(config);
    expect(result.type).toBe("dock-workbench");
    expect((result.props as any).__dockConfig).toBe(config);
  });
});
```

Remove or update tests that verify internal tree structure — those now test `buildTreeFromZones` directly.

- [ ] **Step 3: Run typecheck and tests**

Run: `yarn typecheck && yarn vitest run packages/pages-ui/src/dsl/builders.test.ts`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts
git commit -m "refactor: dockWorkbench() returns opaque typed component Refs #436"
```

---

### Task 7: Activation callback and site.ts changes

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts` — add `dock-workbench` handler
- Modify: `packages/pages-runtime/src/site.ts` — guard dock-toggle handler, update `captureLayout`/`deriveDockState`, remove `findDockConfig`/`initDockZoneGroup`
- Test: `packages/pages-runtime/src/workbench.test.ts` — update integration tests

**Interfaces:**
- Consumes: `PagesDockWorkbench` from `@casehubio/pages-primitives/dock`, `DockWorkbenchConfig`, `DockPanelConfig`, `DockSideConfig` from `@casehubio/pages-component`, `createZoneLayoutEngine` from local
- Produces: `<pages-dock-workbench>` elements in the DOM for `dock-workbench` component types

- [ ] **Step 1: Add dock-workbench handler in activation.ts**

Add after the existing `dock-bar` handler (around line 561). Add the import at the top of the file:

```typescript
import "@casehubio/pages-primitives/dock";
import type { PagesDockWorkbench } from "@casehubio/pages-primitives/dock";
import type { DockWorkbenchConfig, DockPanelConfig, DockSideConfig, DockItem } from "@casehubio/pages-component";
```

Insert the handler:

```typescript
    if (component.type === "dock-workbench" && component.props) {
      const config = component.props.__dockConfig as DockWorkbenchConfig;
      const savedZones = options?.savedLayout?.zones;
      const zoneEngine = createZoneLayoutEngine(config, savedZones);

      const panelMap = new Map<string, DockPanelConfig>();
      function extractPanels(side: readonly DockPanelConfig[] | DockSideConfig | undefined): DockItem[] | undefined {
        if (!side) return undefined;
        const panels: readonly DockPanelConfig[] = Array.isArray(side) ? side : (side as DockSideConfig).panels;
        return panels.map(p => {
          panelMap.set(p.key, p);
          return {
            icon: p.icon, label: p.label, panelId: p.key,
            defaultOpen: p.defaultOpen, zone: p.zone,
            allowedZones: p.allowedZones, fixed: p.fixed,
          };
        });
      }

      const dockEl = document.createElement("pages-dock-workbench") as PagesDockWorkbench;
      dockEl.leftPanels = extractPanels(config.left);
      dockEl.rightPanels = extractPanels(config.right);
      dockEl.bottomPanels = extractPanels(config.bottom);
      dockEl.layoutStore = options?.layoutStore;
      dockEl.persistKey = options?.layoutKey;
      dockEl.zoneMap = zoneEngine.zoneMap;
      dockEl.renderContent = (container: HTMLElement, panelId: string) => {
        const panel = panelMap.get(panelId);
        if (panel) {
          renderComponent(container, panel.content, {
            permissions: options?.permissions ?? ALLOW_ALL,
            onNode: callback,
          });
        }
      };
      dockEl.renderCentre = (container: HTMLElement) => {
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

- [ ] **Step 2: Guard dock-toggle handler in site.ts**

At line 917, add guard at the top of the handler:

```typescript
  target.addEventListener("pages-dock-toggle", ((e: Event) => {
    // Skip events from inside a <pages-dock-workbench> — the Lit component handles its own
    if ((e.target as HTMLElement)?.closest?.("pages-dock-workbench")) return;
    // ... rest of existing handler unchanged
```

- [ ] **Step 3: Update captureLayout in site.ts**

Find the `captureLayout` function and update to read dock state from the Lit component:

```typescript
  function captureLayout(): LayoutState {
    const dockEl = target.querySelector<HTMLElement & { dockState: Record<string, boolean>; zoneMap?: ReadonlyMap<string, unknown> }>("pages-dock-workbench");
    const capturedDocks = dockEl
      ? Object.freeze(dockEl.dockState)
      : Object.freeze(Object.fromEntries(dockState));
    const capturedState = floatingWorkspaceRef.rootContainer
      ? captureContainerState(floatingWorkspaceRef.rootContainer)
      : containerStateStash;
    return Object.freeze({
      splits: Object.freeze(Object.fromEntries(splitRatios)),
      docks: capturedDocks,
      panels: captureHostPanels(),
      ...(dockEl?.zoneMap ? { zones: Object.freeze(Object.fromEntries(dockEl.zoneMap)) } : zoneEngine ? { zones: Object.freeze(Object.fromEntries(zoneEngine.zoneMap)) } : {}),
      ...(capturedState ? { containerState: capturedState } : {}),
    });
  }
```

- [ ] **Step 4: Update deriveDockState for URL sync**

Find `deriveDockState` in site.ts and update:

```typescript
  function deriveDockState(): Record<string, boolean> {
    const dockEl = target.querySelector<HTMLElement & { dockState: Record<string, boolean> }>("pages-dock-workbench");
    if (dockEl) return { ...dockEl.dockState };
    return Object.fromEntries(dockState);
  }
```

- [ ] **Step 5: Remove findDockConfig and zone engine auto-creation**

Delete lines 1219-1251 (the `findDockConfig` function and the `if (!zoneEngine) { ... }` block that auto-creates from `__dockConfig`). The activation callback now creates the Lit component and zone engine directly.

- [ ] **Step 6: Remove initDockZoneGroup**

Delete the `initDockZoneGroup` function and its call site (lines 1289-1326). The Lit component handles init in `firstUpdated`.

- [ ] **Step 7: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 8: Run tests**

Run: `yarn vitest run`
Expected: PASS — runtime tests should still work. Some workbench.test.ts tests may need updating.

- [ ] **Step 9: Commit**

```bash
git add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/site.ts
git commit -m "feat: wire dock-workbench activation to Lit component, guard site.ts handlers Refs #436"
```

---

### Task 8: builder-shell migration

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — replace slot-based dock usage with callback API
- Modify: `packages/pages-builder/src/shell/builder-shell.test.ts` — update tests

**Interfaces:**
- Consumes: `PagesDockWorkbench` from `@casehubio/pages-primitives/dock`, `DockItem` from `@casehubio/pages-component`
- Produces: builder-shell renders correctly with the new dock API

- [ ] **Step 1: Update the dock-workbench usage in render()**

Replace lines 778-823 in `builder-shell.ts`:

```typescript
        <pages-dock-workbench
          persist-key="pages-builder"
          .leftPanels="${[{ icon: '☰', label: 'Tree', panelId: 'tree', defaultOpen: true }]}"
          .rightPanels="${[
            { icon: '☰', label: 'Props', panelId: 'properties', zone: 'top', defaultOpen: true },
            { icon: '▦', label: 'Comps', panelId: 'components', zone: 'bottom' },
          ]}"
          .renderContent="${this._renderDockContent}"
          .renderCentre="${this._renderEditor}"
        >
        </pages-dock-workbench>
```

- [ ] **Step 2: Add renderDockContent callback**

Add a method to `builder-shell.ts`:

```typescript
  private _renderDockContent = (container: HTMLElement, panelId: string): void => {
    if (panelId === 'tree') {
      const treeEl = document.createElement('pages-builder-tree') as any;
      treeEl.document = this._document;
      treeEl.selectedPath = this._selectedPath;
      treeEl.addEventListener('node-select', (e: CustomEvent) => this._handleNodeSelect(e));
      container.appendChild(treeEl);
    } else if (panelId === 'properties') {
      const wrapper = document.createElement('div');
      wrapper.className = 'dock-section';
      wrapper.innerHTML = '<div class="dock-section-header"><span>Properties</span></div>';
      const content = document.createElement('div');
      content.className = 'dock-section-content';
      wrapper.appendChild(content);
      container.appendChild(wrapper);
      this._propsContainer = content;
      this._updatePropertyPalette();
    } else if (panelId === 'components') {
      const wrapper = document.createElement('div');
      wrapper.className = 'dock-section';
      wrapper.innerHTML = '<div class="dock-section-header"><span>Components</span></div>';
      const content = document.createElement('div');
      content.className = 'dock-section-content';
      const palette = document.createElement('pages-builder-palette') as any;
      palette.context = this._paletteContext;
      palette.addEventListener('component-select', (e: CustomEvent) => this._handleComponentSelect(e));
      content.appendChild(palette);
      wrapper.appendChild(content);
      container.appendChild(wrapper);
      this._refreshPaletteContext();
    }
  };
```

- [ ] **Step 3: Add renderEditor callback**

```typescript
  private _renderEditor = (container: HTMLElement): void => {
    const showSource = this._viewMode === 'source' || this._viewMode === 'split';
    const showVisual = this._viewMode === 'visual' || this._viewMode === 'split';

    const sourceDiv = document.createElement('div');
    sourceDiv.className = `editor-source${showSource ? '' : ' hidden'}${this._viewMode === 'split' ? ' split' : ''}`;
    const editorEl = document.createElement('pages-code-editor') as any;
    editorEl.extensions = builderHighlightExtension;
    editorEl.language = 'yaml';
    editorEl.label = 'Page YAML source';
    sourceDiv.appendChild(editorEl);

    const visualDiv = document.createElement('div');
    visualDiv.className = `editor-visual${showVisual ? '' : ' hidden'}${this._viewMode === 'split' ? ' split' : ''}`;
    const previewContainer = document.createElement('div');
    previewContainer.className = 'preview-container';
    visualDiv.appendChild(previewContainer);

    container.appendChild(sourceDiv);
    container.appendChild(visualDiv);
  };
```

- [ ] **Step 4: Remove unused state properties and methods**

Remove or simplify:
- `_treeOpen`, `_propsOpen`, `_compsOpen` `@state()` declarations (lines 45-47) — dock-workbench manages toggle state
- `_toggleDock()` method (lines 506-510) — dock-workbench handles toggles
- `_anyDockOpen` getter (lines 514-515) — no longer needed
- `_renderRightPanel()` method (lines 724-751) — replaced by `_renderDockContent`
- The toolbar tree-toggle button wiring — the dock bar's tree button handles it now

Keep `_dockWidth` if resize needs to be configurable, or let the dock-workbench handle it via its own persistence.

- [ ] **Step 5: Update tests in builder-shell.test.ts**

Update the test at line 127 to verify `pages-dock-workbench` element exists with the new API:

```typescript
  it('renders dock workbench with panels', async () => {
    // ... setup ...
    const dockEl = el.shadowRoot!.querySelector('pages-dock-workbench') as any;
    expect(dockEl).toBeTruthy();
    expect(dockEl.leftPanels).toBeTruthy();
    expect(dockEl.leftPanels.length).toBe(1);
    expect(dockEl.leftPanels[0].panelId).toBe('tree');
    expect(dockEl.rightPanels).toBeTruthy();
    expect(dockEl.rightPanels.length).toBe(2);
  });
```

- [ ] **Step 6: Run typecheck and tests**

Run: `yarn typecheck && yarn vitest run packages/pages-builder/`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `yarn vitest run`
Expected: PASS — all packages compile and test cleanly

- [ ] **Step 8: Commit**

```bash
git add packages/pages-builder/
git commit -m "refactor: migrate builder-shell to new dock-workbench API Refs #436"
```

---

## References

- `specs/issue-436-rework-dock-lit-wrap/2026-09-14-rework-dock-lit-wrap-design.md` — design spec this plan implements
- `packages/pages-primitives/src/dock/pages-dock-workbench.ts` — current Lit component being reworked
- `packages/pages-runtime/src/site.ts:917-1015` — dock-toggle handler being guarded
- `packages/pages-runtime/src/site.ts:1219-1326` — zone engine creation and dock init being removed
- `packages/pages-runtime/src/dock-bar-renderer.ts:4-17` — DockBarItem/DockBarProps being converged
- `packages/pages-runtime/src/activation.ts:555-561` — dock-bar activation, anchor for new handler
- `packages/pages-runtime/src/zone-layout-engine.ts` — ZoneLayoutEngine interface
- `packages/pages-runtime/src/layout-store.ts` — LayoutStore interface being relocated
- `packages/pages-ui/src/dsl/builders.ts:604-900` — builder types and function being changed
- `packages/pages-builder/src/shell/builder-shell.ts:778-823` — consumer being migrated
- `docs/protocols/casehub/workbench-integration-pattern.md` — extend existing packages
- `docs/protocols/casehub/web-component-strategy.md` — all Web Components use Lit
- `docs/protocols/casehub/content-agnostic-workbench.md` — workbench manages layout, not content
- `docs/protocols/casehub/pages-event-contract.md` — pages-dock-toggle reserved event
- GitHub #436 — focal issue
