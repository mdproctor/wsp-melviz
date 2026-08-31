# Showcase gallery: interactive controls for data component samples

**Issue:** casehubio/casehub-pages#352
**Date:** 2026-08-31

## Status of original gaps

| Gap | Status |
|-----|--------|
| Content area doesn't scroll | DONE — 767f391a (overflow: auto) |
| Data components in pills/tabs stack overflow | Investigate and fix |
| No per-sample config UI | Implement |

## Gap 1: Per-sample config UI

### Design

Extend `samples.json` sample entries with an optional `config` array.
Each entry declares a YAML property path, control type, label, and
options:

```json
{
  "name": "Event Timeline",
  "path": "Charts/Event Timeline.dash.yaml",
  "config": [
    {
      "prop": "global.settings.layout",
      "label": "Layout",
      "type": "enum",
      "options": ["vertical", "horizontal", "compact"],
      "default": "vertical"
    }
  ]
}
```

Supported control types:
- **enum** — renders a `<select>` dropdown. `options` is the list of values.
- **boolean** — renders a checkbox. `default` is true/false.
- **string** — renders a text input (same as existing URL fields).

### Config bar rendering

The existing `renderConfigBar` function in `app.js` already renders
text inputs for URL property overrides with an Apply button. Extend it
to also render controls from the sample's `config` array.

**Key change:** config-driven controls apply **immediately** on change
(no Apply button needed for dropdowns and checkboxes — they have
discrete values). URL text inputs retain the Apply button.

### YAML property injection

The existing `applyPropertyOverrides` does regex replacement on flat
`key: value` lines. For `config` properties, the same approach works:
the `prop` field is the YAML key name (not a dotted path — YAML
properties are flat keys under `properties:` or `global.settings`).

When a config control changes:
1. Update `propertyOverrides[prop] = newValue`
2. Call `loadSampleInTarget(currentSample.path)` to reload

### Initial sample: Event Timeline

Add config to the Event Timeline sample as the first interactive example:
- Layout toggle: vertical / horizontal / compact

## Gap 2: Stack overflow with data components in pills/tabs

### Investigation needed

The issue reports that the page renderer recurses when data components
(`event-timeline`, `data-table`) are nested inside `pills` or `tabs`
sub-pages, but navigation components work fine.

This requires runtime investigation:
1. Create a test sample with a data component inside tabs
2. Load it in the gallery
3. If it stack-overflows, trace the recursion in pages-runtime
4. Fix the recursive path

This may already be fixed by other runtime work. If so, verify and
close this gap.

## Test plan

1. Add Event Timeline config to `samples.json` — verify dropdown renders
2. Change layout via dropdown — verify sample reloads with new layout
3. Create a test sample with data-table inside tabs — verify no stack overflow
4. Existing gallery samples continue to work unchanged

## Files changed

| File | Change |
|------|--------|
| `examples/src/app.js` | Extend `renderConfigBar` to handle `config` array |
| `examples/src/styles.css` | Styles for dropdown/checkbox controls in config bar |
| `examples/samples.json` | Add `config` to Event Timeline sample |
| `examples/samples/Charts/Event Timeline.dash.yaml` | May need layout property if not present |

## References

- `examples/src/app.js:264-323` — existing config bar and property overrides
- `examples/samples.json` — sample schema (name, path, category, file, tsPath)
- `767f391a` — scroll fix already landed
- casehubio/casehub-pages#352 — issue with gap analysis
