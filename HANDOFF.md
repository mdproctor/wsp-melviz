# casehub-pages Session Handover — 2026-06-24

## Last Session

Branch `issue-20-chart-click-cross-filter` closed. Unified the `casehub-filter` event protocol across all emitters with a discriminated union (`CasehubFilterApply | CasehubFilterReset`), toggle semantics, visual feedback (ECharts highlight/downplay for charts, `.selected` CSS for tables), and generalized record selection (any component, not just tables). Four spec review rounds caught: selectedMode conflict, series count formula bug, record selection reset path hole, table re-push asymmetry, IframePlugin reset ordering. Issue #20 closed.

## Branch State

Both repos on `main`. Fork and blessed current (`7a71d93`).

## Cross-Module

**We're blocking** (host repos waiting on published packages + theming/export + cross-filter):
- drafthouse #75 — Quinoa integration · S · Low
- claudony #161 — Quinoa integration · S · Low
- devtown #92 — Quinoa integration · XS · Low

## What's Left

- Verify CI publish workflow ran on casehubio/casehub-pages (should have triggered on `7a71d93`)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #12 | Lazy on-demand pagination for datasets | M | High | Next substantive feature |
| #14 | Domain-specific example dashboards | L | Med | Gallery stable |
| #15 | Accessibility: ARIA, keyboard, screen reader | L | High | Deployment gate |
| #16 | CSP compliance: replace new Function() | M | Med | §12 risk |
| #21 | Optional Quarkus backend MVP | XL | High | Gates #22, #23 |
| #24 | View state persistence (URL-based) | M | Med | Bookmarkable dashboards |

## References

- Blog: `blog/2026-06-24-mdp01-event-protocol-wrong.md`
- Spec: `docs/superpowers/specs/2026-06-23-chart-click-cross-filter-design.md`
- LLM guide: `docs/CASEHUB-PAGES.md`
- Previous: `git show HEAD~1:HANDOFF.md`
