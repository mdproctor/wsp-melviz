# Pre-Built Static Assets for Non-Bundled Consumers

**Issue:** casehubio/casehub-pages#247
**Date:** 2026-07-28
**Status:** Approved

## Problem

`pages-ui-tokens` generates theme CSS at runtime via JavaScript. `pages-ui-components` requires an ESM bundler to import and register custom elements. Consumer repos with static HTML pages (claudony login/register, chat-app, future apps) cannot use `<pages-button>` or get `--pages-*` CSS variables without inventing per-repo workarounds.

## Decision

Ship a new Maven artifact `casehub-pages-ui-static` (groupId `io.casehub`, version `0.2-SNAPSHOT`) containing pre-built static files served via `META-INF/resources/pages/`. Consumer Quarkus apps add one Maven dependency; static HTML pages use `<link>` and `<script>` tags at predictable URLs.

### Why a separate artifact

Three Maven artifacts map to three distinct tiers:

| Artifact | Tier | Consumption model |
|----------|------|-------------------|
| `casehub-pages-npm` | Build-time source packages | `maven-dependency-plugin:unpack` → `portal:` |
| `casehub-pages-ui-static` | Runtime design system assets | Classpath → `META-INF/resources/` |
| `casehub-pages-webapp` | Runtime dashboard application | Classpath → `META-INF/resources/` |

Mixing build-time and runtime in `casehub-pages-npm` creates a category error — it is unpacked to a source directory, not classpath-served. Bundling into `casehub-pages-webapp` inverts the dependency arrow — the design system is lower in the stack than the dashboard app.

### Why not the issue's URLs

Issue #247 proposed `/pages-ui-tokens/themes/casehub-dark.css` and `/pages-ui-components/bundle.js` — artifact-name-based paths. This spec uses a unified `/pages/` namespace instead (`/pages/tokens/`, `/pages/ui/`). The shorter, grouped paths are easier for consumers to remember and keep all design system assets under one prefix. The original issue also proposed shipping the files inside the existing `casehub-pages-npm` artifact's `dist/` directories, but that artifact is unpacked to a source directory, not classpath-served — hence the separate artifact.

## Artifact structure

```
META-INF/resources/pages/
  tokens/
    casehub-dark.css
    casehub-light.css
    default-dark.css
    default-light.css
  ui/
    components.js
    components.js.map
```

The `/pages/ui/` path is deliberately distinct from the existing `/pages/component/` path (singular) in `casehub-pages-webapp`, which serves the `llm-prompter` and `svg-heatmap` dashboard components.

## Consumer usage

**Bundled apps** (unchanged):
```ts
import '@casehubio/pages-ui-components';
import { applyTheme } from '@casehubio/pages-ui-tokens';
```

**Static HTML pages** (new):
```html
<html class="pages-theme-casehub-dark">
<head>
  <link rel="stylesheet" href="/pages/tokens/casehub-dark.css">
</head>
<body>
  <script type="module" src="/pages/ui/components.js"></script>
  <pages-button label="Submit" variant="primary"></pages-button>
</body>
</html>
```

The `class="pages-theme-casehub-dark"` on an ancestor element is required — theme CSS files scope all custom properties under a `.pages-theme-{name}` selector. Without the class, no CSS variables take effect and components fall back to hardcoded defaults.

**Maven dependency:**
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-ui-static</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

**Cache headers:** Quarkus serves `META-INF/resources/` files with default cache headers. For SNAPSHOT-based development (all current consumers), Quarkus dev mode handles cache invalidation. Cache-busting for released versions (e.g., versioned query parameters) is deferred — see casehubio/casehub-pages#249.

## Implementation

### 1. Token CSS build integration

The CLI `build` command already exists (`node dist/cli.js build` → `dist/themes/*.css`). The existing `build:tokens` script remains separate from `build` — it is not merged into the default build to avoid generating theme CSS files during `yarn build`, which would cause them to be packed into the `casehub-pages-npm` artifact by `pack-all.sh`.

```json
"build": "tsc -p tsconfig.build.json",
"build:tokens": "node dist/cli.js build"
```

The `assembly.sh` script (§3) calls `build:tokens` explicitly after `yarn build` has completed tsc compilation.

Produces `dist/themes/{casehub-dark,casehub-light,default-dark,default-light}.css` — each containing `.pages-theme-*` class with all CSS custom properties plus `.pages-density-compact`.

### 2. Component ESM bundle

Add `esbuild` as a devDependency to `pages-ui-components`. Add `build-bundle.js` at the package root:

```js
import { build } from 'esbuild';
await build({
  entryPoints: ['dist/index.js'],
  bundle: true,
  format: 'esm',
  outfile: 'dist/components.js',
  minify: true,
  sourcemap: 'linked',
});
```

Entry point is `dist/index.js` (tsc output) — esbuild resolves `lit` from node_modules and inlines it. Output: single self-contained ESM registering all 5 custom elements (`pages-button`, `pages-input`, `pages-select`, `pages-textarea`, `pages-checkbox`), plus a linked source map at `dist/components.js.map`.

Build scripts:
```json
"build": "tsc -p tsconfig.build.json",
"build:bundle": "node build-bundle.js"
```

The bundle step is separate from `build` for the same reason as `build:tokens` (§1) — `pack-all.sh` packs all of `dist/` for every non-private package into `casehub-pages-npm`, so merging the bundle into `build` would put `dist/components.js` into the npm artifact. The npm artifact only needs tree-shakeable modules; the pre-built bundle is exclusively for the `casehub-pages-ui-static` Maven artifact. The `assembly.sh` script (§3) calls `build:bundle` explicitly.

Existing tsc-compiled individual modules remain unchanged — bundled app consumers keep tree-shakeable imports.

Expected bundle size: ~20KB gzipped (Lit core ~16KB + five simple components ~4KB).

### 3. Maven pom and assembly

New `static-assets/` directory at repo root:

```
static-assets/
  pom.xml
  assembly.sh
```

`assembly.sh` generates the static build outputs, validates them, then copies into the Maven resource structure:
1. Runs `yarn workspace @casehubio/pages-ui-tokens run build:tokens` to generate theme CSS
2. Runs `yarn workspace @casehubio/pages-ui-components run build:bundle` to generate the component bundle
3. Validates the bundle is loadable ESM — stubs browser globals that don't exist in Node.js (`customElements` at minimum, since every component has module-level `customElements.get()`/`customElements.define()` calls), then dynamically imports the bundle. This confirms esbuild produced valid, self-contained ESM.
4. Asserts all expected files exist and are non-empty (theme CSS files, `components.js`, `components.js.map`)
5. Copies `packages/pages-ui-tokens/dist/themes/*.css` → `target/static/META-INF/resources/pages/tokens/`
6. Copies `packages/pages-ui-components/dist/components.js` → `target/static/META-INF/resources/pages/ui/`
7. Copies `packages/pages-ui-components/dist/components.js.map` → `target/static/META-INF/resources/pages/ui/`

Fails fast at any step — build, validation, or copy.

`pom.xml`: parent `casehub-parent`, artifact `casehub-pages-ui-static`, packaging `jar`. Uses `exec-maven-plugin` to run `assembly.sh` during `generate-resources`. Maps `target/static` as resource directory. Compiler plugin skipped (no Java source). Includes `<repositories>` section for GitHub Packages resolution (same as `webapp/pom.xml`). No `<distributionManagement>` — CI uses `-DaltDeploymentRepository` flags.

```xml
<repositories>
    <repository>
        <id>github</id>
        <name>casehubio GitHub Packages</name>
        <url>https://maven.pkg.github.com/casehubio/*</url>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>
```

### 4. CI integration

**`maven-publish.yml`** — add `static-assets/**` to the `on.push.paths` and `on.pull_request.paths` trigger arrays. Add to `frontend-artifacts` job after yarn build:

```yaml
- name: Install static-assets artifact
  run: mvn -f static-assets/pom.xml --batch-mode install

- name: Publish static-assets (deploy step)
  run: mvn -f static-assets/pom.xml --batch-mode deploy ...
```

**`pr-validation.yml`** — add `static-assets/**` to the `js-changes` path filter (since the static-assets Maven build depends on JS build output):

```yaml
- name: Check for JavaScript changes
  id: js-changes
  uses: dorny/paths-filter@v3
  with:
    filters: |
      js:
        - 'packages/**'
        - 'components/**'
        - 'webapp/**'
        - 'examples/**'
        - 'static-assets/**'
        - 'package.json'
        - 'yarn.lock'
```

Add `mvn -f static-assets/pom.xml --batch-mode install` as a new step after the `js-build` step, conditioned on `steps.js-changes.outputs.js == 'true'`. This step depends on `yarn build:prod` having completed (tsc output must exist for both `build:tokens` and the esbuild bundle).

Build order: `yarn build` (tsc for all packages) → Maven installs (static-assets assembly calls `build:tokens` and `build:bundle`, then copies all files).

### 5. Testing

All static-asset validation is co-located in `assembly.sh`, which is the single orchestrator for generating, validating, and packaging the artifacts:

- **File assertions:** expected files exist and are non-empty (theme CSS, `components.js`, `components.js.map`)
- **ESM load test:** dynamic import of the component bundle verifies esbuild produced valid ESM output

No separate vitest test file is needed — `dist/components.js` only exists after `build:bundle`, which runs exclusively in assembly.sh. The existing vitest suites in `pages-ui-components` cover individual component correctness via tree-shakeable imports.

## Scope

- **Scale:** S — build-step additions and one new pom.xml
- **Complexity:** Low — no API changes, no new runtime code
- **Files touched:** ~6 (1 package.json edit, 1 new build-bundle.js, 1 new pom.xml, 1 new assembly.sh, 2 CI workflow edits)
