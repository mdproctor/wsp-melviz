# Panel Activation & Hash Binding — Design Spec

**Date:** 2026-08-27
**Status:** Draft
**Scope:** `@casehubio/pages-runtime`, `@casehubio/pages-ui`

## 1. Overview

Workbench consumers (trellis, claudony) need two capabilities that casehub-pages does not yet expose:

1. **Programmatic dock panel activation** — call `site.activateDockPanel("backlog")` to show a specific dock panel without simulating a dock-bar button click.
2. **Hash-based panel deep linking** — `#/page/dashboard?panel=backlog,properties` activates specific dock panels on load and browser navigation.

Both features build on the existing `pages-dock-toggle` event system. No new rendering path or component type is introduced.

### Constraints

- ZoneLayoutEngine remains pure data — no DOM coupling (PP-20260810-72779a)
- Workbench components remain content-agnostic (PP-20260810-cdcc8f)
- Inter-component events follow the pages-event contract — `pages-dock-toggle` is the existing reserved event for dock panel visibility (PP-20260705-bac842)
- The popstate/pushState separation pattern (GE-20260625-fa01da) must be preserved — DOM navigation and URL sync are separate concerns

## 2. Pre-requisite A: Centralize dock-toggle exclusivity

### Problem

Zone-group exclusivity is currently enforced in the **dock-bar-renderer click handler** (`dock-bar-renderer.ts:50-85`), not in the `pages-dock-toggle` event handler in `site.ts`. The click handler iterates sibling buttons, dispatches hide events for active siblings, then dispatches show for the clicked panel.

Any code that dispatches `pages-dock-toggle` directly — `initDockZoneGroup`, `pages-dock-rearrange` handler, and the proposed `activateDockPanel` — bypasses exclusivity. Both panels become visible simultaneously, and dock-bar button `data-active` attributes diverge from actual panel state.

### Fix

Move exclusivity and button-state management into the `pages-dock-toggle` handler in `site.ts`. The handler already receives `{ panelId, visible }` and has access to the DOM tree. When `visible: true`:

1. Find the panel's zone from its DOM context (`data-dock-zone` attribute on the nearest dock-bar button, or from `zoneEngine.zoneMap` if available)
2. Find all other panel IDs in the same zone group
3. For each active sibling: set `display: none`, update `dockState`, update the sibling's dock-bar button `data-active`
4. Then show the target panel, update its button state

The dock-bar-renderer click handler becomes a thin dispatcher — it fires `pages-dock-toggle` and nothing else.

**Files changed:**
- `packages/pages-runtime/src/site.ts` — dock-toggle handler gains exclusivity logic
- `packages/pages-component/src/renderers/dock-bar-renderer.ts` — click handler simplified to dispatch only

This is a prerequisite refactoring. Both `activateDockPanel` and the popstate fix depend on the dock-toggle handler being the single owner of toggle semantics.

## 3. Pre-requisite B: Fix popstate dock state restoration

### Problem

`restoreFromUrl` in `site.ts` (line 349) updates the `dockState` map on popstate but never dispatches `pages-dock-toggle` events. Dock panels don't actually show or hide on browser back/forward navigation.

### Fix

After updating `dockState` entries in `restoreFromUrl`, dispatch `pages-dock-toggle` for each panel whose visibility changed. With exclusivity centralized in the dock-toggle handler (§2), these dispatches correctly enforce zone-group exclusivity.

**File:** `packages/pages-runtime/src/site.ts` — `restoreFromUrl` function (line 319)

```typescript
// Current code (line 349-352):
if (link.dock) {
  for (const [id, state] of Object.entries(link.dock)) {
    dockState.set(id, state === "open");
  }
}

// Proposed — also accepts panel overrides to avoid redundant dispatch:
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
  // Activate panels not in dock= but in panel=
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

This merges `panel` overrides into dock state before dispatching, avoiding redundant hide→show churn when the same key appears in both `dock=` and `panel=`.

Note: on first load when `dockState` is empty, `wasVisible` is `undefined`, so all entries dispatch. This is correct — the first load needs to establish initial DOM state.

## 4. Feature 1: `activateDockPanel` — LiveSite method

### API

Add `activateDockPanel` to the `LiveSite` interface:

**File:** `packages/pages-runtime/src/site.ts` (line 123)

```typescript
export interface LiveSite extends Site {
  navigate(path: string): void;
  setTheme(mode: "light" | "dark"): void;
  dispose(): void;
  readonly layout: LayoutState;
  activateDockPanel(key: string): boolean;  // new
}
```

### Semantics

- **Show, not toggle.** `activateDockPanel("backlog")` always makes the panel visible. To hide, consumers dispatch `pages-dock-toggle` with `visible: false` directly (existing mechanism).
- **Exclusive mode.** The dock-toggle handler enforces zone-group exclusivity (after §2 refactoring) — showing panel B in the same zone automatically hides panel A and updates dock-bar button state.
- **Deferred rendering.** Panels wrapped in `deferred` components render on first activation via the existing `pages-deferred-render` dispatch in the dock-toggle handler. No new logic needed.
- **URL sync.** The dock-toggle handler calls `syncUrl("replaceState")` after toggling. The URL updates automatically.
- **Unknown key.** Returns `false` if the panel element is not found in the DOM. No error thrown.

### Implementation

A closure method inside `loadSite()`, with direct access to `target` and `dockState`. Panel existence is validated via DOM lookup, not `zoneEngine` — this works for both `dockWorkbench()` compositions and manual `dockBar()` + `split()` compositions where no zone engine exists.

**File:** `packages/pages-runtime/src/site.ts` — inside `loadSite()` closure

```typescript
function activateDockPanel(key: string): boolean {
  const panelEl = target.querySelector<HTMLElement>(
    `[data-component-id="${CSS.escape(key)}"]`
  );
  if (!panelEl) return false;
  target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
    bubbles: true, composed: true,
    detail: { panelId: key, visible: true },
  }));
  return true;
}
```

Add to the returned `site` object literal (site.ts ~line 1370):

```typescript
const site: LiveSite = {
  root, page(...), dataset(...), state,
  setTheme(...), navigate(...), get layout(), dispose(),
  activateDockPanel,  // new
};
```

### Consumer usage

```typescript
import { loadSite } from "@casehubio/pages-runtime";

const site = await loadSite(container, config, options);

// Programmatic activation
site.activateDockPanel("backlog");

// Check if panel exists
if (!site.activateDockPanel("unknown-key")) {
  console.warn("Panel not found");
}
```

### Re-export

`activateDockPanel` is a method on `LiveSite`, not a standalone export. The `LiveSite` type is already exported from `packages/pages-runtime/src/index.ts`. No new export needed — consumers access it via the `loadSite()` return value.

## 5. Feature 2: Hash panel binding

### DeepLink extension

Add `panel` field to the `DeepLink` interface:

**File:** `packages/pages-ui/src/model/page-types.ts` (line 27)

```typescript
export interface DeepLink {
  readonly page: string;
  readonly filters?: Readonly<Record<string, readonly string[]>>;
  readonly sort?: Readonly<Record<string, { readonly columnId: string; readonly order: SortOrder }>>;
  readonly pagination?: Readonly<Record<string, number>>;
  readonly textFilter?: Readonly<Record<string, string>>;
  readonly dock?: Readonly<Record<string, "open" | "closed">>;
  readonly panel?: readonly string[];  // new — activation intent
}
```

### Semantic distinction: `panel` vs `dock`

| Aspect | `dock=` | `panel=` |
|--------|---------|----------|
| Semantics | Total state declaration — all panels' open/closed | Activation intent — "show these panels" |
| Format | `dock=nav:open,files:closed,props:closed` | `panel=nav,props` |
| Resilience | Fragile to panel additions (unlisted panels have no declared state) | Additive — works even if new panels are added to the workbench |
| Use case | Internal URL state sync (generated by the runtime) | Human-authored deep links |
| Lifecycle | Bidirectional — produced by `syncUrl`, consumed by `parseFromUrl` | **Consume-only** — parsed from URL on load/popstate, converted to dock state. `syncUrl` never produces `panel=`. After first load, the `panel=` param is consumed and the resulting state is reflected in `dock=`. |

When both are present for the same panel key, `panel` wins — activation intent overrides saved state. The merge happens inside `restoreFromUrl` (§3) so only one `pages-dock-toggle` event fires per panel, not a redundant hide→show pair.

### URL serialization

**File:** `packages/pages-runtime/src/url.ts` — `serializeToUrl` (line 19)

Two changes:

**5a. Panel param serialization** — add after the existing `dock` block:

```typescript
if (link.panel && link.panel.length > 0) {
  params.push(`panel=${link.panel.map(encodeURIComponent).join(",")}`);
}
```

**5b. Empty page path** — when `page` is empty and params exist, use `#?` instead of `#/page/`:

```typescript
if (link.page) {
  url = `#/page/${encodePagePath(link.page)}`;
  if (params.length > 0) url += `?${params.join("&")}`;
} else if (params.length > 0) {
  url = `#?${params.join("&")}`;
} else {
  url = "";
}
```

This ensures `#?panel=backlog` round-trips correctly: parse → serialize → same URL.

### URL parsing

**File:** `packages/pages-runtime/src/url.ts` — `parseFromUrl` (line 79)

Two changes:

**5c. Handle hashes without `#/page/` prefix.** Extend to parse query params from `#?params` format:

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

  // ... existing query param parsing (filters, sort, pagination, textFilter, dock) ...

  // New: parse panel param
  let panel: string[] | undefined;
  const panelStr = params.get("panel");
  if (panelStr) {
    panel = panelStr.split(",").map(decodeURIComponent).filter(Boolean);
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

### Activation on load and popstate

**File:** `packages/pages-runtime/src/site.ts`

**Initial load** — panel activation uses the `deepLink` returned by `restoreFromUrl` at line 1453. `restoreFromUrl` already handles the `panel` merge with dock state (§3), so panel activation is implicit — no separate loop needed at the initial load site. The `restoreFromUrl` call dispatches `pages-dock-toggle` for each panel entry, and the centralized dock-toggle handler (§2) enforces exclusivity.

```typescript
// Existing code at line 1453-1455:
const deepLink = restoreFromUrl(location.hash, filterState, componentViewState);
if (deepLink.page) { navigateInternal(deepLink.page); }
syncUrl("replaceState");
```

No change needed here — `restoreFromUrl` now handles `link.panel` internally (§3).

**Popstate handler** (line 1342) — similarly, `restoreFromUrl` handles panel merge:

```typescript
window.addEventListener("popstate", () => {
  const link = parseFromUrl(location.hash);
  if (link.page !== currentPage) navigateInternal(link.page);
  clearPageFilters(filterState, currentPage);
  componentViewState.clear();
  restoreFromUrl(location.hash, filterState, componentViewState);
  // No separate panel loop — restoreFromUrl handles panel= merge
  pipeline.deliverAll();
}, { signal: abortController.signal });
```

### Multi-panel and zone exclusivity

Processing order is left-to-right in the comma-separated list. If two keys are in the same exclusive zone, the last one wins — this is inherent to exclusive dock bar semantics (enforced by the dock-toggle handler after §2).

Example: `?panel=backlog,cases` where both are in `left-top`:
1. `pages-dock-toggle { panelId: "backlog", visible: true }` → backlog shown
2. `pages-dock-toggle { panelId: "cases", visible: true }` → handler hides backlog (exclusivity), shows cases

Deep link authors should only list one panel per zone.

## 6. Files changed

| File | Change |
|------|--------|
| `packages/pages-ui/src/model/page-types.ts` | Add `panel?: readonly string[]` to `DeepLink` |
| `packages/pages-runtime/src/url.ts` | Extend `parseFromUrl` for `#?` prefix and `panel` param; extend `serializeToUrl` for `panel` and empty-page format |
| `packages/pages-runtime/src/site.ts` | Centralize dock-toggle exclusivity in handler; add `activateDockPanel` to `LiveSite` interface and `loadSite()` closure; fix `restoreFromUrl` dock dispatch with panel merge; add `activateDockPanel` to site object literal |
| `packages/pages-component/src/renderers/dock-bar-renderer.ts` | Simplify click handler to dispatch only (exclusivity moved to site.ts) |

No new files. No new packages. No new component types.

## 7. Testing strategy

### Unit tests

- `url.test.ts`: `parseFromUrl` handles `#/page/dashboard?panel=backlog`, `#/page/dashboard?panel=backlog,properties`, `#?panel=backlog` (panel-only), `#/page/dashboard?dock=nav:open&panel=backlog` (both params), empty `panel=`, unknown keys in panel list
- `url.test.ts`: `serializeToUrl` round-trips panel entries — including `#?panel=backlog` for empty page
- `url.test.ts`: round-trip symmetry — `parseFromUrl(serializeToUrl(link))` preserves `panel` field
- `zone-layout-engine.test.ts`: No changes needed — engine stays pure data

### Integration tests (Playwright)

- `activateDockPanel` shows the panel and hides others in the same zone (exclusivity)
- `activateDockPanel` with unknown key returns false and doesn't throw
- `activateDockPanel` works for manual dock-bar compositions (no zone engine)
- Dock-bar button `data-active` state stays in sync after programmatic activation
- Deep link with `panel=backlog` opens the backlog panel on page load
- Deep link with `panel=backlog,properties` opens panels in different zones
- Browser back/forward correctly toggles dock panels (pre-requisite bug fix)
- `panel=` overrides `dock=` for the same key — no redundant hide→show
- Panel-only deep link `#?panel=backlog` works (no page path)

## References

- `packages/pages-runtime/src/site.ts` — LiveSite interface (line 123), loadSite closure, dock-toggle handler (~line 887), restoreFromUrl (line 319), initDockZoneGroup (line 1252), popstate handler (line 1342), site object literal (~line 1370)
- `packages/pages-runtime/src/url.ts` — serializeToUrl (line 19), parseFromUrl (line 79)
- `packages/pages-runtime/src/zone-layout-engine.ts` — ZoneLayoutEngine interface, zoneMap
- `packages/pages-ui/src/model/page-types.ts` — DeepLink interface (line 27)
- `packages/pages-component/src/renderers/dock-bar-renderer.ts` — dock-bar click handler with exclusivity logic (line 50)
- PP-20260810-72779a — workbench integration pattern (extend existing packages)
- PP-20260810-cdcc8f — content-agnostic workbench
- PP-20260705-bac842 — pages-event contract (pages-dock-toggle reserved)
- GE-20260625-fa01da — pushState/popstate separation pattern
- GE-20260809-aee002 — resolveDockZone zone naming
- GE-20260804-befd45 — dockWorkbench decomposition into 3 primitives
