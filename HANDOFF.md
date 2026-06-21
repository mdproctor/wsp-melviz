# casehub-pages Session Handover — 2026-06-21

## Last Session

Closed #2, #3, #4 on one branch (issue-2-ts-quality-sweep). TypeScript project references with incremental cross-package type checking (6.6s → 0.8s). ESLint strict-type-checked preset (149 auto-fixed, 653 remaining → #6). Post-strict follow-ups: Component<T, P extends object> constraint, ~400 branded ID casts migrated to constructors, expectAggregateColumn() test helper. Three garden entries submitted (rootDir resolution, emitDeclarationOnly technique, tsc --build --noEmit discrepancy).

## Branch State

Both repos on `main`. Fork and blessed current (`b982a8d`). Issues #2, #3, #4 closed.

## What's Left

- Lazy on-demand pagination for datasets · M · High
- Cascading dropdown options · S · Med
- Record navigation controls (next/prev) · S · Med
- New record creation (POST) · S · Med
- Delete records (SaveAdapter.delete unwired) · S · Low
- Form-level error display · XS · Low
- "Warn if dirty" on page exit · XS · Med
- DataSetOptions path in dropdown · XS · Low
- Pre-existing pages-runtime typecheck errors (Element vs HTMLElement, .dataset) · S · Low
- Pre-existing pages-viz test failures (CasehubTable filter events, CasehubBubbleChart sizing) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #5 | Navigation components distinct rendering | M | Med | Tree/menu/tiles all render as identical pills |
| #6 | Fix remaining ESLint strict-type-checked violations (653 errors) | M | Med | Breakdown by rule in issue body |
| — | Fix pages-runtime typecheck errors | S | Low | Element → HTMLElement narrowing, .dataset access |
| — | Fix pages-viz test failures | S | Low | CasehubTable filter event shape, BubbleChart sizing |
| #23 | Domain-specific example dashboards | L | Med | Gallery stable, forms available |

## References

- Blog: 2026-06-21-mdp02-ts-quality-sweep.md
- Garden: GE-20260621-710dfe, GE-20260621-d5e7d4, GE-20260621-f9970f (web/typescript)
