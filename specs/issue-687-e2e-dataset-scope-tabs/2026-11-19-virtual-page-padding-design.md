# Virtual Page Padding Opt-Out — Design Spec

**Issue:** casehubio/casehub-pages#686
**Branch:** issue-687-e2e-dataset-scope-tabs
**Scale:** S | **Complexity:** Med

## Problem

When `targetPage: true` wraps each tab slot in a virtual page component
during desugaring, the virtual page inherits the standard page-level
padding (`var(--pages-space-4, 16px)`) from `render.ts`. This creates
an unwanted 16px inset for edge-to-edge content like data tables and
charts. The padding is a side effect of using `type: "page"` as the
wrapper, not an intentional visual treatment.

## Solution

Set `style: { padding: "0" }` on the synthetic page component created
by the desugar. This uses the existing style-override mechanism —
`render.ts` applies `component.style` after page-level defaults, so
the override wins without any changes to the renderer.

## Changes

### component-desugar.ts (line 462)

Current:
```typescript
slots[name] = [{ type: "page", props: { name }, slots: { default: children } } as Component];
```

After:
```typescript
slots[name] = [{ type: "page", props: { name }, style: { padding: "0" }, slots: { default: children } } as Component];
```

### component-desugar.test.ts

Update the "targetPage wraps each tab slot in a virtual page" test to
assert `style: { padding: "0" }` on the virtual page component.

### Playwright verification

Confirm Tree Table sample renders data table edge-to-edge in the tab
panel with no 16px inset.

## Trade-offs

If someone wants padding on a virtual page, there is no YAML syntax
for it — they would need to add padding to their content component's
style. This is unlikely since the tab container already provides the
visual boundary.

## References

- component-desugar.ts:460-466 — virtual page creation
- render.ts:60-80 — page-level styling and component.style override
- types.ts:8 — Component.style field
- render.test.ts:598 — existing test for style override on pages
