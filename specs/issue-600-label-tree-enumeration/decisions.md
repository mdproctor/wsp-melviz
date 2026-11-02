## D1: Label storage architecture

**Choice:** Consume platform LabelRegistry SPI (platform #694) — no Pages-local storage
**Alternatives:**
- Pages-local SQLite LabelTreeStore — keeps JSON apps self-contained but duplicates platform concept
- Labels as JSON collection — reuses existing data-json but blurs structural metadata vs user data
**Rationale:** Labels are a platform concept, not a Pages concept. The platform has consumption infrastructure (Path, LabelPatternMatcher, LabelRule, SubjectViewSpec) but lacked definition infrastructure. Platform #694 adds LabelRegistry SPI following EndpointRegistry/DataSourceRegistry patterns. Labels are tenant-scoped (not app-scoped) — apps register under PLATFORM_TENANT_ID, tenant extensions merge with tenant-wins precedence. App-level scoping uses root paths (consumer concern).
**Trade-offs:** Pages #600 depends on platform #694 landing first. Pages cannot run label trees without the platform dependency.
**Sources:** platform-api Path.java, LabelPatternMatcher.java, SubjectViewSpec.java; platform #694 design; connectors-project-spi ProjectPlatform.Labels
**Exploration:** deep-analysis
**Status:** captured
