# Handover — issue-443-scenario-tutorials

**Branch:** `issue-443-scenario-tutorials` (pages + workspace)
**Issue:** casehubio/casehub-pages#443
**State:** Batches 1-2 complete (4 of 6 tasks). Batch 3 (content rework + host app) remains.

## What happened

Pivoted tutorial delivery from the custom `yaml-editor` content type (#435) to scenario-driven `hands-on` tutorials that use the existing scenario executor infrastructure. Built the editor automation layer:

1. `ScenarioEditableText` SPI — DOM-level interface for text editor manipulation, Symbol-based discovery with shadow DOM ancestor walk.
2. `CodeEditorBridge` — CM6 implementation in `pages-code-editor`, Position-to-offset conversion, highlight decorations via StateField.
3. Execution path unification — replaced inline `executeAriaStep` in `sectioned-runner.ts` with delegation to `executeStep` in `command-executor.ts` (full ARIA tree walker).
4. Seven new ARIA actions in parser + executor: `editor-insert`, `editor-replace`, `editor-delete`, `editor-set-content`, `editor-cursor`, `editor-highlight`, `editor-completion`. Plus `spotlight` added to parser. Progressive typing reuses `progressiveFill` algorithm through SPI.

## Next action

Execute Batch 3 — Task 5 (rewrite 15-step tutorial from yaml-editor sections to hands-on ARIA steps) and Task 6 (tutorial host app layout with builder-shell as scenario target).

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-443-scenario-tutorials/2026-09-14-scenario-driven-tutorials-design.md` |
| Decisions | `specs/issue-443-scenario-tutorials/decisions.md` |
| Implementation plan | `plans/2026-09-14-scenario-driven-tutorials.md` |
| Diary | `blog/2026-09-14-mdp04-teaching-by-typing.md` |
| SPI interface | `packages/pages-aria/src/executor/editable-text.ts` |
| CM6 bridge | `packages/pages-code-editor/src/code-editor-bridge.ts` |
| Executor dispatch | `packages/pages-aria/src/executor/command-executor.ts` |
| Parser | `packages/pages-aria/src/scenario/parser.ts` |
| Sectioned runner | `packages/pages-aria/src/scenario/sectioned-runner.ts` |
| Tutorial content (to rework) | `tutorials/yaml-composition/tutorial.yaml` |
