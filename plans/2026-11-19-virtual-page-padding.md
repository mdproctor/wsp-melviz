# Virtual Page Padding Opt-Out — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #686 — targetPage virtual pages inherit page-level padding — consider opt-out

**Goal:** Remove the 16px page-level padding from virtual pages created
by `targetPage: true`, so tab content renders edge-to-edge.

**Architecture:** Set `style: { padding: "0" }` on the synthetic page
component in the desugar step. The existing `component.style` override
in `render.ts` applies after page defaults, so this requires zero
renderer changes.

**Tech Stack:** TypeScript, Vitest, Playwright

## Global Constraints

- No changes to `render.ts` — the fix lives entirely in the desugar
- No new Component model concepts (no `virtual` flag, no new props)
- Existing `targetPage: true` YAML samples must still render correctly

---

## Batch 1: Virtual page padding removal

### Task 1: Remove padding from virtual pages

**Files:**
- Modify: `packages/pages-ui/src/parser/component-desugar.ts:462`
- Test: `packages/pages-ui/src/parser/component-desugar.test.ts:579-598`

**Interfaces:**
- Consumes: `Component` type from `packages/pages-component/src/model/types.ts` (has `style?: Record<string, string>`)
- Produces: virtual page components now include `style: { padding: "0" }`

- [ ] **Step 1: Update the existing test to expect `style: { padding: "0" }`**

In `component-desugar.test.ts`, update the "targetPage wraps each tab
slot in a virtual page" test. Add style assertion after the existing
props assertion:

```typescript
// In the test "targetPage wraps each tab slot in a virtual page" (line 579):
// After line 593: expect(pageA?.props).toEqual({ name: "Tab A" });
// Add:
expect(pageA?.style).toEqual({ padding: "0" });
// After line 597: expect(pageB?.type).toBe("page");
// Add:
expect(pageB?.style).toEqual({ padding: "0" });
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `yarn vitest run packages/pages-ui/src/parser/component-desugar.test.ts -t "targetPage wraps each tab slot"`

Expected: FAIL — `expected { padding: "0" }` but received `undefined`

- [ ] **Step 3: Add `style: { padding: "0" }` to the virtual page in desugar**

In `component-desugar.ts` line 462, change:

```typescript
// Before:
slots[name] = [{ type: "page", props: { name }, slots: { default: children } } as Component];

// After:
slots[name] = [{ type: "page", props: { name }, style: { padding: "0" }, slots: { default: children } } as Component];
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `yarn vitest run packages/pages-ui/src/parser/component-desugar.test.ts -t "targetPage wraps each tab slot"`

Expected: PASS

- [ ] **Step 5: Run the full desugar test suite to check for regressions**

Run: `yarn vitest run packages/pages-ui/src/parser/component-desugar.test.ts`

Expected: all tests pass

- [ ] **Step 6: Rebuild the examples bundle and verify in Playwright**

Rebuild the examples dist:
```bash
yarn workspace @casehubio/pages-examples build
```

Then use Playwright to navigate to Tree Table sample at
`http://localhost:8095/#client/Tables/Tables%2FTree%20Table.page.yaml`
and confirm the data table renders edge-to-edge with no 16px inset
in the tab panel.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui/src/parser/component-desugar.ts packages/pages-ui/src/parser/component-desugar.test.ts
git commit -m "feat(#686): remove padding from targetPage virtual pages"
```

## References

- [2026-11-19-virtual-page-padding-design.md] — design spec
- [component-desugar.ts:460-466] — virtual page creation
- [render.ts:60-80] — page-level styling
- [types.ts:8] — Component.style field
- [render.test.ts:598] — existing style override test
- [GitHub #686] — targetPage virtual pages padding opt-out
