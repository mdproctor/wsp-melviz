# HANDOFF — casehub-pages

## Last Session

Scheduled work from fsitrading Epic C13 (casehubio/fsitrading#50) cross-repo
dependencies into the casehub-pages .plan. Four generic scenario/playbook
infrastructure issues, ordered by dependency chain:

1. **#501** — Step catalog browser (S / Low, independent)
2. **#498** — Scenario lifecycle state (M / Med, independent)
3. **#499** — Event-triggered scenario activation (M / Med, depends on #498)
4. **#500** — Scenario outcome tracking + CBR linkage (M / High, blocked on engine#1190)

### Ordering rationale
- #501 first: independent, smallest, quick win
- #498 before #499: lifecycle state is a prerequisite for event triggers
- #500 last: blocked on engine's generic CBR outcome recording bridge (engine#1190)

### Cross-repo context
- Parent epic: casehubio/fsitrading#50 (Trading YAML Playbooks)
- Engine issues: engine#1190, #1191, #1192 (generic CBR integration)
- casehub-pages#502 is the local epic grouping these four issues

## References

- `packages/yaml-core/src/step/invoke/test-helpers.ts` — mock handlers
- `packages/yaml-core/src/step/structural-evaluator.ts` — evaluateInvoke wiring
- `examples/samples/Scenarios/Concurrency Patterns.ts` — combined pipeline example
