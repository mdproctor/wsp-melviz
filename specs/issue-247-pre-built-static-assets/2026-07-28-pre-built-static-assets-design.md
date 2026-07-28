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

## Artifact structure

```
META-INF/resources/pages/
  tokens/
    casehub-dark.css
    casehub-light.css
    default-dark.css
    default-light.css
  components/
    components.js
```

## Consumer usage

**Bundled apps** (unchanged):
```ts
import '@casehubio/pages-ui-components';
import { applyTheme } from '@casehubio/pages-ui-tokens';
```

**Static HTML pages** (new):
```html
<link rel="stylesheet" href="/pages/tokens/casehub-dark.css">
<script type="module" src="/pages/components/components.js"></script>
```

**Maven dependency:**
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-ui-static</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

## Implementation

### 1. Token CSS build integration

The CLI `build` command already exists (`node dist/cli.js build` → `dist/themes/*.css`). Wire it into the normal build:

```json
"build": "tsc -p tsconfig.build.json && node dist/cli.js build"
```

Produces `dist/themes/{casehub-dark,casehub-light,default-dark,default-light}.css` — each containing `.pages-theme-*` class with all CSS custom properties plus `.pages-density-compact`.

### 2. Component ESM bundle

Add `esbuild` as a devDependency to `pages-ui-components`. Add `build-bundle.js` at the package root:

```js
import { build } from 'esbuild';
await build({
  entryPoints: ['dist/index.js'],
  bundle: true,
  format: 'esm',
  outfile: 'dist/bundle.js',
  minify: true,
});
```

Entry point is `dist/index.js` (tsc output) — esbuild resolves `lit` from node_modules and inlines it. Output: single self-contained ESM registering all 5 custom elements (`pages-button`, `pages-input`, `pages-select`, `pages-textarea`, `pages-checkbox`).

Build script:
```json
"build": "tsc -p tsconfig.build.json && node build-bundle.js"
```

Existing tsc-compiled individual modules remain unchanged — bundled app consumers keep tree-shakeable imports.

Expected bundle size: ~20KB gzipped (Lit core ~16KB + five simple components ~4KB).

### 3. Maven pom and assembly

New `static-assets/` directory at repo root:

```
static-assets/
  pom.xml
  assembly.sh
```

`assembly.sh` copies files into the Maven resource structure:
- `packages/pages-ui-tokens/dist/themes/*.css` → `target/static/META-INF/resources/pages/tokens/`
- `packages/pages-ui-components/dist/bundle.js` → `target/static/META-INF/resources/pages/components/`

Fails fast if expected files are missing (non-empty assertions).

`pom.xml`: parent `casehub-parent`, artifact `casehub-pages-ui-static`, packaging `jar`. Uses `exec-maven-plugin` to run `assembly.sh` during `generate-resources`. Maps `target/static` as resource directory. Compiler plugin skipped (no Java source).

### 4. CI integration

`maven-publish.yml` — add to `frontend-artifacts` job after yarn build:

```yaml
- name: Install static-assets artifact
  run: mvn -f static-assets/pom.xml --batch-mode install

- name: Publish static-assets (deploy step)
  run: mvn -f static-assets/pom.xml --batch-mode deploy ...
```

`pr-validation.yml` — add `mvn -f static-assets/pom.xml --batch-mode install` after yarn build.

Build order: `yarn build` (tokens CSS + component bundle) → Maven installs (independent, any order).

### 5. Testing

**Build-time:** `assembly.sh` asserts expected files exist and are non-empty before copying.

**Functional:** vitest test in `pages-ui-components` validates `dist/bundle.js` is a loadable ESM that registers the expected custom elements.

## Scope

- **Scale:** S — build-step additions and one new pom.xml
- **Complexity:** Low — no API changes, no new runtime code
- **Files touched:** ~8 (2 package.json edits, 1 new build-bundle.js, 1 new pom.xml, 1 new assembly.sh, 2 CI workflow edits, 1 test file)
