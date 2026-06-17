# Lazy Tab Rendering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement lazy rendering for one-at-a-time containers (tabs, pills, sidebar, carousel, stack) so only the active slot's components are in the DOM. Fixes registry corruption, resource waste, and broken cross-filtering (#32).

**Architecture:** The renderer creates empty slot container divs for lazy types. `wireInteractivity` receives a `renderSlot` callback and renders only the active slot. On tab switch, the swap function tears down the old slot and renders the new one. A `slotSwapRegistry` WeakMap coordinates click handlers and programmatic `activateSlot` calls through the same swap function. The runtime evicts disconnected registry entries on `casehub-slot-change`.

**Tech Stack:** TypeScript, vitest, `@casehub/component` (renderer), `@casehub/runtime` (site orchestration)

## Global Constraints

- All code is type-safe TypeScript. No `any` casts.
- `LAZY_TYPES = new Set(["tabs", "pills", "sidebar", "carousel", "stack"])` — defined in `render.ts`.
- Accordion is NOT lazy — all sections start expanded.
- Test command for `@casehub/component`: `yarn workspace @casehub/component run test`
- Test command for `@casehub/runtime`: `yarn workspace @casehub/runtime run test`
- Pre-existing: 2 dark-theme test failures in `@casehub/runtime` (site.test.ts lines 150-193). Unrelated to #32.
- All commits reference `Refs #32`.

---

### Task 1: Extract dispatchSlotChange to slot-swap.ts

Pure refactor — no behavior change. Creates the shared module, migrates both consumers.

**Files:**
- Create: `packages/casehub-component/src/renderer/slot-swap.ts`
- Create: `packages/casehub-component/src/renderer/slot-swap.test.ts`
- Modify: `packages/casehub-component/src/renderer/interactive.ts` (delete local `dispatchSlotChange`, import from `slot-swap`)
- Modify: `packages/casehub-component/src/renderer/activate-slot.ts` (delete inline dispatch, import from `slot-swap`)

**Interfaces:**
- Produces: `SwapFn` type, `slotSwapRegistry` WeakMap, `dispatchSlotChange` function — consumed by Tasks 2, 3

- [ ] **Step 1: Write slot-swap.test.ts**

```typescript
import { describe, it, expect } from "vitest";
import { slotSwapRegistry, dispatchSlotChange } from "./slot-swap.js";
import type { SwapFn } from "./slot-swap.js";

describe("slotSwapRegistry", () => {
  it("stores and retrieves swap functions keyed by element", () => {
    const el = document.createElement("div");
    const swap: SwapFn = () => {};
    slotSwapRegistry.set(el, swap);
    expect(slotSwapRegistry.get(el)).toBe(swap);
  });

  it("returns undefined for unregistered elements", () => {
    const el = document.createElement("div");
    expect(slotSwapRegistry.get(el)).toBeUndefined();
  });
});

describe("dispatchSlotChange", () => {
  it("emits CustomEvent with activeSlot and containerId", () => {
    const el = document.createElement("div");
    el.dataset.componentId = "tabs-1";
    const events: Array<{ activeSlot: string; containerId: string }> = [];
    el.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    dispatchSlotChange(el, "Sales");
    expect(events).toHaveLength(1);
    expect(events[0]!.activeSlot).toBe("Sales");
    expect(events[0]!.containerId).toBe("tabs-1");
  });

  it("event bubbles and is composed", () => {
    const parent = document.createElement("div");
    const child = document.createElement("div");
    child.dataset.componentId = "inner";
    parent.appendChild(child);
    let bubbled = false;
    parent.addEventListener("casehub-slot-change", () => {
      bubbled = true;
    });
    dispatchSlotChange(child, "A");
    expect(bubbled).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- slot-swap`
Expected: FAIL — `slot-swap.ts` doesn't exist yet.

- [ ] **Step 3: Create slot-swap.ts**

```typescript
export type SwapFn = (slotName: string) => void;

export const slotSwapRegistry = new WeakMap<HTMLElement, SwapFn>();

export function dispatchSlotChange(
  container: HTMLElement,
  slotName: string,
): void {
  container.dispatchEvent(
    new CustomEvent("casehub-slot-change", {
      bubbles: true,
      composed: true,
      detail: {
        activeSlot: slotName,
        containerId: container.dataset.componentId,
      },
    }),
  );
}
```

- [ ] **Step 4: Run slot-swap tests**

Run: `yarn workspace @casehub/component run test -- slot-swap`
Expected: PASS (4 tests).

- [ ] **Step 5: Migrate interactive.ts — delete local dispatchSlotChange, import from slot-swap**

Add import at top of `interactive.ts`:
```typescript
import { dispatchSlotChange } from "./slot-swap.js";
```

Delete the local `dispatchSlotChange` function (lines 28-38):
```typescript
// DELETE this entire function:
function dispatchSlotChange(container: HTMLElement, slotName: string): void {
  container.dispatchEvent(
    new CustomEvent("casehub-slot-change", {
      bubbles: true,
      composed: true,
      detail: {
        activeSlot: slotName,
        containerId: container.dataset.componentId,
      },
    }),
  );
}
```

- [ ] **Step 6: Migrate activate-slot.ts — replace inline dispatch with shared function**

Add import at top of `activate-slot.ts`:
```typescript
import { dispatchSlotChange } from "./slot-swap.js";
```

Replace the inline dispatch (lines 30-40) with a single call. The function currently ends with:
```typescript
  container.dispatchEvent(
    new CustomEvent("casehub-slot-change", {
      bubbles: true,
      composed: true,
      detail: {
        activeSlot: slotName,
        containerId: container.dataset.componentId,
      },
    }),
  );

  return true;
```

Replace with:
```typescript
  dispatchSlotChange(container, slotName);

  return true;
```

- [ ] **Step 7: Run all component tests**

Run: `yarn workspace @casehub/component run test`
Expected: PASS — 125 tests + 4 new slot-swap tests = 129 tests. All existing behavior unchanged.

- [ ] **Step 8: Commit**

```bash
git add packages/casehub-component/src/renderer/slot-swap.ts packages/casehub-component/src/renderer/slot-swap.test.ts packages/casehub-component/src/renderer/interactive.ts packages/casehub-component/src/renderer/activate-slot.ts
git commit -m "refactor: extract dispatchSlotChange to shared slot-swap module

Consolidates duplicate dispatchSlotChange implementations from
interactive.ts and activate-slot.ts into slot-swap.ts. Adds SwapFn
type and slotSwapRegistry WeakMap for Task 2.

Refs #32"
```

---

### Task 2: Lazy rendering core (render.ts + interactive.ts)

The main feature. Adds LAZY_TYPES, LazyConfig, lazy/eager rendering split in render.ts, and swap-function-based wire functions in interactive.ts.

**Files:**
- Modify: `packages/casehub-component/src/renderer/render.ts` (LAZY_TYPES, lazy/eager slot split)
- Modify: `packages/casehub-component/src/renderer/interactive.ts` (LazyConfig, swap functions per wire-type)
- Modify: `packages/casehub-component/src/renderer/render.test.ts` (lazy/eager assertions)
- Modify: `packages/casehub-component/src/renderer/interactive.test.ts` (swap function, lazy swap tests)

**Interfaces:**
- Consumes: `SwapFn`, `slotSwapRegistry`, `dispatchSlotChange` from `slot-swap.ts` (Task 1)
- Produces: `LazyConfig` type export from `interactive.ts`, swap functions registered in `slotSwapRegistry` — consumed by Task 3

- [ ] **Step 1: Write new render.test.ts tests for lazy/eager behavior**

Add these tests to `render.test.ts`:

```typescript
describe("renderComponent — lazy rendering for one-at-a-time containers", () => {
  it("tabs: only first slot children rendered, other slots empty", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "tabs",
      slots: {
        A: [{ type: "html" }, { type: "html" }],
        B: [{ type: "html" }],
        C: [{ type: "html" }],
      },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots).toHaveLength(3);
    // Slot A (active) has children
    expect(slots[0]!.querySelectorAll("[data-component-type]")).toHaveLength(2);
    // Slots B and C are empty
    expect(slots[1]!.children).toHaveLength(0);
    expect(slots[2]!.children).toHaveLength(0);
  });

  it("pills: only first slot children rendered", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "pills",
      slots: { X: [{ type: "html" }], Y: [{ type: "html" }] },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots[0]!.querySelectorAll("[data-component-type]")).toHaveLength(1);
    expect(slots[1]!.children).toHaveLength(0);
  });

  it("sidebar: only first slot children rendered", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "sidebar",
      slots: { Nav: [{ type: "html" }], Main: [{ type: "html" }] },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots[0]!.querySelectorAll("[data-component-type]")).toHaveLength(1);
    expect(slots[1]!.children).toHaveLength(0);
  });

  it("carousel: only first slot children rendered", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "carousel",
      slots: { S1: [{ type: "html" }], S2: [{ type: "html" }] },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots[0]!.querySelectorAll("[data-component-type]")).toHaveLength(1);
    expect(slots[1]!.children).toHaveLength(0);
  });

  it("stack: only first slot children rendered", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "stack",
      slots: { L1: [{ type: "html" }], L2: [{ type: "html" }] },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots[0]!.querySelectorAll("[data-component-type]")).toHaveLength(1);
    expect(slots[1]!.children).toHaveLength(0);
  });

  it("accordion: all slot children rendered (eager)", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "accordion",
      slots: { S1: [{ type: "html" }], S2: [{ type: "html" }] },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    const slots = container.querySelectorAll<HTMLElement>("[data-slot]");
    expect(slots[0]!.querySelectorAll("[data-component-type]")).toHaveLength(1);
    expect(slots[1]!.querySelectorAll("[data-component-type]")).toHaveLength(1);
  });

  it("onNode fires only for active slot children on initial render", () => {
    const target = document.createElement("div");
    const calls: string[] = [];
    const component: Component = {
      type: "tabs",
      slots: {
        Active: [{ type: "bar-chart" }],
        Hidden: [{ type: "table" }],
      },
    };
    renderComponent(target, component, {
      onNode: (_el, comp) => { calls.push(comp.type); },
    });
    expect(calls).toContain("tabs");
    expect(calls).toContain("bar-chart");
    expect(calls).not.toContain("table");
  });
});
```

- [ ] **Step 2: Write new interactive.test.ts tests for swap function behavior**

Add these tests to `interactive.test.ts`:

```typescript
import type { Component } from "../model/types.js";
import { slotSwapRegistry } from "./slot-swap.js";
import type { LazyConfig } from "./interactive.js";

function makeLazyConfig(
  slots: Record<string, Component[]>,
): LazyConfig {
  const renderCalls: Array<{ slotName: string }> = [];
  return {
    slotChildren: slots,
    renderSlot: (parent, children, slotName) => {
      renderCalls.push({ slotName });
      for (const child of children) {
        const el = document.createElement("div");
        el.dataset.componentType = child.type;
        parent.appendChild(el);
      }
    },
    _renderCalls: renderCalls,
  } as LazyConfig & { _renderCalls: typeof renderCalls };
}

describe("wireInteractivity — lazy tab swap", () => {
  it("tab click clears old slot and renders new slot", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    const lazy = makeLazyConfig({
      A: [{ type: "html" }],
      B: [{ type: "table" }],
    });
    wireInteractivity(container, "tabs", ["A", "B"], panels, document, lazy);

    // Initial: A rendered, B empty
    expect(panels.get("A")!.children.length).toBeGreaterThan(0);
    expect(panels.get("B")!.children).toHaveLength(0);

    // Click B
    const bar = container.querySelector("[data-tab-bar]")!;
    (bar.querySelectorAll("button")[1] as HTMLElement).click();

    // A cleared, B rendered
    expect(panels.get("A")!.children).toHaveLength(0);
    expect(panels.get("B")!.children.length).toBeGreaterThan(0);
    expect(panels.get("B")!.querySelector("[data-component-type='table']")).toBeTruthy();
  });

  it("re-clicking active tab is a no-op", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    const lazy = makeLazyConfig({
      A: [{ type: "html" }],
      B: [{ type: "table" }],
    });
    wireInteractivity(container, "tabs", ["A", "B"], panels, document, lazy);
    const childCount = panels.get("A")!.children.length;

    const events: unknown[] = [];
    container.addEventListener("casehub-slot-change", (e) => events.push(e));

    // Click A (already active)
    const bar = container.querySelector("[data-tab-bar]")!;
    (bar.querySelectorAll("button")[0] as HTMLElement).click();

    // No change, no event
    expect(panels.get("A")!.children.length).toBe(childCount);
    expect(events).toHaveLength(0);
  });

  it("switching back re-renders fresh", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    const lazy = makeLazyConfig({
      A: [{ type: "html" }],
      B: [{ type: "table" }],
    });
    wireInteractivity(container, "tabs", ["A", "B"], panels, document, lazy);

    const bar = container.querySelector("[data-tab-bar]")!;
    const buttons = bar.querySelectorAll("button");

    // Switch to B then back to A
    (buttons[1] as HTMLElement).click();
    (buttons[0] as HTMLElement).click();

    // A re-rendered
    expect(panels.get("A")!.children.length).toBeGreaterThan(0);
    expect(panels.get("B")!.children).toHaveLength(0);
  });

  it("swap function registered in slotSwapRegistry", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    const lazy = makeLazyConfig({
      A: [{ type: "html" }],
      B: [{ type: "table" }],
    });
    wireInteractivity(container, "tabs", ["A", "B"], panels, document, lazy);
    expect(slotSwapRegistry.get(container)).toBeDefined();
  });
});

describe("wireInteractivity — lazy sidebar swap", () => {
  it("sidebar click clears old slot and renders new slot", () => {
    const { container, panels } = makeSlotContainers(["Nav", "Content"]);
    const lazy = makeLazyConfig({
      Nav: [{ type: "html" }],
      Content: [{ type: "table" }],
    });
    wireInteractivity(container, "sidebar", ["Nav", "Content"], panels, document, lazy);

    expect(panels.get("Nav")!.children.length).toBeGreaterThan(0);
    expect(panels.get("Content")!.children).toHaveLength(0);

    const bar = container.querySelector("[data-tab-bar]")!;
    (bar.querySelectorAll("button")[1] as HTMLElement).click();

    expect(panels.get("Nav")!.children).toHaveLength(0);
    expect(panels.get("Content")!.querySelector("[data-component-type='table']")).toBeTruthy();
  });
});

describe("wireInteractivity — lazy carousel swap", () => {
  it("next button clears old and renders new via swap", () => {
    const { container, panels } = makeSlotContainers(["S1", "S2"]);
    const lazy = makeLazyConfig({
      S1: [{ type: "html" }],
      S2: [{ type: "table" }],
    });
    wireInteractivity(container, "carousel", ["S1", "S2"], panels, document, lazy);

    expect(panels.get("S1")!.children.length).toBeGreaterThan(0);

    const next = container.querySelector("[data-carousel-next]") as HTMLElement;
    next.click();

    expect(panels.get("S1")!.children).toHaveLength(0);
    expect(panels.get("S2")!.querySelector("[data-component-type='table']")).toBeTruthy();
  });

  it("prev button wraps and renders via swap", () => {
    const { container, panels } = makeSlotContainers(["S1", "S2"]);
    const lazy = makeLazyConfig({
      S1: [{ type: "html" }],
      S2: [{ type: "table" }],
    });
    wireInteractivity(container, "carousel", ["S1", "S2"], panels, document, lazy);

    const prev = container.querySelector("[data-carousel-prev]") as HTMLElement;
    prev.click();

    expect(panels.get("S1")!.children).toHaveLength(0);
    expect(panels.get("S2")!.querySelector("[data-component-type='table']")).toBeTruthy();
  });
});

describe("wireInteractivity — lazy stack", () => {
  it("renders only first slot, registers swap, no click handlers", () => {
    const { container, panels } = makeSlotContainers(["L1", "L2"]);
    const lazy = makeLazyConfig({
      L1: [{ type: "html" }],
      L2: [{ type: "table" }],
    });
    wireInteractivity(container, "stack", ["L1", "L2"], panels, document, lazy);

    expect(panels.get("L1")!.children.length).toBeGreaterThan(0);
    expect(panels.get("L2")!.children).toHaveLength(0);
    expect(slotSwapRegistry.get(container)).toBeDefined();
    expect(container.querySelector("[data-tab-bar]")).toBeNull();
    expect(container.querySelector("[data-carousel-prev]")).toBeNull();
  });

  it("programmatic swap via registry clears old and renders new", () => {
    const { container, panels } = makeSlotContainers(["L1", "L2"]);
    const lazy = makeLazyConfig({
      L1: [{ type: "html" }],
      L2: [{ type: "table" }],
    });
    wireInteractivity(container, "stack", ["L1", "L2"], panels, document, lazy);

    const swap = slotSwapRegistry.get(container)!;
    swap("L2");

    expect(panels.get("L1")!.children).toHaveLength(0);
    expect(panels.get("L2")!.querySelector("[data-component-type='table']")).toBeTruthy();
    expect(panels.get("L1")!.style.display).toBe("none");
    expect(panels.get("L2")!.style.display).not.toBe("none");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehub/component run test`
Expected: New tests FAIL (no `LazyConfig` export, no lazy rendering path).

- [ ] **Step 4: Add LAZY_TYPES and lazy/eager split to render.ts**

At the top of `render.ts`, after imports, add:

```typescript
// Subset of INTERACTIVE_TYPES (navigation.ts, runtime package). All lazy types
// are interactive; accordion is interactive but not lazy (all sections start expanded).
const LAZY_TYPES = new Set(["tabs", "pills", "sidebar", "carousel", "stack"]);
```

Replace the slot-rendering block (lines 95-113) with:

```typescript
  } else if (component.slots) {
    const slotNames = getSlotNames(component);
    const panels = new Map<string, HTMLElement>();
    const isLazy = LAZY_TYPES.has(component.type);

    for (const slotName of slotNames) {
      const slotContainer = doc.createElement("div");
      slotContainer.dataset.slot = slotName;
      el.appendChild(slotContainer);
      panels.set(slotName, slotContainer);

      if (!isLazy) {
        const children = getSlotChildren(component, slotName);
        for (let i = 0; i < children.length; i++) {
          renderNode(slotContainer, children[i]!, id, slotName, i, permissions, doc, onNode);
        }
      }
    }

    wireInteractivity(el, component.type, slotNames, panels, doc,
      isLazy && component.slots
        ? {
            slotChildren: component.slots,
            renderSlot: (parent: HTMLElement, children: readonly Component[], slot: string) => {
              for (let i = 0; i < children.length; i++) {
                renderNode(parent, children[i]!, id, slot, i, permissions, doc, onNode);
              }
            },
          }
        : undefined,
    );
  }
```

Also add `Component` to the import from `types.js` if not already present (it already is — line 1 imports `Component`).

- [ ] **Step 5: Rewrite interactive.ts with LazyConfig and swap functions**

Replace the entire content of `interactive.ts` with:

```typescript
import type { Component } from "../model/types.js";
import { slotSwapRegistry, dispatchSlotChange } from "./slot-swap.js";
import type { SwapFn } from "./slot-swap.js";

export interface LazyConfig {
  readonly slotChildren: Readonly<Record<string, readonly Component[]>>;
  readonly renderSlot: (
    parent: HTMLElement,
    children: readonly Component[],
    slotName: string,
  ) => void;
}

export function wireInteractivity(
  container: HTMLElement,
  type: string,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document = globalThis.document,
  lazy?: LazyConfig,
): void {
  switch (type) {
    case "tabs":
    case "pills":
      wireTabs(container, type, slotNames, panels, doc, lazy);
      break;
    case "sidebar":
      wireSidebar(container, slotNames, panels, doc, lazy);
      break;
    case "accordion":
      wireAccordion(container, slotNames, panels, doc);
      break;
    case "carousel":
      wireCarousel(container, slotNames, panels, doc, lazy);
      break;
    case "stack":
      wireStack(container, slotNames, panels, lazy);
      break;
  }
}

function applyOneVisible(
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  activeIndex: number,
): void {
  slotNames.forEach((name, i) => {
    const panel = panels.get(name);
    if (panel) {
      panel.style.display = i === activeIndex ? "" : "none";
    }
  });
}

function updateButtons(bar: HTMLElement, activeSlot: string): void {
  for (const btn of bar.querySelectorAll<HTMLElement>("button[data-slot]")) {
    if (btn.dataset.slot === activeSlot) {
      btn.dataset.active = "";
    } else {
      delete btn.dataset.active;
    }
  }
}

function renderInitialSlot(
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  lazy: LazyConfig | undefined,
): string {
  const currentSlot = slotNames[0] ?? "";
  if (lazy && currentSlot) {
    const firstPanel = panels.get(currentSlot);
    if (firstPanel) {
      lazy.renderSlot(
        firstPanel,
        lazy.slotChildren[currentSlot] ?? [],
        currentSlot,
      );
    }
  }
  applyOneVisible(slotNames, panels, 0);
  return currentSlot;
}

function buildSwap(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  lazy: LazyConfig | undefined,
  getCurrentSlot: () => string,
  setCurrentSlot: (s: string) => void,
  afterSwap?: (slotName: string) => void,
): SwapFn {
  const swap: SwapFn = (slotName: string) => {
    if (slotName === getCurrentSlot()) return;

    if (lazy) {
      const oldPanel = panels.get(getCurrentSlot());
      if (oldPanel) oldPanel.innerHTML = "";
      const newPanel = panels.get(slotName);
      if (newPanel) {
        lazy.renderSlot(
          newPanel,
          lazy.slotChildren[slotName] ?? [],
          slotName,
        );
      }
    }

    const newIndex = slotNames.indexOf(slotName);
    if (newIndex >= 0) {
      applyOneVisible(slotNames, panels, newIndex);
    }

    setCurrentSlot(slotName);
    afterSwap?.(slotName);
    dispatchSlotChange(container, slotName);
  };

  slotSwapRegistry.set(container, swap);
  return swap;
}

function wireTabs(
  container: HTMLElement,
  type: string,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
  lazy?: LazyConfig,
): void {
  const bar = doc.createElement("div");
  bar.dataset.tabBar = "";
  bar.className = type === "pills" ? "casehub-pills" : "casehub-tabs";

  slotNames.forEach((name) => {
    const button = doc.createElement("button");
    button.dataset.slot = name;
    button.textContent = name;
    bar.appendChild(button);
  });

  container.insertBefore(bar, container.firstChild);

  let currentSlot = renderInitialSlot(slotNames, panels, lazy);

  const swap = buildSwap(
    container, slotNames, panels, lazy,
    () => currentSlot,
    (s) => { currentSlot = s; },
    (slotName) => updateButtons(bar, slotName),
  );

  bar.addEventListener("click", (e) => {
    const target = e.target as HTMLElement;
    if (target.tagName === "BUTTON") {
      const slotName = target.dataset.slot;
      if (slotName) swap(slotName);
    }
  });
}

function wireSidebar(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
  lazy?: LazyConfig,
): void {
  const bar = doc.createElement("div");
  bar.dataset.tabBar = "";
  bar.className = "casehub-sidebar";

  slotNames.forEach((name) => {
    const button = doc.createElement("button");
    button.dataset.slot = name;
    button.textContent = name;
    bar.appendChild(button);
  });

  container.insertBefore(bar, container.firstChild);

  let currentSlot = renderInitialSlot(slotNames, panels, lazy);

  const swap = buildSwap(
    container, slotNames, panels, lazy,
    () => currentSlot,
    (s) => { currentSlot = s; },
    (slotName) => updateButtons(bar, slotName),
  );

  bar.addEventListener("click", (e) => {
    const target = e.target as HTMLElement;
    if (target.tagName === "BUTTON") {
      const slotName = target.dataset.slot;
      if (slotName) swap(slotName);
    }
  });
}

function wireAccordion(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
): void {
  slotNames.forEach((name) => {
    const panel = panels.get(name);
    if (panel) {
      const header = doc.createElement("button");
      header.dataset.accordionHeader = "";
      header.textContent = name;
      container.insertBefore(header, panel);

      header.addEventListener("click", () => {
        const wasHidden = panel.style.display === "none";
        panel.style.display = wasHidden ? "" : "none";
        if (wasHidden) {
          dispatchSlotChange(container, name);
        }
      });
    }
  });
}

function wireCarousel(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
  lazy?: LazyConfig,
): void {
  let currentSlot = renderInitialSlot(slotNames, panels, lazy);

  const swap = buildSwap(
    container, slotNames, panels, lazy,
    () => currentSlot,
    (s) => { currentSlot = s; },
  );

  const nav = doc.createElement("div");
  const prevButton = doc.createElement("button");
  prevButton.dataset.carouselPrev = "";
  prevButton.textContent = "←";

  const nextButton = doc.createElement("button");
  nextButton.dataset.carouselNext = "";
  nextButton.textContent = "→";

  nav.appendChild(prevButton);
  nav.appendChild(nextButton);
  container.appendChild(nav);

  prevButton.addEventListener("click", () => {
    const currentIndex = slotNames.indexOf(currentSlot);
    const newIndex = (currentIndex - 1 + slotNames.length) % slotNames.length;
    swap(slotNames[newIndex]!);
  });

  nextButton.addEventListener("click", () => {
    const currentIndex = slotNames.indexOf(currentSlot);
    const newIndex = (currentIndex + 1) % slotNames.length;
    swap(slotNames[newIndex]!);
  });
}

function wireStack(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  lazy?: LazyConfig,
): void {
  let currentSlot = renderInitialSlot(slotNames, panels, lazy);

  buildSwap(
    container, slotNames, panels, lazy,
    () => currentSlot,
    (s) => { currentSlot = s; },
  );
}
```

- [ ] **Step 6: Run all component tests**

Run: `yarn workspace @casehub/component run test`
Expected: ALL tests pass — existing 125 + slot-swap 4 + new lazy tests.

- [ ] **Step 7: Commit**

```bash
git add packages/casehub-component/src/renderer/render.ts packages/casehub-component/src/renderer/interactive.ts packages/casehub-component/src/renderer/render.test.ts packages/casehub-component/src/renderer/interactive.test.ts
git commit -m "feat: lazy rendering for one-at-a-time containers

Tabs, pills, sidebar, carousel, and stack now render only the active
slot's children. On switch, the swap function tears down the old slot
and renders the new one. Each wire function builds a swap function
registered in slotSwapRegistry — click handlers delegate to it.

Accordion stays eager (all sections expanded by default).

Refs #32"
```

---

### Task 3: activate-slot swap registry integration

Wires `activateSlot` to delegate to the swap function for lazy containers. This enables programmatic navigation (`site.navigate()`) to trigger lazy rendering.

**Files:**
- Modify: `packages/casehub-component/src/renderer/activate-slot.ts`
- Modify: `packages/casehub-component/src/renderer/activate-slot.test.ts`

**Interfaces:**
- Consumes: `slotSwapRegistry` from `slot-swap.ts` (Task 1), swap functions registered by `wireInteractivity` (Task 2)

- [ ] **Step 1: Write new activate-slot tests**

Add to `activate-slot.test.ts`:

```typescript
import { slotSwapRegistry } from "./slot-swap.js";

describe("activateSlot — swap registry integration", () => {
  it("delegates to swap function for tabs (lazy container)", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "tabs",
      slots: {
        A: [{ type: "html" }],
        B: [{ type: "html" }],
      },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;

    // Verify swap function is registered
    expect(slotSwapRegistry.get(container)).toBeDefined();

    // activateSlot should delegate to swap
    const result = activateSlot(container, "B");
    expect(result).toBe(true);

    // Slot A should be cleared (lazy teardown), B should have content
    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots[0]!.children).toHaveLength(0); // A cleared
    expect(slots[1]!.querySelectorAll("[data-component-type]").length).toBeGreaterThan(0); // B rendered
  });

  it("falls through to display-toggle for accordion (no swap function)", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "accordion",
      slots: {
        S1: [{ type: "html" }],
        S2: [{ type: "html" }],
      },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;

    // Accordion has no swap function
    expect(slotSwapRegistry.get(container)).toBeUndefined();

    const result = activateSlot(container, "S2");
    expect(result).toBe(true);

    // Both slots still have content (eager), display toggled
    const slots = container.querySelectorAll<HTMLElement>("[data-slot]");
    expect(slots[0]!.querySelectorAll("[data-component-type]").length).toBeGreaterThan(0);
    expect(slots[1]!.querySelectorAll("[data-component-type]").length).toBeGreaterThan(0);
    expect(slots[0]!.style.display).toBe("none");
    expect(slots[1]!.style.display).toBe("");
  });

  it("sidebar programmatic activation renders lazily", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "sidebar",
      slots: {
        Nav: [{ type: "html" }],
        Content: [{ type: "html" }],
      },
    };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;

    activateSlot(container, "Content");

    const slots = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
    expect(slots[0]!.children).toHaveLength(0); // Nav cleared
    expect(slots[1]!.querySelectorAll("[data-component-type]").length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run tests to verify new tests fail**

Run: `yarn workspace @casehub/component run test -- activate-slot`
Expected: New "swap registry integration" tests FAIL — `activateSlot` doesn't check the swap registry yet.

- [ ] **Step 3: Update activate-slot.ts to check swap registry**

Add `slotSwapRegistry` to the existing import from `slot-swap.js`:

```typescript
import { slotSwapRegistry, dispatchSlotChange } from "./slot-swap.js";
```

At the top of the `activateSlot` function body, before the panel querySelectorAll, add:

```typescript
  const swap = slotSwapRegistry.get(container);
  if (swap) {
    swap(slotName);
    return true;
  }
```

The complete function becomes:

```typescript
export function activateSlot(
  container: HTMLElement,
  slotName: string,
): boolean {
  const swap = slotSwapRegistry.get(container);
  if (swap) {
    swap(slotName);
    return true;
  }

  const panels = container.querySelectorAll<HTMLElement>(":scope > [data-slot]");
  let found = false;

  for (const panel of panels) {
    if (panel.dataset.slot === slotName) {
      panel.style.display = "";
      found = true;
    } else {
      panel.style.display = "none";
    }
  }

  if (!found) return false;

  const bar = container.querySelector<HTMLElement>(":scope > [data-tab-bar]");
  if (bar) {
    for (const button of bar.querySelectorAll<HTMLElement>("button[data-slot]")) {
      if (button.dataset.slot === slotName) {
        button.dataset.active = "";
      } else {
        delete button.dataset.active;
      }
    }
  }

  dispatchSlotChange(container, slotName);

  return true;
}
```

- [ ] **Step 4: Run all component tests**

Run: `yarn workspace @casehub/component run test`
Expected: ALL pass.

- [ ] **Step 5: Commit**

```bash
git add packages/casehub-component/src/renderer/activate-slot.ts packages/casehub-component/src/renderer/activate-slot.test.ts
git commit -m "feat: activateSlot delegates to swap registry for lazy containers

Programmatic slot activation (site.navigate()) now triggers lazy
rendering through the same swap function as click handlers. Accordion
falls through to the existing display-toggle path.

Refs #32"
```

---

### Task 4: Registry eviction in site.ts

Runtime cleanup — evicts disconnected registry entries when slots swap.

**Files:**
- Modify: `packages/casehub-runtime/src/site.ts` (eviction loop in `casehub-slot-change` handler)
- Modify: `packages/casehub-runtime/src/site.test.ts` (eviction tests)

**Interfaces:**
- Consumes: `ComponentRegistry` (registry.ts), `element.isConnected` DOM API
- Consumes: Lazy rendering from Tasks 2-3 (swap teardown disconnects elements)

- [ ] **Step 1: Write site.test.ts tests for registry eviction**

Add to `site.test.ts`:

```typescript
describe("loadSite — lazy rendering and registry eviction", () => {
  it("tabs render only active slot on initial load", async () => {
    const yaml = `
pages:
  - components:
      - settings:
          type: tabs
        components:
          Tab1:
            - html: "<p>first</p>"
          Tab2:
            - html: "<p>second</p>"
`;
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, yaml);

    // Only first tab content rendered
    const slots = target.querySelectorAll<HTMLElement>("[data-slot]");
    const tab1Slot = Array.from(slots).find((s) => s.dataset.slot === "Tab1");
    const tab2Slot = Array.from(slots).find((s) => s.dataset.slot === "Tab2");
    expect(tab1Slot!.querySelectorAll("[data-component-type]").length).toBeGreaterThan(0);
    expect(tab2Slot!.children).toHaveLength(0);

    site.dispose();
    document.body.removeChild(target);
  });

  it("navigate() to different tab swaps content and evicts old entries", async () => {
    const yaml = `
pages:
  - components:
      - settings:
          type: tabs
          id: nav
        components:
          Tab1:
            - displayer:
                type: BARCHART
                id: chart1
                lookup:
                  uuid: ds1
                  group:
                    - columnGroup:
                        source: Column 0
                      functions:
                        - source: Column 0
          Tab2:
            - html: "<p>tab2 content</p>"
`;
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, yaml);

    // Navigate to Tab2
    site.navigate("Tab2");

    // Tab1 content should be gone (lazy teardown)
    const tab1Slot = target.querySelector("[data-slot='Tab1']") as HTMLElement;
    expect(tab1Slot.children).toHaveLength(0);

    // Tab2 content should be rendered
    const tab2Slot = target.querySelector("[data-slot='Tab2']") as HTMLElement;
    expect(tab2Slot.querySelectorAll("[data-component-type]").length).toBeGreaterThan(0);

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Run tests to verify baseline**

Run: `yarn workspace @casehub/runtime run test`
Expected: New tests should already PASS if Task 2 and 3 are complete (lazy rendering + activate-slot swap registry). If they fail, investigate — the YAML parsing and rendering pipeline may need adjustment.

- [ ] **Step 3: Add registry eviction to site.ts**

In the `casehub-slot-change` handler (around line 94-105), after updating `activeSlots` and `currentPage`, add the eviction loop:

```typescript
  target.addEventListener("casehub-slot-change", ((e: Event) => {
    const detail = (e as CustomEvent).detail;
    if (!detail) return;
    const { activeSlot, containerId } = detail;
    if (typeof activeSlot === "string" && typeof containerId === "string") {
      activeSlots.set(containerId, activeSlot);
      currentPage = computeCurrentPage(root, activeSlots);
    }
    if (!_navigating) {
      syncUrl("pushState");
    }

    // Evict stale registry entries disconnected by lazy slot teardown
    for (const [id, entry] of registry) {
      if (!entry.element.isConnected) {
        registry.delete(id);
      }
    }
  }) as EventListener, { signal: abortController.signal });
```

- [ ] **Step 4: Build packages before running runtime tests**

The runtime package imports from `@casehub/component`. After changing `interactive.ts` and `render.ts`, rebuild the component package:

Run: `yarn workspace @casehub/component run build`

- [ ] **Step 5: Run all runtime tests**

Run: `yarn workspace @casehub/runtime run test`
Expected: All pass (95 existing + new lazy tests). Pre-existing 2 dark-theme failures unrelated.

- [ ] **Step 6: Run full test suite across both packages**

Run: `yarn workspace @casehub/component run test && yarn workspace @casehub/runtime run test`
Expected: All pass in component (129+), runtime pass rate unchanged (95/97 + new tests).

- [ ] **Step 7: Commit**

```bash
git add packages/casehub-runtime/src/site.ts packages/casehub-runtime/src/site.test.ts
git commit -m "feat: evict disconnected registry entries on slot change

After lazy slot teardown, the casehub-slot-change handler scans the
registry and removes entries whose element.isConnected is false.
Async resolutions in flight resolve harmlessly against disconnected
elements — no cancellation needed.

Closes #32"
```
