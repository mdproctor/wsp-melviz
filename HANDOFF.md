# Handover — issue-449-verify-schema-completions

## This Session

Wired schema-driven completions into the workbench text editor, added auto-indent
and smart backspace for YAML editing, fixed the tree view `+` button (was dispatching
`tree-add` into the void), and designed + reviewed the normalized edit pipeline spec.
Began implementation — Batch 1 (coordinated mode for PageDocument) is complete.

Key discovery: CodeMirror extensions silently fail when imported cross-package in a
monorepo due to duplicate `@codemirror/state` instances breaking `instanceof` checks.
Also: Vite serves workspace packages from `dist/` not `src/`, so `pages-code-editor`
changes need `yarn build` before the dev server picks them up.

## Resume Point

**Batch 2 of the edit pipeline plan** — Task 2 (`_applyEdit` coordinator + `_syncViews`).
Plan: `plans/2026-09-16-edit-pipeline.md`. Spec (reviewed, 0 unresolved):
`specs/issue-449-verify-schema-completions/2026-09-16-edit-pipeline-design.md`.

3 tree-add inline picker tests are RED (written but not implemented — deferred
in favour of the pipeline redesign that will provide the proper infrastructure).

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-449-verify-schema-completions/2026-09-16-edit-pipeline-design.md` |
| Implementation plan | `plans/2026-09-16-edit-pipeline.md` |
| Review workspaces | `~/reviews/casehub-slots/edit-pipeline-{coherence,structure,robustness,crosscutting}-*` |
| Garden entries | `GE-20260916-b8eab4` (CodeMirror instanceof), `GE-20260916-e52ecf` (Vite dist/) |
| Protocols | `PP-20260916-b6f3e8`, `PP-20260916-c2d406`, `PP-20260916-187ec0` |
| Test page | `test-completions/` (scratch — can be deleted) |
