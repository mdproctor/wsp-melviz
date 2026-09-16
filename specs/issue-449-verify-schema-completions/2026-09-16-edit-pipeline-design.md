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
   skips the origin view — you never write back to the thing that just changed.

3. **No circular sync.** A → B → A loops are eliminated by construction. The
   origin check makes them impossible.

4. **Cursor preservation.** Editor updates use minimal text diffs, not full
   replacement. The cursor stays where the user left it.

5. **Error tolerance.** Invalid YAML during typing is normal. The model stays at
   the last valid state. The editor shows the user's in-progress text. No crashes.

6. **Single coordinator.** One method (`_applyEdit`) is the only path from edit
   sources to view sync. No shortcuts, no parallel paths.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                       BuilderShell                           │
│                                                              │
│  Edit Sources ──────────────► _applyEdit(origin, fn) ────┐   │
│  ┌─────────────────────────┐          │                  │   │
│  │ Toolbar:  +Page, Undo   │          │ 1. Flush pending │   │
│  │ Tree:     +, ctx menu   │          │    editor debounce│  │
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
  //    changes, flush them first so we don't lose in-progress text
  if (origin !== 'editor' && this._pendingEditorSync) {
    this._flushEditorSync();
  }

  // 2. Snapshot for undo (stores YAML string, ~2KB per snapshot)
  this._undoStack.push(this._document.toString());
  this._redoStack.length = 0;

  // 3. Execute the mutation
  fn();

  // 4. Sync all views, skipping origin
  this._syncViews(origin);
}
```

**Undo granularity for editor edits:** The editor debounce fires after 300ms of
idle time. Each debounce flush pushes one undo snapshot. Continuous typing
produces ~1 snapshot per pause. This matches user expectations — undo reverts
to the last pause point, not each character.

**Flush-before-structured-edit:** When the user is typing in the editor and then
clicks a palette item, the pipeline flushes pending text changes to the model
first, THEN applies the palette insert. This preserves the user's typing. If
the pending text is invalid YAML, the flush fails silently and the structured
edit operates on the last valid model state.

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

### EditorInputHandler (replaces YamlSync for editor→model)

Handles editor text changes with debouncing and active-typing suppression.

```typescript
private _pendingEditorSync: number | undefined;
private _editorDirty = false;

private _handleEditorInput(): void {
  this._editorDirty = true;
  clearTimeout(this._pendingEditorSync);
  this._pendingEditorSync = window.setTimeout(() => {
    this._flushEditorSync();
  }, 300);
}

private _flushEditorSync(): void {
  clearTimeout(this._pendingEditorSync);
  this._pendingEditorSync = undefined;
  if (!this._editorDirty) return;

  const text = this._getEditorText();
  try {
    const newDoc = PageDocument.parse(text);
    this._applyEdit('editor', () => {
      this._document = newDoc;
      this._subscribeToDocument();
    });
    this._editorDirty = false;
  } catch {
    // Invalid YAML — model stays at last valid state.
    // Editor shows the user's text. Lint markers show errors.
  }
}
```

**Why PageDocument.parse (new instance) is acceptable here:** Editor text edits
can produce arbitrarily different YAML. An in-place diff-and-patch on the
model would be complex and fragile. Creating a new PageDocument from the text
is correct — the undo stack lives in the shell (string snapshots), not in the
PageDocument instance.

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
  this._document = PageDocument.parse(prev);
  this._subscribeToDocument();
  this._syncViews('toolbar');
}

private _redo(): void {
  if (this._redoStack.length === 0) return;
  this._undoStack.push(this._document.toString());
  const next = this._redoStack.pop()!;
  this._document = PageDocument.parse(next);
  this._subscribeToDocument();
  this._syncViews('toolbar');
}
```

Stack size limit: 100 entries. Each entry ~2KB. Total: ~200KB. Negligible.

## Edit Flows (all entry points)

### Toolbar: +Page, +Dataset

```
Button click → _applyEdit('toolbar', () => doc.addPage())
            → _syncViews('toolbar')
              → _diffPatchEditor (cursor preserved)
              → _syncTree (new page node appears)
              → _syncProperties (selection unchanged)
              → _syncPreview (new page rendered)
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
```

### Tree: context menu (tree-action)

```
Right-click → context menu → action selected
            → _handleTreeAction
            → _applyEdit('tree', () => {
                match action:
                  'delete' → doc.removeAt(path)
                  'move-up' → doc.moveUp(path)
                  'duplicate' → doc.duplicateAt(path)
                  'add-row' → page.addRow()
                  'add-column' → row.addColumn()
                  'add-child' → node.addComponent(...)
                  'wrap-row' → doc.wrapInRow(path)
              })
            → _syncViews('tree')
```

### Properties: field change

```
Field edit → source.onChange(field, value)
           → _applyEdit('properties', () => node.setProperty(field, value))
           → _syncViews('properties')
             → _diffPatchEditor (cursor preserved)
             → _syncTree (label might change)
             → _syncPreview (visual updates)
             → props NOT refreshed (origin)
```

Wait — properties IS the origin, but we still want to refresh the properties
panel because `source.data` is a getter that reads from the live document.
Since the mutation already happened, the getter returns updated data. The
property palette re-reads `source.data` on its next render cycle. No explicit
refresh needed.

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
```

### Editor: keystroke

```
Keystroke → CodeMirror updates text immediately (user sees change)
         → _handleEditorInput (starts/resets 300ms debounce)
         → ... user keeps typing, debounce resets ...
         → 300ms idle → _flushEditorSync
           → PageDocument.parse(text)
             → success: _applyEdit('editor', () => swap document)
               → _syncViews('editor')
                 → editor SKIPPED (origin)
                 → _syncTree (structure may have changed)
                 → _syncProperties (values may have changed)
                 → _syncPreview (visual updates)
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
2. Coordinator flushes pending editor sync first
3. If flush succeeds: model has user's text changes + palette insert
4. If flush fails (invalid YAML): palette insert operates on last valid model,
   editor text is overwritten with model+insert result
5. In case 4, the user's invalid text is lost — but the palette action they
   requested takes effect. Undo reverts to just before the palette click.

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
- `_subscribeToDocument` → `_pushYamlToEditor` path — replaced by
  `_syncViews` which calls `_diffPatchEditor`
- `_syncSource` flag — replaced by `EditOrigin` parameter
- `PageDocument.undo()` / `PageDocument.redo()` — replaced by shell-managed
  string snapshot stack

## What Gets Added

- `_applyEdit(origin, fn)` — single coordinator method
- `_syncViews(origin)` — fan-out with origin check
- `_diffPatchEditor()` — minimal diff text sync
- `computeMinimalChanges(before, after)` — line-based diff utility
- `_handleEditorInput()` / `_flushEditorSync()` — debounced editor→model
- `_handleTreeAction(e)` — context menu handler (wires all tree actions)
- Shell-managed undo/redo stacks

## Migration Path

Incremental, not big bang. Each step is independently testable.

1. **Add `_applyEdit` coordinator** — wrap all existing mutation sites in
   `_applyEdit`. Keep existing sync paths. Tests: coordinator is called for
   every mutation.

2. **Add `_syncViews` with origin** — replace individual sync calls with
   single fan-out. Tests: each origin skips its own view.

3. **Replace `_pushYamlToEditor` with `_diffPatchEditor`** — cursor
   preservation. Tests: cursor stays after property change.

4. **Remove dual write-back** — delete `_subscribeToDocument` → editor path
   and YamlSync's doc→editor path. Keep only `_syncViews`. Tests: no double
   updates.

5. **Replace YamlSync with inline editor handler** — debounced parse in shell.
   Tests: editor changes flow through `_applyEdit('editor', ...)`.

6. **Wire tree-action** — handle all context menu actions. Tests: delete,
   move, duplicate via tree.

7. **Shell-managed undo** — replace PageDocument undo with string snapshots.
   Tests: undo/redo across all edit origins.

## References

- `packages/pages-builder/src/shell/builder-shell.ts` — current shell impl
- `packages/pages-builder/src/shell/yaml-sync.ts` — current bidirectional sync
- `packages/pages-builder/src/tree/builder-tree.ts:389` — tree-add event
- `packages/pages-builder/src/tree/tree-context-menu.ts` — menu items
- `packages/pages-document/src/page-document.ts` — document model
