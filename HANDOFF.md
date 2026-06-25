# casehub-pages Session Handover — 2026-06-25

## Last Session

Branch `issue-24-view-state-persistence` closed. Centralized sort/pagination state into `ComponentViewState` (same pattern as `FilterState`). Pipeline now applies all state uniformly. CasehubTable stateless for sort/pagination — emits events, reads from VizTarget. URL format extended with `sort=` and `page=` params keyed by component ID. Fixed pre-existing filter restoration race (state before render) and popstate history corruption (`navigateInternal` extraction). Grid auto-ID removed so `component.id` reserved for `withId()`. Unused `DeepLink`/`ViewState` fields cleaned up. Issue #24 closed, #30 filed (double-render), #31 filed (text filter pipeline migration).

## Branch State

Both repos on `main`. Fork and blessed current (`60d59bc`).

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- Verify CI publish workflow ran on casehubio/casehub-pages (should have triggered on `60d59bc`)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #12 | Lazy on-demand pagination for datasets | M | High | Next substantive feature |
| #14 | Domain-specific example dashboards | L | Med | Gallery stable |
| #15 | Accessibility: ARIA, keyboard, screen reader | L | High | Deployment gate |
| #16 | CSP compliance: replace new Function() | M | Med | §12 risk |
| #21 | Optional Quarkus backend MVP | XL | High | Gates #22, #23 |
| #30 | Double render on data push (totalRows setter) | S | Low | Discovered during #24 |
| #31 | Migrate text filter to pipeline | S | Med | Follow-up from #24 |

## References

- Blog: `blog/2026-06-25-mdp01-view-state-split-personality.md`
- Spec: `docs/superpowers/specs/2026-06-25-view-state-persistence-design.md`
- LLM guide: `docs/CASEHUB-PAGES.md`
- Previous: `git show HEAD~1:HANDOFF.md`
