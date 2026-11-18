---
layout: post
title: "The Parameter Nobody Read"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [grid-editing, nested-grids, yaml, api-contracts, typescript]
---

# The Parameter Nobody Read

Six methods in `GridEditBridge` accept a `gridPath` parameter. Only two read it.

`gridPath` tells an operation which grid to target. The root grid is `doc.getPages()[0]` — the top-level items array. But grids nest: a panel item can contain its own grid, and that nested grid is addressed by its YAML path — `['pages', 0, 'grid', 0, 'grid']`. When a user drags a chart inside a nested panel, `gridPath` carries that address.

Move and resize use it. `_applyPlacements` checks for `gridPath`, gets the YAML sequence at that path, and manipulates the nested entries directly. The other six operations — insert, split, merge, swap, remove, add-empty — accept `gridPath` as a parameter, prefix it with `_` to suppress the unused-variable warning, and proceed to operate on the root grid.

The type signature promises routing. The implementation ignores it.

## Why it was invisible

The UI currently dispatches `gridPath` only for move and resize — the two drag interactions. Split, merge, swap, remove, and insert are triggered from context menus and toolbar buttons that don't pass `gridPath`. Nobody called these methods with the parameter, so nobody noticed the parameter was dead.

But the API contract is a promise to future callers. A developer building nested grid editing would see that every operation accepts `gridPath`, assume it works, and discover — after debugging — that their remove deletes from the root grid while their merge reshapes root items. The operations are correct on the root grid. They're correct on the nested grid. The bug is that the caller asked for one and got the other.

## The two APIs

The root grid uses the PageNode API — `page.getItems()`, `item.setPlacement()`, `page.removeItem()`. These methods understand the document model, track undo, and fire change notifications.

Nested grids use raw YAML manipulation — `yamlDoc.getIn(gridPath)` returns a `YAMLSeq`, and you work with YAML maps directly: `entry.get('at', true)`, `entry.set('at', node)`, `seq.delete(index)`. This is what `_applyPlacements` already does for move/resize, and what `applyCrossGridEdit` does for cross-grid drags.

Each operation now has a `gridPath` branch at the top:

```typescript
if (gridPath) {
  const yamlDoc = doc._getDoc();
  const seq = yamlDoc.getIn(gridPath);
  if (!isSeq(seq)) return;
  // ... YAML-level manipulation
  return;
}
// ... existing PageNode path
```

The two paths share structure but not mechanics. The root path calls `page.removeItem(idx)`. The nested path calls `seq.delete(idx)`. The root path calls `item.setPlacement()`. The nested path calls `entry.set('at', yamlDoc.createNode({...}))`. Same operation, different substrate.

## What stays different

No compaction for nested grids. The root grid compacts after merge and remove — gravity pulls items down to fill gaps left by deleted or merged items. Nested grids skip this. The compaction algorithm runs against the root page's items, and running it against a nested YAML sequence would need a separate implementation. Skipping it is correct for now: nested grids are small (2–4 items in a panel) and the user controls their layout directly.

## The contract

An underscore prefix (`_gridPath`) is the TypeScript convention for "I accept this but don't use it." It satisfies the compiler and signals intent to the reader. But a function parameter is an API contract — something a caller relies on. Silencing the compiler doesn't silence the caller's expectation. The fix isn't to remove the parameter (the operations genuinely need it for nested grids) or to throw (the operations should work). The fix is to read it.
