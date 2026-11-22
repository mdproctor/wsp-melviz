# HANDOFF — casehub-pages

## Last Session: 2026-11-22

Six issues completed across three branch stacks, all yaml-core and step-viewer:

**Branch: issue-723-extract-democatalog** (from main)
- **#723** — Extracted `demoCatalog` from `playbook-split-view.ts` and `gallery.html` into shared `src/demo-catalog.ts` with proper `Catalog` typing.

**Branch: issue-722-match-address-mismatch** (from main)
- **#722** — Added `scanMatchCases` to StepPositionScanner for the `cases:` list format. Scanner now produces numeric addresses (`0.match.0.0`) matching step-click's scheme. Handles both `when:` pattern cases and `default:` cases.

**Branch stack: issue-734 → 736 → 737 → 738** (stacked from main)
- **#734** — Reject `steps:` key inside parallel map form with migration message.
- **#736** — Reject `steps:` document-level wrapper in `parseAndResolve`.
- **#737** — Default `advance-after` to branch count when `cancel-remaining` or `collection-timeout` is present.
- **#738** — Remove stale `concurrent` from `RESERVED_KEYS`.

Code review found and fixed a no-op ternary in #736. Filed three follow-up issues:
- **#739** — Verify `cancel-remaining: false` platform behavior (S/Low)
- **#740** — Resolve `concurrent` keyword inconsistency (XS/Low)
- **#741** — Add scanner tests for match cases with guard and nested control flow (XS/Low)

Blog: "Parity as a Forcing Function" — platform parity audit as a forcing function for cleaning up implicit contracts.

## Immediate Next Step

Queue has 3 open items (#741, #740, #739) — all XS/S from code review findings. Use `work continue` to pick up #741.

## Merge Note

Branches #723 and #736/#738 both modify `playbook-split-view.ts`. The #723 branch changes imports (demoCatalog extraction) while #736/#738 changes `parseAndResolve` logic. Resolution is mechanical but needs attention during rebase.

## Open Issues (by category)

**Quick wins:** #741 (scanner tests, XS/Low), #740 (concurrent keyword, XS/Low), #739 (cancel-remaining false, S/Low)
**Mid-size:** #671, #619, #646, #719, #735
**Large:** #669, #616, #92, #94
**Epics:** #426, #95
