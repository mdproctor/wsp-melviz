# Design Decisions — Structural Editing (Tree + Visual)

## D1: Cut/Copy-Paste over Drag-and-Drop

**Choice:** Cut/copy-paste as the primary move mechanism, replacing drag-and-drop
**Alternatives:**
- Drag-and-drop — one gesture but hard to be precise in deep trees, source and target often not visible simultaneously
- Both DnD + cut/paste — more code surface, two ways to do the same thing
**Rationale:** YAML trees are deeply nested; source and target are frequently off-screen from each other. Two-step cut/paste lets you scroll between steps. Also supports copy (clone), which DnD doesn't naturally provide.
**Trade-offs:** Slightly more steps than DnD for nearby moves
**Sources:** `packages/pages-builder/src/tree/tree-dnd.ts`, issue #433
**Exploration:** quick
**Status:** captured

## D2: Tree item buttons — Add, Insert, Cut, Copy

**Choice:** Four inline buttons per tree node: Add (+), Insert (↓), Cut (✂), Copy (⎘). All opacity-hidden until hover.
**Alternatives:**
- Toolbar buttons — always visible but spatially disconnected from the node
- Context menu only — discoverable but slow
**Rationale:** Inline keeps actions close to the node. Fixed positions (no conditional rendering) avoid the "dancing" layout-shift problem. Opacity-hidden until hover keeps the tree clean.
**Trade-offs:** Four buttons per row is dense, but opacity-hidden mitigates clutter
**Sources:** Existing `+` button pattern in `builder-tree.ts`
**Exploration:** quick
**Status:** captured

## D3: Add vs Insert distinction

**Choice:** Two operations — Add (append child inside container) and Insert (sibling before/after)
**Alternatives:**
- Single + button with position strategy popup after selection
**Rationale:** "Add" and "Insert" are semantically different. Add is a container operation; Insert is a sibling operation. Separate buttons make intent clear without a sub-menu.
**Trade-offs:** Two buttons instead of one, but each has a clear single purpose
**Sources:** issue #433 §2 (position-complete insertion)
**Exploration:** quick
**Status:** captured

## D4: Dual-function Add/Insert — picker or paste depending on clipboard

**Choice:** Add/Insert check clipboard state. If YAML fragment present → paste. If empty → open component picker.
**Alternatives:**
- Separate Paste button that appears when clipboard has content
- Always open picker, paste is a separate action
**Rationale:** Reduces button count. Natural workflow: cut → navigate → insert. Visual indicators (highlighted targets, clipboard banner) make the state obvious.
**Trade-offs:** Hidden state changes button behavior, mitigated by visual indicators
**Sources:** Brainstorming session discussion
**Exploration:** quick
**Status:** captured

## D5: Global BuilderClipboard state

**Choice:** Global singleton state (`BuilderClipboard`) holding YAML fragment, fragment type, operation, and insert-mode flag. Any view subscribes.
**Alternatives:**
- Per-instance clipboard on each builder-shell
- System clipboard only (no metadata)
**Rationale:** Cross-file paste requires shared state. Any tree, visual overlay, or builder instance needs to detect clipboard content and show valid targets. System clipboard holds raw YAML text for cross-app paste; global state holds parsed metadata for guided in-app UX.
**Trade-offs:** Global mutable state, but clipboard is inherently global
**Sources:** Brainstorming session discussion
**Exploration:** quick
**Status:** captured

## D6: Insert mode and Esc behavior

**Choice:** Two independent states — clipboard content persists until overwritten; insert mode (highlighted targets) is transient, exited with Esc. Re-enter insert mode by clicking Add/Insert when clipboard has content.
**Alternatives:**
- No insert mode — just paste directly without guided targets
- Esc clears both clipboard and mode
**Rationale:** Guided valid-target highlighting is the key UX value. Separating clipboard from insert mode lets users browse freely after Esc while retaining the ability to re-enter paste mode. Clipboard indicator has ✕ for explicit clearing.
**Trade-offs:** Two states to manage, but they map to user intent cleanly
**Sources:** Brainstorming session discussion
**Exploration:** quick
**Status:** captured

## D7: YAML fragment serialization to system clipboard

**Choice:** Cut/Copy serializes the selected node as a YAML text fragment to the system clipboard. Paste parses the fragment and validates by type against the target position's schema.
**Alternatives:**
- Internal path reference (breaks across files/sessions)
- JSON serialization (not YAML-native)
**Rationale:** YAML text is portable across files, tabs, sessions, and even apps. Type-based validation (parse fragment, check what it is) is more robust than path-based validation. Cross-file paste comes for free.
**Trade-offs:** Requires parsing on paste, but YAML parsing is already fast in the codebase
**Sources:** `packages/pages-document/src/page-document.ts` — existing YAML parse infrastructure
**Exploration:** quick
**Status:** captured
