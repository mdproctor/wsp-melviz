# Runtime Improvements (#17) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement four batched runtime improvements from code review #13: tree-walk navigation, URL encoding, lazy-page activation, and accordion initial state.

**Architecture:** Four independent improvements to the melviz TypeScript runtime. M5 (accordion) is a one-line fix. M3 (URL encoding) is self-contained in `url.ts`. M2 (tree-walk) adds `walkNavigate` to `navigation.ts` and rewires `site.ts`. M4 (lazy-page) is the largest — extends the activation callback with fetch/parse/render/tree-integration for `lazy-page` components.

**Tech Stack:** TypeScript 4.6, Vitest, js-yaml, @casehub/component, @casehub/runtime

## Global Constraints

- Test runner: `yarn workspace @casehub/runtime run test` or `yarn workspace @casehub/component run test`
- Run specific test: `yarn workspace @casehub/runtime run test -- navigation.test`
- Build before test if types change: `yarn build:packages`
- All commits reference issue: `Refs #17`
- No browser/visual verification — unit tests only
- `Component` interface has all `readonly` properties — do not mutate existing tree nodes

---

### Task 1: M5 — Accordion explicit initial state

**Files:**
- Modify: `packages/casehub-component/src/renderer/interactive.ts:272-288`
- Test: `packages/casehub-component/src/renderer/interactive.test.ts`

**Interfaces:**
- Consumes: existing `wireAccordion` internal function
- Produces: no interface change — behavior identical, contract explicit

- [ ] **Step 1: Verify existing tests pass**

Run: `yarn workspace @casehub/component run test -- interactive.test`
Expected: all accordion tests PASS

- [ ] **Step 2: Add explicit initial display in wireAccordion**

In `packages/casehub-component/src/renderer/interactive.ts`, in the `wireAccordion` function, add `panel.style.display = ""` immediately after the `if (panel)` guard:

```typescript
function wireAccordion(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  doc: Document,
): void {
  slotNames.forEach((name) => {
    const panel = panels.get(name);
    if (panel) {
      panel.style.display = "";
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
```

- [ ] **Step 3: Run tests to verify no regression**

Run: `yarn workspace @casehub/component run test -- interactive.test`
Expected: all tests PASS (behavior unchanged, contract now explicit)

- [ ] **Step 4: Commit**

```
git add packages/casehub-component/src/renderer/interactive.ts
git commit -m "fix: explicit accordion initial display state  Refs #17"
```

---

### Task 2: M3 — URL encoding for page path segments

**Files:**
- Modify: `packages/casehub-runtime/src/url.ts`
- Modify: `packages/casehub-runtime/src/url.test.ts`

**Interfaces:**
- Consumes: `DeepLink` type from `@casehub/ui/dist/model/page-types.js`
- Produces: `serializeToUrl(link: DeepLink): string` and `parseFromUrl(hash: string): DeepLink` — same signatures, encoded page path segments

- [ ] **Step 1: Write failing tests for special character page names**

Add to `packages/casehub-runtime/src/url.test.ts`:

```typescript
describe("serializeToUrl — encoding", () => {
  it("encodes spaces in page name", () => {
    const link: DeepLink = { page: "Q1 Report" };
    expect(serializeToUrl(link)).toBe("#/page/Q1%20Report");
  });

  it("encodes special characters in nested page path", () => {
    const link: DeepLink = { page: "R&D/Q1 Report" };
    expect(serializeToUrl(link)).toBe("#/page/R%26D/Q1%20Report");
  });

  it("encodes hash in page name", () => {
    const link: DeepLink = { page: "Section#2" };
    expect(serializeToUrl(link)).toBe("#/page/Section%232");
  });

  it("encodes question mark in page name", () => {
    const link: DeepLink = { page: "FAQ?" };
    expect(serializeToUrl(link)).toBe("#/page/FAQ%3F");
  });
});

describe("parseFromUrl — decoding", () => {
  it("decodes encoded page segments", () => {
    const link = parseFromUrl("#/page/Q1%20Report");
    expect(link.page).toBe("Q1 Report");
  });

  it("decodes nested encoded page path", () => {
    const link = parseFromUrl("#/page/R%26D/Q1%20Report");
    expect(link.page).toBe("R&D/Q1 Report");
  });
});

describe("round-trip — encoding", () => {
  it("round-trips page names with special characters", () => {
    const original: DeepLink = {
      page: "R&D/Q1 Report",
      filters: { "col name": ["val?1", "val#2"] },
    };
    const url = serializeToUrl(original);
    const parsed = parseFromUrl(url);
    expect(parsed.page).toBe(original.page);
    expect(parsed.filters).toEqual(original.filters);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- url.test`
Expected: new encoding tests FAIL (page names not encoded), existing tests PASS

- [ ] **Step 3: Implement encoding in serializeToUrl and decoding in parseFromUrl**

Replace the contents of `packages/casehub-runtime/src/url.ts`:

```typescript
import type { DeepLink } from "@casehub/ui/dist/model/page-types.js";

function encodePagePath(page: string): string {
  return page
    .split("/")
    .filter(Boolean)
    .map(encodeURIComponent)
    .join("/");
}

function decodePagePath(encoded: string): string {
  return encoded
    .split("/")
    .filter(Boolean)
    .map(decodeURIComponent)
    .join("/");
}

export function serializeToUrl(link: DeepLink): string {
  let url = `#/page/${encodePagePath(link.page)}`;

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
  const rawPage = qIndex === -1 ? withoutPrefix : withoutPrefix.substring(0, qIndex);
  const page = decodePagePath(rawPage);

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

- [ ] **Step 4: Update existing serialize test expectation**

The `"root page (empty path)"` test currently expects `"#/page/"`. With the new `encodePagePath`, an empty string produces `""` after split/filter/join, so the URL becomes `"#/page/"`. Verify this still passes — `"".split("/").filter(Boolean)` returns `[]`, `.join("/")` returns `""`, so `#/page/` + `""` = `#/page/`. No change needed.

- [ ] **Step 5: Run all URL tests**

Run: `yarn workspace @casehub/runtime run test -- url.test`
Expected: ALL tests PASS (existing + new)

- [ ] **Step 6: Commit**

```
git add packages/casehub-runtime/src/url.ts packages/casehub-runtime/src/url.test.ts
git commit -m "fix: encode/decode page path segments in URL serialization  Refs #17"
```

---

### Task 3: M2 — walkNavigate tree-walk for navigate()

**Files:**
- Modify: `packages/casehub-runtime/src/navigation.ts`
- Modify: `packages/casehub-runtime/src/navigation.test.ts`
- Modify: `packages/casehub-runtime/src/site.ts`

**Interfaces:**
- Consumes: `INTERACTIVE_TYPES` (internal to navigation.ts), `activateSlot` from `@casehub/component`, `Component` type
- Produces: `walkNavigate(root: Component, segments: string[], target: HTMLElement, lazyPageResolutions: Map<Component, Component>): string`

- [ ] **Step 1: Write failing tests for walkNavigate**

Add to `packages/casehub-runtime/src/navigation.test.ts`:

```typescript
import { walkNavigate } from "./navigation.js";
import { activateSlot } from "@casehub/component/dist/renderer/activate-slot.js";
import { wireInteractivity } from "@casehub/component/dist/renderer/interactive.js";

function renderInteractive(
  target: HTMLElement,
  component: Component,
): HTMLElement {
  const el = document.createElement("div");
  el.dataset.componentType = component.type;
  el.dataset.componentId = component.id!;
  target.appendChild(el);

  if (component.slots) {
    const slotNames = Object.keys(component.slots);
    const panels = new Map<string, HTMLElement>();
    for (const name of slotNames) {
      const panel = document.createElement("div");
      panel.dataset.slot = name;
      el.appendChild(panel);
      panels.set(name, panel);
    }
    wireInteractivity(el, component.type, slotNames, panels);
  }
  return el;
}

describe("walkNavigate", () => {
  it("activates single-level path", () => {
    const sales: Component = { type: "page", props: { name: "Sales" } };
    const overview: Component = { type: "page", props: { name: "Overview" } };
    const nav: Component = {
      type: "sidebar",
      id: "nav-1",
      slots: { Overview: [overview], Sales: [sales] },
    };
    const root: Component = {
      type: "page",
      slots: { default: [nav] },
    };

    const target = document.createElement("div");
    renderInteractive(target, nav);

    const result = walkNavigate(root, ["Sales"], target, new Map());
    expect(result).toBe("Sales");
  });

  it("activates multi-level nested path", () => {
    const revenue: Component = { type: "page", props: { name: "Revenue" } };
    const costs: Component = { type: "page", props: { name: "Costs" } };
    const tabs: Component = {
      type: "tabs",
      id: "tabs-1",
      slots: { Revenue: [revenue], Costs: [costs] },
    };
    const sales: Component = {
      type: "page",
      props: { name: "Sales" },
      slots: { default: [tabs] },
    };
    const overview: Component = { type: "page", props: { name: "Overview" } };
    const nav: Component = {
      type: "sidebar",
      id: "nav-1",
      slots: { Overview: [overview], Sales: [sales] },
    };
    const root: Component = {
      type: "page",
      slots: { default: [nav] },
    };

    const target = document.createElement("div");
    const navEl = renderInteractive(target, nav);
    // Render nested tabs inside the Sales panel
    const salesPanel = navEl.querySelector("[data-slot='Sales']")!;
    renderInteractive(salesPanel as HTMLElement, tabs);

    const result = walkNavigate(root, ["Sales", "Revenue"], target, new Map());
    expect(result).toBe("Sales/Revenue");
  });

  it("returns partial path on missing segment", () => {
    const sales: Component = { type: "page", props: { name: "Sales" } };
    const nav: Component = {
      type: "sidebar",
      id: "nav-1",
      slots: { Sales: [sales] },
    };
    const root: Component = {
      type: "page",
      slots: { default: [nav] },
    };

    const target = document.createElement("div");
    renderInteractive(target, nav);

    const result = walkNavigate(root, ["Sales", "Nonexistent"], target, new Map());
    expect(result).toBe("Sales");
  });

  it("returns empty string when first segment has no match", () => {
    const sales: Component = { type: "page", props: { name: "Sales" } };
    const nav: Component = {
      type: "sidebar",
      id: "nav-1",
      slots: { Sales: [sales] },
    };
    const root: Component = {
      type: "page",
      slots: { default: [nav] },
    };

    const target = document.createElement("div");
    renderInteractive(target, nav);

    const result = walkNavigate(root, ["Missing"], target, new Map());
    expect(result).toBe("");
  });

  it("works with tree, menu, and tiles types", () => {
    for (const type of ["tree", "menu", "tiles"]) {
      const page: Component = { type: "page", props: { name: "Child" } };
      const container: Component = {
        type,
        id: `${type}-1`,
        slots: { Child: [page] },
      };
      const root: Component = {
        type: "page",
        slots: { default: [container] },
      };

      const target = document.createElement("div");
      renderInteractive(target, container);

      const result = walkNavigate(root, ["Child"], target, new Map());
      expect(result).toBe("Child");
    }
  });

  it("follows lazyPageResolutions overlay", () => {
    const innerPage: Component = { type: "page", props: { name: "Detail" } };
    const innerTabs: Component = {
      type: "tabs",
      id: "inner-tabs",
      slots: { Detail: [innerPage] },
    };
    const resolvedRoot: Component = {
      type: "page",
      slots: { default: [innerTabs] },
    };
    const lazyPage: Component = {
      type: "lazy-page",
      props: { name: "LazySection", href: "/lazy.yaml" },
    };
    const nav: Component = {
      type: "sidebar",
      id: "nav-1",
      slots: { LazySection: [lazyPage] },
    };
    const root: Component = {
      type: "page",
      slots: { default: [nav] },
    };

    const lazyResolutions = new Map<Component, Component>();
    lazyResolutions.set(lazyPage, resolvedRoot);

    const target = document.createElement("div");
    const navEl = renderInteractive(target, nav);
    // Simulate resolved lazy-page content rendered inside the LazySection panel
    const lazyPanel = navEl.querySelector("[data-slot='LazySection']")!;
    renderInteractive(lazyPanel as HTMLElement, innerTabs);

    const result = walkNavigate(root, ["LazySection", "Detail"], target, lazyResolutions);
    expect(result).toBe("LazySection/Detail");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- navigation.test`
Expected: FAIL — `walkNavigate` is not exported

- [ ] **Step 3: Implement walkNavigate**

Add to `packages/casehub-runtime/src/navigation.ts`, after the existing imports:

```typescript
import { activateSlot } from "@casehub/component/dist/renderer/activate-slot.js";
```

Add the `walkNavigate` function after `computeCurrentPage`:

```typescript
export function walkNavigate(
  root: Component,
  segments: string[],
  target: HTMLElement,
  lazyPageResolutions: Map<Component, Component>,
): string {
  const reached: string[] = [];
  let currentNodes: readonly Component[] = root.slots
    ? Object.values(root.slots).flat()
    : [];

  for (const segment of segments) {
    const container = findInteractiveWithSlot(currentNodes, segment, lazyPageResolutions);
    if (!container) break;

    const domEl = target.querySelector<HTMLElement>(
      `[data-component-id="${container.id}"]`,
    );
    if (!domEl || !activateSlot(domEl, segment)) break;

    reached.push(segment);

    const slotChildren = container.slots![segment]!;
    currentNodes = descendIntoChildren(slotChildren, lazyPageResolutions);
  }

  return reached.join("/");
}

function findInteractiveWithSlot(
  nodes: readonly Component[],
  slotName: string,
  lazyResolutions: Map<Component, Component>,
): Component | undefined {
  for (const node of nodes) {
    if (INTERACTIVE_TYPES.has(node.type) && node.id && node.slots?.[slotName]) {
      return node;
    }

    const resolved = node.type === "lazy-page" ? lazyResolutions.get(node) : undefined;
    const children = resolved
      ? (resolved.slots ? Object.values(resolved.slots).flat() : [])
      : [
          ...(node.slots ? Object.values(node.slots).flat() : []),
          ...(node.items ? node.items.map((i) => i.component) : []),
        ];

    if (children.length > 0) {
      const found = findInteractiveWithSlot(children, slotName, lazyResolutions);
      if (found) return found;
    }
  }
  return undefined;
}

function descendIntoChildren(
  slotChildren: readonly Component[],
  lazyResolutions: Map<Component, Component>,
): readonly Component[] {
  const result: Component[] = [];
  for (const child of slotChildren) {
    const resolved = child.type === "lazy-page" ? lazyResolutions.get(child) : undefined;
    if (resolved) {
      result.push(...(resolved.slots ? Object.values(resolved.slots).flat() : []));
    } else {
      result.push(child);
    }
  }
  return result;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehub/runtime run test -- navigation.test`
Expected: ALL tests PASS

- [ ] **Step 5: Rewire site.ts navigate() to use walkNavigate**

In `packages/casehub-runtime/src/site.ts`, add import:

```typescript
import { buildPageIndex, computeCurrentPage, walkNavigate } from "./navigation.js";
```

Replace the `navigate` method body (lines 231-259):

```typescript
    navigate(path: string): void {
      _navigating = true;
      const segments = path.split("/").filter(Boolean);
      currentPage = walkNavigate(root, segments, target, lazyPageResolutions);
      _navigating = false;

      if (typeof history !== "undefined") {
        const filters = deriveActiveFilters(filterState, currentPage);
        const hasFilters = Object.keys(filters).length > 0;
        const link: DeepLink = { page: currentPage, ...(hasFilters ? { filters } : {}) };
        history.pushState(null, "", serializeToUrl(link));
      }
    },
```

Add `lazyPageResolutions` map creation after the other maps near the top of `loadSite` (after line 52):

```typescript
  const lazyPageResolutions: Map<Component, Component> = new Map();
```

- [ ] **Step 6: Run full runtime test suite**

Run: `yarn workspace @casehub/runtime run test`
Expected: ALL tests PASS

- [ ] **Step 7: Commit**

```
git add packages/casehub-runtime/src/navigation.ts packages/casehub-runtime/src/navigation.test.ts packages/casehub-runtime/src/site.ts
git commit -m "feat: tree-walk navigate() with all interactive types  Refs #17"
```

---

### Task 4: M4 — Lazy page activation — incremental extension APIs

**Files:**
- Modify: `packages/casehub-runtime/src/page-paths.ts`
- Modify: `packages/casehub-runtime/src/page-paths.test.ts`
- Modify: `packages/casehub-runtime/src/dataset-scope.ts`
- Modify: `packages/casehub-runtime/src/dataset-scope.test.ts`
- Modify: `packages/casehub-runtime/src/navigation.ts`

**Interfaces:**
- Consumes: existing `PagePathMap`, `DataSetScope`, `PageIndex` types
- Produces:
  - `extendPagePathMap(root: Component, basePath: string, map: PagePathMap): void`
  - `extendDataSetScope(root: Component, inheritedScope: Map<DataSetId, ExternalDataSetDef>, paths: PagePathMap, scope: DataSetScope): void`
  - `extendPageIndex(root: Component, paths: PagePathMap, index: PageIndex): void`

- [ ] **Step 1: Write failing test for extendPagePathMap**

Add to `packages/casehub-runtime/src/page-paths.test.ts`:

```typescript
import { buildPagePathMap, extendPagePathMap } from "./page-paths.js";

describe("extendPagePathMap", () => {
  it("extends existing map with subtree rooted at basePath", () => {
    const existingRoot: Component = { type: "page", props: { name: "App" } };
    const map = buildPagePathMap(existingRoot);
    expect(map.get(existingRoot)).toBe("");

    const detail: Component = { type: "page", props: { name: "Detail" } };
    const fetchedRoot: Component = {
      type: "page",
      slots: { Detail: [detail] },
    };

    extendPagePathMap(fetchedRoot, "Sales", map);

    expect(map.get(fetchedRoot)).toBe("Sales");
    expect(map.get(detail)).toBe("Sales/Detail");
  });

  it("handles nested subtree with non-page components", () => {
    const map: PagePathMap = new Map();
    const chart: Component = { type: "bar-chart" };
    const page: Component = {
      type: "page",
      props: { name: "Stats" },
      slots: { default: [chart] },
    };
    const fetchedRoot: Component = {
      type: "page",
      slots: { Stats: [page] },
    };

    extendPagePathMap(fetchedRoot, "Parent", map);

    expect(map.get(fetchedRoot)).toBe("Parent");
    expect(map.get(page)).toBe("Parent/Stats");
    expect(map.get(chart)).toBe("Parent/Stats");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- page-paths.test`
Expected: FAIL — `extendPagePathMap` not exported

- [ ] **Step 3: Implement extendPagePathMap**

In `packages/casehub-runtime/src/page-paths.ts`, export the new function. The existing `walk` function already does the right thing — `extendPagePathMap` is a thin wrapper that calls it with a different starting path and writes into an existing map:

```typescript
export function extendPagePathMap(
  root: Component,
  basePath: string,
  map: PagePathMap,
): void {
  walk(root, basePath, undefined, map);
}
```

Note: `walk` is already a private function that takes `(component, currentPath, slotName, map)`. When called with `slotName = undefined` and `component.type === "page"`, it won't push a new segment — which is correct because `basePath` already includes the lazy-page's slot name. The root of the fetched content is a page that inherits `basePath` as its path.

Wait — re-examining `walk`: when `component.type === "page" && slotName !== undefined`, it pushes a new segment. When `slotName === undefined`, it doesn't push — so `fetchedRoot` gets `path = basePath`. But `fetchedRoot` IS a page — it should get `basePath` as its path, not push an additional segment. This is correct because the lazy-page's slot name in the parent container already determined the path segment.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/runtime run test -- page-paths.test`
Expected: ALL PASS

- [ ] **Step 5: Write failing test for extendPageIndex**

Add to `packages/casehub-runtime/src/navigation.test.ts`:

```typescript
import { buildPageIndex, computeCurrentPage, walkNavigate, extendPageIndex } from "./navigation.js";

describe("extendPageIndex", () => {
  it("extends existing index with subtree pages", () => {
    const existingRoot: Component = { type: "page", props: { name: "App" } };
    const existingPaths = buildPagePathMap(existingRoot);
    const index = buildPageIndex(existingRoot, existingPaths);
    expect(index.get("")).toBe(existingRoot);

    const detail: Component = { type: "page", props: { name: "Detail" } };
    const fetchedRoot: Component = {
      type: "page",
      slots: { Detail: [detail] },
    };

    const newPaths: PagePathMap = new Map();
    extendPagePathMap(fetchedRoot, "Sales", newPaths);
    extendPageIndex(fetchedRoot, newPaths, index);

    expect(index.get("Sales")).toBe(fetchedRoot);
    expect(index.get("Sales/Detail")).toBe(detail);
    expect(index.get("")).toBe(existingRoot);
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehub/runtime run test -- navigation.test`
Expected: FAIL — `extendPageIndex` not exported

- [ ] **Step 7: Implement extendPageIndex**

In `packages/casehub-runtime/src/navigation.ts`, the existing `walkPages` private function already does the right thing. Export a thin wrapper:

```typescript
export function extendPageIndex(
  root: Component,
  paths: PagePathMap,
  index: PageIndex,
): void {
  walkPages(root, paths, index);
}
```

- [ ] **Step 8: Run tests**

Run: `yarn workspace @casehub/runtime run test -- navigation.test`
Expected: ALL PASS

- [ ] **Step 9: Write failing test for extendDataSetScope**

Add to `packages/casehub-runtime/src/dataset-scope.test.ts`:

```typescript
import { buildDataSetScope, resolveDataSetDef, extendDataSetScope } from "./dataset-scope.js";
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import type { ExternalDataSetDef } from "@casehub/data/dist/dataset/external/types.js";

describe("extendDataSetScope", () => {
  it("extends scope with fetched subtree, inheriting parent datasets", () => {
    const parentDs: ExternalDataSetDef = {
      uuid: "parent-ds" as DataSetId,
      url: "http://example.com/data",
      columns: [],
    };
    const parentPage: Component = {
      type: "page",
      props: { name: "Sales", datasets: [parentDs] },
    };
    const root: Component = {
      type: "page",
      slots: { Sales: [parentPage] },
    };
    const paths = buildPagePathMap(root);
    const scope = buildDataSetScope(root, paths);

    const childDs: ExternalDataSetDef = {
      uuid: "child-ds" as DataSetId,
      url: "http://example.com/child",
      columns: [],
    };
    const childPage: Component = {
      type: "page",
      props: { name: "Detail", datasets: [childDs] },
    };
    const fetchedRoot: Component = {
      type: "page",
      slots: { Detail: [childPage] },
    };

    const newPaths: PagePathMap = new Map();
    extendPagePathMap(fetchedRoot, "Sales", newPaths);

    const inherited = scope.get("Sales") ?? new Map();
    extendDataSetScope(fetchedRoot, inherited, newPaths, scope);

    // Child page inherits parent dataset AND has its own
    const detailScope = scope.get("Sales/Detail");
    expect(detailScope).toBeTruthy();
    expect(detailScope!.get("parent-ds" as DataSetId)).toBe(parentDs);
    expect(detailScope!.get("child-ds" as DataSetId)).toBe(childDs);
  });
});
```

- [ ] **Step 10: Run test to verify it fails**

Run: `yarn workspace @casehub/runtime run test -- dataset-scope.test`
Expected: FAIL — `extendDataSetScope` not exported

- [ ] **Step 11: Implement extendDataSetScope**

In `packages/casehub-runtime/src/dataset-scope.ts`, the existing `walkScope` private function already does the right thing. Export a thin wrapper:

```typescript
export function extendDataSetScope(
  root: Component,
  inherited: Map<DataSetId, ExternalDataSetDef>,
  paths: PagePathMap,
  scope: DataSetScope,
): void {
  walkScope(root, inherited, paths, scope);
}
```

- [ ] **Step 12: Run tests**

Run: `yarn workspace @casehub/runtime run test -- dataset-scope.test`
Expected: ALL PASS

- [ ] **Step 13: Commit**

```
git add packages/casehub-runtime/src/page-paths.ts packages/casehub-runtime/src/page-paths.test.ts packages/casehub-runtime/src/navigation.ts packages/casehub-runtime/src/navigation.test.ts packages/casehub-runtime/src/dataset-scope.ts packages/casehub-runtime/src/dataset-scope.test.ts
git commit -m "feat: incremental extension APIs for pagePathMap, dataSetScope, pageIndex  Refs #17"
```

---

### Task 5: M4 — Lazy page activation callback and site wiring

**Files:**
- Modify: `packages/casehub-runtime/src/activation.ts`
- Modify: `packages/casehub-runtime/src/activation.test.ts`
- Modify: `packages/casehub-runtime/src/site.ts`

**Interfaces:**
- Consumes:
  - `extendPagePathMap(root, basePath, map)` from `page-paths.ts` (Task 4)
  - `extendDataSetScope(root, inherited, paths, scope)` from `dataset-scope.ts` (Task 4)
  - `extendPageIndex(root, paths, index)` from `navigation.ts` (Task 4)
  - `renderComponent(target, component, options)` from `@casehub/component`
  - `parsePage(input)` from `@casehub/ui`
  - `load as yamlLoad` from `js-yaml`
- Produces: Updated `createActivationCallback` with lazy-page handling. Signature changes — gains new parameters:
  ```typescript
  function createActivationCallback(
    registry: ComponentRegistry,
    pagePathMap: PagePathMap,
    options: {
      fetchFn: typeof globalThis.fetch;
      baseUrl: string | undefined;
      abortSignal: AbortSignal;
      permissions: PermissionContext;
      pageIndex: PageIndex;
      dataSetScope: DataSetScope;
      lazyPageResolutions: Map<Component, Component>;
    },
  ): (el: HTMLElement, component: Component) => void
  ```

- [ ] **Step 1: Write failing tests for lazy-page activation**

Add to `packages/casehub-runtime/src/activation.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import type { Component, PermissionContext } from "@casehub/component/dist/model/types.js";
import { ALLOW_ALL } from "@casehub/component/dist/model/types.js";
import { createActivationCallback } from "./activation.js";
import type { ComponentRegistry } from "./registry.js";
import type { PagePathMap } from "./page-paths.js";
import type { PageIndex } from "./navigation.js";
import type { DataSetScope } from "./dataset-scope.js";
import type { DataSetId } from "@casehub/data/dist/dataset/types.js";
import type { ExternalDataSetDef } from "@casehub/data/dist/dataset/external/types.js";

function lazySetup() {
  const registry: ComponentRegistry = new Map();
  const pagePathMap: PagePathMap = new Map();
  const pageIndex: PageIndex = new Map();
  const dataSetScope: DataSetScope = new Map();
  const lazyPageResolutions = new Map<Component, Component>();
  const abortController = new AbortController();

  const fetchFn = vi.fn<typeof globalThis.fetch>();

  const callback = createActivationCallback(registry, pagePathMap, {
    fetchFn: fetchFn as unknown as typeof globalThis.fetch,
    baseUrl: "http://example.com/",
    abortSignal: abortController.signal,
    permissions: ALLOW_ALL,
    pageIndex,
    dataSetScope,
    lazyPageResolutions,
  });

  return { registry, pagePathMap, pageIndex, dataSetScope, lazyPageResolutions, fetchFn, callback, abortController };
}

describe("lazy-page activation", () => {
  it("fetches href and renders content (Path C — async)", async () => {
    const { callback, fetchFn, lazyPageResolutions, pagePathMap } = lazySetup();

    const lazyComponent: Component = {
      type: "lazy-page",
      props: { name: "Lazy", href: "lazy.yaml" },
    };
    pagePathMap.set(lazyComponent, "Section");

    const yamlContent = "type: page\nprops:\n  name: Content";
    fetchFn.mockResolvedValueOnce(new Response(yamlContent));

    const el = document.createElement("div");
    el.dataset.componentId = "lazy-1";
    el.dataset.componentType = "lazy-page";
    document.body.appendChild(el);

    callback(el, lazyComponent);

    // Await the async fetch
    await vi.waitFor(() => {
      expect(lazyPageResolutions.has(lazyComponent)).toBe(true);
    });

    expect(fetchFn).toHaveBeenCalledTimes(1);
    expect(el.children.length).toBeGreaterThan(0);

    document.body.removeChild(el);
  });

  it("re-renders from lazyPageResolutions on re-activation (Path A — sync)", async () => {
    const { callback, fetchFn, lazyPageResolutions, pagePathMap } = lazySetup();

    const lazyComponent: Component = {
      type: "lazy-page",
      props: { name: "Lazy", href: "lazy.yaml" },
    };
    pagePathMap.set(lazyComponent, "Section");

    const yamlContent = "type: page\nprops:\n  name: Content";
    fetchFn.mockResolvedValueOnce(new Response(yamlContent));

    // First activation
    const el1 = document.createElement("div");
    el1.dataset.componentId = "lazy-1";
    el1.dataset.componentType = "lazy-page";
    document.body.appendChild(el1);
    callback(el1, lazyComponent);

    await vi.waitFor(() => {
      expect(lazyPageResolutions.has(lazyComponent)).toBe(true);
    });
    expect(el1.children.length).toBeGreaterThan(0);

    // Simulate slot swap: DOM destroyed
    el1.innerHTML = "";
    document.body.removeChild(el1);

    // Re-activation (new element, same Component)
    const el2 = document.createElement("div");
    el2.dataset.componentId = "lazy-1";
    el2.dataset.componentType = "lazy-page";
    document.body.appendChild(el2);
    callback(el2, lazyComponent);

    // Path A is synchronous — content rendered immediately
    expect(el2.children.length).toBeGreaterThan(0);
    expect(fetchFn).toHaveBeenCalledTimes(1); // no second fetch

    document.body.removeChild(el2);
  });

  it("extends pagePathMap for fetched content", async () => {
    const { callback, fetchFn, lazyPageResolutions, pagePathMap } = lazySetup();

    const lazyComponent: Component = {
      type: "lazy-page",
      props: { name: "Lazy", href: "lazy.yaml" },
    };
    pagePathMap.set(lazyComponent, "Section");

    const yamlContent = `
type: page
props:
  name: Root
slots:
  Detail:
    - type: page
      props:
        name: Detail
`;
    fetchFn.mockResolvedValueOnce(new Response(yamlContent));

    const el = document.createElement("div");
    el.dataset.componentId = "lazy-1";
    el.dataset.componentType = "lazy-page";
    document.body.appendChild(el);
    callback(el, lazyComponent);

    await vi.waitFor(() => {
      expect(lazyPageResolutions.has(lazyComponent)).toBe(true);
    });

    const resolvedRoot = lazyPageResolutions.get(lazyComponent)!;
    expect(pagePathMap.get(resolvedRoot)).toBe("Section");

    const detailPage = resolvedRoot.slots!["Detail"]![0]!;
    expect(pagePathMap.get(detailPage)).toBe("Section/Detail");

    document.body.removeChild(el);
  });

  it("passes abort signal to fetch", async () => {
    const { callback, fetchFn, abortController, pagePathMap } = lazySetup();

    const lazyComponent: Component = {
      type: "lazy-page",
      props: { name: "Lazy", href: "lazy.yaml" },
    };
    pagePathMap.set(lazyComponent, "Section");

    fetchFn.mockImplementation(() => new Promise(() => {})); // never resolves

    const el = document.createElement("div");
    el.dataset.componentId = "lazy-1";
    el.dataset.componentType = "lazy-page";
    callback(el, lazyComponent);

    expect(fetchFn).toHaveBeenCalledWith(
      expect.any(String),
      expect.objectContaining({ signal: abortController.signal }),
    );
  });

  it("renders error on fetch failure", async () => {
    const { callback, fetchFn, pagePathMap } = lazySetup();

    const lazyComponent: Component = {
      type: "lazy-page",
      props: { name: "Lazy", href: "lazy.yaml" },
    };
    pagePathMap.set(lazyComponent, "Section");

    fetchFn.mockRejectedValueOnce(new Error("Network error"));

    const el = document.createElement("div");
    el.dataset.componentId = "lazy-1";
    el.dataset.componentType = "lazy-page";
    document.body.appendChild(el);
    callback(el, lazyComponent);

    await vi.waitFor(() => {
      expect(el.textContent).toContain("Network error");
    });

    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/runtime run test -- activation.test`
Expected: FAIL — `createActivationCallback` signature doesn't accept options

- [ ] **Step 3: Implement lazy-page handling in activation.ts**

Replace `packages/casehub-runtime/src/activation.ts`:

```typescript
import type { Component, PermissionContext } from "@casehub/component/dist/model/types.js";
import type { DataSetLookup } from "@casehub/data/dist/dataset/lookup.js";
import { ColumnType } from "@casehub/data/dist/dataset/types.js";
import type { ColumnId, DataSetId } from "@casehub/data/dist/dataset/types.js";
import type { ExternalDataSetDef } from "@casehub/data/dist/dataset/external/types.js";
import { toTypedDataSet } from "@casehub/data/dist/dataset/conversion.js";
import type { CasehubElement } from "@casehub/viz/dist/base/CasehubElement.js";
import type { VizComponentProps } from "@casehub/viz/dist/base/types.js";
import { renderComponent } from "@casehub/component/dist/renderer/render.js";
import { parsePage } from "@casehub/ui/dist/parser/page-parser.js";
import { load as yamlLoad } from "js-yaml";
import type { ComponentRegistry } from "./registry.js";
import type { PagePathMap } from "./page-paths.js";
import { extendPagePathMap } from "./page-paths.js";
import type { PageIndex } from "./navigation.js";
import { extendPageIndex } from "./navigation.js";
import type { DataSetScope } from "./dataset-scope.js";
import { extendDataSetScope } from "./dataset-scope.js";
import { renderTitle, renderHtml, renderMarkdown } from "./content.js";

const DATA_COMPONENT_TYPES = new Set([
  "bar-chart",
  "line-chart",
  "area-chart",
  "pie-chart",
  "scatter-chart",
  "bubble-chart",
  "timeseries",
  "table",
  "metric",
  "meter",
  "selector",
  "map",
  "iframe-plugin",
]);

export interface LazyPageOptions {
  readonly fetchFn: typeof globalThis.fetch;
  readonly baseUrl: string | undefined;
  readonly abortSignal: AbortSignal;
  readonly permissions: PermissionContext;
  readonly pageIndex: PageIndex;
  readonly dataSetScope: DataSetScope;
  readonly lazyPageResolutions: Map<Component, Component>;
}

export function createActivationCallback(
  registry: ComponentRegistry,
  pagePathMap: PagePathMap,
  options?: LazyPageOptions,
): (el: HTMLElement, component: Component) => void {
  const yamlCache = new Map<string, string>();

  let callback: (el: HTMLElement, component: Component) => void;
  callback = (el: HTMLElement, component: Component): void => {
    const componentId = el.dataset.componentId;
    if (!componentId) return;

    const pagePath = pagePathMap.get(component) ?? "";

    if (DATA_COMPONENT_TYPES.has(component.type)) {
      const tagName = `casehub-${component.type}`;
      const vizEl = document.createElement(tagName) as CasehubElement<VizComponentProps>;

      const lookup = (component.props as Record<string, unknown> | undefined)?.lookup as
        | DataSetLookup
        | undefined;

      const entry = {
        element: el,
        vizElement: vizEl,
        component,
        pagePath,
        ...(lookup !== undefined && { originalLookup: lookup }),
      };
      registry.set(componentId, entry);

      if (component.props) {
        vizEl.props = component.props as VizComponentProps;
      }
      el.appendChild(vizEl);

      const inlineData = (component.props as Record<string, unknown> | undefined)?.inlineDataSet;
      if (inlineData !== undefined && lookup === undefined) {
        resolveInlineDataSet(vizEl, inlineData);
      }
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

    if (component.type === "lazy-page" && component.props && options) {
      const props = component.props as { name?: string; href?: string };
      if (!props.href) return;

      const { fetchFn, baseUrl, abortSignal, permissions, pageIndex, dataSetScope, lazyPageResolutions } = options;

      // Path A: re-activation — resolved root available, re-render synchronously
      const resolved = lazyPageResolutions.get(component);
      if (resolved) {
        renderComponent(el, resolved, { permissions, onNode: callback });
        return;
      }

      const url = baseUrl ? new URL(props.href, baseUrl).href : props.href;
      const cached = yamlCache.get(url);

      if (cached) {
        // Path B: YAML cache hit — synchronous
        const parsed = parsePage(yamlLoad(cached));
        integrateAndRender(el, component, parsed, pagePath, pagePathMap, pageIndex, dataSetScope, lazyPageResolutions, permissions, callback);
      } else {
        // Path C: cache miss — async
        fetchFn(url, { signal: abortSignal })
          .then((response) => response.text())
          .then((text) => {
            yamlCache.set(url, text);
            const parsed = parsePage(yamlLoad(text));
            integrateAndRender(el, component, parsed, pagePath, pagePathMap, pageIndex, dataSetScope, lazyPageResolutions, permissions, callback);
          })
          .catch((err) => {
            if (err instanceof DOMException && err.name === "AbortError") return;
            el.textContent = `Failed to load lazy page: ${err instanceof Error ? err.message : String(err)}`;
          });
      }
      return;
    }
  };

  return callback;
}

function integrateAndRender(
  el: HTMLElement,
  lazyPageComponent: Component,
  parsedRoot: Component,
  basePath: string,
  pagePathMap: PagePathMap,
  pageIndex: PageIndex,
  dataSetScope: DataSetScope,
  lazyPageResolutions: Map<Component, Component>,
  permissions: PermissionContext,
  onNode: (el: HTMLElement, component: Component) => void,
): void {
  extendPagePathMap(parsedRoot, basePath, pagePathMap);
  const inheritedScope = dataSetScope.get(basePath) ?? new Map();
  extendDataSetScope(parsedRoot, inheritedScope, pagePathMap, dataSetScope);
  extendPageIndex(parsedRoot, pagePathMap, pageIndex);
  lazyPageResolutions.set(lazyPageComponent, parsedRoot);
  renderComponent(el, parsedRoot, { permissions, onNode });
}

function resolveInlineDataSet(
  vizEl: CasehubElement<VizComponentProps>,
  inlineData: unknown,
): void {
  try {
    let raw: unknown;
    if (typeof inlineData === "string") {
      let cleaned = inlineData.replace(/,\s*([\]}])/g, "$1");
      cleaned = cleaned.replace(/'/g, '"');
      raw = JSON.parse(cleaned);
    } else {
      raw = inlineData;
    }

    if (!Array.isArray(raw)) return;

    const isFlat = raw.every((v: unknown) => typeof v !== "object" || v === null);
    const rows: unknown[][] = isFlat ? [raw] : (raw as unknown[][]);

    const maxCols = rows.reduce((max: number, row: unknown[]) => Math.max(max, row.length), 0);
    const columns = Array.from({ length: maxCols }, (_: unknown, i: number) => ({
      id: `Column ${i}` as ColumnId,
      name: `Column ${i}`,
      type: typeof rows[0]?.[i] === "number" ? ColumnType.NUMBER : ColumnType.LABEL,
    }));

    const data = rows.map((row: unknown[]) =>
      Array.from({ length: maxCols }, (_: unknown, i: number) =>
        row[i] === undefined || row[i] === null ? null : String(row[i]),
      ),
    );

    const dataset = toTypedDataSet({ columns, data });
    vizEl.dataSet = dataset;
  } catch {
    vizEl.error = "Failed to parse inline dataSet";
  }
}
```

- [ ] **Step 4: Update existing activation tests for new signature**

The existing tests call `createActivationCallback(registry, pagePathMap)` without the third parameter. Since `options` is optional (`options?: LazyPageOptions`), existing tests continue to work without changes. Verify:

Run: `yarn workspace @casehub/runtime run test -- activation.test`
Expected: existing tests PASS, new lazy-page tests PASS

- [ ] **Step 5: Wire lazy-page options in site.ts**

In `packages/casehub-runtime/src/site.ts`, update the `createActivationCallback` call (around line 199). Change:

```typescript
  const onNode = createActivationCallback(registry, pagePathMap);
```

To:

```typescript
  const onNode = createActivationCallback(registry, pagePathMap, {
    fetchFn: options?.fetch ?? globalThis.fetch?.bind(globalThis),
    baseUrl: options?.baseUrl,
    abortSignal: abortController.signal,
    permissions,
    pageIndex,
    dataSetScope,
    lazyPageResolutions,
  });
```

Add imports at the top of `site.ts`:

```typescript
import type { DataSetScope } from "./dataset-scope.js";
```

(The `pageIndex`, `dataSetScope`, `lazyPageResolutions` variables are already in scope from earlier in `loadSite`.)

- [ ] **Step 6: Run full test suites**

Run: `yarn workspace @casehub/runtime run test`
Run: `yarn workspace @casehub/component run test`
Expected: ALL tests PASS

- [ ] **Step 7: Commit**

```
git add packages/casehub-runtime/src/activation.ts packages/casehub-runtime/src/activation.test.ts packages/casehub-runtime/src/site.ts
git commit -m "feat: lazy-page activation with fetch, caching, and tree integration  Refs #17"
```

---

### Task 6: Final verification and ARC42STORIES update

**Files:**
- Modify: `ARC42STORIES.MD`

**Interfaces:**
- Consumes: all prior tasks
- Produces: updated ARC42STORIES status

- [ ] **Step 1: Run all package tests**

Run: `yarn workspace @casehub/component run test`
Run: `yarn workspace @casehub/runtime run test`
Expected: ALL tests PASS

- [ ] **Step 2: Build packages to verify TypeScript compilation**

Run: `yarn build:packages`
Expected: clean build, no type errors

- [ ] **Step 3: Update ARC42STORIES.MD**

In the backlog section, update issue #17 status from `🔲 open` to `✅ closed`:

Change line:
```
| #17 | Minor runtime improvements | 🔲 open |
```
To:
```
| #17 | Minor runtime improvements | ✅ closed |
```

- [ ] **Step 4: Commit**

```
git add ARC42STORIES.MD
git commit -m "docs: mark #17 complete in ARC42STORIES  Refs #17"
```
