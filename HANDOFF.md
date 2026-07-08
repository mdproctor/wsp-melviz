# casehub-pages Session Handover — 2026-07-08

## Last Session

Closed #136 (CDI/Quarkus integration for EventBroadcaster) — new `casehub-pages-push-runtime` module with `PushProducers` class making `EventBroadcaster`, `TopicRegistry`, `EventStore` injectable. `@DefaultBean` InMemoryEventStore with configurable capacity. Consumer provides `SessionSender` (gateway pattern). Design reviewed (4 rounds, 10 issues, all resolved). 9 @QuarkusTest tests. Landed as 0f1f566 on main.

Investigated platform's notification/endpoint/subscription infrastructure before designing — confirmed pages-push is a different pattern (client-driven WebSocket pub/sub with wildcard topic routing + sequence-numbered replay vs platform's server-side CDI-event-driven SSE broadcasters). No overlap, stays in pages. #147 filed for future cross-node distribution.

## Branch State

Both repos on `main`. Pause stack empty.

## What's Left

- Fleet Monitor gauge overlap — #133 filed · M · Med
- Panel-initiated dataset refresh — #134 filed · S · Med
- Typed payload overload for EventBroadcaster — #137 filed · XS · Low
- Production mutableRestSource write path — #144 filed · M · Med
- Cross-node event distribution for push-runtime — #147 filed · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #142 | Scenario Engine — composition, triggers, remote control, demo UI | L | High | Plan at `docs/plans/2026-07-07-scenario-engine.md`. Next up per user request. |
| #143 | Cross-repo Migration — examples, blocks-ui, aml, clinical | L | Med | Plan at `docs/plans/2026-07-07-cross-repo-migration.md` |
| #133 | Fleet Monitor gauge overlap | M | Med | YAML layout or gauge sizing |
| #137 | Typed payload overload for EventBroadcaster | XS | Low | |

## References

- Spec: `docs/specs/2026-07-08-push-runtime-cdi-design.md`
- Plan: `docs/plans/2026-07-08-push-runtime-cdi.md`
- Design review: `~/adr/casehub-pages/push-runtime-cdi-20260708-171234/`
- Previous: `git show HEAD~1:HANDOFF.md`
