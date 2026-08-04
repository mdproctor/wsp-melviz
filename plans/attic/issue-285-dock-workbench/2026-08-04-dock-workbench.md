# Dock Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #285 — feat: dock-workbench — IntelliJ-style dock layout compositor
**Issue group:** #285

**Goal:** Compose existing workbench primitives (split, dockBar, hostPanel)
into an IntelliJ-style dock layout via three new primitives and a DSL builder.

**Architecture:** Three focused primitives — `deferred` rendering, `exclusive`
dock-bar mode, and component-level dock-toggle with cascading collapse — plus a
`dockWorkbench()` DSL builder that generates a Component tree from existing
types. No new rendering path; standard `renderComponent()` pipeline throughout.

**Tech Stack:** TypeScript, Vitest, pages-component (render.ts), pages-runtime
(activation.ts, site.ts), pages-ui (builders.ts, component-desugar.ts)

## Global Constraints

- Build order: `yarn build:packages` (packages must build before tests run)
- Test runner: `vitest run` per package
- All component types use `pages-` prefix for custom element names
- `pages-dock-toggle` and `pages-split-resize` are reserved framework events
- `pages-deferred-render` will be added as a reserved framework event

---

### Task 1: `deferred` type — render gating + activation callback

**Files:**
- Modify: `packages/pages-component/src/renderer/render.ts:12` (LAZY_TYPES set)
- Modify: `packages/pages-runtime/src/activation.ts` (add deferred activation)
- Test: `packages/pages-component/src/renderer/render.test.ts` (new tests)
- Test: `packages/pages-runtime/src/activation.test.ts` (new tests)

**Interfaces:**
- Consumes: `renderComponent()` from pages-component, `ALLOW_ALL` from types
- Produces: `"deferred"` component type recognized by render pipeline; triggers
  on `pages-deferred-render` DOM event

- [ ] **Step 1: Write the failing render test**

In `packages/pages-component/src/renderer/render.test.ts`, add:

```typescript
describe("deferred component type", () => {
  it("creates container but does not render children", () => {
    const child: Component = { type: "html", props: { content: "hello" } };
    const root: Component = {
      type: "deferred",
      slots: { default: [child] },
    };
    const target = document.createElement("div");
    renderComponent(target, root);

    const deferredEl = target.querySelector('[data-component-type="deferred"]');
    expect(deferredEl).toBeTruthy();
    expect(deferredEl!.dataset.deferred).toBeUndefined(); // activation not wired in render test
    // Child should NOT be rendered — LAZY_TYPES gate prevents recursion
    const childEl = target.querySelector('[data-component-type="html"]');
    expect(childEl).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- --reporter=verbose 2>&1 | tail -20`
Expected: FAIL — child IS rendered (deferred not in LAZY_TYPES yet)

- [ ] **Step 3: Add `"deferred"` to LAZY_TYPES**

In `packages/pages-component/src/renderer/render.ts`, line 12:

```typescript
const LAZY_TYPES = new Set(["tabs", "pills", "sidebar", "carousel", "stack", "tree", "menu", "tiles", "deferred"]);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-component run test -- --reporter=verbose 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 5: Write the failing activation test**

In `packages/pages-runtime/src/activation.test.ts`, add:

```typescript
describe("deferred activation", () => {
  it("sets data-deferred=pending and renders children on pages-deferred-render", () => {
    const el = document.createElement("div");
    el.dataset.componentId = "lazy-1";
    document.body.appendChild(el);

    const child: Component = { type: "html", props: { content: "deferred content" } };
    const component: Component = {
      type: "deferred",
      slots: { default: [child] },
    };

    const callback = createActivationCallback(
      new Map(),
      new Map(),
    );
    callback(el, component);

    // Should be marked pending
    expect(el.dataset.deferred).toBe("pending");
    // Child not rendered yet
    expect(el.querySelector('[data-component-type="html"]')).toBeNull();

    // Trigger deferred render
    el.dispatchEvent(new Event("pages-deferred-render"));

    // Child rendered
    expect(el.querySelector('[data-component-type="html"]')).toBeTruthy();
    // Attribute removed
    expect(el.dataset.deferred).toBeUndefined();

    // Second dispatch is no-op (one-shot)
    el.innerHTML = "";
    el.dispatchEvent(new Event("pages-deferred-render"));
    expect(el.querySelector('[data-component-type="html"]')).toBeNull();

    document.body.removeChild(el);
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "deferred activation" 2>&1 | tail -20`
Expected: FAIL — no deferred handler in activation.ts

- [ ] **Step 7: Implement deferred activation**

In `packages/pages-runtime/src/activation.ts`, add this block after the
existing `dock-bar` handler (after line 495) and before the `lazy-page` handler:

```typescript
    if (component.type === "deferred") {
      const children = component.slots?.default ?? [];
      el.dataset.deferred = "pending";
      el.addEventListener("pages-deferred-render", () => {
        for (const child of children) {
          renderComponent(el, child, {
            permissions: options?.permissions ?? ALLOW_ALL,
            onNode: callback,
          });
        }
        delete el.dataset.deferred;
      }, { once: true });
      return;
    }
```

- [ ] **Step 8: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "deferred activation" 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 9: Run full test suite for both packages**

Run: `yarn workspace @casehubio/pages-component run test 2>&1 | tail -5`
Run: `yarn workspace @casehubio/pages-runtime run test 2>&1 | tail -5`
Expected: All existing tests pass (no regressions)

- [ ] **Step 10: Commit**

```
git add packages/pages-component/src/renderer/render.ts packages/pages-component/src/renderer/render.test.ts packages/pages-runtime/src/activation.ts packages/pages-runtime/src/activation.test.ts
git commit -m "feat(#285): deferred component type — LAZY_TYPES gate + activation callback"
```

---

### Task 2: Exclusive dock-bar mode

**Files:**
- Modify: `packages/pages-component/src/model/component-props.ts:35-38` (DockBarProps)
- Modify: `packages/pages-runtime/src/activation.ts:450-495` (dock-bar handler)
- Modify: `packages/pages-ui/src/dsl/builders.ts:475-481` (dockBar builder)
- Test: `packages/pages-runtime/src/activation.test.ts` (new tests)

**Interfaces:**
- Consumes: `DockBarProps`, `DockItem` from pages-component
- Produces: `exclusive` prop on dock-bar; two synchronous `pages-dock-toggle`
  events on panel switch, one on zone close

- [ ] **Step 1: Write the failing test**

In `packages/pages-runtime/src/activation.test.ts`, add:

```typescript
describe("exclusive dock-bar", () => {
  it("dispatches hide-previous then show-new on switch", () => {
    const el = document.createElement("div");
    el.dataset.componentId = "bar-1";
    document.body.appendChild(el);

    const component: Component = {
      type: "dock-bar",
      props: {
        orientation: "vertical",
        exclusive: true,
        items: [
          { icon: "📥", label: "Inbox", panelId: "inbox", defaultOpen: true },
          { icon: "📋", label: "Cases", panelId: "cases" },
        ],
      },
    };

    const callback = createActivationCallback(new Map(), new Map());
    callback(el, component);

    const events: Array<{ panelId: string; visible: boolean }> = [];
    el.addEventListener("pages-dock-toggle", ((e: Event) => {
      events.push((e as CustomEvent).detail);
    }) as EventListener);

    // Click "Cases" button (while "Inbox" is active)
    const casesBtn = el.querySelectorAll("button")[1]!;
    casesBtn.click();

    expect(events).toHaveLength(2);
    expect(events[0]).toEqual({ panelId: "inbox", visible: false });
    expect(events[1]).toEqual({ panelId: "cases", visible: true });

    document.body.removeChild(el);
  });

  it("dispatches hide on clicking active button", () => {
    const el = document.createElement("div");
    el.dataset.componentId = "bar-2";
    document.body.appendChild(el);

    const component: Component = {
      type: "dock-bar",
      props: {
        orientation: "vertical",
        exclusive: true,
        items: [
          { icon: "📥", label: "Inbox", panelId: "inbox", defaultOpen: true },
        ],
      },
    };

    const callback = createActivationCallback(new Map(), new Map());
    callback(el, component);

    const events: Array<{ panelId: string; visible: boolean }> = [];
    el.addEventListener("pages-dock-toggle", ((e: Event) => {
      events.push((e as CustomEvent).detail);
    }) as EventListener);

    const inboxBtn = el.querySelector("button")!;
    inboxBtn.click();

    expect(events).toHaveLength(1);
    expect(events[0]).toEqual({ panelId: "inbox", visible: false });

    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "exclusive dock-bar" 2>&1 | tail -20`
Expected: FAIL — exclusive prop not handled

- [ ] **Step 3: Add `exclusive` to DockBarProps**

In `packages/pages-component/src/model/component-props.ts`:

```typescript
export interface DockBarProps {
  readonly orientation: "vertical" | "horizontal";
  readonly items: readonly DockItem[];
  readonly exclusive?: boolean;
}
```

- [ ] **Step 4: Update dock-bar activation for exclusive mode**

In `packages/pages-runtime/src/activation.ts`, replace the dock-bar click
handler (inside the `if (component.type === "dock-bar" ...)` block). The
existing handler is at lines ~478-494. Replace the `button.addEventListener`
block:

```typescript
        button.addEventListener("click", () => {
          const isExclusive = (component.props as { exclusive?: boolean }).exclusive === true;
          const isActive = button.dataset.active !== undefined;

          if (isExclusive) {
            if (isActive) {
              // Close zone — hide this panel
              delete button.dataset.active;
              el.dispatchEvent(new CustomEvent("pages-dock-toggle", {
                bubbles: true, composed: true,
                detail: { panelId: item.panelId, visible: false },
              }));
            } else {
              // Switch — hide previous, show new
              for (const sibling of el.querySelectorAll<HTMLElement>("button[data-dock-panel-id]")) {
                if (sibling.dataset.active !== undefined) {
                  delete sibling.dataset.active;
                  el.dispatchEvent(new CustomEvent("pages-dock-toggle", {
                    bubbles: true, composed: true,
                    detail: { panelId: sibling.dataset.dockPanelId!, visible: false },
                  }));
                }
              }
              button.dataset.active = "";
              el.dispatchEvent(new CustomEvent("pages-dock-toggle", {
                bubbles: true, composed: true,
                detail: { panelId: item.panelId, visible: true },
              }));
            }
          } else {
            // Independent toggle (existing behavior)
            if (isActive) {
              delete button.dataset.active;
            } else {
              button.dataset.active = "";
            }
            el.dispatchEvent(new CustomEvent("pages-dock-toggle", {
              bubbles: true, composed: true,
              detail: { panelId: item.panelId, visible: !isActive },
            }));
          }
        });
```

Also add `button.dataset.dockPanelId = item.panelId;` when creating each
button (before the click listener) so sibling lookup works.

- [ ] **Step 5: Update dockBar builder signature**

In `packages/pages-ui/src/dsl/builders.ts`, update the `dockBar` function:

```typescript
export function dockBar(
  orientation: "vertical" | "horizontal",
  items: DockItem[],
  options?: { exclusive?: boolean },
): TypedComponent<"dock-bar"> {
  const props: DockBarProps = {
    orientation,
    items,
    ...(options?.exclusive ? { exclusive: true } : {}),
  };
  return freeze({ type: "dock-bar" as const, props });
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "exclusive dock-bar" 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 7: Run full test suites**

Run: `yarn workspace @casehubio/pages-component run test 2>&1 | tail -5`
Run: `yarn workspace @casehubio/pages-runtime run test 2>&1 | tail -5`
Run: `yarn workspace @casehubio/pages-ui run test 2>&1 | tail -5`
Expected: All pass

- [ ] **Step 8: Commit**

```
git add packages/pages-component/src/model/component-props.ts packages/pages-runtime/src/activation.ts packages/pages-runtime/src/activation.test.ts packages/pages-ui/src/dsl/builders.ts
git commit -m "feat(#285): exclusive dock-bar mode — zone-exclusive panel switching"
```

---

### Task 3: Component-level dock-toggle + cascading collapse/expand

**Files:**
- Modify: `packages/pages-runtime/src/site.ts:834-889` (dock-toggle handler)
- Test: `packages/pages-runtime/src/site.test.ts` (new tests)

**Interfaces:**
- Consumes: `pages-dock-toggle` events with `{ panelId, visible }` detail
- Produces: component-level show/hide with cascading collapse through
  `[data-slot]` and parent containers. Triggers `pages-deferred-render`
  on pending deferred elements.

- [ ] **Step 1: Write the failing test — shared slot toggle**

In `packages/pages-runtime/src/site.test.ts`, add:

```typescript
describe("component-level dock-toggle", () => {
  it("hides individual component in shared slot without hiding siblings", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const workbench: Component = {
      type: "rows",
      slots: {
        default: [
          { type: "html", id: "panel-a", props: { content: "A" } },
          { type: "html", id: "panel-b", props: { content: "B" } },
        ],
      },
    };

    const site = await loadSite(target, workbench);

    const panelA = target.querySelector('[data-component-id="panel-a"]') as HTMLElement;
    const panelB = target.querySelector('[data-component-id="panel-b"]') as HTMLElement;
    expect(panelA).toBeTruthy();
    expect(panelB).toBeTruthy();

    // Toggle panel-a hidden
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "panel-a", visible: false },
    }));

    expect(panelA.style.display).toBe("none");
    expect(panelB.style.display).not.toBe("none"); // sibling stays visible

    site.dispose();
    document.body.removeChild(target);
  });

  it("cascades collapse when all components in shared slot are hidden", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const workbench: Component = {
      type: "split",
      props: { direction: "horizontal", ratio: [50, 50] },
      slots: {
        "0": [{
          type: "rows",
          slots: {
            default: [
              { type: "html", id: "zone-panel", props: { content: "Z" } },
            ],
          },
        }],
        "1": [{ type: "html", props: { content: "Centre" } }],
      },
    };

    const site = await loadSite(target, workbench);

    // Hide zone panel → zone rows collapse → split slot "0" collapse
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "zone-panel", visible: false },
    }));

    const splitSlot0 = target.querySelector('[data-slot="0"]') as HTMLElement;
    expect(splitSlot0.style.display).toBe("none");

    // Show zone panel → cascade expand
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "zone-panel", visible: true },
    }));

    expect(splitSlot0.style.display).not.toBe("none");

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "component-level dock-toggle" 2>&1 | tail -20`
Expected: FAIL — current handler hides slot container, not component

- [ ] **Step 3: Rewrite dock-toggle handler**

In `packages/pages-runtime/src/site.ts`, replace the `pages-dock-toggle`
handler (lines 834-889) with:

```typescript
  target.addEventListener("pages-dock-toggle", ((e: Event) => {
    const { panelId, visible } = (e as CustomEvent<{ panelId: string; visible: boolean }>).detail;
    dockState.set(panelId, visible);

    const escapedId = typeof CSS !== "undefined" && CSS.escape ? CSS.escape(panelId) : panelId;
    const panelEl = target.querySelector<HTMLElement>(`[data-component-id="${escapedId}"]`);
    if (!panelEl) return;

    if (visible) {
      // Cascade expand: ensure all ancestor slots are visible (bottom-up)
      let ancestor: HTMLElement | null = panelEl;
      while (ancestor && ancestor !== target) {
        const slot = ancestor.closest<HTMLElement>("[data-slot]");
        if (!slot || slot === ancestor) { ancestor = ancestor.parentElement; continue; }
        if (slot.style.display === "none") {
          slot.style.display = slot.dataset.pagesDisplay ?? "";
          delete slot.dataset.pagesDisplay;
          // Show adjacent drag handles
          const next = slot.nextElementSibling as HTMLElement | null;
          if (next?.dataset.splitHandle !== undefined) next.style.display = "";
          const prev = slot.previousElementSibling as HTMLElement | null;
          if (prev?.dataset.splitHandle !== undefined) prev.style.display = "";
        }
        ancestor = slot.parentElement;
      }

      // Trigger deferred render if pending
      if (panelEl.dataset.deferred === "pending") {
        panelEl.dispatchEvent(new Event("pages-deferred-render"));
      }

      // Show the component
      panelEl.style.display = panelEl.dataset.pagesDisplay ?? "";
      delete panelEl.dataset.pagesDisplay;
    } else {
      // Hide the component
      panelEl.dataset.pagesDisplay = panelEl.style.display;
      panelEl.style.display = "none";

      // Cascade collapse: walk up through slots and containers
      let current: HTMLElement | null = panelEl;
      while (current && current !== target) {
        const slot = current.closest<HTMLElement>("[data-slot]");
        if (!slot || slot === current) break;

        // Check if all component children in this slot are hidden
        const siblings = slot.querySelectorAll<HTMLElement>(":scope > [data-component-id]");
        const allHidden = siblings.length > 0 && Array.from(siblings).every(s => s.style.display === "none");

        if (!allHidden) break;

        // Collapse the slot
        slot.dataset.pagesDisplay = slot.style.display;
        slot.style.display = "none";

        // Hide adjacent drag handles
        const next = slot.nextElementSibling as HTMLElement | null;
        if (next?.dataset.splitHandle !== undefined) next.style.display = "none";
        const prev = slot.previousElementSibling as HTMLElement | null;
        if (prev?.dataset.splitHandle !== undefined && prev.nextElementSibling === slot) {
          prev.style.display = "none";
        }

        // Continue cascading up from the slot's parent container
        const parentContainer = slot.parentElement;
        if (!parentContainer || parentContainer === target) break;

        // Check if all slots in the parent container are hidden
        const parentSlots = parentContainer.querySelectorAll<HTMLElement>(":scope > [data-slot]");
        const allParentSlotsHidden = parentSlots.length > 0 && Array.from(parentSlots).every(s => s.style.display === "none");

        if (!allParentSlotsHidden) break;

        // Continue cascade from the parent container
        current = parentContainer;
      }
    }

    syncUrl("replaceState");
    scheduleLayoutSave();
  }), { signal: abortController.signal });
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "component-level dock-toggle" 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 5: Run full runtime test suite — check for regressions**

Run: `yarn workspace @casehubio/pages-runtime run test 2>&1 | tail -10`
Expected: All existing dock-toggle tests pass. The component-level handler
produces the same result as the slot-level handler for splits (where each
component is the sole child of its slot).

If regressions: existing tests may assert slot-level hiding. Update assertions
to check component-level hiding instead — the behavior is equivalent for
single-child slots.

- [ ] **Step 6: Commit**

```
git add packages/pages-runtime/src/site.ts packages/pages-runtime/src/site.test.ts
git commit -m "feat(#285): component-level dock-toggle with cascading collapse/expand"
```

---

### Task 4: DSL builders — `deferred()` and `dockWorkbench()`

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts` (add deferred + dockWorkbench)
- Test: `packages/pages-ui/src/dsl/builders.test.ts` (new tests)

**Interfaces:**
- Consumes: `split()`, `dockBar()`, `rows()`, `columns()`, `withId()`,
  `withStyle()` from builders; `Component`, `DockItem` from pages-component
- Produces: `deferred(child: Component): Component`,
  `dockWorkbench(config: DockWorkbenchConfig): Component`

- [ ] **Step 1: Write the failing test for `deferred()`**

In `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
describe("deferred builder", () => {
  it("wraps child in deferred type with default slot", () => {
    const child = html("content");
    const result = deferred(child);
    expect(result.type).toBe("deferred");
    expect(result.slots?.default).toHaveLength(1);
    expect(result.slots?.default?.[0]).toEqual(child);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter=verbose -t "deferred builder" 2>&1 | tail -10`
Expected: FAIL — deferred not exported

- [ ] **Step 3: Implement `deferred()` builder**

In `packages/pages-ui/src/dsl/builders.ts`, after the `hostPanel` function:

```typescript
export function deferred(child: Component): Component {
  return freeze({ type: "deferred" as const, slots: freeze({ default: [child] }) });
}
```

Export it from the package barrel.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter=verbose -t "deferred builder" 2>&1 | tail -10`
Expected: PASS

- [ ] **Step 5: Write the failing test for `dockWorkbench()`**

In `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
describe("dockWorkbench builder", () => {
  it("generates correct tree for left + centre + bottom config", () => {
    const result = dockWorkbench({
      storageKey: "test-wb",
      centre: html("centre"),
      left: [
        { key: "inbox", label: "Inbox", icon: "📥", defaultOpen: true, content: hostPanel("inbox-panel") },
        { key: "cases", label: "Cases", icon: "📋", content: hostPanel("cases-panel") },
      ],
      bottom: [
        { key: "chat", label: "Chat", icon: "💬", content: hostPanel("chat-panel") },
      ],
    });

    // Root is rows
    expect(result.type).toBe("rows");
    const rootChildren = result.slots?.default ?? [];
    expect(rootChildren.length).toBe(2); // columns + bottom dock bar

    // First child: columns (left bar + split + no right bar)
    const cols = rootChildren[0]!;
    expect(cols.type).toBe("columns");

    // Bottom dock bar exists
    const bottomBar = rootChildren[1]!;
    expect(bottomBar.type).toBe("dock-bar");
    expect((bottomBar.props as any).exclusive).toBe(true);

    // Verify all panels have style.display = "none"
    function findById(c: Component, id: string): Component | undefined {
      if (c.id === id) return c;
      for (const children of Object.values(c.slots ?? {})) {
        for (const child of children) {
          const found = findById(child, id);
          if (found) return found;
        }
      }
      if (c.items) {
        for (const item of c.items) {
          const found = findById(item.component, id);
          if (found) return found;
        }
      }
      return undefined;
    }

    const inbox = findById(result, "inbox");
    expect(inbox).toBeTruthy();
    expect(inbox!.style?.display).toBe("none");
    expect(inbox!.type).toBe("deferred"); // all panels deferred

    const cases = findById(result, "cases");
    expect(cases).toBeTruthy();
    expect(cases!.style?.display).toBe("none");
  });

  it("generates simple tree for centre-only config", () => {
    const result = dockWorkbench({
      centre: html("just centre"),
    });
    // No splits, no dock bars — just the centre content
    expect(result.type).toBe("html");
  });

  it("omits right dock bar when no right panels", () => {
    const result = dockWorkbench({
      centre: html("c"),
      left: [{ key: "a", label: "A", icon: "a", content: html("a") }],
    });
    // Should not contain a right dock bar
    function collectTypes(c: Component): string[] {
      const types = [c.type];
      for (const children of Object.values(c.slots ?? {})) {
        for (const child of children) types.push(...collectTypes(child));
      }
      return types;
    }
    const barCount = collectTypes(result).filter(t => t === "dock-bar").length;
    expect(barCount).toBe(1); // only left dock bar
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter=verbose -t "dockWorkbench builder" 2>&1 | tail -10`
Expected: FAIL — dockWorkbench not exported

- [ ] **Step 7: Implement `dockWorkbench()` builder**

In `packages/pages-ui/src/dsl/builders.ts`, add the interfaces and builder
function. The builder generates the tree structure documented in the spec.

```typescript
export interface DockPanelConfig {
  readonly key: string;
  readonly label: string;
  readonly icon: string;
  readonly defaultOpen?: boolean;
  readonly content: Component;
  readonly minSize?: number;
}

export interface DockWorkbenchConfig {
  readonly storageKey?: string;
  readonly centre: Component | Component[];
  readonly left?: readonly DockPanelConfig[];
  readonly right?: readonly DockPanelConfig[];
  readonly bottom?: readonly DockPanelConfig[];
}

export function dockWorkbench(config: DockWorkbenchConfig): Component {
  const centreContent = Array.isArray(config.centre)
    ? rows(...config.centre)
    : config.centre;

  const hasLeft = config.left && config.left.length > 0;
  const hasRight = config.right && config.right.length > 0;
  const hasBottom = config.bottom && config.bottom.length > 0;

  // No zones — just centre
  if (!hasLeft && !hasRight && !hasBottom) return centreContent;

  function wrapPanel(panel: DockPanelConfig): Component {
    return withStyle({ display: "none" }, withId(panel.key, deferred(panel.content)));
  }

  function zoneContainer(panels: readonly DockPanelConfig[]): Component {
    return rows(...panels.map(wrapPanel));
  }

  function zoneDockBar(
    orientation: "vertical" | "horizontal",
    panels: readonly DockPanelConfig[],
  ): Component {
    const items: DockItem[] = panels.map(p => ({
      icon: p.icon,
      label: p.label,
      panelId: p.key,
      ...(p.defaultOpen ? { defaultOpen: true } : {}),
    }));
    return dockBar(orientation, items, { exclusive: true });
  }

  // Build the centre + bottom vertical split (or just centre)
  const centreArea = hasBottom
    ? split("vertical", [centreContent, zoneContainer(config.bottom!)],
        { minSizes: [100, config.bottom![0]?.minSize ?? 50] })
    : centreContent;

  // Build the horizontal split with zone containers
  const splitChildren: Component[] = [];
  const splitMinSizes: number[] = [];

  if (hasLeft) {
    splitChildren.push(zoneContainer(config.left!));
    splitMinSizes.push(config.left![0]?.minSize ?? 50);
  }

  splitChildren.push(centreArea);
  splitMinSizes.push(100); // centre min

  if (hasRight) {
    splitChildren.push(zoneContainer(config.right!));
    splitMinSizes.push(config.right![0]?.minSize ?? 50);
  }

  const mainSplit = splitChildren.length > 1
    ? split("horizontal", splitChildren, { minSizes: splitMinSizes })
    : splitChildren[0]!;

  // Build the columns row: left bar + split + right bar
  const middleChildren: Component[] = [];
  if (hasLeft) middleChildren.push(zoneDockBar("vertical", config.left!));
  middleChildren.push(mainSplit);
  if (hasRight) middleChildren.push(zoneDockBar("vertical", config.right!));

  const middleRow = middleChildren.length > 1
    ? columns([...middleChildren.map(() => 1)], ...middleChildren.map(c => [c]))
    : middleChildren[0]!;

  // Build the outer rows: middle + bottom bar
  if (hasBottom) {
    return rows(middleRow, zoneDockBar("horizontal", config.bottom!));
  }
  return middleRow;
}
```

Export `dockWorkbench`, `DockWorkbenchConfig`, `DockPanelConfig`, and
`deferred` from the package barrel.

- [ ] **Step 8: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter=verbose -t "dockWorkbench builder" 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 9: Run full test suite**

Run: `yarn workspace @casehubio/pages-ui run test 2>&1 | tail -5`
Expected: All pass

- [ ] **Step 10: Commit**

```
git add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts
git commit -m "feat(#285): deferred() and dockWorkbench() DSL builders"
```

---

### Task 5: `applyDockState` post-render initialization

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` (add applyDockState after render)
- Test: `packages/pages-runtime/src/workbench.test.ts` (new integration tests)

**Interfaces:**
- Consumes: `dockState` (Map<string, boolean>), rendered DOM with dock-bar
  buttons and deferred panels, `pages-dock-toggle` handler (from Task 3)
- Produces: correct initial visibility — defaultOpen panels shown (deferred
  triggered), non-defaultOpen hidden, dockState seeded

- [ ] **Step 1: Write the failing integration test**

In `packages/pages-runtime/src/workbench.test.ts`, add:

```typescript
import { loadSite } from "./site.js";
import { registerPanel, clearPanelRegistry } from "./panel-registry.js";
import type { Component } from "@casehubio/pages-component";

describe("applyDockState integration", () => {
  afterEach(() => {
    clearPanelRegistry();
    history.replaceState(null, "", location.pathname);
  });

  it("shows defaultOpen panels and hides others after render", async () => {
    customElements.define("test-wb-panel", class extends HTMLElement {
      configure() {}
    });
    registerPanel("wb-test", "test-wb-panel");

    const target = document.createElement("div");
    document.body.appendChild(target);

    // Simulate dockWorkbench output: two panels in a zone, one defaultOpen
    const workbench: Component = {
      type: "rows",
      slots: {
        default: [
          {
            type: "split",
            props: { direction: "horizontal", ratio: [30, 70] },
            slots: {
              "0": [{
                type: "rows",
                slots: {
                  default: [
                    { type: "deferred", id: "inbox",
                      style: { display: "none" },
                      slots: { default: [{ type: "host-panel", props: { typeName: "wb-test" } }] } },
                    { type: "deferred", id: "cases",
                      style: { display: "none" },
                      slots: { default: [{ type: "host-panel", props: { typeName: "wb-test" } }] } },
                  ],
                },
              }],
              "1": [{ type: "html", props: { content: "Centre" } }],
            },
          },
          {
            type: "dock-bar",
            props: {
              orientation: "vertical",
              exclusive: true,
              items: [
                { icon: "📥", label: "Inbox", panelId: "inbox", defaultOpen: true },
                { icon: "📋", label: "Cases", panelId: "cases" },
              ],
            },
          },
        ],
      },
    };

    const site = await loadSite(target, workbench);

    // inbox should be visible (deferred rendered)
    const inboxEl = target.querySelector('[data-component-id="inbox"]') as HTMLElement;
    expect(inboxEl.style.display).not.toBe("none");
    expect(inboxEl.dataset.deferred).toBeUndefined(); // rendered

    // cases should be hidden
    const casesEl = target.querySelector('[data-component-id="cases"]') as HTMLElement;
    expect(casesEl.style.display).toBe("none");
    expect(casesEl.dataset.deferred).toBe("pending"); // not rendered

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "applyDockState" 2>&1 | tail -20`
Expected: FAIL — no applyDockState step, inbox still display:none

- [ ] **Step 3: Implement `applyDockState`**

In `packages/pages-runtime/src/site.ts`, add the function and call it after
`applySavedSplitRatios`:

```typescript
function applyDockState(
  target: HTMLElement,
  dockState: Map<string, boolean>,
): void {
  // Collect zones from dock-bar elements
  const dockBars = target.querySelectorAll<HTMLElement>('[data-component-type="dock-bar"]');
  for (const bar of dockBars) {
    const buttons = bar.querySelectorAll<HTMLElement>("button[data-dock-panel-id]");
    let activePanel: string | undefined;

    // Check saved state first (dockState pre-seeded from LayoutStore)
    for (const btn of buttons) {
      const panelId = btn.dataset.dockPanelId!;
      if (dockState.get(panelId) === true) {
        if (activePanel === undefined) activePanel = panelId;
      }
    }

    // Fall back to defaultOpen
    if (activePanel === undefined) {
      for (const btn of buttons) {
        if (btn.dataset.active !== undefined) {
          activePanel = btn.dataset.dockPanelId!;
          break;
        }
      }
    }

    // Dispatch show for the active panel
    if (activePanel) {
      target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
        bubbles: true, composed: true,
        detail: { panelId: activePanel, visible: true },
      }));
    }

    // Sync button state and seed dockState for inactive panels
    for (const btn of buttons) {
      const panelId = btn.dataset.dockPanelId!;
      if (panelId === activePanel) {
        btn.dataset.active = "";
      } else {
        delete btn.dataset.active;
        dockState.set(panelId, false);
      }
    }
  }
}
```

Call it in `loadSite()` after `applySavedSplitRatios(target)`:

```typescript
  applySavedSplitRatios(target);
  applyDockState(target, dockState);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "applyDockState" 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test 2>&1 | tail -10`
Expected: All pass

- [ ] **Step 6: Commit**

```
git add packages/pages-runtime/src/site.ts packages/pages-runtime/src/workbench.test.ts
git commit -m "feat(#285): applyDockState post-render initialization"
```

---

### Task 6: YAML desugaring + protocol update

**Files:**
- Modify: `packages/pages-ui/src/parser/component-desugar.ts` (add dock-workbench)
- Test: `packages/pages-ui/src/parser/desugar-new-types.test.ts` (new tests)
- Modify: `docs/protocols/casehub/pages-event-contract.md` (reserved event)

**Interfaces:**
- Consumes: `dockWorkbench()` builder from Task 4
- Produces: YAML `type: dock-workbench` desugared to Component tree

- [ ] **Step 1: Write the failing desugar test**

In `packages/pages-ui/src/parser/desugar-new-types.test.ts`, add:

```typescript
describe("dock-workbench desugaring", () => {
  it("desugars dock-workbench YAML to Component tree", () => {
    const yaml = {
      type: "dock-workbench",
      storageKey: "test",
      centre: [{ type: "html", properties: { content: "centre" } }],
      left: [
        { key: "inbox", label: "Inbox", icon: "📥", defaultOpen: true,
          content: { type: "host-panel", properties: { typeName: "inbox-panel" } } },
      ],
    };

    const result = desugarComponent(yaml);
    expect(result.type).toBe("rows"); // dockWorkbench generates rows at root
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter=verbose -t "dock-workbench desugaring" 2>&1 | tail -10`
Expected: FAIL

- [ ] **Step 3: Add dock-workbench to desugar pipeline**

In `packages/pages-ui/src/parser/component-desugar.ts`, add `"dock-workbench"`
to the `DATA_COMPONENT_TYPES` set (alongside `"split"`, `"dock-bar"`,
`"host-panel"`).

Then add a handler in the desugar function that recognizes `type: dock-workbench`
and calls the `dockWorkbench()` builder with the parsed config. The handler maps
YAML content entries to `Component` objects by recursively calling
`desugarComponent` on each content entry.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-ui run test -- --reporter=verbose -t "dock-workbench desugaring" 2>&1 | tail -10`
Expected: PASS

- [ ] **Step 5: Update reserved events protocol**

In `docs/protocols/casehub/pages-event-contract.md`, add to the reserved
framework events table:

```markdown
| `pages-deferred-render` | Trigger deferred child rendering | Dock-toggle handler, deferred activation |
```

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/pages-ui run test 2>&1 | tail -5`
Expected: All pass

- [ ] **Step 7: Full cross-package build + test**

Run: `yarn build:packages && yarn typecheck 2>&1 | tail -10`
Expected: Build and type-check pass

- [ ] **Step 8: Commit**

```
git add packages/pages-ui/src/parser/component-desugar.ts packages/pages-ui/src/parser/desugar-new-types.test.ts docs/protocols/casehub/pages-event-contract.md
git commit -m "feat(#285): dock-workbench YAML desugaring + pages-deferred-render reserved event"
```

---

### Task 7: Full integration test

**Files:**
- Modify: `packages/pages-runtime/src/workbench.test.ts` (expand tests)

**Interfaces:**
- Consumes: all primitives from Tasks 1-5, `dockWorkbench()` builder from Task 4
- Produces: confidence that the full stack works end-to-end

- [ ] **Step 1: Write zone switch integration test**

In `packages/pages-runtime/src/workbench.test.ts`:

```typescript
  it("zone switch: exclusive dock bar switches panels with deferred render", async () => {
    // Use dockWorkbench builder for a realistic tree
    const workbench = dockWorkbench({
      centre: { type: "html", props: { content: "Centre" } },
      left: [
        { key: "panel-a", label: "A", icon: "a", defaultOpen: true,
          content: { type: "html", props: { content: "Panel A" } } },
        { key: "panel-b", label: "B", icon: "b",
          content: { type: "html", props: { content: "Panel B" } } },
      ],
    });

    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, workbench);

    // Panel A should be visible (defaultOpen, deferred rendered)
    const panelA = target.querySelector('[data-component-id="panel-a"]') as HTMLElement;
    expect(panelA.style.display).not.toBe("none");

    // Panel B should be hidden and not rendered
    const panelB = target.querySelector('[data-component-id="panel-b"]') as HTMLElement;
    expect(panelB.style.display).toBe("none");
    expect(panelB.dataset.deferred).toBe("pending");

    // Click dock bar button B → switch
    const buttons = target.querySelectorAll<HTMLElement>("button[data-dock-panel-id]");
    const btnB = Array.from(buttons).find(b => b.dataset.dockPanelId === "panel-b")!;
    btnB.click();

    // Panel A hidden, Panel B visible and rendered
    expect(panelA.style.display).toBe("none");
    expect(panelB.style.display).not.toBe("none");
    expect(panelB.dataset.deferred).toBeUndefined();

    site.dispose();
    document.body.removeChild(target);
  });

  it("zone close and reopen: collapse and expand cascade", async () => {
    const workbench = dockWorkbench({
      centre: { type: "html", props: { content: "Centre" } },
      left: [
        { key: "only-panel", label: "Only", icon: "o", defaultOpen: true,
          content: { type: "html", props: { content: "Only" } } },
      ],
    });

    const target = document.createElement("div");
    document.body.appendChild(target);
    const site = await loadSite(target, workbench);

    // Panel is visible
    const panel = target.querySelector('[data-component-id="only-panel"]') as HTMLElement;
    expect(panel.style.display).not.toBe("none");

    // Click active button → close zone
    const btn = target.querySelector<HTMLElement>("button[data-dock-panel-id='only-panel']")!;
    btn.click();

    expect(panel.style.display).toBe("none");

    // Click again → reopen
    btn.click();
    expect(panel.style.display).not.toBe("none");

    site.dispose();
    document.body.removeChild(target);
  });
```

- [ ] **Step 2: Run integration tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose -t "zone switch|zone close" 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 3: Run full cross-package test suite**

Run: `yarn build:packages && yarn workspace @casehubio/pages-component run test && yarn workspace @casehubio/pages-runtime run test && yarn workspace @casehubio/pages-ui run test`
Expected: All pass

- [ ] **Step 4: Commit**

```
git add packages/pages-runtime/src/workbench.test.ts
git commit -m "test(#285): full dock-workbench integration tests — zone switch, close, reopen"
```
