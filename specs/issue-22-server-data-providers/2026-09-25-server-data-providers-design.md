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

4. **Two input modes, one pipeline.** Declarative operations (from page YAML
   or a visual builder) and native query strings (from a CodeMirror-based
   query editor) both flow through `DataProvider.query()`.

## Module Structure

```
backend/
  data/                    core SPI: DataProvider, DataSetLookup, DataSetOp, DataResource
                           zero runtime deps beyond Jackson + CDI
  data-sql/                optional: SQL provider (exists)
                           depends on: data/ + Agroal
  data-prometheus/         optional: Prometheus provider (this issue)
                           depends on: data/ + HTTP client (java.net.http or Vert.x)
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

### TimeRangeOp

New sealed variant of `DataSetOp`:

```java
public record TimeRangeOp(
    String column,    // timestamp column name
    String from,      // ISO-8601 or relative expression ("now-1h")
    String to,        // ISO-8601 or relative expression ("now")
    String step       // aggregation interval ("5m", "1h") — nullable
) implements DataSetOp {}
```

The `@JsonSubTypes` annotation on `DataSetOp` gains a fourth entry:

```java
@JsonSubTypes.Type(value = TimeRangeOp.class, name = "timeRange")
```

Operation sequence becomes: `TimeRange? -> Filter* -> Group* -> Sort?`

`TimeRangeOp` is first because it determines the scan window. A Prometheus
provider maps it to `start`/`end`/`step` params. A SQL provider maps it to
`WHERE <column> BETWEEN ? AND ?`. Providers that don't understand time ranges
ignore it; the client falls through to a date-column FilterOp equivalent.

**Relative time expressions:** `from` and `to` accept ISO-8601 timestamps or
relative expressions anchored to `now`: `now`, `now-1h`, `now-7d`,
`now-30m`. The provider resolves these at query time. The syntax follows
Grafana/Prometheus conventions.

### nativeQuery on DataSetLookup

```java
public record DataSetLookup(
    String dataSetId,
    List<DataSetOp> operations,
    Integer refreshTimeSeconds,
    String nativeQuery          // raw PromQL, SQL, etc. — nullable
) {}
```

When `nativeQuery` is non-null, the provider executes it verbatim and returns
the result. The `operations` list may still contain ops for client-side
post-processing (Sort, additional Filter). Providers that don't support
native queries throw `UnsupportedOperationException`.

### ProviderCapability

```java
public record ProviderCapability(
    String type,                    // "prometheus", "sql", etc.
    List<String> supportedOps,      // ["timeRange", "filter", "group", "sort"]
    String nativeQueryLanguage,     // "promql", "sql", null
    boolean supportsStreaming
) {}
```

### DataProvider interface

```java
public interface DataProvider {
    String type();
    boolean canHandle(String dataSetId);
    DataSetResult query(DataSetLookup lookup);

    default ProviderCapability capability() {
        return new ProviderCapability(type(), List.of(), null, false);
    }
}
```

The `capability()` default returns an empty capability set — existing
providers (NoOpDataProvider) don't need to change. New providers override
to declare what they support.

### ServiceCapabilities

```java
public record ServiceCapabilities(
    boolean serverSideQuery,
    List<String> dataProviders,
    boolean dataProxy,
    boolean serverSideCache,
    Map<String, ProviderCapability> providerCapabilities
) {}
```

`DataResource.capabilities()` populates `providerCapabilities` by iterating
all discovered `DataProvider` beans and calling `capability()`.

## Prometheus Provider (backend/data-prometheus/)

### Configuration

```properties
# Global Prometheus endpoint
casehub.pages.data.prometheus.endpoint=http://prometheus:9090

# Per-dataset metric mapping
casehub.pages.data.prometheus.datasets.cpu-metrics.metric=node_cpu_seconds_total
casehub.pages.data.prometheus.datasets.memory-usage.metric=node_memory_MemAvailable_bytes
```

### Query Translation

The provider translates `DataSetLookup` operations into a Prometheus HTTP API
request:

| DataSetOp | PromQL translation |
|---|---|
| `TimeRangeOp(col, from, to, step)` | `start=<from>&end=<to>&step=<step>` query params |
| `FilterOp` with `EQUALS_TO` on a label column | Label matcher: `{<column>="<value>"}` |
| `FilterOp` with `NOT_EQUALS_TO` | Label matcher: `{<column>!="<value>"}` |
| `FilterOp` with `LIKE_TO` | Regex matcher: `{<column>=~"<value>"}` |
| `GroupOp` with aggregation function | `<fn> by (<column>) (metric{...})` |
| `SortOp` | Not translatable — falls through to client-side |

When `nativeQuery` is present, the provider sends it as the `query` param
directly, using `TimeRangeOp` values for `start`/`end`/`step` if present.

### Response Mapping

Prometheus range query returns a matrix (list of time series, each with
`[timestamp, value]` pairs). The provider flattens this into `DataSetResult`:

| Column | Type | Source |
|---|---|---|
| `timestamp` | NUMBER | Sample timestamp (epoch seconds) |
| `value` | NUMBER | Sample value |
| One column per label | TEXT | Label values from the series |

### Capability Declaration

```java
@Override
public ProviderCapability capability() {
    return new ProviderCapability(
        "prometheus",
        List.of("timeRange", "filter", "group"),
        "promql",
        false
    );
}
```

Sort is absent — PromQL doesn't sort time series. The client applies
`SortOp` on the returned `DataSetResult`.

## TypeScript Changes (packages/pages-data/)

### TimeRangeOp type

```typescript
export interface TimeRangeOp {
  readonly type: 'timeRange';
  readonly column: ColumnId;
  readonly from: string;
  readonly to: string;
  readonly step?: string;
}

export type DataSetOp = FilterOp | GroupOp | SortOp | TimeRangeOp;
```

Update `validateOpOrder` to accept the new sequence:
`/^T?F*G*S?$/` where T = timeRange.

### nativeQuery on DataSetLookup

```typescript
export interface DataSetLookup {
  readonly dataSetId: DataSetId;
  readonly operations: readonly DataSetOp[];
  readonly nativeQuery?: string;
  readonly refreshTimeSeconds?: number;
}
```

### Extended ServiceCapabilities

```typescript
export interface ProviderCapability {
  readonly type: string;
  readonly supportedOps: readonly string[];
  readonly nativeQueryLanguage?: string;
  readonly supportsStreaming: boolean;
}

export interface ServiceCapabilities {
  readonly serverSideQuery: boolean;
  readonly dataProviders: readonly string[];
  readonly dataProxy: boolean;
  readonly serverSideCache: boolean;
  readonly providerCapabilities?: Readonly<Record<string, ProviderCapability>>;
}
```

### Operation Splitting in the Resolver

When the resolver knows the provider's capabilities (from cached
`ServiceCapabilities`), it splits the operation pipeline before sending
to the server:

```typescript
function splitOps(
  ops: readonly DataSetOp[],
  supported: readonly string[],
): { server: DataSetOp[]; client: DataSetOp[] } {
  const server: DataSetOp[] = [];
  const client: DataSetOp[] = [];
  for (const op of ops) {
    (supported.includes(op.type) ? server : client).push(op);
  }
  return { server, client };
}
```

The resolver sends a `DataSetLookup` with `server` ops to the backend,
receives a `TypedDataSet`, then applies `client` ops via the existing
`applyOps()` function.

## Page YAML Integration

No new YAML syntax is needed. The existing `serverQuery: true` flag routes
the query to the server. The `TimeRangeOp` is a standard operation:

```yaml
datasets:
  - uuid: cpu-metrics
    serverQuery: true
    operations:
      - type: timeRange
        column: timestamp
        from: now-1h
        to: now
        step: 5m
      - type: filter
        column: mode
        function: EQUALS_TO
        args: [idle]
      - type: group
        columnId: instance
        function: AVG
```

The server resolves `cpu-metrics` to the Prometheus provider via config.
The provider translates the operations into:

```
avg by (instance) (node_cpu_seconds_total{mode="idle"})
```

with `start=<now-1h>&end=<now>&step=300`.

## Scope Boundary

**In scope for issue #22:**
- `TimeRangeOp` in core (Java + TypeScript)
- `nativeQuery` on `DataSetLookup` (Java + TypeScript)
- `ProviderCapability` and extended `ServiceCapabilities` (Java + TypeScript)
- `PrometheusDataProvider` implementation (Java)
- Operation splitting in the TypeScript resolver
- Tests for all of the above

**Out of scope (future issues):**
- InfluxDB, Elasticsearch, or other provider implementations
- Query editor component (CodeMirror-based PromQL/SQL editor)
- Visual query builder that generates `DataSetOp` lists
- Streaming / push-based time-series subscriptions
- Cross-provider joins (combining Prometheus + SQL results)
- Metric discovery / schema introspection API

## References

- `backend/data/src/main/java/io/casehub/pages/data/DataProvider.java` — existing SPI
- `backend/data/src/main/java/io/casehub/pages/data/DataSetOp.java` — sealed interface to extend
- `backend/data/src/main/java/io/casehub/pages/data/DataResource.java` — REST endpoint with CDI routing
- `backend/data-sql/src/main/java/io/casehub/pages/data/sql/SqlDataProvider.java` — reference implementation
- `packages/pages-data/src/dataset/ops.ts` — TypeScript DataSetOp union
- `packages/pages-data/src/dataset/external/resolver.ts` — client-side resolver (operation splitting target)
- `packages/pages-data/src/dataset/external/providers/server-query.ts` — ServerQueryClient
- Grafana datasource plugin SDK — backend-specific queries, unified data frame output
- Metabase MBQL — abstract IR with central translation, `database-supports?` capability negotiation
- Cube.js semantic layer — measures/dimensions defined once, translated per-driver
