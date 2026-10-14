# Reference Platform Playbook Examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #548 — Reference platform playbook examples instead of local copies
**Issue group:** #548

**Goal:** Replace the local `playbooks/` directory with a Maven dependency on `casehub-platform-playbook-catalog` and a REST endpoint, eliminating drift between pages and platform.

**Architecture:** Platform already publishes `casehub-platform-playbook-catalog` with 57 playbook YAMLs + manifest on the classpath under `/playbooks/`. Pages adds the dependency to the examples server, creates a thin `@Path("/api/playbooks")` REST endpoint that reads from classpath and serves content to browser consumers. All TS fetch paths update from `/playbooks/...` to `/api/playbooks/...`. The local `playbooks/` directory is removed from git.

**Tech Stack:** Java 21, Quarkus (JAX-RS), TypeScript, Vite, Maven

## Global Constraints

- Version of `casehub-platform-playbook-catalog` comes from the parent BOM — no version tag in the dependency declaration
- Playbook YAML content must be byte-identical between platform and pages — the single JAR guarantees this
- All browser consumers use `fetch()` — the REST endpoint is the single serving path
- The examples Quarkus server must be running for any playbook access during dev

---

## Batch 1: REST endpoint and dependency wiring

### Task 1: Add dependency and create PlaybookResource

**Files:**
- Modify: `examples/server/pom.xml` — add `casehub-platform-playbook-catalog` dependency
- Create: `examples/server/src/main/java/io/casehub/pages/examples/PlaybookResource.java`
- Create: `examples/server/src/test/java/io/casehub/pages/examples/PlaybookResourceTest.java`

**Interfaces:**
- Consumes: `casehub-platform-playbook-catalog` JAR (classpath resources under `/playbooks/`)
- Produces: `GET /api/playbooks/{path:.+}` — serves manifest.json as `application/json`, `.yaml` files as `text/yaml`, 404 for missing resources. `Cache-Control: max-age=300` header on all responses.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.pages.examples;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class PlaybookResourceTest {

    @Test
    void manifest_returnsJson() {
        given()
            .when().get("/api/playbooks/manifest.json")
            .then()
            .statusCode(200)
            .contentType("application/json")
            .header("Cache-Control", containsString("max-age=300"))
            .body("version", is("1.0.0"))
            .body("categories.size()", greaterThan(0));
    }

    @Test
    void yamlPlaybook_returnsYaml() {
        given()
            .when().get("/api/playbooks/flow-control/sequential.playbook.yaml")
            .then()
            .statusCode(200)
            .contentType("text/yaml")
            .header("Cache-Control", containsString("max-age=300"))
            .body(containsString("scenario:"));
    }

    @Test
    void missingPlaybook_returns404() {
        given()
            .when().get("/api/playbooks/nonexistent/missing.yaml")
            .then()
            .statusCode(404);
    }

    @Test
    void pathTraversal_returns404() {
        given()
            .when().get("/api/playbooks/../../../etc/passwd")
            .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml test -Dtest=PlaybookResourceTest -pl .`
Expected: FAIL — `PlaybookResource` class does not exist

- [ ] **Step 3: Add the Maven dependency**

In `examples/server/pom.xml`, add inside the `<dependencies>` block (before `<!-- Test -->`):

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-playbook-catalog</artifactId>
</dependency>
```

- [ ] **Step 4: Write the PlaybookResource implementation**

```java
package io.casehub.pages.examples;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.core.Response;
import java.io.IOException;
import java.io.InputStream;

@Path("/api/playbooks")
public class PlaybookResource {

    @GET
    @Path("{path:.+}")
    public Response serve(@PathParam("path") String path) {
        String normalized = path.replace("..", "");
        String classpathPath = "/playbooks/" + normalized;
        InputStream stream = getClass().getResourceAsStream(classpathPath);
        if (stream == null) {
            return Response.status(404).build();
        }
        try {
            byte[] content = stream.readAllBytes();
            String contentType = normalized.endsWith(".json") ? "application/json" : "text/yaml";
            return Response.ok(content, contentType)
                    .header("Cache-Control", "max-age=300")
                    .build();
        } catch (IOException e) {
            return Response.serverError().build();
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml test -Dtest=PlaybookResourceTest -pl .`
Expected: PASS — all 4 tests green

- [ ] **Step 6: Commit**

```bash
git add examples/server/pom.xml
git add examples/server/src/main/java/io/casehub/pages/examples/PlaybookResource.java
git add examples/server/src/test/java/io/casehub/pages/examples/PlaybookResourceTest.java
git commit -m "feat(#548): add PlaybookResource serving platform catalog via classpath"
```

---

## Batch 2: Update all browser consumers

### Task 2: Update TS fetch paths and Vite proxy

**Files:**
- Modify: `packages/pages-aria/src/controller/playbook-controller.ts:629` — update fetch URL
- Modify: `packages/pages-aria/src/controller/playbook-yaml-viewer.ts:244` — update fetch URL
- Modify: `packages/pages-aria/src/tutorial/tutorial-host.ts:226-234` — simplify fetch path
- Modify: `examples/samples/Scenarios/Composition.ts:247,256` — update fetch URLs
- Modify: `examples/samples/Scenarios/Data Injection.ts:197,206` — update fetch URLs
- Modify: `examples/samples/Scenarios/Step Modifiers.ts:212,221` — update fetch URLs
- Modify: `examples/samples/Scenarios/Synchronization.ts:190,199` — update fetch URLs
- Modify: `examples/scripts/copy-samples.js:47-52` — remove playbooks copy block
- Modify: `tutorials/vite.config.ts` — add proxy rule

**Interfaces:**
- Consumes: `GET /api/playbooks/{path}` from Task 1
- Produces: All browser consumers fetch from `/api/playbooks/...` instead of `/playbooks/...`

- [ ] **Step 1: Update playbook-controller.ts**

At line 629, change:
```typescript
const resp = await fetch(`${this._conn.restBase}/playbooks/${this.scenario}.yaml`, { cache: 'no-store' });
```
to:
```typescript
const resp = await fetch(`${this._conn.restBase}/api/playbooks/${this.scenario}.yaml`, { cache: 'no-store' });
```

- [ ] **Step 2: Update playbook-yaml-viewer.ts**

At line 244, change:
```typescript
const resp = await fetch(`${base}/playbooks/${this.scenario}.yaml`);
```
to:
```typescript
const resp = await fetch(`${base}/api/playbooks/${this.scenario}.yaml`);
```

- [ ] **Step 3: Update tutorial-host.ts**

At lines 226-234, replace `_loadScenarioRef`:
```typescript
private async _loadScenarioRef(ref: string): Promise<string | null> {
    if (this._scenarioRefCache.has(ref)) return this._scenarioRefCache.get(ref)!;
    try {
      const resp = await fetch(`/api/playbooks/${ref}`);
      if (!resp.ok) return null;
      const text = await resp.text();
      this._scenarioRefCache.set(ref, text);
      return text;
    } catch {
      return null;
    }
  }
```

- [ ] **Step 4: Update Composition.ts**

At lines 247 and 256, change:
```typescript
fetch('../../playbooks/manifest.json')
```
to:
```typescript
fetch('/api/playbooks/manifest.json')
```

And:
```typescript
return fetch('../../playbooks/' + entry.file)
```
to:
```typescript
return fetch('/api/playbooks/' + entry.file)
```

- [ ] **Step 5: Update Data Injection.ts**

Same pattern as Step 4. At lines 197 and 206:
```typescript
fetch('../../playbooks/manifest.json')  →  fetch('/api/playbooks/manifest.json')
fetch('../../playbooks/' + entry.file)  →  fetch('/api/playbooks/' + entry.file)
```

- [ ] **Step 6: Update Step Modifiers.ts**

Same pattern. At lines 212 and 221:
```typescript
fetch('../../playbooks/manifest.json')  →  fetch('/api/playbooks/manifest.json')
fetch('../../playbooks/' + entry.file)  →  fetch('/api/playbooks/' + entry.file)
```

- [ ] **Step 7: Update Synchronization.ts**

Same pattern. At lines 190 and 199:
```typescript
fetch('../../playbooks/manifest.json')  →  fetch('/api/playbooks/manifest.json')
fetch('../../playbooks/' + entry.file)  →  fetch('/api/playbooks/' + entry.file)
```

- [ ] **Step 8: Remove playbooks copy from copy-samples.js**

In `examples/scripts/copy-samples.js`, remove lines 47-52:
```javascript
// Copy playbooks (shared scenario YAML files)
const playbooksDir = path.join(__dirname, '../../playbooks');
if (fs.existsSync(playbooksDir)) {
  console.log('Copying playbooks...');
  copyRecursive(playbooksDir, path.join(targetDir, 'playbooks'));
}
```

- [ ] **Step 9: Add Vite proxy to tutorials**

In `tutorials/vite.config.ts`, update the `server` config:
```typescript
server: {
    port: 5174,
    open: true,
    proxy: {
      '/api/playbooks': 'http://localhost:8080',
    },
  },
```

- [ ] **Step 10: Run TS type check**

Run: `yarn tsc --noEmit`
Expected: PASS — no type errors (changes are string-only, no type impact)

- [ ] **Step 11: Commit**

```bash
git add packages/pages-aria/src/controller/playbook-controller.ts
git add packages/pages-aria/src/controller/playbook-yaml-viewer.ts
git add packages/pages-aria/src/tutorial/tutorial-host.ts
git add examples/samples/Scenarios/
git add examples/scripts/copy-samples.js
git add tutorials/vite.config.ts
git commit -m "feat(#548): update all consumers to fetch from /api/playbooks REST endpoint"
```

---

## Batch 3: Remove local playbooks and verify

### Task 3: Delete local playbooks directory and verify end-to-end

**Files:**
- Delete: `playbooks/` directory (57 YAML files + manifest.json) — use `git rm -r`
- Modify: `examples/dist/playbooks/` — verify this directory is no longer generated

**Interfaces:**
- Consumes: All consumer updates from Task 2 are in place
- Produces: Clean repo with no local playbook copy

- [ ] **Step 1: Delete the local playbooks directory**

```bash
git rm -r playbooks/
```

- [ ] **Step 2: Verify examples dist build no longer copies playbooks**

Run: `node examples/scripts/copy-samples.js`
Expected: Output does NOT include "Copying playbooks..."

- [ ] **Step 3: Run the full test suite**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml test`
Expected: PASS — PlaybookResourceTest passes, all existing tests pass

- [ ] **Step 4: Run TS tests for pages-aria**

Run: `yarn workspace @casehubio/pages-aria test`
Expected: PASS — no test references local playbooks paths

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#548): remove local playbooks directory — platform catalog is canonical source"
```

## References

- [2026-10-14-platform-playbook-ref-design.md] — design spec this plan implements
- [platform/playbook-catalog/pom.xml] — platform Maven module packaging playbooks
- [examples/server/pom.xml] — Quarkus examples server dependencies
- [examples/server/src/main/java/io/casehub/pages/examples/DemoResource.java] — existing REST resource pattern
- [playbook-controller.ts:629] — server-based fetch consumer
- [playbook-yaml-viewer.ts:244] — connection-based fetch consumer
- [tutorial-host.ts:229] — content-relative fetch consumer
- [examples/scripts/copy-samples.js:48] — filesystem copy to dist
- [examples/samples/Scenarios/Composition.ts:247] — example scenario fetch
- [GitHub #548] — focal issue
- [GitHub platform#609] — platform playbook examples (canonical source)
