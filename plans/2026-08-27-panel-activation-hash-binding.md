# Panel Activation & Hash Binding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — issues to be created before implementation
**Issue group:** TBD

**Goal:** Add programmatic dock panel activation (`site.activateDockPanel(key)`) and hash-based panel deep linking (`?panel=backlog,properties`) to the workbench.

**Architecture:** Three layered changes: (1) centralize dock-toggle exclusivity in the `pages-dock-toggle` handler in site.ts so any dispatch source gets correct zone-exclusive behavior, (2) add `activateDockPanel` as a `LiveSite` method using DOM lookup, (3) extend `DeepLink` with a `panel` field and wire it through URL parsing, serialization, and `restoreFromUrl`.

**Tech Stack:** TypeScript, Vitest, `@casehubio/pages-runtime`, `@casehubio/pages-ui`

## Global Constraints

- No new packages — extend `pages-runtime` and `pages-ui` (PP-20260810-72779a)
- `pages-dock-toggle` is a reserved framework event — no new event names (PP-20260705-bac842)
- ZoneLayoutEngine stays pure data — no DOM access (PP-20260810-72779a)
- DOM navigation and URL sync are separate concerns (GE-20260625-fa01da)
- Tests use Vitest with `jsdom` environment

---

## Batch 1: Dock-toggle exclusivity centralization

After this batch: dock-toggle handler in site.ts is the single owner of zone-exclusive panel visibility. Dock-bar click handler is a thin dispatcher. All existing dock behavior works identically.

### Task 1: Centralize exclusivity logic in dock-toggle handler

**Files:**
- Modify: `packages/pages-runtime/src/dock-bar-renderer.ts` — simplify click handler
- Modify: `packages/pages-runtime/src/site.ts:887-959` — dock-toggle handler gains exclusivity
- Test: `packages/pages-runtime/src/dock-bar-zones.test.ts` — extend with exclusivity tests

**Interfaces:**
- Consumes: `pages-dock-toggle` CustomEvent with `{ panelId: string; visible: boolean }` detail
- Produces: Same event contract — no interface change. Behavior change: the handler now enforces exclusivity and syncs dock-bar button `data-active` state.

- [ ] **Step 1: Write the failing test — exclusivity via direct dispatch**

Add to `dock-bar-zones.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from "vitest";

describe("dock-toggle handler exclusivity", () => {
  let target: HTMLElement;

  beforeEach(() => {
    target = document.createElement("div");
    document.body.appendChild(target);

    // Set up a dock zone with two panels and dock-bar buttons
    const bar = document.createElement("div");
    bar.dataset.componentType = "dock-bar";
    const zoneGroup = document.createElement("div");
    zoneGroup.dataset.dockZone = "top";

    const btn1 = document.createElement("button");
    btn1.dataset.dockPanelId = "panel-a";
    btn1.dataset.dockZone = "top";
    const btn2 = document.createElement("button");
    btn2.dataset.dockPanelId = "panel-b";
    btn2.dataset.dockZone = "top";

    zoneGroup.appendChild(btn1);
    zoneGroup.appendChild(btn2);
    bar.appendChild(zoneGroup);
    target.appendChild(bar);

    // Two panel elements
    const panelA = document.createElement("div");
    panelA.dataset.componentId = "panel-a";
    panelA.style.display = "none";
    const panelB = document.createElement("div");
    panelB.dataset.componentId = "panel-b";
    panelB.style.display = "none";
    target.appendChild(panelA);
    target.appendChild(panelB);
  });

  it("showing panel-b hides panel-a in same zone when dispatched directly", () => {
    // First show panel-a
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "panel-a", visible: true },
    }));

    const panelA = target.querySelector<HTMLElement>('[data-component-id="panel-a"]')!;
    expect(panelA.style.display).not.toBe("none");

    // Now show panel-b — should hide panel-a
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "panel-b", visible: true },
    }));

    const panelB = target.querySelector<HTMLElement>('[data-component-id="panel-b"]')!;
    expect(panelB.style.display).not.toBe("none");
    expect(panelA.style.display).toBe("none");
  });

  it("dock-bar button data-active syncs on direct dispatch", () => {
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "panel-a", visible: true },
    }));

    const btn1 = target.querySelector<HTMLElement>('button[data-dock-panel-id="panel-a"]')!;
    const btn2 = target.querySelector<HTMLElement>('button[data-dock-panel-id="panel-b"]')!;
    expect(btn1.dataset.active).toBeDefined();
    expect(btn2.dataset.active).toBeUndefined();

    // Show panel-b
    target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
      bubbles: true, composed: true,
      detail: { panelId: "panel-b", visible: true },
    }));

    expect(btn1.dataset.active).toBeUndefined();
    expect(btn2.dataset.active).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test dock-bar-zones.test.ts`
Expected: FAIL — current dock-toggle handler has no exclusivity logic

- [ ] **Step 3: Add exclusivity and button-sync to dock-toggle handler**

In `packages/pages-runtime/src/site.ts`, modify the `pages-dock-toggle` handler (line 887). After `dockState.set(panelId, visible)` and before the panel show/hide logic, add:

```typescript
target.addEventListener("pages-dock-toggle", ((e: Event) => {
  const { panelId, visible } = (e as CustomEvent<{ panelId: string; visible: boolean }>).detail;
  dockState.set(panelId, visible);

  const escapedId = typeof CSS !== "undefined" && CSS.escape ? CSS.escape(panelId) : panelId;
  const panelEl = target.querySelector<HTMLElement>(`[data-component-id="${escapedId}"]`);
  if (!panelEl) return;

  // Zone-exclusive: if showing, hide other panels in same zone group
  if (visible) {
    const btn = target.querySelector<HTMLElement>(`button[data-dock-panel-id="${escapedId}"]`);
    const zoneName = btn?.dataset.dockZone;
    if (zoneName) {
      const zoneBtns = target.querySelectorAll<HTMLElement>(`button[data-dock-zone="${zoneName}"]`);
      for (const sibling of zoneBtns) {
        const siblingId = sibling.dataset.dockPanelId!;
        if (siblingId !== panelId && dockState.get(siblingId) === true) {
          dockState.set(siblingId, false);
          delete sibling.dataset.active;
          const sibEscaped = typeof CSS !== "undefined" && CSS.escape ? CSS.escape(siblingId) : siblingId;
          const sibPanel = target.querySelector<HTMLElement>(`[data-component-id="${sibEscaped}"]`);
          if (sibPanel) {
            sibPanel.dataset.pagesDisplay = sibPanel.style.display;
            sibPanel.style.display = "none";
          }
        }
      }
    }
    // Sync this button's active state
    if (btn) btn.dataset.active = "";
  } else {
    // Sync button: remove active
    const btn = target.querySelector<HTMLElement>(`button[data-dock-panel-id="${escapedId}"]`);
    if (btn) delete btn.dataset.active;
  }

  // ... existing cascade expand/collapse logic unchanged ...
```

- [ ] **Step 4: Simplify dock-bar-renderer click handler**

In `packages/pages-runtime/src/dock-bar-renderer.ts`, replace the click handler (line 48-89) with a thin dispatcher:

```typescript
button.addEventListener("click", () => {
  const isActive = button.dataset.active !== undefined;
  eventTarget.dispatchEvent(new CustomEvent("pages-dock-toggle", {
    bubbles: true, composed: true,
    detail: { panelId: item.panelId, visible: !isActive },
  }));
});
```

The handler no longer manages `data-active` or sibling exclusivity — the dock-toggle handler in site.ts now owns both.

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test dock-bar-zones.test.ts`
Expected: PASS

- [ ] **Step 6: Run the full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All existing tests pass — behavior is unchanged, just centralized.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/dock-bar-renderer.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/dock-bar-zones.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: centralize dock-toggle exclusivity in site.ts handler

Move zone-group exclusivity logic and dock-bar button data-active
management from dock-bar-renderer click handler into the
pages-dock-toggle event handler in site.ts. The click handler is
now a thin dispatcher. Any code dispatching pages-dock-toggle
directly (initDockZoneGroup, rearrange, future activateDockPanel)
gets correct exclusivity behavior.

Refs #TBD"
```

---

## Batch 2: activateDockPanel API + popstate fix

After this batch: `site.activateDockPanel("key")` works for programmatic activation. Browser back/forward correctly toggles dock panels.

### Task 2: Fix restoreFromUrl dock state dispatch

**Files:**
- Modify: `packages/pages-runtime/src/site.ts:319-354` — `restoreFromUrl` function
- Test: `packages/pages-runtime/src/site.test.ts` — add popstate dock restoration tests

**Interfaces:**
- Consumes: `parseFromUrl` result (`DeepLink` with `dock` field)
- Produces: `pages-dock-toggle` events for changed dock state entries. No interface change — internal behavior fix.

- [ ] **Step 1: Write the failing test — restoreFromUrl dispatches dock-toggle**

Add to `site.test.ts`:

```typescript
describe("restoreFromUrl dock dispatch", () => {
  it("dock state changes dispatch pages-dock-toggle events", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    // Build a dock workbench with two panels
    const { dockWorkbench, hostPanel } = await import("@casehubio/pages-ui/dist/dsl/builders.js");
    const config = dockWorkbench({
      centre: { type: "html", props: { content: "<p>Centre</p>" } },
      left: [
        { key: "nav", label: "Nav", icon: "N", defaultOpen: true,
          content: { type: "html", props: { content: "<p>Nav</p>" } } },
        { key: "files", label: "Files", icon: "F",
          content: { type: "html", props: { content: "<p>Files</p>" } } },
      ],
    });

    const site = await loadSite(target, config);

    // Track dock-toggle events
    const events: Array<{ panelId: string; visible: boolean }> = [];
    target.addEventListener("pages-dock-toggle", ((e: Event) => {
      const d = (e as CustomEvent).detail;
      events.push({ panelId: d.panelId, visible: d.visible });
    }) as EventListener);

    // Simulate popstate with dock state change
    window.location.hash = "#/page/?dock=nav:closed,files:open";
    window.dispatchEvent(new PopStateEvent("popstate"));

    // Should have dispatched toggle events for changed panels
    expect(events.some(e => e.panelId === "nav" && !e.visible)).toBe(true);
    expect(events.some(e => e.panelId === "files" && e.visible)).toBe(true);

    site.dispose();
    document.body.removeChild(target);
    window.location.hash = "";
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test site.test.ts`
Expected: FAIL — `restoreFromUrl` currently updates dockState map but doesn't dispatch events

- [ ] **Step 3: Fix restoreFromUrl to dispatch dock-toggle events**

In `packages/pages-runtime/src/site.ts`, modify `restoreFromUrl` (line 349-352):

```typescript
if (link.dock) {
  for (const [id, state] of Object.entries(link.dock)) {
    const visible = state === "open";
    const wasVisible = dockState.get(id);
    dockState.set(id, visible);
    if (visible !== wasVisible) {
      target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
        bubbles: true, composed: true,
        detail: { panelId: id, visible },
      }));
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test site.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/site.ts packages/pages-runtime/src/site.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "fix: restoreFromUrl dispatches dock-toggle events on popstate

Previously, restoreFromUrl updated the dockState map but never
dispatched pages-dock-toggle events, so dock panels didn't
actually show/hide on browser back/forward navigation.

Refs #TBD"
```

### Task 3: Add activateDockPanel to LiveSite

**Files:**
- Modify: `packages/pages-runtime/src/site.ts:123` — LiveSite interface
- Modify: `packages/pages-runtime/src/site.ts:1370` — site object literal
- Modify: `packages/pages-runtime/src/site.ts` — add closure function inside loadSite()
- Test: `packages/pages-runtime/src/site.test.ts` — add activateDockPanel tests

**Interfaces:**
- Consumes: `loadSite()` closure state (`target`, `dockState`)
- Produces: `activateDockPanel(key: string): boolean` on `LiveSite`

- [ ] **Step 1: Write the failing test — activateDockPanel shows panel**

Add to `site.test.ts`:

```typescript
describe("activateDockPanel", () => {
  it("shows the panel and returns true", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const { dockWorkbench } = await import("@casehubio/pages-ui/dist/dsl/builders.js");
    const config = dockWorkbench({
      centre: { type: "html", props: { content: "<p>Centre</p>" } },
      left: [
        { key: "nav", label: "Nav", icon: "N",
          content: { type: "html", props: { content: "<p>Nav</p>" } } },
      ],
    });

    const site = await loadSite(target, config);

    const result = site.activateDockPanel("nav");
    expect(result).toBe(true);

    const panel = target.querySelector<HTMLElement>('[data-component-id="nav"]')!;
    expect(panel.style.display).not.toBe("none");

    site.dispose();
    document.body.removeChild(target);
  });

  it("returns false for unknown key", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const site = await loadSite(target, { type: "html", props: { content: "<p>Hello</p>" } });

    const result = site.activateDockPanel("nonexistent");
    expect(result).toBe(false);

    site.dispose();
    document.body.removeChild(target);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test site.test.ts`
Expected: FAIL — `activateDockPanel` is not a property on LiveSite

- [ ] **Step 3: Add activateDockPanel to LiveSite interface**

In `packages/pages-runtime/src/site.ts`, line 123:

```typescript
export interface LiveSite extends Site {
  navigate(path: string): void;
  setTheme(mode: "light" | "dark"): void;
  dispose(): void;
  readonly layout: LayoutState;
  activateDockPanel(key: string): boolean;
}
```

- [ ] **Step 4: Add closure function and wire into site object**

Inside `loadSite()`, before the `const site: LiveSite = {` block (around line 1365):

```typescript
function activateDockPanel(key: string): boolean {
  const escapedId = typeof CSS !== "undefined" && CSS.escape ? CSS.escape(key) : key;
  const panelEl = target.querySelector<HTMLElement>(`[data-component-id="${escapedId}"]`);
  if (!panelEl) return false;
  target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
    bubbles: true, composed: true,
    detail: { panelId: key, visible: true },
  }));
  return true;
}
```

Add to the site object literal (line 1370):

```typescript
const site: LiveSite = {
  // ... existing methods ...
  activateDockPanel,
};
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test site.test.ts`
Expected: PASS

- [ ] **Step 6: Run the full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/site.ts packages/pages-runtime/src/site.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add activateDockPanel to LiveSite interface

Programmatic dock panel activation via site.activateDockPanel(key).
Dispatches pages-dock-toggle with visible:true — the centralized
handler enforces zone exclusivity and syncs dock-bar button state.
Uses DOM lookup (data-component-id), not zoneEngine, so it works
for both dockWorkbench() and manual dock-bar compositions.

Refs #TBD"
```

---

## Batch 3: Hash panel binding

After this batch: deep links with `?panel=backlog,properties` activate dock panels on load and browser navigation. Panel-only deep links (`#?panel=backlog`) work.

### Task 4: Extend DeepLink and URL parsing/serialization with panel= support

**Files:**
- Modify: `packages/pages-ui/src/model/page-types.ts:27` — DeepLink interface
- Modify: `packages/pages-runtime/src/url.ts:19` — serializeToUrl
- Modify: `packages/pages-runtime/src/url.ts:79` — parseFromUrl
- Test: `packages/pages-runtime/src/url.test.ts` — add panel tests

**Interfaces:**
- Consumes: Nothing from prior tasks
- Produces: `DeepLink.panel?: readonly string[]`, updated `parseFromUrl` and `serializeToUrl`

- [ ] **Step 1: Write the failing tests — panel param parsing and serialization**

Add to `url.test.ts`:

```typescript
describe("panel param", () => {
  describe("parseFromUrl", () => {
    it("parses single panel", () => {
      const link = parseFromUrl("#/page/dashboard?panel=backlog");
      expect(link.panel).toEqual(["backlog"]);
    });

    it("parses multiple panels", () => {
      const link = parseFromUrl("#/page/dashboard?panel=backlog,properties");
      expect(link.panel).toEqual(["backlog", "properties"]);
    });

    it("parses panel-only deep link without page prefix", () => {
      const link = parseFromUrl("#?panel=backlog");
      expect(link.page).toBe("");
      expect(link.panel).toEqual(["backlog"]);
    });

    it("parses panel alongside dock", () => {
      const link = parseFromUrl("#/page/dashboard?dock=nav:open&panel=backlog");
      expect(link.dock).toEqual({ nav: "open" });
      expect(link.panel).toEqual(["backlog"]);
    });

    it("ignores empty panel param", () => {
      const link = parseFromUrl("#/page/dashboard?panel=");
      expect(link.panel).toBeUndefined();
    });

    it("returns empty page for unknown hash format", () => {
      const link = parseFromUrl("#unknown");
      expect(link.page).toBe("");
      expect(link.panel).toBeUndefined();
    });
  });

  describe("serializeToUrl", () => {
    it("serializes panel param", () => {
      const link: DeepLink = { page: "dashboard", panel: ["backlog"] };
      expect(serializeToUrl(link)).toBe("#/page/dashboard?panel=backlog");
    });

    it("serializes multiple panels", () => {
      const link: DeepLink = { page: "dashboard", panel: ["backlog", "properties"] };
      expect(serializeToUrl(link)).toBe("#/page/dashboard?panel=backlog,properties");
    });

    it("panel-only link uses #? format", () => {
      const link: DeepLink = { page: "", panel: ["backlog"] };
      expect(serializeToUrl(link)).toBe("#?panel=backlog");
    });

    it("empty page with no params produces empty string", () => {
      const link: DeepLink = { page: "" };
      expect(serializeToUrl(link)).toBe("");
    });
  });

  describe("round-trip", () => {
    it("panel with page round-trips", () => {
      const original: DeepLink = { page: "dashboard", panel: ["backlog", "props"] };
      const url = serializeToUrl(original);
      const parsed = parseFromUrl(url);
      expect(parsed.page).toBe("dashboard");
      expect(parsed.panel).toEqual(["backlog", "props"]);
    });

    it("panel-only round-trips", () => {
      const original: DeepLink = { page: "", panel: ["backlog"] };
      const url = serializeToUrl(original);
      const parsed = parseFromUrl(url);
      expect(parsed.page).toBe("");
      expect(parsed.panel).toEqual(["backlog"]);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test url.test.ts`
Expected: FAIL — `panel` not on DeepLink, parser doesn't handle it

- [ ] **Step 3: Add panel field to DeepLink**

In `packages/pages-ui/src/model/page-types.ts`, add after line 33 (`dock?`):

```typescript
readonly panel?: readonly string[];
```

- [ ] **Step 4: Extend parseFromUrl**

In `packages/pages-runtime/src/url.ts`, replace the opening guard (lines 79-86) and add panel parsing:

```typescript
export function parseFromUrl(hash: string): DeepLink {
  if (!hash) return { page: "" };

  let page = "";
  let queryStr = "";

  if (hash.startsWith("#/page/")) {
    const withoutPrefix = hash.substring("#/page/".length);
    const qIndex = withoutPrefix.indexOf("?");
    page = qIndex === -1
      ? decodePagePath(withoutPrefix)
      : decodePagePath(withoutPrefix.substring(0, qIndex));
    if (qIndex !== -1) queryStr = withoutPrefix.substring(qIndex + 1);
  } else if (hash.startsWith("#?")) {
    queryStr = hash.substring(2);
  } else {
    return { page: "" };
  }

  let filters: Record<string, readonly string[]> | undefined;
  let sort: Record<string, { readonly columnId: string; readonly order: "ASCENDING" | "DESCENDING" }> | undefined;
  let pagination: Record<string, number> | undefined;
  let textFilter: Record<string, string> | undefined;
  let dock: Record<string, "open" | "closed"> | undefined;
  let panel: string[] | undefined;

  if (queryStr) {
    const params = new URLSearchParams(queryStr);

    // ... existing filter/sort/pagination/textFilter/dock parsing unchanged ...

    const panelStr = params.get("panel");
    if (panelStr) {
      panel = panelStr.split(",").map(decodeURIComponent).filter(Boolean);
      if (panel.length === 0) panel = undefined;
    }
  }

  return {
    page,
    ...(filters ? { filters } : {}),
    ...(sort ? { sort } : {}),
    ...(pagination ? { pagination } : {}),
    ...(textFilter ? { textFilter } : {}),
    ...(dock ? { dock } : {}),
    ...(panel ? { panel } : {}),
  };
}
```

- [ ] **Step 5: Extend serializeToUrl**

In `packages/pages-runtime/src/url.ts`, modify `serializeToUrl`:

Add panel serialization after the dock block:

```typescript
if (link.panel && link.panel.length > 0) {
  params.push(`panel=${link.panel.map(encodeURIComponent).join(",")}`);
}
```

Replace the URL assembly at the end of the function:

```typescript
if (link.page) {
  let url = `#/page/${encodePagePath(link.page)}`;
  if (params.length > 0) url += `?${params.join("&")}`;
  return url;
} else if (params.length > 0) {
  return `#?${params.join("&")}`;
} else {
  return "";
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test url.test.ts`
Expected: PASS

- [ ] **Step 7: Check existing url tests still pass**

Run: `yarn workspace @casehubio/pages-runtime run test url.test.ts`
Expected: All tests pass. Note: the existing test `"root page (empty path)"` asserts `serializeToUrl({ page: "" })` returns `"#/page/"`. This will now return `""` (empty string). Update the test:

```typescript
it("root page (empty path)", () => {
  const link: DeepLink = { page: "" };
  expect(serializeToUrl(link)).toBe("");
});
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui/src/model/page-types.ts packages/pages-runtime/src/url.ts packages/pages-runtime/src/url.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add panel= param to DeepLink for dock panel deep linking

Extends DeepLink with panel?: readonly string[]. parseFromUrl handles
panel= as comma-separated list, also accepts #?panel=backlog format
for panel-only deep links (no page path prefix required).
serializeToUrl produces #? format when page is empty.

Refs #TBD"
```

### Task 5: Wire panel merge into restoreFromUrl

**Files:**
- Modify: `packages/pages-runtime/src/site.ts:319-354` — restoreFromUrl adds panel merge
- Test: `packages/pages-runtime/src/site.test.ts` — add panel merge tests

**Interfaces:**
- Consumes: `DeepLink.panel` from Task 4, `restoreFromUrl` from Task 2
- Produces: `restoreFromUrl` handles `link.panel` — merges panel overrides into dock state before dispatching events

- [ ] **Step 1: Write the failing test — panel overrides dock in restoreFromUrl**

Add to `site.test.ts`:

```typescript
describe("panel merge in restoreFromUrl", () => {
  it("panel=nav activates nav panel even when dock=nav:closed", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const { dockWorkbench } = await import("@casehubio/pages-ui/dist/dsl/builders.js");
    const config = dockWorkbench({
      centre: { type: "html", props: { content: "<p>Centre</p>" } },
      left: [
        { key: "nav", label: "Nav", icon: "N", defaultOpen: true,
          content: { type: "html", props: { content: "<p>Nav</p>" } } },
      ],
    });

    const site = await loadSite(target, config);

    // Simulate popstate with conflicting dock and panel
    window.location.hash = "#/page/?dock=nav:closed&panel=nav";
    window.dispatchEvent(new PopStateEvent("popstate"));

    // Panel override should win — nav is visible
    const panel = target.querySelector<HTMLElement>('[data-component-id="nav"]')!;
    expect(panel.style.display).not.toBe("none");

    site.dispose();
    document.body.removeChild(target);
    window.location.hash = "";
  });

  it("panel activates panels not mentioned in dock param", async () => {
    const target = document.createElement("div");
    document.body.appendChild(target);

    const { dockWorkbench } = await import("@casehubio/pages-ui/dist/dsl/builders.js");
    const config = dockWorkbench({
      centre: { type: "html", props: { content: "<p>Centre</p>" } },
      left: [
        { key: "nav", label: "Nav", icon: "N",
          content: { type: "html", props: { content: "<p>Nav</p>" } } },
      ],
    });

    const site = await loadSite(target, config);

    // panel=nav with no dock param
    window.location.hash = "#/page/?panel=nav";
    window.dispatchEvent(new PopStateEvent("popstate"));

    const panel = target.querySelector<HTMLElement>('[data-component-id="nav"]')!;
    expect(panel.style.display).not.toBe("none");

    site.dispose();
    document.body.removeChild(target);
    window.location.hash = "";
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test site.test.ts`
Expected: FAIL — restoreFromUrl doesn't handle panel field

- [ ] **Step 3: Add panel merge to restoreFromUrl**

In `packages/pages-runtime/src/site.ts`, modify `restoreFromUrl` to handle `link.panel`:

```typescript
if (link.dock || link.panel) {
  const panelOverrides = new Set(link.panel ?? []);
  if (link.dock) {
    for (const [id, state] of Object.entries(link.dock)) {
      const visible = panelOverrides.has(id) ? true : state === "open";
      const wasVisible = dockState.get(id);
      dockState.set(id, visible);
      if (visible !== wasVisible) {
        target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
          bubbles: true, composed: true,
          detail: { panelId: id, visible },
        }));
      }
    }
  }
  for (const key of panelOverrides) {
    if (!link.dock || !(key in link.dock)) {
      const wasVisible = dockState.get(key);
      dockState.set(key, true);
      if (wasVisible !== true) {
        target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
          bubbles: true, composed: true,
          detail: { panelId: key, visible: true },
        }));
      }
    }
  }
}
```

This replaces the existing `if (link.dock) { ... }` block from Task 2.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test site.test.ts`
Expected: PASS

- [ ] **Step 5: Run the full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/site.ts packages/pages-runtime/src/site.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: wire panel= deep link through restoreFromUrl

restoreFromUrl merges panel overrides into dock state before
dispatching pages-dock-toggle events. panel= entries override
dock= entries for the same key (activation intent wins over
saved state). Panels listed in panel= but not in dock= get
separate activation dispatches.

Refs #TBD"
```

---

## References

- [2026-08-27-panel-activation-hash-binding-design.md] — design spec this plan implements
- `packages/pages-runtime/src/site.ts:123` — LiveSite interface
- `packages/pages-runtime/src/site.ts:887-959` — dock-toggle handler
- `packages/pages-runtime/src/site.ts:319-354` — restoreFromUrl
- `packages/pages-runtime/src/site.ts:1252-1297` — initDockZoneGroup
- `packages/pages-runtime/src/site.ts:1370-1440` — site object literal
- `packages/pages-runtime/src/site.ts:1342-1359` — popstate handler
- `packages/pages-runtime/src/dock-bar-renderer.ts:48-89` — click handler with exclusivity
- `packages/pages-runtime/src/url.ts:19-77` — serializeToUrl
- `packages/pages-runtime/src/url.ts:79-176` — parseFromUrl
- `packages/pages-ui/src/model/page-types.ts:27-34` — DeepLink interface
- PP-20260810-72779a — workbench integration pattern
- PP-20260810-cdcc8f — content-agnostic workbench
- PP-20260705-bac842 — pages-event contract
- GE-20260625-fa01da — pushState/popstate separation
