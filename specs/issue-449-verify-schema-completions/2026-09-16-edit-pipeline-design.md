# Normalized Edit Pipeline — Design Spec

## Problem

The workbench has six entry points that mutate document state, three independent
write-back paths, and no coordination between them. The result:

- **Double write-back**: shell's `_subscribeToDocument` and YamlSync both push to
  the editor on every model change
- **Document destruction**: every editor keystroke creates a new PageDocument,
  killing undo history and node references
- **Cursor obliteration**: `_pushYamlToEditor` replaces ALL editor content on every
  model change, destroying cursor position
- **Sync fighting**: model-originated write-backs interrupt active user typing
- **Unhandled events**: tree context menu actions (`tree-action`) fire into the void

## Design Principles

1. **Single source of truth.** PageDocument is the authority. All views derive
   from it. The editor text is one view, not the document.

2. **Edit origin tracking.** Every mutation carries its origin. The sync fan-out
   uses the origin to avoid destructive write-back — specifically, the editor is
   skipped when it originates the edit because replacing editor content would
   destroy cursor position and in-progress typing. Other views (tree, properties,
   preview) derive from the model via getters, so re-syncing them from any origin
   is idempotent and safe.

3. **No circular sync.** A → B → A loops are eliminated by construction. View
   syncs read from the model and render — they don't trigger mutations, so
   re-syncing the origin view cannot create a feedback loop.

4. **Cursor preservation.** Editor updates use minimal text diffs, not full
   replacement. The cursor stays where the user left it.

5. **Error tolerance.** Invalid YAML during typing is normal. The model stays at
   the last valid state. The editor shows the user's in-progress text. No crashes.

6. **Single coordinator.** One method (`_applyEdit`) is the only path from edit
   sources to view sync. No shortcuts, no parallel paths.

## Coordinated Mode

PageDocument has its own undo stack and change notification, designed for
standalone use. Every node method (`PageNode.addRow()`, `ComponentNode.setProperty()`,
etc.) calls `_pushUndoInternal()` and `_notifyInternal()` after each mutation.
PageDocument's own mutation methods (`addPage()`, `setProperty()`, etc.) call
`_pushUndo()` and `_notify()` directly.

When the shell takes over coordination, these internal mechanisms conflict:
- **Dual undo:** both the shell's undo stack and PageDocument's internal stack
  record every mutation.
- **Mid-pipeline notification:** `_notifyInternal()` fires listeners during
  `fn()` execution — before `_syncViews` runs — causing partial syncs,
  double editor writes, and double change emissions.

**Solution:** PageDocument gains a `_coordinated` flag. When set:
- `_pushUndo()` / `_pushUndoInternal()` → no-op (shell manages undo)
- `_notify()` / `_notifyInternal()` → no-op (shell manages view sync)
- `beginTransaction()` skips its undo push (but still sets `_inTransaction`
  for rollback via `abortTransaction()`)
- `commitTransaction()` skips notification (but still clears `_inTransaction`)
- `abortTransaction()` skips `_undoStack.pop()` and `_redoStack` clear —
  since `beginTransaction()` didn't push, there's nothing to pop. Document
  restore from `_transactionSnapshot` and `_inTransaction` reset still
  apply. The rollback mechanism is unaffected.

The transaction API remains functional for **atomicity** — complex multi-step
operations like `replaceWith()`, `moveToSlot()`, and `wrapIn()` still use
`beginTransaction/abortTransaction` for rollback on error. The only change
is that notification and undo recording are deferred to the coordinator.

The shell uses `PageDocument.parseCoordinated(yaml)` — a factory method that
returns a new instance with `_coordinated = true` already set. This ensures
coordination is never forgotten at a call site:

```typescript
static parseCoordinated(yaml: string): PageDocument {
  const doc = PageDocument.parse(yaml);
  doc._coordinated = true;
  return doc;
}
```

Tests and standalone consumers continue to use `PageDocument.parse()` (which
returns `_coordinated = false` by default). The factory is the only API that
sets the flag — no public setter needed.

**Consequence:** `_subscribeToDocument()` in the shell is removed entirely.
The shell no longer registers an `onChange` listener on PageDocument — all
view updates flow through `_syncViews`, all change notifications through
`_emitChange()` in `_syncViews`.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                       BuilderShell                           │
│                                                              │
│  Edit Sources ──────────────► _applyEdit(origin, fn) ────┐   │
│  ┌─────────────────────────┐          │                  │   │
│  │ Toolbar:  +Page, Undo   │          │ 1. Flush pending │   │
│  │ Tree:     +, ctx, drop  │          │    editor debounce│  │
│  │ Palette:  component pick│          │ 2. Snapshot undo │   │
│  │ Props:    field change   │          │ 3. Execute fn()  │   │
│  │ Editor:   text debounce  │          │ 4. _syncViews()  │   │
│  └─────────────────────────┘          │                  │   │
│                                       ▼                  │   │
│                              ┌────────────────────┐      │   │
│                              │   PageDocument      │      │   │
│                              │   (source of truth) │      │   │
│                              └────────┬───────────┘      │   │
│                                       │                  │   │
│                              ┌────────▼───────────┐      │   │
│                              │  _syncViews(origin) │◄─────┘   │
│                              │                     │          │
│                              │  if origin≠editor:  │          │
│                              │    diff-patch editor │          │
│                              │  always:             │          │
│                              │    rebuild tree      │          │
│                              │    refresh props     │          │
│                              │    re-render preview │          │
│                              │    _emitChange()     │          │
│                              │    requestUpdate()   │          │
│                              └─────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

## Components

### EditOrigin

```typescript
type EditOrigin = 'editor' | 'tree' | 'properties' | 'palette' | 'toolbar';
```

Carried through every edit. Used by `_syncViews` to decide what to skip.

### _applyEdit (the coordinator)

Single entry point for all document mutations.

```typescript
private _applyEdit(origin: EditOrigin, fn: () => void): void {
  // 1. If a structured edit arrives while editor has pending debounced
  //    changes, flush them first so we don't lose in-progress text.
  //    _flushEditorSync pushes its own undo snapshot directly (no recursion).
  if (origin !== 'editor' && this._pendingEditorSync) {
    this._flushEditorSync();
  }

  // 2. Snapshot for undo (stores YAML string, ~2KB per snapshot)
  this._undoStack.push(this._document.toString());
  this._redoStack.length = 0;

  // 3. Execute the mutation
  fn();

  // 4. Sync all views, notify external consumers, trigger re-render
  this._syncViews(origin);
}
```

**Undo granularity for editor edits:** The editor debounce fires after 300ms of
idle time. Each debounce flush pushes one undo snapshot. Continuous typing
produces ~1 snapshot per pause. This matches user expectations — undo reverts
to the last pause point, not each character.

**Flush-before-structured-edit:** When the user is typing in the editor and then
clicks a palette item, the pipeline flushes pending text changes to the model
first, THEN applies the palette insert. `_flushEditorSync` pushes its own undo
snapshot and swaps the document directly — it does NOT call `_applyEdit`, which
avoids nested undo snapshots and double `_syncViews` calls. The outer
`_applyEdit` then pushes a second snapshot and syncs all views once. If the
pending text is invalid YAML, the flush fails silently and the structured edit
operates on the last valid model state.

**`_emitChange()` and `requestUpdate()`:** The current `_subscribeToDocument`
fires these on every model change. In the new pipeline, they move into
`_syncViews` — at the end, after all view updates. Every code path that
updates the document calls `_syncViews`, so these always fire. `_emitChange()`
dispatches `builder-change` (consumed by `pages-runtime/site.ts` for
persisting changes and `pages-aria/tutorial-host.ts` for tutorial validation).
`requestUpdate()` triggers Lit's re-render cycle for reactive properties that
depend on document state (e.g. undo/redo button enablement).

### _parseDocument (coordinated PageDocument factory)

Shell-level helper that delegates to `PageDocument.parseCoordinated()` (see
§Coordinated Mode). All shell code paths use this helper — never
`PageDocument.parse()` directly.

```typescript
private _parseDocument(yaml: string): PageDocument {
  return PageDocument.parseCoordinated(yaml);
}
```

### _syncViews

Fan-out to all views, skipping the origin.

```typescript
private _syncViews(origin: EditOrigin): void {
  if (origin !== 'editor') {
    this._diffPatchEditor();
  }
  this._syncTree();
  this._syncProperties();
  this._syncPreview();
  this._emitChange();
  this.requestUpdate();
}
```

### _diffPatchEditor (replaces _pushYamlToEditor)

Computes a minimal text diff between the editor's current content and the
document's `toString()`, then applies only the changed ranges. CodeMirror
preserves cursor position for unchanged ranges.

```typescript
private _diffPatchEditor(): void {
  const editorText = this._getEditorText();
  const modelText = this._document.toString();
  if (editorText === modelText) return;

  const changes = computeMinimalChanges(editorText, modelText);
  this._editorView.dispatch({ changes });
}
```

`computeMinimalChanges` produces CodeMirror `ChangeSpec[]` from a line-based
diff. Only changed lines produce replacement ranges. Unchanged ranges keep
their cursor position, selection, and decorations.

**Algorithm:** Line-based prefix/suffix matching. Split both strings into lines.
Find the longest common prefix (lines identical from the top) and longest common
suffix (lines identical from the bottom). The changed region is everything
between. Produce a single `ChangeSpec` replacing that region's character range.
O(n) for the common case (single contiguous edit region). For typical page
documents (<100 lines), this is sub-millisecond.

This approach produces a single replacement range, not per-line diffs. A full
Myers/LIS diff would find multiple disjoint change regions, but the extra
complexity isn't warranted: structured edits (property change, add component)
typically touch one contiguous YAML block, and the prefix/suffix approach
correctly identifies it. The cursor is preserved for all content outside the
replaced region — which is the design goal.

### EditorInputHandler (replaces YamlSync for editor→model)

Handles editor text changes with debouncing and active-typing suppression.

```typescript
private _pendingEditorSync: number | undefined;
private _editorDirty = false;

private _handleEditorInput(): void {
  this._editorDirty = true;
  clearTimeout(this._pendingEditorSync);
  this._pendingEditorSync = window.setTimeout(() => {
    this._pendingEditorSync = undefined;
    const text = this._getEditorText();
    const newDoc = this._parseDocument(text);
    if (newDoc.diagnostics.some(d => d.severity === 'error')) {
      // Invalid YAML — keep editor dirty, model stays at last valid state.
      // _editorDirty stays true so the next valid parse will flush.
      return;
    }
    this._applyEdit('editor', () => {
      this._document = newDoc;
    });
    this._editorDirty = false;
  }, 300);
}

private _flushEditorSync(): void {
  clearTimeout(this._pendingEditorSync);
  this._pendingEditorSync = undefined;
  if (!this._editorDirty) return;

  const text = this._getEditorText();
  const newDoc = this._parseDocument(text);
  // PageDocument.parseCoordinated() never throws — it always returns a
  // document, recording parse failures as diagnostics. Check for errors.
  if (newDoc.diagnostics.some(d => d.severity === 'error')) {
    // Invalid YAML — model stays at last valid state.
    // _editorDirty stays true intentionally: when the user fixes the YAML
    // and the next debounce fires, the parse will succeed and flush.
    // _pendingEditorSync is already cleared (top of method), so subsequent
    // structured edits won't attempt another flush until new input arrives.
    return;
  }
  // Push undo snapshot and swap document directly — NOT via _applyEdit.
  // This avoids nested undo/sync when _applyEdit calls _flushEditorSync
  // as part of flush-before-structured-edit.
  this._undoStack.push(this._document.toString());
  this._redoStack.length = 0;
  this._document = newDoc;
  this._editorDirty = false;
  // No _syncViews here — the calling _applyEdit will sync after
  // its own mutation. When called standalone (debounce timeout),
  // use _applyEdit instead (see _handleEditorInput).
}
```

**Why 300ms debounce (up from YamlSync's 150ms):** 150ms fires while many users
are still composing — the inter-keystroke gap for a moderate typist (60 WPM)
is ~200ms. At 150ms, partial words trigger parse attempts that will fail and
immediately be superseded. 300ms aligns with typical pause-between-words timing,
reducing unnecessary parse failures during active typing while remaining
responsive enough that the model updates feel immediate when the user pauses.

**Why PageDocument.parse (new instance) is acceptable here:** Editor text edits
can produce arbitrarily different YAML. An in-place diff-and-patch on the
model would be complex and fragile. Creating a new PageDocument from the text
is correct — the undo stack lives in the shell (string snapshots), not in the
PageDocument instance.

### External Property Changes (`willUpdate`)

When the host sets `shell.yaml = "new yaml"` (e.g. loading a different page),
the Lit `willUpdate` lifecycle handles it outside the edit pipeline:

```typescript
override willUpdate(changed: Map<PropertyKey, unknown>): void {
  if (changed.has('yaml') && changed.get('yaml') !== undefined) {
    this._document = this._parseDocument(this.yaml);
    // External property change = new document baseline.
    // Clear undo stacks — the previous document's history is irrelevant.
    this._undoStack.length = 0;
    this._redoStack.length = 0;
    // Cancel any pending editor sync from the previous document.
    clearTimeout(this._pendingEditorSync);
    this._pendingEditorSync = undefined;
    this._editorDirty = false;
    // Full view sync — this is not an edit, it's a document replacement.
    // _syncViews handles _emitChange() and requestUpdate() internally.
    this._syncViews('toolbar');
  }
}
```

This is NOT routed through `_applyEdit` because:
- It's not a user edit — no undo snapshot should be pushed
- It clears the undo stack (new document baseline)
- The origin is external, not any of the five EditOrigin panels

### Undo/Redo

Shell-managed stack of YAML string snapshots. Simple, robust, works across all
edit origins.

```typescript
private _undoStack: string[] = [];
private _redoStack: string[] = [];

private _undo(): void {
  if (this._undoStack.length === 0) return;
  this._redoStack.push(this._document.toString());
  const prev = this._undoStack.pop()!;
  this._document = this._parseDocument(prev);
  this._syncViews('toolbar');
}

private _redo(): void {
  if (this._redoStack.length === 0) return;
  this._undoStack.push(this._document.toString());
  const next = this._redoStack.pop()!;
  this._document = this._parseDocument(next);
  this._syncViews('toolbar');
}
```

No `_subscribeToDocument()` — the coordinator pattern means no onChange listener
is needed on the new document. `_syncViews` handles `_emitChange()` and
`requestUpdate()` internally, so callers just call `_syncViews(origin)`.

Stack size limit: 50 entries (matching PageDocument's current `MAX_UNDO_STACK`).
Per-snapshot size depends on document complexity: ~2KB for minimal documents,
up to ~20–50KB for large documents (10 pages × 5 rows × 3 columns × 2
components each). Worst case at 50 entries: ~2.5MB. Acceptable for a desktop
editor; the stack is capped and entries are plain strings.

## Edit Flows (all entry points)

### Toolbar: +Page, +Dataset

```
Button click → _applyEdit('toolbar', () => doc.addPage())
            → _syncViews('toolbar')
              → _diffPatchEditor (cursor preserved)
              → _syncTree (new page node appears)
              → _syncProperties (selection unchanged)
              → _syncPreview (new page rendered)
              → _emitChange()
              → requestUpdate()
```

### Toolbar: Undo, Redo

```
Button click → _undo() / _redo()
            → _syncViews('toolbar')
              → full sync (all views update)
```

### Tree: node-select

```
Click node → _handleNodeSelect (NO mutation)
           → _updatePropertySource
           → scroll/highlight editor
           → highlight preview
```

Not an edit — no `_applyEdit`. Selection-only updates.

### Tree: + button (tree-add)

```
Click + → _handleTreeAdd
        → select target node
        → open inline picker
        → user picks component
        → _applyEdit('tree', () => parent.addComponent(type, props))
        → _syncViews('tree')
          → _diffPatchEditor
          → _syncTree
          → _syncProperties
          → _syncPreview
          → _emitChange()
          → requestUpdate()
```

### Tree: context menu (tree-action)

```
Right-click → context menu → action selected
            → _handleTreeAction(action, path, nodeType)
            → _applyEdit('tree', fn)
            → _syncViews('tree')
```

The handler resolves `path` to the appropriate node using existing shell path
helpers (`_findPageAtPath`, `_findRowAtPath`, `_findColumnAtPath`,
`_findComponentAtPath`, `_findDatasetAtPath`), then calls the existing
node-level API.

**Complete action set** (derived from `tree-context-menu.ts`):

| Action | Node types | API call |
|--------|-----------|----------|
| `delete` | all | `parentNode.removeChild(index)` / `row.removeColumn(index)` / `col.removeComponent(index)` |
| `move-up` | component, row, column | `component.moveToIndex(parent, index - 1)` / sequence swap for rows/columns |
| `move-down` | component, row, column | `component.moveToIndex(parent, index + 1)` / sequence swap for rows/columns |
| `duplicate` | component, dataset | `component.duplicate()` |
| `add-row` | page | `page.addRow()` |
| `add-column` | row | `row.addColumn()` |
| `add-child` | page, column, container component | `node.addComponent(type, props)` — opens palette picker |
| `wrap-row` | component | `page.wrapInRow([componentIndex])` |
| `wrap-column` | component | `component.wrapIn('column-container')` or equivalent container type |
| `wrap-tabs` | component | `component.wrapIn('tabs')` |
| `replace-with` | component | `component.replaceWith(newType)` — opens type picker |

**Move up/down for rows and columns** requires adding `moveToIndex` support on
`RowNode` and `ColumnNode`, or implementing sequence-level swaps directly. The
existing `ComponentNode.moveToIndex` demonstrates the pattern — extract the YAML
node, delete, splice at the new position. This is a PageDocument API extension
deferred to the implementation issue.

**`add-child` and `replace-with`** open a picker UI before the mutation. The
handler dispatches the picker, and the mutation happens in the picker's callback
via `_applyEdit('tree', ...)`. This is the same pattern as `_handleTreeAdd`.

**Template wiring:** `_syncTree` must wire `@tree-action` and `@tree-drop`
event listeners on the `<pages-builder-tree>` element — currently it only
wires `@node-select` and `@tree-add`.

### Tree: drag-drop (tree-drop)

```
Drag component → drop on target node
               → _handleTreeDrop(sourcePath, dropTarget)
               → _applyEdit('tree', () => {
                   component.moveToIndex(dropTarget.parent, dropTarget.index)
                 })
               → _syncViews('tree')
```

The tree component (`builder-tree.ts:473-485`) fires `tree-drop` events with
`sourcePath` and a `dropTarget` object (computed by `computeDropTarget`). The
shell resolves the source component and calls `ComponentNode.moveToIndex()` or
`ComponentNode.moveToSlot()` depending on the drop target type.

### Properties: field change

```
Field edit → source.onChange(field, value)
           → _applyEdit('properties', () => node.setProperty(field, value))
           → _syncViews('properties')
             → _diffPatchEditor (cursor preserved)
             → _syncTree (label might change)
             → _syncProperties (re-renders with live model getters)
             → _syncPreview (visual updates)
             → _emitChange()
             → requestUpdate()
```

`_syncProperties` IS called even though properties is the origin. This is safe
and intentional: `source.data` is a getter that reads from the live document,
so re-rendering reflects the committed mutation. No circular sync risk — the
render reads, it doesn't write.

### Palette: component select

```
Click tile → _applyEdit('palette', () => {
               target.addComponent(entry.type, entry.defaultProps)
             })
           → _syncViews('palette')
             → _diffPatchEditor
             → _syncTree (new component node)
             → _syncProperties (select new component)
             → _syncPreview
             → _emitChange()
             → requestUpdate()
```

### Editor: keystroke

```
Keystroke → CodeMirror updates text immediately (user sees change)
         → _handleEditorInput (starts/resets 300ms debounce)
         → ... user keeps typing, debounce resets ...
         → 300ms idle → _applyEdit('editor', () => parse + swap document)
           → PageDocument.parse(text)
             → success:
               → _syncViews('editor')
                 → editor SKIPPED (origin)
                 → _syncTree (structure may have changed)
                 → _syncProperties (values may have changed)
                 → _syncPreview (visual updates)
                 → _emitChange()
                 → requestUpdate()
             → failure: no-op (invalid YAML, model stays at last valid)
```

### Preview: click/double-click

```
Click → find component at click target
      → _handleNodeSelect (NO mutation)
      → selection-only updates
```

## Edge Cases

### E1: Structured edit during active typing

User types in editor, then clicks a palette tile before debounce fires.

1. `_applyEdit('palette', ...)` is called
2. `_flushEditorSync()` runs — checks diagnostics on parsed text. If valid:
   pushes undo snapshot #1, swaps document to reflect user's pending text.
   No `_syncViews` (that's the caller's job).
3. Outer `_applyEdit` pushes undo snapshot #2 (state after flush)
4. Palette mutation executes on the flushed document
5. Single `_syncViews('palette')` — includes `_emitChange()` and `requestUpdate()`
6. Result: two undo entries, one sync pass. Undo #1 reverts the palette insert.
   Undo #2 reverts the flushed editor text.
7. If flush fails (diagnostics contain errors): no snapshot #1 pushed, palette
   operates on last valid model, editor text is overwritten with model+insert
   result. `_editorDirty` stays true — next valid editor input will flush.

### E2: Rapid edits from different sources

User changes a property, then immediately clicks +Page.

Both go through `_applyEdit`. Since JS is single-threaded, they execute
sequentially. Each pushes an undo snapshot. Each triggers `_syncViews`.
No race condition possible.

### E3: Editor text becomes valid after being invalid

User types invalid YAML → lint markers appear, model stays stale.
User fixes the error → next debounce fires → parse succeeds → model updates →
tree/props/preview update to match.

The transition from invalid to valid is seamless. No special handling needed.

### E4: Document toString() produces different formatting than editor text

After a structured edit, `document.toString()` may format differently than
what the user typed (different quoting, different whitespace). The diff-patch
approach handles this gracefully — only changed lines are replaced, and cursor
on unchanged lines is preserved.

For lines that ARE changed by formatting normalization, the cursor may shift.
This is acceptable — the user didn't edit those lines, so cursor position
there is coincidental.

### E5: Undo after editor edit vs. structured edit

Undo always pops the last snapshot, regardless of origin. If the last edit was
a keystroke sequence (debounced), undo reverts to the state before that
sequence. If the last edit was a palette insert, undo removes the insert.

The user doesn't need to know which undo stack entry came from where — undo
is undo.

## What Gets Removed

- `YamlSync` class — replaced by `_handleEditorInput` + `_flushEditorSync`
  in the shell
- `_subscribeToDocument` method — fully removed. In the new architecture,
  `_syncViews` handles `_emitChange` and `requestUpdate` internally. All
  code paths (`_applyEdit`, `_undo`, `_redo`, `willUpdate`) call `_syncViews`
  to get the full post-mutation sequence. No onChange listener on PageDocument
  is needed. All calls to `_subscribeToDocument` in `_undo()`, `_redo()`,
  `_flushEditorSync()`, and `willUpdate()` are removed.
- `_pushYamlToEditor` — replaced by `_diffPatchEditor` in `_syncViews`
- `_docUnsub` field — no listener to unsubscribe
- The render template binds toolbar button state to the shell's stacks:
  `?disabled="${this._undoStack.length === 0}"` (currently
  `?disabled="${!this._document.canUndo()}"`)
- `PageDocument.undo()` / `PageDocument.redo()` / `PageDocument.canUndo()` /
  `PageDocument.canRedo()` — public API removed. Replaced by shell-managed
  string snapshot stacks.
- `PageDocument._undoStack` and `_redoStack` — **private fields kept**.
  In coordinated mode, `_pushUndo` is a no-op so they stay empty. In
  standalone mode, they continue to function for internal undo tracking.
  Removing them would crash `_pushUndoInternal()` (called from 25 mutation
  methods across 6 node classes) and `abortTransaction()` (called from 4
  compound operations: `wrapInRow`, `replaceWith`, `moveToSlot`, `wrapIn`).
- PageDocument's internal `_pushUndo()` / `_pushUndoInternal()` and
  `_notify()` / `_notifyInternal()` become no-ops via the `_coordinated`
  flag (see §Coordinated Mode). The methods remain in PageDocument for
  standalone use (tests, non-shell consumers); they are suppressed, not
  deleted.

## What Gets Preserved

- **`_syncSource` flag** — this is a selection-sync mechanism, orthogonal to
  `EditOrigin`. `_syncSource` (`'tree' | 'yaml' | 'visual' | null`) tracks
  which panel initiated a *selection* (click/cursor move), so
  `_updatePropertySource` knows whether to scroll or highlight the YAML
  editor. `EditOrigin` tracks which panel initiated a *mutation*, so
  `_syncViews` knows what to skip. These solve different problems:
  - Selection sync: "I clicked in the preview — don't scroll the editor to
    where I already am"
  - Edit sync: "The editor changed the YAML — don't push YAML back to the
    editor"
  `EditOrigin` doesn't include `'visual'` (preview clicks aren't edits).
  `_syncSource` doesn't include `'properties'` or `'palette'` (those trigger
  mutations, not selections). Both mechanisms coexist.

- **`PageDocument` transaction API** — `beginTransaction()` /
  `commitTransaction()` / `abortTransaction()` remain, simplified by the
  `_coordinated` flag (see §Coordinated Mode). In coordinated mode:
  - `beginTransaction()` skips `_undoStack.push()` and `_redoStack` clear
  - `commitTransaction()` skips notification
  - `abortTransaction()` skips `_undoStack.pop()` and `_redoStack` clear
    (symmetry: nothing was pushed, so nothing to pop)
  - All three still manage `_inTransaction` and `_transactionSnapshot` — the
    atomicity and rollback mechanism is fully functional.

  In standalone mode (tests, non-shell consumers), the full original
  behavior — undo push, notification coalescing, rollback — is preserved.

  Methods using transactions: `PageNode.wrapInRow`, `ComponentNode.replaceWith`,
  `ComponentNode.moveToSlot`, `ComponentNode.wrapIn`.

## What Gets Added

- `_applyEdit(origin, fn)` — single coordinator method
- `_syncViews(origin)` — fan-out with origin check
- `_diffPatchEditor()` — minimal diff text sync
- `computeMinimalChanges(before, after)` — line-based diff utility
- `_handleEditorInput()` / `_flushEditorSync()` — debounced editor→model
- `_handleTreeAction(e)` — context menu handler (wires all 11 tree actions)
- `_handleTreeDrop(e)` — drag-drop handler (wires `tree-drop` events)
- `PageDocument.parseCoordinated(yaml)` — factory returning a coordinated instance
- `_parseDocument(yaml)` — private shell helper, delegates to `PageDocument.parseCoordinated()`
- Shell-managed undo/redo stacks

## Migration Path

Incremental, not big bang. Each step is independently testable.

1. **Add `_applyEdit` coordinator** — wrap all existing mutation sites in
   `_applyEdit`. This includes 7 onChange closures in `_updatePropertySource`
   (page name, column span, component properties, dataset properties, nav-item
   properties), plus `_addPage`, `_addDataset`, and `_handleComponentSelect`.
   Each wrapping is mechanical: `this._applyEdit(origin, () => { existing code })`.
   Keep existing sync paths as a fallback during migration. Tests: coordinator
   is called for every mutation.

2. **Add `_syncViews` with origin** — replace individual sync calls with
   single fan-out. Move `_emitChange()` and `requestUpdate()` into `_syncViews`.
   Tests: editor is skipped when origin is `'editor'`; tree, properties,
   and preview always sync regardless of origin.

3. **Replace `_pushYamlToEditor` with `_diffPatchEditor`** — cursor
   preservation. Tests: cursor stays after property change.

4. **Add coordinated mode to PageDocument** — add `_coordinated` flag and
   `parseCoordinated(yaml)` factory method. When coordinated:
   `_pushUndo`/`_pushUndoInternal` and `_notify`/`_notifyInternal` become
   no-ops; `beginTransaction` skips undo push; `commitTransaction` skips
   notification. Shell uses `parseCoordinated()` for every document it
   creates. Tests: no internal undo entries, no mid-pipeline notifications
   when coordinated.

5. **Remove `_subscribeToDocument`** — the coordinated mode flag makes the
   onChange listener inert. Remove the method, the `_docUnsub` field, and all
   call sites (`willUpdate`, `_undo`, `_redo`, `_flushEditorSync`). The tree
   component's own `_subscribeToDocument` also becomes inert — tree rebuilds
   are driven by `_syncTree` from the shell. Tests: no listener registration,
   no `_docUnsub`.

6. **Replace YamlSync with inline editor handler** — debounced parse in shell
   using diagnostics check (not try/catch). Tests: editor changes flow through
   `_applyEdit('editor', ...)`, invalid YAML leaves model untouched.

7. **Wire tree-action and tree-drop** — handle all 11 context menu actions
   and drag-drop moves. Add `@tree-action` and `@tree-drop` listeners to the
   `_syncTree` template. Tests: delete, move-up, move-down, duplicate,
   wrap-row, wrap-column, wrap-tabs, replace-with, drag-drop reorder.

8. **Shell-managed undo** — replace PageDocument undo with string snapshots.
   Remove public API: `PageDocument.undo()`, `redo()`, `canUndo()`,
   `canRedo()`. Keep private fields `_undoStack`/`_redoStack` (needed by
   standalone mode and `abortTransaction()`). Update toolbar button bindings
   to use shell stack state. Tests: undo/redo across all edit origins,
   `abortTransaction()` rollback in coordinated mode.

## References

- `packages/pages-builder/src/shell/builder-shell.ts` — current shell impl
- `packages/pages-builder/src/shell/yaml-sync.ts` — current bidirectional sync
- `packages/pages-builder/src/tree/builder-tree.ts:389` — tree-add event
- `packages/pages-builder/src/tree/tree-context-menu.ts` — menu items
- `packages/pages-document/src/page-document.ts` — document model
