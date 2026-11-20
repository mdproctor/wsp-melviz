# HANDOFF — casehub-pages

## Last Session: 2026-11-20

Completed #731 — reuse ObjectMapper instance in PushPublishResource (XS).

- Extracted per-request `new ObjectMapper()` to `private static final ObjectMapper MAPPER`
- Fixed fully-qualified import in PushProducers (was `new com.fasterxml.jackson.databind.ObjectMapper()`)
- Fixed inconsistent indentation in PushProducersTest
- Added `publish_null_body_returns_400` test for missing body branch
- Revised mdp02 blog entry: "Two Things" → "Three Things Code Review Found in push-runtime" — added ObjectMapper reuse section with SVG diagram
- Filed #733 (TopicRegistry listeners leak across `@QuarkusTest` classes) and queued it

All tests pass. Branch squashed to single commit, landing via work-end.

## Immediate Next Step

`work start 733` — TopicRegistry listeners leak across `@QuarkusTest` classes (XS).

## References

- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`
- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushProducers.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushProducersTest.java`
