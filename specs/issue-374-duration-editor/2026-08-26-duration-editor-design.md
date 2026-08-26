# Design: pages-duration-input — ISO 8601 Duration Editor

**Issue:** casehubio/casehub-pages#374
**Branch:** `issue-374-duration-editor`
**Date:** 2026-08-26
**Parent:** casehubio/casehub-pages#373 (pages-property-palette)

## Overview

A new `<pages-duration-input>` standalone Lit control in `@casehubio/pages-ui-components` that edits ISO 8601 duration strings (`PT1H30M`, `P2DT4H5M6S`) via a multi-field number-input UI. The control shows hours, minutes, and seconds by default, with a configurable `fields` property for adding days, months, or years.

Integrates with the property palette via a `format: "duration"` mapping in the default resolver.

## Architecture

### Package Placement

New files in `packages/pages-ui-components/`:
- `src/duration-input/pages-duration-input.ts` — component
- `src/duration-input/index.ts` — barrel + registration
- `src/duration-input/pages-duration-input.test.ts` — tests

New sub-path export: `@casehubio/pages-ui-components/duration-input`

No new package — this is a leaf control in the existing `pages-ui-components` package, alongside `pages-date-input`, `pages-number-input`, and the other form primitives.

### Why Not a New Package

The duration editor follows the same contract as every other form control (value, label, disabled, readonly, error, change event). It belongs with its peers in `pages-ui-components` per ARC42STORIES §5 and the web-component-strategy protocol (PP-20260705-c7687d): Lit UI primitives live at the leaf level.

## Public API

```typescript
type DurationField = 'years' | 'months' | 'days' | 'hours' | 'minutes' | 'seconds';

export class PagesDurationInput extends LitElement {
  @property() value: string = '';
  @property({ attribute: false }) fields: DurationField[] = ['hours', 'minutes', 'seconds'];
  @property() label: string | undefined;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) readonly = false;
  @property({ type: Boolean }) disabled = false;
  @property() error: string | undefined;
}
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `value` | `string` | `''` | ISO 8601 duration string (e.g. `PT1H30M`) |
| `fields` | `DurationField[]` | `['hours', 'minutes', 'seconds']` | Which unit fields to display. Order in the array determines render order. |
| `label` | `string \| undefined` | `undefined` | Label text above the field group |
| `required` | `boolean` | `false` | Whether the field is required |
| `readonly` | `boolean` | `false` | Makes all sub-inputs non-editable (focusable, no opacity reduction) |
| `disabled` | `boolean` | `false` | Disables all sub-inputs (not focusable, dimmed) |
| `error` | `string \| undefined` | `undefined` | Error message displayed below the field group |

### Events

| Event | When | Detail |
|-------|------|--------|
| `change` | Any sub-input loses focus after value change | None — read `value` property for the new ISO 8601 string |

The `change` event fires with `{ bubbles: true, composed: true }`, matching all other `pages-ui-components` form controls. It fires on blur (not on every keystroke) to match the palette's validation-on-blur contract.

## Component Structure

```
<pages-duration-input>
  #shadow-root
    <div class="field">
      <label>Duration</label>              <!-- when label is set -->
      <div class="duration-fields" role="group" aria-label="Duration">
        <div class="unit">
          <input type="number" min="0" step="1" aria-label="Duration hours" />
          <span class="unit-label">h</span>
        </div>
        <div class="unit">
          <input type="number" min="0" step="1" aria-label="Duration minutes" />
          <span class="unit-label">m</span>
        </div>
        <div class="unit">
          <input type="number" min="0" step="1" aria-label="Duration seconds" />
          <span class="unit-label">s</span>
        </div>
      </div>
      <span class="error" role="alert">Error message</span>  <!-- when error is set -->
    </div>
```

### Unit Labels

Short abbreviations below each number input:

| Field | Label |
|-------|-------|
| years | y |
| months | mo |
| days | d |
| hours | h |
| minutes | m |
| seconds | s |

### Layout

The `.duration-fields` container uses `display: inline-flex` with `gap: var(--pages-space-2, 8px)`. Each `.unit` is a vertical stack: number input on top, unit label below, centered.

Number inputs are narrow — `width: 3.5em` is sufficient for values 0–999. The inputs use `text-align: center` for visual alignment with the unit label.

## ISO 8601 Parsing and Serialization

### Parsing (string → field values)

Parse the `value` string using a regex that extracts each ISO 8601 duration component:

```
/^P(?:(\d+)Y)?(?:(\d+)M)?(?:(\d+)D)?(?:T(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?)?$/
```

- If the string doesn't match, all fields default to 0.
- Only populate fields that are in the `fields` array — ignore parsed values for hidden units. Hidden units are silently dropped, not accumulated into a visible unit.
- Empty string → all fields 0.

### Serialization (field values → string)

Build the ISO 8601 string from current field values:

1. Collect date parts (Y, M, D) and time parts (H, M, S)
2. Omit any unit with value 0
3. If any time parts are non-zero, include the `T` separator
4. If all parts are zero, emit `PT0S` (the canonical empty duration)
5. Prefix with `P`

Examples:
- hours=1, minutes=30, seconds=0 → `PT1H30M`
- days=2, hours=4 → `P2DT4H`
- all zeros → `PT0S`

### Round-Trip Behavior

When `value` is set externally, only the units present in `fields` are populated. If the incoming string contains units not in `fields` (e.g. `P1Y2M3DT4H5M6S` but `fields` only shows H/M/S), those hidden values are **not preserved** — the next serialization will omit them. This is intentional: the control edits only what it shows.

## Styling

CSS follows the existing form control pattern with `--pages-*` design tokens:

```css
:host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
.field { display: flex; flex-direction: column; gap: 6px; }
label {
  font-size: var(--pages-font-size-base, 14px);
  font-weight: var(--pages-font-weight-medium, 500);
  color: var(--pages-neutral-12, #333);
}
.duration-fields { display: inline-flex; gap: var(--pages-space-2, 8px); align-items: flex-start; }
.unit { display: flex; flex-direction: column; align-items: center; gap: 2px; }
.unit input {
  width: 3.5em;
  text-align: center;
  padding: var(--pages-space-1, 4px) var(--pages-space-1, 4px);
  border: 1px solid var(--pages-neutral-6, #e0e0e0);
  border-radius: var(--pages-radius-sm, 4px);
  font-size: var(--pages-font-size-base, 14px);
  font-family: inherit;
  background: var(--pages-neutral-1, #fff);
  color: var(--pages-neutral-12, #333);
  transition: border-color var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out);
}
.unit input:focus { outline: none; border-color: var(--pages-accent-9, #5470c6); }
.unit input:read-only { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; }
.unit input:disabled { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; opacity: 0.6; }
.unit-label {
  font-size: var(--pages-font-size-xs, 11px);
  color: var(--pages-neutral-9, #6b7280);
}
.error {
  color: var(--pages-danger-9, #dc2626);
  font-size: var(--pages-font-size-xs, 11px);
  margin-top: var(--pages-space-0-5, 2px);
}
```

## Accessibility

- The `.duration-fields` container has `role="group"` with `aria-label` set to the component's `label` property (or "Duration" if no label).
- Each number input has `aria-label="${label} ${fullUnitName}"` (e.g. "Duration hours", "Duration minutes").
- `aria-required` is set on the group container when `required` is true.
- `aria-invalid` is set on the group container when `error` is non-empty.
- Tab order follows the natural left-to-right field order.
- Readonly inputs use the native `readonly` attribute (focusable, not editable).
- Disabled inputs use the native `disabled` attribute (not focusable).

## Resolver Integration

Add the `format: "duration"` mapping to `resolveEditor()` in `packages/pages-property-palette/src/resolver.ts`:

```typescript
if (schema.format === 'duration') return { kind: 'tag', tag: 'pages-duration-input' };
```

This line goes in the `type === 'string'` branch, alongside the existing `format: "date"` and `format: "date-time"` mappings.

The palette's `renderField` method needs to import `@casehubio/pages-ui-components/duration-input` to ensure the element is registered when the resolver returns it. This follows the same pattern as the existing form control imports.

## Validation

The `validateField()` utility in `@casehubio/pages-ui-components/validation` does not need changes — it already handles `required` validation (empty string check). Duration-specific validation (if needed) would be added to `validateField` in a future pass.

Individual sub-inputs enforce `min=0` and `step=1` via native HTML number input constraints.

## Testing Strategy

Tests in `src/duration-input/pages-duration-input.test.ts` using Vitest + jsdom:

1. **Registration:** custom element registers as `pages-duration-input`
2. **Rendering:** renders correct number of inputs for default fields (3)
3. **Custom fields:** renders correct inputs when `fields` is changed (e.g. `['days', 'hours', 'minutes']`)
4. **Parsing:** setting `value="PT1H30M"` populates hours=1, minutes=30, seconds=0
5. **Parsing edge cases:** invalid string → all zeros, empty string → all zeros, `PT0S` → all zeros
6. **Serialization:** changing a field value updates the `value` property correctly
7. **Zero omission:** zeros are omitted from serialized string, all-zero → `PT0S`
8. **Hidden unit dropping:** `value="P1YT2H"` with `fields=['hours']` → only hours=2, years dropped
9. **Change event:** fires on sub-input blur with updated value
10. **Readonly:** all inputs have readonly attribute when `readonly=true`
11. **Disabled:** all inputs have disabled attribute when `disabled=true`
12. **Label:** renders label text when provided
13. **Error:** renders error message with `role="alert"`
14. **ARIA:** group has `role="group"`, inputs have `aria-label`

Resolver test in `packages/pages-property-palette/src/resolver.test.ts`:
15. `resolveEditor({ type: 'string', format: 'duration' })` returns `{ kind: 'tag', tag: 'pages-duration-input' }`

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `packages/pages-ui-components/src/duration-input/pages-duration-input.ts` | Create | Component implementation |
| `packages/pages-ui-components/src/duration-input/index.ts` | Create | Barrel export + registration |
| `packages/pages-ui-components/src/duration-input/pages-duration-input.test.ts` | Create | Unit tests |
| `packages/pages-ui-components/src/index.ts` | Edit | Re-export from `./duration-input` |
| `packages/pages-ui-components/package.json` | Edit | Add `./duration-input` sub-path export |
| `packages/pages-ui-components/tsconfig.build.json` | Edit | Add duration-input to references if needed |
| `packages/pages-property-palette/src/resolver.ts` | Edit | Add `format: "duration"` → `pages-duration-input` |
| `packages/pages-property-palette/src/resolver.test.ts` | Edit | Add resolver test for duration format |
| `packages/pages-property-palette/src/palette/pages-property-palette.ts` | Edit | Import `@casehubio/pages-ui-components/duration-input` |

## References

- `packages/pages-ui-components/src/date-input/pages-date-input.ts` — closest pattern (form control with ISO string value)
- `packages/pages-ui-components/src/number-input/pages-number-input.ts` — number input pattern (sub-inputs follow this)
- `packages/pages-ui-components/src/slider/pages-slider.ts` — precedent for inline plain `<input>` instead of custom element to avoid side-effect registration
- `packages/pages-property-palette/src/resolver.ts` — resolver mapping target
- `docs/specs/issue-373-property-palette/2026-08-26-property-palette-design.md` — parent spec, §Deferred: Duration Editor
- `docs/protocols/casehub/web-component-strategy.md` — PP-20260705-c7687d (Lit conventions)
- `docs/protocols/casehub/css-design-tokens.md` — PP-20260705-2ae91d (token naming)
- casehubio/casehub-pages#373 — parent issue
- casehubio/casehub-pages#374 — this issue
