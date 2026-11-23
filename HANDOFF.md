# HANDOFF — casehub-pages

## Last Session: 2026-11-23

**Branch: issue-741-scanner-match-guard-tests** (from issue-722-match-address-mismatch)
- **#741** — Added two scanner tests for `scanMatchCases`: guard key skipping and nested control flow inside match cases. The control flow test uncovered a real bug — `findBranchEnd` truncated the step boundary at sibling structural keys (`then:`/`else:`) and `scanControlFlow` received the wrong indent level (`caseContentIndent` instead of `caseListIndent`). Fixed with a two-line change: use `caseEnd` as step boundary, pass `caseListIndent` for correct branch indent computation.

Blog: "The Boundary Nobody Tested" — tests as specification: writing tests for assumed-working code uncovered a real boundary computation bug.

## Immediate Next Step

Queue has 2 open items (#740, #739). Use `work continue` to pick up #740 (resolve concurrent keyword inconsistency, XS/Low).

## Merge Note

Branches #723 and #736/#738 both modify `playbook-split-view.ts`. The #723 branch changes imports (demoCatalog extraction) while #736/#738 changes `parseAndResolve` logic. Resolution is mechanical but needs attention during rebase.

## Open Issues (by category)

**Quick wins:** #740 (concurrent keyword, XS/Low), #739 (cancel-remaining false, S/Low)
**Mid-size:** #671, #619, #646, #719, #735
**Large:** #669, #616, #92, #94
**Epics:** #426, #95
