# HANDOFF — casehub-pages

## Last Session: 2026-11-23

**Branch: issue-739-cancel-remaining-false** (done, awaiting land)
- **#739** — Fixed `advanceAfter` defaulting: `cancelRemaining !== undefined` → `cancelRemaining === true`. The old check treated `cancel-remaining: false` the same as `true`, routing through the discriminator path unnecessarily. Added test for the missing row in the boolean test matrix. Clarified existing test title.
- **#742** filed — evaluator-level tests for parallel policy keys (advanceAfter, cancelRemaining, collectionTimeout) are missing. Queued.

Blog: "Defined Is Not True" — YAML boolean presence vs truthiness in config parsers.

## Immediate Next Step

Queue has 1 open item (#742). Use `work continue` to pick up #742 (evaluator tests for parallel policy keys, S/Med).

## Merge Note

Branches #723 and #736/#738 both modify `playbook-split-view.ts`. The #723 branch changes imports (demoCatalog extraction) while #736/#738 changes `parseAndResolve` logic. Resolution is mechanical but needs attention during rebase.

## Open Issues (by category)

**Quick wins:** #742 (evaluator parallel policy tests, S/Med)
**Mid-size:** #671, #619, #646, #719, #735
**Large:** #669, #616, #92, #94
**Epics:** #426, #95
