# HANDOFF — casehub-pages

## Last Session: 2026-11-20

Completed #729 and #730, both follow-ups from #721 code review:

- **#729** — Added JSON body validation to `PushPublishResource.publish()`. Guard clause using `ObjectMapper.readTree()` returns 400 on malformed JSON before broadcast. Test added.
- **#730** — Replaced all exact seq assertions in `BroadcastIntegrationTest` and `PushPublishResourceTest` with relative checks (`greaterThan(0)`, monotonic ordering). Prevents flaky failures from shared `@ApplicationScoped` `InMemoryEventStore` state across `@QuarkusTest` classes.

Filed #731 (reuse ObjectMapper instance — XS improvement). Updated mdp01 blog to correct now-inaccurate claim about `equalTo(1)` safety. Wrote mdp02 blog entry.

Both branches landing via work-end. Queue has #731 as next item.

## Immediate Next Step

`work start 731` — Reuse ObjectMapper instance in PushPublishResource (XS).

## References

- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/BroadcastIntegrationTest.java`
- `blog/2026-11-20-mdp02-boundary-validation-test-isolation.md`
