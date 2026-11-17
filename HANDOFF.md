# HANDOFF — casehub-pages (slot 218)

## Status: CLOSED — ready for archive

All work in this slot is complete and landed on main.

## Completed Issues

### #390 — Scenario format refinements (TS side)
Extracted dispatch protocol types to `playbook/types.ts`. Renamed CommandPayload.target to element, added DispatchStep.delay, made speed opt-in, added REST action definition. Landed on main.

### #531 — Inferred method-call dispatch for playbook steps
Added secure inferred dispatch so unrecognised step names resolve as method calls on target elements (TS) or beans (Java). Key deliverables:
- `PLAYBOOK_ACTIONS` Symbol SPI in pages-primitives
- `Catalog.resolveInferred()` in yaml-core (Walker accepts unknown step keys)
- Secure `invokePlaybookMethod` replacing raw `invokeMethod` in command-executor.ts
- `InferredMethodHandler` + catalog factory wiring in pages-aria
- `playbook-handler.ts` default case extended for server-dispatched path
- `pages-document-diff` as first adopter (9 methods declared)
- `@PlaybookInterface` annotation + `ActionRegistry` fallback on Java side
- Post-audit fix: `ElementMethodBinding` export, `executeAriaAction` made async
Landed on main, issue closed.

## Design Artifacts (workspace)

- `specs/issue-531-inferred-method-dispatch/` — design spec, decisions, pipeline state
- `plans/2026-10-20-inferred-method-dispatch.md` — implementation plan
- `specs/issue-571-decorator-structural-keys/` — prior work spec
- `plans/2026-10-20-scenario-format-refinements.md` — prior work plan

## References

- `packages/pages-primitives/src/playbook/` — PLAYBOOK_ACTIONS SPI
- `packages/pages-aria/src/executor/playbook-resolve.ts` — shared dispatch utilities
- `packages/pages-aria/src/playbook/invoke/inferred-method-handler.ts` — InferredMethodHandler
- `packages/pages-aria/src/playbook/catalog-factory.ts` — resolveInferred on catalog
- `packages/pages-document-diff/src/pages-document-diff.ts` — first PLAYBOOK_ACTIONS adopter
- `backend/playbook-client/src/main/java/.../PlaybookInterface.java` — Java annotation
- `backend/playbook-client/src/main/java/.../ActionRegistry.java` — interface scanning fallback

# Session Handover

**Branch:** `main` (issue-437-lsp4ij-completions closed)
**Issue:** #437 — fix(intellij): LSP4IJ not delivering completions
**Date:** 2026-09-14

## What happened

Debugged and fixed LSP4IJ completion delivery for the IntelliJ plugin. Root cause was a dual-plugin conflict: `io.casehub.pages` (CaseHub Pages, from pages repo) and `io.casehub.yaml` (CaseHub YAML, from blocks-ui repo) were both installed, both registering LSP servers for the same YAML file patterns. LSP4IJ couldn't route documents with two competing servers.

Secondary issues fixed: stale bundle cache (extractServer never re-extracted), TextDocumentSync bare number form (LSP4IJ needs object form with `openClose: true`), missing Node.js macOS fallback paths in blocks-ui plugin, CompletionWeigher for YAML `{}` item deprioritization.

Verified end-to-end: file-based logging at `/tmp/casehub-lsp.log` confirms initialize → didOpen → completion handshake completes. Page completions appear in IntelliJ.

## Decisions / gotchas

- **Two plugins must never coexist.** CaseHub Pages (`io.casehub.pages`) and CaseHub YAML (`io.casehub.yaml`) have different plugin IDs but claim the same files. IntelliJ treats them as independent plugins. The rename from `io.casehub.yaml` → `io.casehub.pages` left the old installation behind.
- CaseHub YAML is the superset plugin (all 5 formats) but its bundle build fails — `.casehub-packages` in blocks-ui is stale (missing `lookupSchema`, `externalDataSetDefSchema` from pages-data).
- Currently CaseHub YAML is installed (without CaseHub Pages). It uses a pages-lsp bundle with Page schemas only — SWF/Case/HTN/Org return empty completions.
- Indentation bug: completion selection inserts text at column 0, losing YAML context indentation.
- File-based diagnostic logging (`/tmp/casehub-lsp.log`) is committed to pages main — remove after debugging is complete.

## Follow-up (3 items for next session)

1. **Sync `.casehub-packages` in blocks-ui** — rebuild from current pages source so lsp-schemas bundle builds. Then rebuild + reinstall CaseHub YAML with domain schemas.
2. **Fix indentation bug** — completion insertText doesn't preserve YAML indent context.
3. **Apply pages fixes to blocks-ui plugin** — TextDocumentSync object form, serverInfo, CompletionWeigher, stale cache removal. The blocks-ui `CaseHubLspServerDescriptor.kt` already has Node.js fallback paths and stale cache fix from this session.

## References

| Artifact | Path |
|----------|------|
| Diary | `blog/2026-09-14-mdp01-lsp4ij-silent-server.md` |
| Garden: CompletionWeigher | `GE-20260914-54f581` |
| Garden: TextDocumentSync | `GE-20260914-330473` |
| Build integration issue | #438 |
| Diagnostic log | `/tmp/casehub-lsp.log` |
| blocks-ui plugin | `blocks-ui/plugins/intellij-casehub/` |
| pages plugin | `pages/plugins/intellij/` |
