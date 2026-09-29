# HANDOFF — casehub-pages

## Last Session (2026-09-29)

**Completed:** #501 — Step catalog browser. Landed on main as 3 squashed commits (096bf861..54bdeb65).

**What was built:**
- Java `@McpDomain("step-catalog")` resolver — `StepCatalogService` parses YAML step definition files, `StepCatalogResolver` exposes via GraphQL/MCP
- TS `catalog-execute-handler` — REST endpoint for live step execution via `StructuralStepEvaluator`
- `<pages-step-catalog>` Lit component — search, source filter chips, detail view with schema tables, try-it execution panel, YAML template generation (CustomEvent + clipboard)
- Integrated into `PagesScenarioController` as `'catalog'` view mode alongside `'outline'` and `'library'`
- Gallery demo sample (Step Catalog)

**Filed during design discussion:**
- **platform#483** — Unified step runtime API with Java/TS parity. Key decisions:
  - `registry.register()` is the API — `@StepPlugin` + CDI scanning is sugar
  - APT demoted to optional validation — CDI runtime scanner replaces code generation
  - Portability model: universal (protocol-based invoke) / java / ts / both
  - Script portability = most restrictive step. Executor validates all-or-nothing.
- **pages#506** — Unified step catalog with portability model and cross-runtime discovery (depends on platform#483)

**Next session:** `work start #506` — but front-load TS-only work (portability field, REST endpoint, catalog badges). Java integration waits for platform#483.

### Remaining epic #502 queue
- **#498** — Scenario lifecycle state (M / Med, independent)
- **#499** — Event-triggered scenario activation (M / Med, depends on #498)
- **#500** — Scenario outcome tracking + CBR linkage (M / High, blocked on engine#1190)

### Known issues
- Gallery sample fetch mock conflict: the gallery's `galleryFetch` wrapper runs after the sample's `window.fetch` override, preventing the mock from intercepting `/graphql` calls. The `customElements.whenDefined` fix and lowercase variable naming fix are committed. A gallery infrastructure fix is needed to let sample scripts override fetch reliably.
- Pre-existing Java compilation errors in `backend/scenario-runtime` test classes (`SequencePartitionerTest`, `ScriptRegistryTest`, etc.) — unrelated to #501 work.

### Cross-repo context
- Parent epic: casehubio/fsitrading#50 (Trading YAML Playbooks)
- casehub-pages#502 is the local epic grouping scenario infrastructure issues

## References

- `packages/yaml-core/src/step/` — StepCatalog, StepDefinition, StepPluginRegistry, invoke handlers
- `packages/pages-aria/src/controller/step-catalog.ts` — catalog browser component
- `packages/pages-aria/src/controller/scenario-controller.ts` — controller integration (catalog view mode)
- `packages/pages-aria/src/server/catalog-execute-handler.ts` — execution handler
- `backend/scenario-runtime/src/main/java/.../StepCatalogService.java` — Java catalog service
- `backend/mcp/src/main/java/.../StepCatalogResolver.java` — @McpDomain resolver
- `platform/yaml-plugin-api/` — @StepPlugin annotation
- `platform/yaml-plugin-processor/` — APT processor (to be demoted per platform#483)
- `docs/specs/issue-501-step-catalog-browser/` — design spec and decisions
