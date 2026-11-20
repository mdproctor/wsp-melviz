# HANDOFF — casehub-pages

## Last Session: 2026-11-20

Completed #731 and #733, both push-runtime test isolation improvements:

- **#731** — Extracted per-request `new ObjectMapper()` to `private static final` in PushPublishResource. Fixed fully-qualified import in PushProducers, reformatted PushProducersTest, added null body test.
- **#733** — Added `TopicRegistry.clear()` to reset trie and connection map. Wired into `@BeforeEach` in BroadcastIntegrationTest and PushPublishResourceTest alongside existing `testConfig.clear()`. Prevents listener state leaking across `@ApplicationScoped` test classes.

Revised mdp02 blog entry: "Three Things Code Review Found in push-runtime" — added ObjectMapper reuse section, TopicRegistry follow-up paragraph, and SVG diagram. Filed #733 during #731 review, implemented same session.

Both branches squashed to single commits and landed on main. Queue drained.

## Immediate Next Step

No queued items. Use `work find` or `work start #N` to pick up new work from the open issue list.

## Open Issues (by category)

**Quick wins:** #723 (extract demoCatalog, XS/Low), #722 (match address mismatch, S/Low)
**Mid-size:** #671, #619, #646, #719
**Large:** #669, #616, #92, #94
**Epics:** #426, #95

## References

- `backend/push/src/main/java/io/casehub/pages/push/TopicRegistry.java`
- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/BroadcastIntegrationTest.java`
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`
