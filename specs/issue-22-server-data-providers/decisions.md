# Design Decisions — Server-Side Data Providers

## D1: Query model strategy

**Choice:** Extend `DataSetLookup` as the universal query envelope — add `TimeRangeOp` as a new `DataSetOp` variant alongside Filter/Group/Sort. Each server-side provider translates what it can natively; client applies the rest.
**Alternatives:**
- Backend-specific query types (Grafana model) — each provider defines its own query schema. More expressive per-backend but forces page authors to know which backend they're targeting. Precludes backend-transparent pages.
- Dual model (`TimeSeriesLookup` + `DataSetLookup`) — type-safe separation but doubles the surface area, two paths through the system, two sets of tests.
- SQL as universal language (Superset model) — forces non-SQL backends through an adapter. Prometheus doesn't speak SQL; adapter loses native capabilities (counter-reset handling, instant vectors).
**Rationale:** One query model keeps one path through the system. Page YAML doesn't change when the backing provider changes. The operation sequence (TimeRange? → Filter* → Group* → Sort?) is expressive enough for declarative composition, and `nativeQuery` covers everything else.
**Trade-offs:** Providers that can't handle an operation silently fall through to client-side — the page author won't know unless they check capability negotiation. Operations like PromQL's `rate()` with counter-reset handling can't be expressed as DataSetOps — they require `nativeQuery`.
**Sources:** Grafana datasource plugin SDK, Metabase MBQL, Cube.js semantic layer, existing `backend/data/DataSetOp.java`
**Exploration:** deep-analysis
**Status:** captured

## D2: Native query support

**Choice:** Add a `nativeQuery` field to `DataSetLookup`. When present, the provider executes it verbatim. Operations list can still contain client-side-only ops (Sort, additional Filter) applied after.
**Alternatives:**
- No native query — force everything through DataSetOps. Simpler contract but blocks power users and prevents a future query editor component from using the same pipeline.
- Separate native query endpoint — `/api/dataset/native-query` with its own request type. Keeps DataSetLookup pure but creates a parallel path with duplicate auth, caching, and routing logic.
**Rationale:** A query editor component built on Pages (using CodeMirror) would send `{ dataSetId, nativeQuery: "rate(...)", operations: [] }`. The two input modes (declarative ops, native string) compose through the same provider pipeline. The `nativeQueryLanguage` field in `ProviderCapability` tells the editor which syntax mode to load.
**Trade-offs:** `nativeQuery` bypasses the structured operation model — no automatic capability splitting, no operation-level caching granularity.
**Sources:** Grafana's per-plugin query model, CodeMirror integration in `packages/pages-code-editor/`
**Exploration:** deep-analysis
**Status:** captured

## D3: Provider routing mechanism

**Choice:** Route by dataset ID via `DataProvider.canHandle(dataSetId)` — the existing pattern. Each provider is configured at deployment time with which dataset IDs it owns. No `provider:` field in the page YAML.
**Alternatives:**
- Explicit `provider:` field on dataset definition — page YAML names the backend. More transparent but couples page content to deployment topology.
- Type-prefix routing (`prometheus:cpu-metrics`) — encodes the provider in the dataset ID. Discoverable but pollutes the namespace.
**Rationale:** Dataset ID routing separates page content from deployment. The same page YAML works against a Prometheus backend in production and a mock provider in tests. Configuration is a deployment concern (`application.properties`), not a page concern.
**Trade-offs:** Page authors can't tell from the YAML which backend serves a dataset. Debugging requires checking server config.
**Sources:** Existing `SqlDataProvider.canHandle()`, `DataResource.resolveProvider()`
**Exploration:** quick
**Status:** captured

## D4: Module structure

**Choice:** One Maven module per provider (`data-prometheus/`, `data-influxdb/`, etc.), each depending only on `data/` (core SPI) plus the backend-specific client library. CDI auto-discovers providers on the classpath.
**Alternatives:**
- Single `data-providers` module with all backends — simpler build but drags in all client libraries even when only one is needed.
- SPI via ServiceLoader — works without CDI but loses Quarkus config injection, health checks, and metrics integration.
**Rationale:** Consuming apps pick what they need — include the Maven dependency, the provider appears. No registration, no wiring. This works for both Pages standalone and apps that consume Pages modules as libraries.
**Trade-offs:** More Maven modules to maintain. Each provider module must be independently testable (its own `@QuarkusTest` setup).
**Sources:** Existing `backend/data-sql/` pattern, Quarkus CDI bean discovery
**Exploration:** quick
**Status:** captured

## D5: Capability negotiation

**Choice:** Each `DataProvider` declares a `ProviderCapability` (supported ops, native query language, streaming support). `ServiceCapabilities` exposes per-provider capabilities. The client splits the operation pipeline: server handles supported ops, client applies the rest on the returned `TypedDataSet`.
**Alternatives:**
- No capability negotiation — client sends everything, server ignores what it can't handle and returns partial results. Simpler but the client doesn't know what was applied and what wasn't.
- Server returns `handledOps` in the response — instead of pre-negotiation, the server reports what it did after the fact. More accurate but requires client to re-apply potentially expensive operations.
**Rationale:** Pre-negotiation lets the client make intelligent decisions: skip sending ops the server can't handle (saves bandwidth), prepare to apply them client-side, adapt UI controls (hide unsupported options). This matches Metabase's `database-supports?` and Superset's engine specs — proven at scale.
**Trade-offs:** Capabilities are declared statically, not per-query. A SQL provider might support `timeRange` only on tables with a timestamp column, but declares it globally.
**Sources:** Metabase `database-supports?`, Superset `db_engine_spec`, Grafana plugin capabilities
**Exploration:** deep-analysis
**Status:** captured
