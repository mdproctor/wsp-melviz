# casehub-pages Session Handover — 2026-07-12

## Last Session

Closed #152 (TypedDataSet native API), #150 (examples gallery restructure), #154 (table runtime wiring + gallery fixes). The new `<pages-table>` Lit component is fully wired into the runtime pipeline — VizTarget compliance, .props setter for YAML activation, pipeline events (sort/page/filter), cross-filter, row styles, tree tables, column expressions. Gallery renamed from Melviz to CaseHub Pages. Many broken examples fixed. Filed 11 new issues (#155-165) for remaining gaps.

## Branch State

Both repos on main. No pause stack.

## Immediate Next Step

Pick up #164 (gallery examples sweep epic) — many examples still broken or incomplete. The gallery is the primary showcase and needs a dedicated sweep session.

## What's Left

- Gallery examples sweep — #164 (epic) · L · Med
- Table filter overlaps headers — #165 · S · Low
- Tree root-only pagination — #158 · M · Med
- Chart titles on canvas — #160 · S · Low
- Fleet Monitor gauge overlap — #133 · M · Med
- Typed payload overload for EventBroadcaster — #137 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #164 | Gallery examples sweep — fix all broken examples | L | Med | Epic covering #155,#158,#159,#160,#162,#163 |
| #159 | pages-schema-form — JSON Schema forms | L | High | Unblocks Developer Registration, Form Components |
| #142 | Scenario Engine — composition, triggers, demo UI | L | High | Plan ready |
| #143 | Cross-repo Migration — examples, blocks-ui, aml, clinical | L | Med | Plan ready |

## References

- Spec: `/Users/mdproctor/.claude/plans/twinkly-sniffing-coral.md`
- Blog: `2026-07-11-mdp01-the-type-already-there.md`
- Previous: `git show HEAD~1:HANDOFF.md`
