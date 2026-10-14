## D1: Playbook sharing mechanism

**Choice:** Consume `casehub-platform-playbook-catalog` Maven dependency (already exists). Two consumption paths: REST endpoint for browser consumers, `target/` extraction for static/dev build.
**Alternatives:**
- Symlink to sibling repo — fragile, assumes co-located checkouts, not CI-friendly
- npm package — custom logic to find/extract from .m2, fragile version string parsing
- Copy to repo root and gitignore — fights Maven conventions, `target/` already exists for derived content
**Rationale:** Platform already publishes `casehub-platform-playbook-catalog` with playbooks on the classpath under `/playbooks/`. Version comes from parent BOM. Clean data artifact — consumers decide their own HTTP contract.
**Trade-offs:** Requires `mvn` step before pure-TS dev work that touches playbooks. Acceptable because playbooks change infrequently.
**Sources:** platform `casehub-platform-playbook-catalog` artifact, examples/server/pom.xml, playbook-controller.ts:629, playbook-yaml-viewer.ts:244, tutorial-host.ts:229, examples/scripts/copy-samples.js:48
**Exploration:** deep-analysis
**Status:** captured

## D2: HTTP serving of playbooks

**Choice:** Thin REST endpoint in pages examples-server at `@Path("/api/playbooks")` that reads from classpath and returns content. Pages controls URL shape and caching headers.
**Alternatives:**
- META-INF/resources auto-serving — blurs packaging; catalog should be pure data, not dictate HTTP contract
- Quarkus static-resources config pointing to target/ — runtime config coupling, fragile paths
**Rationale:** Catalog stays as clean classpath data. Pages owns the URL shape and can add caching headers. Non-Quarkus consumers (Spring, standalone) still use the clean classpath path directly. Browser fetch() calls update to `/api/playbooks/...` instead of `/playbooks/...`.
**Trade-offs:** 3 TS files need their fetch URLs updated from `/playbooks/...` to `/api/playbooks/...`. Small, well-scoped change.
**Depends on:** D1 (classpath consumption mechanism)
**Sources:** playbook-controller.ts:629, playbook-yaml-viewer.ts:244, tutorial-host.ts:229
**Exploration:** quick
**Status:** captured

## D3: Remove filesystem copy from examples dist build

**Choice:** Remove the playbooks copy from `copy-samples.js`. The 4 scenario `.ts` files in `examples/samples/Scenarios/` fetch from the REST endpoint like all other consumers. No more `../../playbooks/` filesystem references.
**Alternatives:**
- Extract from JAR to `target/playbooks/` and update copy-samples.js — keeps dist self-contained but adds Maven extraction step for a problem the REST endpoint already solves
**Rationale:** Single serving path for all browser consumers. Eliminates the filesystem copy entirely. The examples server already runs during dev and CI.
**Trade-offs:** Examples dist is no longer self-contained — requires the server running to access playbooks. Acceptable because the examples always run with the server.
**Depends on:** D2 (REST endpoint)
**Sources:** examples/scripts/copy-samples.js:48, examples/samples/Scenarios/Composition.ts:247
**Exploration:** quick
**Status:** captured

## D4: Tutorials Vite dev server playbook access

**Choice:** Add Vite proxy rule `/api/playbooks/*` → `localhost:8080` to `tutorials/vite.config.ts`. Tutorial-host fetches from the same REST endpoint as all other consumers.
**Alternatives:**
- Keep relative filesystem path for tutorials only — splits the serving strategy, tutorials remain a special case with local playbook files
**Rationale:** Single serving path for all consumers. Tutorials dev requires the examples server running alongside Vite, which is already the normal dev setup.
**Trade-offs:** Tutorials can't run fully standalone without the Quarkus server. Acceptable because the tutorials exercise scenario features that need the server anyway.
**Depends on:** D2 (REST endpoint)
**Sources:** tutorials/vite.config.ts, tutorial-host.ts:229
**Exploration:** quick
**Status:** captured
