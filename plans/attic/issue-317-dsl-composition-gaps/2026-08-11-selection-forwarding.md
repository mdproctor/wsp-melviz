# Selection Forwarding via Host-Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #300 — feat: custom component selection forwarding via host-panel
**Issue group:** #300

**Goal:** Dispatch `pages-selection-changed` CustomEvent on host-panel elements when RuntimeContext.selection changes for their declared `selectionSource`, so custom Web Components can react to table row selection without coupling to the internal RuntimeContext API.

**Architecture:** Add `selectionSource?: string` to `HostPanelProps`. In `site.ts`, after `contextManager.updateSelection()` is called (in the selection-change handler, data refresh path, and page navigation clear), scan the ComponentRegistry for host-panel entries with matching `selectionSource` and dispatch `pages-selection-changed` on each. No ContextManager changes — all dispatch logic lives in site.ts alongside the existing framework event handlers.

**Tech Stack:** TypeScript 5, Vitest, Yarn workspaces

## Global Constraints

- `HostPanelProps` lives in `@casehubio/pages-component`; dispatch logic lives in `@casehubio/pages-runtime`
- Build order: `packages/pages-component` must build before `packages/pages-runtime`
- `pages-selection-changed` is a framework event (like `pages-filter`, `pages-sort`) — NOT a `pages-event` topic
- Event detail shape: `{ sourceDatasetId: string, selectedRow: Record<string, unknown> | null }`
- No event on initial mount when no selection exists

---

### Task 1: Add `selectionSource` to `HostPanelProps`, dispatch in site.ts, update protocol

**Files:**
- Modify: `packages/pages-component/src/model/component-props.ts:46-50` — add `selectionSource` field
- Modify: `packages/pages-runtime/src/site.ts:673-690` — dispatch after updateSelection in selection-change handler
- Modify: `packages/pages-runtime/src/site.ts:183` — dispatch after updateSelection(null) on data refresh
- Modify: `packages/pages-runtime/src/site.ts:353` — dispatch after clearAllSelections on page nav
- Modify: `docs/protocols/casehub/pages-event-contract.md` — add `pages-selection-changed` to reserved events
- Test: `packages/pages-runtime/src/selection-forwarding.test.ts`

**Interfaces:**
- Consumes: `HostPanelProps` from `@casehubio/pages-component` (line 8 of site.ts — already imported)
- Consumes: `ComponentRegistry` from `./registry.js` (already available in site.ts as `registry`)
- Produces: `dispatchSelectionToHostPanels(registry: ComponentRegistry, sourceDatasetId: string, selectedRow: Record<string, unknown> | null): void`

- [ ] **Step 1: Write failing tests for `dispatchSelectionToHostPanels()`**

Create `packages/pages-runtime/src/selection-forwarding.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import type { ComponentRegistry, ComponentEntry } from "./registry.js";
import { dispatchSelectionToHostPanels, dispatchSelectionClearAll } from "./selection-forwarding.js";

function makeHostPanelEntry(selectionSource?: string): ComponentEntry {
  const element = document.createElement("div");
  return {
    element,
    component: {
      type: "host-panel",
      props: {
        typeName: "test-panel",
        ...(selectionSource !== undefined && { selectionSource }),
      },
    },
    pagePath: "",
    hasExplicitId: false,
  };
}

function makeChartEntry(): ComponentEntry {
  return {
    element: document.createElement("div"),
    component: { type: "bar-chart", props: { lookup: { dataSetId: "ds", operations: [] } } },
    pagePath: "",
    hasExplicitId: false,
  };
}

describe("dispatchSelectionToHostPanels", () => {
  it("dispatches to host-panel with matching selectionSource", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeHostPanelEntry("adverse-events");
    registry.set("panel-1", entry);

    const received: CustomEvent[] = [];
    entry.element.addEventListener("pages-selection-changed", ((e: Event) => {
      received.push(e as CustomEvent);
    }));

    const row = { id: 42, name: "Test" };
    dispatchSelectionToHostPanels(registry, "adverse-events", row);

    expect(received).toHaveLength(1);
    expect(received[0]!.detail).toEqual({
      sourceDatasetId: "adverse-events",
      selectedRow: { id: 42, name: "Test" },
    });
  });

  it("does not dispatch to host-panel with different selectionSource", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeHostPanelEntry("other-dataset");
    registry.set("panel-1", entry);

    const received: Event[] = [];
    entry.element.addEventListener("pages-selection-changed", (e) => received.push(e));

    dispatchSelectionToHostPanels(registry, "adverse-events", { id: 1 });

    expect(received).toHaveLength(0);
  });

  it("does not dispatch to host-panel without selectionSource", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeHostPanelEntry();
    registry.set("panel-1", entry);

    const received: Event[] = [];
    entry.element.addEventListener("pages-selection-changed", (e) => received.push(e));

    dispatchSelectionToHostPanels(registry, "adverse-events", { id: 1 });

    expect(received).toHaveLength(0);
  });

  it("does not dispatch to non-host-panel components", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeChartEntry();
    registry.set("chart-1", entry);

    const received: Event[] = [];
    entry.element.addEventListener("pages-selection-changed", (e) => received.push(e));

    dispatchSelectionToHostPanels(registry, "ds", { id: 1 });

    expect(received).toHaveLength(0);
  });

  it("dispatches null selectedRow for deselection", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeHostPanelEntry("events");
    registry.set("panel-1", entry);

    const received: CustomEvent[] = [];
    entry.element.addEventListener("pages-selection-changed", ((e: Event) => {
      received.push(e as CustomEvent);
    }));

    dispatchSelectionToHostPanels(registry, "events", null);

    expect(received).toHaveLength(1);
    expect(received[0]!.detail.selectedRow).toBeNull();
  });

  it("dispatches to multiple panels with same selectionSource", () => {
    const registry: ComponentRegistry = new Map();
    const entry1 = makeHostPanelEntry("events");
    const entry2 = makeHostPanelEntry("events");
    registry.set("panel-1", entry1);
    registry.set("panel-2", entry2);

    const received1: Event[] = [];
    const received2: Event[] = [];
    entry1.element.addEventListener("pages-selection-changed", (e) => received1.push(e));
    entry2.element.addEventListener("pages-selection-changed", (e) => received2.push(e));

    dispatchSelectionToHostPanels(registry, "events", { id: 1 });

    expect(received1).toHaveLength(1);
    expect(received2).toHaveLength(1);
  });

  it("guards against re-entrant dispatch", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeHostPanelEntry("events");
    registry.set("panel-1", entry);

    const received: Event[] = [];
    entry.element.addEventListener("pages-selection-changed", () => {
      received.push(new Event("marker"));
      dispatchSelectionToHostPanels(registry, "events", { id: 99 });
    });

    dispatchSelectionToHostPanels(registry, "events", { id: 1 });

    expect(received).toHaveLength(1);
  });

  it("dispatches with bubbles and composed", () => {
    const registry: ComponentRegistry = new Map();
    const entry = makeHostPanelEntry("events");
    registry.set("panel-1", entry);

    const received: CustomEvent[] = [];
    entry.element.addEventListener("pages-selection-changed", ((e: Event) => {
      received.push(e as CustomEvent);
    }));

    dispatchSelectionToHostPanels(registry, "events", { id: 1 });

    expect(received[0]!.bubbles).toBe(true);
    expect(received[0]!.composed).toBe(true);
  });
});

describe("dispatchSelectionClearAll", () => {
  it("dispatches null to all host-panels with any selectionSource", () => {
    const registry: ComponentRegistry = new Map();
    const entry1 = makeHostPanelEntry("events");
    const entry2 = makeHostPanelEntry("orders");
    const entry3 = makeHostPanelEntry(); // no selectionSource
    registry.set("panel-1", entry1);
    registry.set("panel-2", entry2);
    registry.set("panel-3", entry3);

    const received1: CustomEvent[] = [];
    const received2: CustomEvent[] = [];
    const received3: CustomEvent[] = [];
    entry1.element.addEventListener("pages-selection-changed", ((e: Event) => received1.push(e as CustomEvent)));
    entry2.element.addEventListener("pages-selection-changed", ((e: Event) => received2.push(e as CustomEvent)));
    entry3.element.addEventListener("pages-selection-changed", ((e: Event) => received3.push(e as CustomEvent)));

    dispatchSelectionClearAll(registry);

    expect(received1).toHaveLength(1);
    expect(received1[0]!.detail).toEqual({ sourceDatasetId: "events", selectedRow: null });
    expect(received2).toHaveLength(1);
    expect(received2[0]!.detail).toEqual({ sourceDatasetId: "orders", selectedRow: null });
    expect(received3).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/102/pages workspace @casehubio/pages-runtime run test -- --reporter verbose src/selection-forwarding.test.ts 2>&1 | tail -15`
Expected: FAIL — `selection-forwarding.js` module not found

- [ ] **Step 3: Add `selectionSource` to `HostPanelProps`**

In `packages/pages-component/src/model/component-props.ts`, add `selectionSource` as the last field in `HostPanelProps`:

```typescript
export interface HostPanelProps {
  readonly typeName: string;
  readonly panelProps?: Readonly<Record<string, unknown>>;
  readonly lookup?: DataSetLookup;
  readonly selectionSource?: string;
}
```

- [ ] **Step 4: Implement `dispatchSelectionToHostPanels` and `dispatchSelectionClearAll`**

Create `packages/pages-runtime/src/selection-forwarding.ts`:

```typescript
import type { ComponentRegistry } from "./registry.js";
import type { HostPanelProps } from "@casehubio/pages-component";

let dispatching = false;

export function dispatchSelectionToHostPanels(
  registry: ComponentRegistry,
  sourceDatasetId: string,
  selectedRow: Record<string, unknown> | null,
): void {
  if (dispatching) return;
  dispatching = true;
  try {
    for (const [, entry] of registry) {
      if (entry.component.type !== "host-panel") continue;
      const props = entry.component.props as HostPanelProps | undefined;
      if (props?.selectionSource !== sourceDatasetId) continue;
      entry.element.dispatchEvent(new CustomEvent("pages-selection-changed", {
        bubbles: true,
        composed: true,
        detail: { sourceDatasetId, selectedRow },
      }));
    }
  } finally {
    dispatching = false;
  }
}

export function dispatchSelectionClearAll(
  registry: ComponentRegistry,
): void {
  if (dispatching) return;
  dispatching = true;
  try {
    for (const [, entry] of registry) {
      if (entry.component.type !== "host-panel") continue;
      const props = entry.component.props as HostPanelProps | undefined;
      if (!props?.selectionSource) continue;
      entry.element.dispatchEvent(new CustomEvent("pages-selection-changed", {
        bubbles: true,
        composed: true,
        detail: { sourceDatasetId: props.selectionSource, selectedRow: null },
      }));
    }
  } finally {
    dispatching = false;
  }
}
```

- [ ] **Step 5: Wire dispatch into site.ts**

Add import at the top of `packages/pages-runtime/src/site.ts` (after line 37, the `typedRowToRecord` import):

```typescript
import { dispatchSelectionToHostPanels, dispatchSelectionClearAll } from "./selection-forwarding.js";
```

**Call site 1 — selection-change handler (~line 686-688):** After `contextManager.updateSelection`, dispatch to host-panels:

```typescript
    if (detail.selectedRows.length > 0) {
      const row = detail.selectedRows[0]!;
      const ds = entry.vizElement?.dataSet as TypedDataSet | undefined;
      if (!ds) return;
      const rowData = typedRowToRecord(row, ds.columns);
      contextManager.updateSelection(datasetId, rowData);
      dispatchSelectionToHostPanels(registry, datasetId, rowData);
    } else {
      contextManager.updateSelection(datasetId, null);
      dispatchSelectionToHostPanels(registry, datasetId, null);
    }
```

**Call site 2 — data refresh clear (~line 183):** After `contextManager.updateSelection(id, null)`:

```typescript
    onChanged: (id, dataset) => {
      contextManager.updateDataset(id, dataset);
      contextManager.updateSelection(id as string, null);
      dispatchSelectionToHostPanels(registry, id as string, null);
      pipeline.deliverDataSet(id);
    },
```

**Call site 3 — page navigation (~line 353):** After `contextManager.clearAllSelections()`:

```typescript
    contextManager.clearAllSelections();
    dispatchSelectionClearAll(registry);
```

- [ ] **Step 6: Update event contract protocol**

In `docs/protocols/casehub/pages-event-contract.md`, add to the reserved events table after `selection-change`:

```markdown
| `pages-selection-changed` | Selection context forwarded to host-panel | Runtime selection bridge (site.ts) |
```

- [ ] **Step 7: Build pages-component then pages-runtime**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/102/pages workspace @casehubio/pages-component run build 2>&1 | tail -5`
Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/102/pages workspace @casehubio/pages-runtime run build 2>&1 | tail -5`
Expected: Both build (pre-existing floating workspace type errors may appear — unrelated).

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/102/pages workspace @casehubio/pages-runtime run test -- --reporter verbose src/selection-forwarding.test.ts 2>&1 | tail -20`
Expected: All 8 tests PASS.

- [ ] **Step 9: Run full runtime test suite for regressions**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/102/pages workspace @casehubio/pages-runtime run test -- --reporter verbose 2>&1 | tail -10`
Expected: All existing tests still PASS.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/102/pages add packages/pages-component/src/model/component-props.ts packages/pages-runtime/src/selection-forwarding.ts packages/pages-runtime/src/selection-forwarding.test.ts packages/pages-runtime/src/site.ts docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/casehub/slots/102/pages commit -m "feat(#300): selection forwarding to host-panel via pages-selection-changed

Add selectionSource to HostPanelProps. Dispatch pages-selection-changed
on matching host-panel elements when RuntimeContext.selection changes.
Covers selection, deselection, data refresh clear, and page nav clear.

Closes #300"
```
