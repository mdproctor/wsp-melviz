# HANDOFF — casehub-pages

## Last Session (2026-09-30)

**Branch:** on main (two branches landed this session)
**Next issue:** #507 — unify scenario execution with platform yaml-core declaration model
**Status:** Both #506 and fix/gallery-catalog-rename landed. Ready to start #507.

### What was done

**#506 work-end (feat/506-unified-step-catalog):**
- Completed Batches 3-4 (multi-source catalog merge, portability badges/filters, pre-flight validation)
- Full work-end: code review clean, 4-dimension audit clean, squashed 12→8 commits
- Landed on main, #506 closed, 976/976 tests pass

**fix/gallery-catalog-rename (ceremony elimination):**
- Gallery catalog sample: renamed `<pages-step-catalog>` → `<pages-action-catalog>`, added portability mock data
- Webpack: fixed sideEffects field that was silently tree-shaking all customElements.define calls
- Walker: `steps` and `do` now rejected with fail-fast error
- Walker: select branches and match cases use inline sibling keys — no wrapper ceremony
- Gallery: removed all `steps:` and `do:` ceremony from 7 sample files
- Import schema: renamed `steps` → `actions` field across type, schema, expanders, tests
- casehub-entry.ts: fixed remaining Step-prefixed type exports

### Pending work across repos

**casehubio/casehub-pages#508 — comprehensive unification drift epic:**
- Scenario document `steps:` (22 occurrences in 5 gallery files) — blocked on front matter format
- Front matter format: YAML `---` separator for scenario metadata/actions (design decided, not implemented)
- Spec docs: 2 historical files with old names (annotate or update)

**casehubio/platform#496 — Java StepWalker alignment (BLOCKED — platform busy):**
- Inline action resolution in `resolveMatchCases()` and `resolveSelectBranches()` (strip config keys, resolve rest)
- Reject both `steps` and `do` as keys
- Type rename: `StepWalker` → `Walker`, `StepDefinitionParser` → `DefinitionParser`, `YamlStepDefinitionSource` → `YamlDefinitionSource`
- Note: `work.progress.StepDefinition` is a different domain — assess separately

**casehubio/casehub-pages#507 — next up:**
- Unify `ScenarioStep`/`ScenarioParser`/`ScenarioExecutor` with yaml-core's `Declaration`/`Walker`/`StructuralEvaluator`
- Pages scenarios gain control flow (if/match/forEach/loop/parallel/block) for free
- Only legitimate runtime differences: Aria binding (browser-only), MCP binding (Java-only)

### Design decisions made this session

- `then:`/`else:`/`catch:`/`finally:`/`cases:` are semantic branch labels — they stay
- `block:`/`parallel:`/`try:` take bare arrays — no wrapper needed
- `wait:`/`subscribe:`/`pattern:`/`when:` — actions are inline sibling keys; multiple actions use `block:`
- Standalone action lists are bare arrays — no top-level wrapper
- Scenario documents can use YAML front matter (`---`) to separate metadata from actions (not yet implemented)
- Scenario vs playbook: same format and runtime, different intent (demo vs production workflow)
- YAML mappings can have multiple keys — a `wait:` branch's child action is a sibling key, not nested under a wrapper

### Resume command

```
work start #507
```

### Test state

639/639 yaml-core tests pass. 344/344 pages-aria tests pass.

### Known issues
- IntelliJ heap pressure: file rename operations can time out with many projects open
- Gallery sample fetch mock conflict (pre-existing from #501)
- Pre-existing Java compilation errors in `backend/scenario-runtime` test classes
- Platform #496 blocked until platform repo is free

### Cross-repo context
- platform#483 landed — Java API contract is stable
- platform#496 filed — Java Walker alignment (blocked, waiting for platform availability)
- casehub-pages#508 — unification drift tracking epic
- Parent epic: casehubio/fsitrading#50 (Trading YAML Playbooks)
- casehub-pages#502 — local epic grouping scenario infrastructure issues

### Garden entries captured
- `GE-20260930-1cdc32` — webpack sideEffects silently kills customElements.define
- `GE-20260930-b8ebec` — YAML sibling keys eliminate wrapper ceremony

## References

- `packages/yaml-core/src/step/walker.ts` — Walker with inline action resolution, REMOVED_KEYS fail-fast
- `packages/yaml-core/src/step/` — types.ts, portability.ts, definition-parser.ts, plugin-registry.ts, catalog.ts
- `packages/pages-aria/src/controller/step-catalog.ts` — PagesActionCatalog (multi-source, badges, portability pre-check)
- `packages/pages-aria/src/controller/catalog-data-source.ts` — CatalogDataSource + 3 implementations
- `packages/pages-aria/src/server/catalog-execute-handler.ts` — execution handler with portability validation
- `examples/samples/Scenarios/` — gallery samples (ceremony removed)
- `examples/src/casehub-entry.ts` — gallery entry point (all types renamed)
