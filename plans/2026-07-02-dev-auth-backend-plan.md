# Dev-Auth & Backend Module Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional Quarkus backend to casehub-pages with auth (SmallRye JWT dev-auth), layout persistence (SPI + SQLite), and a data module scaffold — plus TypeScript frontend integration (REST layout store, login gate, identity widget).

**Architecture:** New `backend/` directory at repo root with 4 Maven modules inheriting `casehub-parent`. Frontend changes in existing `packages/pages-runtime` and `packages/pages-ui`. Backend is optional — the frontend works fully client-side without it.

**Tech Stack:** Quarkus 3.32.2, SmallRye JWT, SQLite (xerial JDBC + HikariCP + Flyway), TypeScript/Vitest, Web Components

## Global Constraints

- Java 21, Quarkus 3.32.2 (inherited from `casehub-parent` 0.2-SNAPSHOT)
- `io.casehub` groupId for all Maven artifacts
- Submodule folder naming: short names, no repo prefix (`auth/` not `pages-backend-auth/`)
- All CDI modules require `jandex-maven-plugin` 3.3.1 for bean discovery
- No dependency on `casehub-platform-api` from layout module — use `JsonWebToken` directly
- `@UnlessBuildProfile("prod")` for dev/test-only endpoints (not `@IfBuildProfile("dev")`)
- SQLite pattern: xerial JDBC + HikariCP + WAL mode + Flyway (reference: `casehub-platform/memory-sqlite/`)
- Flyway migration location: `classpath:db/{module}/migration/` (never `db/migration/{module}/`)
- TypeScript error handling: `load()` returns `null` on any error, `save()`/`delete()` catch and `console.warn`
- TDD: write failing test first, implement minimal code, verify pass, commit
- Skills: `java-dev` for Java tasks, `ts-dev` for TypeScript tasks, `superpowers:test-driven-development` before each task
- Design spec: `docs/superpowers/specs/2026-07-02-dev-auth-backend-design.md`

---

### Task 1: Maven Scaffolding + Data Module Scaffold

**Goal:** Create the Maven multi-module structure. All modules compile with `mvn verify`.

**Files:**
- Create: `backend/pom.xml` — parent POM, `io.casehub:casehub-pages-backend:0.1-SNAPSHOT`
- Create: `backend/auth/pom.xml` — `io.casehub:casehub-pages-auth`
- Create: `backend/layout/pom.xml` — `io.casehub:casehub-pages-layout`
- Create: `backend/layout-sqlite/pom.xml` — `io.casehub:casehub-pages-layout-sqlite`
- Create: `backend/data/pom.xml` — `io.casehub:casehub-pages-data-backend`
- Create: `backend/data/src/main/java/io/casehub/pages/data/package-info.java`

**Interfaces:**
- Produces: Maven module structure that all subsequent Java tasks depend on

**Reference files:**
- `casehub-parent` POM: `/Users/mdproctor/claude/casehub/parent/pom.xml` (groupId, version, properties)
- `casehub-platform-parent` POM: `/Users/mdproctor/claude/casehub/platform/pom.xml` (multi-module pattern, dependency management for sqlite-jdbc 3.53.1.0, HikariCP 7.0.2, jandex 3.3.1)
- `casehub-platform/memory-sqlite/pom.xml` (submodule POM pattern, dependency list)

- [ ] **Step 1: Create backend parent POM**

`backend/pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-pages-backend</artifactId>
    <version>0.1-SNAPSHOT</version>
    <packaging>pom</packaging>
    <name>CaseHub Pages Backend</name>

    <modules>
        <module>auth</module>
        <module>layout</module>
        <module>layout-sqlite</module>
        <module>data</module>
    </modules>

    <properties>
        <jandex-maven-plugin.version>3.3.1</jandex-maven-plugin.version>
        <sqlite-jdbc.version>3.53.1.0</sqlite-jdbc.version>
        <hikaricp.version>7.0.2</hikaricp.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>${quarkus.platform.groupId}</groupId>
                <artifactId>quarkus-bom</artifactId>
                <version>${quarkus.platform.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.xerial</groupId>
                <artifactId>sqlite-jdbc</artifactId>
                <version>${sqlite-jdbc.version}</version>
            </dependency>
            <dependency>
                <groupId>com.zaxxer</groupId>
                <artifactId>HikariCP</artifactId>
                <version>${hikaricp.version}</version>
            </dependency>
            <dependency>
                <groupId>io.casehub</groupId>
                <artifactId>casehub-pages-layout</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.casehub</groupId>
                <artifactId>casehub-pages-auth</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>io.smallrye</groupId>
                    <artifactId>jandex-maven-plugin</artifactId>
                    <version>${jandex-maven-plugin.version}</version>
                    <executions>
                        <execution>
                            <id>make-index</id>
                            <goals><goal>jandex</goal></goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>

    <distributionManagement>
        <repository>
            <id>github</id>
            <url>https://maven.pkg.github.com/casehubio/casehub-pages</url>
        </repository>
    </distributionManagement>
</project>
```

- [ ] **Step 2: Create auth module POM**

`backend/auth/pom.xml` — dependencies: `quarkus-rest-jackson`, `quarkus-smallrye-jwt`, `smallrye-jwt-build`, `quarkus-arc`. Test deps: `quarkus-junit5`, `rest-assured`. Plugin: `jandex-maven-plugin`.

- [ ] **Step 3: Create layout module POM**

`backend/layout/pom.xml` — dependencies: `quarkus-rest-jackson`, `quarkus-smallrye-jwt`, `quarkus-arc`. Test deps: `quarkus-junit5`, `rest-assured`, `casehub-pages-auth` (test scope, for token generation in tests).

- [ ] **Step 4: Create layout-sqlite module POM**

`backend/layout-sqlite/pom.xml` — dependencies: `casehub-pages-layout`, `sqlite-jdbc`, `HikariCP`, `flyway-core`, `quarkus-arc`, `microprofile-config-api` (provided). Test deps: `quarkus-junit5`, `rest-assured`, `casehub-pages-auth` (test), `casehub-pages-layout` (test-jar or test scope for NoOp). No `quarkus-maven-plugin` build goal.

- [ ] **Step 5: Create data module POM + package-info**

`backend/data/pom.xml` — minimal: `quarkus-arc`. `backend/data/src/main/java/io/casehub/pages/data/package-info.java`.

- [ ] **Step 6: Verify build**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml verify`
Expected: BUILD SUCCESS (all modules compile, no tests yet)

- [ ] **Step 7: Commit**

```
feat: Maven backend module scaffolding  Refs #88
```

---

### Task 2: Auth Module — JWT Dev-Auth Endpoint (TDD)

**Goal:** `POST /dev/auth/login` that mints JWTs in dev/test mode. Gated by `@UnlessBuildProfile("prod")`.

**Files:**
- Create: `backend/auth/src/test/java/io/casehub/pages/auth/DevAuthResourceTest.java`
- Create: `backend/auth/src/test/java/io/casehub/pages/auth/ProtectedTestResource.java`
- Create: `backend/auth/src/main/java/io/casehub/pages/auth/DevAuthResource.java`
- Create: `backend/auth/src/main/java/io/casehub/pages/auth/LoginRequest.java`
- Create: `backend/auth/src/main/java/io/casehub/pages/auth/TokenResponse.java`
- Create: `backend/auth/src/main/resources/application.properties`

**Interfaces:**
- Produces: `POST /dev/auth/login` — accepts `{"name":"alice","roles":["user","admin"]}`, returns `{"token":"eyJ..."}`
- JWT claims: `sub`=name, `groups`=roles, `iss`=casehub-dev, `tenant_id`=configurable (default "dev"), exp=24h

**Reference files:**
- REST resource pattern: `/Users/mdproctor/claude/casehub/clinical/runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- `@UnlessBuildProfile` usage: `/Users/mdproctor/claude/casehub/quarkmind/src/main/java/io/quarkmind/qa/` (QA resources)
- SmallRye JWT token building: `io.smallrye.jwt.build.Jwt` API

- [ ] **Step 1: Write test class `DevAuthResourceTest.java`**

```java
@QuarkusTest
class DevAuthResourceTest {
    // Test: POST with name+roles returns 200 + token with correct claims
    // Test: POST with name only defaults roles to ["user"]
    // Test: POST without name returns 400
    // Test: Token has correct sub, groups, iss, tenant_id, ~24h exp
    // Test: Round-trip — use returned token on @Authenticated endpoint
}
```

And `ProtectedTestResource.java` — a test-only `@Path("/test/protected")` `@Authenticated` resource that returns the JWT `sub` claim.

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml verify -pl auth`
Expected: compilation failures (classes don't exist yet)

- [ ] **Step 3: Implement LoginRequest, TokenResponse (records)**

```java
public record LoginRequest(String name, List<String> roles) {}
public record TokenResponse(String token) {}
```

- [ ] **Step 4: Implement DevAuthResource**

```java
@Path("/dev/auth")
@UnlessBuildProfile("prod")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class DevAuthResource {
    @ConfigProperty(name = "casehub.pages.auth.default-tenant", defaultValue = "dev")
    String defaultTenant;

    @POST
    @Path("/login")
    public Response login(LoginRequest request) {
        if (request == null || request.name() == null || request.name().isBlank()) {
            return Response.status(400).entity(Map.of("error", "name is required")).build();
        }
        List<String> roles = request.roles() != null ? request.roles() : List.of("user");
        String token = Jwt.claims()
            .subject(request.name())
            .groups(new HashSet<>(roles))
            .claim("tenant_id", defaultTenant)
            .expiresIn(Duration.ofHours(24))
            .sign();
        return Response.ok(new TokenResponse(token)).build();
    }
}
```

- [ ] **Step 5: Create application.properties**

```properties
%dev.mp.jwt.verify.issuer=casehub-dev
%test.mp.jwt.verify.issuer=casehub-dev
%dev.smallrye.jwt.new-token.issuer=casehub-dev
%test.smallrye.jwt.new-token.issuer=casehub-dev
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml verify -pl auth`
Expected: all tests GREEN

- [ ] **Step 7: Commit**

```
feat: dev-auth JWT login endpoint  Refs #88
```

---

### Task 3: Layout SPI + REST Module (TDD)

**Goal:** `LayoutPersistenceStore` SPI, `NoOpLayoutPersistenceStore` `@DefaultBean`, and `LayoutResource` REST endpoint with `@Authenticated`.

**Files:**
- Create: `backend/layout/src/main/java/io/casehub/pages/layout/LayoutPersistenceStore.java`
- Create: `backend/layout/src/main/java/io/casehub/pages/layout/NoOpLayoutPersistenceStore.java`
- Create: `backend/layout/src/main/java/io/casehub/pages/layout/LayoutResource.java`
- Create: `backend/layout/src/test/java/io/casehub/pages/layout/LayoutResourceTest.java`
- Create: `backend/layout/src/test/java/io/casehub/pages/layout/NoOpLayoutPersistenceStoreTest.java`
- Create: `backend/layout/src/main/resources/application.properties`

**Interfaces:**
- Produces: `LayoutPersistenceStore` SPI — `load(key, tenantId, userId)`, `save(key, tenantId, userId, payload)`, `delete(key, tenantId, userId)`
- Produces: REST — `GET/PUT/DELETE /api/layouts/{key}` with `@Authenticated`, tenant from JWT `tenant_id` claim, user from JWT `sub`
- Consumes: SmallRye JWT `JsonWebToken` directly (NOT `casehub-platform-api`)

**Reference files:**
- LayoutStore TS interface: `packages/pages-runtime/src/layout-store.ts` (the contract the REST endpoints serve)
- Platform auth pattern: `/Users/mdproctor/claude/casehub/platform/oidc/src/main/java/io/casehub/platform/oidc/SecurityIdentityCurrentPrincipal.java` (JWT claim extraction pattern — but we use JsonWebToken directly, not CurrentPrincipal)

- [ ] **Step 1: Write NoOpLayoutPersistenceStoreTest**

Unit test: `load()` → `Optional.empty()`, `save()` is no-op, `delete()` is no-op.

- [ ] **Step 2: Write LayoutResourceTest**

```java
@QuarkusTest
class LayoutResourceTest {
    // Test: GET /api/layouts/{key} with valid JWT → 204 (no-op store)
    // Test: PUT /api/layouts/{key} with valid JWT + body → 204
    // Test: DELETE /api/layouts/{key} with valid JWT → 204
    // Test: All endpoints → 401 without JWT
    // Test: Missing tenant_id claim → 401 with descriptive message
    // Token generation: use Jwt.claims().subject("alice").claim("tenant_id","dev").sign()
}
```

- [ ] **Step 3: Run tests — verify they fail**

- [ ] **Step 4: Implement LayoutPersistenceStore SPI**

```java
public interface LayoutPersistenceStore {
    Optional<String> load(String key, String tenantId, String userId);
    void save(String key, String tenantId, String userId, String payload);
    void delete(String key, String tenantId, String userId);
}
```

- [ ] **Step 5: Implement NoOpLayoutPersistenceStore**

```java
@DefaultBean
@ApplicationScoped
public class NoOpLayoutPersistenceStore implements LayoutPersistenceStore {
    // load → Optional.empty(), save/delete → no-op
}
```

- [ ] **Step 6: Implement LayoutResource**

```java
@Path("/api/layouts")
@Authenticated
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class LayoutResource {
    @Inject LayoutPersistenceStore store;
    @Inject JsonWebToken jwt;
    @ConfigProperty(name = "casehub.pages.layout.tenant-claim", defaultValue = "tenant_id")
    String tenantClaim;

    @GET @Path("/{key}")
    public Response load(@PathParam("key") String key) {
        String tenantId = extractTenant();
        if (tenantId == null) return missingTenantResponse();
        Optional<String> payload = store.load(key, tenantId, jwt.getSubject());
        return payload.map(p -> Response.ok(p).build())
                      .orElse(Response.noContent().build());
    }
    // PUT, DELETE similar — extract tenant, delegate to store
}
```

- [ ] **Step 7: Run tests — verify they pass**

- [ ] **Step 8: Commit**

```
feat: layout persistence SPI + REST endpoint  Refs #88
```

---

### Task 4: Layout SQLite Module (TDD)

**Goal:** `SqliteLayoutPersistenceStore` — SQLite persistence backend that beats the no-op `@DefaultBean` via CDI priority.

**Files:**
- Create: `backend/layout-sqlite/src/main/java/io/casehub/pages/layout/sqlite/SqliteLayoutPersistenceStore.java`
- Create: `backend/layout-sqlite/src/main/resources/db/layout-sqlite/migration/V1__layout_state.sql`
- Create: `backend/layout-sqlite/src/test/java/io/casehub/pages/layout/sqlite/SqliteLayoutPersistenceStoreTest.java`
- Create: `backend/layout-sqlite/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `LayoutPersistenceStore` SPI from Task 3
- Produces: Working SQLite persistence, activated by classpath presence

**Reference files:**
- SqliteMemoryStore: `/Users/mdproctor/claude/casehub/platform/memory-sqlite/src/main/java/io/casehub/platform/memory/sqlite/SqliteMemoryStore.java` (exact pattern to follow for HikariCP + WAL + Flyway + lifecycle)
- Flyway migration: `/Users/mdproctor/claude/casehub/platform/memory-sqlite/src/main/resources/db/memory-sqlite/migration/V1__memory_sqlite_entry.sql`

- [ ] **Step 1: Write SqliteLayoutPersistenceStoreTest**

```java
@QuarkusTest
class SqliteLayoutPersistenceStoreTest {
    @Inject LayoutPersistenceStore store;
    // Test: round-trip save→load returns same payload
    // Test: load missing key → Optional.empty()
    // Test: delete→load → Optional.empty()
    // Test: save overwrites (upsert)
    // Test: tenant isolation — same key+user, different tenant → independent
    // Test: user isolation — same key+tenant, different user → independent
}
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Create Flyway migration**

`db/layout-sqlite/migration/V1__layout_state.sql`:
```sql
CREATE TABLE IF NOT EXISTS layout_state (
    key        TEXT NOT NULL,
    tenant_id  TEXT NOT NULL,
    user_id    TEXT NOT NULL,
    payload    TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    PRIMARY KEY (key, tenant_id, user_id)
);
```

- [ ] **Step 4: Implement SqliteLayoutPersistenceStore**

Follow the SqliteMemoryStore pattern exactly:
- `@ApplicationScoped` (Tier 2 — beats `@DefaultBean` NoOp)
- `@ConfigProperty` for path, pool size, busy timeout
- `@PostConstruct init()`: SQLiteConfig (WAL, NORMAL sync, busyTimeout=5000, cacheSize=64000) → SQLiteDataSource → HikariDataSource → Flyway
- `@PreDestroy shutdown()`: close datasource
- SQL: `INSERT ... ON CONFLICT DO UPDATE` for upsert, standard SELECT/DELETE

- [ ] **Step 5: Create test application.properties**

```properties
casehub.pages.layout.sqlite.path=:memory:
%test.mp.jwt.verify.issuer=casehub-dev
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml verify -pl layout-sqlite`

- [ ] **Step 7: Commit**

```
feat: SQLite layout persistence backend  Refs #88
```

---

### Task 5: TypeScript — REST Layout Store + Auth Utilities (TDD)

**Goal:** `createRestLayoutStore()`, `createDevAuthTokenFn()`, configurable `layoutSaveDelayMs`, `DevAuthConfig` in `SiteOptions`.

**Files:**
- Create: `packages/pages-runtime/src/rest-layout-store.ts`
- Create: `packages/pages-runtime/src/rest-layout-store.test.ts`
- Create: `packages/pages-runtime/src/dev-auth.ts`
- Create: `packages/pages-runtime/src/dev-auth.test.ts`
- Modify: `packages/pages-runtime/src/site.ts` — add `layoutSaveDelayMs` + `devAuth` to `SiteOptions`, make debounce delay configurable
- Modify: `packages/pages-runtime/src/index.ts` — add exports

**Interfaces:**
- Consumes: `LayoutStore` interface from `layout-store.ts`
- Consumes: `LayoutState` type from `@casehubio/pages-component`
- Produces: `createRestLayoutStore(baseUrl, tokenFn) → LayoutStore`
- Produces: `createDevAuthTokenFn(sessionStorageKey?) → () => string | null`
- Produces: `DevAuthConfig` type (`{ backendUrl: string; identities?: string[] }`)
- Produces: `SiteOptions.layoutSaveDelayMs` (optional, default 500)
- Produces: `SiteOptions.devAuth` (optional `DevAuthConfig`)
- Produces: `pages-auth-expired` CustomEvent on `document` on 401

**Reference files:**
- LayoutStore: `packages/pages-runtime/src/layout-store.ts` (interface + error handling pattern)
- RestAdapter: `packages/pages-runtime/src/adapters/rest-adapter.ts` (fetch pattern)
- Site debounce: `packages/pages-runtime/src/site.ts` lines 874-881 (hardcoded 500ms → configurable)
- Test pattern: `packages/pages-runtime/src/adapters/rest-adapter.test.ts` (vi.fn fetch mocking)

- [ ] **Step 1: Write rest-layout-store.test.ts**

Test with `vi.fn()` mock fetch:
- `load()` returns `LayoutState` on 200
- `load()` returns `null` on 404, network error, invalid JSON
- `save()` sends PUT with correct URL/headers/body
- `save()` catches errors, calls `console.warn`
- `delete()` sends DELETE, catches errors
- Authorization header present when `tokenFn` returns token
- No Authorization header when `tokenFn` returns null
- On 401: dispatches `pages-auth-expired` on `document`

- [ ] **Step 2: Write dev-auth.test.ts**

- `createDevAuthTokenFn()` reads from sessionStorage
- Returns null when no token stored
- Custom key parameter works

- [ ] **Step 3: Run tests — verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test`

- [ ] **Step 4: Implement rest-layout-store.ts**

```typescript
export function createRestLayoutStore(
  baseUrl: string,
  tokenFn: () => string | null,
): LayoutStore {
  return {
    async load(key) {
      try {
        const headers: Record<string, string> = {};
        const token = tokenFn();
        if (token) headers["Authorization"] = `Bearer ${token}`;
        const response = await fetch(
          `${baseUrl}/api/layouts/${encodeURIComponent(key)}`, { headers });
        if (response.status === 401) {
          document.dispatchEvent(new CustomEvent("pages-auth-expired"));
          return null;
        }
        if (!response.ok) return null;
        const text = await response.text();
        if (!text) return null;
        return JSON.parse(text) as LayoutState;
      } catch { return null; }
    },
    async save(key, state) { /* PUT, catch, warn, dispatch on 401 */ },
    async delete(key) { /* DELETE, catch, warn, dispatch on 401 */ },
  };
}
```

- [ ] **Step 5: Implement dev-auth.ts**

```typescript
export interface DevAuthConfig {
  readonly backendUrl: string;
  readonly identities?: readonly string[];
}

export function createDevAuthTokenFn(
  sessionStorageKey = "pages-dev-auth-token",
): () => string | null {
  return () => {
    try { return sessionStorage.getItem(sessionStorageKey); }
    catch { return null; }
  };
}
```

- [ ] **Step 6: Modify site.ts — configurable debounce + DevAuthConfig**

In `SiteOptions` interface, add:
```typescript
readonly layoutSaveDelayMs?: number;
readonly devAuth?: DevAuthConfig;
```

In `scheduleLayoutSave()`, change hardcoded `500` to `options?.layoutSaveDelayMs ?? 500`.

- [ ] **Step 7: Add exports to index.ts**

```typescript
export { createRestLayoutStore } from "./rest-layout-store.js";
export { createDevAuthTokenFn } from "./dev-auth.js";
export type { DevAuthConfig } from "./dev-auth.js";
```

- [ ] **Step 8: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test`

- [ ] **Step 9: Commit**

```
feat: REST layout store + auth utilities  Refs #88
```

---

### Task 6: TypeScript — Login Gate + Identity Widget (TDD)

**Goal:** `<pages-dev-auth>` and `<pages-identity>` Web Components in `pages-ui`.

**Files:**
- Create: `packages/pages-ui/src/auth/dev-auth-gate.ts`
- Create: `packages/pages-ui/src/auth/dev-auth-gate.test.ts`
- Create: `packages/pages-ui/src/auth/identity-widget.ts`
- Create: `packages/pages-ui/src/auth/identity-widget.test.ts`
- Create: `packages/pages-ui/src/auth/index.ts`
- Modify: `packages/pages-ui/src/index.ts` — add auth exports
- May need: `packages/pages-ui/vitest.config.ts` if not present (follow pages-runtime pattern)

**Interfaces:**
- Consumes: `DevAuthConfig` type from Task 5 (`pages-runtime`)
- Consumes: `pages-auth-expired` CustomEvent from Task 5
- Consumes: sessionStorage key `pages-dev-auth-token` (public contract)
- Produces: `<pages-dev-auth>` custom element — login gate overlay
- Produces: `<pages-identity>` custom element — identity display + switcher

**Reference files:**
- pages-ui existing structure: `packages/pages-ui/src/` (model/, parser/, dsl/)
- Component model: `packages/pages-component/src/model/types.ts` (LayoutState, AccessControl)
- pages-runtime test config: `packages/pages-runtime/vitest.config.ts` (for happy-dom setup)

- [ ] **Step 1: Set up vitest for pages-ui if needed**

Check if `packages/pages-ui` already has vitest configured. If not, create `vitest.config.ts` following the pages-runtime pattern with `happy-dom` environment.

- [ ] **Step 2: Write dev-auth-gate.test.ts**

- Renders nothing when sessionStorage has valid JWT
- Renders overlay when no JWT
- Renders overlay when JWT expired (mock `exp` in past)
- Base64url decoding works correctly
- On selection: POST to `/dev/auth/login`
- On success: stores token in sessionStorage, dismisses overlay
- Dropdown renders when `identities` attribute set
- Free-text input when no identities
- `pages-auth-expired` event triggers re-render

- [ ] **Step 3: Write identity-widget.test.ts**

- Shows current user name from JWT `sub`
- Click opens picker popover
- Identity switch: POST + update sessionStorage

- [ ] **Step 4: Run tests — verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test`

- [ ] **Step 5: Implement dev-auth-gate.ts**

```typescript
const SESSION_KEY = "pages-dev-auth-token";

export class PagesDevAuth extends HTMLElement {
  static get observedAttributes(): string[] {
    return ["backend-url", "identities"];
  }

  connectedCallback(): void {
    document.addEventListener("pages-auth-expired", this.handleAuthExpired);
    this.checkAuth();
  }

  disconnectedCallback(): void {
    document.removeEventListener("pages-auth-expired", this.handleAuthExpired);
  }

  private handleAuthExpired = (): void => {
    try { sessionStorage.removeItem(SESSION_KEY); } catch { /* */ }
    this.renderOverlay();
  };

  private isExpired(token: string): boolean {
    try {
      const parts = token.split(".");
      if (parts.length !== 3) return true;
      const payload = JSON.parse(
        atob(parts[1]!.replace(/-/g, "+").replace(/_/g, "/")));
      return typeof payload.exp === "number" && payload.exp < Date.now() / 1000;
    } catch { return true; }
  }

  private async login(name: string): Promise<void> {
    const backendUrl = this.getAttribute("backend-url") ?? "";
    const resp = await fetch(`${backendUrl}/dev/auth/login`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name }),
    });
    if (resp.ok) {
      const { token } = await resp.json() as { token: string };
      sessionStorage.setItem(SESSION_KEY, token);
      this.dismissOverlay();
    }
  }
  // renderOverlay(), dismissOverlay(), checkAuth() ...
}

customElements.define("pages-dev-auth", PagesDevAuth);
```

- [ ] **Step 6: Implement identity-widget.ts**

`<pages-identity>` — reads JWT `sub` from sessionStorage, displays name, click to switch.

- [ ] **Step 7: Create auth/index.ts + update pages-ui index.ts**

```typescript
// auth/index.ts
export { PagesDevAuth } from "./dev-auth-gate.js";
export { PagesIdentity } from "./identity-widget.js";

// pages-ui/src/index.ts — add:
export { PagesDevAuth, PagesIdentity } from "./auth/index.js";
```

- [ ] **Step 8: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test`

- [ ] **Step 9: Commit**

```
feat: login gate + identity widget components  Refs #88
```

---

## Execution Order

```
Task 1 (Maven scaffolding) → Task 2 (Auth) → Task 3 (Layout SPI) → Task 4 (SQLite)
                                                                          ↓
Task 5 (TS REST store + auth utils) → Task 6 (TS login gate + widget)
```

Java track (1→2→3→4) and TypeScript track (5→6) are independent after Task 1. Recommended sequential order: 1, 2, 3, 4, 5, 6.

## Verification

After all tasks:
1. `mvn -f backend/pom.xml verify` — all Java modules green
2. `yarn workspace @casehubio/pages-runtime run test` — all TS runtime tests green
3. `yarn workspace @casehubio/pages-ui run test` — all TS UI tests green
4. `yarn typecheck` — cross-package type check passes
5. `yarn lint` — ESLint passes

## Platform Coherence Review (post-implementation)

Before closing the branch:
1. Re-read PLATFORM.md capability ownership — update the `casehub-pages` entry to reflect new backend modules
2. Verify submodule folder naming (short names, no prefix)
3. Verify CDI annotations match the persistence-backend-cdi-priority protocol
4. Verify Flyway migration location follows `classpath:db/{module}/migration/` convention
5. Create follow-up GitHub issues for: data module full design, layout JPA backend, cross-repo consumer updates
