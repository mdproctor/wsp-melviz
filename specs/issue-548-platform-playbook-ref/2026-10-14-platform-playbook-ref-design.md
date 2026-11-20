# Reference platform playbook examples instead of local copies

**Issue:** casehubio/casehub-pages#548
**Date:** 2026-10-14
**Status:** Draft

## Problem

Pages maintains a local `playbooks/` directory with 57 YAML examples + `manifest.json` that is identical to platform's canonical copy. Both repos can diverge silently — there's no mechanism to detect or prevent drift.

## Solution

Replace the local `playbooks/` directory with a Maven dependency on `casehub-platform-playbook-catalog` and a thin REST endpoint that serves playbooks to browser consumers.

## Architecture

```
platform                          pages
┌─────────────────────────┐       ┌──────────────────────────────────┐
│ playbook-catalog/       │       │ examples/server/                 │
│   src/main/resources/   │  .m2  │   PlaybookResource.java          │
│     playbooks/          │ ────► │     @Path("/api/playbooks")      │
│       manifest.json     │       │     reads from classpath          │
│       flow-control/     │       │     serves YAML + JSON            │
│       composition/      │       │                                  │
│       ...               │       │ Browser consumers:               │
└─────────────────────────┘       │   playbook-controller.ts         │
                                  │   playbook-yaml-viewer.ts        │
                                  │   tutorial-host.ts               │
                                  │   examples/samples/Scenarios/*.ts │
                                  │     all fetch /api/playbooks/... │
                                  └──────────────────────────────────┘
```

## Changes

### 1. Add dependency to examples-server

In `examples/server/pom.xml`, add:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-playbook-catalog</artifactId>
</dependency>
```

Version comes from the parent BOM — no version tag needed.

### 2. REST endpoint — PlaybookResource

New file: `examples/server/src/main/java/.../PlaybookResource.java`

- `@Path("/api/playbooks")`
- `GET /api/playbooks/manifest.json` → reads `/playbooks/manifest.json` from classpath, returns as `application/json`
- `GET /api/playbooks/{category}/{file}` → reads `/playbooks/{category}/{file}` from classpath, returns as `text/yaml`
- Caching headers: `Cache-Control: max-age=300` (playbooks are static within a version)
- 404 if the resource doesn't exist on the classpath

### 3. Update browser consumers

Three TS files update their fetch URLs:

**playbook-controller.ts** (line 629):
```typescript
// Before:
fetch(`${this._conn.restBase}/playbooks/${this.scenario}.yaml`, ...)
// After:
fetch(`${this._conn.restBase}/api/playbooks/${this.scenario}.yaml`, ...)
```

**playbook-yaml-viewer.ts** (line 244):
```typescript
// Before:
fetch(`${base}/playbooks/${this.scenario}.yaml`)
// After:
fetch(`${base}/api/playbooks/${this.scenario}.yaml`)
```

**tutorial-host.ts** (line 229):
```typescript
// Before:
const base = this.contentBase ? `${this.contentBase}/../../playbooks/` : 'playbooks/';
const resp = await fetch(`${base}${ref}`);
// After:
const resp = await fetch(`/api/playbooks/${ref}`);
```

Tutorial-host's path simplifies — no more relative path navigation. The REST endpoint is the canonical path.

### 4. Update examples scenario files

Four files in `examples/samples/Scenarios/` (`Composition.ts`, `Data Injection.ts`, `Step Modifiers.ts`, `Synchronization.ts`):

```typescript
// Before:
fetch('../../playbooks/manifest.json')
fetch('../../playbooks/' + entry.file)
// After:
fetch('/api/playbooks/manifest.json')
fetch('/api/playbooks/' + entry.file)
```

### 5. Remove playbooks copy from copy-samples.js

In `examples/scripts/copy-samples.js`, remove lines 47-52 that copy `../../playbooks/` to `dist/playbooks/`. The REST endpoint serves playbooks at runtime — no filesystem copy needed.

### 6. Tutorials Vite proxy

In `tutorials/vite.config.ts`, add a proxy rule:

```typescript
server: {
  port: 5174,
  open: true,
  proxy: {
    '/api/playbooks': 'http://localhost:8080',
  },
},
```

This routes playbook requests from the Vite dev server to the running Quarkus examples server.

### 7. Delete local playbooks/ directory

Remove `playbooks/` from git. It is now a derived artifact from the Maven dependency.

### 8. Update existing tests

Check `playbook-controller.test.ts`, `playbook-yaml-viewer.test.ts`, `tutorial-host.ts` tests for hardcoded `/playbooks/` paths and update to `/api/playbooks/`.

## Constraints

- The `fetch()` pattern requires the examples server running during dev. This is already the normal setup — scenario features need the server.
- Playbook YAML content must be byte-identical between what platform tests (via Walker parsing) and what pages renders. The single JAR artifact guarantees this.
- Pages' TS `yaml-core` package parses playbook front matter — this stays in pages (it's TS-specific parsing, not the YAML content itself).

## Out of scope

- Moving the TS playbook execution runtime from pages into platform (future epic per issue body)
- Creating a `PlaybookParsingTest` in platform (separate platform concern, tracked under platform#609)

## References

- `casehub-platform-playbook-catalog` — platform Maven module (`platform/playbook-catalog/pom.xml`)
- `examples/server/pom.xml` — Quarkus examples server dependencies
- `static-assets/pom.xml` — existing `target/` extraction pattern
- `playbook-controller.ts:629` — server-based fetch consumer
- `playbook-yaml-viewer.ts:244` — connection-based fetch consumer
- `tutorial-host.ts:229` — content-relative fetch consumer
- `examples/scripts/copy-samples.js:48` — filesystem copy to dist
- `examples/samples/Scenarios/Composition.ts:247` — example scenario fetch
- casehubio/platform#609 — platform playbook examples (canonical source)
- #545, #546, #547 — pages parity work (landed)
