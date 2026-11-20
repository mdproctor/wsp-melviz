# HANDOFF — casehub-pages

## Last Session: 2026-11-20

Implemented #721: REST endpoint for publishing to push topics. Added `PushPublishResource` to `push-runtime` — thin JAX-RS wrapper over `EventBroadcaster` with `@Authenticated`. Workers can now broadcast events via `call: http` in `do:` blocks.

Filed #729 (JSON body validation) and #730 (flaky seq assertions in shared `@ApplicationScoped` EventStore) from code review. Both queued.

## Immediate Next Step

`work start 716` — Category-level playbook execution tests.

## References

- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`
- `specs/issue-721-rest-push-publish/2026-11-20-rest-push-publish-design.md`
- `plans/2026-11-20-rest-push-publish.md`
- `blog/2026-11-20-mdp01-rest-push-publish.md`
