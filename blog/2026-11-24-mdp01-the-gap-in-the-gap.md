---
layout: post
title: "The Gap in the Gap"
date: 2026-11-24
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [css-grid, layout, formula]
---

# The Gap in the Gap

Fixed-width panels have been leaving dead space in the grid since the POC. A panel declared at 160px occupying two columns of a 12-column grid renders its 160 pixels and then... nothing. The remaining 50-odd pixels sit there, visually empty but structurally owned. No other panel can use them because CSS Grid's `grid-column: 9 / span 2` claims both columns regardless of what's inside.

The fix is obvious: compute how many columns the panel actually needs and use that for the span. `Math.ceil(fixedWidth / cellWidth)` — done.

Except it's wrong.

The first version of the formula gave me the right answer for most cases, but I caught it during the spec self-review. Consider: `fixedWidth = 210`, `cellWidth = 100`, `gap = 12`. The naive formula gives `ceil(210/100) = 3` columns. But two columns actually provide `2 × 100 + 12 = 212` pixels — the gap between the columns is available space within the span. 210 fits in 212. The panel needs two columns, not three.

The correct formula accounts for inter-column gaps:

```
effectiveW = ceil((fixedWidth + gap) / (cellWidth + gap))
```

Adding `gap` to both numerator and denominator solves for the minimum `n` where `n × cellWidth + (n-1) × gap ≥ fixedWidth`. The algebra is a one-liner but the intuition took a moment: a CSS Grid span of `n` columns isn't `n × cellWidth` wide. It's `n × cellWidth + (n-1) × gap`. Every column you span beyond the first brings its left gap with it.

We clamped the result to the declared `w` from YAML. If the author says `w: 3` and the panel only needs 1 column at the current viewport width, it gets 1. But it never grows beyond 3 — the declared span is a ceiling, not a suggestion. On narrow viewports where `fixedWidth` genuinely needs all three columns, the panel behaves normally. On wider viewports, it shrinks. The `justify-self: start` pins it to the left edge of the reduced span.

The implementation threads `fixedWidth` through four layers: type model (`GridPlacement`), YAML document parsing, the renderer, and the edit bridge. The renderer measures the grid container's width before placing items and computes `cellWidth` from the actual rendered dimensions — no hardcoded assumptions about container size. The edit bridge was the tedious part: every operation that constructs an `at` object from an existing item (move, split, insert, merge, swap, cross-grid move, compaction) needed to carry `fixedWidth` forward. The code review caught one I'd missed — the non-gridPath split was silently stripping it because `setPlacement` deletes `fixedWidth` when the new placement omits it.

The hover bar action buttons got test coverage too — split, delete, merge, add-empty dispatch verification. Seven tests, all checking event shapes and bubbling behaviour. The kind of thing that should have been there from the start but got deferred when the buttons were rewired in #665.

The gap formula applies anywhere CSS Grid spans include inter-column space. I suspect it'll come back when responsive re-computation lands — the same math needs to run inside a ResizeObserver callback, with the same gap awareness. The hardcoded `gap = 12` will need to become a computed value at that point, since compact density overrides it to 8px.
