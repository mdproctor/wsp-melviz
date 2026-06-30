# Workbench Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement three composable workbench primitives (split, dockBar, hostPanel) plus unified event bus extension, covering epic #64 and sub-issues #65–#69.

**Architecture:** Three new component types added to the existing recursive component model. Split is a flex-based resizable layout (decoration on existing columns/rows). DockBar is an icon strip that toggles panel visibility by ID. HostPanel mounts registered external Web Components. The event bus extends the existing WebSocket data pipeline with an `event` op type and adds a `pages-event` DOM custom event. `app-grid` is removed.

**Tech Stack:** TypeScript 5, Vitest (jsdom), Yarn 4.10 workspaces

**Spec:** `docs/superpowers/specs/2026-06-30-workbench-primitives-design.md`

## Global Constraints

- All tests use Vitest with `globals: true` (no import needed for describe/it/expect), `environment: "jsdom"`, files named `*.test.ts`
- Run tests per-package: `yarn workspace @casehubio/pages-component run test` (etc.)
- Build packages before testing runtime/ui: `yarn build:packages`
- Commits reference issue: `Refs #64`
- Breaking changes are acceptable — no backward compatibility shims
- TDD: write failing test → verify failure → implement → verify pass → commit

---

### Task 1: Component Model — Types, Type Guards, app-grid Removal

**Files:**
- Modify: `packages/pages-component/src/model/component-props.ts`
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Test: `packages/pages-component/src/model/type-guards.test.ts`

**Interfaces:**
- Consumes: existing `Component`, `ComponentTypeRegistry`, type guard patterns
- Produces: `SplitProps`, `DockBarProps`, `DockItem`, `HostPanelProps`, `isSplit()`, `isDockBar()`, `isHostPanel()` type guards; removes `AppGridProps`, `isAppGrid()`

- [ ] **Step 1: Write failing tests for new type guards**

Add to `packages/pages-component/src/model/type-guards.test.ts`:

```typescript
describe("isSplit", () => {
  it("returns true for split component", () => {
    const c: Component = { type: "split", props: { direction: "horizontal" } };
    expect(isSplit(c)).toBe(true);
  });
  it("returns false for non-split", () => {
    expect(isSplit({ type: "rows" })).toBe(false);
  });
});

describe("isDockBar", () => {
  it("returns true for dock-bar component", () => {
    const c: Component = { type: "dock-bar", props: { orientation: "vertical", items: [] } };
    expect(isDockBar(c)).toBe(true);
  });
});

describe("isHostPanel", () => {
  it("returns true for host-panel component", () => {
    const c: Component = { type: "host-panel", props: { typeName: "diff-viewer" } };
    expect(isHostPanel(c)).toBe(true);
  });
});

describe("isAppGrid — removed", () => {
  it("isAppGrid is no longer exported", () => {
    expect(typeof isAppGrid).toBe("undefined");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: FAIL — `isSplit`, `isDockBar`, `isHostPanel` not defined; `isAppGrid` still exported

- [ ] **Step 3: Add prop types to component-props.ts**

Add to `packages/pages-component/src/model/component-props.ts`:

```typescript
export interface SplitProps {
  readonly direction: "horizontal" | "vertical";
  readonly ratio?: readonly number[];
  readonly minSizes?: readonly number[];
}

export interface DockItem {
  readonly icon: string;
  readonly label: string;
  readonly panelId: string;
  readonly defaultOpen?: boolean;
}

export interface DockBarProps {
  readonly orientation: "vertical" | "horizontal";
  readonly items: readonly DockItem[];
}

export interface HostPanelProps {
  readonly typeName: string;
  readonly panelProps?: Readonly<Record<string, unknown>>;
}
```

Remove: `export type AppGridProps = Record<string, never>;`

- [ ] **Step 4: Update type guards**

In `packages/pages-component/src/model/type-guards.ts`:

Add to `ComponentTypeRegistry`:
```typescript
  "split": SplitProps;
  "dock-bar": DockBarProps;
  "host-panel": HostPanelProps;
```

Remove from `ComponentTypeRegistry`:
```typescript
  "app-grid": AppGridProps;
```

Add type guard functions:
```typescript
export function isSplit(c: Component): c is TypedComponent<"split"> {
  return c.type === "split";
}

export function isDockBar(c: Component): c is TypedComponent<"dock-bar"> {
  return c.type === "dock-bar";
}

export function isHostPanel(c: Component): c is TypedComponent<"host-panel"> {
  return c.type === "host-panel";
}
```

Remove: `isAppGrid` function

Update imports: add `SplitProps`, `DockBarProps`, `HostPanelProps`; remove `AppGridProps`

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(pages-component): add split, dockBar, hostPanel types; remove app-grid

Refs #64
```

---

### Task 2: Split Rendering — Layout CSS + Resize Interactivity

**Files:**
- Modify: `packages/pages-component/src/renderer/layout.ts`
- Modify: `packages/pages-component/src/renderer/interactive.ts`
- Modify: `packages/pages-component/src/renderer/render.ts` (LAZY_TYPES)
- Test: `packages/pages-component/src/renderer/layout.test.ts`
- Test: `packages/pages-component/src/renderer/interactive.test.ts`

**Interfaces:**
- Consumes: `isSplit`, `SplitProps` from Task 1; existing `LAYOUT_TYPES`, `applyLayoutCSS`, `wireInteractivity`, `LAZY_TYPES` patterns
- Produces: split layout CSS (flex container), split interactivity (drag handles, flex ratios on slot containers, resize handlers)

- [ ] **Step 1: Write failing layout CSS tests**

Add to `packages/pages-component/src/renderer/layout.test.ts`:

```typescript
describe("split layout", () => {
  it("split is a layout type", () => {
    expect(isLayoutType("split")).toBe(true);
  });

  it("app-grid is no longer a layout type", () => {
    expect(isLayoutType("app-grid")).toBe(false);
  });

  it("applies horizontal split CSS — flex row", () => {
    const el = document.createElement("div");
    const component: Component = { type: "split", props: { direction: "horizontal" } };
    applyLayoutCSS(el, component);
    expect(el.style.display).toBe("flex");
    expect(el.style.flexDirection).toBe("row");
  });

  it("applies vertical split CSS — flex column", () => {
    const el = document.createElement("div");
    const component: Component = { type: "split", props: { direction: "vertical" } };
    applyLayoutCSS(el, component);
    expect(el.style.display).toBe("flex");
    expect(el.style.flexDirection).toBe("column");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: FAIL — "split" not in LAYOUT_TYPES, no case in switch

- [ ] **Step 3: Implement split layout CSS**

In `packages/pages-component/src/renderer/layout.ts`:

Add "split" to `LAYOUT_TYPES` set. Remove "app-grid" from `LAYOUT_TYPES`.

Add case to `applyLayoutCSS`:
```typescript
    case "split": {
      element.style.display = "flex";
      const direction = (component.props as { direction?: string } | undefined)?.direction;
      element.style.flexDirection = direction === "vertical" ? "column" : "row";
      break;
    }
```

Remove the `case "app-grid":` block.

- [ ] **Step 4: Run layout tests to verify they pass**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: PASS

- [ ] **Step 5: Write failing interactivity tests**

Add to `packages/pages-component/src/renderer/interactive.test.ts`:

```typescript
describe("wireSplit", () => {
  function makeSplitSlots(count: number): {
    container: HTMLDivElement;
    panels: Map<string, HTMLDivElement>;
    slotNames: string[];
  } {
    const container = document.createElement("div");
    container.dataset.componentType = "split";
    container.dataset.componentProps = JSON.stringify({ direction: "horizontal", ratio: [60, 40] });
    const panels = new Map<string, HTMLDivElement>();
    const slotNames: string[] = [];
    for (let i = 0; i < count; i++) {
      const key = String(i);
      const panel = document.createElement("div");
      panel.dataset.slot = key;
      // Add a child component inside each slot
      const child = document.createElement("div");
      child.dataset.componentType = "host-panel";
      child.dataset.componentId = `panel-${key}`;
      panel.appendChild(child);
      container.appendChild(panel);
      panels.set(key, panel);
      slotNames.push(key);
    }
    return { container, panels, slotNames };
  }

  it("applies flex ratios to slot containers", () => {
    const { container, panels, slotNames } = makeSplitSlots(2);
    wireInteractivity(container, "split", slotNames, panels);
    expect(panels.get("0")!.style.flex).toBe("60");
    expect(panels.get("1")!.style.flex).toBe("40");
  });

  it("defaults to equal ratios when no ratio prop", () => {
    const { container, panels, slotNames } = makeSplitSlots(3);
    container.dataset.componentProps = JSON.stringify({ direction: "horizontal" });
    wireInteractivity(container, "split", slotNames, panels);
    expect(panels.get("0")!.style.flex).toBe("1");
    expect(panels.get("1")!.style.flex).toBe("1");
    expect(panels.get("2")!.style.flex).toBe("1");
  });

  it("inserts drag handles between slot containers", () => {
    const { container, panels, slotNames } = makeSplitSlots(3);
    wireInteractivity(container, "split", slotNames, panels);
    const handles = container.querySelectorAll("[data-split-handle]");
    expect(handles).toHaveLength(2);
  });

  it("drag handle has correct cursor for horizontal split", () => {
    const { container, panels, slotNames } = makeSplitSlots(2);
    wireInteractivity(container, "split", slotNames, panels);
    const handle = container.querySelector("[data-split-handle]") as HTMLElement;
    expect(handle.style.cursor).toBe("col-resize");
  });

  it("drag handle has correct cursor for vertical split", () => {
    const { container, panels, slotNames } = makeSplitSlots(2);
    container.dataset.componentProps = JSON.stringify({ direction: "vertical", ratio: [50, 50] });
    wireInteractivity(container, "split", slotNames, panels);
    const handle = container.querySelector("[data-split-handle]") as HTMLElement;
    expect(handle.style.cursor).toBe("row-resize");
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: FAIL — no "split" case in wireInteractivity

- [ ] **Step 7: Implement split interactivity**

In `packages/pages-component/src/renderer/interactive.ts`, add the `wireSplit` function and wire it into the `wireInteractivity` switch:

```typescript
case "split":
  wireSplit(container, slotNames, panels, doc);
  break;
```

```typescript
function wireSplit(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
): void {
  const propsStr = container.dataset.componentProps;
  const props = propsStr ? JSON.parse(propsStr) as { direction?: string; ratio?: number[]; minSizes?: number[] } : {};
  const direction = props.direction ?? "horizontal";
  const ratio = props.ratio;
  const minSizes = props.minSizes;

  // Apply flex ratios to slot containers
  for (let i = 0; i < slotNames.length; i++) {
    const name = slotNames[i]!;
    const panel = panels.get(name);
    if (panel) {
      panel.style.flex = String(ratio?.[i] ?? 1);
      panel.style.overflow = "hidden";
      if (minSizes?.[i] !== undefined) {
        const prop = direction === "horizontal" ? "minWidth" : "minHeight";
        panel.style[prop] = `${String(minSizes[i])}px`;
      }
    }
  }

  // Insert drag handles between slot containers
  for (let i = 0; i < slotNames.length - 1; i++) {
    const currentName = slotNames[i]!;
    const currentPanel = panels.get(currentName);
    if (!currentPanel) continue;

    const handle = doc.createElement("div");
    handle.dataset.splitHandle = String(i);
    handle.style.flexShrink = "0";
    handle.style.cursor = direction === "horizontal" ? "col-resize" : "row-resize";
    if (direction === "horizontal") {
      handle.style.width = "6px";
    } else {
      handle.style.height = "6px";
    }
    handle.style.background = "var(--pages-border, #e0e0e0)";
    handle.style.userSelect = "none";

    currentPanel.insertAdjacentElement("afterend", handle);

    attachDragHandler(handle, i, direction, slotNames, panels, minSizes);
  }
}

function attachDragHandler(
  handle: HTMLElement,
  index: number,
  direction: string,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  minSizes: readonly number[] | undefined,
): void {
  handle.addEventListener("mousedown", (startEvent: MouseEvent) => {
    startEvent.preventDefault();
    const beforeName = slotNames[index]!;
    const afterName = slotNames[index + 1]!;
    const before = panels.get(beforeName);
    const after = panels.get(afterName);
    if (!before || !after) return;

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
    }

    document.addEventListener("mousemove", onMouseMove);
    document.addEventListener("mouseup", onMouseUp);
  });
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: PASS

- [ ] **Step 9: Commit**

```
feat(pages-component): split rendering with flex layout and drag handles

Adds split to LAYOUT_TYPES, applyLayoutCSS (flex container), and
wireInteractivity (flex ratios on slot containers, drag handles with
mousedown/mousemove/mouseup resize, minSize enforcement).
Removes app-grid from layout pipeline.

Refs #64, Refs #66
```

---

### Task 3: HostPanel Lifecycle — Registration + Activation

**Files:**
- Create: `packages/pages-runtime/src/panel-registry.ts`
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/index.ts`
- Test: `packages/pages-runtime/src/panel-registry.test.ts`
- Test: `packages/pages-runtime/src/activation.test.ts` (new or add to existing)

**Interfaces:**
- Consumes: `HostPanelProps` from Task 1; existing activation callback pattern from `activation.ts`
- Produces: `registerPanel(typeName, tagName)` API; `host-panel` activation case in `createActivationCallback`

- [ ] **Step 1: Write failing panel registry tests**

Create `packages/pages-runtime/src/panel-registry.test.ts`:

```typescript
import { registerPanel, lookupPanel, clearPanelRegistry } from "./panel-registry.js";

describe("panel-registry", () => {
  afterEach(() => { clearPanelRegistry(); });

  it("registers and looks up a panel type", () => {
    registerPanel("diff-viewer", "drafthouse-diff");
    expect(lookupPanel("diff-viewer")).toBe("drafthouse-diff");
  });

  it("returns undefined for unregistered type", () => {
    expect(lookupPanel("unknown")).toBeUndefined();
  });

  it("overwrites on duplicate registration", () => {
    registerPanel("diff-viewer", "old-tag");
    registerPanel("diff-viewer", "new-tag");
    expect(lookupPanel("diff-viewer")).toBe("new-tag");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement panel registry**

Create `packages/pages-runtime/src/panel-registry.ts`:

```typescript
const registry = new Map<string, string>();

export function registerPanel(typeName: string, tagName: string): void {
  registry.set(typeName, tagName);
}

export function lookupPanel(typeName: string): string | undefined {
  return registry.get(typeName);
}

export function clearPanelRegistry(): void {
  registry.clear();
}
```

- [ ] **Step 4: Run registry tests**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 5: Write failing activation tests for host-panel**

Create or add to `packages/pages-runtime/src/activation.test.ts`:

```typescript
import { createActivationCallback } from "./activation.js";
import { registerPanel, clearPanelRegistry } from "./panel-registry.js";
import { buildPagePathMap } from "./page-paths.js";
import type { Component } from "@casehubio/pages-component/dist/model/types.js";

describe("host-panel activation", () => {
  afterEach(() => { clearPanelRegistry(); });

  function activate(component: Component): HTMLElement {
    const el = document.createElement("div");
    el.dataset.componentId = "test-panel";
    el.dataset.componentType = "host-panel";
    const registry = new Map();
    const pagePathMap = new Map();
    const callback = createActivationCallback(registry, pagePathMap);
    callback(el, component);
    return el;
  }

  it("mounts registered Web Component into container", () => {
    customElements.define("test-wc", class extends HTMLElement {});
    registerPanel("test-type", "test-wc");
    const el = activate({ type: "host-panel", props: { typeName: "test-type" } });
    expect(el.querySelector("test-wc")).toBeTruthy();
  });

  it("calls configure(props) before appendChild", () => {
    const configureOrder: string[] = [];
    customElements.define("test-cfg", class extends HTMLElement {
      configure(props: Record<string, unknown>) {
        configureOrder.push("configure");
      }
      connectedCallback() {
        configureOrder.push("connected");
      }
    });
    registerPanel("cfg-type", "test-cfg");
    activate({
      type: "host-panel",
      props: { typeName: "cfg-type", panelProps: { doc: "abc" } },
    });
    expect(configureOrder).toEqual(["configure", "connected"]);
  });

  it("renders error placeholder for unregistered type", () => {
    const el = activate({ type: "host-panel", props: { typeName: "missing" } });
    expect(el.textContent).toContain("Unknown panel type");
    expect(el.querySelector("*[data-component-type]")).toBeNull();
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: FAIL — no host-panel case in activation callback

- [ ] **Step 7: Implement host-panel activation**

In `packages/pages-runtime/src/activation.ts`, add after the `alert` handler block (before the final comment):

```typescript
    if (component.type === "host-panel" && component.props) {
      const { typeName, panelProps } = component.props as { typeName?: string; panelProps?: Record<string, unknown> };
      if (!typeName) return;

      const { lookupPanel } = await import("./panel-registry.js");
      const tagName = lookupPanel(typeName);
      if (!tagName) {
        el.textContent = `Unknown panel type: ${typeName}`;
        console.warn(`hostPanel: unregistered type "${typeName}"`);
        return;
      }

      const panel = document.createElement(tagName);
      if (typeof (panel as Record<string, unknown>).configure === "function") {
        (panel as unknown as { configure: (p: Record<string, unknown>) => void }).configure(panelProps ?? {});
      }
      el.appendChild(panel);
      return;
    }
```

Note: use a static import instead of dynamic import — add `import { lookupPanel } from "./panel-registry.js";` at the top of the file.

- [ ] **Step 8: Export registerPanel from package index**

In `packages/pages-runtime/src/index.ts`, add:

```typescript
export { registerPanel } from "./panel-registry.js";
```

- [ ] **Step 9: Build and run tests**

Run: `yarn build:packages && yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 10: Commit**

```
feat(pages-runtime): hostPanel lifecycle — registerPanel + activation

Mounts registered external Web Components into the pages component tree.
configure(props) called before appendChild per GE-20260617-0b0dba.
Fail-soft with visible error placeholder for unregistered types.

Refs #64, Refs #68
```

---

### Task 4: DockBar Rendering + Toggle + URL State

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts` (dock-bar case)
- Modify: `packages/pages-runtime/src/site.ts` (pages-dock-toggle handler, dock state)
- Modify: `packages/pages-runtime/src/url.ts` (dock serialization)
- Modify: `packages/pages-ui/src/model/page-types.ts` (DeepLink.dock field)
- Test: `packages/pages-runtime/src/activation.test.ts` (dock-bar tests)
- Test: `packages/pages-runtime/src/site.test.ts` (toggle integration tests)
- Test: `packages/pages-runtime/src/url.test.ts` (dock URL tests)

**Interfaces:**
- Consumes: `DockBarProps`, `DockItem` from Task 1; split interactivity from Task 2; activation callback pattern from Task 3
- Produces: dock-bar activation (icon buttons + click handlers), `pages-dock-toggle` event handling in runtime, dock state management (in-memory Map), URL dock serialization/deserialization, split collapse cascade

- [ ] **Step 1: Write failing dock-bar activation tests**

Add to `packages/pages-runtime/src/activation.test.ts`:

```typescript
describe("dock-bar activation", () => {
  function activate(component: Component): HTMLElement {
    const el = document.createElement("div");
    el.dataset.componentId = "dock-1";
    el.dataset.componentType = "dock-bar";
    const registry = new Map();
    const pagePathMap = new Map();
    const callback = createActivationCallback(registry, pagePathMap);
    callback(el, component);
    return el;
  }

  it("renders icon buttons from items", () => {
    const el = activate({
      type: "dock-bar",
      props: {
        orientation: "vertical",
        items: [
          { icon: "📁", label: "Explorer", panelId: "explorer", defaultOpen: true },
          { icon: "🔍", label: "Search", panelId: "search" },
        ],
      },
    });
    const buttons = el.querySelectorAll("button[data-dock-panel-id]");
    expect(buttons).toHaveLength(2);
    expect((buttons[0] as HTMLElement).dataset.dockPanelId).toBe("explorer");
    expect((buttons[0] as HTMLElement).dataset.active).toBeDefined();
    expect((buttons[1] as HTMLElement).dataset.active).toBeUndefined();
  });

  it("dispatches pages-dock-toggle on click", () => {
    const el = activate({
      type: "dock-bar",
      props: {
        orientation: "vertical",
        items: [{ icon: "📁", label: "Explorer", panelId: "explorer", defaultOpen: true }],
      },
    });
    const events: Array<{ panelId: string; visible: boolean }> = [];
    el.addEventListener("pages-dock-toggle", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    const button = el.querySelector("button[data-dock-panel-id]") as HTMLElement;
    button.click();
    expect(events).toHaveLength(1);
    expect(events[0]).toEqual({ panelId: "explorer", visible: false });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn build:packages && yarn workspace @casehubio/pages-runtime run test`
Expected: FAIL — no dock-bar case in activation

- [ ] **Step 3: Implement dock-bar activation**

In `packages/pages-runtime/src/activation.ts`, add a dock-bar case (after the host-panel case):

```typescript
    if (component.type === "dock-bar" && component.props) {
      const { orientation, items } = component.props as {
        orientation?: string;
        items?: Array<{ icon: string; label: string; panelId: string; defaultOpen?: boolean }>;
      };
      if (!items) return;

      el.style.display = "flex";
      el.style.flexDirection = orientation === "horizontal" ? "row" : "column";
      el.style.gap = "2px";
      el.style.padding = "4px";

      for (const item of items) {
        const button = document.createElement("button");
        button.dataset.dockPanelId = item.panelId;
        button.title = item.label;
        button.textContent = item.icon;
        button.style.border = "none";
        button.style.background = "transparent";
        button.style.cursor = "pointer";
        button.style.padding = "6px";
        button.style.borderRadius = "var(--pages-radius, 4px)";
        button.style.fontSize = "16px";

        if (item.defaultOpen) {
          button.dataset.active = "";
        }

        button.addEventListener("click", () => {
          const isActive = button.dataset.active !== undefined;
          if (isActive) {
            delete button.dataset.active;
          } else {
            button.dataset.active = "";
          }
          el.dispatchEvent(new CustomEvent("pages-dock-toggle", {
            bubbles: true,
            composed: true,
            detail: { panelId: item.panelId, visible: !isActive },
          }));
        });

        el.appendChild(button);
      }
      return;
    }
```

- [ ] **Step 4: Run activation tests**

Run: `yarn build:packages && yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 5: Write failing URL dock serialization tests**

Add to `packages/pages-runtime/src/url.test.ts`:

```typescript
describe("dock state serialization", () => {
  it("serializes dock state to URL", () => {
    const url = serializeToUrl({
      page: "Home",
      dock: { explorer: "open", search: "closed" },
    });
    expect(url).toContain("dock=");
    expect(url).toContain("explorer:open");
    expect(url).toContain("search:closed");
  });

  it("parses dock state from URL", () => {
    const link = parseFromUrl("#/page/Home?dock=explorer:open,search:closed");
    expect(link.dock).toEqual({ explorer: "open", search: "closed" });
  });

  it("roundtrips dock state", () => {
    const original = {
      page: "Home",
      dock: { debate: "open", review: "closed" },
    };
    const url = serializeToUrl(original);
    const parsed = parseFromUrl(url);
    expect(parsed.dock).toEqual(original.dock);
  });
});
```

- [ ] **Step 6: Run URL tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: FAIL — `dock` not on DeepLink type

- [ ] **Step 7: Extend DeepLink type with dock field**

In `packages/pages-ui/src/model/page-types.ts`, add to the `DeepLink` interface:

```typescript
  readonly dock?: Readonly<Record<string, "open" | "closed">>;
```

- [ ] **Step 8: Implement dock URL serialization/deserialization**

In `packages/pages-runtime/src/url.ts`:

Add dock serialization to `serializeToUrl` (after `textFilter` block):

```typescript
  if (link.dock) {
    const entries = Object.entries(link.dock);
    if (entries.length > 0) {
      const dockStr = entries
        .map(([id, state]) => `${encodeURIComponent(id)}:${state}`)
        .join(",");
      params.push(`dock=${dockStr}`);
    }
  }
```

Add dock parsing to `parseFromUrl` (after `tfStr` block):

```typescript
  let dock: Record<string, "open" | "closed"> | undefined;
```

And in the params parsing section:

```typescript
    const dockStr = params.get("dock");
    if (dockStr) {
      dock = {};
      for (const entry of dockStr.split(",")) {
        const colonIdx = entry.indexOf(":");
        if (colonIdx === -1) continue;
        const id = decodeURIComponent(entry.substring(0, colonIdx));
        const state = entry.substring(colonIdx + 1);
        if (state === "open" || state === "closed") {
          dock[id] = state;
        }
      }
    }
```

Add to the return object:

```typescript
    ...(dock ? { dock } : {}),
```

- [ ] **Step 9: Write failing toggle integration test**

Add to `packages/pages-runtime/src/site.test.ts`:

```typescript
describe("dock toggle integration", () => {
  it("pages-dock-toggle hides targeted panel", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const tree: Component = {
      type: "rows",
      slots: {
        default: [
          {
            type: "split",
            props: { direction: "horizontal", ratio: [50, 50] },
            slots: {
              "0": [{ type: "html", props: { content: "Center" } }],
              "1": [{ type: "html", id: "side", props: { content: "Side" } }],
            },
          },
        ],
      },
    };

    const site = await loadSite(target, tree);

    const sideEl = target.querySelector('[data-component-id="side"]')!;
    expect(sideEl).toBeTruthy();
    const slotContainer = sideEl.closest("[data-slot]") as HTMLElement;
    expect(slotContainer.style.display).not.toBe("none");

    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true,
      composed: true,
      detail: { panelId: "side", visible: false },
    }));

    expect(slotContainer.style.display).toBe("none");

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 10: Run test to verify it fails**

Run: `yarn build:packages && yarn workspace @casehubio/pages-runtime run test`
Expected: FAIL — no pages-dock-toggle handler in site.ts

- [ ] **Step 11: Implement dock toggle handler in site.ts**

In `packages/pages-runtime/src/site.ts`, add after the existing event listeners (in the event delegation section):

```typescript
  const dockState = new Map<string, boolean>();

  target.addEventListener("pages-dock-toggle", ((e: Event) => {
    const { panelId, visible } = (e as CustomEvent<{ panelId: string; visible: boolean }>).detail;
    dockState.set(panelId, visible);

    const panelEl = target.querySelector<HTMLElement>(`[data-component-id="${CSS.escape(panelId)}"]`);
    if (!panelEl) return;

    const slotContainer = panelEl.closest<HTMLElement>("[data-slot]");
    if (!slotContainer) return;

    if (visible) {
      slotContainer.style.display = "";
      // Show adjacent drag handle
      const adjacentHandle = slotContainer.nextElementSibling as HTMLElement | null;
      if (adjacentHandle?.dataset.splitHandle !== undefined) {
        adjacentHandle.style.display = "";
      }
      const prevHandle = slotContainer.previousElementSibling as HTMLElement | null;
      if (prevHandle?.dataset.splitHandle !== undefined) {
        prevHandle.style.display = "";
      }
      // Restore parent split if it was collapsed
      const parentSplit = slotContainer.closest<HTMLElement>('[data-component-type="split"]');
      if (parentSplit && parentSplit.style.display === "none") {
        parentSplit.style.display = "";
      }
    } else {
      slotContainer.style.display = "none";
      // Hide adjacent drag handle
      const adjacentHandle = slotContainer.nextElementSibling as HTMLElement | null;
      if (adjacentHandle?.dataset.splitHandle !== undefined) {
        adjacentHandle.style.display = "none";
      }
      const prevHandle = slotContainer.previousElementSibling as HTMLElement | null;
      if (prevHandle?.dataset.splitHandle !== undefined && prevHandle.nextElementSibling === slotContainer) {
        prevHandle.style.display = "none";
      }
      // Collapse parent split if all children hidden
      const parentSplit = slotContainer.closest<HTMLElement>('[data-component-type="split"]');
      if (parentSplit) {
        const slotChildren = parentSplit.querySelectorAll<HTMLElement>(":scope > [data-slot]");
        const allHidden = Array.from(slotChildren).every(s => s.style.display === "none");
        if (allHidden) {
          parentSplit.style.display = "none";
        }
      }
    }

    syncUrl("replaceState");
  }), { signal: abortController.signal });
```

Add dock state to `syncUrl`:
```typescript
  function deriveDockState(): Readonly<Record<string, "open" | "closed">> | undefined {
    if (dockState.size === 0) return undefined;
    const result: Record<string, "open" | "closed"> = {};
    for (const [id, visible] of dockState) {
      result[id] = visible ? "open" : "closed";
    }
    return result;
  }
```

Update the `link` construction in `syncUrl` to include:
```typescript
      ...(deriveDockState() ? { dock: deriveDockState() } : {}),
```

Add dock state restore in `restoreFromUrl`:
```typescript
    if (link.dock) {
      for (const [id, state] of Object.entries(link.dock)) {
        dockState.set(id, state === "open");
        if (state === "closed") {
          const panelEl = target.querySelector<HTMLElement>(`[data-component-id="${CSS.escape(id)}"]`);
          const slotContainer = panelEl?.closest<HTMLElement>("[data-slot]");
          if (slotContainer) slotContainer.style.display = "none";
        }
      }
    }
```

- [ ] **Step 12: Build and run all tests**

Run: `yarn build:packages && yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 13: Commit**

```
feat(pages-runtime): dockBar rendering, toggle mechanics, URL state

DockBar renders as icon strip via activation callback. pages-dock-toggle
event toggles panel visibility with flex redistribution, drag handle
toggling, and split collapse cascade. Dock state persists in URL hash.

Refs #64, Refs #65
```

---

### Task 5: Event Bus — pages-event + WebSocket Event Op

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` (pages-event handler)
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.ts` (type→op migration, event op routing)
- Test: `packages/pages-runtime/src/site.test.ts` (pages-event tests)
- Test: `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts` (event op tests)

**Interfaces:**
- Consumes: existing WebSocketSource `WireMessage`/`processMessage`, existing event delegation pattern in `site.ts`
- Produces: `pages-event` DOM custom event handling; `event` op routing in WebSocketSource; `type` → `op` wire protocol migration

- [ ] **Step 1: Write failing pages-event test**

Add to `packages/pages-runtime/src/site.test.ts`:

```typescript
describe("pages-event inter-panel communication", () => {
  it("pages-event bubbles to document from within loadSite target", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const tree: Component = {
      type: "rows",
      slots: { default: [{ type: "html", props: { content: "Panel" } }] },
    };
    const site = await loadSite(target, tree);

    const received: Array<{ topic: string; payload: unknown }> = [];
    document.addEventListener("pages-event", ((e: Event) => {
      received.push((e as CustomEvent).detail);
    }) as EventListener);

    const inner = target.querySelector("[data-component-type='html']")!;
    inner.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true,
      composed: true,
      detail: { topic: "test-topic", payload: { value: 42 } },
    }));

    expect(received).toHaveLength(1);
    expect(received[0]!.topic).toBe("test-topic");

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Write failing WebSocket event op test**

Add to `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`:

```typescript
describe("event op routing", () => {
  it("dispatches pages-event for event op messages", () => {
    // Test the wire protocol migration: type → op
    // And the new event op routing
    const msg = { op: "event", topic: "selection-changed", payload: { line: 42 } };
    // Verify processMessage handles this without a dataset field
    // (specific test shape depends on the processMessage refactoring)
  });
});
```

- [ ] **Step 3: Migrate wire protocol from type to op**

In `packages/pages-data/src/dataset/external/sources/websocket-source.ts`:

Rename the `type` field to `op` in the `WireMessage` interface:

```typescript
interface WireMessage {
  dataset?: string;
  op?: string;    // was: type
  seq?: string;
  columns?: Column[];
  rows?: (string | null)[][];
  row?: (string | null)[];
  key?: string;
  // New: event op fields
  topic?: string;
  payload?: unknown;
}
```

Update `processMessage` to check `msg.op` instead of `msg.type`. Add early `event` op handling before dataset routing:

```typescript
function processMessage(msg: WireMessage): void {
  // Event op: dispatch as DOM event, bypass dataset routing
  if (msg.op === "event" && msg.topic) {
    const container = /* needs reference to target element */;
    container?.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true,
      composed: true,
      detail: { topic: msg.topic, payload: msg.payload },
    }));
    return;
  }

  // Existing dataset routing...
  if (msg.seq) lastSeq = msg.seq;
  // ... switch on msg.op (was msg.type)
}
```

Note: the event op needs a reference to a DOM element to dispatch on. The `createWebSocketSource` function needs a new optional `eventTarget` parameter — the container element where `pages-event` events will be dispatched.

- [ ] **Step 4: Update createWebSocketSource signature**

Add `eventTarget?: HTMLElement` to `WebSocketSourceConfig`:

```typescript
export interface WebSocketSourceConfig {
  // ... existing fields
  readonly eventTarget?: HTMLElement;
}
```

The `processMessage` closure captures this from the config and dispatches `pages-event` on it for `event` ops.

- [ ] **Step 5: Run all tests**

Run: `yarn build:packages && yarn test`
Expected: PASS (existing tests updated for type→op rename)

- [ ] **Step 6: Commit**

```
feat(pages-data,pages-runtime): unified event bus — pages-event + event op

Migrates WebSocket wire protocol from type to op field. Adds event op
that dispatches pages-event DOM events, bypassing dataset routing.
pages-event bubbles through the component tree for inter-panel
communication.

Refs #64, Refs #69
```

---

### Task 6: DSL Builders + YAML Desugaring

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/parser/component-desugar.ts`
- Modify: `packages/pages-ui/src/dsl/index.ts` (re-exports)
- Modify: `packages/pages-ui/src/index.ts` (re-exports)
- Test: `packages/pages-ui/src/dsl/builders.test.ts`
- Test: `packages/pages-ui/src/parser/component-desugar.test.ts`

**Interfaces:**
- Consumes: `SplitProps`, `DockBarProps`, `DockItem`, `HostPanelProps` from Task 1
- Produces: `split()`, `dockBar()`, `hostPanel()` DSL builder functions; YAML desugaring for `SPLIT`, `DOCK_BAR`, `HOST_PANEL`

- [ ] **Step 1: Write failing builder tests**

Add to `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
describe("split builder", () => {
  it("creates a split component with numbered slots", () => {
    const child1: Component = { type: "html", props: { content: "A" } };
    const child2: Component = { type: "html", props: { content: "B" } };
    const result = split("horizontal", [child1, child2], { ratio: [60, 40] });
    expect(result.type).toBe("split");
    expect(result.props).toEqual({ direction: "horizontal", ratio: [60, 40] });
    expect(result.slots?.["0"]).toEqual([child1]);
    expect(result.slots?.["1"]).toEqual([child2]);
  });

  it("defaults minSizes to undefined", () => {
    const result = split("vertical", [{ type: "html", props: { content: "A" } }]);
    expect(result.props).toEqual({ direction: "vertical" });
  });
});

describe("dockBar builder", () => {
  it("creates a dock-bar component", () => {
    const result = dockBar("vertical", [
      { icon: "📁", label: "Explorer", panelId: "explorer" },
    ]);
    expect(result.type).toBe("dock-bar");
    expect(result.props).toEqual({
      orientation: "vertical",
      items: [{ icon: "📁", label: "Explorer", panelId: "explorer" }],
    });
  });
});

describe("hostPanel builder", () => {
  it("creates a host-panel component", () => {
    const result = hostPanel("diff-viewer", { pathA: "a.md" });
    expect(result.type).toBe("host-panel");
    expect(result.props).toEqual({ typeName: "diff-viewer", panelProps: { pathA: "a.md" } });
  });

  it("works without props", () => {
    const result = hostPanel("gauge");
    expect(result.props).toEqual({ typeName: "gauge" });
  });
});

describe("appGrid removed", () => {
  it("appGrid is no longer exported", () => {
    expect(typeof appGrid).toBe("undefined");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test`
Expected: FAIL — `split`, `dockBar`, `hostPanel` not defined

- [ ] **Step 3: Implement builders**

In `packages/pages-ui/src/dsl/builders.ts`:

```typescript
import type { SplitProps, DockBarProps, DockItem, HostPanelProps } from "@casehubio/pages-component/dist/model/component-props.js";

export function split(
  direction: "horizontal" | "vertical",
  children: Component[],
  options?: { ratio?: number[]; minSizes?: number[] },
): TypedComponent<"split"> {
  const slots: Record<string, readonly Component[]> = {};
  for (let i = 0; i < children.length; i++) {
    const child = children[i];
    if (!child) continue;
    slots[String(i)] = [child];
  }
  const props: SplitProps = {
    direction,
    ...(options?.ratio ? { ratio: options.ratio } : {}),
    ...(options?.minSizes ? { minSizes: options.minSizes } : {}),
  };
  return freeze({ type: "split" as const, props, slots: freeze(slots) });
}

export function dockBar(
  orientation: "vertical" | "horizontal",
  items: DockItem[],
): TypedComponent<"dock-bar"> {
  const props: DockBarProps = { orientation, items };
  return freeze({ type: "dock-bar" as const, props });
}

export function hostPanel(
  typeName: string,
  panelProps?: Record<string, unknown>,
): TypedComponent<"host-panel"> {
  const props: HostPanelProps = {
    typeName,
    ...(panelProps ? { panelProps } : {}),
  };
  return freeze({ type: "host-panel" as const, props });
}
```

Remove the `appGrid` function.

- [ ] **Step 4: Update YAML desugaring**

In `packages/pages-ui/src/parser/component-desugar.ts`:

Remove `APP_GRID: "app-grid"` from `NAV_TYPE_MAP`.

Add desugaring cases for `SPLIT`, `DOCK_BAR`, `HOST_PANEL` in `desugarComponent`:

```typescript
  // Split
  if (raw["split"]) {
    const splitConfig = raw["split"] as { direction?: string; children?: unknown[]; ratio?: number[]; minSizes?: number[] };
    const children = (splitConfig.children ?? []).map((c: unknown) => desugarComponent(c as Record<string, unknown>, displayerDefaults));
    return {
      type: "split",
      props: {
        direction: splitConfig.direction ?? "horizontal",
        ...(splitConfig.ratio ? { ratio: splitConfig.ratio } : {}),
        ...(splitConfig.minSizes ? { minSizes: splitConfig.minSizes } : {}),
      },
      slots: Object.fromEntries(children.map((c, i) => [String(i), [c]])),
    };
  }

  // Dock bar
  if (raw["dock-bar"]) {
    const config = raw["dock-bar"] as { orientation?: string; items?: DockItem[] };
    return {
      type: "dock-bar",
      props: {
        orientation: config.orientation ?? "vertical",
        items: config.items ?? [],
      },
    };
  }

  // Host panel
  if (raw["host-panel"]) {
    const config = raw["host-panel"] as { type?: string; props?: Record<string, unknown> };
    return {
      type: "host-panel",
      props: {
        typeName: config.type ?? "",
        ...(config.props ? { panelProps: config.props } : {}),
      },
    };
  }
```

- [ ] **Step 5: Update exports**

In `packages/pages-ui/src/dsl/index.ts`, add exports for `split`, `dockBar`, `hostPanel`. Remove `appGrid`.

- [ ] **Step 6: Run tests**

Run: `yarn build:packages && yarn workspace @casehubio/pages-ui run test`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(pages-ui): split, dockBar, hostPanel DSL builders + YAML desugaring

Adds builder functions and YAML desugaring for all three workbench
primitives. Removes appGrid builder and APP_GRID YAML mapping.

Refs #64
```

---

### Task 7: Integration Test + Package Exports + Cleanup

**Files:**
- Modify: `packages/pages-component/src/index.ts` (exports)
- Modify: `packages/pages-runtime/src/index.ts` (exports)
- Modify: `packages/pages-ui/src/index.ts` (exports)
- Create: `packages/pages-runtime/src/workbench.test.ts` (integration test)
- Modify: any remaining references to `app-grid` across the codebase

**Interfaces:**
- Consumes: all previous tasks
- Produces: verified end-to-end workbench rendering, clean package exports

- [ ] **Step 1: Clean up all app-grid references**

Search and remove all remaining `app-grid` references:
- `packages/pages-component/src/model/type-guards.ts` — already done in Task 1
- `packages/pages-component/src/renderer/layout.ts` — already done in Task 2
- `packages/pages-ui/src/dsl/builders.ts` — already done in Task 6
- `packages/pages-ui/src/parser/component-desugar.ts` — already done in Task 6
- Verify no other references: `grep -r "app-grid\|appGrid\|AppGrid" packages/*/src/ --include="*.ts" | grep -v ".test.ts" | grep -v "dist/"`

- [ ] **Step 2: Write integration test**

Create `packages/pages-runtime/src/workbench.test.ts`:

```typescript
import { loadSite } from "./site.js";
import { registerPanel, clearPanelRegistry } from "./panel-registry.js";
import type { Component } from "@casehubio/pages-component/dist/model/types.js";

describe("workbench integration", () => {
  afterEach(() => {
    clearPanelRegistry();
    history.replaceState(null, "", location.pathname);
  });

  it("renders a full workbench with split, dockBar, and hostPanel", async () => {
    // Register a test Web Component
    customElements.define("test-panel", class extends HTMLElement {
      configure(props: Record<string, unknown>) {
        this.textContent = `Panel: ${String(props.name ?? "")}`;
      }
    });
    registerPanel("test", "test-panel");

    const target = document.createElement("div");
    document.body.appendChild(target);

    const workbench: Component = {
      type: "rows",
      slots: {
        default: [
          // Topbar
          { type: "html", props: { content: "<h1>App</h1>" } },
          // Main content with dock bar and split
          {
            type: "split",
            props: { direction: "horizontal", ratio: [70, 30] },
            slots: {
              "0": [{ type: "host-panel", id: "main", props: { typeName: "test", panelProps: { name: "Main" } } }],
              "1": [{ type: "host-panel", id: "side", props: { typeName: "test", panelProps: { name: "Side" } } }],
            },
          },
        ],
      },
    };

    const site = await loadSite(target, workbench);

    // Verify split rendered with flex
    const splitEl = target.querySelector('[data-component-type="split"]') as HTMLElement;
    expect(splitEl).toBeTruthy();
    expect(splitEl.style.display).toBe("flex");

    // Verify hosted panels mounted
    const panels = target.querySelectorAll("test-panel");
    expect(panels).toHaveLength(2);
    expect(panels[0]!.textContent).toBe("Panel: Main");
    expect(panels[1]!.textContent).toBe("Panel: Side");

    // Verify drag handle
    const handle = target.querySelector("[data-split-handle]");
    expect(handle).toBeTruthy();

    site.dispose();
    document.body.removeChild(target);
  });

  it("dock toggle hides panel and redistributes space", async () => {
    customElements.define("test-panel-2", class extends HTMLElement {});
    registerPanel("p2", "test-panel-2");

    const target = document.createElement("div");
    document.body.appendChild(target);

    const workbench: Component = {
      type: "split",
      props: { direction: "horizontal", ratio: [70, 30] },
      slots: {
        "0": [{ type: "host-panel", props: { typeName: "p2" } }],
        "1": [{ type: "host-panel", id: "toggled", props: { typeName: "p2" } }],
      },
    };

    const site = await loadSite(target, workbench);

    // Toggle panel hidden
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true,
      composed: true,
      detail: { panelId: "toggled", visible: false },
    }));

    const toggledSlot = target.querySelector('[data-component-id="toggled"]')!
      .closest("[data-slot]") as HTMLElement;
    expect(toggledSlot.style.display).toBe("none");

    // Toggle back visible
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true,
      composed: true,
      detail: { panelId: "toggled", visible: true },
    }));
    expect(toggledSlot.style.display).not.toBe("none");

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 3: Verify package exports are complete**

Verify `packages/pages-runtime/src/index.ts` exports `registerPanel`.
Verify `packages/pages-ui/src/index.ts` re-exports `split`, `dockBar`, `hostPanel` from `./dsl/index.js`.
Verify `packages/pages-component/src/index.ts` re-exports new type guards (via `./model/index.js`).

- [ ] **Step 4: Full build and test**

Run: `yarn build && yarn test`
Expected: all packages build, all tests pass

- [ ] **Step 5: Type check**

Run: `yarn typecheck`
Expected: no type errors

- [ ] **Step 6: Lint**

Run: `yarn lint`
Expected: no lint errors (fix any that arise)

- [ ] **Step 7: Commit**

```
feat: workbench primitives integration — end-to-end test + cleanup

Full workbench rendering with split, dockBar, hostPanel, dock toggle,
and event bus verified. All app-grid references removed. Package
exports updated.

Closes #64, Closes #65, Closes #66, Closes #67, Closes #68, Closes #69
```

---

## Topbar/Statusbar (#67)

No new component types needed. Topbar and statusbar are expressed using existing `rows`, `columns`, `panel`, `html` primitives. The integration test in Task 7 demonstrates the pattern:

```typescript
rows(
  panel("App", logo(), nav()),        // topbar
  split("horizontal", [...]),          // main workspace
  html("<footer>Connected</footer>"),  // statusbar
)
```

This is tested in the integration test (Task 7, Step 2) as part of the full workbench layout.

---

## After Implementation Checklist

Per spec "After Implementation" section — update these files once all tasks land:

- [ ] ARC42STORIES.MD — re-add workbench descriptions to §1, §3, §4, §5, §6, §10, §13
- [ ] PLATFORM.md capability entry — update from "YAML dashboard rendering" to include workbench primitives (#78)
- [ ] CASEHUB-PAGES.md — add workbench primitives section (DSL builders, hostPanel, event bus)
