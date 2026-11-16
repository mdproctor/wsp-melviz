---
entry_type: article
subtype: diary
title: "When the Key Tells You the Layout"
date: 2026-11-10
tags: [casehub-pages, yaml, grid, layout-model, developer-experience]
projects: [casehubio/casehub-pages]
series: issue-640-redundant-items-components
---

# When the Key Tells You the Layout

The `items` / `placement` / `component` vocabulary from the #625 migration was a stepping stone, not a destination. It unified three structural layers into one — but it still carried unnecessary ceremony. A page author writing a simple vertical layout had to know that `components:` meant "flat" while `items:` meant "grid", and every grid entry had to wrap its content inside a `component:` key and specify `placement:` with nine characters before getting to the coordinates.

The format looked like this:

```yaml
items:
  - placement: { x: 0, y: 0, w: 6, h: 2 }
    component:
      type: title
      properties: { text: Revenue Overview }
  - placement: { x: 6, y: 0, w: 6, h: 2 }
    component:
      type: chart
      properties: { dataset: sales }
```

Two things are wrong here. First, `items:` doesn't tell you anything about the layout. Is it a grid? A list? The only way to know is to scan the entries for `placement` keys. Second, the `component:` wrapper adds an indentation level that serves no purpose — a grid entry IS a component with a position. There's nothing else it could be.

## The fix: let the key declare the layout

```yaml
grid:
  - at: { x: 0, y: 0, w: 6, h: 2 }
    type: title
    properties: { text: Revenue Overview }
  - at: { x: 6, y: 0, w: 6, h: 2 }
    type: chart
    properties: { dataset: sales }
```

Three changes, each earning its place.

**`grid:` instead of `items:`** — the key tells you what kind of layout this is. No scanning, no inference. A page that uses `flat:` is a vertical flow. A page that uses `grid:` is a positioned grid. The parser dispatches on the key name, not on the shape of the entries.

**`at:` instead of `placement:`** — two characters instead of nine. Reads naturally: "this component is at position (0,0), spanning 6 columns and 2 rows." The coordinates are the same `{ x, y, w, h }` that map directly to CSS Grid's `grid-column` and `grid-row`.

**No `component:` wrapper** — `type` and `properties` sit directly on the grid entry alongside `at:`. A grid entry with `at:` and no `type` is an empty container — the same semantics as before, just one indentation level shallower.

![Format evolution from rows/columns through items to grid/flat](images/yaml-format-evolution.svg)

## What flat pages look like

A page with no grid positioning — just components flowing top to bottom:

```yaml
flat:
  - type: title
    properties: { text: Hello }
  - type: markdown
    properties: { content: Welcome to the app }
```

This used to be `components:` and still works as `components:` for backwards compatibility. But `flat:` makes the intent explicit. You know the layout mode from the first line.

## Nested grids

A grid entry can contain its own `grid:` — the recursion is clean:

```yaml
grid:
  - at: { x: 0, y: 0, w: 6, h: 3 }
    type: grid
    grid:
      - at: { x: 0, y: 0, w: 6, h: 1 }
        type: metric
        properties: { label: Users, value: 1234 }
      - at: { x: 6, y: 0, w: 6, h: 1 }
        type: metric
        properties: { label: Revenue, value: $5.6M }
  - at: { x: 6, y: 0, w: 6, h: 3 }
    type: chart
    properties: { dataset: sales }
```

Same key at every level. No special "sub-grid" vocabulary. A component that has children positions them with `grid:` — whether it's the page root or a nested container.

## Why naming matters more than it looks

The previous vocabulary — `items`, `components`, `rows`, `columns`, `placement` — had a problem beyond length. Each key described a structural concept that only made sense if you already understood the layout model's history. "Items" are... items in what? "Components" are components of what? These are words that carry no information about how the page will render.

`grid:` and `flat:` are predictions. They tell the page author what CSS layout strategy will be used before they read a single entry. `at:` is a preposition — it connects a component to a position in the way English naturally expresses location.

This matters for the platform because the `.page.yaml` format is the primary authoring surface. Every person who writes a page reads these keys hundreds of times. Shaving nesting and using self-describing keys isn't optimisation for its own sake — it's reducing the distance between the author's intent and the YAML that expresses it.

## Backwards compatibility

The parser accepts all three format generations. A page written with `rows:` / `columns:` from the Java platform still renders. A page written with `items:` / `placement:` from the #625 migration still renders. The new format is canonical — it's what the builder writes and what all sample files in this repo now use.

A separate work-slot will search across all platform repos and migrate them to the canonical format. Until then, the TS runtime accepts everything and normalises internally.

The migration touched 244 `.page.yaml` files in this repo. The parser priority is: `grid:` > `flat:` > `items:` > `components:` > `rows:` > `columns:`. Each layer is a strict superset of the one below — nothing breaks, nothing is removed, the old formats just stop being the default.
