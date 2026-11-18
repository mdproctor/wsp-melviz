---
layout: post
title: "When Improvements Break Undo"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [grid-editing, undo, transactions, typescript, compound-operations]
---

# When Improvements Break Undo

Every `setPlacement` call in the grid editor pushes an undo snapshot. That's the right design — each mutation gets its own undo entry, so Ctrl-Z reverses exactly one change. The problem arrives when you add a second mutation that runs silently after the first.

Issue #651 added compaction to the grid editor. After every move or resize, gravity pulls items downward to fill gaps — the same behaviour you see in any drag-and-drop dashboard. The implementation is straightforward: after `_applyPlacements` updates the coordinates, `_compactGrid` scans for items that can drop to a lower y-position and calls `setPlacement` again for each one it finds.

Each of those compaction `setPlacement` calls pushes another undo snapshot.

A user moves a chart panel from row 0 to row 1. Compaction notices the space above is empty and pulls it back to row 0. The grid looks unchanged. Under the surface, the undo stack now has two entries: one for the original move, one for the compaction adjustment. Press Ctrl-Z and undo reverses the compaction — restoring the panel to row 1, the intermediate position nobody saw. Press Ctrl-Z again and you get back to row 0, the actual starting point. The user sees: press undo, panel jumps somewhere unexpected, press undo again, it goes home. That's the kind of bug that makes users distrust an editor.

![Undo stack — without and with transaction wrapping](images/undo-stack-transaction.svg)

## The transaction fix

The fix is `beginTransaction`/`commitTransaction` — PageDocument's mechanism for grouping multiple mutations into a single undo entry. During a transaction, individual `setPlacement` calls skip undo pushes entirely. `beginTransaction` captures the pre-transaction state once. `commitTransaction` fires the change notification once. One undo press reverses everything — the move and the compaction — atomically.

```typescript
doc.beginTransaction();
try {
  this._applyPlacements(doc, detail.items, gp);
  const page = doc.getPages()[0];
  if (page && (page.getLayoutMode() === 'grid'
      || page.getLayoutMode() === 'items')) {
    this._compactGrid(page);
  }
  doc.commitTransaction();
} catch (e) {
  doc.abortTransaction();
  throw e;
}
```

The `try`/`catch` matters. Without it, a thrown exception leaves the document stuck in transaction mode — every subsequent `setPlacement` silently skips undo pushes and notification callbacks. The document looks functional but undo stops working and change listeners stop firing. `abortTransaction` restores the pre-transaction state and clears the flag.

## The pattern

This pattern — composite operations needing transaction boundaries — surfaces whenever you layer a post-processing step onto an existing mutation pipeline. Compaction is the case here, but the same structure appears any time a correctness improvement adds implicit mutations after an explicit user action. The merge and remove operations in the same bridge have the same gap: both call `_compactGrid` after their primary mutation, both push multiple undo entries.

The boundary isn't "one function call" — it's "one user-visible operation." If the user pressed one button, undo should reverse one button press, regardless of how many internal mutations that button triggered.

## The typecheck session

The same session surfaced a parallel story. Running `yarn typecheck` produced 488 errors. Most were ghosts — stale `.typecheck/` declaration files from deleted source, the same species described in [The Ghosts in .typecheck](2026-11-18-mdp03-the-ghosts-in-typecheck.md). After `--build --clean`, the count dropped to 32. All 32 were real: `TreeNodeType` missing `'row'` and `'column'` values that `yaml-path.ts` returned, `rows` and `columns` functions in the DSL that were private but imported by examples, and `getButtonsForNodeType` consumed by its test file but not exported.

The 488-to-32 drop is the interesting number. For months, running typecheck with stale build output produced so much noise that the genuine errors were invisible. Fixing the clean script in #688 didn't fix the type errors — it made them findable.

Both fixes share a shape. Compaction made grid edits work better — and broke undo. The typecheck clean script made output accurate — and revealed 32 real errors hiding behind 488 stale ones. Improvements that expose latent bugs aren't regressions. They're the moment the system stops lying about its state.
