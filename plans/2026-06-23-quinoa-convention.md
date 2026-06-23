# Quinoa Convention Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish casehub-pages packages to GitHub Packages and establish the Quinoa convention for Quarkus host app integration.

**Architecture:** Rename npm scope `@casehub` → `@casehubio` (GitHub Packages requirement), fix package.json publishing fields, add side-effect import to pages-runtime, bump versions to 0.2.0, create CI publish workflow, and provide a reference template with convention doc.

**Tech Stack:** TypeScript, Yarn 4, esbuild, GitHub Actions, GitHub Packages npm registry

## Global Constraints

- All commits reference `Refs #26` (or `Closes #26` on the final commit)
- Prerequisite #28 (iframe ECharts removal) executes first on this branch
- Packages use `@casehubio` scope (GitHub org = `casehubio`)
- Version 0.2.0 for all publishable packages
- `workspace:*` cross-references stay — Yarn replaces them at publish time
- `private: true` packages are never published or version-bumped
- IntelliJ "Replace in Files" for scope rename — not rename refactoring (string literals, not symbols)

---

### Task 1: Remove redundant iframe ECharts component (#28)

**Files:**
- Delete: `components/pages-component-echarts/` (entire directory)
- Delete: `packages/pages-echarts-base/` (entire directory)
- Modify: `webapp/webpack.config.js` — remove `"echarts"` from components array
- Modify: `tsconfig.json` (root) — remove references to deleted packages
- Modify: `package.json` (root) — `build:packages` script: remove `pages-echarts-base` build step

**Interfaces:**
- Consumes: nothing
- Produces: clean build with no echarts iframe component; two fewer packages in monorepo

- [ ] **Step 1: Delete the iframe ECharts component directory**

```bash
rm -rf components/pages-component-echarts/
```

- [ ] **Step 2: Delete the ECharts base package directory**

```bash
rm -rf packages/pages-echarts-base/
```

- [ ] **Step 3: Update webapp/webpack.config.js — remove echarts from copy list**

Change the components array from:
```js
const components = ["echarts", "llm-prompter", "svg-heatmap"];
```
to:
```js
const components = ["llm-prompter", "svg-heatmap"];
```

- [ ] **Step 4: Update root tsconfig.json — remove deleted project references**

Remove these two lines from the `references` array:
```json
{ "path": "packages/pages-echarts-base" },
{ "path": "components/pages-component-echarts" },
```

- [ ] **Step 5: Update root package.json — remove pages-echarts-base from build:packages**

In the `build:packages` script, remove:
```
yarn workspace @casehub/pages-echarts-base run build &&
```

(Note: this still uses `@casehub` scope — Task 2 will rename it.)

- [ ] **Step 6: Verify full build passes**

Run: `yarn install && yarn build`
Expected: clean build, no errors referencing echarts-base or component-echarts

Run: `yarn typecheck`
Expected: no type errors

Run: `yarn test`
Expected: all tests pass (echarts component tests were deleted with the directory)

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor: remove redundant iframe ECharts component  Refs #28

pages-viz inline Web Components are the single ECharts integration
point. Deletes components/pages-component-echarts/ and
packages/pages-echarts-base/."
```

---

### Task 2: Scope rename `@casehub` → `@casehubio`

**Files:**
- Modify: every `package.json` under `packages/` and `components/` — `name` field
- Modify: every `.ts` file with `@casehub/` imports — import paths
- Modify: every `package.json` with `workspace:*` cross-references — dependency names
- Modify: every `tsconfig.json` that extends `@casehub/pages-tsconfig` — extends path
- Modify: `eslint.config.mjs` — if it references `@casehub/` packages
- Modify: `scripts/bump-version.mjs` — no changes needed (reads from directory structure, not scope)

**Interfaces:**
- Consumes: Task 1 (no echarts packages to rename)
- Produces: all packages under `@casehubio` scope; build and tests pass with new scope

- [ ] **Step 1: Replace `@casehub/` with `@casehubio/` across all files**

Using IntelliJ "Replace in Files" (Ctrl+Shift+R):
- Search: `@casehub/`
- Replace: `@casehubio/`
- Scope: entire project (excluding `node_modules/`, `dist/`, `.yarn/`, `_legacy/`)
- File mask: `*.json,*.ts,*.tsx,*.js,*.mjs,*.yml,*.yaml,*.md,*.html`

Review the preview before applying — verify no false positives (e.g., in documentation that refers to the scope change itself).

- [ ] **Step 2: Verify no remaining `@casehub/` references in source files**

Run: `grep -r "@casehub/" --include="*.ts" --include="*.json" --include="*.mjs" --include="*.yml" packages/ components/ webapp/ examples/ .github/ scripts/ | grep -v node_modules | grep -v dist | grep -v "@casehubio/"`
Expected: no output (all references updated)

- [ ] **Step 3: Clean and rebuild**

```bash
rm -rf node_modules
yarn install
yarn build
```

Expected: clean install and build with new scope names. Yarn resolves `workspace:*` references using the new `@casehubio/` package names.

- [ ] **Step 4: Verify type checking passes**

Run: `yarn typecheck`
Expected: no type errors

- [ ] **Step 5: Run all tests**

Run: `yarn test`
Expected: all tests pass

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "refactor: rename @casehub → @casehubio scope  Refs #26

GitHub Packages requires npm scope to match the org name.
Mechanical replacement across all package.json, imports,
workspace references, and tsconfig extends."
```

---

### Task 3: Package.json publishing fields and private flags

**Files:**
- Modify: `packages/pages-data/package.json` — add repository, publishConfig
- Modify: `packages/pages-ui/package.json` — add repository, publishConfig
- Modify: `packages/pages-viz/package.json` — add repository, publishConfig, sideEffects
- Modify: `packages/pages-component/package.json` — add repository, publishConfig
- Modify: `packages/pages-runtime/package.json` — add repository, publishConfig
- Modify: `packages/pages-iframe-api/package.json` — add publishConfig (already has repository)
- Modify: `packages/pages-iframe-dev/package.json` — add `"private": true`
- Modify: `components/pages-component-llm-prompter/package.json` — remove `"private": true`, add publishConfig, add `"files": ["dist"]`
- Modify: `components/pages-component-svg-heatmap/package.json` — remove `"private": true`, add publishConfig, add `"files": ["dist"]`

**Interfaces:**
- Consumes: Task 2 (scope already renamed)
- Produces: all publishable packages have correct fields for GitHub Packages; `npm publish --dry-run` succeeds

- [ ] **Step 1: Add repository and publishConfig to core packages**

For each of `pages-data`, `pages-ui`, `pages-component`, `pages-runtime` — add these fields after the existing `"description"` field:

```json
"repository": {
  "type": "git",
  "url": "https://github.com/casehubio/casehub-pages.git"
},
"publishConfig": {
  "registry": "https://npm.pkg.github.com"
},
```

- [ ] **Step 2: Add repository, publishConfig, and sideEffects to pages-viz**

Add the same `repository` and `publishConfig` as Step 1, plus:

```json
"sideEffects": true,
```

Place `sideEffects` after the `"types"` field. This prevents bundlers from tree-shaking the bare import that registers Web Component custom elements.

- [ ] **Step 3: Add publishConfig to pages-iframe-api**

`pages-iframe-api` already has `repository`. Add only:

```json
"publishConfig": {
  "registry": "https://npm.pkg.github.com"
},
```

- [ ] **Step 4: Add private: true to pages-iframe-dev**

Add `"private": true` as the first field in `packages/pages-iframe-dev/package.json`:

```json
{
  "private": true,
  "name": "@casehubio/pages-iframe-dev",
  ...
}
```

- [ ] **Step 5: Fix iframe component packages for publishing**

For both `components/pages-component-llm-prompter/package.json` and `components/pages-component-svg-heatmap/package.json`:

1. Remove `"private": true`
2. Add `"publishConfig": { "registry": "https://npm.pkg.github.com" }`
3. Add `"files": ["dist"]` (root `.gitignore` excludes `dist/` — without this, npm publish won't include the built bundles)

- [ ] **Step 6: Verify publishable state with dry-run**

Run `scripts/bump-version.mjs --check` to confirm the correct packages are publishable vs private.

Expected output:
```
Publishable packages:
  0.0.1    @casehubio/pages-component
  0.0.1    @casehubio/pages-data
  0.0.0    @casehubio/pages-iframe-api
  0.0.1    @casehubio/pages-runtime
  0.0.1    @casehubio/pages-ui
  0.0.1    @casehubio/pages-viz
  0.0.0    @casehubio/pages-component-llm-prompter
  0.0.0    @casehubio/pages-component-svg-heatmap

Skipped (private: true):
  0.0.0    @casehubio/pages-tsconfig
  0.0.0    @casehubio/pages-webpack-base
  0.0.0    @casehubio/pages-iframe-dev
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "build: add publishing fields to package.json files  Refs #26

Adds repository, publishConfig, sideEffects (pages-viz), and
files (iframe components) for GitHub Packages publishing.
Marks pages-iframe-dev as private. Unprivates iframe component
packages for publishing."
```

---

### Task 4: pages-runtime side-effect import

**Files:**
- Modify: `packages/pages-runtime/src/index.ts` — add bare import
- Modify: `examples/src/casehub-entry.ts` — remove now-redundant bare import

**Interfaces:**
- Consumes: Task 3 (pages-viz has sideEffects: true)
- Produces: `loadSite()` works without consumers needing a bare `import "@casehubio/pages-viz"`

- [ ] **Step 1: Add side-effect import to pages-runtime entry point**

Add this line as the first line of `packages/pages-runtime/src/index.ts`:

```ts
import "@casehubio/pages-viz";
```

The file should now begin:
```ts
import "@casehubio/pages-viz";
export { loadSite } from "./site.js";
export type { LiveSite, SiteOptions } from "./site.js";
// ... rest of exports
```

- [ ] **Step 2: Remove redundant bare import from examples gallery**

In `examples/src/casehub-entry.ts`, remove the first line:
```ts
import "@casehubio/pages-viz";
```

The file should now be:
```ts
import { loadSite } from "@casehubio/pages-runtime";
import type { LiveSite, SiteOptions } from "@casehubio/pages-runtime";

export { loadSite };
export type { LiveSite, SiteOptions };
```

- [ ] **Step 3: Rebuild and verify**

Run: `yarn build`
Expected: clean build

Run: `yarn test`
Expected: all tests pass

- [ ] **Step 4: Verify side effects survive in examples build**

Run: `yarn build:prod`

Then verify the examples gallery output contains custom element registrations:

```bash
grep -c "customElements" examples/dist/*.js
```

Expected: non-zero count — confirms `customElements.define` calls survived bundling through the pages-runtime → pages-viz import chain.

- [ ] **Step 5: Commit**

```bash
git add packages/pages-runtime/src/index.ts examples/src/casehub-entry.ts
git commit -m "refactor: pages-runtime owns pages-viz side-effect import  Refs #26

Consumers no longer need a bare import of pages-viz — custom element
registration is an internal concern of pages-runtime."
```

---

### Task 5: Version bump to 0.2.0

**Files:**
- Modify: all publishable `package.json` files — `version` field

**Interfaces:**
- Consumes: Task 3 (private flags correct so bump-version skips the right packages)
- Produces: all publishable packages at version 0.2.0

- [ ] **Step 1: Run the version bump script**

```bash
node scripts/bump-version.mjs 0.2.0
```

Expected output:
```
  0.0.1 → 0.2.0  @casehubio/pages-component
  0.0.1 → 0.2.0  @casehubio/pages-data
  0.0.0 → 0.2.0  @casehubio/pages-iframe-api
  0.0.1 → 0.2.0  @casehubio/pages-runtime
  0.0.1 → 0.2.0  @casehubio/pages-ui
  0.0.1 → 0.2.0  @casehubio/pages-viz
  0.0.0 → 0.2.0  @casehubio/pages-component-llm-prompter
  0.0.0 → 0.2.0  @casehubio/pages-component-svg-heatmap
  skip  @casehubio/pages-tsconfig (private)
  skip  @casehubio/pages-webpack-base (private)
  skip  @casehubio/pages-iframe-dev (private)

8 updated, 3 skipped
```

- [ ] **Step 2: Verify with --check**

```bash
node scripts/bump-version.mjs --check
```

Expected: all publishable packages show `0.2.0`

- [ ] **Step 3: Rebuild to confirm version change doesn't break anything**

Run: `yarn build && yarn typecheck && yarn test`
Expected: all pass

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "build: bump publishable packages to 0.2.0  Refs #26

Aligns with casehub 0.2 release line."
```

---

### Task 6: GitHub Actions publish workflow

**Files:**
- Create: `.github/workflows/publish-packages.yml`

**Interfaces:**
- Consumes: Task 5 (packages at 0.2.0 with correct fields)
- Produces: CI workflow that publishes packages to GitHub Packages on push to main

- [ ] **Step 1: Create the publish workflow**

Create `.github/workflows/publish-packages.yml`:

```yaml
name: Publish Packages

on:
  push:
    branches: [main]
    paths:
      - 'packages/*/package.json'
      - 'components/*/package.json'
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          registry-url: 'https://npm.pkg.github.com'
          scope: '@casehubio'
          cache: 'yarn'

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: yarn install --immutable

      - name: Build all packages
        run: yarn build

      - name: Publish packages
        run: |
          for dir in packages/*/  components/*/; do
            if [ ! -f "$dir/package.json" ]; then continue; fi

            PRIVATE=$(node -p "try{require('./$dir/package.json').private||false}catch{false}")
            if [ "$PRIVATE" = "true" ]; then
              echo "⏭ Skip (private): $dir"
              continue
            fi

            NAME=$(node -p "require('./$dir/package.json').name")
            VERSION=$(node -p "require('./$dir/package.json').version")

            # Check if version already exists
            if npm view "$NAME@$VERSION" version 2>/dev/null; then
              echo "⏭ Skip (exists): $NAME@$VERSION"
            else
              echo "📦 Publishing: $NAME@$VERSION"
              cd "$dir"
              npm publish
              cd -
            fi
          done
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: Verify workflow syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/publish-packages.yml'))" && echo "Valid YAML"
```

Expected: `Valid YAML`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/publish-packages.yml
git commit -m "ci: add GitHub Packages publish workflow  Refs #26

Publishes all non-private @casehubio/pages-* packages to GitHub
Packages on push to main. Skips already-published versions."
```

---

### Task 7: Reference template

**Files:**
- Create: `templates/quinoa-host/package.json`
- Create: `templates/quinoa-host/tsconfig.json`
- Create: `templates/quinoa-host/esbuild.config.mjs`
- Create: `templates/quinoa-host/.npmrc`
- Create: `templates/quinoa-host/src/index.ts`
- Create: `templates/quinoa-host/.gitignore`

**Interfaces:**
- Consumes: Task 5 (version number for dependency declarations)
- Produces: copy-pasteable template for `src/main/webui/` in a Quarkus host app

- [ ] **Step 1: Create template package.json**

Create `templates/quinoa-host/package.json`:

```json
{
  "name": "webui",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "0.2.0",
    "@casehubio/pages-ui": "0.2.0"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

- [ ] **Step 2: Create template tsconfig.json**

Create `templates/quinoa-host/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Create template esbuild.config.mjs**

Create `templates/quinoa-host/esbuild.config.mjs`:

```js
import { build, context } from "esbuild";

const isWatch = process.argv.includes("--watch");

const options = {
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/app.js",
  format: "esm",
  target: "es2020",
  minify: !isWatch,
  sourcemap: isWatch,
};

if (isWatch) {
  const ctx = await context(options);
  await ctx.watch();
  console.log("Watching for changes...");
} else {
  await build(options);
}
```

- [ ] **Step 4: Create template .npmrc**

Create `templates/quinoa-host/.npmrc`:

```
@casehubio:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

- [ ] **Step 5: Create template entry point**

Create `templates/quinoa-host/src/index.ts`:

```ts
import { loadSite } from "@casehubio/pages-runtime";
import { page, table, dataset, lookup, groupBy, col, count } from "@casehubio/pages-ui";

const dashboard = page("Example Dashboard",
  table({
    lookup: lookup("sample", groupBy("category", col("category"), count("total")))
  }),
  {
    datasets: [
      dataset("sample", "/api/data/sample")
    ]
  }
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, dashboard).catch(console.error);
}
```

- [ ] **Step 6: Create template .gitignore**

Create `templates/quinoa-host/.gitignore`:

```
node_modules/
dist/
```

- [ ] **Step 7: Commit**

```bash
git add templates/
git commit -m "feat: add Quinoa host reference template  Refs #26

Copy-pasteable template for src/main/webui/ in Quarkus host apps.
Two dependencies (pages-runtime + pages-ui), esbuild bundler,
TypeScript strict mode."
```

---

### Task 8: Convention doc and ARC42STORIES update

**Files:**
- Create: `docs/quinoa-convention.md`
- Modify: `ARC42STORIES.MD` — update §7 and §9

**Interfaces:**
- Consumes: Task 7 (template exists to reference)
- Produces: convention documentation and updated architecture doc

- [ ] **Step 1: Create convention doc**

Create `docs/quinoa-convention.md`:

```markdown
# Quinoa Convention: Quarkus + TypeScript Frontend

## Why Quinoa

- Single `mvn quarkus:dev` hot-reloads both Java and TypeScript
- Single `mvn package` produces one JAR with frontend included
- No Node.js at runtime — Quinoa runs `npm build` at compile time
- TypeScript type checking catches component composition errors at build time

## Why esbuild (not webpack)

The casehub-pages monorepo uses webpack internally for its multi-package build with loaders, polyfills, and dev-server integration. Host apps use esbuild because they have a single entry point and no internal package dependencies to resolve — esbuild handles this in a few lines of config with sub-second builds.

## Prerequisites

1. GitHub token with `read:packages` scope for GitHub Packages access
2. Quarkus Quinoa extension in host app `pom.xml`

## Setup

### 1. Add Quinoa extension

```xml
<dependency>
  <groupId>io.quarkiverse.quinoa</groupId>
  <artifactId>quarkus-quinoa</artifactId>
</dependency>
```

### 2. Copy the reference template

Copy `templates/quinoa-host/` contents to `src/main/webui/` in your host app.

### 3. Configure Quinoa

Add to `application.properties`:

```properties
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true
```

### 4. Install and build

```bash
cd src/main/webui
npm install
npm run build
```

Or via Maven: `mvn quarkus:dev` (Quinoa runs the build automatically).

## Dependencies

**Required (every host app):**
- `@casehubio/pages-runtime` — `loadSite()` entry point; transitively provides pages-viz (chart/table Web Components), pages-component (layout), pages-data (datasets)
- `@casehubio/pages-ui` — TypeScript DSL for composing pages (page, table, barChart, dataset, lookup, etc.)

**Optional (iframe-isolated components):**
- `@casehubio/pages-component-llm-prompter` — LLM prompt engineering UI
- `@casehubio/pages-component-svg-heatmap` — SVG-based heatmaps

## Adding Iframe Components

1. Add the component package to `package.json` dependencies
2. Create `scripts/copy-components.mjs` (copies built bundles from node_modules to dist):

```js
import { cpSync, readdirSync, existsSync } from "fs";
import { join, dirname } from "path";
import { fileURLToPath } from "url";
import { createRequire } from "module";

const __dirname = dirname(fileURLToPath(import.meta.url));
const root = join(__dirname, "..");
const require = createRequire(import.meta.url);

const pkg = JSON.parse(
  (await import("fs")).readFileSync(join(root, "package.json"), "utf-8")
);

const deps = Object.keys(pkg.dependencies || {});
for (const dep of deps) {
  const match = dep.match(/^@casehubio\/pages-component-(.+)$/);
  if (!match) continue;

  const name = match[1];
  const depDir = dirname(require.resolve(dep + "/package.json"));
  const src = join(depDir, "dist");
  const dest = join(root, "dist", "pages", "component", name);

  if (existsSync(src)) {
    cpSync(src, dest, { recursive: true });
    console.log(`Copied ${dep} → dist/pages/component/${name}/`);
  }
}
```

3. Update the build script: `"build": "node esbuild.config.mjs && node scripts/copy-components.mjs"`

## Migration Path (Existing Hosts)

For hosts with existing JS/HTML in `META-INF/resources/`:

1. Move static files to `src/main/webui/src/`
2. Add `package.json` with casehub-pages dependencies
3. Convert JavaScript to TypeScript incrementally
4. Add Quinoa extension to `pom.xml`
5. Remove manual static file serving (e.g., UiResource.java)

## Applies To

- casehub-claudony
- casehub-devtown
- casehub-drafthouse (Electron + embedded Quarkus native)
- Any future Quarkus app composing UI with casehub-pages
```

- [ ] **Step 2: Update ARC42STORIES.MD §7 (Deployment View)**

Replace the current §7 content with:

```markdown
## §7 Deployment View

Packages published to GitHub Packages (`npm.pkg.github.com`) under the `@casehubio` scope. Host applications consume packages via Quarkus Quinoa — `npm install` at build time, static serving from `META-INF/resources/` at runtime. No Node.js at runtime.

Convention: `docs/quinoa-convention.md`. Reference template: `templates/quinoa-host/`.

The examples gallery serves at `localhost:8080` via webpack-dev-server (monorepo-internal, not published).
```

- [ ] **Step 3: Update ARC42STORIES.MD §9 backlog — mark #26 as shipped**

Change the #26 row in the backlog table from:
```
| #26 | Platform convention: Quarkus + TypeScript frontend via Quinoa | 🔲 open |
```
to:
```
| #26 | Platform convention: Quarkus + TypeScript frontend via Quinoa | ✅ shipped |
```

Add #28 as shipped too:
```
| #28 | Remove redundant iframe ECharts component — migrate to inline Web Components | ✅ shipped |
```

- [ ] **Step 4: Commit**

```bash
git add docs/quinoa-convention.md ARC42STORIES.MD
git commit -m "docs: add Quinoa convention doc and update ARC42STORIES  Closes #26

Convention doc covers setup, dependencies, iframe opt-in, migration
path for existing hosts. ARC42STORIES §7 updated with deployment
via GitHub Packages + Quinoa."
```

---

## Post-Implementation Verification

After all tasks complete:

1. `yarn build && yarn typecheck && yarn test` — full build green
2. `node scripts/bump-version.mjs --check` — 8 packages at 0.2.0, 3 private
3. `grep -r "@casehub/" packages/ components/ webapp/ examples/ | grep -v node_modules | grep -v dist | grep -v "@casehubio/"` — no stale scope references
4. Template esbuild builds: `cd templates/quinoa-host && npm install && npm run build` (requires published packages — test after first CI publish)
5. CI publish workflow triggers on push to main and publishes to GitHub Packages
