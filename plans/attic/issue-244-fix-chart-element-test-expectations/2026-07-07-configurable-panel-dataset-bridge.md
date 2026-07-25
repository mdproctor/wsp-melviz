# ConfigurablePanel Interface & Dataset Pipeline Bridge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #109 — Export ConfigurablePanel interface for hosted Web Components
**Issue group:** #109, #110 (closing epic #111)

**Goal:** Formalize the hosting contract for Web Components mounted via `host-panel`, and bridge them into the pages dataset pipeline so they receive data through the same mechanism as viz components.

**Architecture:** Two new interfaces (`ConfigurablePanel`, `DataReceiver`) in `pages-component`. `VizTarget` in `pages-runtime` extends `DataReceiver`, removing `theme`. `ComponentEntry.vizElement` widens from `PagesElement<VizComponentProps>` to `VizTarget`. Activation creates a proxy adapter for data-bound host panels — zero pipeline logic changes. `handleSubtreeRemoved` uses `entry.element` for cleanup instead of casting `vizElement`.

**Tech Stack:** TypeScript 5, Vitest

## Global Constraints

- `strict: true` in all tsconfigs — setter contravariance is enforced
- PagesElement does NOT implement DataReceiver explicitly (strict-mode violation)
- No parser changes — YAML produces generic Component objects
- `pages-component` already depends on `pages-data` (`package.json` line 24)
- Protocol update required: `pages-event-contract.md` reserved events table

---

## File Map

| File | Action | Task |
|------|--------|------|
| `packages/pages-component/src/model/component-props.ts` | Modify — add `DataSetLookup` import, add `lookup` to `HostPanelProps` | 1 |
| `packages/pages-component/src/model/hosting.ts` | Create — `ConfigurablePanel`, `DataReceiver` interfaces | 1 |
| `packages/pages-component/src/model/hosting.test.ts` | Create — compile-time contract tests | 1 |
| `packages/pages-component/src/model/index.ts` | Modify — export new interfaces | 1 |
| `packages/pages-runtime/src/data-pipeline.ts` | Modify — `VizTarget extends DataReceiver`, remove `theme` | 2 |
| `packages/pages-runtime/src/registry.ts` | Modify — `vizElement: VizTarget`, update imports | 2 |
| `packages/pages-runtime/src/data-pipeline.test.ts` | Modify — add cleanup test for entry.element path | 2 |
| `packages/pages-runtime/src/activation.ts` | Modify — ConfigurablePanel cast, proxy, data bridge | 3 |
| `packages/pages-runtime/src/activation-host-panel.test.ts` | Modify — tests for ConfigurablePanel, proxy, data request | 3 |
| `components/pages-component-terminal/src/PagesTerminal.ts` | Modify — `implements ConfigurablePanel<TerminalProps>` | 4 |
| `components/pages-component-terminal/src/PagesTerminal.test.ts` | Modify — add ConfigurablePanel contract test | 4 |
| `docs/protocols/casehub/pages-event-contract.md` | Modify — update `pages-data-request` dispatchers | 4 |

---

### Task 1: ConfigurablePanel and DataReceiver interfaces (#109)

**Files:**
- Create: `packages/pages-component/src/model/hosting.ts`
- Create: `packages/pages-component/src/model/hosting.test.ts`
- Modify: `packages/pages-component/src/model/component-props.ts`
- Modify: `packages/pages-component/src/model/index.ts`

**Interfaces:**
- Consumes: nothing (foundational)
- Produces: `ConfigurablePanel<P>`, `DataReceiver`, updated `HostPanelProps` with `lookup?: DataSetLookup`

- [ ] **Step 1: Write compile-time contract tests for ConfigurablePanel and DataReceiver**

Create `packages/pages-component/src/model/hosting.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { ConfigurablePanel, DataReceiver } from "./hosting.js";

describe("ConfigurablePanel", () => {
  it("accepts a default Record<string, unknown> implementation", () => {
    const panel: ConfigurablePanel = {
      configure(props: Record<string, unknown>) {
        void props;
      },
    };
    panel.configure({ endpoint: "/api" });
    expect(panel).toBeDefined();
  });

  it("accepts a typed generic implementation", () => {
    interface MyProps extends Record<string, unknown> {
      endpoint: string;
      mode?: string;
    }
    const panel: ConfigurablePanel<MyProps> = {
      configure(props: MyProps) {
        void props.endpoint;
      },
    };
    panel.configure({ endpoint: "/api" });
    expect(panel).toBeDefined();
  });
});

describe("DataReceiver", () => {
  it("accepts an implementation with the mutual-clearing invariant", () => {
    let _data: unknown;
    let _error = "";
    const receiver: DataReceiver = {
      get dataSet() { return _data; },
      set dataSet(v: unknown) { _data = v; _error = ""; },
      get error() { return _error; },
      set error(v: string) { _error = v; _data = undefined; },
    };
    receiver.dataSet = [1, 2, 3];
    expect(receiver.error).toBe("");
    receiver.error = "fail";
    expect(receiver.dataSet).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- hosting.test.ts`
Expected: FAIL — cannot resolve `./hosting.js`

- [ ] **Step 3: Create hosting.ts with interfaces**

Create `packages/pages-component/src/model/hosting.ts`:

```typescript
/**
 * Pre-attachment configuration contract for hosted Web Components.
 *
 * **Call timing:** `configure(props)` is called before the element is appended
 * to the DOM — before `connectedCallback()` fires. Components should store
 * configuration without triggering rendering at this point.
 *
 * **Re-configuration:** `configure()` may be called again after initial render
 * (e.g. navigation to a different item). Implementations must handle re-entry:
 * tear down prior state and re-initialize with the new props.
 *
 * **Props content:** `props` contains the YAML `panelProps` values. The generic
 * `P` gives component authors type safety for their specific props shape; the
 * runtime calls with `Record<string, unknown>`.
 */
export interface ConfigurablePanel<P extends Record<string, unknown> = Record<string, unknown>> {
  configure(props: P): void;
}

/**
 * Data delivery contract for components receiving pipeline data.
 *
 * **Mutual-clearing invariant:** implementations must clear `error` when
 * `dataSet` is set, and clear `dataSet` when `error` is set. The pipeline
 * delivers one or the other per cycle, never both — but stale values from
 * a prior cycle must not persist alongside fresh values from the current one.
 */
export interface DataReceiver {
  dataSet: unknown;
  error: string;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-component run test -- hosting.test.ts`
Expected: PASS

- [ ] **Step 5: Add lookup to HostPanelProps**

In `packages/pages-component/src/model/component-props.ts`, add the import and the property:

Add at top (after existing imports):
```typescript
import type { DataSetLookup } from "@casehubio/pages-data/dist/dataset/lookup.js";
```

Change `HostPanelProps` to:
```typescript
export interface HostPanelProps {
  readonly typeName: string;
  readonly panelProps?: Readonly<Record<string, unknown>>;
  readonly lookup?: DataSetLookup;
}
```

Note: `displayer-types.ts` in the same package already imports `DataSetLookup` from `@casehubio/pages-data`. This follows the established pattern.

- [ ] **Step 6: Write test for HostPanelProps with lookup**

Add to `packages/pages-component/src/model/hosting.test.ts`:

```typescript
import type { HostPanelProps } from "./component-props.js";

describe("HostPanelProps", () => {
  it("accepts lookup for dataset binding", () => {
    const props: HostPanelProps = {
      typeName: "my-panel",
      panelProps: { mode: "dark" },
      lookup: { dataSetId: "workitems" as any, operations: [] },
    };
    expect(props.lookup?.dataSetId).toBe("workitems");
  });

  it("lookup is optional — existing usage without it compiles", () => {
    const props: HostPanelProps = {
      typeName: "my-panel",
    };
    expect(props.lookup).toBeUndefined();
  });
});
```

- [ ] **Step 7: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-component run test -- hosting.test.ts`
Expected: PASS

- [ ] **Step 8: Export new interfaces from model/index.ts**

In `packages/pages-component/src/model/index.ts`, add after the `component-props.js` exports:

```typescript
// Hosting contracts
export type { ConfigurablePanel, DataReceiver } from "./hosting.js";
```

- [ ] **Step 9: Run typecheck**

Run: `yarn typecheck`
Expected: PASS — no type errors across all packages

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/hosting.ts packages/pages-component/src/model/hosting.test.ts packages/pages-component/src/model/component-props.ts packages/pages-component/src/model/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-component): add ConfigurablePanel, DataReceiver interfaces and HostPanelProps.lookup

Refs #109, #110"
```

---

### Task 2: VizTarget decomposition and ComponentEntry widening

**Files:**
- Modify: `packages/pages-runtime/src/data-pipeline.ts`
- Modify: `packages/pages-runtime/src/registry.ts`
- Modify: `packages/pages-runtime/src/data-pipeline.test.ts` (or relevant test file)
- Modify: `packages/pages-runtime/src/index.ts` (export DataReceiver re-export if needed)

**Interfaces:**
- Consumes: `DataReceiver` from `@casehubio/pages-component`
- Produces: `VizTarget extends DataReceiver` (theme removed), `ComponentEntry.vizElement: VizTarget`

- [ ] **Step 1: Write test for handleSubtreeRemoved using entry.element**

Check if `data-pipeline.test.ts` exists and find the relevant test section. Add a test that verifies cleanup works when `vizElement` is a non-HTMLElement proxy:

Add to the data-pipeline test file (create if needed as `packages/pages-runtime/src/data-pipeline-cleanup.test.ts`):

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { createDataPipeline } from "./data-pipeline.js";
import { createDataSetManager } from "@casehubio/pages-data/dist/dataset/manager.js";
import type { ComponentRegistry } from "./registry.js";
import type { VizTarget } from "./data-pipeline.js";
import type { DataSetLookup } from "@casehubio/pages-data/dist/dataset/lookup.js";

describe("handleSubtreeRemoved with proxy vizElement", () => {
  it("cleans up when wrapper element is removed even if vizElement is not an HTMLElement", async () => {
    const manager = createDataSetManager({ onChanged: () => {} });
    const registry: ComponentRegistry = new Map();
    const target = document.createElement("div");
    document.body.appendChild(target);

    const wrapper = document.createElement("div");
    wrapper.dataset.componentId = "host-1";
    target.appendChild(wrapper);

    const proxy: VizTarget = {
      get dataSet() { return undefined; },
      set dataSet(_: unknown) {},
      get error() { return ""; },
      set error(_: string) {},
      get totalRows() { return 0; },
      set totalRows(_: number) {},
      get activeSort() { return undefined; },
      set activeSort(_) {},
      get activePage() { return undefined; },
      set activePage(_) {},
    };

    const lookup: DataSetLookup = { dataSetId: "test" as any, operations: [] };
    registry.set("host-1", {
      element: wrapper,
      vizElement: proxy,
      component: { type: "host-panel", props: { typeName: "test" } },
      pagePath: "",
      originalLookup: lookup,
      hasExplicitId: false,
    });

    const pipeline = createDataPipeline(
      manager, new Map(), registry, new Map(),
      new Map(), new Map(), undefined, target,
    );

    wrapper.remove();
    await new Promise(resolve => queueMicrotask(resolve));

    pipeline.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- data-pipeline-cleanup.test.ts`
Expected: FAIL — `VizTarget` doesn't exist yet with the new shape (or `ComponentEntry.vizElement` type mismatch)

- [ ] **Step 3: Modify VizTarget to extend DataReceiver and remove theme**

In `packages/pages-runtime/src/data-pipeline.ts`, replace the VizTarget definition (lines 26-33):

Old:
```typescript
export interface VizTarget {
  dataSet: unknown;
  totalRows: number;
  theme: string;
  error: string;
  activeSort: SortColumn | undefined;
  activePage: number | undefined;
}
```

New:
```typescript
import type { DataReceiver } from "@casehubio/pages-component/dist/model/hosting.js";

export interface VizTarget extends DataReceiver {
  totalRows: number;
  activeSort: SortColumn | undefined;
  activePage: number | undefined;
}
```

- [ ] **Step 4: Fix handleSubtreeRemoved to use entry.element**

In `packages/pages-runtime/src/data-pipeline.ts`, replace the `handleSubtreeRemoved` function (lines 93-100):

Old:
```typescript
  function handleSubtreeRemoved(removed: HTMLElement): void {
    const affected: Array<[string, HTMLElement]> = [];
    for (const [componentId, entry] of registry) {
      const el = entry.vizElement as unknown as HTMLElement | undefined;
      if (!el || !(el instanceof HTMLElement)) continue;
      if (removed !== el && !removed.contains(el)) continue;
      affected.push([componentId, el]);
    }
```

New:
```typescript
  function handleSubtreeRemoved(removed: HTMLElement): void {
    const affected: Array<[string, HTMLElement]> = [];
    for (const [componentId, entry] of registry) {
      const el = entry.element;
      if (removed !== el && !removed.contains(el)) continue;
      affected.push([componentId, el]);
    }
```

- [ ] **Step 5: Modify ComponentEntry.vizElement type in registry.ts**

In `packages/pages-runtime/src/registry.ts`, replace imports and interface:

Old:
```typescript
import type { Component } from "@casehubio/pages-component/dist/model/types.js";
import type { DataSetLookup } from "@casehubio/pages-data/dist/dataset/lookup.js";
import type { PagesElement } from "@casehubio/pages-viz/dist/base/PagesElement.js";
import type { VizComponentProps } from "@casehubio/pages-viz/dist/base/types.js";

export interface ComponentEntry {
  readonly element: HTMLElement;
  readonly vizElement?: PagesElement<VizComponentProps>;
  readonly component: Component;
  readonly pagePath: string;
  readonly originalLookup?: DataSetLookup;
  readonly hasExplicitId: boolean;
}
```

New:
```typescript
import type { Component } from "@casehubio/pages-component/dist/model/types.js";
import type { DataSetLookup } from "@casehubio/pages-data/dist/dataset/lookup.js";
import type { VizTarget } from "./data-pipeline.js";

export interface ComponentEntry {
  readonly element: HTMLElement;
  readonly vizElement?: VizTarget;
  readonly component: Component;
  readonly pagePath: string;
  readonly originalLookup?: DataSetLookup;
  readonly hasExplicitId: boolean;
}
```

- [ ] **Step 6: Update pages-runtime index.ts to re-export DataReceiver**

In `packages/pages-runtime/src/index.ts`, add:

```typescript
export type { DataReceiver } from "@casehubio/pages-component/dist/model/hosting.js";
```

This lets consumers import DataReceiver from either `@casehubio/pages-component` or `@casehubio/pages-runtime`.

- [ ] **Step 7: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- data-pipeline-cleanup.test.ts`
Expected: PASS

- [ ] **Step 8: Run full typecheck**

Run: `yarn typecheck`
Expected: PASS — all packages compile. The `vizElement` type widening and `theme` removal should cause no errors (verified during design review).

- [ ] **Step 9: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: PASS — existing tests still pass with the type changes.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/data-pipeline.ts packages/pages-runtime/src/registry.ts packages/pages-runtime/src/index.ts packages/pages-runtime/src/data-pipeline-cleanup.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor(pages-runtime): VizTarget extends DataReceiver, widen ComponentEntry.vizElement

Remove theme from VizTarget (never set by pipeline). Use entry.element
for MutationObserver cleanup instead of casting vizElement.

Refs #109, #110"
```

---

### Task 3: Activation data bridge for host panels (#110)

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/activation-host-panel.test.ts`

**Interfaces:**
- Consumes: `ConfigurablePanel` and `DataReceiver` from `@casehubio/pages-component`, `VizTarget` from `./data-pipeline.js`, `HostPanelProps` from `@casehubio/pages-component`
- Produces: Updated `host-panel` activation with ConfigurablePanel cast, proxy adapter, and data bridge

- [ ] **Step 1: Write test for ConfigurablePanel cast**

Add to `packages/pages-runtime/src/activation-host-panel.test.ts`:

```typescript
  it("uses ConfigurablePanel interface for configure() call", () => {
    const configured: Record<string, unknown>[] = [];
    customElements.define("test-cfg-iface", class extends HTMLElement {
      configure(props: Record<string, unknown>) {
        configured.push(props);
      }
    });
    registerPanel("cfg-iface", "test-cfg-iface");
    activate({
      type: "host-panel",
      props: { typeName: "cfg-iface", panelProps: { endpoint: "/api" } },
    });
    expect(configured).toEqual([{ endpoint: "/api" }]);
  });
```

- [ ] **Step 2: Write test for data bridge — proxy dispatches pages-data-request**

```typescript
  it("dispatches pages-data-request when lookup is present", () => {
    const received: Array<{ element: unknown; lookup: unknown }> = [];
    customElements.define("test-data-panel", class extends HTMLElement {
      private _data: unknown;
      private _error = "";
      get dataSet() { return this._data; }
      set dataSet(v: unknown) { this._data = v; this._error = ""; }
      get error() { return this._error; }
      set error(v: string) { this._error = v; this._data = undefined; }
      configure(props: Record<string, unknown>) { void props; }
    });
    registerPanel("data-panel", "test-data-panel");

    const container = document.createElement("div");
    container.addEventListener("pages-data-request", ((e: Event) => {
      const detail = (e as CustomEvent).detail;
      received.push({ element: detail.element, lookup: detail.lookup });
    }) as EventListener);
    document.body.appendChild(container);

    const el = document.createElement("div");
    el.dataset.componentId = "data-test";
    el.dataset.componentType = "host-panel";
    container.appendChild(el);

    const registry = new Map();
    const pagePathMap = new Map();
    const callback = createActivationCallback(registry, pagePathMap);
    callback(el, {
      type: "host-panel",
      props: {
        typeName: "data-panel",
        lookup: { dataSetId: "items", operations: [] },
        panelProps: { mode: "list" },
      },
    });

    expect(received).toHaveLength(1);
    expect(received[0].lookup).toEqual({ dataSetId: "items", operations: [] });
    expect(registry.get("data-test")?.vizElement).toBeDefined();
    expect(registry.get("data-test")?.originalLookup).toEqual({ dataSetId: "items", operations: [] });

    document.body.removeChild(container);
  });
```

- [ ] **Step 3: Write test for runtime guard — lookup without DataReceiver**

```typescript
  it("warns and skips data binding when panel lacks DataReceiver properties", () => {
    const warns: string[] = [];
    const origWarn = console.warn;
    console.warn = (...args: unknown[]) => { warns.push(String(args[0])); };

    customElements.define("test-no-data", class extends HTMLElement {
      configure(props: Record<string, unknown>) { void props; }
    });
    registerPanel("no-data", "test-no-data");
    const el = activate({
      type: "host-panel",
      props: {
        typeName: "no-data",
        lookup: { dataSetId: "items", operations: [] },
        panelProps: {},
      },
    });
    expect(warns.some(w => w.includes("lacks DataReceiver"))).toBe(true);
    console.warn = origWarn;
  });
```

- [ ] **Step 4: Write test for proxy forwards dataSet and error**

```typescript
  it("proxy forwards dataSet and error to panel", () => {
    customElements.define("test-proxy-fwd", class extends HTMLElement {
      private _data: unknown;
      private _error = "";
      get dataSet() { return this._data; }
      set dataSet(v: unknown) { this._data = v; this._error = ""; }
      get error() { return this._error; }
      set error(v: string) { this._error = v; this._data = undefined; }
      configure(props: Record<string, unknown>) { void props; }
    });
    registerPanel("proxy-fwd", "test-proxy-fwd");

    const el = document.createElement("div");
    el.dataset.componentId = "proxy-test";
    el.dataset.componentType = "host-panel";
    const registry = new Map();
    const pagePathMap = new Map();

    const container = document.createElement("div");
    container.appendChild(el);
    document.body.appendChild(container);

    const callback = createActivationCallback(registry, pagePathMap);
    callback(el, {
      type: "host-panel",
      props: {
        typeName: "proxy-fwd",
        lookup: { dataSetId: "test", operations: [] },
      },
    });

    const entry = registry.get("proxy-test");
    expect(entry?.vizElement).toBeDefined();

    entry!.vizElement!.dataSet = { columns: [], rows: [] };
    const panel = el.querySelector("test-proxy-fwd") as any;
    expect(panel.dataSet).toEqual({ columns: [], rows: [] });

    entry!.vizElement!.error = "something broke";
    expect(panel.error).toBe("something broke");
    expect(panel.dataSet).toBeUndefined();

    document.body.removeChild(container);
  });
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- activation-host-panel.test.ts`
Expected: FAIL — ConfigurablePanel import missing, proxy not implemented, data bridge not wired

- [ ] **Step 6: Implement the activation changes**

In `packages/pages-runtime/src/activation.ts`:

Add imports at top:
```typescript
import type { ConfigurablePanel, DataReceiver } from "@casehubio/pages-component/dist/model/hosting.js";
import type { HostPanelProps } from "@casehubio/pages-component/dist/model/component-props.js";
import type { VizTarget } from "./data-pipeline.js";
import type { SortColumn } from "@casehubio/pages-data/dist/dataset/sort.js";
```

Add the proxy factory function (before `createActivationCallback`):

```typescript
function createHostPanelProxy(panel: DataReceiver): VizTarget {
  return {
    set dataSet(v: unknown) { panel.dataSet = v; },
    get dataSet() { return panel.dataSet; },
    set error(v: string) { panel.error = v; },
    get error() { return panel.error; },
    set totalRows(_: number) {},
    get totalRows() { return 0; },
    set activeSort(_: SortColumn | undefined) {},
    get activeSort() { return undefined; },
    set activePage(_: number | undefined) {},
    get activePage() { return undefined; },
  };
}
```

Replace the `host-panel` branch (lines 247-273) with:

```typescript
    if (component.type === "host-panel" && component.props) {
      const { typeName, panelProps, lookup } = component.props as HostPanelProps;
      if (!typeName) return;

      const tagName = lookupPanel(typeName);
      if (!tagName) {
        el.textContent = `Unknown panel type: ${typeName}`;
        console.warn(`hostPanel: unregistered type "${typeName}"`);
        return;
      }

      const panel = document.createElement(tagName);

      const configurable = panel as unknown as ConfigurablePanel;
      if (typeof configurable.configure === "function") {
        configurable.configure(panelProps ?? {});
      }

      if (lookup) {
        const panelAsReceiver = panel as unknown as Partial<DataReceiver>;
        if (!("dataSet" in panel)) {
          console.warn(`hostPanel "${typeName}": lookup specified but panel lacks DataReceiver properties`);
          registry.set(componentId, {
            element: el,
            component,
            pagePath,
            hasExplicitId: component.id !== undefined,
          });
        } else {
          const proxy = createHostPanelProxy(panelAsReceiver as DataReceiver);
          registry.set(componentId, {
            element: el,
            vizElement: proxy,
            component,
            pagePath,
            originalLookup: lookup,
            hasExplicitId: component.id !== undefined,
          });
          el.appendChild(panel);
          panel.dispatchEvent(new CustomEvent("pages-data-request", {
            bubbles: true,
            composed: true,
            detail: { element: proxy, lookup },
          }));
          return;
        }
      } else {
        registry.set(componentId, {
          element: el,
          component,
          pagePath,
          hasExplicitId: component.id !== undefined,
        });
      }

      el.appendChild(panel);
      return;
    }
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- activation-host-panel.test.ts`
Expected: PASS

- [ ] **Step 8: Run full runtime test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 9: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/activation-host-panel.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-runtime): dataset pipeline bridge for host panels

Host panels with a lookup in YAML props now receive data through the
standard pages pipeline via a DataReceiver→VizTarget proxy adapter.

Refs #109, #110"
```

---

### Task 4: PagesTerminal migration and protocol update

**Files:**
- Modify: `components/pages-component-terminal/src/PagesTerminal.ts`
- Modify: `components/pages-component-terminal/src/PagesTerminal.test.ts`
- Modify: `docs/protocols/casehub/pages-event-contract.md`

**Interfaces:**
- Consumes: `ConfigurablePanel` from `@casehubio/pages-component`
- Produces: PagesTerminal implementing ConfigurablePanel, updated protocol

- [ ] **Step 1: Write test for PagesTerminal ConfigurablePanel contract**

Add to `components/pages-component-terminal/src/PagesTerminal.test.ts`:

```typescript
import type { ConfigurablePanel } from "@casehubio/pages-component";
import { PagesTerminal, type TerminalProps } from "./PagesTerminal.js";

describe("PagesTerminal ConfigurablePanel contract", () => {
  it("satisfies ConfigurablePanel<TerminalProps> at compile time", () => {
    const terminal = new PagesTerminal();
    const configurable: ConfigurablePanel<TerminalProps> = terminal;
    expect(typeof configurable.configure).toBe("function");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component-terminal run test -- PagesTerminal.test.ts`
Expected: FAIL — PagesTerminal doesn't implement ConfigurablePanel yet (assignment may fail at type level)

- [ ] **Step 3: Add implements clause to PagesTerminal**

In `components/pages-component-terminal/src/PagesTerminal.ts`, add the import and implements:

Add import:
```typescript
import type { ConfigurablePanel } from "@casehubio/pages-component";
```

Change class declaration:
```typescript
export class PagesTerminal extends HTMLElement implements ConfigurablePanel<TerminalProps> {
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-component-terminal run test -- PagesTerminal.test.ts`
Expected: PASS

- [ ] **Step 5: Update pages-event-contract.md**

In `docs/protocols/casehub/pages-event-contract.md`, update the `pages-data-request` row in the reserved events table:

Old:
```
| `pages-data-request` | Component requests dataset resolution | `PagesElement` base class (`connectedCallback`) |
```

New:
```
| `pages-data-request` | Component requests dataset resolution | `PagesElement` base class (`connectedCallback`) and runtime activation layer (host panel data binding) |
```

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `yarn workspace @casehubio/pages-component-terminal run test`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add components/pages-component-terminal/src/PagesTerminal.ts components/pages-component-terminal/src/PagesTerminal.test.ts docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: PagesTerminal implements ConfigurablePanel, update event protocol

Migrate the only in-repo configure() consumer to the formal interface.
Update pages-data-request protocol with the new activation dispatch site.

Refs #109, #110"
```
