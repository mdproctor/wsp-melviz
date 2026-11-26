# Migrate playbook-split-view to pages-three-way-viewer

**Issue:** casehubio/casehub-pages#749
**Date:** 2026-11-26

## Summary

Replace PlaybookSplitView's inline layout, splitter, and debounce code with
ThreeWayViewer composition. PagesStepViewer implements ViewerSyncTarget for
bidirectional cursor↔step sync. PlaybookSplitView drops ~100 lines, keeping
only the parseAndResolve wiring.

## Architecture

```
PlaybookSplitView (thin orchestrator, ~25 lines)
  └─ <pages-three-way-viewer language="yaml" .value=${...}>
       └─ <pages-step-viewer slot="visual-pane"
            .steps=${...}
            .positionInfos=${...}
            .error=${...}>
```

### Data flow

1. User edits YAML in ThreeWayViewer's built-in CodeMirror editor
2. ThreeWayViewer fires `value-change` with `{ value: string }`
3. PlaybookSplitView listens, calls `parseAndResolve(value)` → `{ steps, positionInfos, error }`
4. PlaybookSplitView sets `.steps`, `.positionInfos`, `.error` on PagesStepViewer

### Cursor sync (editor → step)

1. User moves cursor in CodeMirror
2. ThreeWayViewer calls `syncLine(lineNumber)` on the slotted visual element
3. PagesStepViewer.syncLine maps line → step address via `StepPositionScanner.findByLine`
4. PagesStepViewer calls its own `setActiveStep(address)` to highlight the step

### Step click sync (step → editor)

1. User clicks a step card in PagesStepViewer
2. PagesStepViewer maps the step's address → line via `StepPositionScanner.findByAddress`
3. PagesStepViewer fires `viewer-select` with `{ line, endLine }` (the ThreeWayViewer sync protocol event)
4. ThreeWayViewer handles `viewer-select` and scrolls the editor to that line

## Changes

### PagesStepViewer — new capabilities

**New properties:**
- `positionInfos: StepLineInfo[]` — reactive property, default `[]`. Set by the consumer after parsing.
- `error: string | null` — reactive property, default `null`. When set, renders an error banner instead of step cards.

**New method:**
- `syncLine(line: number): void` — implements `ViewerSyncTarget`. Uses `StepPositionScanner.findByLine(this.positionInfos, line)` to find the step, then calls `this.setActiveStep(info?.address ?? null)`.

**Modified behaviour:**
- On `step-click`: if `positionInfos` is non-empty, also dispatch `viewer-select` CustomEvent with `{ line: info.line, endLine: info.endLine }` (bubbles, composed). The existing `step-click` event continues to fire unchanged.
- Render: when `error` is set, show error banner. When `steps` is empty and no error, show empty state message. Otherwise render step cards as before.

**Error banner styles:** moved from PlaybookSplitView's CSS into PagesStepViewer. Same visual treatment (monospace, red border, pre-wrap).

### PlaybookSplitView — thinned to orchestrator

**Removed (~100 lines):**
- All layout CSS (`:host` flex, `.left-panel`, `.right-panel`, `.splitter`, `.error-banner`, `.empty-state`, `::slotted`)
- `_onSplitterPointerDown` and all pointer-capture splitter logic
- Manual render with left/right panels and splitter div
- `_debounceTimer` and debounced `updateYaml` (ThreeWayViewer handles its own input events)

**Kept:**
- `parseAndResolve()` function — exported, playbook-specific parsing logic (Walker.resolve, StepPositionScanner.scan, error handling)
- `StepSelectDetail` and `SplitViewState` type exports — public API

**New render:**
```typescript
render() {
  return html`
    <pages-three-way-viewer
      language="yaml"
      .value=${this.value}
      @value-change=${this._onValueChange}>
      <pages-step-viewer slot="visual-pane"
        .steps=${this._steps}
        .positionInfos=${this._positionInfos}
        .error=${this._parseError}>
      </pages-step-viewer>
    </pages-three-way-viewer>
  `;
}
```

**Public API changes:**
- `updateYaml(yaml)` → sets `.value` on the three-way-viewer (backward compat preserved)
- `syncCursorLine(line)` → delegates to PagesStepViewer's `syncLine` (backward compat preserved, though ThreeWayViewer handles this automatically now)
- `splitRatio` property → passed through to ThreeWayViewer's `split-ratio` attribute

### Gallery update

Section 33 updated: replace `<playbook-split-view>` + raw `<textarea>` with the new thin component. The textarea wiring (input → updateYaml, click/keyup → syncCursorLine, step-select → cursor) is no longer needed — ThreeWayViewer's built-in CodeMirror editor and the ViewerSyncTarget protocol handle it.

### Package dependency

`pages-step-viewer` gains a dependency on `@casehubio/pages-three-way-viewer` in its `package.json`.

### Exports

`pages-step-viewer/src/index.ts` — no changes needed. `PlaybookSplitView`, `parseAndResolve`, `StepSelectDetail`, `SplitViewState` continue to be exported.

## Testing

- Existing `playbook-split-view.test.ts` tests for `parseAndResolve` are unaffected (pure function, no changes)
- New unit tests for PagesStepViewer:
  - `syncLine` with valid positionInfos highlights the correct step
  - `syncLine` with line outside any step clears the active step
  - `syncLine` with empty positionInfos is a no-op
  - Step click with positionInfos dispatches `viewer-select` with correct line/endLine
  - Step click without positionInfos does not dispatch `viewer-select`
  - Error property renders error banner, hides step cards
  - Empty steps with no error renders empty state

## References

- `packages/pages-three-way-viewer/src/sync-protocol.ts` — ViewerSyncTarget interface
- `packages/pages-three-way-viewer/src/three-way-viewer.ts:149-153` — slotted element sync target lookup
- `packages/pages-three-way-viewer/src/three-way-viewer.ts:179-189` — viewer-select handler
- `packages/pages-step-viewer/src/playbook-split-view.ts` — current implementation being replaced
- `packages/pages-step-viewer/src/pages-step-viewer.ts:30` — existing setActiveStep method
- `casehubio/casehub-pages#749` — issue
- `casehubio/examples#217` — originating three-way-viewer work
- `decisions.md D1` — ViewerSyncTarget placement decision
