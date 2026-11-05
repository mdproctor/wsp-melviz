# Layout Spanning Layer 1 — Model + Rendering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #613 — Epic: Layout spanning and drag-and-drop panel placement
**Issue group:** #613

**Goal:** Enhance the `grid` layout type with a discriminated union
`GridPlacement`, auto-flow rendering via CSS `grid-auto-flow: row`, and
optional `rowHeight` support — Layer 1 of the three-layer design.

**Architecture:** `GridPlacement` becomes a discriminated union with
`mode: 'positioned'` (existing `{x, y, w, h}`) and `mode: 'auto'`
(span-only, CSS auto-placed). The renderer branches on mode to apply
CSS grid properties. A new `items:` YAML format exposes placement
objects directly to authors.

**Tech Stack:** TypeScript, Vitest, CSS Grid, jsdom

## Global Constraints

- All existing grid YAML must render identically (backward compat).
- WCAG 2.1 SC 1.3.2: DOM order must match visual order — use
  `grid-auto-flow: row` (not `dense`).
- No new runtime dependencies.
- Layer 2 (DnD) is out of scope for this plan.

---

## Batch 1: Type System Foundation

### Task 1: GridPlacement discriminated union + call site migration

**Files:**
- Modify: `packages/pages-component/src/model/types.ts:20-25`
- Modify: `packages/pages-component/src/renderer/grid.ts:1-9`
- Modify: `packages/pages-component/src/renderer/grid.test.ts:1-18`
- Modify: `packages/pages-component/src/renderer/render.ts:97-109`
- Modify: `packages/pages-ui/src/parser/page-parser.ts:284,324-326,358-360,379`
- Modify: `packages/pages-ui/src/parser/component-desugar.ts:406,414`

**Interfaces:**
- Consumes: existing `GridPlacement` interface, `applyGridPlacement` function
- Produces: discriminated union `GridPlacement` type, updated
  `applyGridPlacement(element, placement)` that branches on `mode`

- [ ] **Step 1: Write failing tests for auto mode in applyGridPlacement**

Add test cases for both modes in `grid.test.ts`:

```typescript
it("applies positioned mode with explicit coordinates", () => {
  const el = document.createElement("div");
  applyGridPlacement(el, { mode: "positioned", x: 0, y: 0, w: 6, h: 2 });
  expect(el.style.gridColumn).toBe("1 / span 6");
  expect(el.style.gridRow).toBe("1 / span 2");
});

it("applies auto mode with span only — no position", () => {
  const el = document.createElement("div");
  applyGridPlacement(el, { mode: "auto", w: 4, h: 3 });
  expect(el.style.gridColumn).toBe("span 4");
  expect(el.style.gridRow).toBe("span 3");
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-component/src/renderer/grid.test.ts`
Expected: FAIL — `mode` property doesn't exist on `GridPlacement`

- [ ] **Step 3: Change GridPlacement to discriminated union**

In `packages/pages-component/src/model/types.ts`, replace lines 20-25:

```typescript
export type GridPlacement =
  | { readonly mode: "positioned"; readonly x: number; readonly y: number; readonly w: number; readonly h: number }
  | { readonly mode: "auto"; readonly w: number; readonly h: number };
```

- [ ] **Step 4: Update applyGridPlacement to branch on mode**

In `packages/pages-component/src/renderer/grid.ts`:

```typescript
import type { GridPlacement } from "../model/types.js";

export function applyGridPlacement(
  element: HTMLElement,
  placement: GridPlacement,
): void {
  if (placement.mode === "positioned") {
    element.style.gridColumn = `${String(placement.x + 1)} / span ${String(placement.w)}`;
    element.style.gridRow = `${String(placement.y + 1)} / span ${String(placement.h)}`;
  } else {
    element.style.gridColumn = `span ${String(placement.w)}`;
    element.style.gridRow = `span ${String(placement.h)}`;
  }
}
```

- [ ] **Step 5: Update existing grid.test.ts test data**

Existing tests use `{ x, y, w, h }` without `mode`. Add
`mode: "positioned"` to both:

```typescript
it("maps 0-based placement to 1-based CSS Grid", () => {
  const el = document.createElement("div");
  applyGridPlacement(el, { mode: "positioned", x: 0, y: 0, w: 6, h: 2 });
  expect(el.style.gridColumn).toBe("1 / span 6");
  expect(el.style.gridRow).toBe("1 / span 2");
});

it("handles offset positions", () => {
  const el = document.createElement("div");
  applyGridPlacement(el, { mode: "positioned", x: 8, y: 3, w: 4, h: 1 });
  expect(el.style.gridColumn).toBe("9 / span 4");
  expect(el.style.gridRow).toBe("4 / span 1");
});
```

- [ ] **Step 6: Update render.ts items path for mode narrowing**

In `packages/pages-component/src/renderer/render.ts`, replace the items
loop (lines 97-109) with mode-aware rendering:

```typescript
if (component.items && component.items.length > 0) {
  el.style.display = "grid";
  el.style.gridTemplateColumns = "repeat(12, 1fr)";
  el.style.gridAutoRows = "min-content";
  el.style.alignContent = "start";
  el.style.gap = "var(--pages-space-3, 12px)";
  for (let i = 0; i < component.items.length; i++) {
    const item = component.items[i]!;
    if (item.placement.mode === "positioned") {
      renderNode(el, item.component, id, item.placement.x, item.placement.y, permissions, doc, onNode);
    } else {
      renderNode(el, item.component, id, "auto", i, permissions, doc, onNode);
    }
    const child = el.lastElementChild;
    if (child instanceof HTMLElement) {
      applyGridPlacement(child, item.placement);
    }
  }
}
```

- [ ] **Step 7: Migrate parser placement constructions**

In `packages/pages-ui/src/parser/page-parser.ts`, add
`mode: "positioned" as const` to every `GridPlacement` object literal:

Line 284: `{ mode: "positioned" as const, x: 0, y: i, w: 12, h: 1 }`
Lines 324-326: add `mode: "positioned" as const` to both branch placements
Lines 358-360: add `mode: "positioned" as const` to nested row placements
Line 379: add `mode: "positioned" as const` to row wrapper placement

In `packages/pages-ui/src/parser/component-desugar.ts`:

Line 406 type annotation: update to include `mode`:
```typescript
const items: { placement: GridPlacement; component: Component }[] = [];
```
Add import: `import type { GridPlacement } from "@casehubio/pages-component";`

Line 414: `{ mode: "positioned" as const, x, y: ri, w: span, h: 1 }`

- [ ] **Step 8: Run full test suite to verify backward compat**

Run: `yarn vitest run packages/pages-component/ packages/pages-ui/`
Expected: ALL PASS — existing behaviour unchanged

- [ ] **Step 9: Commit**

```bash
git add packages/pages-component/src/model/types.ts \
       packages/pages-component/src/renderer/grid.ts \
       packages/pages-component/src/renderer/grid.test.ts \
       packages/pages-component/src/renderer/render.ts \
       packages/pages-ui/src/parser/page-parser.ts \
       packages/pages-ui/src/parser/component-desugar.ts
git commit -m "feat(#613): GridPlacement discriminated union — positioned + auto modes"
```

---

## Batch 2: Renderer Enhancements

### Task 2: Fix 12-column hardcode + auto-flow + auto item rendering

**Files:**
- Modify: `packages/pages-component/src/renderer/render.ts:97-109`
- Modify: `packages/pages-component/src/renderer/layout.ts:21-27`
- Modify: `packages/pages-component/src/renderer/render.test.ts`

**Interfaces:**
- Consumes: `GridPlacement` discriminated union from Task 1,
  `applyLayoutCSS(element, component)`, `isGrid(component)`
- Produces: renderer that respects `GridProps.columns`, applies
  `grid-auto-flow: row` when auto items exist, generates correct IDs
  for auto items

- [ ] **Step 1: Write failing test for GridProps.columns in items path**

In `render.test.ts`, add:

```typescript
describe("grid items rendering", () => {
  it("uses GridProps.columns instead of hardcoded 12", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "grid",
      props: { columns: 6 },
      items: [
        { placement: { mode: "positioned", x: 0, y: 0, w: 3, h: 1 }, component: { type: "html" } },
      ],
    };
    renderComponent(target, component);
    const grid = target.firstElementChild as HTMLElement;
    expect(grid.style.gridTemplateColumns).toBe("repeat(6, 1fr)");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts -t "uses GridProps.columns"`
Expected: FAIL — gets `repeat(12, 1fr)` instead of `repeat(6, 1fr)`

- [ ] **Step 3: Write failing test for auto-flow**

```typescript
it("adds grid-auto-flow: row when auto items present", () => {
  const target = document.createElement("div");
  const component: Component = {
    type: "grid",
    items: [
      { placement: { mode: "auto", w: 4, h: 1 }, component: { type: "html" } },
    ],
  };
  renderComponent(target, component);
  const grid = target.firstElementChild as HTMLElement;
  expect(grid.style.gridAutoFlow).toBe("row");
});

it("omits grid-auto-flow when all items are positioned", () => {
  const target = document.createElement("div");
  const component: Component = {
    type: "grid",
    items: [
      { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html" } },
    ],
  };
  renderComponent(target, component);
  const grid = target.firstElementChild as HTMLElement;
  expect(grid.style.gridAutoFlow).toBe("");
});
```

- [ ] **Step 4: Write failing test for DOM ordering (WCAG §2.3.5)**

```typescript
it("renders positioned items sorted by (y, x) for WCAG DOM order", () => {
  const target = document.createElement("div");
  const component: Component = {
    type: "grid",
    items: [
      { placement: { mode: "positioned", x: 6, y: 1, w: 6, h: 1 }, component: { type: "html", id: "bottom-right" } },
      { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html", id: "top-left" } },
      { placement: { mode: "positioned", x: 6, y: 0, w: 6, h: 1 }, component: { type: "html", id: "top-right" } },
    ],
  };
  renderComponent(target, component);
  const grid = target.firstElementChild as HTMLElement;
  const children = grid.querySelectorAll(":scope > [data-component-id]");
  expect(children[0]!.getAttribute("data-component-id")).toBe("top-left");
  expect(children[1]!.getAttribute("data-component-id")).toBe("top-right");
  expect(children[2]!.getAttribute("data-component-id")).toBe("bottom-right");
});

it("renders positioned items before auto items in mixed grids", () => {
  const target = document.createElement("div");
  const component: Component = {
    type: "grid",
    items: [
      { placement: { mode: "auto", w: 6, h: 1 }, component: { type: "html", id: "auto-1" } },
      { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html", id: "pos-1" } },
    ],
  };
  renderComponent(target, component);
  const grid = target.firstElementChild as HTMLElement;
  const children = grid.querySelectorAll(":scope > [data-component-id]");
  expect(children[0]!.getAttribute("data-component-id")).toBe("pos-1");
  expect(children[1]!.getAttribute("data-component-id")).toBe("auto-1");
});
```

- [ ] **Step 5: Write failing test for auto item IDs**

```typescript
it("generates auto IDs for auto-placed items", () => {
  const target = document.createElement("div");
  const component: Component = {
    type: "grid",
    items: [
      { placement: { mode: "auto", w: 6, h: 1 }, component: { type: "html" } },
      { placement: { mode: "auto", w: 6, h: 1 }, component: { type: "html" } },
    ],
  };
  renderComponent(target, component);
  const grid = target.firstElementChild as HTMLElement;
  const children = grid.querySelectorAll(":scope > [data-component-id]");
  expect(children[0]!.getAttribute("data-component-id")).toBe("root::auto::0");
  expect(children[1]!.getAttribute("data-component-id")).toBe("root::auto::1");
});

it("generates coordinate IDs for positioned items", () => {
  const target = document.createElement("div");
  const component: Component = {
    type: "grid",
    items: [
      { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html" } },
      { placement: { mode: "positioned", x: 6, y: 0, w: 6, h: 1 }, component: { type: "html" } },
    ],
  };
  renderComponent(target, component);
  const grid = target.firstElementChild as HTMLElement;
  const children = grid.querySelectorAll(":scope > [data-component-id]");
  expect(children[0]!.getAttribute("data-component-id")).toBe("root::0::0");
  expect(children[1]!.getAttribute("data-component-id")).toBe("root::6::0");
});
```

- [ ] **Step 6: Run all new tests to verify they fail**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts -t "grid items"`
Expected: FAIL

- [ ] **Step 7: Fix the 12-column hardcode + add auto-flow + DOM ordering**

In `render.ts`, replace the items path (the block inside
`if (component.items && component.items.length > 0)`) with:

```typescript
if (component.items && component.items.length > 0) {
  el.style.gridAutoRows = "min-content";
  el.style.alignContent = "start";
  el.style.gap = "var(--pages-space-3, 12px)";

  const hasAutoItems = component.items.some(item => item.placement.mode === "auto");
  if (hasAutoItems) {
    el.style.gridAutoFlow = "row";
  }

  const positioned = component.items
    .map((item, i) => ({ item, originalIndex: i }))
    .filter(e => e.item.placement.mode === "positioned");
  const auto = component.items
    .map((item, i) => ({ item, originalIndex: i }))
    .filter(e => e.item.placement.mode === "auto");

  positioned.sort((a, b) => {
    const ap = a.item.placement as { mode: "positioned"; x: number; y: number; w: number; h: number };
    const bp = b.item.placement as { mode: "positioned"; x: number; y: number; w: number; h: number };
    return ap.y !== bp.y ? ap.y - bp.y : ap.x - bp.x;
  });

  const ordered = [...positioned, ...auto];
  for (const { item, originalIndex } of ordered) {
    if (item.placement.mode === "positioned") {
      renderNode(el, item.component, id, item.placement.x, item.placement.y, permissions, doc, onNode);
    } else {
      renderNode(el, item.component, id, "auto", originalIndex, permissions, doc, onNode);
    }
    const child = el.lastElementChild;
    if (child instanceof HTMLElement) {
      applyGridPlacement(child, item.placement);
    }
  }
}
```

This removes the hardcoded `display: grid` and `gridTemplateColumns`
from the items path — `applyLayoutCSS` (called at line 55-57) already
sets these from `GridProps.columns`. DOM children are ordered:
positioned items sorted by (y, x) first, then auto items in array order.

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts`
Expected: ALL PASS

- [ ] **Step 9: Run full component test suite**

Run: `yarn vitest run packages/pages-component/`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add packages/pages-component/src/renderer/render.ts \
       packages/pages-component/src/renderer/render.test.ts
git commit -m "feat(#613): fix 12-column hardcode, add auto-flow, DOM ordering, auto IDs"
```

### Task 3: rowHeight support + overflow handling

**Files:**
- Modify: `packages/pages-component/src/model/component-props.ts:4-6`
- Modify: `packages/pages-component/src/renderer/layout.ts:21-27`
- Modify: `packages/pages-component/src/renderer/render.ts` (items path)
- Modify: `packages/pages-component/src/renderer/render.test.ts`

**Interfaces:**
- Consumes: `GridProps`, `applyLayoutCSS`, items rendering path from Tasks 1-2
- Produces: `GridProps.rowHeight?: number`, renderer applies
  `grid-auto-rows: ${rowHeight}px` and `overflow: hidden` on items

- [ ] **Step 1: Write failing tests for rowHeight**

In `render.test.ts`:

```typescript
describe("grid rowHeight", () => {
  it("applies grid-auto-rows from rowHeight", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "grid",
      props: { columns: 12, rowHeight: 80 },
      items: [
        { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html" } },
      ],
    };
    renderComponent(target, component);
    const grid = target.firstElementChild as HTMLElement;
    expect(grid.style.gridAutoRows).toBe("80px");
  });

  it("uses min-content when rowHeight absent", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "grid",
      props: { columns: 12 },
      items: [
        { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html" } },
      ],
    };
    renderComponent(target, component);
    const grid = target.firstElementChild as HTMLElement;
    expect(grid.style.gridAutoRows).toBe("min-content");
  });

  it("sets overflow hidden on items when rowHeight present", () => {
    const target = document.createElement("div");
    const component: Component = {
      type: "grid",
      props: { columns: 12, rowHeight: 80 },
      items: [
        { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html" } },
      ],
    };
    renderComponent(target, component);
    const grid = target.firstElementChild as HTMLElement;
    const item = grid.firstElementChild as HTMLElement;
    expect(item.style.overflow).toBe("hidden");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts -t "grid rowHeight"`
Expected: FAIL

- [ ] **Step 3: Add rowHeight to GridProps**

In `packages/pages-component/src/model/component-props.ts`:

```typescript
export interface GridProps {
  readonly columns: number;
  readonly rowHeight?: number;
}
```

- [ ] **Step 4: Update layout.ts grid case for rowHeight**

In `packages/pages-component/src/renderer/layout.ts`, update the
`"grid"` case:

```typescript
case "grid": {
  element.style.display = "grid";
  if (isGrid(component)) {
    element.style.gridTemplateColumns = `repeat(${String(component.props?.columns ?? 12)}, 1fr)`;
    if (component.props?.rowHeight !== undefined) {
      element.style.gridAutoRows = `${String(component.props.rowHeight)}px`;
    }
  }
  break;
}
```

- [ ] **Step 5: Update items path for rowHeight-based gridAutoRows and overflow**

In `render.ts`, update the items path. Replace the `gridAutoRows` line
and add overflow to items:

```typescript
if (component.items && component.items.length > 0) {
  // Grid auto-rows: rowHeight from props, or min-content
  const gridProps = isGrid(component) ? component.props : undefined;
  const rowHeight = gridProps?.rowHeight;
  if (rowHeight === undefined) {
    el.style.gridAutoRows = "min-content";
  }
  // rowHeight is applied by applyLayoutCSS via layout.ts

  el.style.alignContent = "start";
  el.style.gap = "var(--pages-space-3, 12px)";

  const hasAutoItems = component.items.some(item => item.placement.mode === "auto");
  if (hasAutoItems) {
    el.style.gridAutoFlow = "row";
  }

  for (let i = 0; i < component.items.length; i++) {
    const item = component.items[i]!;
    if (item.placement.mode === "positioned") {
      renderNode(el, item.component, id, item.placement.x, item.placement.y, permissions, doc, onNode);
    } else {
      renderNode(el, item.component, id, "auto", i, permissions, doc, onNode);
    }
    const child = el.lastElementChild;
    if (child instanceof HTMLElement) {
      applyGridPlacement(child, item.placement);
      if (rowHeight !== undefined) {
        child.style.overflow = "hidden";
      }
    }
  }
}
```

Add the `isGrid` import at the top of render.ts:
```typescript
import { isGrid } from "../model/type-guards.js";
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts -t "grid rowHeight"`
Expected: ALL PASS

- [ ] **Step 7: Run full component test suite**

Run: `yarn vitest run packages/pages-component/`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-component/src/model/component-props.ts \
       packages/pages-component/src/renderer/layout.ts \
       packages/pages-component/src/renderer/render.ts \
       packages/pages-component/src/renderer/render.test.ts
git commit -m "feat(#613): optional rowHeight on GridProps with overflow handling"
```

---

## Batch 3: Parser

### Task 4: Parser — items format + backward compat

**Files:**
- Modify: `packages/pages-ui/src/parser/page-parser.ts:272-395`
- Modify: `packages/pages-ui/src/parser/page-parser.test.ts`

**Interfaces:**
- Consumes: `GridPlacement` discriminated union from Task 1,
  `desugarComponent`, `assignIdIfMissing`
- Produces: `parsePageContent` handles three formats: `items:` (new),
  `rows:` (existing), `components:` (existing). All produce
  `GridItem[]` with `mode` field.

- [ ] **Step 1: Write failing tests for new items format**

In `page-parser.test.ts`, add a new describe block:

```typescript
describe("items format", () => {
  it("parses positioned items with explicit placement", () => {
    const root = parsePage({
      pages: [{
        items: [
          { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 }, component: { type: "metric" } },
          { placement: { mode: "positioned", x: 6, y: 0, w: 6, h: 1 }, component: { type: "html" } },
        ],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items).toHaveLength(2);
    expect(page.items![0]!.placement).toEqual({ mode: "positioned", x: 0, y: 0, w: 6, h: 2 });
    expect(page.items![1]!.placement).toEqual({ mode: "positioned", x: 6, y: 0, w: 6, h: 1 });
  });

  it("parses auto items with span only", () => {
    const root = parsePage({
      pages: [{
        items: [
          { placement: { mode: "auto", w: 4, h: 1 }, component: { type: "metric" } },
        ],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items).toHaveLength(1);
    expect(page.items![0]!.placement).toEqual({ mode: "auto", w: 4, h: 1 });
  });

  it("parses mixed positioned and auto items", () => {
    const root = parsePage({
      pages: [{
        items: [
          { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 }, component: { type: "metric" } },
          { placement: { mode: "auto", w: 6, h: 1 }, component: { type: "html" } },
        ],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items).toHaveLength(2);
    expect(page.items![0]!.placement.mode).toBe("positioned");
    expect(page.items![1]!.placement.mode).toBe("auto");
  });

  it("treats placement without mode as positioned (backward compat)", () => {
    const root = parsePage({
      pages: [{
        items: [
          { placement: { x: 0, y: 0, w: 6, h: 1 }, component: { type: "html" } },
        ],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items![0]!.placement.mode).toBe("positioned");
  });

  it("items format takes priority over components format", () => {
    const root = parsePage({
      pages: [{
        items: [
          { placement: { mode: "auto", w: 12, h: 1 }, component: { type: "metric" } },
        ],
        components: [{ html: "ignored" }],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items).toHaveLength(1);
    expect(page.items![0]!.placement.mode).toBe("auto");
  });

  it("clamps w to 12 columns by default", () => {
    const root = parsePage({
      pages: [{
        items: [
          { placement: { mode: "auto", w: 15, h: 1 }, component: { type: "html" } },
        ],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items![0]!.placement.w).toBe(12);
  });
});
```

- [ ] **Step 2: Write failing test for mode field on existing formats**

```typescript
describe("existing formats produce mode: positioned", () => {
  it("components format adds mode: positioned", () => {
    const root = parsePage({
      pages: [{ components: [{ html: "Hello" }] }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items![0]!.placement.mode).toBe("positioned");
  });

  it("rows format adds mode: positioned", () => {
    const root = parsePage({
      pages: [{
        rows: [{
          columns: [{ span: 6, components: [{ html: "A" }] }],
        }],
      }],
    });
    const page = root.slots!["content"]![0]!;
    expect(page.items![0]!.placement.mode).toBe("positioned");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-ui/src/parser/page-parser.test.ts -t "items format"`
Expected: FAIL — `items:` key not handled, no `mode` field

- [ ] **Step 4: Add items format branch to parsePageContent**

In `packages/pages-ui/src/parser/page-parser.ts`, add the `items:`
branch at the start of `parsePageContent` (before the `components:`
check). Update the function:

```typescript
export function parsePageContent(
  pageRaw: Record<string, unknown>,
  pageIndex: number,
  displayerDefaults?: Record<string, unknown>,
): { items?: readonly GridItem[] } | { slots?: Readonly<Record<string, readonly Component[]>> } {
  const itemsRaw = pageRaw["items"] as unknown[] | undefined;
  const componentsRaw = pageRaw["components"] as unknown[] | undefined;
  const rowsRaw = pageRaw["rows"] as unknown[] | undefined;

  // New: items format — explicit placement objects
  if (itemsRaw) {
    const maxColumns = 12;
    const items: GridItem[] = itemsRaw.map((entryRaw, i) => {
      const entry = entryRaw as Record<string, unknown>;
      const placementRaw = entry["placement"] as Record<string, unknown> | undefined;
      const componentRaw = entry["component"] as Record<string, unknown> | undefined;
      const component = desugarComponent(componentRaw ?? {}, displayerDefaults);

      let placement: GridPlacement;
      if (placementRaw) {
        const mode = (placementRaw["mode"] as string) ?? "positioned";
        const w = Math.min(Number(placementRaw["w"] ?? 1), maxColumns);
        const h = Number(placementRaw["h"] ?? 1);
        if (mode === "auto") {
          placement = { mode: "auto", w, h };
        } else {
          const x = Number(placementRaw["x"] ?? 0);
          const y = Number(placementRaw["y"] ?? 0);
          placement = { mode: "positioned", x, y, w, h };
        }
      } else {
        placement = { mode: "auto", w: 1, h: 1 };
      }

      return {
        placement,
        component: assignIdIfMissing(component, `grid_${String(pageIndex)}_item_${String(i)}`),
      };
    });
    return { items };
  }

  // Existing componentsRaw and rowsRaw branches are unchanged here —
  // mode: "positioned" was already added in Task 1 Step 7.
  if (componentsRaw) {
```

Add the `GridPlacement` type import at the top of the file:
```typescript
import type { GridPlacement } from "@casehubio/pages-component";
```

- [ ] **Step 5: Add test for type: columns desugar path**

The `component-desugar.ts` columns path was updated in Task 1 Step 7
to include `mode: "positioned"`. Verify it works end-to-end:

```typescript
it("type: columns desugar produces mode: positioned", () => {
  const root = parsePage({
    pages: [{
      components: [
        { type: "columns", columns: [
          { components: [{ html: "A" }] },
          { components: [{ html: "B" }] },
        ]},
      ],
    }],
  });
  const page = root.slots!["content"]![0]!;
  const gridItem = page.items![0]!;
  const nestedGrid = gridItem.component;
  expect(nestedGrid.type).toBe("grid");
  expect(nestedGrid.items![0]!.placement.mode).toBe("positioned");
});
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-ui/src/parser/page-parser.test.ts`
Expected: ALL PASS

- [ ] **Step 7: Run full pages-ui test suite**

Run: `yarn vitest run packages/pages-ui/`
Expected: ALL PASS

- [ ] **Step 8: Run cross-package build and test**

Run: `yarn build:packages && yarn vitest run`
Expected: Build succeeds, all tests pass

- [ ] **Step 9: Commit**

```bash
git add packages/pages-ui/src/parser/page-parser.ts \
       packages/pages-ui/src/parser/page-parser.test.ts
git commit -m "feat(#613): items YAML format with auto/positioned placement modes"
```

---

## References

- `specs/issue-613-layout-spanning-dnd/2026-11-05-layout-spanning-dnd-design.md` — design spec (Layer 1: §2)
- `packages/pages-component/src/model/types.ts:20-25` — current `GridPlacement` interface
- `packages/pages-component/src/model/component-props.ts:4-6` — current `GridProps`
- `packages/pages-component/src/renderer/grid.ts:1-9` — `applyGridPlacement`
- `packages/pages-component/src/renderer/render.ts:96-109` — items rendering path (12-col hardcode)
- `packages/pages-component/src/renderer/layout.ts:21-27` — `applyLayoutCSS` grid case
- `packages/pages-component/src/renderer/id.ts:1-8` — `generateId`
- `packages/pages-ui/src/parser/page-parser.ts:272-395` — `parsePageContent`
- `packages/pages-ui/src/parser/component-desugar.ts:396-424` — columns layout desugaring
- `packages/pages-document/src/container-descriptors.ts:12-26` — `CONTAINER_DESCRIPTORS`
- GitHub #613 — Epic: Layout spanning and drag-and-drop panel placement
