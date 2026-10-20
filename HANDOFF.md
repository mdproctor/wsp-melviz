# HANDOFF — casehub-pages

## Last Session

Completed #390 (scenario format refinements, TS side). Extracted dispatch protocol types from local interfaces in playbook-handler.ts to exported types in playbook/types.ts. Renamed CommandPayload.target to element, added DispatchStep.delay for per-step timing, made speed opt-in (default 0 = no delay), added REST action definition alongside GraphQL. Branch landed on main, issue closed.

## Immediate Next Step

Recommended #531 (inferred method-call dispatch for playbook steps) — directly extends the executor framework just refined. Alternative: #499 (event-triggered scenarios, builds on #454 emit-event).

## References

- `packages/pages-aria/src/playbook/types.ts` — dispatch protocol types (new canonical location)
- `packages/pages-aria/src/playbook/invoke/definitions.ts` — action catalog (REST added)
- `packages/pages-aria/src/server/playbook-handler.ts` — executor handler (simplified)
- `plans/2026-10-20-scenario-format-refinements.md` — implementation plan
