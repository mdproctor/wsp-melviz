# Melviz Session Handover — 2026-06-16

## Last Session

Closed #13. Implemented `@casehub/runtime` — new package that wires `@casehub/component` (layout renderer), `@casehub/ui` (page model), `@casehub/data` (dataset engine), and `@casehub/viz` (Web Components) into a working site runtime. Prerequisite changes to `@casehub/data` (LookupResult with totalRows, injectable fetch) and `@casehub/component` (onNode callback, sidebar wiring, casehub-slot-change events, activateSlot). Spec went through 3 review rounds. 764 tests across 3 packages. Squashed to 5 commits on fork/main.

## Branch State

Both repos on `main`. Branch `issue-13-site-runtime-navigation` closed. Fork remote (`mdproctor/melviz`) is current. Blessed repo (`melviz-org/melviz`) denied push (403 — no write access).

## What's Left

- #16 — Source-level dataset refresh timers (runtime creates timer after first resolution for datasets with `refreshTime`)
- #17 — Minor runtime improvements (navigate tree-walk, URL encoding, lazy-page activation, accordion test fragility)
- #18 — Integration tests for casehub-filter, casehub-page, casehub-sort, popstate event handlers

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #16 | Source-level dataset refresh timers | S | Low | Infrastructure in place, just needs setInterval after first resolution |
| #17 | Minor runtime improvements (4 items) | S | Low | Tree-walk navigate, URL encoding, lazy-page, test fix |
| #18 | Event handler integration tests | M | Med | Requires real DataSetManager, DOM events, custom event dispatch |
| #3 | Expression evaluator (JSONata bridge) | M | Med | Part of Phase 1 epic #1 |
| — | View state persistence (sessionStorage/localStorage/IndexedDB) | M | Med | Follow-up #15 |
| — | DnD visual builder | XL | High | Pure TS, operates over component model |
| — | Zod schemas + JSON Schema generation | M | Med | |

## Cross-Module

`@casehub/runtime` connects the four @casehub packages into a working pipeline. DraftHouse and Claudony can use `loadSite()` to render dashboard sites. The runtime's event protocol (casehub-data-request, casehub-filter, casehub-slot-change, casehub-page, casehub-sort) is the contract for any host that wants to embed a site.

## References

- Design spec: `docs/superpowers/specs/2026-06-15-site-runtime-design.md`
- Prior specs: `2026-06-15-casehub-component-grid-layout-design.md`, `2026-06-14-dashboard-model-design.md`, `2026-06-14-casehub-viz-design.md`
- Tests: 70 in `packages/casehub-runtime/src/`, 125 in `packages/casehub-component/src/`, 569 in `packages/core/src/`
- Garden: GE-20260616-e268d7 (Yarn workspace stale .d.ts silent parameter dropping)
- Epic #1: `mdproctor/melviz#1` (Phase 1)
