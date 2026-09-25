---
title: "Server-Side Data Providers — Holistic Architecture"
date: 2026-09-25
issue: 22
---

# Server-Side Data Providers — Holistic Architecture

The Pages data pipeline currently supports two query paths: client-side fetch
(URL/content → extract → filter/group/sort) and server-side query
(`DataSetLookup` → `ServerQueryClient` → Quarkus `DataResource` → `DataProvider`).
The server path already has a provider SPI with CDI auto-discovery and one
implementation (`SqlDataProvider`). This design extends that foundation to
support time-series backends (Prometheus first, then InfluxDB, Elasticsearch,
etc.) without changing the query model or adding parallel paths.

## Design Principles

1. **One query model, many translators.** `DataSetLookup` is the universal
   query envelope. Each provider translates what it can natively; the client
   applies the rest.

2. **Unify at the response, not the query.** Every provider returns
   `DataSetResult` (columns + rows). The consumer doesn't know which backend
   produced it.

3. **Optional modules.** Each provider is a separate Maven artifact depending
   only on `data/` (core SPI) + its backend client library. Include the
   dependency → the provider appears. Don't → it doesn't exist.

4. **Server translates, client completes.** The provider handles the ops it
   can natively and returns the rest as `remainingOps`. The client applies
   those via the existing `applyOps()` pipeline. No capability negotiation
   needed — the provider is the single source of truth for what it can handle.

## Module Structure

```
backend/
  data/                    core SPI: DataProvider, DataSetLookup, DataSetOp, DataResource
                           zero runtime deps beyond Jackson + CDI
  data-sql/                optional: SQL provider (exists)
                           depends on: data/ + Agroal
  data-prometheus/         optional: Prometheus provider (this issue)
                           depends on: data/ + java.net.http
  data-influxdb/           optional: future
  data-elasticsearch/      optional: future
```

Consuming apps pick what they need:

```xml
<!-- App that needs SQL + Prometheus -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-data-sql</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-data-prometheus</artifactId>
</dependency>
```

Pages standalone includes all providers. Apps consuming Pages as a library
cherry-pick modules. CDI discovers whatever is on the classpath.

## Core SPI Changes (backend/data/)

### QueryResult

New record wrapping the data result plus any operations the provider
could not handle natively:

```java
public record QueryResult(DataSetResult result, List<DataSetOp> remainingOps) {
    public static QueryResult complete(DataSetResult result) {
        return new QueryResult(result, List.of());
    }
}
```

Providers that handle all operations (e.g. `SqlDataProvider`) use the
`complete()` factory. Providers with partial translation (e.g. Prometheus)
construct with the ops they couldn't translate.

**Ordering constraint:** The `remainingOps` list must preserve the relative
ordering of operations from the original `DataSetLookup.operations` list.
The client's `applyOps()` calls `validateOpOrder()` which enforces
`/^F*G*S?$/` — reordered ops would be rejected.

### DataProvider interface

```java
public interface DataProvider {
    String type();
    boolean canHandle(String dataSetId);
    QueryResult query(DataSetLookup lookup);
}
```

The return type changes from `DataSetResult` to `QueryResult`. Existing
implementations (`SqlDataProvider`, `NoOpDataProvider`) wrap their result
with `QueryResult.complete(result)`.

### TIME_FRAME filter handling

Time-range constraints use the existing `FilterOp` + `DateLeaf` +
`TIME_FRAME` mechanism (see `packages/pages-data/src/dataset/filter.ts`
and `packages/pages-data/src/dataset/timeframe.ts`). No new operation
type is needed.

In YAML (unresolved form):

```yaml
operations:
  - type: filter
    column: timestamp
    function: TIME_FRAME
    args: ["now-1HOUR till now"]
```

The backend provider resolves this using the existing `TimeFrame` model
(parsed via `parseTimeFrame()`). Each provider maps the resolved from/to
dates to its native time constraint:

- **SQL**: `WHERE <column> BETWEEN ? AND ?` with bind parameters
- **Prometheus**: `start=<from>&end=<to>` query parameters

`SqlQueryBuilder.appendLeafFilter()` gains a `TIME_FRAME` case that parses
the time-frame expression from `args`, resolves it against the server's
current time, and generates a `BETWEEN` clause with bind parameters.

### Migration

**DataProvider.query() return type**: Changes from `DataSetResult` to
`QueryResult`. All implementations and test call sites must update:

| Call site | Change |
|---|---|
| `NoOpDataProvider.query()` | Return `QueryResult.complete(emptyResult)` |
| `SqlDataProvider.query()` | Return `QueryResult.complete(result)` |
| `DataResourceQueryTest` (mock provider) | Return `QueryResult` |
| `NoOpDataProviderTest` | Assert on `QueryResult` |
| `SqlDataProviderTest` | Assert on `QueryResult` |
| `DataCacheService.queryCached()` | Generic type: `DataSetResult` → `QueryResult` |
| `manager.ts` | Extract `resolveOps` to `ops-resolve.ts`, import from there |

**DataResource.query()**: Return type changes to `QueryResult`. The endpoint
serializes both the result and `remainingOps` in the response.

### DataQueryException

New exception class for provider errors, used by all providers:

```java
public class DataQueryException extends RuntimeException {
    private final String code;

    public DataQueryException(String code, String message) {
        super(message);
        this.code = code;
    }

    public DataQueryException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String code() { return code; }
}
```

Error codes: `INVALID_QUERY`, `FETCH_FAILED`, `RESULT_TOO_LARGE`.

### DataQueryExceptionMapper

JAX-RS `ExceptionMapper` that maps provider errors to structured HTTP
responses with appropriate status codes:

```java
@Provider
public class DataQueryExceptionMapper implements ExceptionMapper<DataQueryException> {
    @Override
    public Response toResponse(DataQueryException e) {
        Response.Status status = switch (e.code()) {
            case "INVALID_QUERY" -> Response.Status.BAD_REQUEST;           // 400
            case "RESULT_TOO_LARGE" -> Response.Status.REQUEST_ENTITY_TOO_LARGE; // 413
            case "FETCH_FAILED" -> Response.Status.BAD_GATEWAY;            // 502
            default -> Response.Status.INTERNAL_SERVER_ERROR;              // 500
        };
        return Response.status(status)
            .entity(Map.of("error", e.getMessage(), "code", e.code()))
            .type(MediaType.APPLICATION_JSON)
            .build();
    }
}
```

This lives in the `data/` core module so all providers benefit. Without it,
any exception from `provider.query()` propagates to Quarkus's default
handler and becomes HTTP 500 Internal Server Error — unhelpful for
diagnosable errors like invalid metric names (400) or Prometheus
unreachable (502).

## Prometheus Provider (backend/data-prometheus/)

### Configuration

```properties
# Global Prometheus endpoint
casehub.pages.data.prometheus.endpoint=http://prometheus:9090

# Authentication (default: none)
casehub.pages.data.prometheus.auth.type=none
# For type=bearer:
casehub.pages.data.prometheus.auth.token=<token>
# For type=basic:
casehub.pages.data.prometheus.auth.username=<user>
casehub.pages.data.prometheus.auth.password=<pass>

# HTTP timeouts
casehub.pages.data.prometheus.timeout.connect=5s
casehub.pages.data.prometheus.timeout.read=30s

# Cardinality protection — max samples in response
casehub.pages.data.prometheus.max-samples=10000

# Per-dataset metric mapping
casehub.pages.data.prometheus.datasets.cpu-metrics.metric=node_cpu_seconds_total
casehub.pages.data.prometheus.datasets.cpu-metrics.step=5m
casehub.pages.data.prometheus.datasets.memory-usage.metric=node_memory_MemAvailable_bytes
casehub.pages.data.prometheus.datasets.memory-usage.step=1m
```

`step` is a per-dataset configuration property controlling the Prometheus
aggregation interval. It is not an operation because it controls query
resolution, not data transformation.

### Query Translation

The provider receives a `DataSetLookup` containing the full operation
pipeline. It translates what it can natively and returns the rest as
`remainingOps` in the `QueryResult`.

#### FilterOp translation

The provider walks the `FilterExpression` tree. The actual `FilterOp`
contains `List<FilterExpression>` where `FilterExpression` is a sealed
interface with 7 variants: `And`, `Or`, `Not`, `Unresolved`, `Numeric`,
`StringLeaf`, `DateLeaf`.

| FilterExpression variant | Translatable? | PromQL translation |
|---|---|---|
| `StringLeaf(col, {fn: EQUALS_TO, value: v})` | Yes | `{col="v"}` |
| `StringLeaf(col, {fn: NOT_EQUALS_TO, value: v})` | Yes | `{col!="v"}` |
| `StringLeaf(col, {fn: LIKE_TO, pattern: p})` | Yes | `{col=~"<regex>"}` — see conversion below |
| `And(children)` where ALL children translatable | Yes | Merged label matchers in `{}` |
| `DateLeaf(col, {fn: TIME_FRAME, ...})` | Special | Resolved to `start`/`end` query params |
| `Or(...)` | No | → `remainingOps` |
| `Not(...)` | No | → `remainingOps` |
| `Numeric(...)` | No | → `remainingOps` (Prometheus labels are strings) |
| `DateLeaf` (non-TIME_FRAME) | No | → `remainingOps` |
| `Unresolved` (non-TIME_FRAME) | No | → `remainingOps` |
| `Unresolved(col, "TIME_FRAME", args)` | Special | Resolved to `start`/`end` query params |

When an `And` node contains a mix of translatable and untranslatable
children, the provider extracts the translatable children as label matchers
and returns a new `FilterOp` containing only the untranslatable children
as `remainingOps`.

**Multiple TIME_FRAME precedence:** Prometheus accepts a single `start`/`end`
pair. If the operations contain multiple TIME_FRAME filters (on different
columns or as duplicates), the provider consumes the first TIME_FRAME for
`start`/`end` query parameters and returns any additional TIME_FRAME filters
in `remainingOps` for client-side application.

**LIKE_TO → PromQL regex conversion:**

SQL `LIKE_TO` patterns use `%` (any string) and `_` (single character).
PromQL `=~` uses RE2 regex syntax. The conversion:

1. Escape RE2 metacharacters in the pattern: `.` `*` `+` `?` `(` `)` `[` `]` `{` `}` `^` `$` `|` `\` → prefix with `\`
2. Replace `%` → `.*`
3. Replace `_` → `.`
4. Anchor the result: `^<pattern>$`

Example: `prod%` → `^prod.*$`; `node_cpu_%_total` → `^node_cpu_.*_total$`

#### GroupOp translation

The actual `GroupOp` record has four fields: `GroupingKey groupingKey`,
`List<ResultColumn> columns`, `List<String> selectedIntervals`,
`Boolean join`. `GroupingKey` carries a `GroupStrategy` with four modes:
`Distinct`, `FixedCalendar`, `DynamicRange`, `Dynamic`.

| GroupOp configuration | Translatable? | PromQL translation |
|---|---|---|
| `GroupStrategy.Distinct` | Yes | `by (<labels>)` |
| `ResultColumn.Aggregate` with COUNT | Yes | `count by (...) (metric{...})` |
| `ResultColumn.Aggregate` with SUM | Yes | `sum by (...) (metric{...})` |
| `ResultColumn.Aggregate` with AVERAGE | Yes | `avg by (...) (metric{...})` |
| `ResultColumn.Aggregate` with MIN | Yes | `min by (...) (metric{...})` |
| `ResultColumn.Aggregate` with MAX | Yes | `max by (...) (metric{...})` |
| `GroupStrategy.FixedCalendar` | No | → `remainingOps` |
| `GroupStrategy.DynamicRange` | No | → `remainingOps` |
| `GroupStrategy.Dynamic` | No | → `remainingOps` |
| `ResultColumn.Select` | No | → `remainingOps` (no PromQL equivalent) |
| `ResultColumn.Key` beyond group labels | No | → `remainingOps` |
| `selectedIntervals` (non-empty) | No | → `remainingOps` |
| `join` flag (true) | No | → `remainingOps` |
| Aggregation: DISTINCT, JOIN, DISTINCTJOIN, MEDIAN | No | → `remainingOps` |

When a `GroupOp` is only partially translatable (e.g. a translatable
aggregation function with an untranslatable `GroupStrategy`), the entire
`GroupOp` goes to `remainingOps`. Partial group translation would produce
incorrect intermediate results.

#### SortOp translation

Not translatable — PromQL does not sort time series. Always returned as
`remainingOps`. The client applies `SortOp` via `applySort()`.

### Response Mapping

Prometheus range query returns a matrix (list of time series, each with
`[timestamp, value]` pairs). The provider flattens this into `DataSetResult`:

| Column | Type | Source |
|---|---|---|
| `timestamp` | DATE | Sample timestamp — epoch seconds converted to ISO-8601 |
| `value` | NUMBER | Sample value |
| One column per label | TEXT | Label values from the series |

Timestamps are mapped to `DATE` type (not `NUMBER`). The provider converts
epoch seconds to ISO-8601 strings during response flattening, preserving
type fidelity for time-series visualization components that expect `DATE`-
typed columns for axis formatting.

Prometheus instant query returns a vector (single value per series). The
provider maps this to:

| Column | Type | Source |
|---|---|---|
| `value` | NUMBER | Instant value |
| One column per label | TEXT | Label values from the series |

No `timestamp` column for instant queries — the query time is implicit.

### Instant vs Range Queries

The provider selects the Prometheus API endpoint based on whether a time-
range filter is present in the operations:

| Condition | API endpoint | Response type |
|---|---|---|
| `DateLeaf` with `TIME_FRAME` present | `/api/v1/query_range` | Matrix |
| No time-range filter | `/api/v1/query` | Vector |

For range queries, `step` comes from the dataset configuration
(`casehub.pages.data.prometheus.datasets.<id>.step`). If no step is
configured, the provider uses a default of `60s`.

### Error Handling

| Prometheus response | Mapped to |
|---|---|
| HTTP 422 (invalid PromQL) | `throw new DataQueryException("INVALID_QUERY", response.error)` |
| HTTP 503 (overloaded) | `throw new DataQueryException("FETCH_FAILED", "Prometheus unavailable")` |
| HTTP 4xx/5xx (other) | `throw new DataQueryException("FETCH_FAILED", "HTTP <status>: <error>")` |
| Sample count > `max-samples` | `throw new DataQueryException("RESULT_TOO_LARGE", "...")` |
| Connect timeout | `throw new DataQueryException("FETCH_FAILED", "Connection timed out")` |
| Read timeout | `throw new DataQueryException("FETCH_FAILED", "Read timed out")` |

The `DataQueryExceptionMapper` (see §Core SPI Changes) translates these
to structured HTTP responses: `INVALID_QUERY` → 400, `FETCH_FAILED` → 502,
`RESULT_TOO_LARGE` → 413. The SQL provider should also migrate from
`IllegalStateException` to `DataQueryException` for consistency — this
is a minor follow-up, not blocking.

## TypeScript Changes (packages/pages-data/)

### QueryResult type

```typescript
export interface QueryResult {
  readonly result: ServerQueryResponse;
  readonly remainingOps: readonly DataSetOp[];
}
```

### ServerQueryClient

Updated to return `QueryResult`. The response JSON now includes
`remainingOps` alongside the data result:

```typescript
async query(lookup: DataSetLookup): Promise<{ dataset: TypedDataSet; remainingOps: DataSetOp[] }> {
    // ... fetch as before ...
    const body = await response.json() as {
        result: ServerQueryResponse;
        remainingOps: DataSetOp[];
    };
    return {
        dataset: toTypedDataSet(toDataSet(body.result)),
        remainingOps: body.remainingOps ?? [],
    };
}
```

The `remainingOps ?? []` fallback provides backward compatibility: if the
server returns no `remainingOps` field (e.g. older backend), all ops are
assumed handled.

### Resolver serverQuery path

The resolver applies `remainingOps` after receiving the server response.
The remaining ops may contain `Unresolved` filter expressions that must
be resolved before `applyOps()` can consume them. Resolution requires
the column schema from the returned dataset.

The existing `resolveOps()` function in `manager.ts` already performs
this resolution — it maps filter ops through `resolveFilterTypes(expr,
columns)` and passes group/sort ops through unchanged. Extract it to a
shared utility (`ops-resolve.ts`) so both the manager and the resolver
can use it:

```typescript
// ops-resolve.ts (extracted from manager.ts)
export function resolveOps(
  ops: readonly DataSetOp[],
  columns: readonly Column[],
): ResolvedDataSetOp[] {
  return ops.map(op => {
    if (op.type !== "filter") return op;
    return {
      type: "filter" as const,
      expressions: op.expressions.map(expr => resolveFilterTypes(expr, columns)),
    };
  });
}
```

The resolver then uses it with the returned dataset's columns:

```typescript
if (def.serverQuery) {
    // ... setup client as before ...
    const effectiveLookup = lookup ?? { dataSetId: def.uuid, operations: [] };
    const { dataset, remainingOps } = await client.query(effectiveLookup);
    const final = remainingOps.length > 0
        ? applyOps(dataset, resolveOps(remainingOps, dataset.columns))
        : dataset;
    ctx.manager.apply(def.uuid, { type: "snapshot", dataset: final });
    return { dataset: final, inferredColumns: false, source: "serverQuery" };
}
```

### No changes to existing types

- `DataSetOp` union stays `FilterOp | GroupOp | SortOp` (no TimeRangeOp)
- `DataSetLookup` stays unchanged (no nativeQuery)
- `ServiceCapabilities` stays unchanged (no providerCapabilities)
- `validateOpOrder` regex stays `/^F*G*S?$/`
- `applyOps` unchanged — handles filter/group/sort as before

## Page YAML Integration

No new YAML syntax is needed. The existing `serverQuery: true` flag routes
the query to the server. Time-range constraints use the existing filter
model with the `TIME_FRAME` function:

```yaml
datasets:
  - uuid: cpu-metrics
    serverQuery: true
    operations:
      - type: filter
        column: timestamp
        function: TIME_FRAME
        args: ["now-1HOUR till now"]
      - type: filter
        column: mode
        function: EQUALS_TO
        args: [idle]
      - type: group
        columnId: instance
        function: AVG
```

The server resolves `cpu-metrics` to the Prometheus provider via config.
The provider translates the translatable operations into:

```
avg by (instance) (node_cpu_seconds_total{mode="idle"})
```

with `start=<resolved-from>&end=<resolved-to>&step=300`.

The `SortOp` (if any) and untranslatable filter/group operations return
as `remainingOps` and are applied client-side.

## Scope Boundary

**In scope for issue #22:**
- `QueryResult` record in core SPI (Java + TypeScript)
- `DataProvider.query()` return type change (Java)
- `DataQueryException` and `DataQueryExceptionMapper` in core SPI (Java)
- `TIME_FRAME` handling in `SqlQueryBuilder` (Java)
- `PrometheusDataProvider` implementation (Java)
- Extract `resolveOps` to shared `ops-resolve.ts` (TypeScript)
- Updated `ServerQueryClient` and resolver for `remainingOps` (TypeScript)
- Tests for all of the above

**Out of scope (future issues):**
- Native query editor — separate endpoint with its own authorization model
- `ProviderCapability` for UI/query builder consumption
- Dynamic `step` configuration (runtime-adjustable aggregation interval)
- InfluxDB, Elasticsearch, or other provider implementations
- Visual query builder that generates `DataSetOp` lists
- Streaming / push-based time-series subscriptions
- Cross-provider joins (combining Prometheus + SQL results)
- Metric discovery / schema introspection API

## References

- `backend/data/src/main/java/io/casehub/pages/data/DataProvider.java` — existing SPI
- `backend/data/src/main/java/io/casehub/pages/data/DataSetOp.java` — sealed interface (unchanged)
- `backend/data/src/main/java/io/casehub/pages/data/DataResource.java` — REST endpoint with CDI routing
- `backend/data/src/main/java/io/casehub/pages/data/FilterExpression.java` — sealed filter tree
- `backend/data-sql/src/main/java/io/casehub/pages/data/sql/SqlQueryBuilder.java` — reference implementation
- `packages/pages-data/src/dataset/ops.ts` — TypeScript DataSetOp union
- `packages/pages-data/src/dataset/filter.ts` — TypeScript filter model with TIME_FRAME
- `packages/pages-data/src/dataset/timeframe.ts` — TimeFrame/TimeInstant infrastructure
- `packages/pages-data/src/dataset/external/resolver.ts` — client-side resolver
- `packages/pages-data/src/dataset/external/providers/server-query.ts` — ServerQueryClient
- Grafana datasource plugin SDK — backend-specific queries, unified data frame output
- Metabase MBQL — abstract IR with central translation
- Cube.js semantic layer — measures/dimensions defined once, translated per-driver
