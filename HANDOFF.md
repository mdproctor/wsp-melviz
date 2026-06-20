# casehub-pages Session Handover — 2026-06-20

## Last Session

Renamed melviz to casehub-pages (#24). Removed GWT from build (818 Java files → `_legacy/`). Renamed all 16 packages to `@casehub/pages-*` with architectural naming: standalone components get `pages-component-*`, iframe bridge gets `pages-iframe-api`/`pages-iframe-dev`. Wire protocol renamed (`casehub-pages-dataset`). Created `casehubio/casehub-pages` and `mdproctor/casehub-pages` on GitHub. Moved local dirs to `casehub/pages/`. Registered in casehub-parent (build-all.sh, CI workflows, PLATFORM.md, dashboards) and casehub-all (.gitmodules, update-pointers.yml).

## Branch State

Both repos on `main`. Fork and blessed current (`e6b10c2` squash + `b5202b7` ARC42 fix). Old repos (`melviz-org/melviz`, `mdproctor/melviz`) left as-is.

## What's Left

- Lazy on-demand pagination for datasets · M · High
- Cascading dropdown options · S · Med
- Record navigation controls (next/prev) · S · Med
- New record creation (POST) · S · Med
- Delete records (SaveAdapter.delete unwired) · S · Low
- Form-level error display · XS · Low
- "Warn if dirty" on page exit · XS · Med
- DataSetOptions path in dropdown · XS · Low
- Audit and remove `_legacy/` (#36) · S · Med
- PLATFORM.md TypeScript module conventions (#37) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #36 | Audit and remove GWT `_legacy/` | S | Med | Verify TS feature parity first |
| #37 | TypeScript module conventions protocol | S | Low | Document pages-prefix pattern for future TS modules |
| #23 | Domain-specific example dashboards | L | Med | Gallery stable, forms available |
| — | Lazy dataset pagination | M | High | Protocol-level change to datasets |
| — | Cascading dropdowns | S | Med | Options reactive to other field values |

## References

- Spec: docs/superpowers/specs/2026-06-19-casehub-pages-rename-design.md
- Plan: docs/superpowers/plans/2026-06-20-casehub-pages-rename-plan.md
- Blog: 2026-06-20-mdp01-casehub-pages-rename.md
