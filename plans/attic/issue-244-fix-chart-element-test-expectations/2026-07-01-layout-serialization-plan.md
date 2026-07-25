# Layout Serialization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make workbench layout a first-class runtime concept — observable split resize, serializable layout state, optionally persistent via pluggable LayoutStore.

**Architecture:** Six files touched across two packages. `pages-component` gets the `LayoutState`/`PanelEntry` types and the `pages-split-resize` event. `pages-runtime` gets the `LayoutStore` interface, `createLocalLayoutStore()`, and all layout state wiring in `site.ts` (state maps, event handlers, debounced auto-save, `layout` getter, post-render + lazy restore).

**Tech Stack:** TypeScript 5, Vitest, DOM CustomEvent API, localStorage

## Global Constraints

- No new packages — all code goes in `pages-component` or `pages-runtime`
- `LayoutState` keyed by explicit component IDs only — auto-generated IDs are unstable and silently skipped
- `LayoutStore` implementations must not throw — catch internally, return `null`/log warnings
- Split ratios are proportional (flex-grow values), not pixels
- Foundation tier: zero casehub upstream dependencies

---

### Task 1: LayoutState and PanelEntry types + observable split resize event

The foundational types and the root fix — making split resize observable.

**Files:**
- Modify: `packages/pages-component/src/model/types.ts` — add `LayoutState`, `PanelEntry`
- Modify: `packages/pages-component/src/model/index.ts` — export new types
- Modify: `packages/pages-component/src/renderer/interactive.ts` — `wireSplit` passes `componentId` to `attachDragHandler`; `attachDragHandler` fires `pages-split-resize` on mouseup
- Create: `packages/pages-component/src/renderer/interactive-split.test.ts` — split resize event tests
- Modify: `packages/pages-component/src/model/types.test.ts` (or create if absent) — type assertion tests

**Interfaces:**
- Produces: `LayoutState` type (`{ splits, docks, panels }`), `PanelEntry` type (`{ typeName, props? }`), `pages-split-resize` CustomEvent with detail `{ componentId: string, ratios: number[] }`

#### Steps

- [ ] **Step 1: Write failing test — LayoutState type shape**

```typescript
// packages/pages-component/src/model/types.test.ts
import { describe, it, expect } from "vitest";
import type { LayoutState, PanelEntry } from "./types.js";

describe("LayoutState type", () => {
  it("accepts valid layout state", () => {
    const state: LayoutState = {
      splits: { "main-split": [60, 40] },
      docks: { "debug-panel": false },
      panels: { "editor": { typeName: "diff-viewer", props: { pathA: "a.md" } } },
    };
    expect(state.splits["main-split"]).toEqual([60, 40]);
    expect(state.docks["debug-panel"]).toBe(false);
    expect(state.panels["editor"]!.typeName).toBe("diff-viewer");
  });

  it("accepts empty layout state", () => {
    const state: LayoutState = { splits: {}, docks: {}, panels: {} };
    expect(Object.keys(state.splits)).toHaveLength(0);
  });

  it("accepts panel entry without props", () => {
    const entry: PanelEntry = { typeName: "terminal" };
    expect(entry.props).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- src/model/types.test.ts`
Expected: FAIL — `LayoutState` not exported

- [ ] **Step 3: Add LayoutState and PanelEntry types**

Add to `packages/pages-component/src/model/types.ts`:

```typescript
export interface PanelEntry {
  readonly typeName: string;
  readonly props?: Readonly<Record<string, unknown>>;
}

export interface LayoutState {
  readonly splits: Readonly<Record<string, readonly number[]>>;
  readonly docks: Readonly<Record<string, boolean>>;
  readonly panels: Readonly<Record<string, PanelEntry>>;
}
```

Add to `packages/pages-component/src/model/index.ts` in the first export block:

```typescript
export type {
  Component,
  AccessControl,
  GridPlacement,
  GridItem,
  PermissionContext,
  LayoutState,
  PanelEntry,
} from "./types.js";
```

- [ ] **Step 4: Run type test to verify it passes**

Run: `yarn workspace @casehubio/pages-component run test -- src/model/types.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing test — pages-split-resize event fires on mouseup**

```typescript
// packages/pages-component/src/renderer/interactive-split.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { renderComponent } from "./render.js";
import type { Component } from "../model/types.js";

describe("split resize event", () => {
  let container: HTMLElement;

  beforeEach(() => {
    container = document.createElement("div");
    document.body.appendChild(container);
  });

  afterEach(() => {
    container.remove();
  });

  function buildSplit(id?: string): Component {
    return {
      type: "split",
      ...(id ? { id } : {}),
      props: { direction: "horizontal", ratio: [60, 40] },
      slots: {
        "0": [{ type: "html", props: { content: "A" } }],
        "1": [{ type: "html", props: { content: "B" } }],
      },
    };
  }

  it("fires pages-split-resize with componentId and proportional ratios on mouseup", () => {
    renderComponent(container, buildSplit("main-split"));

    const handle = container.querySelector("[data-split-handle]") as HTMLElement;
    expect(handle).toBeTruthy();

    const events: Array<{ componentId: string; ratios: number[] }> = [];
    container.addEventListener("pages-split-resize", ((e: Event) => {
      events.push((e as CustomEvent).detail);
    }) as EventListener);

    // Simulate drag: mousedown on handle, mousemove, mouseup
    handle.dispatchEvent(new MouseEvent("mousedown", { clientX: 100, bubbles: true }));
    document.dispatchEvent(new MouseEvent("mousemove", { clientX: 120 }));
    document.dispatchEvent(new MouseEvent("mouseup"));

    expect(events).toHaveLength(1);
    expect(events[0]!.componentId).toBe("main-split");
    expect(events[0]!.ratios).toHaveLength(2);
    expect(events[0]!.ratios.every(r => typeof r === "number")).toBe(true);
  });

  it("uses auto-generated id when no explicit id provided", () => {
    renderComponent(container, buildSplit());

    const splitEl = container.querySelector("[data-component-type='split']") as HTMLElement;
    const autoId = splitEl.dataset.componentId!;

    const handle = container.querySelector("[data-split-handle]") as HTMLElement;
    const events: Array<{ componentId: string; ratios: number[] }> = [];
    container.addEventListener("pages-split-resize", ((e: Event) => {
      events.push((e as CustomEvent).detail);
    }) as EventListener);

    handle.dispatchEvent(new MouseEvent("mousedown", { clientX: 100, bubbles: true }));
    document.dispatchEvent(new MouseEvent("mouseup"));

    expect(events[0]!.componentId).toBe(autoId);
  });

  it("computes proportional ratios normalized to percentages", () => {
    renderComponent(container, buildSplit("test-split"));

    // Mock offsetWidth on slot containers
    const slots = container.querySelectorAll("[data-slot]");
    Object.defineProperty(slots[0], "offsetWidth", { value: 600, configurable: true });
    Object.defineProperty(slots[1], "offsetWidth", { value: 400, configurable: true });

    const handle = container.querySelector("[data-split-handle]") as HTMLElement;
    const events: Array<{ componentId: string; ratios: number[] }> = [];
    container.addEventListener("pages-split-resize", ((e: Event) => {
      events.push((e as CustomEvent).detail);
    }) as EventListener);

    handle.dispatchEvent(new MouseEvent("mousedown", { clientX: 100, bubbles: true }));
    document.dispatchEvent(new MouseEvent("mouseup"));

    expect(events[0]!.ratios).toEqual([60, 40]);
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- src/renderer/interactive-split.test.ts`
Expected: FAIL — no `pages-split-resize` event fired

- [ ] **Step 7: Implement observable split resize**

Modify `packages/pages-component/src/renderer/interactive.ts`:

In `wireSplit`, change the call to `attachDragHandler` to pass the component ID:

```typescript
// In wireSplit, replace the attachDragHandler call:
attachDragHandler(handle, i, direction, slotNames, panels, minSizes, container.dataset.componentId ?? "");
```

Update `attachDragHandler` signature and add mouseup event:

```typescript
function attachDragHandler(
  handle: HTMLElement,
  index: number,
  direction: string,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  minSizes: readonly number[] | undefined,
  componentId: string,
): void {
  handle.addEventListener("mousedown", (startEvent: MouseEvent) => {
    startEvent.preventDefault();
    const beforeName = slotNames[index]!;
    const afterName = slotNames[index + 1]!;
    const beforeMaybe = panels.get(beforeName);
    const afterMaybe = panels.get(afterName);
    if (!beforeMaybe || !afterMaybe) return;

    const before: HTMLElement = beforeMaybe;
    const after: HTMLElement = afterMaybe;

    const startPos = direction === "horizontal" ? startEvent.clientX : startEvent.clientY;
    const beforeSize = direction === "horizontal" ? before.offsetWidth : before.offsetHeight;
    const afterSize = direction === "horizontal" ? after.offsetWidth : after.offsetHeight;
    const totalSize = beforeSize + afterSize;
    const minBefore = minSizes?.[index] ?? 50;
    const minAfter = minSizes?.[index + 1] ?? 50;

    function onMouseMove(moveEvent: MouseEvent): void {
      const currentPos = direction === "horizontal" ? moveEvent.clientX : moveEvent.clientY;
      const delta = currentPos - startPos;
      let newBeforeSize = Math.max(minBefore, Math.min(totalSize - minAfter, beforeSize + delta));
      let newAfterSize = totalSize - newBeforeSize;
      if (newAfterSize < minAfter) {
        newAfterSize = minAfter;
        newBeforeSize = totalSize - newAfterSize;
      }
      before.style.flex = `0 0 ${String(newBeforeSize)}px`;
      after.style.flex = `0 0 ${String(newAfterSize)}px`;
    }

    function onMouseUp(): void {
      document.removeEventListener("mousemove", onMouseMove);
      document.removeEventListener("mouseup", onMouseUp);

      const ratios: number[] = [];
      for (const name of slotNames) {
        const panel = panels.get(name);
        if (panel) {
          const size = direction === "horizontal" ? panel.offsetWidth : panel.offsetHeight;
          ratios.push(size);
        }
      }
      const total = ratios.reduce((a, b) => a + b, 0);
      const normalized = total > 0 ? ratios.map(r => Math.round(r / total * 100)) : ratios;

      handle.dispatchEvent(new CustomEvent("pages-split-resize", {
        bubbles: true,
        composed: true,
        detail: { componentId, ratios: normalized },
      }));
    }

    document.addEventListener("mousemove", onMouseMove);
    document.addEventListener("mouseup", onMouseUp);
  });
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-component run test -- src/renderer/interactive-split.test.ts`
Expected: PASS

Run: `yarn workspace @casehubio/pages-component run test`
Expected: All existing tests still pass (no regressions)

- [ ] **Step 9: Commit**

```
git add packages/pages-component/src/model/types.ts packages/pages-component/src/model/types.test.ts packages/pages-component/src/model/index.ts packages/pages-component/src/renderer/interactive.ts packages/pages-component/src/renderer/interactive-split.test.ts
git commit -m "feat: LayoutState types and observable split resize event

Add LayoutState and PanelEntry types to pages-component model.
Fire pages-split-resize CustomEvent on mouseup with proportional
ratios and componentId. attachDragHandler gains componentId param.

Refs #76"
```

---

### Task 2: LayoutStore interface and localStorage adapter

The persistence contract and built-in implementation.

**Files:**
- Create: `packages/pages-runtime/src/layout-store.ts` — `LayoutStore` interface + `createLocalLayoutStore()`
- Create: `packages/pages-runtime/src/layout-store.test.ts` — adapter tests

**Interfaces:**
- Consumes: `LayoutState` from `@casehubio/pages-component`
- Produces: `LayoutStore` interface (`load`, `save`, `delete`), `createLocalLayoutStore(prefix?)` factory

#### Steps

- [ ] **Step 1: Write failing tests — LayoutStore localStorage adapter**

```typescript
// packages/pages-runtime/src/layout-store.test.ts
import { describe, it, expect, beforeEach, vi } from "vitest";
import { createLocalLayoutStore } from "./layout-store.js";
import type { LayoutState } from "@casehubio/pages-component/dist/model/types.js";

describe("createLocalLayoutStore", () => {
  beforeEach(() => {
    localStorage.clear();
  });

  const sampleState: LayoutState = {
    splits: { "main-split": [60, 40] },
    docks: { "sidebar": false },
    panels: { "editor": { typeName: "diff-viewer" } },
  };

  it("round-trips save and load", async () => {
    const store = createLocalLayoutStore();
    await store.save("test-key", sampleState);
    const loaded = await store.load("test-key");
    expect(loaded).toEqual(sampleState);
  });

  it("returns null for missing key", async () => {
    const store = createLocalLayoutStore();
    const loaded = await store.load("nonexistent");
    expect(loaded).toBeNull();
  });

  it("uses prefix for localStorage keys", async () => {
    const store = createLocalLayoutStore("my-prefix:");
    await store.save("key1", sampleState);
    expect(localStorage.getItem("my-prefix:key1")).not.toBeNull();
    expect(localStorage.getItem("pages-layout:key1")).toBeNull();
  });

  it("uses default prefix pages-layout:", async () => {
    const store = createLocalLayoutStore();
    await store.save("key1", sampleState);
    expect(localStorage.getItem("pages-layout:key1")).not.toBeNull();
  });

  it("delete removes entry", async () => {
    const store = createLocalLayoutStore();
    await store.save("key1", sampleState);
    await store.delete("key1");
    const loaded = await store.load("key1");
    expect(loaded).toBeNull();
  });

  it("delete on missing key is no-op", async () => {
    const store = createLocalLayoutStore();
    await expect(store.delete("nonexistent")).resolves.toBeUndefined();
  });

  it("returns null on corrupted JSON", async () => {
    const store = createLocalLayoutStore();
    localStorage.setItem("pages-layout:corrupt", "not-json{{{");
    const warnSpy = vi.spyOn(console, "warn").mockImplementation(() => {});
    const loaded = await store.load("corrupt");
    expect(loaded).toBeNull();
    expect(warnSpy).toHaveBeenCalled();
    warnSpy.mockRestore();
  });

  it("save catches QuotaExceededError and logs warning", async () => {
    const store = createLocalLayoutStore();
    const original = Storage.prototype.setItem;
    Storage.prototype.setItem = () => { throw new DOMException("quota", "QuotaExceededError"); };
    const warnSpy = vi.spyOn(console, "warn").mockImplementation(() => {});
    await expect(store.save("key1", sampleState)).resolves.toBeUndefined();
    expect(warnSpy).toHaveBeenCalled();
    warnSpy.mockRestore();
    Storage.prototype.setItem = original;
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- src/layout-store.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement LayoutStore and createLocalLayoutStore**

```typescript
// packages/pages-runtime/src/layout-store.ts
import type { LayoutState } from "@casehubio/pages-component/dist/model/types.js";

export interface LayoutStore {
  load(key: string): Promise<LayoutState | null>;
  save(key: string, state: LayoutState): Promise<void>;
  delete(key: string): Promise<void>;
}

export function createLocalLayoutStore(prefix = "pages-layout:"): LayoutStore {
  return {
    async load(key: string): Promise<LayoutState | null> {
      try {
        const raw = localStorage.getItem(prefix + key);
        if (raw === null) return null;
        return JSON.parse(raw) as LayoutState;
      } catch (err) {
        console.warn(`[pages] Failed to load layout "${key}":`, err);
        return null;
      }
    },

    async save(key: string, state: LayoutState): Promise<void> {
      try {
        localStorage.setItem(prefix + key, JSON.stringify(state));
      } catch (err) {
        console.warn(`[pages] Failed to save layout "${key}":`, err);
      }
    },

    async delete(key: string): Promise<void> {
      try {
        localStorage.removeItem(prefix + key);
      } catch (err) {
        console.warn(`[pages] Failed to delete layout "${key}":`, err);
      }
    },
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- src/layout-store.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add packages/pages-runtime/src/layout-store.ts packages/pages-runtime/src/layout-store.test.ts
git commit -m "feat: LayoutStore contract and localStorage adapter

LayoutStore interface with load/save/delete — all must-not-throw.
createLocalLayoutStore() uses localStorage with configurable prefix.
Graceful on QuotaExceededError and corrupted JSON.

Refs #76"
```

---

### Task 3: Layout state wiring in site.ts — state maps, event handlers, getter, restore, auto-save

The core integration — wiring LayoutState into the runtime lifecycle.

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` — `SiteOptions` fields, internal state maps, `pages-split-resize` handler with hidden-panel correction, `pages-dock-toggle` triggers debounced save, `layout` getter, post-render `applySavedSplitRatios`, lazy container restore via `pages-slot-change`, debounce timer cleanup on dispose
- Modify: `packages/pages-runtime/src/index.ts` — export new types
- Create: `packages/pages-runtime/src/layout-integration.test.ts` — integration tests for layout round-trip
- Modify: `packages/pages-runtime/src/workbench.test.ts` — add layout-specific integration tests

**Interfaces:**
- Consumes: `LayoutState`, `PanelEntry` from pages-component; `LayoutStore`, `createLocalLayoutStore` from Task 2; `pages-split-resize` event from Task 1; existing `dockState`, `registry`, `ComponentEntry`, `pages-dock-toggle` handler in site.ts
- Produces: `SiteOptions.layout`, `SiteOptions.layoutStore`, `SiteOptions.layoutKey`; `LiveSite.layout` getter; `applySavedSplitRatios()` internal function

#### Steps

- [ ] **Step 1: Write failing test — SiteOptions accepts layout fields**

```typescript
// packages/pages-runtime/src/layout-integration.test.ts
import { describe, it, expect, afterEach, vi, beforeEach } from "vitest";
import { loadSite } from "./site.js";
import { createLocalLayoutStore } from "./layout-store.js";
import type { Component } from "@casehubio/pages-component/dist/model/types.js";
import type { LayoutState } from "@casehubio/pages-component/dist/model/types.js";
import { registerPanel, clearPanelRegistry } from "./panel-registry.js";

describe("layout serialization", () => {
  let container: HTMLElement;

  beforeEach(() => {
    container = document.createElement("div");
    document.body.appendChild(container);
    localStorage.clear();
  });

  afterEach(() => {
    container.remove();
    clearPanelRegistry();
  });

  function buildWorkbench(): Component {
    return {
      type: "split",
      id: "main-split",
      props: { direction: "horizontal", ratio: [70, 30] },
      slots: {
        "0": [{ type: "html", props: { content: "Left" } }],
        "1": [{ type: "html", props: { content: "Right" } }],
      },
    };
  }

  it("site.layout returns empty state for fresh site", async () => {
    const site = await loadSite(container, buildWorkbench());
    const layout = site.layout;
    expect(layout.splits).toEqual({});
    expect(layout.docks).toEqual({});
    expect(layout.panels).toEqual({});
    site.dispose();
  });

  it("direct layout injection overrides split ratios", async () => {
    const savedLayout: LayoutState = {
      splits: { "main-split": [40, 60] },
      docks: {},
      panels: {},
    };
    const site = await loadSite(container, buildWorkbench(), { layout: savedLayout });

    const slots = container.querySelectorAll("[data-component-type='split'] > [data-slot]");
    expect((slots[0] as HTMLElement).style.flex).toBe("40");
    expect((slots[1] as HTMLElement).style.flex).toBe("60");

    site.dispose();
  });

  it("layout injection with ratio count mismatch is discarded", async () => {
    const savedLayout: LayoutState = {
      splits: { "main-split": [33, 33, 33] }, // 3-way ratio on 2-way split
      docks: {},
      panels: {},
    };
    const site = await loadSite(container, buildWorkbench(), { layout: savedLayout });

    // Should use component-tree defaults (70, 30) since mismatch
    const slots = container.querySelectorAll("[data-component-type='split'] > [data-slot]");
    expect((slots[0] as HTMLElement).style.flex).toBe("70");
    expect((slots[1] as HTMLElement).style.flex).toBe("30");

    site.dispose();
  });

  it("pages-split-resize event updates site.layout.splits", async () => {
    const site = await loadSite(container, buildWorkbench());
    expect(site.layout.splits).toEqual({});

    container.dispatchEvent(new CustomEvent("pages-split-resize", {
      bubbles: true,
      detail: { componentId: "main-split", ratios: [55, 45] },
    }));

    expect(site.layout.splits).toEqual({ "main-split": [55, 45] });

    site.dispose();
  });

  it("dock toggle updates site.layout.docks", async () => {
    const workbench: Component = {
      type: "split",
      id: "ws",
      props: { direction: "horizontal", ratio: [70, 30] },
      slots: {
        "0": [{ type: "html", props: { content: "Main" } }],
        "1": [{ type: "html", id: "sidebar", props: { content: "Side" } }],
      },
    };
    const site = await loadSite(container, workbench);

    container.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true,
      detail: { panelId: "sidebar", visible: false },
    }));

    expect(site.layout.docks["sidebar"]).toBe(false);

    site.dispose();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- src/layout-integration.test.ts`
Expected: FAIL — `site.layout` not defined, `SiteOptions.layout` not accepted

- [ ] **Step 3: Add SiteOptions fields and internal layout state maps**

In `packages/pages-runtime/src/site.ts`:

Add imports at top:
```typescript
import type { LayoutState, PanelEntry } from "@casehubio/pages-component/dist/model/types.js";
import type { LayoutStore } from "./layout-store.js";
import type { HostPanelProps } from "@casehubio/pages-component/dist/model/component-props.js";
```

Extend `SiteOptions`:
```typescript
export interface SiteOptions {
  readonly permissions?: PermissionContext;
  readonly fetch?: typeof globalThis.fetch;
  readonly baseUrl?: string;
  readonly providerConfig?: DataProviderConfig;
  readonly adapters?: Readonly<Record<string, SaveAdapter>>;
  readonly layout?: LayoutState;
  readonly layoutStore?: LayoutStore;
  readonly layoutKey?: string;
}
```

After the `dockState` declaration (around line 137), add:
```typescript
const splitRatios = new Map<string, readonly number[]>();
let layoutSaveTimer: ReturnType<typeof setTimeout> | undefined;
```

- [ ] **Step 4: Implement layout seeding in the init sequence**

After `const dockState = ...` and `const splitRatios = ...` block, before the render call, add the layout seeding logic:

```typescript
// Seed layout state from store or direct injection
let seedLayout: LayoutState | null = null;
if (options?.layoutStore && options?.layoutKey) {
  try {
    seedLayout = await options.layoutStore.load(options.layoutKey);
  } catch (err) {
    console.warn("[pages] Failed to load layout from store:", err);
  }
}
if (!seedLayout && options?.layout) {
  seedLayout = options.layout;
}
if (seedLayout) {
  for (const [id, ratios] of Object.entries(seedLayout.splits)) {
    splitRatios.set(id, ratios);
  }
  for (const [id, visible] of Object.entries(seedLayout.docks)) {
    dockState.set(id, visible);
  }
}
```

Note: this block must be placed BEFORE `renderComponent` but AFTER the event listeners are registered. The current code has event listeners first, then render. The seed block goes between them.

- [ ] **Step 5: Implement applySavedSplitRatios and post-render restore**

Add helper function inside `loadSite` (before the render call):

```typescript
function applySavedSplitRatios(scope: HTMLElement): void {
  for (const [componentId, ratios] of splitRatios) {
    const splitEl = scope.querySelector<HTMLElement>(`[data-component-id="${componentId}"]`);
    if (!splitEl) continue;
    const slots = splitEl.querySelectorAll<HTMLElement>(`:scope > [data-slot]`);
    if (ratios.length !== slots.length) continue;
    slots.forEach((slot, i) => {
      slot.style.flex = String(ratios[i]);
    });
  }
}
```

After the `renderComponent(target, root, ...)` call, add:
```typescript
applySavedSplitRatios(target);
```

- [ ] **Step 6: Implement pages-split-resize event handler with hidden panel correction**

Add event listener (alongside existing dock-toggle listener):

```typescript
target.addEventListener("pages-split-resize", ((e: Event) => {
  const { componentId, ratios } = (e as CustomEvent<{ componentId: string; ratios: number[] }>).detail;
  // Hidden panel correction: if ratio is 0 and we had a non-zero value, keep the old value
  const previous = splitRatios.get(componentId);
  const corrected = ratios.map((r, i) => {
    if (r === 0 && previous && previous[i] !== undefined && previous[i] !== 0) {
      return previous[i]!;
    }
    return r;
  });
  splitRatios.set(componentId, corrected);
  scheduleLayoutSave();
}), { signal: abortController.signal });
```

- [ ] **Step 7: Wire dock toggle to debounced layout save**

In the existing `pages-dock-toggle` handler, after `syncUrl("replaceState");`, add:
```typescript
scheduleLayoutSave();
```

- [ ] **Step 8: Implement debounced save with store guard**

Add inside `loadSite`:
```typescript
function scheduleLayoutSave(): void {
  if (!options?.layoutStore || !options?.layoutKey) return;
  if (layoutSaveTimer !== undefined) clearTimeout(layoutSaveTimer);
  layoutSaveTimer = setTimeout(() => {
    layoutSaveTimer = undefined;
    options!.layoutStore!.save(options!.layoutKey!, site.layout).catch(() => {});
  }, 500);
}
```

- [ ] **Step 9: Implement layout getter on LiveSite**

Add panel capture helper inside `loadSite`:
```typescript
function captureHostPanels(): Readonly<Record<string, PanelEntry>> {
  const result: Record<string, PanelEntry> = {};
  for (const [id, entry] of registry) {
    if (entry.component.type === "host-panel" && entry.hasExplicitId) {
      const hp = entry.component.props as HostPanelProps | undefined;
      if (hp?.typeName) {
        result[id] = hp.panelProps
          ? { typeName: hp.typeName, props: hp.panelProps }
          : { typeName: hp.typeName };
      }
    }
  }
  return Object.freeze(result);
}
```

Add `layout` getter to the `site` object:
```typescript
get layout(): LayoutState {
  return Object.freeze({
    splits: Object.freeze(Object.fromEntries(splitRatios)),
    docks: Object.freeze(Object.fromEntries(dockState)),
    panels: captureHostPanels(),
  });
},
```

- [ ] **Step 10: Add lazy container restore**

In the existing `pages-slot-change` handler, after the existing slot-change logic, add:
```typescript
// Restore saved split ratios in newly-rendered lazy content
const slotContainerId = (e as CustomEvent<SlotChangeDetail>).detail.containerId;
const containerEl = target.querySelector<HTMLElement>(`[data-component-id="${slotContainerId}"]`);
if (containerEl) {
  applySavedSplitRatios(containerEl);
}
```

- [ ] **Step 11: Add dispose cleanup for debounce timer**

In the `dispose()` method, add before the existing `saveTimers` cleanup:
```typescript
if (layoutSaveTimer !== undefined) {
  clearTimeout(layoutSaveTimer);
  layoutSaveTimer = undefined;
}
```

- [ ] **Step 12: Update LiveSite type export**

In `packages/pages-runtime/src/site.ts`, update `LiveSite` interface:
```typescript
export interface LiveSite extends Site {
  navigate(path: string): void;
  setTheme(theme: "light" | "dark" | PagesTheme): void;
  dispose(): void;
  readonly layout: LayoutState;
}
```

- [ ] **Step 13: Update exports in index.ts**

In `packages/pages-runtime/src/index.ts`, add:
```typescript
export type { LayoutStore } from "./layout-store.js";
export { createLocalLayoutStore } from "./layout-store.js";
export type { LayoutState, PanelEntry } from "@casehubio/pages-component/dist/model/types.js";
```

- [ ] **Step 14: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- src/layout-integration.test.ts`
Expected: PASS

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass (no regressions in workbench.test.ts or other tests)

Run: `yarn workspace @casehubio/pages-component run test`
Expected: All tests pass

- [ ] **Step 15: Write additional integration tests**

Add to `layout-integration.test.ts`:

```typescript
  describe("auto-persistence via LayoutStore", () => {
    it("auto-loads layout from store on init", async () => {
      const store = createLocalLayoutStore("test:");
      await store.save("ws", {
        splits: { "main-split": [40, 60] },
        docks: {},
        panels: {},
      });

      const site = await loadSite(container, buildWorkbench(), {
        layoutStore: store,
        layoutKey: "ws",
      });

      const slots = container.querySelectorAll("[data-component-type='split'] > [data-slot]");
      expect((slots[0] as HTMLElement).style.flex).toBe("40");
      expect((slots[1] as HTMLElement).style.flex).toBe("60");

      site.dispose();
    });

    it("auto-saves layout on split resize (debounced)", async () => {
      vi.useFakeTimers();
      const store = createLocalLayoutStore("test:");
      const site = await loadSite(container, buildWorkbench(), {
        layoutStore: store,
        layoutKey: "ws",
      });

      container.dispatchEvent(new CustomEvent("pages-split-resize", {
        bubbles: true,
        detail: { componentId: "main-split", ratios: [55, 45] },
      }));

      // Not saved yet (debounced)
      expect(await store.load("ws")).toBeNull();

      vi.advanceTimersByTime(500);

      const saved = await store.load("ws");
      expect(saved?.splits["main-split"]).toEqual([55, 45]);

      site.dispose();
      vi.useRealTimers();
    });

    it("store takes precedence over direct layout injection", async () => {
      const store = createLocalLayoutStore("test:");
      await store.save("ws", {
        splits: { "main-split": [25, 75] },
        docks: {},
        panels: {},
      });

      const site = await loadSite(container, buildWorkbench(), {
        layout: { splits: { "main-split": [50, 50] }, docks: {}, panels: {} },
        layoutStore: store,
        layoutKey: "ws",
      });

      const slots = container.querySelectorAll("[data-component-type='split'] > [data-slot]");
      expect((slots[0] as HTMLElement).style.flex).toBe("25");

      site.dispose();
    });

    it("layout injection used as fallback when store returns null", async () => {
      const store = createLocalLayoutStore("test:");
      // store is empty

      const site = await loadSite(container, buildWorkbench(), {
        layout: { splits: { "main-split": [45, 55] }, docks: {}, panels: {} },
        layoutStore: store,
        layoutKey: "ws",
      });

      const slots = container.querySelectorAll("[data-component-type='split'] > [data-slot]");
      expect((slots[0] as HTMLElement).style.flex).toBe("45");

      site.dispose();
    });

    it("dispose cancels pending layout save", async () => {
      vi.useFakeTimers();
      const store = createLocalLayoutStore("test:");
      const saveSpy = vi.spyOn(store, "save");

      const site = await loadSite(container, buildWorkbench(), {
        layoutStore: store,
        layoutKey: "ws",
      });

      container.dispatchEvent(new CustomEvent("pages-split-resize", {
        bubbles: true,
        detail: { componentId: "main-split", ratios: [55, 45] },
      }));

      site.dispose();
      vi.advanceTimersByTime(1000);

      expect(saveSpy).not.toHaveBeenCalled();

      vi.useRealTimers();
    });

    it("no save when store not configured", async () => {
      vi.useFakeTimers();
      const site = await loadSite(container, buildWorkbench());

      container.dispatchEvent(new CustomEvent("pages-split-resize", {
        bubbles: true,
        detail: { componentId: "main-split", ratios: [55, 45] },
      }));

      vi.advanceTimersByTime(1000);

      // No crash, no localStorage writes
      expect(localStorage.length).toBe(0);

      site.dispose();
      vi.useRealTimers();
    });
  });

  describe("panel capture", () => {
    it("captures host-panel entries with explicit IDs", async () => {
      customElements.define("test-layout-panel", class extends HTMLElement {});
      registerPanel("test-p", "test-layout-panel");

      const workbench: Component = {
        type: "split",
        id: "ws",
        props: { direction: "horizontal", ratio: [50, 50] },
        slots: {
          "0": [{ type: "host-panel", id: "my-panel", props: { typeName: "test-p", panelProps: { foo: 1 } } }],
          "1": [{ type: "html", props: { content: "Right" } }],
        },
      };

      const site = await loadSite(container, workbench);
      const panels = site.layout.panels;

      expect(panels["my-panel"]).toBeDefined();
      expect(panels["my-panel"]!.typeName).toBe("test-p");
      expect(panels["my-panel"]!.props).toEqual({ foo: 1 });

      site.dispose();
    });
  });

  describe("hidden dock panel correction", () => {
    it("preserves last known ratio when panel hidden during resize", async () => {
      const site = await loadSite(container, buildWorkbench());

      // First resize: set known ratios
      container.dispatchEvent(new CustomEvent("pages-split-resize", {
        bubbles: true,
        detail: { componentId: "main-split", ratios: [60, 40] },
      }));

      // Resize with one panel hidden (measures as 0)
      container.dispatchEvent(new CustomEvent("pages-split-resize", {
        bubbles: true,
        detail: { componentId: "main-split", ratios: [100, 0] },
      }));

      // Hidden panel should keep its last known ratio
      expect(site.layout.splits["main-split"]).toEqual([100, 40]);

      site.dispose();
    });
  });
```

- [ ] **Step 16: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 17: Commit**

```
git add packages/pages-runtime/src/site.ts packages/pages-runtime/src/index.ts packages/pages-runtime/src/layout-integration.test.ts
git commit -m "feat: layout state wiring — getter, restore, auto-save

Wire LayoutState into site.ts: SiteOptions.layout for direct injection,
SiteOptions.layoutStore + layoutKey for managed persistence.
LiveSite.layout getter captures splits, docks, panels.
Post-render split ratio restore with ratio count guard.
Lazy container restore via pages-slot-change.
Hidden dock panel ratio correction.
Debounced auto-save with store guard and dispose cleanup.

Closes #76"
```

---

### Post-Implementation Checklist

After all tasks complete:

- [ ] Run full monorepo typecheck: `yarn typecheck` (expect pre-existing errors in data-pipeline tests only, no new errors)
- [ ] Run all component tests: `yarn workspace @casehubio/pages-component run test`
- [ ] Run all runtime tests: `yarn workspace @casehubio/pages-runtime run test`
- [ ] Build all packages: `yarn build:packages`
- [ ] Invoke `superpowers:requesting-code-review` before final commit
- [ ] Invoke `implementation-doc-sync` after committing
- [ ] Update PLATFORM.md with approved wording (cross-repo)
- [ ] File GitHub issues for any deferred concerns from code review
