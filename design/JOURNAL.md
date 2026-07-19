# Design Journal — issue-210-cell-spanning

## §Implementation — Single-Grid Rendering Model

### 2026-07-19 — Initial implementation (Tasks 1–6 of 8)

Implemented the cell spanning feature following the approved design spec from blocks-ui. Seven commits covering #211–#216, with #217 (Playwright visual tests) deferred.

**Key implementation decisions:**

1. **SpanMap computation order** — mergeRows runs first per column, then cellSpan overrides. This matches the spec's "cellSpan overrides mergeRows" semantics. First-writer-wins for overlapping spans is enforced by checking `map.has(suppColId)` before setting suppressed cells, and checking `isSuppressed(existing)` before processing cellSpan for a cell already claimed.

2. **Grid placement model** — all cells get explicit `grid-row: N; grid-column: M` inline styles. For virtual scroll, the body grid has `grid-template-rows: repeat(totalRows, rowHeight)` which defines the scrollable height via track metadata. No `translateY` — cells are placed at their actual grid positions.

3. **Row-level state on cells** — hover, selection, striping, filter-selected, row-style classes, and detail-expanded all moved from `.row` to `.cell`. The row wrapper exists only for ARIA (`role="row"`, `tabindex`, `aria-selected`) and event delegation (click/dblclick handlers bubble through `display: contents`).

4. **Hover range** — `_hoverRowIndex` and `_hoverRowSpan` track the hovered span range. Non-span rows have `_hoverRowSpan = 1`. All cells in the range `[hoverRowIndex, hoverRowIndex + hoverRowSpan)` receive the hover class.

5. **Keyboard span-awareness** — ArrowDown from a row with a span skips by the span's rowSpan. ArrowUp into a suppressed row resolves to the origin row. This uses the SpanMap directly in `_handleKeyDown`.

6. **Selection for spanned cells** — a spanned origin cell shows the `selected` class only when ALL rows in its span range are selected. This is computed per-cell in the rendering map, overriding the row-level `isSelected` flag.

**What's deferred:** Playwright visual tests (#217) require the full webapp build pipeline and browser infrastructure.
