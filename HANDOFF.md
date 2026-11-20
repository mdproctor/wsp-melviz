# HANDOFF — casehub-pages (slot 224)

## Last Session: 2026-11-20

### Workspace fix
Slot 224's `pages/wksp` symlink was pointing to `wsp-melviz` (canonical's workspace), causing the work pipeline to read canonical's queue instead of the slot's. Created `wsp-casehub-pages` as a proper slot-local workspace with `local` remote to canonical, and repointed the symlink.

### Completed
- **#716** — Category-level playbook execution tests. 15 behavioral `.test.yaml` tests across 5 categories (flow-control, concurrency-patterns, coordination-primitives, step-workflows, composition). Wired via `describeYamlTests`. Data-delivery omitted — test executor doesn't propagate outer services.
- **#732** — Concurrency parallel decorator. Ported platform's `concurrency` limiter: `concurrency: N` on parallel steps limits simultaneous branch execution via promise-queue gate. Added to RESERVED_KEYS, DECORATOR_KEYS, ParallelStep type, walker parsing, and evaluator.
- **#715** — Already verified and closed in #707 work. Marked complete in queue.

### Parity audit
Ran full playbook parity audit (platform vs TS). Filed 5 remaining gaps:

| Order | Issue | Title | Scale |
|-------|-------|-------|-------|
| 1 | #734 | Reject `steps:` wrapper in parallel map form | XS |
| 2 | #738 | Remove stale `concurrent` keyword | XS |
| 3 | #736 | Reject `steps:` wrapper in playbook documents | XS |
| 4 | #737 | Default `advance-after` to all branches | XS |
| 5 | #735 | Migrate test YAML to multi-document format | S |

All queued in `wsp-casehub-pages/.queue`.

## Immediate Next Step

`work start` — picks up #734 (Reject steps: wrapper in parallel map form).

## State

- Both repos on `main`, synced with canonical
- Queue: 5 items (#734, #738, #736, #737, #735)
- All published to remote

## Key Files

- `packages/yaml-core/src/step/walker.ts` — step parsing, keywords, types
- `packages/yaml-core/src/step/structural-evaluator.ts` — step execution
- `packages/yaml-core/src/test/fixtures/categories/` — category test fixtures
- `packages/yaml-core/src/test/category-execution.test.ts` — vitest wiring
