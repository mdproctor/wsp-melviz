# HANDOFF — casehub-pages

## Last Session: 2026-11-25

**Cross-repo scaffold (casehubio/examples#214):** Headless playbook testing.
Branch `issue-214-headless-playbook-testing` — scaffold only, not ready to land.
- Added Playwright Java dependency (optional) to `playbook-runtime/pom.xml`
- Created `HeadlessPlaybookRunner.java` — translates playbook YAML steps to Playwright API calls (fill/click/select/navigate via ARIA locators, verify via REST polling)
- Created `HeadlessPlaybookRunnerTest.java` — unit tests for config, step results, run result aggregation
- TODO: implement `checkExpectations()` JSON matching, `executeBootstrap()`, full integration test

**Finished:** #748 — integration test verifying `Optional<Map>` `@ConfigProperty` injection for PlaybookConfig endpoints. Branch `issue-748-verify-optional-map-config`, ready to land.

Changes:
- Added `quarkus-junit5` test dependency to `playbook-runtime` module
- Created `PlaybookConfigInjectionTest` (`@QuarkusTest`) — 3 tests covering populated maps and default fallback
- Created `application.properties` for test with endpoint map entries and `quarkus.arc.exclude-types` for simulation beans
- All 183 module tests green

## Immediate Next Step

Use `work start` to pick up the next queued item.

## Open Issues (by category)

**New:** #743 (full sizing model, M/Med)
**Mid-size:** #646, #719, #725
**Large:** #669, #616, #92, #94
**Epics:** #426, #95
