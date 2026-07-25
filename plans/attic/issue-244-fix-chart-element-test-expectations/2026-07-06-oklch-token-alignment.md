# OKLCH Token Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #124 — chore: align examples and auth widgets with OKLCH token system, remove superseded code
**Issue group:** #124

**Goal:** Migrate the examples gallery, auth widgets, and PagesBadge from
hardcoded hex colours/px values to the OKLCH design token system, making
them the reference implementation for downstream consumers.

**Architecture:** Fresh `styles.css` rewrite (port, not patch) on the token
system. Theme injection via `casehub-entry.ts` (webpack bundle entry point).
Auth widgets use CSS custom property inheritance through Shadow DOM (same
pattern as `PagesElement`). PagesBadge palette replaced with semantic scale
token references.

**Tech Stack:** TypeScript, CSS, Webpack 5, `@casehubio/pages-ui-tokens`

## Global Constraints

- All colours use `var(--pages-*)` token references, not hex/rgb literals (code panel excepted)
- Spacing uses `var(--pages-space-*)` scale; values between steps round to nearest token
- Sizing values (`width`, `height`) that aren't spacing stay as literal px
- Display-size typography exceeding the scale (e.g., 36px) stays literal
- Code panel keeps hardcoded dark colours — its colour system is theme-independent
- No gallery layout/structural changes — same selectors, same JS logic
- `injectTheme()` must be called in `casehub-entry.ts`, not `app.js` (vanilla JS can't import npm packages)

## Pre-implementation reading

- `docs/protocols/casehub/css-design-tokens.md` — token naming, OKLCH scales, 11 categories
- `docs/specs/2026-07-06-oklch-token-alignment-design.md` — the reviewed spec

---

### Task 1: Fresh gallery `styles.css` + theme injection + toggles

**Files:**
- Create: `examples/src/styles.css` (overwrite existing 445-line file)
- Modify: `examples/src/casehub-entry.ts` (add theme injection + toggle exports)
- Modify: `examples/src/app.js` (add toggle button handlers)
- Modify: `examples/src/index.html` (add toggle buttons to header, add `pages-theme-light` class to `<html>`)
- Modify: `examples/package.json` (add `@casehubio/pages-ui-tokens` dependency)

**Interfaces:**
- Consumes: `injectTheme()`, `applyThemeMode()`, `DEFAULT_THEME` from `@casehubio/pages-ui-tokens`
- Produces: tokenised gallery shell; theme/density toggles available to user

- [ ] **Step 1: Add `@casehubio/pages-ui-tokens` dependency**

In `examples/package.json`, add to `dependencies`:

```json
"@casehubio/pages-ui-tokens": "workspace:*"
```

Run `yarn install` from the monorepo root.

- [ ] **Step 2: Update `casehub-entry.ts` with theme injection**

Replace the full content of `examples/src/casehub-entry.ts`:

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import type { LiveSite, SiteOptions } from "@casehubio/pages-runtime";
import { injectTheme, applyThemeMode, DEFAULT_THEME } from "@casehubio/pages-ui-tokens";

injectTheme(DEFAULT_THEME);
applyThemeMode(document.documentElement, "light");

export { loadSite, injectTheme, applyThemeMode, DEFAULT_THEME };
export type { LiveSite, SiteOptions };
```

- [ ] **Step 3: Add `pages-theme-light` class to `<html>` and toggle buttons to header**

In `examples/src/index.html`:

Add class to `<html>`:
```html
<html lang="en" class="pages-theme-light">
```

Add toggle buttons inside `.header`, after the `.info` div:

```html
<div class="header">
    <h1>Melviz Examples</h1>
    <div class="info">
        <span id="sample-count">Loading...</span>
    </div>
    <div class="header-toggles">
        <button id="theme-toggle" class="toggle-btn" title="Toggle dark mode">🌙</button>
        <button id="density-toggle" class="toggle-btn" title="Toggle compact density">⊟</button>
    </div>
</div>
```

- [ ] **Step 4: Add toggle handlers to `app.js`**

At the end of the `setupEventListeners()` function in `examples/src/app.js`, add:

```javascript
// Theme toggle
const themeToggle = document.getElementById('theme-toggle');
let isDark = false;
themeToggle.addEventListener('click', () => {
    isDark = !isDark;
    const mode = isDark ? 'dark' : 'light';
    document.documentElement.classList.remove('pages-theme-light', 'pages-theme-dark');
    document.documentElement.classList.add(`pages-theme-${mode}`);
    if (currentSite) {
        currentSite.setTheme(mode);
    }
    themeToggle.textContent = isDark ? '☀️' : '🌙';
});

// Density toggle
const densityToggle = document.getElementById('density-toggle');
let isCompact = false;
densityToggle.addEventListener('click', () => {
    isCompact = !isCompact;
    document.documentElement.classList.toggle('pages-density-compact', isCompact);
    densityToggle.textContent = isCompact ? '⊞' : '⊟';
});
```

- [ ] **Step 5: Write the fresh `styles.css`**

Overwrite `examples/src/styles.css` with a complete new file. This is the
largest single step — every selector from the original is ported.

The full CSS follows. Each section maps to the original structure.

```css
/* ============================================================
   Melviz Examples Gallery — OKLCH Token System
   All values use var(--pages-*) tokens from pages-ui-tokens.
   Code panel is theme-independent (hardcoded dark).
   ============================================================ */

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: var(--pages-font-family);
    background: var(--pages-neutral-2);
    color: var(--pages-neutral-11);
}

.container {
    display: flex;
    height: 100vh;
    overflow: visible;
    position: relative;
}

/* --- Sidebar --- */

.sidebar {
    width: 320px;
    background: var(--pages-neutral-1);
    border-right: 1px solid var(--pages-neutral-4);
    display: flex;
    flex-direction: column;
    transition: margin-left var(--pages-duration-normal) var(--pages-ease-out);
    position: relative;
    overflow: hidden;
}

.sidebar.collapsed {
    margin-left: -320px;
}

.sidebar-toggle-btn {
    position: fixed;
    top: var(--pages-space-1-5);
    left: 295px;
    width: 20px;
    height: 20px;
    background: var(--pages-accent-9);
    border: none;
    border-radius: var(--pages-radius-sm);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: var(--pages-shadow-1);
    transition: all var(--pages-duration-normal) var(--pages-ease-out);
    z-index: 10000;
    color: white;
}

.sidebar-toggle-btn:hover {
    background: var(--pages-accent-10);
    box-shadow: var(--pages-shadow-2);
    transform: translateY(-2px);
}

.sidebar-toggle-btn:active {
    transform: translateY(0) scale(0.95);
}

.sidebar-toggle-btn.collapsed {
    left: var(--pages-space-1-5);
}

/* --- Header --- */

.header {
    padding: var(--pages-space-5);
    border-bottom: 1px solid var(--pages-neutral-4);
    background: var(--pages-accent-9);
    color: white;
    display: flex;
    flex-wrap: wrap;
    align-items: baseline;
    gap: var(--pages-space-2);
}

.header h1 {
    font-size: var(--pages-font-size-2xl);
    font-weight: var(--pages-font-weight-semibold);
    margin-bottom: var(--pages-space-2);
}

.info {
    font-size: var(--pages-font-size-base);
    opacity: 0.9;
}

.header-toggles {
    margin-left: auto;
    display: flex;
    gap: var(--pages-space-1-5);
}

.toggle-btn {
    background: rgba(255, 255, 255, 0.15);
    border: 1px solid rgba(255, 255, 255, 0.25);
    border-radius: var(--pages-radius-sm);
    color: white;
    cursor: pointer;
    padding: var(--pages-space-1) var(--pages-space-2);
    font-size: var(--pages-font-size-sm);
    transition: background var(--pages-duration-fast) var(--pages-ease-out);
}

.toggle-btn:hover {
    background: rgba(255, 255, 255, 0.25);
}

/* --- Search --- */

.search-box {
    padding: var(--pages-space-4);
    border-bottom: 1px solid var(--pages-neutral-4);
}

#search {
    width: 100%;
    padding: var(--pages-space-2-5) var(--pages-space-3);
    border: 1px solid var(--pages-neutral-4);
    border-radius: var(--pages-radius-md);
    font-size: var(--pages-font-size-base);
    font-family: var(--pages-font-family);
    color: var(--pages-neutral-12);
    background: var(--pages-neutral-1);
    outline: none;
    transition: border-color var(--pages-duration-fast) var(--pages-ease-out);
}

#search:focus {
    border-color: var(--pages-accent-9);
}

/* --- Categories nav --- */

.categories {
    flex: 1;
    overflow-y: auto;
    padding: var(--pages-space-2) 0;
}

.category {
    margin-bottom: var(--pages-space-1);
}

.category-header {
    padding: var(--pages-space-3) var(--pages-space-5);
    font-weight: var(--pages-font-weight-semibold);
    font-size: var(--pages-font-size-sm);
    text-transform: uppercase;
    color: var(--pages-neutral-9);
    letter-spacing: 0.5px;
    cursor: pointer;
    user-select: none;
    display: flex;
    align-items: center;
    justify-content: space-between;
    transition: background-color var(--pages-duration-fast) var(--pages-ease-out);
}

.category-header:hover {
    background: var(--pages-neutral-2);
}

.category-toggle {
    transition: transform var(--pages-duration-fast) var(--pages-ease-out);
}

.category.collapsed .category-toggle {
    transform: rotate(-90deg);
}

.category-items {
    display: block;
}

.category.collapsed .category-items {
    display: none;
}

.sample-item {
    padding: var(--pages-space-2-5) var(--pages-space-5) var(--pages-space-2-5) var(--pages-space-8);
    cursor: pointer;
    font-size: var(--pages-font-size-base);
    color: var(--pages-neutral-10);
    transition: all var(--pages-duration-fast) var(--pages-ease-out);
    border-left: 3px solid transparent;
}

.sample-item:hover {
    background: var(--pages-neutral-2);
    color: var(--pages-accent-9);
}

.sample-item.active {
    background: var(--pages-accent-3);
    color: var(--pages-accent-9);
    border-left-color: var(--pages-accent-9);
    font-weight: var(--pages-font-weight-medium);
}

.sample-item.hidden {
    display: none;
}

/* --- Main content --- */

.main-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    position: relative;
    z-index: 1;
}

.welcome {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: var(--pages-space-10);
    text-align: center;
    min-width: 0;
}

.welcome h1 {
    font-size: 32px;
    font-weight: var(--pages-font-weight-semibold);
    color: var(--pages-neutral-12);
    margin-bottom: var(--pages-space-4);
}

.welcome p {
    font-size: var(--pages-font-size-lg);
    color: var(--pages-neutral-9);
    margin-bottom: var(--pages-space-8);
}

/* --- Stats cards --- */

.stats {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: var(--pages-space-5);
    max-width: 800px;
    width: 100%;
    margin-top: var(--pages-space-5);
}

.stat-card {
    background: var(--pages-neutral-1);
    padding: var(--pages-space-6);
    border-radius: var(--pages-radius-lg);
    border: 1px solid var(--pages-neutral-4);
    box-shadow: var(--pages-shadow-1);
    text-align: left;
}

.stat-card h3 {
    font-size: var(--pages-font-size-sm);
    font-weight: var(--pages-font-weight-semibold);
    color: var(--pages-neutral-9);
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: var(--pages-space-2);
}

.stat-card .value {
    font-size: 36px;
    font-weight: 700;
    color: var(--pages-accent-9);
}

/* --- Sample container --- */

.sample-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    background: var(--pages-neutral-1);
    min-width: 0;
    min-height: 0;
}

.sample-content-with-code {
    flex: 1;
    display: flex;
    flex-direction: row;
    min-height: 0;
    overflow: hidden;
}

.sample-content-wrapper {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    min-width: 0;
}

.sample-header {
    padding: var(--pages-space-4) var(--pages-space-6);
    border-bottom: 1px solid var(--pages-neutral-4);
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.sample-header h2 {
    font-size: var(--pages-font-size-xl);
    font-weight: var(--pages-font-weight-semibold);
    color: var(--pages-neutral-12);
}

.sample-actions {
    display: flex;
    gap: var(--pages-space-2);
}

.sample-actions button {
    padding: var(--pages-space-2) var(--pages-space-3);
    border: 1px solid var(--pages-neutral-4);
    background: var(--pages-neutral-1);
    border-radius: var(--pages-radius-md);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--pages-neutral-10);
    transition: all var(--pages-duration-fast) var(--pages-ease-out);
}

.sample-actions button:hover {
    background: var(--pages-neutral-2);
    border-color: var(--pages-accent-9);
    color: var(--pages-accent-9);
}

#sample-target {
    flex: 1;
    overflow: auto;
    padding: var(--pages-space-4);
    min-width: 0;
    min-height: 0;
}

/* --- Scrollbar --- */

.categories::-webkit-scrollbar {
    width: 8px;
}

.categories::-webkit-scrollbar-track {
    background: var(--pages-neutral-2);
}

.categories::-webkit-scrollbar-thumb {
    background: var(--pages-neutral-5);
    border-radius: var(--pages-radius-sm);
}

.categories::-webkit-scrollbar-thumb:hover {
    background: var(--pages-neutral-8);
}

/* --- Code sidebar (theme-independent) --- */

.code-sidebar {
    flex: 0 0 35%;
    background: #1e1e1e;
    border-left: 1px solid var(--pages-neutral-6);
    transition: flex-basis var(--pages-duration-normal) var(--pages-ease-out);
    overflow-y: auto;
    overflow-x: auto;
    min-height: 0;
}

.code-sidebar.collapsed {
    flex-basis: 0;
    overflow: hidden;
    border-left: none;
}

.code-content {
    padding: 0;
}

.code-content pre {
    margin: 0;
    padding: var(--pages-space-1-5);
    background: #1e1e1e;
    color: #d4d4d4;
    font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Consolas', 'source-code-pro', monospace;
    font-size: 11px;
    line-height: 1.5;
}

.code-content code {
    display: block;
    white-space: pre;
}

.code-sidebar::-webkit-scrollbar {
    width: 8px;
}

.code-sidebar::-webkit-scrollbar-track {
    background: #252526;
}

.code-sidebar::-webkit-scrollbar-thumb {
    background: #555;
    border-radius: var(--pages-radius-sm);
}

.code-sidebar::-webkit-scrollbar-thumb:hover {
    background: #777;
}

/* --- Config bar --- */

.config-bar {
    display: flex;
    flex-wrap: wrap;
    gap: var(--pages-space-3);
    align-items: end;
    padding: var(--pages-space-2-5) var(--pages-space-4);
    margin: 0 0 var(--pages-space-3);
    background: var(--pages-neutral-2);
    border: 1px solid var(--pages-neutral-4);
    border-radius: var(--pages-radius-md);
    font-size: var(--pages-font-size-sm);
}

.config-bar .config-field {
    display: flex;
    flex-direction: column;
    gap: var(--pages-space-1);
    flex: 1;
    min-width: 200px;
}

.config-bar label {
    font-weight: var(--pages-font-weight-semibold);
    color: var(--pages-neutral-9);
    font-size: var(--pages-font-size-xs);
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

.config-bar input {
    padding: var(--pages-space-1-5) var(--pages-space-2-5);
    border: 1px solid var(--pages-neutral-4);
    border-radius: var(--pages-radius-md);
    font-size: var(--pages-font-size-sm);
    font-family: monospace;
    color: var(--pages-neutral-12);
    background: var(--pages-neutral-1);
    transition: border-color var(--pages-duration-fast) var(--pages-ease-out);
}

.config-bar input:focus {
    outline: none;
    border-color: var(--pages-accent-9);
    box-shadow: 0 0 0 2px var(--pages-accent-3);
}

.config-bar input::placeholder {
    color: var(--pages-neutral-8);
}

.config-bar .config-apply {
    padding: var(--pages-space-1-5) var(--pages-space-4);
    background: var(--pages-accent-9);
    color: white;
    border: none;
    border-radius: var(--pages-radius-md);
    font-size: var(--pages-font-size-sm);
    font-weight: var(--pages-font-weight-medium);
    cursor: pointer;
    transition: background var(--pages-duration-fast) var(--pages-ease-out);
    white-space: nowrap;
}

.config-bar .config-apply:hover {
    background: var(--pages-accent-10);
}

.config-bar .config-status {
    font-size: var(--pages-font-size-xs);
    color: var(--pages-neutral-8);
    align-self: center;
}
```

- [ ] **Step 6: Run `yarn install` and build**

```bash
cd /Users/mdproctor/claude/casehub/pages && yarn install && yarn build:packages && yarn build:examples
```

Expected: BUILD SUCCESS — webpack bundles the gallery with token injection.

- [ ] **Step 7: Serve and visually verify**

```bash
yarn workspace @casehub/pages-examples run serve
```

Open `http://localhost:8080`. Check:
- Gallery shell uses token colours (not purple gradient)
- Sidebar items highlight correctly
- Theme toggle switches light/dark
- Density toggle tightens spacing
- Click through several dashboards in each category
- Code panel remains dark in both themes

- [ ] **Step 8: Run Playwright smoke tests**

```bash
yarn workspace @casehub/pages-examples run test
```

Expected: all smoke tests pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: migrate examples gallery to OKLCH token system

Replace 75+ hardcoded hex colours and raw px values with
var(--pages-*) token references. Add theme (light/dark) and
density (comfortable/compact) toggles. Wire injectTheme() via
casehub-entry.ts bundle entry point.

Refs #124"
```

---

### Task 2: Auth widget token migration

**Files:**
- Modify: `packages/pages-ui/src/auth/identity-widget.ts`
- Modify: `packages/pages-ui/src/auth/dev-auth-gate.ts`

**Interfaces:**
- Consumes: CSS custom properties inherited from ancestor with `injectTheme()`
- Produces: token-based auth widgets compatible with light/dark themes

- [ ] **Step 1: Migrate `identity-widget.ts` inline styles**

In `packages/pages-ui/src/auth/identity-widget.ts`, replace the `<style>` block
inside the `render()` method (lines 39-79) with token references:

```typescript
this.shadowRoot.innerHTML = `
  <style>
    .identity-display {
      display: inline-block;
      padding: var(--pages-space-2, 0.5rem) var(--pages-space-4, 1rem);
      background: var(--pages-neutral-2, #f0f0f0);
      border-radius: var(--pages-radius-sm, 4px);
      cursor: pointer;
      user-select: none;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      color: var(--pages-neutral-12, #333);
    }
    .identity-display:hover {
      background: var(--pages-neutral-4, #e0e0e0);
    }
    .picker-popover {
      position: absolute;
      background: var(--pages-neutral-1, white);
      border: 1px solid var(--pages-neutral-5, #ccc);
      border-radius: var(--pages-radius-sm, 4px);
      padding: var(--pages-space-4, 1rem);
      box-shadow: var(--pages-shadow-2, 0 2px 8px rgba(0,0,0,0.2));
      z-index: 1000;
      min-width: 200px;
    }
    .picker-popover select,
    .picker-popover input[type="text"] {
      width: 100%;
      padding: var(--pages-space-2, 0.5rem);
      margin-bottom: var(--pages-space-2, 0.5rem);
      box-sizing: border-box;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      border: 1px solid var(--pages-neutral-4, #ddd);
      border-radius: var(--pages-radius-sm, 4px);
    }
    .picker-popover button {
      width: 100%;
      padding: var(--pages-space-2, 0.5rem) var(--pages-space-4, 1rem);
      background: var(--pages-accent-9, #007bff);
      color: white;
      border: none;
      border-radius: var(--pages-radius-sm, 4px);
      cursor: pointer;
      font-family: var(--pages-font-family, system-ui, sans-serif);
    }
    .picker-popover button:hover {
      background: var(--pages-accent-10, #0056b3);
    }
  </style>
  <div class="identity-display" id="identity-display"></div>
  ${this.pickerVisible ? this.renderPicker() : ""}
`;
```

- [ ] **Step 2: Migrate `dev-auth-gate.ts` inline styles**

In `packages/pages-ui/src/auth/dev-auth-gate.ts`, replace the `<style>` block
inside `renderOverlay()` (lines 93-136) with token references:

```typescript
this.shadowRoot.innerHTML = `
  <style>
    .overlay {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: oklch(0% 0 0 / 0.5);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 10000;
    }
    .dialog {
      background: var(--pages-neutral-1, white);
      padding: var(--pages-space-8, 2rem);
      border-radius: var(--pages-radius-lg, 8px);
      box-shadow: var(--pages-shadow-3, 0 4px 12px rgba(0,0,0,0.3));
      min-width: 300px;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      color: var(--pages-neutral-12, #333);
    }
    .dialog h2 {
      margin-top: 0;
      font-weight: var(--pages-font-weight-semibold, 600);
    }
    .dialog select,
    .dialog input[type="text"] {
      width: 100%;
      padding: var(--pages-space-2, 0.5rem);
      margin-bottom: var(--pages-space-4, 1rem);
      box-sizing: border-box;
      border: 1px solid var(--pages-neutral-4, #ddd);
      border-radius: var(--pages-radius-sm, 4px);
    }
    .dialog button {
      width: 100%;
      padding: var(--pages-space-2, 0.5rem) var(--pages-space-4, 1rem);
      background: var(--pages-accent-9, #007bff);
      color: white;
      border: none;
      border-radius: var(--pages-radius-sm, 4px);
      cursor: pointer;
    }
    .dialog button:hover {
      background: var(--pages-accent-10, #0056b3);
    }
  </style>
  <div class="overlay">
    <div class="dialog">
      <h2>Login Required</h2>
      <div id="identity-container"></div>
      <button id="login-btn">Login</button>
    </div>
  </div>
`;
```

- [ ] **Step 3: Run existing tests**

```bash
yarn workspace @casehubio/pages-ui run test
```

Expected: all tests pass (auth widget tests verify behaviour, not styling).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui/src/auth/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: migrate auth widgets to OKLCH token system

Replace hardcoded hex colours with var(--pages-*, fallback) references.
Tokens inherit through Shadow DOM — same pattern as PagesElement.

Refs #124"
```

---

### Task 3: PagesBadge palette replacement

**Files:**
- Modify: `packages/pages-viz/src/components/PagesBadge.ts` (lines 36-46)

**Interfaces:**
- Consumes: CSS custom properties from ancestor theme injection
- Produces: badge colours that adapt to theme changes

- [ ] **Step 1: Replace `DEFAULT_PALETTE` with token references**

In `packages/pages-viz/src/components/PagesBadge.ts`, replace lines 36-46:

```typescript
const DEFAULT_PALETTE = [
  "var(--pages-accent-9)",
  "var(--pages-success-9)",
  "var(--pages-warning-9)",
  "var(--pages-danger-9)",
  "var(--pages-info-9)",
  "var(--pages-accent-11)",
  "var(--pages-success-11)",
  "var(--pages-warning-11)",
  "var(--pages-danger-11)",
];
```

- [ ] **Step 2: Run existing tests**

```bash
yarn workspace @casehubio/pages-viz run test
```

Expected: all tests pass (badge tests verify rendering logic, not specific colours).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/components/PagesBadge.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: replace PagesBadge hardcoded palette with token-derived colours

Step 9 and step 11 from semantic OKLCH scales. Both pass WCAG AA
contrast with white badge text.

Closes #124"
```

---

## Verification

After all tasks:

1. **Build succeeds:** `yarn build` from monorepo root
2. **All tests pass:** `yarn test` across all packages
3. **Gallery visual verification:** serve examples, walk through dashboards in both themes
4. **No hardcoded hex in migrated files:** confirm `styles.css`, auth widgets, and PagesBadge have zero `#` colour literals (code panel excepted)
5. **Theme toggle works:** light/dark switch applies to both gallery shell and rendered dashboards
6. **Density toggle works:** compact mode tightens spacing
