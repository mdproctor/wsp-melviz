# Compact Theme Picker with Flyout Popover

**Issue:** casehubio/casehub-pages#245
**Date:** 2026-07-30
**Status:** Approved

## Context

`PagesThemePickerElement` in `packages/pages-ui-tokens/src/theme-picker.ts` has a `compact` boolean property. The current compact mode just renders inline sun/moon buttons without the family dropdown. For space-constrained UIs (toolbars, embedded dashboards), a proper compact mode is needed: a single icon button that opens a flyout popover with theme family selection and light/dark toggle.

## Approach

**Native Popover API + CSS Anchor Positioning** — the 2026 web platform provides both as Baseline standards. The Popover API gives light-dismiss, Escape-to-close, `aria-expanded`, top-layer rendering, and tab-order repositioning with zero JavaScript. CSS Anchor Positioning handles placement and viewport flipping declaratively.

No new dependencies. No `FocusTrapMixin`. No click-outside handlers. The component stays in `pages-ui-tokens` with only `lit` as a dependency.

## Architecture

Everything stays in `PagesThemePickerElement` — no new components. When `compact` is true, `_renderCompact()` renders:

1. A trigger `<button>` with a palette SVG icon (16×16) and a 6px colour dot showing the current accent (`--pages-accent-9`)
2. A `<div popover="auto">` containing the theme selection UI

Both elements live in the same shadow root, so `popovertarget` ID references work within the shadow tree. The full (non-compact) render path is unchanged.

## Popover Content

The popover adapts based on registered theme families:

**Multi-family (2+ families):**
```
┌─────────────────────────┐
│  Theme                  │
│  ○ Default              │
│  ● CaseHub              │
│                         │
│  Mode                   │
│  [☀ Light] [☾ Dark]     │
└─────────────────────────┘
```

- Families as native `<input type="radio">` in a `<fieldset>` with `<legend>`
- Active family pre-selected
- Selection applies theme immediately; popover stays open

**Single-family (1 family):**
```
┌─────────────────────────┐
│  [☀ Light] [☾ Dark]     │
└─────────────────────────┘
```

- Family section omitted — just the mode toggle
- Smaller, tighter popover

**Mode toggle** is a segmented button pair with sun/moon SVG icons. Active mode gets `--pages-interactive` highlight. Same pattern as the existing full-mode toggle.

**Dismissal:** Popover stays open on selection. Closes only on click-outside (light-dismiss) or Escape. This is the default `popover="auto"` behaviour.

## CSS & Positioning

```css
button[part="trigger"] {
  anchor-name: --theme-picker-trigger;
}

[popover] {
  position-anchor: --theme-picker-trigger;
  position-area: block-end span-inline-end;
  position-try-fallbacks: flip-block;
  margin: 0;
  margin-block-start: var(--pages-space-1, 4px);
}
```

- Popover appears below the trigger, aligned to its start edge
- Flips above if no room below
- All styling uses `--pages-` design tokens: `--pages-surface-primary` background, `--pages-border-default` border, `--pages-shadow-2` elevation, `--pages-radius-md` corners
- Content-sized with `min-width` to prevent collapse
- `anchor-scope: all` on host prevents collisions with multiple picker instances

### Browser compatibility

- Popover API: Baseline Widely Available (April 2025)
- CSS Anchor Positioning: Baseline 2026, ~91% support
- Safari 18.2-18.3: supports `anchor()` but not `@position-try` — base positioning works, viewport flipping doesn't on those versions. Acceptable tradeoff.

## Accessibility & Keyboard

**Browser-provided (Popover API):**
- `aria-expanded` on trigger button (auto-toggled)
- Escape closes popover, returns focus to trigger
- Light-dismiss on click outside
- Popover content in tab order after trigger

**Explicitly added:**
- Trigger: `aria-haspopup="dialog"`, `aria-label="Theme settings"`
- Popover: `role="group"`, `aria-label="Theme settings"`
- Family radios: native `<input type="radio">` in `<fieldset>` + `<legend>` — no extra ARIA needed
- Mode buttons: `aria-pressed` (existing pattern)

**Keyboard flow:**
- Enter/Space on trigger → opens popover, focus to first interactive element
- Tab/Shift+Tab → moves between radios and mode buttons
- Arrow keys → navigate within radio group (native behaviour)
- Escape → close, focus returns to trigger

No focus trapping — the popover is non-modal, focus can leave naturally (light-dismiss closes it).

## Testing

**Unit tests** (vitest + jsdom, in `theme-picker.test.ts`):

- Compact renders trigger button with palette icon and colour dot
- Click trigger opens popover (mocked — jsdom lacks Popover API)
- Multi-family: radio buttons for each family
- Single-family: family section omitted, only mode toggle
- Family selection calls `applyTheme()` with correct name
- Mode toggle calls `applyTheme()` with correct name
- Popover stays open after selection
- Colour dot reflects current accent
- ARIA attributes present on trigger

**jsdom Popover API mock** (garden GE-20260713-9e6bf5 — jsdom doesn't support `showPopover`):

```ts
const popover = picker.shadowRoot.querySelector('[popover]');
popover.showPopover = () => { popover.toggleAttribute('open', true); };
popover.hidePopover = () => { popover.removeAttribute('open'); };
```

**Manual verification:** Examples gallery dev server — confirm positioning, light-dismiss, keyboard flow, theme application.

## Scope

**In scope:**
- Compact popover trigger with palette icon + accent dot
- Popover with family radios + mode toggle
- Single-family simplification
- CSS anchor positioning with flip-block fallback
- Unit tests with jsdom popover mock

**Out of scope:**
- Configurable `position` attribute (add later if needed)
- System/auto mode detection
- Animation/transitions on open/close
- Changes to the full (non-compact) render path
