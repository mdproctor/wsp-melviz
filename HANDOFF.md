# HANDOFF — casehub-pages

## Last Session: 2026-11-24

Two issues completed across two stacked branches:

**Branch: issue-619-fixed-width-panel-autosize** (done, ready to land)
- **#619** — Added `fixedWidth?: number` to `GridPlacement`. At render time, measures grid container width, computes `effectiveW` using gap-aware formula `ceil((fixedWidth + gap) / (cellWidth + gap))`, clamped to declared `w`. Applies reduced `grid-column` span with `justify-self: start` and explicit `width`. Preserves `fixedWidth` through YAML parse/serialize and all edit bridge operations. Code review caught non-gridPath split dropping fixedWidth — fixed.

**Branch: issue-671-grid-hover-bar-tests** (done, stacked on #619)
- **#671** — Added 7 tests for hover bar action buttons: split-h, split-v, delete, merge, add-empty dispatch, bubbling/composed flags, merge non-bubbling.

Blog: "The Gap in the Gap" — why `ceil(fixedWidth / cellWidth)` is wrong and the gap-aware correction.

Filed: #743 (full sizing model — widthMode, heightMode, fixedWidth, fixedHeight across all layers).

## Immediate Next Step

Queue is empty. Use `work find` to discover new issues, or `work start #N` with a specific issue.

## Merge Note

Branches #723 and #736/#738 both modify `playbook-split-view.ts`. The #723 branch changes imports (demoCatalog extraction) while #736/#738 changes `parseAndResolve` logic. Resolution is mechanical but needs attention during rebase.

## Open Issues (by category)

**New:** #743 (full sizing model, M/Med)
**Mid-size:** #646, #719, #725
**Large:** #669, #616, #92, #94
**Epics:** #426, #95
