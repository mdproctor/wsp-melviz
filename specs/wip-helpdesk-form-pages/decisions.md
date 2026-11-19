## D1: IntelliJ rendering approach

**Choice:** JCEF web component — reuse Lit step-viewer components in embedded browser
**Alternatives:**
- Native Swing/Kotlin — native look-and-feel but requires reimplementing 16 step viewer components
- JCEF now, native later — defers decision but adds migration cost
**Rationale:** One codebase, pixel-identical rendering in browser and IDE. JCEF is standard in modern IntelliJ plugins.
**Trade-offs:** JCEF adds ~50MB memory overhead per panel; startup slightly slower than native
**Sources:** `plugins/intellij/` — current LSP-only plugin structure
**Exploration:** quick
**Status:** captured

## D2: StepPositionScanner nesting support

**Choice:** Full hierarchical nesting from the start
**Alternatives:**
- Flat-only first — ship faster but incomplete experience for nested playbooks
- Flat + highlight parent — cheap approximation, covers basic cases
**Rationale:** Playbooks are inherently nested (if-else, try-catch, match, swim lanes). Flat-only sync would be misleading for real-world playbooks.
**Trade-offs:** More upfront work on the scanner; address format needs design
**Sources:** `yaml-core/src/annotation/step-position-scanner.ts` — current flat scanner
**Exploration:** quick
**Status:** captured

## D3: Split-view container package location

**Choice:** In pages-step-viewer alongside existing viewer components
**Alternatives:**
- New pages-playbook-editor package — cleaner separation but more packages
- In pages-editor-core — inverts the dependency (editor depends on viewer)
**Rationale:** The split-view is viewer-centric — the editor is an input mechanism. Keeps the playbook visualization story in one package.
**Trade-offs:** pages-step-viewer gains a dependency on pages-code-editor (or editor-core)
**Sources:** `packages/pages-step-viewer/` — existing component structure
**Exploration:** quick
**Status:** captured

## D4: IntelliJ IDE layout

**Choice:** Editor tab split (like IntelliJ's Markdown preview)
**Alternatives:**
- Tool window — persists across file switches but takes dedicated screen space
- Both — start with tab split, add tool window later
**Rationale:** Standard pattern that users expect — open a file, see editor + preview. FileEditorProvider is the established IntelliJ API for this.
**Trade-offs:** Preview disappears when switching to another file (tool window would persist)
**Sources:** `plugins/intellij/` — no existing FileEditorProvider; IntelliJ Markdown plugin as reference
**Exploration:** quick
**Status:** captured
