# Server-Side Data Providers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #22 — Server-side data provider: Prometheus
**Issue group:** #22

**Goal:** Extend the data provider SPI with `QueryResult` (partial
translation support), structured error handling, and a Prometheus
provider module. Update the TypeScript client to apply `remainingOps`.

**Architecture:** The existing `DataProvider` SPI gains a `QueryResult`
return type carrying both the data result and any operations the provider
could not handle. A new `data-prometheus` Maven module implements
`DataProvider` by translating `FilterOp`/`GroupOp` to PromQL and
returning untranslatable ops as `remainingOps`. The TypeScript resolver
applies those client-side via the existing `applyOps()`.

**Tech Stack:** Java 21, Quarkus, CDI, java.net.http, TypeScript, Vitest

## Global Constraints

- Each provider module depends only on `backend/data` + its backend client library
- All new Java types use `sealed interface` / `record` patterns matching the existing codebase
- Jackson `@JsonSubTypes` annotations required for polymorphic serialization
- Tests use `@QuarkusTest` with AssertJ assertions (Java), Vitest (TypeScript)
- Commit after every task with `Refs #22`

---

## Batch 1: Core SPI Changes

### Task 1: QueryResult record and DataProvider return type change

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/QueryResult.java`
- Modify: `backend/data/src/main/java/io/casehub/pages/data/DataProvider.java`
- Modify: `backend/data/src/main/java/io/casehub/pages/data/NoOpDataProvider.java`
- Modify: `backend/data/src/main/java/io/casehub/pages/data/DataResource.java:64-78`
- Modify: `backend/data/src/main/java/io/casehub/pages/data/DataCacheService.java:80-88`
- Modify: `backend/data-sql/src/main/java/io/casehub/pages/data/sql/SqlDataProvider.java:55`
- Modify: `backend/data/src/test/java/io/casehub/pages/data/DataResourceQueryTest.java`
- Modify: `backend/data/src/test/java/io/casehub/pages/data/NoOpDataProviderTest.java`
- Modify: `backend/data-sql/src/test/java/io/casehub/pages/data/sql/SqlDataProviderTest.java`

**Interfaces:**
- Produces: `QueryResult(DataSetResult result, List<DataSetOp> remainingOps)` with `QueryResult.complete(DataSetResult)` factory
- Produces: `DataProvider.query(DataSetLookup): QueryResult` (changed return type)

- [ ] **Step 1: Write the QueryResult record**

```java
package io.casehub.pages.data;

import java.util.List;

public record QueryResult(DataSetResult result, List<DataSetOp> remainingOps) {
    public static QueryResult complete(DataSetResult result) {
        return new QueryResult(result, List.of());
    }
}
```

- [ ] **Step 2: Change DataProvider.query() return type**

Change `DataProvider.java` line 6 from:
```java
DataSetResult query(DataSetLookup lookup);
```
to:
```java
QueryResult query(DataSetLookup lookup);
```

- [ ] **Step 3: Update NoOpDataProvider**

Change `NoOpDataProvider.java` line 22 return:
```java
return QueryResult.complete(new DataSetResult(List.of(), List.of()));
```

- [ ] **Step 4: Update SqlDataProvider**

Change `SqlDataProvider.java` line 55 return type to `QueryResult` and wrap:
```java
public QueryResult query(DataSetLookup lookup) {
    // ... existing code ...
    return QueryResult.complete(result);
}
```

- [ ] **Step 5: Update DataCacheService.queryCached()**

Change signature from `DataSetResult` to `QueryResult`:
```java
public QueryResult queryCached(String tenantId, DataSetLookup lookup, Supplier<QueryResult> loader) {
```
Update the `hashQuery` and cache entry handling accordingly. The cache stores `QueryResult` — both the data and the remaining ops are cached together.

- [ ] **Step 6: Update DataResource.query()**

Change return type and unwrap:
```java
@POST
@Path("/query")
public Response query(DataSetLookup lookup) {
    String tenantId = extractTenant();
    if (tenantId == null) {
        return missingTenantResponse();
    }

    DataProvider provider = resolveProvider(lookup.dataSetId());
    if (provider == null) {
        return Response.status(Response.Status.BAD_REQUEST)
            .entity(Map.of("error", "No provider found for dataset: " + lookup.dataSetId()))
            .build();
    }

    QueryResult queryResult = cacheService.queryCached(tenantId, lookup, () -> provider.query(lookup));
    return Response.ok(queryResult).build();
}
```

- [ ] **Step 7: Update all test files**

`DataResourceQueryTest.java` — change `TestDataProvider.query()` return to `QueryResult.complete(result)`. Update HTTP assertions to check `result.columns` and `result.rows` (nested under `result` now).

`NoOpDataProviderTest.java` — assert on `QueryResult`.

`SqlDataProviderTest.java` — change all `DataSetResult result = provider.query(lookup)` to `QueryResult qr = provider.query(lookup)` then `DataSetResult result = qr.result()`. Assert `qr.remainingOps()` is empty.

- [ ] **Step 8: Run tests**

Run: `/opt/homebrew/bin/mvn -pl backend/data,backend/data-sql -am test -q`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/data/ backend/data-sql/
git commit -m "feat(data): add QueryResult record, change DataProvider.query() return type

Refs #22"
```

---

### Task 2: DataQueryException and ExceptionMapper

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/DataQueryException.java`
- Create: `backend/data/src/main/java/io/casehub/pages/data/DataQueryExceptionMapper.java`
- Test: `backend/data/src/test/java/io/casehub/pages/data/DataQueryExceptionMapperTest.java`

**Interfaces:**
- Produces: `DataQueryException(String code, String message)` and `DataQueryException(String code, String message, Throwable cause)`
- Produces: Error codes: `INVALID_QUERY` → 400, `FETCH_FAILED` → 502, `RESULT_TOO_LARGE` → 413

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.pages.data;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.smallrye.jwt.build.Jwt;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class DataQueryExceptionMapperTest {

    @Alternative
    @Priority(2)
    @ApplicationScoped
    static class FailingProvider implements DataProvider {
        static String failCode = "INVALID_QUERY";
        static String failMessage = "bad metric name";

        @Override
        public String type() { return "failing"; }

        @Override
        public boolean canHandle(String dataSetId) {
            return "fail-dataset".equals(dataSetId);
        }

        @Override
        public QueryResult query(DataSetLookup lookup) {
            throw new DataQueryException(failCode, failMessage);
        }
    }

    private String token() {
        return Jwt.claims().subject("alice")
            .claim("tenant_id", "dev").groups(Set.of("user")).sign();
    }

    @Test
    void invalidQueryReturns400() {
        FailingProvider.failCode = "INVALID_QUERY";
        FailingProvider.failMessage = "bad metric name";

        given().auth().oauth2(token())
            .contentType(ContentType.JSON)
            .body(new DataSetLookup("fail-dataset", List.of(), null))
            .when().post("/api/dataset/query")
            .then()
            .statusCode(400)
            .body("code", equalTo("INVALID_QUERY"))
            .body("error", equalTo("bad metric name"));
    }

    @Test
    void fetchFailedReturns502() {
        FailingProvider.failCode = "FETCH_FAILED";
        FailingProvider.failMessage = "connection refused";

        given().auth().oauth2(token())
            .contentType(ContentType.JSON)
            .body(new DataSetLookup("fail-dataset", List.of(), null))
            .when().post("/api/dataset/query")
            .then()
            .statusCode(502)
            .body("code", equalTo("FETCH_FAILED"));
    }

    @Test
    void resultTooLargeReturns413() {
        FailingProvider.failCode = "RESULT_TOO_LARGE";
        FailingProvider.failMessage = "exceeded 10000 samples";

        given().auth().oauth2(token())
            .contentType(ContentType.JSON)
            .body(new DataSetLookup("fail-dataset", List.of(), null))
            .when().post("/api/dataset/query")
            .then()
            .statusCode(413)
            .body("code", equalTo("RESULT_TOO_LARGE"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=DataQueryExceptionMapperTest -q`
Expected: FAIL — `DataQueryException` class not found

- [ ] **Step 3: Write DataQueryException**

```java
package io.casehub.pages.data;

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

- [ ] **Step 4: Write DataQueryExceptionMapper**

```java
package io.casehub.pages.data;

import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;

import java.util.Map;

@Provider
public class DataQueryExceptionMapper implements ExceptionMapper<DataQueryException> {
    @Override
    public Response toResponse(DataQueryException e) {
        Response.Status status = switch (e.code()) {
            case "INVALID_QUERY" -> Response.Status.BAD_REQUEST;
            case "RESULT_TOO_LARGE" -> Response.Status.fromStatusCode(413);
            case "FETCH_FAILED" -> Response.Status.fromStatusCode(502);
            default -> Response.Status.INTERNAL_SERVER_ERROR;
        };
        return Response.status(status)
            .entity(Map.of("error", e.getMessage(), "code", e.code()))
            .type(MediaType.APPLICATION_JSON)
            .build();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -q`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/data/
git commit -m "feat(data): add DataQueryException with JAX-RS ExceptionMapper

Maps INVALID_QUERY → 400, FETCH_FAILED → 502, RESULT_TOO_LARGE → 413.

Refs #22"
```

---

## Batch 2: Prometheus Provider

### Task 3: Maven module scaffold and PrometheusConfig

**Files:**
- Create: `backend/data-prometheus/pom.xml`
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/PrometheusConfig.java`
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/DatasetConfig.java`
- Modify: `backend/pom.xml` (add `<module>data-prometheus</module>`)
- Create: `backend/data-prometheus/src/test/resources/application.properties`

**Interfaces:**
- Produces: `PrometheusConfig` — CDI config interface reading `casehub.pages.data.prometheus.*`
- Produces: `DatasetConfig` — per-dataset metric/step mapping

- [ ] **Step 1: Create pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-pages-backend</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-pages-data-prometheus</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Pages Data Prometheus Provider</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-data-backend</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add module to backend/pom.xml**

Add `<module>data-prometheus</module>` to the `<modules>` section.

- [ ] **Step 3: Create PrometheusConfig**

```java
package io.casehub.pages.data.prometheus;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

import java.util.Map;
import java.util.Optional;

@ConfigMapping(prefix = "casehub.pages.data.prometheus")
public interface PrometheusConfig {
    String endpoint();

    @WithDefault("none")
    String authType();

    Optional<String> authToken();
    Optional<String> authUsername();
    Optional<String> authPassword();

    @WithDefault("5s")
    String connectTimeout();

    @WithDefault("30s")
    String readTimeout();

    @WithDefault("10000")
    int maxSamples();

    Map<String, DatasetConfig> datasets();
}
```

- [ ] **Step 4: Create DatasetConfig**

```java
package io.casehub.pages.data.prometheus;

import io.smallrye.config.WithDefault;

public interface DatasetConfig {
    String metric();

    @WithDefault("60s")
    String step();
}
```

- [ ] **Step 5: Create test application.properties**

```properties
casehub.pages.data.prometheus.endpoint=http://localhost:9090
casehub.pages.data.prometheus.datasets.test-cpu.metric=node_cpu_seconds_total
casehub.pages.data.prometheus.datasets.test-cpu.step=5m
```

- [ ] **Step 6: Verify build**

Run: `/opt/homebrew/bin/mvn -pl backend/data-prometheus -am compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/data-prometheus/ backend/pom.xml
git commit -m "feat(data-prometheus): scaffold module with config mapping

Refs #22"
```

---

### Task 4: PromQL query builder

**Files:**
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/PromQLBuilder.java`
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/PromQLQuery.java`
- Test: `backend/data-prometheus/src/test/java/io/casehub/pages/data/prometheus/PromQLBuilderTest.java`

**Interfaces:**
- Consumes: `FilterExpression` sealed hierarchy, `FilterOp`, `GroupOp`, `SortOp`, `DataSetOp`
- Produces: `PromQLQuery(String expr, String start, String end, String step, List<DataSetOp> remainingOps)`
- Produces: `PromQLBuilder.build(String metric, List<DataSetOp> operations): PromQLQuery`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.pages.data.prometheus;

import io.casehub.pages.data.*;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class PromQLBuilderTest {

    @Test
    void bareMetricWithNoOps() {
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(), "60s");
        assertThat(q.expr()).isEqualTo("node_cpu");
        assertThat(q.remainingOps()).isEmpty();
    }

    @Test
    void equalsToFilterBecomesLabelMatcher() {
        FilterOp filter = new FilterOp(List.of(
            new FilterExpression.Unresolved("mode", "EQUALS_TO", List.of("idle"))
        ));
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(filter), "60s");
        assertThat(q.expr()).isEqualTo("node_cpu{mode=\"idle\"}");
        assertThat(q.remainingOps()).isEmpty();
    }

    @Test
    void notEqualsToFilterBecomesNegMatcher() {
        FilterOp filter = new FilterOp(List.of(
            new FilterExpression.Unresolved("mode", "NOT_EQUALS_TO", List.of("idle"))
        ));
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(filter), "60s");
        assertThat(q.expr()).isEqualTo("node_cpu{mode!=\"idle\"}");
    }

    @Test
    void likeToFilterBecomesRegexMatcher() {
        FilterOp filter = new FilterOp(List.of(
            new FilterExpression.Unresolved("job", "LIKE_TO", List.of("prod%"))
        ));
        PromQLQuery q = PromQLBuilder.build("http_requests", List.of(filter), "60s");
        assertThat(q.expr()).isEqualTo("http_requests{job=~\"^prod.*$\"}");
    }

    @Test
    void multipleFiltersAndInOneLabelBlock() {
        FilterOp filter = new FilterOp(List.of(
            new FilterExpression.And(List.of(
                new FilterExpression.Unresolved("mode", "EQUALS_TO", List.of("idle")),
                new FilterExpression.Unresolved("cpu", "EQUALS_TO", List.of("0"))
            ))
        ));
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(filter), "60s");
        assertThat(q.expr()).isEqualTo("node_cpu{mode=\"idle\",cpu=\"0\"}");
    }

    @Test
    void groupWithSumWrapsInAggregation() {
        GroupOp group = new GroupOp(
            new GroupingKey("instance", "instance", new GroupStrategy.Distinct(), 100, false, true, null, null),
            List.of(new ResultColumn.Aggregate("value", "total", new Aggregation("SUM", null))),
            null, null
        );
        PromQLQuery q = PromQLBuilder.build("http_requests", List.of(group), "60s");
        assertThat(q.expr()).isEqualTo("sum by (instance) (http_requests)");
    }

    @Test
    void sortOpGoesToRemainingOps() {
        SortOp sort = new SortOp(List.of(new SortColumn("value", true)));
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(sort), "60s");
        assertThat(q.expr()).isEqualTo("node_cpu");
        assertThat(q.remainingOps()).hasSize(1);
        assertThat(q.remainingOps().get(0)).isInstanceOf(SortOp.class);
    }

    @Test
    void untranslatableFilterGoesToRemainingOps() {
        FilterOp filter = new FilterOp(List.of(
            new FilterExpression.Numeric("value", java.util.Map.of("fn", "GREATER_THAN", "value", 100))
        ));
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(filter), "60s");
        assertThat(q.expr()).isEqualTo("node_cpu");
        assertThat(q.remainingOps()).hasSize(1);
    }

    @Test
    void timeFrameFilterSetsStartEnd() {
        FilterOp filter = new FilterOp(List.of(
            new FilterExpression.Unresolved("timestamp", "TIME_FRAME", List.of("now-1HOUR till now"))
        ));
        PromQLQuery q = PromQLBuilder.build("node_cpu", List.of(filter), "5m");
        assertThat(q.expr()).isEqualTo("node_cpu");
        assertThat(q.start()).isNotNull();
        assertThat(q.end()).isNotNull();
        assertThat(q.step()).isEqualTo("5m");
        assertThat(q.remainingOps()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -pl backend/data-prometheus -am test -Dtest=PromQLBuilderTest -q`
Expected: FAIL — `PromQLBuilder` not found

- [ ] **Step 3: Create PromQLQuery record**

```java
package io.casehub.pages.data.prometheus;

import io.casehub.pages.data.DataSetOp;
import java.util.List;

public record PromQLQuery(
    String expr,
    String start,
    String end,
    String step,
    List<DataSetOp> remainingOps
) {}
```

- [ ] **Step 4: Implement PromQLBuilder**

```java
package io.casehub.pages.data.prometheus;

import io.casehub.pages.data.*;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

public final class PromQLBuilder {

    private PromQLBuilder() {}

    public static PromQLQuery build(String metric, List<DataSetOp> operations, String defaultStep) {
        List<String> labelMatchers = new ArrayList<>();
        List<DataSetOp> remaining = new ArrayList<>();
        String aggregation = null;
        String groupByLabel = null;
        String start = null;
        String end = null;
        String step = defaultStep;

        for (DataSetOp op : operations) {
            switch (op) {
                case FilterOp f -> processFilter(f, labelMatchers, remaining, /* start/end holder */);
                case GroupOp g -> { /* process group */ }
                case SortOp s -> remaining.add(s);
            }
        }

        // Build expression
        StringBuilder expr = new StringBuilder();
        if (aggregation != null) {
            expr.append(aggregation).append(" by (").append(groupByLabel).append(") (");
        }
        expr.append(metric);
        if (!labelMatchers.isEmpty()) {
            expr.append("{").append(String.join(",", labelMatchers)).append("}");
        }
        if (aggregation != null) {
            expr.append(")");
        }

        return new PromQLQuery(expr.toString(), start, end, step, List.copyOf(remaining));
    }

    // ... private helper methods for filter/group translation ...
}
```

The full implementation handles `FilterExpression.Unresolved` with `EQUALS_TO`/`NOT_EQUALS_TO`/`LIKE_TO`, `FilterExpression.And` with all-translatable children, `TIME_FRAME` for start/end extraction, `GroupOp` with `Distinct` strategy and simple aggregations (SUM/AVG/MIN/MAX/COUNT), and routes everything else to `remainingOps`.

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -pl backend/data-prometheus -am test -Dtest=PromQLBuilderTest -q`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/data-prometheus/
git commit -m "feat(data-prometheus): PromQL query builder with filter/group translation

Translates FilterOp label matchers, GroupOp aggregations, and TIME_FRAME
to PromQL. Untranslatable ops return as remainingOps.

Refs #22"
```

---

### Task 5: PrometheusClient (HTTP API wrapper)

**Files:**
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/PrometheusClient.java`
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/PrometheusResponse.java`
- Test: `backend/data-prometheus/src/test/java/io/casehub/pages/data/prometheus/PrometheusClientTest.java`

**Interfaces:**
- Consumes: `PrometheusConfig` (endpoint, auth, timeouts)
- Consumes: `PromQLQuery` (expr, start, end, step)
- Produces: `PrometheusClient.rangeQuery(PromQLQuery): PrometheusResponse`
- Produces: `PrometheusClient.instantQuery(String expr): PrometheusResponse`
- Produces: `PrometheusResponse` — parsed Prometheus HTTP API JSON response

- [ ] **Step 1: Write the failing test with a mock HTTP server**

Use a Quarkus test with a `@WireMock` or a simple `HttpServer` to simulate the Prometheus API. Test that `rangeQuery` constructs the correct URL and parses the response.

```java
@QuarkusTest
class PrometheusClientTest {

    @Test
    void rangeQueryConstructsCorrectUrlAndParsesMatrix() {
        // Start a local HTTP server that returns a canned Prometheus response
        // Verify the client sends the right query params and parses the matrix
    }

    @Test
    void instantQueryParsesVector() { /* ... */ }

    @Test
    void authHeaderSentWhenConfigured() { /* ... */ }

    @Test
    void connectionTimeoutThrowsDataQueryException() { /* ... */ }

    @Test
    void invalidPromQLReturns422ThrowsDataQueryException() { /* ... */ }
}
```

- [ ] **Step 2: Implement PrometheusResponse**

```java
package io.casehub.pages.data.prometheus;

import java.util.List;
import java.util.Map;

public record PrometheusResponse(
    String status,
    Data data,
    String errorType,
    String error
) {
    public record Data(String resultType, List<Result> result) {}
    public record Result(Map<String, String> metric, List<List<Object>> values, List<Object> value) {}
}
```

- [ ] **Step 3: Implement PrometheusClient**

Uses `java.net.http.HttpClient` with timeouts from config. Builds query URL, sends request, parses JSON response via Jackson. Throws `DataQueryException` for errors.

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -pl backend/data-prometheus -am test -Dtest=PrometheusClientTest -q`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/data-prometheus/
git commit -m "feat(data-prometheus): HTTP client for Prometheus range/instant queries

Refs #22"
```

---

### Task 6: PrometheusDataProvider (CDI bean wiring it all together)

**Files:**
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/PrometheusDataProvider.java`
- Create: `backend/data-prometheus/src/main/java/io/casehub/pages/data/prometheus/ResponseMapper.java`
- Test: `backend/data-prometheus/src/test/java/io/casehub/pages/data/prometheus/PrometheusDataProviderTest.java`

**Interfaces:**
- Consumes: `PrometheusConfig`, `PrometheusClient`, `PromQLBuilder`
- Produces: `PrometheusDataProvider implements DataProvider` — CDI `@ApplicationScoped` bean

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class PrometheusDataProviderTest {

    @Inject
    PrometheusDataProvider provider;

    @Test
    void typeReturnsPrometheus() {
        assertThat(provider.type()).isEqualTo("prometheus");
    }

    @Test
    void canHandleConfiguredDataset() {
        assertThat(provider.canHandle("test-cpu")).isTrue();
    }

    @Test
    void cannotHandleUnknownDataset() {
        assertThat(provider.canHandle("unknown")).isFalse();
    }

    @Test
    void queryWithNoOpsReturnsInstantResult() {
        // Mock Prometheus returns a vector for instant query
        // Verify: columns include value + labels, no timestamp column
    }

    @Test
    void queryWithTimeFrameReturnsRangeResult() {
        // Mock Prometheus returns a matrix for range query
        // Verify: columns include timestamp (DATE), value, labels
    }

    @Test
    void sortOpPassedThroughAsRemainingOp() {
        // Add a SortOp, verify it appears in remainingOps
    }
}
```

- [ ] **Step 2: Implement ResponseMapper**

Converts `PrometheusResponse` → `DataSetResult`. Handles both matrix (range) and vector (instant) result types. Timestamps converted to ISO-8601, labeled as DATE type.

- [ ] **Step 3: Implement PrometheusDataProvider**

```java
@ApplicationScoped
public class PrometheusDataProvider implements DataProvider {

    @Inject PrometheusConfig config;
    @Inject PrometheusClient client;

    @Override
    public String type() { return "prometheus"; }

    @Override
    public boolean canHandle(String dataSetId) {
        return config.datasets().containsKey(dataSetId);
    }

    @Override
    public QueryResult query(DataSetLookup lookup) {
        DatasetConfig ds = config.datasets().get(lookup.dataSetId());
        if (ds == null) {
            throw new DataQueryException("INVALID_QUERY",
                "No Prometheus dataset configured: " + lookup.dataSetId());
        }

        PromQLQuery promql = PromQLBuilder.build(ds.metric(), lookup.operations(), ds.step());

        PrometheusResponse response;
        if (promql.start() != null && promql.end() != null) {
            response = client.rangeQuery(promql);
        } else {
            response = client.instantQuery(promql.expr());
        }

        // Cardinality protection
        long sampleCount = countSamples(response);
        if (sampleCount > config.maxSamples()) {
            throw new DataQueryException("RESULT_TOO_LARGE",
                "Response contains " + sampleCount + " samples (max: " + config.maxSamples() + ")");
        }

        DataSetResult result = ResponseMapper.toDataSetResult(response);
        return new QueryResult(result, promql.remainingOps());
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -pl backend/data-prometheus -am test -q`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/data-prometheus/
git commit -m "feat(data-prometheus): PrometheusDataProvider with filter/group translation

CDI-discovered provider that translates DataSetLookup to PromQL,
executes against Prometheus HTTP API, and returns untranslatable
ops as remainingOps for client-side application.

Refs #22"
```

---

## Batch 3: TypeScript Client Changes

### Task 7: TypeScript resolver — remainingOps handling

**Files:**
- Create: `packages/pages-data/src/dataset/ops-resolve.ts`
- Modify: `packages/pages-data/src/dataset/manager.ts:33-44` (extract `resolveOps`)
- Modify: `packages/pages-data/src/dataset/external/providers/server-query.ts`
- Modify: `packages/pages-data/src/dataset/external/resolver.ts:93-117`
- Modify: `packages/pages-data/src/index.ts` (export new module)
- Test: `packages/pages-data/src/dataset/ops-resolve.test.ts`
- Test: `packages/pages-data/src/dataset/external/providers/server-query.test.ts`

**Interfaces:**
- Produces: `resolveOps(ops: readonly DataSetOp[], columns: readonly Column[]): ResolvedDataSetOp[]`
- Produces: Updated `ServerQueryClient.query()` returning `{ dataset, remainingOps }`

- [ ] **Step 1: Write the test for ops-resolve**

```typescript
import { describe, it, expect } from "vitest";
import { resolveOps } from "./ops-resolve.js";
import type { Column } from "./types.js";
import { ColumnType } from "./types.js";

describe("resolveOps", () => {
  const columns: Column[] = [
    { id: "name" as any, name: "Name", type: ColumnType.LABEL },
    { id: "value" as any, name: "Value", type: ColumnType.NUMBER },
  ];

  it("passes group and sort ops through unchanged", () => {
    const ops = [
      { type: "group" as const, /* ... */ },
      { type: "sort" as const, /* ... */ },
    ];
    const resolved = resolveOps(ops, columns);
    expect(resolved).toHaveLength(2);
  });

  it("resolves filter ops against column types", () => {
    const ops = [{
      type: "filter" as const,
      expressions: [{ type: "unresolved", columnId: "value", fn: "GREATER_THAN", args: ["50"] }],
    }];
    const resolved = resolveOps(ops, columns);
    expect(resolved[0].type).toBe("filter");
  });
});
```

- [ ] **Step 2: Extract resolveOps to ops-resolve.ts**

Move the `resolveOps` function from `manager.ts` to a new file `ops-resolve.ts`. Update `manager.ts` to import from the new location.

- [ ] **Step 3: Update ServerQueryClient to return remainingOps**

```typescript
async query(lookup: DataSetLookup): Promise<{ dataset: TypedDataSet; remainingOps: DataSetOp[] }> {
    // ... existing fetch logic ...
    const body = await response.json() as {
        result: { columns: ...; rows: ... };
        remainingOps?: DataSetOp[];
    };
    return {
        dataset: toTypedDataSet(toDataSet(body.result)),
        remainingOps: body.remainingOps ?? [],
    };
}
```

- [ ] **Step 4: Update resolver serverQuery path**

In `resolver.ts`, the `serverQuery` branch applies `remainingOps`:

```typescript
if (def.serverQuery) {
    // ... existing setup ...
    const effectiveLookup = lookup ?? { dataSetId: def.uuid, operations: [] };
    const { dataset, remainingOps } = await client.query(effectiveLookup);
    const final = remainingOps.length > 0
        ? applyOps(dataset, resolveOps(remainingOps, dataset.columns))
        : dataset;
    ctx.manager.apply(def.uuid, { type: "snapshot", dataset: final });
    return { dataset: final, inferredColumns: false, source: "serverQuery" };
}
```

- [ ] **Step 5: Run TypeScript tests**

Run: `yarn --cwd packages/pages-data test`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/
git commit -m "feat(pages-data): handle remainingOps from server query response

Extract resolveOps to shared module. ServerQueryClient returns
remainingOps alongside dataset. Resolver applies remaining ops
client-side via applyOps().

Refs #22"
```

---

## References

- [2026-09-25-server-data-providers-design.md] — design spec this plan implements
- `backend/data/src/main/java/io/casehub/pages/data/DataProvider.java:6` — current SPI return type
- `backend/data/src/main/java/io/casehub/pages/data/DataSetOp.java` — sealed op hierarchy
- `backend/data/src/main/java/io/casehub/pages/data/FilterExpression.java` — sealed filter tree
- `backend/data-sql/pom.xml` — reference module structure
- `backend/data-sql/src/main/java/io/casehub/pages/data/sql/SqlDataProvider.java` — reference provider
- `packages/pages-data/src/dataset/manager.ts:33-44` — resolveOps to extract
- `packages/pages-data/src/dataset/external/resolver.ts:93-117` — serverQuery path
- `packages/pages-data/src/dataset/external/providers/server-query.ts` — client to update
- GitHub #22 — focal issue
