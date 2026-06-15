# Site Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `@casehub/runtime` — the integration layer that wires component rendering, data resolution, cross-filtering, and page navigation into a working site.

**Architecture:** New `@casehub/runtime` package at the top of the DAG, depending on `@casehub/viz`, `@casehub/ui`, `@casehub/component`, and `@casehub/data`. Prerequisite changes to `@casehub/data` (LookupResult, injectable fetch) and `@casehub/component` (onNode callback, casehub-slot-change event, sidebar wiring, activateSlot). Pure function modules first, then the event-driven runtime orchestrator.

**Tech Stack:** TypeScript 5.6, Vitest (jsdom environment), Yarn workspaces

**Spec:** `docs/superpowers/specs/2026-06-15-site-runtime-design.md`

---

### Task 1: `@casehub/data` — LookupResult return type

Change `DataSetManager.lookup()` to return `LookupResult { dataset, totalRows }` instead of bare `TypedDataSet`. The `totalRows` value is the row count after ops but before pagination — already computed internally.

**Files:**
- Modify: `packages/core/src/dataset/manager.ts`
- Modify: `packages/core/src/dataset/manager.test.ts`

- [ ] **Step 1: Add LookupResult interface and update DataSetManager**

In `packages/core/src/dataset/manager.ts`, add the interface and change the return type:

```typescript
export interface LookupResult {
  readonly dataset: TypedDataSet;
  readonly totalRows: number;
}

export interface DataSetManager {
  register(id: DataSetId, dataset: TypedDataSet): void;
  get(id: DataSetId): TypedDataSet | undefined;
  remove(id: DataSetId): boolean;
  has(id: DataSetId): boolean;
  accumulate(id: DataSetId, dataset: TypedDataSet, maxRows?: number): void;
  lookup(query: DataSetLookup, options?: LookupOptions): LookupResult;
}
```

Update `DataSetManagerImpl.lookup()`:

```typescript
lookup(query: DataSetLookup, options?: LookupOptions): LookupResult {
  const offset = options?.rowOffset ?? 0;
  if (offset < 0) {
    throw new DataSetError("INVALID_OPERATION", `rowOffset cannot be negative: ${offset}`);
  }

  const dataset = this.datasets.get(query.dataSetId);
  if (!dataset) {
    throw new DataSetError("UNKNOWN_PROVIDER", `Dataset "${query.dataSetId}" not registered`);
  }

  const resolvedOps = resolveOps(query.operations, dataset.columns);
  const opsOptions = options?.referenceDate !== undefined ? { referenceDate: options.referenceDate } : undefined;
  const result = applyOps(dataset, resolvedOps, opsOptions);
  const totalRows = result.rows.length;
  const paginated = paginate(result, offset, options?.rowCount ?? -1);
  return { dataset: paginated, totalRows };
}
```

- [ ] **Step 2: Update all test call sites**

In `packages/core/src/dataset/manager.test.ts`, every `mgr.lookup(...)` now returns `LookupResult`. Update each assertion from `result.rows` / `result.columns` to `result.dataset.rows` / `result.dataset.columns`. Add `totalRows` assertions where pagination is tested.

For non-paginated tests, add:
```typescript
expect(result.totalRows).toBe(result.dataset.rows.length);
```

For paginated tests (the `rowOffset` / `rowCount` tests), verify:
```typescript
// e.g. 3-row dataset, rowCount=2
expect(result.dataset.rows).toHaveLength(2);
expect(result.totalRows).toBe(3);
```

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehub/data run test`
Expected: All tests pass.

- [ ] **Step 4: Commit**

```
feat(@casehub/data): return LookupResult from DataSetManager.lookup()

Breaking change — lookup() now returns { dataset, totalRows } instead of
bare TypedDataSet. totalRows is the row count after ops but before
pagination. Zero new computation — the value was already available.

Refs #13
```

---

### Task 2: `@casehub/data` — Injectable fetch on BrowserFetchProvider

Add optional `fetch` parameter to `BrowserFetchProvider` constructor and `createDataProviderFactory`.

**Files:**
- Modify: `packages/core/src/dataset/external/providers/browser-fetch.ts`
- Modify: `packages/core/src/dataset/external/provider-factory.ts`
- Modify: `packages/core/src/dataset/external/provider-factory.test.ts`

- [ ] **Step 1: Write test for injectable fetch**

In `packages/core/src/dataset/external/provider-factory.test.ts`, add:

```typescript
it("passes custom fetch to BrowserFetchProvider when provided", () => {
  const customFetch = vi.fn().mockResolvedValue(
    new Response(JSON.stringify([1, 2, 3]), {
      headers: { "content-type": "application/json" },
    }),
  );
  const factory = createDataProviderFactory(customFetch);
  const provider = factory.create(
    { uuid: "test" as DataSetId, url: "https://example.com/data.json" } as ExternalDataSetDef,
    {},
  );
  expect(provider).toBeTruthy();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/data run test -- provider-factory`
Expected: FAIL — `createDataProviderFactory` doesn't accept arguments.

- [ ] **Step 3: Implement injectable fetch**

In `packages/core/src/dataset/external/providers/browser-fetch.ts`, add constructor parameter:

```typescript
export class BrowserFetchProvider implements DataProvider {
  private readonly _fetch: typeof globalThis.fetch;

  constructor(fetchFn?: typeof globalThis.fetch) {
    this._fetch = fetchFn ?? globalThis.fetch;
  }

  async fetch(request: DataRequest): Promise<FetchResult> {
    const url = new URL(request.url);
    for (const [k, v] of Object.entries(request.query)) {
      url.searchParams.set(k, v);
    }

    const headers = new Headers(request.headers);
    const init: RequestInit = { method: request.method, headers };

    if (request.body !== undefined) {
      init.body = request.body;
    } else if (request.form !== undefined) {
      const params = new URLSearchParams(request.form);
      init.body = params.toString();
      headers.set("Content-Type", "application/x-www-form-urlencoded");
    }

    const response = await this._fetch(url.toString(), init);

    if (!response.ok) {
      const text = await response.text();
      throw new Error(`HTTP ${response.status} ${response.statusText}: ${text}`);
    }

    const contentType = response.headers.get("content-type");

    if (contentType && contentType.includes("json")) {
      const data: unknown = await response.json();
      return { data, contentType };
    }

    const data = await response.text();
    return contentType ? { data, contentType } : { data };
  }
}
```

In `packages/core/src/dataset/external/provider-factory.ts`, pass `fetchFn` through:

```typescript
export function createDataProviderFactory(
  fetchFn?: typeof globalThis.fetch,
): DataProviderFactory {
  return {
    create(def: ExternalDataSetDef, config: DataProviderConfig): DataProvider | undefined {
      if (def.content !== undefined) {
        return new InlineProvider(def.content);
      }

      if (def.join !== undefined) {
        return undefined;
      }

      let provider: DataProvider =
        config.defaultProvider === "server-relay" && config.serverRelay
          ? new ServerRelayProvider(config.serverRelay.endpoint)
          : new BrowserFetchProvider(fetchFn);

      if (config.corsProxy?.enabled && config.corsProxy.url) {
        provider = new CorsProxyProvider(provider, config.corsProxy.url);
      }

      return provider;
    },
  };
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/data run test`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```
feat(@casehub/data): injectable fetch on BrowserFetchProvider

BrowserFetchProvider constructor accepts optional fetch function,
defaulting to globalThis.fetch. createDataProviderFactory() passes
the function through. Enables testing and auth proxying.

Refs #13
```

---

### Task 3: `@casehub/component` — onNode callback in RenderOptions

Add the `onNode` callback to `RenderOptions` and call it in `renderNode` after `parent.appendChild(el)` but before child recursion.

**Files:**
- Modify: `packages/casehub-component/src/renderer/render.ts`
- Modify: `packages/casehub-component/src/renderer/render.test.ts`

- [ ] **Step 1: Write tests for onNode callback**

In `packages/casehub-component/src/renderer/render.test.ts`, add:

```typescript
describe("renderComponent — onNode callback", () => {
  it("fires onNode for each component with element and component model", () => {
    const target = document.createElement("div");
    const calls: Array<{ type: string; id: string }> = [];
    const component: Component = {
      type: "rows",
      slots: {
        default: [
          { type: "bar-chart", props: { title: "Revenue" } },
          { type: "table", props: { title: "Sales" } },
        ],
      },
    };
    renderComponent(target, component, {
      onNode: (el, comp) => {
        calls.push({ type: comp.type, id: el.dataset.componentId! });
      },
    });
    expect(calls).toHaveLength(3);
    expect(calls[0]!.type).toBe("rows");
    expect(calls[1]!.type).toBe("bar-chart");
    expect(calls[2]!.type).toBe("table");
  });

  it("element is connected to DOM when onNode fires", () => {
    const target = document.createElement("div");
    document.body.appendChild(target);
    let connected = false;
    const component: Component = { type: "bar-chart" };
    renderComponent(target, component, {
      onNode: (el) => {
        connected = el.isConnected;
      },
    });
    expect(connected).toBe(true);
    document.body.removeChild(target);
  });

  it("children not yet rendered when onNode fires for parent", () => {
    const target = document.createElement("div");
    let childCount = -1;
    const component: Component = {
      type: "tabs",
      slots: { A: [{ type: "bar-chart" }], B: [{ type: "table" }] },
    };
    renderComponent(target, component, {
      onNode: (el, comp) => {
        if (comp.type === "tabs") {
          childCount = el.children.length;
        }
      },
    });
    expect(childCount).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/component run test -- render`
Expected: FAIL — `onNode` not recognized in `RenderOptions`.

- [ ] **Step 3: Implement onNode**

In `packages/casehub-component/src/renderer/render.ts`, update `RenderOptions`:

```typescript
export interface RenderOptions {
  readonly permissions?: PermissionContext;
  readonly document?: Document;
  readonly onNode?: (el: HTMLElement, component: Component) => void;
}
```

Update `renderComponent` to pass `onNode` into `renderNode`:

```typescript
export function renderComponent(
  target: HTMLElement,
  component: Component,
  options?: RenderOptions,
): void {
  const doc = options?.document ?? globalThis.document;
  const permissions = options?.permissions ?? ALLOW_ALL;
  const onNode = options?.onNode;
  target.innerHTML = "";
  renderNode(target, component, undefined, undefined, undefined, permissions, doc, onNode);
}
```

Update `renderNode` signature and add the callback after `parent.appendChild(el)`:

```typescript
function renderNode(
  parent: HTMLElement,
  component: Component,
  parentId: string | undefined,
  slotOrX: string | number | undefined,
  indexOrY: number | undefined,
  permissions: PermissionContext,
  doc: Document,
  onNode: ((el: HTMLElement, component: Component) => void) | undefined,
): void {
  if (!checkAccess(component.access, permissions)) return;

  const el = doc.createElement("div");
  const id = component.id ?? generateId(parentId, slotOrX, indexOrY);
  el.dataset.componentType = component.type;
  el.dataset.componentId = id;
  if (component.props) {
    el.dataset.componentProps = JSON.stringify(component.props);
  }

  if (isLayoutType(component.type)) {
    applyLayoutCSS(el, component.type, component.props);
  }

  if (component.style) {
    for (const [prop, value] of Object.entries(component.style)) {
      el.style.setProperty(prop, value);
    }
  }

  if (component.type === "panel" && component.props) {
    const { title } = component.props as Readonly<Record<string, unknown>>;
    if (typeof title === "string" && title) {
      const titleEl = doc.createElement("div");
      titleEl.dataset.panelTitle = "";
      titleEl.textContent = title;
      el.appendChild(titleEl);
    }
  }

  parent.appendChild(el);

  // onNode fires after appendChild, before children
  onNode?.(el, component);

  if (component.items && component.items.length > 0) {
    for (const item of component.items) {
      renderNode(el, item.component, id, item.placement.x, item.placement.y, permissions, doc, onNode);
      const child = el.lastElementChild as HTMLElement;
      if (child) {
        applyGridPlacement(child, item.placement);
      }
    }
  } else if (component.slots) {
    const slotNames = getSlotNames(component);
    const panels = new Map<string, HTMLElement>();

    for (const slotName of slotNames) {
      const slotContainer = doc.createElement("div");
      slotContainer.dataset.slot = slotName;
      el.appendChild(slotContainer);
      panels.set(slotName, slotContainer);

      const children = getSlotChildren(component, slotName);
      for (let i = 0; i < children.length; i++) {
        renderNode(slotContainer, children[i]!, id, slotName, i, permissions, doc, onNode);
      }
    }

    wireInteractivity(el, component.type, slotNames, panels, doc);
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/component run test`
Expected: All tests pass (existing + new).

- [ ] **Step 5: Commit**

```
feat(@casehub/component): add onNode callback to RenderOptions

The callback fires after parent.appendChild(el) but before recursing
into children. This is the integration point for the site runtime to
activate viz and content components inline during the render pass.

Refs #13
```

---

### Task 4: `@casehub/component` — sidebar wiring + casehub-slot-change event

Add `wireSidebar` to `wireInteractivity` and dispatch `casehub-slot-change` events from all interactive types when the active slot changes.

**Files:**
- Modify: `packages/casehub-component/src/renderer/interactive.ts`
- Modify: `packages/casehub-component/src/renderer/interactive.test.ts`

- [ ] **Step 1: Write tests for casehub-slot-change events and sidebar**

In `packages/casehub-component/src/renderer/interactive.test.ts`, add:

```typescript
describe("wireInteractivity — casehub-slot-change events", () => {
  it("tabs emit casehub-slot-change on click", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    container.dataset.componentId = "nav-1";
    wireInteractivity(container, "tabs", ["A", "B"], panels);
    const events: Array<{ activeSlot: string; containerId: string }> = [];
    container.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    const bar = container.querySelector("[data-tab-bar]")!;
    (bar.querySelectorAll("button")[1] as HTMLElement).click();
    expect(events).toHaveLength(1);
    expect(events[0]!.activeSlot).toBe("B");
    expect(events[0]!.containerId).toBe("nav-1");
  });

  it("accordion emits casehub-slot-change on click", () => {
    const { container, panels } = makeSlotContainers(["X", "Y"]);
    container.dataset.componentId = "acc-1";
    wireInteractivity(container, "accordion", ["X", "Y"], panels);
    const events: Array<{ activeSlot: string }> = [];
    container.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    const headers = container.querySelectorAll("[data-accordion-header]");
    (headers[1] as HTMLElement).click();
    expect(events).toHaveLength(1);
    expect(events[0]!.activeSlot).toBe("Y");
  });

  it("carousel emits casehub-slot-change on next", () => {
    const { container, panels } = makeSlotContainers(["P1", "P2"]);
    container.dataset.componentId = "car-1";
    wireInteractivity(container, "carousel", ["P1", "P2"], panels);
    const events: Array<{ activeSlot: string }> = [];
    container.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    const nextBtn = container.querySelector("[data-carousel-next]") as HTMLElement;
    nextBtn.click();
    expect(events).toHaveLength(1);
    expect(events[0]!.activeSlot).toBe("P2");
  });
});

describe("wireInteractivity — sidebar", () => {
  it("creates sidebar nav with buttons for each slot", () => {
    const { container, panels } = makeSlotContainers(["Overview", "Sales"]);
    wireInteractivity(container, "sidebar", ["Overview", "Sales"], panels);
    const bar = container.querySelector("[data-tab-bar]") as HTMLElement;
    expect(bar).toBeTruthy();
    expect(bar.classList.contains("casehub-sidebar")).toBe(true);
    const buttons = bar.querySelectorAll("button");
    expect(buttons).toHaveLength(2);
    expect(buttons[0]!.textContent).toBe("Overview");
  });

  it("first slot visible by default, rest hidden", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    wireInteractivity(container, "sidebar", ["A", "B"], panels);
    expect(panels.get("A")!.style.display).not.toBe("none");
    expect(panels.get("B")!.style.display).toBe("none");
  });

  it("clicking sidebar item shows target, hides others", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    wireInteractivity(container, "sidebar", ["A", "B"], panels);
    const buttons = container.querySelector("[data-tab-bar]")!.querySelectorAll("button");
    (buttons[1] as HTMLElement).click();
    expect(panels.get("A")!.style.display).toBe("none");
    expect(panels.get("B")!.style.display).not.toBe("none");
  });

  it("emits casehub-slot-change on click", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    container.dataset.componentId = "side-1";
    wireInteractivity(container, "sidebar", ["A", "B"], panels);
    const events: Array<{ activeSlot: string; containerId: string }> = [];
    container.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    const buttons = container.querySelector("[data-tab-bar]")!.querySelectorAll("button");
    (buttons[1] as HTMLElement).click();
    expect(events).toHaveLength(1);
    expect(events[0]!.activeSlot).toBe("B");
    expect(events[0]!.containerId).toBe("side-1");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/component run test -- interactive`
Expected: FAIL — sidebar not handled, no casehub-slot-change events.

- [ ] **Step 3: Implement sidebar and slot-change events**

In `packages/casehub-component/src/renderer/interactive.ts`, add the event dispatch helper function and update all handlers:

```typescript
function dispatchSlotChange(
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

Add `"sidebar"` to the switch in `wireInteractivity`:

```typescript
case "sidebar":
  wireSidebar(container, slotNames, panels, doc);
  break;
```

Add `wireSidebar` function (same structure as `wireTabs` but with `casehub-sidebar` class):

```typescript
function wireSidebar(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
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
  applyOneVisible(slotNames, panels, 0);

  bar.addEventListener("click", (e) => {
    const target = e.target as HTMLElement;
    if (target.tagName === "BUTTON") {
      const slotName = target.dataset.slot;
      if (slotName) {
        slotNames.forEach((name) => {
          const panel = panels.get(name);
          if (panel) {
            panel.style.display = name === slotName ? "" : "none";
          }
        });
        dispatchSlotChange(container, slotName);
      }
    }
  });
}
```

Add `dispatchSlotChange` calls to the existing handlers:

In `wireTabs` click handler, after the visibility toggle loop:
```typescript
dispatchSlotChange(container, slotName);
```

In `wireAccordion`, after the toggle:
```typescript
header.addEventListener("click", () => {
  panel.style.display = panel.style.display === "none" ? "" : "none";
  if (panel.style.display !== "none") {
    dispatchSlotChange(container, name);
  }
});
```

In `wireCarousel`, after each index change:
```typescript
prevButton.addEventListener("click", () => {
  currentIndex = (currentIndex - 1 + slotNames.length) % slotNames.length;
  applyOneVisible(slotNames, panels, currentIndex);
  dispatchSlotChange(container, slotNames[currentIndex]!);
});

nextButton.addEventListener("click", () => {
  currentIndex = (currentIndex + 1) % slotNames.length;
  applyOneVisible(slotNames, panels, currentIndex);
  dispatchSlotChange(container, slotNames[currentIndex]!);
});
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/component run test`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```
feat(@casehub/component): sidebar wiring + casehub-slot-change events

Add wireSidebar to wireInteractivity (vertical nav, same slot contract).
All interactive types (tabs, pills, sidebar, accordion, carousel) now
dispatch casehub-slot-change when the active slot changes.

Refs #13
```

---

### Task 5: `@casehub/component` — activateSlot public export

Export `activateSlot()` for programmatic slot activation (navigation, URL-driven, back/forward).

**Files:**
- Create: `packages/casehub-component/src/renderer/activate-slot.ts`
- Create: `packages/casehub-component/src/renderer/activate-slot.test.ts`
- Modify: `packages/casehub-component/src/renderer/index.ts`

- [ ] **Step 1: Write tests**

Create `packages/casehub-component/src/renderer/activate-slot.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { renderComponent } from "./render.js";
import { activateSlot } from "./activate-slot.js";
import type { Component } from "../model/types.js";

function renderTabs(target: HTMLElement, slotNames: string[]): HTMLElement {
  const slots: Record<string, Component[]> = {};
  for (const name of slotNames) {
    slots[name] = [{ type: "html", props: { content: name } }];
  }
  const component: Component = { type: "tabs", slots };
  renderComponent(target, component);
  return target.firstElementChild as HTMLElement;
}

function renderSidebar(target: HTMLElement, slotNames: string[]): HTMLElement {
  const slots: Record<string, Component[]> = {};
  for (const name of slotNames) {
    slots[name] = [{ type: "html", props: { content: name } }];
  }
  const component: Component = { type: "sidebar", slots };
  renderComponent(target, component);
  return target.firstElementChild as HTMLElement;
}

describe("activateSlot", () => {
  it("activates target slot, hides others (tabs)", () => {
    const target = document.createElement("div");
    const container = renderTabs(target, ["A", "B", "C"]);
    const result = activateSlot(container, "B");
    expect(result).toBe(true);
    const panels = container.querySelectorAll("[data-slot]");
    expect((panels[0] as HTMLElement).style.display).toBe("none");
    expect((panels[1] as HTMLElement).style.display).not.toBe("none");
    expect((panels[2] as HTMLElement).style.display).toBe("none");
  });

  it("returns false for non-existent slot", () => {
    const target = document.createElement("div");
    const container = renderTabs(target, ["A", "B"]);
    const result = activateSlot(container, "Z");
    expect(result).toBe(false);
  });

  it("dispatches casehub-slot-change", () => {
    const target = document.createElement("div");
    const container = renderTabs(target, ["A", "B"]);
    container.dataset.componentId = "test-nav";
    const events: Array<{ activeSlot: string; containerId: string }> = [];
    container.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
      events.push(e.detail);
    }) as EventListener);
    activateSlot(container, "B");
    expect(events).toHaveLength(1);
    expect(events[0]!.activeSlot).toBe("B");
    expect(events[0]!.containerId).toBe("test-nav");
  });

  it("works with sidebar", () => {
    const target = document.createElement("div");
    const container = renderSidebar(target, ["X", "Y"]);
    activateSlot(container, "Y");
    const panels = container.querySelectorAll("[data-slot]");
    expect((panels[0] as HTMLElement).style.display).toBe("none");
    expect((panels[1] as HTMLElement).style.display).not.toBe("none");
  });

  it("accordion shows only target panel (not toggle)", () => {
    const target = document.createElement("div");
    const slots: Record<string, Component[]> = {
      A: [{ type: "html", props: { content: "A" } }],
      B: [{ type: "html", props: { content: "B" } }],
    };
    const component: Component = { type: "accordion", slots };
    renderComponent(target, component);
    const container = target.firstElementChild as HTMLElement;
    activateSlot(container, "B");
    const panels = container.querySelectorAll("[data-slot]");
    expect((panels[0] as HTMLElement).style.display).toBe("none");
    expect((panels[1] as HTMLElement).style.display).not.toBe("none");
  });

  it("does not dispatch event when slot not found", () => {
    const target = document.createElement("div");
    const container = renderTabs(target, ["A"]);
    let fired = false;
    container.addEventListener("casehub-slot-change", () => { fired = true; });
    activateSlot(container, "Z");
    expect(fired).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/component run test -- activate-slot`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement activateSlot**

Create `packages/casehub-component/src/renderer/activate-slot.ts`:

```typescript
export function activateSlot(
  container: HTMLElement,
  slotName: string,
): boolean {
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

  // Update header active state if a tab/pill/sidebar bar exists
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
}
```

- [ ] **Step 4: Export from index**

In `packages/casehub-component/src/renderer/index.ts`, add:

```typescript
export { activateSlot } from "./activate-slot.js";
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehub/component run test`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```
feat(@casehub/component): add activateSlot() public export

Programmatic slot activation for navigation, URL-driven state, and
back/forward. Always ensures exactly one slot is visible (differs from
accordion's native toggle). Dispatches casehub-slot-change.

Refs #13
```

---

### Task 6: `@casehub/runtime` — Package scaffold

Create the package with build config, dependencies, and empty index.

**Files:**
- Create: `packages/casehub-runtime/package.json`
- Create: `packages/casehub-runtime/tsconfig.json`
- Create: `packages/casehub-runtime/vitest.config.ts`
- Create: `packages/casehub-runtime/src/index.ts`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehub/runtime",
  "version": "0.0.1",
  "description": "CaseHub Runtime — site loading, data pipeline, navigation, cross-filtering",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehub/component": "workspace:*",
    "@casehub/data": "workspace:*",
    "@casehub/ui": "workspace:*",
    "@casehub/viz": "workspace:*"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM"],
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "esModuleInterop": false,
    "isolatedModules": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

- [ ] **Step 3: Create vitest.config.ts**

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "jsdom",
    include: ["src/**/*.test.ts"],
    coverage: {
      provider: "v8",
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.test.ts", "src/index.ts"],
    },
  },
});
```

- [ ] **Step 4: Create empty src/index.ts**

```typescript
// @casehub/runtime — site loading, data pipeline, navigation, cross-filtering
```

- [ ] **Step 5: Install dependencies**

Run: `yarn install`
Expected: Workspace resolution succeeds, `@casehub/runtime` recognized.

- [ ] **Step 6: Verify build**

Run: `yarn workspace @casehub/runtime run build`
Expected: Compiles with empty output.

- [ ] **Step 7: Commit**

```
chore(@casehub/runtime): scaffold package

New package at the top of the @casehub DAG. Depends on
viz, ui, component, and data. Vitest with jsdom.

Refs #13
```

---

### Task 7: `@casehub/runtime` — page-paths module

Pure function: walk a Component tree and produce a `PagePathMap` (Component → pagePath by object identity).

**Files:**
- Create: `packages/casehub-runtime/src/page-paths.ts`
- Create: `packages/casehub-runtime/src/page-paths.test.ts`

- [ ] **Step 1: Write tests**

Create `packages/casehub-runtime/src/page-paths.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "@casehub/component/dist/model/types.js";
import { buildPagePathMap } from "./page-paths.js";

describe("buildPagePathMap", () => {
  it("root page maps to empty string", () => {
    const root: Component = { type: "page", props: { name: "App" } };
    const map = buildPagePathMap(root);
    expect(map.get(root)).toBe("");
  });

  it("child pages use slot names as path segments", () => {
    const child: Component = { type: "page", props: { name: "Sales" } };
    const root: Component = {
      type: "page",
      props: { name: "App" },
      slots: { Sales: [child] },
    };
    const map = buildPagePathMap(root);
    expect(map.get(child)).toBe("Sales");
  });

  it("nested pages produce multi-segment paths", () => {
    const revenue: Component = { type: "page", props: { name: "Revenue" } };
    const sales: Component = {
      type: "page",
      props: { name: "Sales" },
      slots: { Revenue: [revenue] },
    };
    const root: Component = {
      type: "page",
      props: { name: "App" },
      slots: { Sales: [sales] },
    };
    const map = buildPagePathMap(root);
    expect(map.get(revenue)).toBe("Sales/Revenue");
  });

  it("non-page components inherit nearest page ancestor path", () => {
    const chart: Component = { type: "bar-chart", props: { title: "Rev" } };
    const sales: Component = {
      type: "page",
      props: { name: "Sales" },
      slots: { default: [chart] },
    };
    const root: Component = {
      type: "page",
      props: { name: "App" },
      slots: { Sales: [sales] },
    };
    const map = buildPagePathMap(root);
    expect(map.get(chart)).toBe("Sales");
  });

  it("every component in the tree gets an entry", () => {
    const chart: Component = { type: "bar-chart" };
    const tabs: Component = { type: "tabs", slots: { Tab1: [chart] } };
    const root: Component = {
      type: "page",
      props: { name: "Root" },
      slots: { default: [tabs] },
    };
    const map = buildPagePathMap(root);
    expect(map.size).toBe(3);
    expect(map.has(root)).toBe(true);
    expect(map.has(tabs)).toBe(true);
    expect(map.has(chart)).toBe(true);
  });

  it("handles grid items", () => {
    const chart: Component = { type: "bar-chart" };
    const grid: Component = {
      type: "grid",
      props: { columns: 12 },
      items: [{ placement: { x: 0, y: 0, w: 12, h: 1 }, component: chart }],
    };
    const root: Component = {
      type: "page",
      props: { name: "App" },
      slots: { default: [grid] },
    };
    const map = buildPagePathMap(root);
    expect(map.get(chart)).toBe("");
    expect(map.get(grid)).toBe("");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- page-paths`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement buildPagePathMap**

Create `packages/casehub-runtime/src/page-paths.ts`:

```typescript
import type { Component } from "@casehub/component/dist/model/types.js";

export type PagePathMap = Map<Component, string>;

export function buildPagePathMap(root: Component): PagePathMap {
  const map: PagePathMap = new Map();
  walk(root, "", undefined, map);
  return map;
}

function walk(
  component: Component,
  currentPath: string,
  slotName: string | undefined,
  map: PagePathMap,
): void {
  let path = currentPath;
  if (component.type === "page" && slotName !== undefined) {
    path = currentPath ? `${currentPath}/${slotName}` : slotName;
  }

  map.set(component, path);

  if (component.items) {
    for (const item of component.items) {
      walk(item.component, path, undefined, map);
    }
  }

  if (component.slots) {
    for (const [name, children] of Object.entries(component.slots)) {
      for (const child of children) {
        walk(child, path, name, map);
      }
    }
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/runtime run test -- page-paths`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(@casehub/runtime): page-paths module

Pure function buildPagePathMap() walks a Component tree and maps every
node (by object identity) to its page path. Page-type components push
a new segment from their slot name; all others inherit from their
nearest page ancestor.

Refs #13
```

---

### Task 8: `@casehub/runtime` — dataset-scope module

Pure function: walk a Component tree and produce a `DataSetScope` map (pagePath → dataSetId → ExternalDataSetDef), with cascading inheritance from parent pages.

**Files:**
- Create: `packages/casehub-runtime/src/dataset-scope.ts`
- Create: `packages/casehub-runtime/src/dataset-scope.test.ts`

- [ ] **Step 1: Write tests**

Create `packages/casehub-runtime/src/dataset-scope.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "@casehub/component/dist/model/types.js";
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import { buildDataSetScope } from "./dataset-scope.js";
import { buildPagePathMap } from "./page-paths.js";

function makeDef(uuid: string) {
  return { uuid: uuid as DataSetId, content: "[]" } as any;
}

describe("buildDataSetScope", () => {
  it("root page datasets scoped to empty path", () => {
    const ds = makeDef("sales");
    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [ds] },
    };
    const paths = buildPagePathMap(root);
    const scope = buildDataSetScope(root, paths);
    expect(scope.get("")?.get("sales" as DataSetId)).toBe(ds);
  });

  it("child page inherits parent datasets", () => {
    const ds = makeDef("global");
    const child: Component = { type: "page", props: { name: "Sales" } };
    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [ds] },
      slots: { Sales: [child] },
    };
    const paths = buildPagePathMap(root);
    const scope = buildDataSetScope(root, paths);
    expect(scope.get("Sales")?.get("global" as DataSetId)).toBe(ds);
  });

  it("child page overrides parent dataset with same id", () => {
    const parentDs = makeDef("data");
    const childDs = makeDef("data");
    const child: Component = {
      type: "page",
      props: { name: "Sales", datasets: [childDs] },
    };
    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [parentDs] },
      slots: { Sales: [child] },
    };
    const paths = buildPagePathMap(root);
    const scope = buildDataSetScope(root, paths);
    expect(scope.get("Sales")?.get("data" as DataSetId)).toBe(childDs);
  });

  it("resolveDataSet walks up ancestors", () => {
    const ds = makeDef("root-ds");
    const grandchild: Component = { type: "page", props: { name: "Detail" } };
    const child: Component = {
      type: "page",
      props: { name: "Sales" },
      slots: { Detail: [grandchild] },
    };
    const root: Component = {
      type: "page",
      props: { name: "App", datasets: [ds] },
      slots: { Sales: [child] },
    };
    const paths = buildPagePathMap(root);
    const scope = buildDataSetScope(root, paths);
    expect(scope.get("Sales/Detail")?.get("root-ds" as DataSetId)).toBe(ds);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- dataset-scope`
Expected: FAIL.

- [ ] **Step 3: Implement buildDataSetScope**

Create `packages/casehub-runtime/src/dataset-scope.ts`:

```typescript
import type { Component } from "@casehub/component/dist/model/types.js";
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import type { ExternalDataSetDef } from "@casehub/data/dist/dataset/external/types.js";
import type { PagePathMap } from "./page-paths.js";

export type DataSetScope = Map<string, Map<DataSetId, ExternalDataSetDef>>;

export function buildDataSetScope(
  root: Component,
  paths: PagePathMap,
): DataSetScope {
  const scope: DataSetScope = new Map();
  walkScope(root, new Map(), paths, scope);
  return scope;
}

function walkScope(
  component: Component,
  inherited: Map<DataSetId, ExternalDataSetDef>,
  paths: PagePathMap,
  scope: DataSetScope,
): void {
  let current = inherited;

  if (component.type === "page") {
    const pagePath = paths.get(component) ?? "";
    const datasets = (component.props as Record<string, unknown> | undefined)?.datasets as
      | readonly ExternalDataSetDef[]
      | undefined;

    current = new Map(inherited);
    if (datasets) {
      for (const ds of datasets) {
        current.set(ds.uuid, ds);
      }
    }
    scope.set(pagePath, current);
  }

  if (component.items) {
    for (const item of component.items) {
      walkScope(item.component, current, paths, scope);
    }
  }

  if (component.slots) {
    for (const children of Object.values(component.slots)) {
      for (const child of children) {
        walkScope(child, current, paths, scope);
      }
    }
  }
}

export function resolveDataSetDef(
  dataSetId: DataSetId,
  pagePath: string,
  scope: DataSetScope,
): ExternalDataSetDef | undefined {
  let path = pagePath;
  while (true) {
    const pageScope = scope.get(path);
    if (pageScope) {
      const def = pageScope.get(dataSetId);
      if (def) return def;
    }
    if (path === "") return undefined;
    const lastSlash = path.lastIndexOf("/");
    path = lastSlash === -1 ? "" : path.substring(0, lastSlash);
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/runtime run test -- dataset-scope`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(@casehub/runtime): dataset-scope module

Pure function buildDataSetScope() produces a pagePath → dataSetId →
ExternalDataSetDef map with cascading inheritance. resolveDataSetDef()
walks up ancestors until a match is found.

Refs #13
```

---

### Task 9: `@casehub/runtime` — URL serialization module

Pure functions `serializeToUrl` and `parseFromUrl` using `DeepLink` as the domain type.

**Files:**
- Create: `packages/casehub-runtime/src/url.ts`
- Create: `packages/casehub-runtime/src/url.test.ts`

- [ ] **Step 1: Write tests**

Create `packages/casehub-runtime/src/url.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { DeepLink } from "@casehub/ui/dist/model/page-types.js";
import { serializeToUrl, parseFromUrl } from "./url.js";

describe("serializeToUrl", () => {
  it("page path only", () => {
    const link: DeepLink = { page: "Sales/Revenue" };
    expect(serializeToUrl(link)).toBe("#/page/Sales/Revenue");
  });

  it("page path with single filter", () => {
    const link: DeepLink = { page: "Overview", filters: { region: ["North"] } };
    expect(serializeToUrl(link)).toBe("#/page/Overview?filter=region:North");
  });

  it("multi-value filter uses pipe separator", () => {
    const link: DeepLink = { page: "Overview", filters: { region: ["North", "South"] } };
    expect(serializeToUrl(link)).toBe("#/page/Overview?filter=region:North|South");
  });

  it("multiple filter columns separated by comma", () => {
    const link: DeepLink = {
      page: "Overview",
      filters: { region: ["North"], year: ["2024"] },
    };
    const url = serializeToUrl(link);
    expect(url).toContain("#/page/Overview?filter=");
    expect(url).toContain("region:North");
    expect(url).toContain("year:2024");
  });

  it("empty filters omitted", () => {
    const link: DeepLink = { page: "Home", filters: {} };
    expect(serializeToUrl(link)).toBe("#/page/Home");
  });

  it("root page (empty path)", () => {
    const link: DeepLink = { page: "" };
    expect(serializeToUrl(link)).toBe("#/page/");
  });
});

describe("parseFromUrl", () => {
  it("parses page path", () => {
    const link = parseFromUrl("#/page/Sales/Revenue");
    expect(link.page).toBe("Sales/Revenue");
    expect(link.filters).toBeUndefined();
  });

  it("parses single filter", () => {
    const link = parseFromUrl("#/page/Overview?filter=region:North");
    expect(link.page).toBe("Overview");
    expect(link.filters).toEqual({ region: ["North"] });
  });

  it("parses multi-value filter", () => {
    const link = parseFromUrl("#/page/Overview?filter=region:North|South");
    expect(link.filters).toEqual({ region: ["North", "South"] });
  });

  it("parses multiple filter columns", () => {
    const link = parseFromUrl("#/page/Overview?filter=region:North,year:2024");
    expect(link.filters).toEqual({ region: ["North"], year: ["2024"] });
  });

  it("empty hash returns root page", () => {
    const link = parseFromUrl("");
    expect(link.page).toBe("");
  });

  it("hash without /page/ prefix returns root", () => {
    const link = parseFromUrl("#/something");
    expect(link.page).toBe("");
  });
});

describe("round-trip", () => {
  it("serialize then parse produces same DeepLink", () => {
    const original: DeepLink = {
      page: "Sales/Revenue",
      filters: { region: ["North", "South"], year: ["2024"] },
    };
    const url = serializeToUrl(original);
    const parsed = parseFromUrl(url);
    expect(parsed.page).toBe(original.page);
    expect(parsed.filters).toEqual(original.filters);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- url`
Expected: FAIL.

- [ ] **Step 3: Implement URL functions**

Create `packages/casehub-runtime/src/url.ts`:

```typescript
import type { DeepLink } from "@casehub/ui/dist/model/page-types.js";

export function serializeToUrl(link: DeepLink): string {
  let url = `#/page/${link.page}`;

  if (link.filters) {
    const entries = Object.entries(link.filters).filter(([, v]) => v.length > 0);
    if (entries.length > 0) {
      const filterStr = entries
        .map(([col, values]) => `${encodeURIComponent(col)}:${values.map(encodeURIComponent).join("|")}`)
        .join(",");
      url += `?filter=${filterStr}`;
    }
  }

  return url;
}

export function parseFromUrl(hash: string): DeepLink {
  if (!hash || !hash.startsWith("#/page/")) {
    return { page: "" };
  }

  const withoutPrefix = hash.substring("#/page/".length);
  const qIndex = withoutPrefix.indexOf("?");
  const page = qIndex === -1 ? withoutPrefix : withoutPrefix.substring(0, qIndex);

  let filters: Record<string, readonly string[]> | undefined;

  if (qIndex !== -1) {
    const queryStr = withoutPrefix.substring(qIndex + 1);
    const params = new URLSearchParams(queryStr);
    const filterStr = params.get("filter");
    if (filterStr) {
      filters = {};
      for (const entry of filterStr.split(",")) {
        const colonIdx = entry.indexOf(":");
        if (colonIdx === -1) continue;
        const col = decodeURIComponent(entry.substring(0, colonIdx));
        const values = entry.substring(colonIdx + 1).split("|").map(decodeURIComponent);
        filters[col] = values;
      }
    }
  }

  return { page, ...(filters ? { filters } : {}) };
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/runtime run test -- url`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(@casehub/runtime): URL serialization module

Pure functions serializeToUrl() and parseFromUrl() using DeepLink as
the domain type. Filter values pipe-separated, columns comma-separated.
Sort excluded from URL (component-scoped ephemeral state).

Refs #13
```

---

### Task 10: `@casehub/runtime` — registry and activation modules

ComponentRegistry type + activation logic (the `onNode` callback that classifies components and creates viz Web Components).

**Files:**
- Create: `packages/casehub-runtime/src/registry.ts`
- Create: `packages/casehub-runtime/src/activation.ts`
- Create: `packages/casehub-runtime/src/activation.test.ts`
- Create: `packages/casehub-runtime/src/content.ts`

- [ ] **Step 1: Create registry types**

Create `packages/casehub-runtime/src/registry.ts`:

```typescript
import type { Component } from "@casehub/component/dist/model/types.js";
import type { DataSetLookup } from "@casehub/data/dist/dataset/lookup.js";
import type { CasehubElement } from "@casehub/viz/dist/base/CasehubElement.js";
import type { VizComponentProps } from "@casehub/viz/dist/base/types.js";

export interface ComponentEntry {
  readonly element: HTMLElement;
  readonly vizElement?: CasehubElement<VizComponentProps>;
  readonly component: Component;
  readonly pagePath: string;
  readonly originalLookup?: DataSetLookup;
}

export type ComponentRegistry = Map<string, ComponentEntry>;
```

- [ ] **Step 2: Create content rendering helpers**

Create `packages/casehub-runtime/src/content.ts`:

```typescript
export function renderTitle(el: HTMLElement, props: Record<string, unknown>): void {
  const text = typeof props.text === "string" ? props.text : "";
  const size = typeof props.size === "string" ? props.size : "h1";
  const validSizes = ["h1", "h2", "h3", "h4", "h5", "h6"];
  const tag = validSizes.includes(size) ? size : "h1";
  const heading = document.createElement(tag);
  heading.textContent = text;
  el.appendChild(heading);
}

export function renderHtml(el: HTMLElement, props: Record<string, unknown>): void {
  if (typeof props.content === "string") {
    el.innerHTML = props.content;
  }
}

export function renderMarkdown(el: HTMLElement, props: Record<string, unknown>): void {
  const pre = document.createElement("pre");
  pre.textContent = typeof props.content === "string" ? props.content : "";
  el.appendChild(pre);
}
```

- [ ] **Step 3: Write activation tests**

Create `packages/casehub-runtime/src/activation.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "@casehub/component/dist/model/types.js";
import { createActivationCallback } from "./activation.js";
import type { ComponentRegistry } from "./registry.js";
import type { PagePathMap } from "./page-paths.js";

const DATA_TYPES = [
  "bar-chart", "line-chart", "area-chart", "pie-chart",
  "scatter-chart", "bubble-chart", "timeseries",
  "table", "metric", "meter", "selector", "map",
];

describe("createActivationCallback", () => {
  function setup(component: Component) {
    const registry: ComponentRegistry = new Map();
    const pagePathMap: PagePathMap = new Map([[component, "TestPage"]]);
    const callback = createActivationCallback(registry, pagePathMap);
    const el = document.createElement("div");
    el.dataset.componentId = "test-id";
    el.dataset.componentType = component.type;
    document.body.appendChild(el);
    callback(el, component);
    document.body.removeChild(el);
    return { registry, el };
  }

  for (const type of DATA_TYPES) {
    it(`creates casehub-${type} element for ${type}`, () => {
      const component: Component = { type, props: { lookup: { dataSetId: "ds", operations: [] } } };
      const { el } = setup(component);
      const vizEl = el.querySelector(`casehub-${type}`);
      expect(vizEl).toBeTruthy();
    });
  }

  it("registers data component in ComponentRegistry", () => {
    const component: Component = {
      type: "bar-chart",
      props: { lookup: { dataSetId: "ds", operations: [] } },
    };
    const { registry } = setup(component);
    expect(registry.get("test-id")).toBeTruthy();
    expect(registry.get("test-id")!.pagePath).toBe("TestPage");
    expect(registry.get("test-id")!.originalLookup).toEqual({ dataSetId: "ds", operations: [] });
  });

  it("renders title as heading element", () => {
    const component: Component = { type: "title", props: { text: "Hello", size: "h2" } };
    const { el } = setup(component);
    const heading = el.querySelector("h2");
    expect(heading).toBeTruthy();
    expect(heading!.textContent).toBe("Hello");
  });

  it("renders html content", () => {
    const component: Component = { type: "html", props: { content: "<b>bold</b>" } };
    const { el } = setup(component);
    expect(el.querySelector("b")?.textContent).toBe("bold");
  });

  it("renders markdown as pre", () => {
    const component: Component = { type: "markdown", props: { content: "# Hello" } };
    const { el } = setup(component);
    expect(el.querySelector("pre")?.textContent).toBe("# Hello");
  });

  it("does not activate layout types", () => {
    const component: Component = { type: "grid", props: { columns: 12 } };
    const { registry } = setup(component);
    expect(registry.size).toBe(0);
  });

  it("does not activate unknown types", () => {
    const component: Component = { type: "custom-widget" };
    const { registry } = setup(component);
    expect(registry.size).toBe(0);
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- activation`
Expected: FAIL.

- [ ] **Step 5: Implement createActivationCallback**

Create `packages/casehub-runtime/src/activation.ts`:

```typescript
import type { Component } from "@casehub/component/dist/model/types.js";
import type { DataSetLookup } from "@casehub/data/dist/dataset/lookup.js";
import type { CasehubElement } from "@casehub/viz/dist/base/CasehubElement.js";
import type { VizComponentProps } from "@casehub/viz/dist/base/types.js";
import type { ComponentRegistry } from "./registry.js";
import type { PagePathMap } from "./page-paths.js";
import { renderTitle, renderHtml, renderMarkdown } from "./content.js";

const DATA_COMPONENT_TYPES = new Set([
  "bar-chart", "line-chart", "area-chart", "pie-chart",
  "scatter-chart", "bubble-chart", "timeseries",
  "table", "metric", "meter", "selector", "map",
  "iframe-plugin",
]);

const LAYOUT_TYPES = new Set([
  "grid", "columns", "rows", "stack",
  "tabs", "pills", "sidebar", "accordion", "carousel",
  "app-grid", "panel",
]);

export function createActivationCallback(
  registry: ComponentRegistry,
  pagePathMap: PagePathMap,
): (el: HTMLElement, component: Component) => void {
  return (el: HTMLElement, component: Component): void => {
    const componentId = el.dataset.componentId;
    if (!componentId) return;

    const pagePath = pagePathMap.get(component) ?? "";

    if (DATA_COMPONENT_TYPES.has(component.type)) {
      const tagName = `casehub-${component.type}`;
      const vizEl = document.createElement(tagName) as CasehubElement<VizComponentProps>;
      if (component.props) {
        vizEl.props = component.props as VizComponentProps;
      }
      el.appendChild(vizEl);

      const lookup = (component.props as Record<string, unknown> | undefined)?.lookup as
        | DataSetLookup
        | undefined;

      registry.set(componentId, {
        element: el,
        vizElement: vizEl,
        component,
        pagePath,
        originalLookup: lookup,
      });
      return;
    }

    if (component.type === "title" && component.props) {
      renderTitle(el, component.props as Record<string, unknown>);
      return;
    }

    if (component.type === "html" && component.props) {
      renderHtml(el, component.props as Record<string, unknown>);
      return;
    }

    if (component.type === "markdown" && component.props) {
      renderMarkdown(el, component.props as Record<string, unknown>);
      return;
    }

    // Layout, page, lazy-page, unknown: no activation
  };
}
```

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehub/runtime run test -- activation`
Expected: All pass.

- [ ] **Step 7: Commit**

```
feat(@casehub/runtime): registry and activation modules

ComponentRegistry maps componentId → ComponentEntry. Activation callback
classifies components by type and creates viz Web Components, renders
content, or skips layout types. Tag name rule: "casehub-" + type.

Refs #13
```

---

### Task 11: `@casehub/runtime` — data-pipeline module

Event handler for `casehub-data-request` with on-demand resolution, deduplication, and filter-aware data push.

**Files:**
- Create: `packages/casehub-runtime/src/data-pipeline.ts`
- Create: `packages/casehub-runtime/src/data-pipeline.test.ts`
- Create: `packages/casehub-runtime/src/cross-filter.ts`

- [ ] **Step 1: Create cross-filter types and helpers**

Create `packages/casehub-runtime/src/cross-filter.ts`:

```typescript
import type { DataSetOp } from "@casehub/data/dist/dataset/ops.js";

export type FilterState = Map<string, Map<string | undefined, Map<string, string[]>>>;

export function createFilterState(): FilterState {
  return new Map();
}

export function getActiveFilterOps(
  filterState: FilterState,
  pagePath: string,
  group: string | undefined,
): DataSetOp[] {
  const pageFilters = filterState.get(pagePath);
  if (!pageFilters) return [];

  const ops: DataSetOp[] = [];
  const groups = group !== undefined
    ? [pageFilters.get(group)]
    : [...pageFilters.values()];

  // Also include ungrouped filters for grouped components
  if (group !== undefined) {
    const ungrouped = pageFilters.get(undefined);
    if (ungrouped) groups.push(ungrouped);
  }

  for (const columnMap of groups) {
    if (!columnMap) continue;
    for (const [columnId, values] of columnMap) {
      for (const value of values) {
        ops.push({
          type: "filter" as const,
          expressions: [{
            type: "CORE",
            columnId,
            function: { type: "EQUALS_TO", args: [value] },
          }],
        });
      }
    }
  }

  return ops;
}
```

- [ ] **Step 2: Write data-pipeline tests**

Create `packages/casehub-runtime/src/data-pipeline.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import type { DataSetId, TypedDataSet } from "@casehub/data/dist/dataset/types.js";
import { createDataSetManager } from "@casehub/data/dist/dataset/manager.js";
import { createDataPipeline } from "./data-pipeline.js";
import type { ComponentRegistry } from "./registry.js";
import type { DataSetScope } from "./dataset-scope.js";
import { createFilterState } from "./cross-filter.js";

function makeDataSet(rows: number): TypedDataSet {
  return {
    columns: [{ id: "col1" as any, type: "TEXT", settings: { id: "col1" as any } }],
    rows: Array.from({ length: rows }, (_, i) => ({
      cell(colId: string) { return { type: "TEXT" as const, value: `row${i}` }; },
      cells: [{ type: "TEXT" as const, value: `row${i}` }],
    })),
  };
}

describe("createDataPipeline", () => {
  it("resolves data-request for registered dataset", () => {
    const manager = createDataSetManager();
    const ds = makeDataSet(3);
    manager.register("sales" as DataSetId, ds);

    const registry: ComponentRegistry = new Map();
    registry.set("chart-1", {
      element: document.createElement("div"),
      component: { type: "bar-chart" },
      pagePath: "",
    });

    const pipeline = createDataPipeline(manager, new Map() as DataSetScope, registry, createFilterState());

    const mockElement = { dataSet: undefined as any, totalRows: -1, theme: "", error: "" };
    const lookup = { dataSetId: "sales" as DataSetId, operations: [] };

    pipeline.handleDataRequest(mockElement as any, lookup, "chart-1");

    expect(mockElement.dataSet).toBeTruthy();
    expect(mockElement.totalRows).toBe(3);
  });

  it("sets error for unknown dataset with no scope entry", () => {
    const manager = createDataSetManager();
    const registry: ComponentRegistry = new Map();
    registry.set("chart-1", {
      element: document.createElement("div"),
      component: { type: "bar-chart" },
      pagePath: "",
    });

    const pipeline = createDataPipeline(manager, new Map() as DataSetScope, registry, createFilterState());

    const mockElement = { dataSet: undefined as any, totalRows: -1, theme: "", error: "" };
    const lookup = { dataSetId: "unknown" as DataSetId, operations: [] };

    pipeline.handleDataRequest(mockElement as any, lookup, "chart-1");

    expect(mockElement.error).toContain("unknown");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- data-pipeline`
Expected: FAIL.

- [ ] **Step 4: Implement data-pipeline**

Create `packages/casehub-runtime/src/data-pipeline.ts`:

```typescript
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import type { DataSetManager, LookupResult } from "@casehub/data/dist/dataset/manager.js";
import type { DataSetLookup } from "@casehub/data/dist/dataset/lookup.js";
import type { ResolverContext } from "@casehub/data/dist/dataset/external/resolver.js";
import type { ResolveResult } from "@casehub/data/dist/dataset/external/types.js";
import type { CasehubElement } from "@casehub/viz/dist/base/CasehubElement.js";
import type { VizComponentProps } from "@casehub/viz/dist/base/types.js";
import type { ComponentRegistry } from "./registry.js";
import type { DataSetScope } from "./dataset-scope.js";
import { resolveDataSetDef } from "./dataset-scope.js";
import type { FilterState } from "./cross-filter.js";
import { getActiveFilterOps } from "./cross-filter.js";

export interface DataPipeline {
  handleDataRequest(
    element: CasehubElement<VizComponentProps>,
    lookup: DataSetLookup,
    componentId: string,
  ): void;
  readonly pendingResolutions: Map<DataSetId, Promise<ResolveResult>>;
  resolverCtx?: ResolverContext;
  refreshTimers: Map<DataSetId, ReturnType<typeof setInterval>>;
}

export function createDataPipeline(
  manager: DataSetManager,
  scope: DataSetScope,
  registry: ComponentRegistry,
  filterState: FilterState,
): DataPipeline {
  const pendingResolutions = new Map<DataSetId, Promise<ResolveResult>>();
  const refreshTimers = new Map<DataSetId, ReturnType<typeof setInterval>>();

  const pipeline: DataPipeline = {
    pendingResolutions,
    refreshTimers,

    handleDataRequest(
      element: CasehubElement<VizComponentProps>,
      lookup: DataSetLookup,
      componentId: string,
    ): void {
      const entry = registry.get(componentId);
      if (!entry) return;

      if (manager.has(lookup.dataSetId)) {
        pushData(element, lookup, entry.pagePath, entry.component);
        return;
      }

      const def = resolveDataSetDef(lookup.dataSetId, entry.pagePath, scope);
      if (!def) {
        element.error = `Dataset "${lookup.dataSetId}" not found in scope for page "${entry.pagePath}"`;
        return;
      }

      let pending = pendingResolutions.get(lookup.dataSetId);
      if (!pending && pipeline.resolverCtx) {
        const { resolveExternalDataSet } = require("@casehub/data/dist/dataset/external/resolver.js");
        pending = resolveExternalDataSet(def, pipeline.resolverCtx) as Promise<ResolveResult>;
        pendingResolutions.set(lookup.dataSetId, pending);
      }

      if (pending) {
        pending
          .then(() => {
            pendingResolutions.delete(lookup.dataSetId);
            pushData(element, lookup, entry.pagePath, entry.component);
          })
          .catch((err: Error) => {
            pendingResolutions.delete(lookup.dataSetId);
            element.error = err.message;
          });
      }
    },
  };

  function pushData(
    element: CasehubElement<VizComponentProps>,
    lookup: DataSetLookup,
    pagePath: string,
    component: { type: string; props?: Readonly<Record<string, unknown>> },
  ): void {
    try {
      const filterProps = component.props?.filter as { group?: string } | undefined;
      const filterOps = getActiveFilterOps(filterState, pagePath, filterProps?.group);
      const effectiveOps = [...filterOps, ...lookup.operations];
      const effectiveLookup: DataSetLookup = { ...lookup, operations: effectiveOps };
      const result: LookupResult = manager.lookup(effectiveLookup);
      element.dataSet = result.dataset;
      element.totalRows = result.totalRows;
    } catch (err) {
      element.error = err instanceof Error ? err.message : String(err);
    }
  }

  return pipeline;
}
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehub/runtime run test -- data-pipeline`
Expected: All pass.

- [ ] **Step 6: Commit**

```
feat(@casehub/runtime): data-pipeline and cross-filter modules

On-demand dataset resolution with deduplication. Filter-aware data push
unifies deep-link and interactive filter code paths. CrossFilter state
is pagePath → group → column → values.

Refs #13
```

---

### Task 12: `@casehub/runtime` — navigation module

PageIndex, ActiveSlots tracking, current page computation.

**Files:**
- Create: `packages/casehub-runtime/src/navigation.ts`
- Create: `packages/casehub-runtime/src/navigation.test.ts`

- [ ] **Step 1: Write tests**

Create `packages/casehub-runtime/src/navigation.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "@casehub/component/dist/model/types.js";
import { buildPageIndex, computeCurrentPage } from "./navigation.js";
import { buildPagePathMap } from "./page-paths.js";

describe("buildPageIndex", () => {
  it("maps page paths to components", () => {
    const sales: Component = { type: "page", props: { name: "Sales" } };
    const root: Component = {
      type: "page",
      props: { name: "App" },
      slots: { Sales: [sales] },
    };
    const paths = buildPagePathMap(root);
    const index = buildPageIndex(root, paths);
    expect(index.get("")).toBe(root);
    expect(index.get("Sales")).toBe(sales);
  });
});

describe("computeCurrentPage", () => {
  it("returns empty string with no active slots", () => {
    const root: Component = { type: "page", props: { name: "App" } };
    const activeSlots = new Map<string, string>();
    const result = computeCurrentPage(root, activeSlots);
    expect(result).toBe("");
  });

  it("returns single segment for one-level navigation", () => {
    const root: Component = {
      type: "page",
      props: { name: "App" },
      slots: {
        default: [{
          type: "sidebar",
          id: "nav-1",
          slots: {
            Overview: [{ type: "page", props: { name: "Overview" } }],
            Sales: [{ type: "page", props: { name: "Sales" } }],
          },
        }],
      },
    };
    const activeSlots = new Map([["nav-1", "Sales"]]);
    const result = computeCurrentPage(root, activeSlots);
    expect(result).toBe("Sales");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- navigation`
Expected: FAIL.

- [ ] **Step 3: Implement navigation module**

Create `packages/casehub-runtime/src/navigation.ts`:

```typescript
import type { Component } from "@casehub/component/dist/model/types.js";
import type { PagePathMap } from "./page-paths.js";

export type PageIndex = Map<string, Component>;
export type ActiveSlots = Map<string, string>;

const INTERACTIVE_TYPES = new Set([
  "tabs", "pills", "sidebar", "accordion", "carousel", "stack",
]);

export function buildPageIndex(
  root: Component,
  paths: PagePathMap,
): PageIndex {
  const index: PageIndex = new Map();
  walkPages(root, paths, index);
  return index;
}

function walkPages(
  component: Component,
  paths: PagePathMap,
  index: PageIndex,
): void {
  if (component.type === "page") {
    const path = paths.get(component) ?? "";
    index.set(path, component);
  }

  if (component.items) {
    for (const item of component.items) {
      walkPages(item.component, paths, index);
    }
  }

  if (component.slots) {
    for (const children of Object.values(component.slots)) {
      for (const child of children) {
        walkPages(child, paths, index);
      }
    }
  }
}

export function computeCurrentPage(
  root: Component,
  activeSlots: ActiveSlots,
): string {
  const segments: string[] = [];
  walkActive(root, activeSlots, segments);
  return segments.join("/");
}

function walkActive(
  component: Component,
  activeSlots: ActiveSlots,
  segments: string[],
): void {
  if (!component.slots) return;

  if (INTERACTIVE_TYPES.has(component.type) && component.id) {
    const activeSlot = activeSlots.get(component.id);
    if (activeSlot) {
      const children = component.slots[activeSlot];
      if (children) {
        for (const child of children) {
          if (child.type === "page") {
            segments.push(activeSlot);
          }
          walkActive(child, activeSlots, segments);
        }
      }
      return;
    }
  }

  for (const children of Object.values(component.slots)) {
    for (const child of children) {
      walkActive(child, activeSlots, segments);
    }
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/runtime run test -- navigation`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(@casehub/runtime): navigation module

PageIndex maps pagePath → Component. computeCurrentPage walks the
component tree collecting active slot names at navigation containers
to produce the current page path.

Refs #13
```

---

### Task 13: `@casehub/runtime` — site orchestrator (loadSite + LiveSite)

The main orchestrator that ties everything together: `loadSite()`, event delegation, navigate(), dispose().

**Files:**
- Create: `packages/casehub-runtime/src/site.ts`
- Create: `packages/casehub-runtime/src/site.test.ts`
- Modify: `packages/casehub-runtime/src/index.ts`

- [ ] **Step 1: Write integration tests**

Create `packages/casehub-runtime/src/site.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "@casehub/component/dist/model/types.js";
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import { loadSite } from "./site.js";
import type { LiveSite } from "./site.js";

function simpleSite(): Component {
  return {
    type: "page",
    props: {
      name: "App",
      datasets: [{
        uuid: "sales" as DataSetId,
        content: JSON.stringify([
          { region: "North", revenue: 100 },
          { region: "South", revenue: 200 },
        ]),
      }],
    },
    slots: {
      default: [
        { type: "title", props: { text: "Dashboard" } },
      ],
    },
  };
}

describe("loadSite", () => {
  it("renders component tree into target", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, simpleSite());
    expect(target.children.length).toBeGreaterThan(0);
    expect(target.querySelector("[data-component-type='page']")).toBeTruthy();
    site.dispose();
    document.body.removeChild(target);
  });

  it("returns working Site interface", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, simpleSite());
    expect(site.root).toBeTruthy();
    expect(site.root.type).toBe("page");
    expect(site.state).toBeTruthy();
    site.dispose();
    document.body.removeChild(target);
  });

  it("page() returns component by path", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, simpleSite());
    const root = site.page("");
    expect(root).toBeTruthy();
    expect(root?.type).toBe("page");
    site.dispose();
    document.body.removeChild(target);
  });

  it("dataset() resolves from page scope", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, simpleSite());
    const ds = site.dataset("sales" as DataSetId);
    expect(ds).toBeTruthy();
    expect(ds?.uuid).toBe("sales");
    site.dispose();
    document.body.removeChild(target);
  });

  it("dispose clears target", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, simpleSite());
    site.dispose();
    expect(target.children.length).toBe(0);
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- site`
Expected: FAIL.

- [ ] **Step 3: Implement loadSite and SiteImpl**

Create `packages/casehub-runtime/src/site.ts`:

```typescript
import type { Component, PermissionContext } from "@casehub/component/dist/model/types.js";
import { ALLOW_ALL } from "@casehub/component/dist/model/types.js";
import { renderComponent } from "@casehub/component/dist/renderer/render.js";
import { activateSlot } from "@casehub/component/dist/renderer/activate-slot.js";
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import type { DataProviderConfig, ExternalDataSetDef } from "@casehub/data/dist/dataset/external/types.js";
import { createDataSetManager } from "@casehub/data/dist/dataset/manager.js";
import { createDataProviderFactory, createPresetRegistry } from "@casehub/data/dist/dataset/external/index.js";
import type { Site, ViewState, DeepLink } from "@casehub/ui/dist/model/page-types.js";
import type { CasehubDataRequestDetail } from "@casehub/viz/dist/base/CasehubElement.js";
import { parsePage } from "@casehub/ui/dist/parser/page-parser.js";
import { buildPagePathMap } from "./page-paths.js";
import { buildDataSetScope, resolveDataSetDef } from "./dataset-scope.js";
import { buildPageIndex, computeCurrentPage } from "./navigation.js";
import type { ActiveSlots } from "./navigation.js";
import { createActivationCallback } from "./activation.js";
import type { ComponentRegistry } from "./registry.js";
import { createDataPipeline } from "./data-pipeline.js";
import { createFilterState } from "./cross-filter.js";
import { serializeToUrl, parseFromUrl } from "./url.js";

export interface LiveSite extends Site {
  navigate(path: string): void;
  dispose(): void;
}

export interface SiteOptions {
  readonly permissions?: PermissionContext;
  readonly fetch?: typeof globalThis.fetch;
  readonly providerConfig?: DataProviderConfig;
}

export async function loadSite(
  target: HTMLElement,
  source: string | Component,
  options?: SiteOptions,
): Promise<LiveSite> {
  const root = typeof source === "string" ? parsePage(source) : source;
  const permissions = options?.permissions ?? ALLOW_ALL;

  // Pre-compute indices
  const pagePathMap = buildPagePathMap(root);
  const dataSetScope = buildDataSetScope(root, pagePathMap);
  const pageIndex = buildPageIndex(root, pagePathMap);

  // Create runtime state
  const registry: ComponentRegistry = new Map();
  const activeSlots: ActiveSlots = new Map();
  const filterState = createFilterState();
  const abortController = new AbortController();
  const manager = createDataSetManager();

  // Data pipeline
  const pipeline = createDataPipeline(manager, dataSetScope, registry, filterState);
  pipeline.resolverCtx = {
    manager,
    providerFactory: createDataProviderFactory(options?.fetch),
    providerConfig: options?.providerConfig ?? {},
    presetRegistry: createPresetRegistry(),
  };

  // Navigation state
  let _navigating = false;
  let currentPage = "";

  // Render
  const onNode = createActivationCallback(registry, pagePathMap);
  renderComponent(target, root, { permissions, onNode });

  // Event delegation
  target.addEventListener("casehub-data-request", ((e: CustomEvent<CasehubDataRequestDetail>) => {
    const { element, lookup } = e.detail;
    const el = e.target as HTMLElement;
    const componentId = el.closest("[data-component-id]")?.getAttribute("data-component-id") ??
      (element as unknown as HTMLElement).parentElement?.dataset.componentId;
    if (componentId) {
      pipeline.handleDataRequest(element, lookup, componentId);
    }
  }) as EventListener, { signal: abortController.signal });

  target.addEventListener("casehub-slot-change", ((e: CustomEvent) => {
    const { activeSlot, containerId } = e.detail;
    activeSlots.set(containerId, activeSlot);
    currentPage = computeCurrentPage(root, activeSlots);
    if (!_navigating && typeof history !== "undefined") {
      const link: DeepLink = { page: currentPage };
      history.pushState(null, "", serializeToUrl(link));
    }
  }) as EventListener, { signal: abortController.signal });

  // ViewState
  const state: ViewState = {
    get currentPage() { return currentPage; },
  } as ViewState;

  // Site handle
  const site: LiveSite = {
    root,

    page(path: string): Component | null {
      return pageIndex.get(path) ?? null;
    },

    dataset(id: DataSetId, fromPage?: string): ExternalDataSetDef | null {
      return resolveDataSetDef(id, fromPage ?? currentPage, dataSetScope) ?? null;
    },

    state,

    navigate(path: string): void {
      _navigating = true;
      const segments = path.split("/").filter(Boolean);
      // Walk tree to find navigation containers and activate slots
      // Simplified: for each segment, find container and activate
      let reached = "";
      for (const segment of segments) {
        const containers = target.querySelectorAll<HTMLElement>(
          "[data-component-type='tabs'], [data-component-type='pills'], [data-component-type='sidebar'], [data-component-type='accordion'], [data-component-type='carousel'], [data-component-type='stack']",
        );
        let found = false;
        for (const container of containers) {
          if (activateSlot(container, segment)) {
            reached = reached ? `${reached}/${segment}` : segment;
            found = true;
            break;
          }
        }
        if (!found) break;
      }
      currentPage = reached;
      _navigating = false;
      if (typeof history !== "undefined") {
        const link: DeepLink = { page: currentPage };
        history.pushState(null, "", serializeToUrl(link));
      }
    },

    dispose(): void {
      abortController.abort();
      for (const timer of pipeline.refreshTimers.values()) {
        clearInterval(timer);
      }
      pipeline.refreshTimers.clear();
      registry.clear();
      target.innerHTML = "";
    },
  };

  // Apply initial URL state
  if (typeof location !== "undefined" && location.hash) {
    const deepLink = parseFromUrl(location.hash);
    if (deepLink.page) {
      site.navigate(deepLink.page);
    }
  }

  return site;
}
```

- [ ] **Step 4: Update index.ts**

In `packages/casehub-runtime/src/index.ts`:

```typescript
export { loadSite } from "./site.js";
export type { LiveSite, SiteOptions } from "./site.js";
export { serializeToUrl, parseFromUrl } from "./url.js";
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehub/runtime run test`
Expected: All tests pass across all modules.

- [ ] **Step 6: Run full package build**

Run: `yarn workspace @casehub/runtime run build`
Expected: TypeScript compiles without errors.

- [ ] **Step 7: Commit**

```
feat(@casehub/runtime): implement loadSite() and LiveSite

Main site orchestrator: loadSite() renders the component tree, activates
viz/content components, sets up event delegation for data-request and
slot-change, builds page/dataset indices, and returns a LiveSite handle
with navigate() and dispose().

Refs #13
```

---

### Task 14: Build verification and cross-package test

Verify the full build chain works and all tests pass across all modified packages.

**Files:** None (verification only)

- [ ] **Step 1: Build all packages in order**

Run: `yarn build:packages`
Expected: All packages build successfully.

- [ ] **Step 2: Run all tests**

Run: `yarn build:packages && yarn workspace @casehub/data run test && yarn workspace @casehub/component run test && yarn workspace @casehub/runtime run test`
Expected: All tests pass.

- [ ] **Step 3: Update CLAUDE.md build commands**

In `CLAUDE.md`, add `@casehub/runtime` to the package build order note:

```
# Shared TypeScript packages only (order matters: @casehub/component → @casehub/data → @casehub/ui → @casehub/viz → @casehub/runtime)
```

- [ ] **Step 4: Commit**

```
chore: add @casehub/runtime to build order documentation

Refs #13
```
