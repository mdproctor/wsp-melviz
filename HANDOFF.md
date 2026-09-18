# Session Handover

**Branch:** `issue-433-structural-editing` (30 commits ahead of main)
**Issue:** #433 — Tree/visual structural editing
**Date:** 2026-09-18

## What Was Built

### Clipboard Foundation
- `BuilderClipboard` singleton with subscribe/notify
- YAML fragment serialization and type detection

### Tree Structural Editing (Complete)
- 4 inline buttons per node: Add (+), Insert (↓), Cut (✂), Copy (⎘)
- Insert flow: Before/After picker → filtered component picker → schema-compliant insertion
- Cut/Copy serialize to clipboard, Insert mode highlights valid targets
- Keyboard shortcuts: Ctrl+X, Ctrl+C, Escape
- Auto-expand and auto-select new nodes, correct nodeType via classifyPath

### Visual Preview
- Component overlay engine, scope highlight in overlay-root
- composedPath for shadow DOM click traversal
- Async renderPreview awaited before highlighting
- Double-click navigates up parent chain one level at a time

### 289 tests passing across 17 test files

## Design Decision — Next Session

**Visual toolbar should be selection-based, not hover-based:**
- Buttons appear ONLY on the selected container's blue outline (top-right)
- No hover noise — buttons follow the selection, not the cursor
- Double-click refinement (metric → column → row → page) gives copy/paste precision
- User agreed this is the right direction

## What's Next

| Priority | Item | Scale |
|----------|------|-------|
| 1 | Visual toolbar redesign — selection-based | M |
| 2 | Wire delete button in overlay | S |
| 3 | Wire paste execution for Insert flow | S |
| 4 | CodeMirror paste gutter indicators | M |
