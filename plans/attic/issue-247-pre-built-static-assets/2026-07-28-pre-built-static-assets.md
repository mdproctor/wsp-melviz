# Pre-Built Static Assets Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #247 — feat: ship pre-built static assets for non-bundled consumers
**Issue group:** #247

**Goal:** Ship a new Maven artifact `casehub-pages-ui-static` containing pre-built theme CSS files and a bundled ESM component script, served via `META-INF/resources/pages/` in consumer Quarkus apps.

**Architecture:** Two existing packages (`pages-ui-tokens`, `pages-ui-components`) get new build scripts that produce static outputs (`dist/themes/*.css`, `dist/components.js`). A new `static-assets/` directory at repo root contains `assembly.sh` (orchestrates generation, validation, and file copying) and `pom.xml` (packages into a Maven JAR with `META-INF/resources/pages/` structure). CI workflows updated to build and publish the new artifact.

**Tech Stack:** esbuild (ESM bundling), Maven (artifact packaging), Node.js (build scripts, validation)

## Global Constraints

- All CSS custom properties use `--pages-` prefix (protocol: `css-design-tokens.md`)
- Theme CSS scoped under `.pages-theme-{name}` class selector
- Component bundle must be self-contained ESM (Lit inlined, no external imports)
- `build:tokens` and `build:bundle` must NOT merge into `build` — prevents leaking into `casehub-pages-npm` via `pack-all.sh`
- Maven artifact follows `casehub-parent` version `0.2-SNAPSHOT`
- URL namespace: `/pages/tokens/` for CSS, `/pages/ui/` for components (avoids `/pages/component/` collision with webapp)

---

### Task 1: Add esbuild and component bundle script

**Files:**
- Modify: `packages/pages-ui-components/package.json` (add esbuild devDep, add `build:bundle` script)
- Create: `packages/pages-ui-components/build-bundle.js`

**Interfaces:**
- Consumes: tsc output at `dist/index.js` (already produced by existing `build` script)
- Produces: `dist/components.js` (self-contained ESM), `dist/components.js.map` (linked source map)

- [ ] **Step 1: Add esbuild as devDependency**

```bash
yarn workspace @casehubio/pages-ui-components add -D esbuild
```

- [ ] **Step 2: Add `build:bundle` script to package.json**

Edit `packages/pages-ui-components/package.json` — add to `scripts`:
```json
"build:bundle": "node build-bundle.js"
```

Do NOT modify the existing `"build"` script — it must remain `"tsc -p tsconfig.build.json"` only.

- [ ] **Step 3: Create build-bundle.js**

Create `packages/pages-ui-components/build-bundle.js`:
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

- [ ] **Step 4: Verify bundle builds**

```bash
yarn workspace @casehubio/pages-ui-components run build
yarn workspace @casehubio/pages-ui-components run build:bundle
```

Expected: `packages/pages-ui-components/dist/components.js` and `dist/components.js.map` exist. The JS file should be ~60-80KB (uncompressed, minified, Lit inlined).

```bash
ls -la packages/pages-ui-components/dist/components.js packages/pages-ui-components/dist/components.js.map
wc -c packages/pages-ui-components/dist/components.js
```

- [ ] **Step 5: Verify existing tests still pass**

```bash
yarn workspace @casehubio/pages-ui-components run test
```

Expected: all existing component tests pass — bundle addition doesn't affect tsc output.

- [ ] **Step 6: Verify token CSS generation**

The `build:tokens` script already exists. Confirm it works after tsc build:

```bash
yarn workspace @casehubio/pages-ui-tokens run build
yarn workspace @casehubio/pages-ui-tokens run build:tokens
ls packages/pages-ui-tokens/dist/themes/
```

Expected: `casehub-dark.css`, `casehub-light.css`, `default-dark.css`, `default-light.css`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui-components/package.json packages/pages-ui-components/build-bundle.js yarn.lock
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#247): add esbuild component bundle script to pages-ui-components

Adds build:bundle script that produces dist/components.js — a self-contained
ESM with Lit inlined. Separate from build to prevent leaking into npm artifact.

Refs #247"
```

---

### Task 2: Create static-assets assembly and Maven pom

**Files:**
- Create: `static-assets/assembly.sh`
- Create: `static-assets/pom.xml`

**Interfaces:**
- Consumes: `packages/pages-ui-tokens/dist/themes/*.css` (from `build:tokens`), `packages/pages-ui-components/dist/components.js` and `.map` (from `build:bundle`)
- Produces: `target/static/META-INF/resources/pages/tokens/*.css`, `target/static/META-INF/resources/pages/ui/components.js{,.map}`

- [ ] **Step 1: Create assembly.sh**

Create `static-assets/assembly.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Assemble pre-built static assets for the casehub-pages-ui-static Maven artifact.
# Runs build:tokens and build:bundle, validates outputs, copies to META-INF structure.
#
# Usage: ./assembly.sh <output-dir>
#   output-dir: typically target/static (Maven resource directory)

OUTPUT_DIR="${1:?Usage: assembly.sh <output-dir>}"
REPO_ROOT="$(cd "$(dirname "$0")/.." && pwd)"

echo "=== Building static assets ==="

# 1. Generate token CSS files
echo "  Building token CSS..."
yarn --cwd "$REPO_ROOT" workspace @casehubio/pages-ui-tokens run build:tokens

# 2. Generate component ESM bundle
echo "  Building component bundle..."
yarn --cwd "$REPO_ROOT" workspace @casehubio/pages-ui-components run build:bundle

# 3. Validate ESM bundle is loadable
echo "  Validating component bundle..."
node --input-type=module <<'VALIDATE'
globalThis.HTMLElement = class {};
globalThis.customElements = { get: () => undefined, define: () => {} };
globalThis.document = { createElement: () => ({ setAttribute: () => {}, style: {} }), head: { prepend: () => {} } };
globalThis.CSSStyleSheet = class { replaceSync() {} };
globalThis.window = globalThis;
const path = process.argv[1] || process.env.REPO_ROOT + '/packages/pages-ui-components/dist/components.js';
await import(new URL('file://' + process.env.REPO_ROOT + '/packages/pages-ui-components/dist/components.js'));
console.log("  ✓ Bundle is valid ESM");
VALIDATE

# 4. Assert expected files exist and are non-empty
TOKENS_DIR="$REPO_ROOT/packages/pages-ui-tokens/dist/themes"
COMPONENTS_DIR="$REPO_ROOT/packages/pages-ui-components/dist"

for css in casehub-dark.css casehub-light.css default-dark.css default-light.css; do
  [ -s "$TOKENS_DIR/$css" ] || { echo "FAIL: $TOKENS_DIR/$css missing or empty"; exit 1; }
done
[ -s "$COMPONENTS_DIR/components.js" ] || { echo "FAIL: components.js missing or empty"; exit 1; }
[ -s "$COMPONENTS_DIR/components.js.map" ] || { echo "FAIL: components.js.map missing or empty"; exit 1; }
echo "  ✓ All expected files present"

# 5. Copy to META-INF structure
TOKENS_OUT="$OUTPUT_DIR/META-INF/resources/pages/tokens"
UI_OUT="$OUTPUT_DIR/META-INF/resources/pages/ui"
mkdir -p "$TOKENS_OUT" "$UI_OUT"

cp "$TOKENS_DIR"/*.css "$TOKENS_OUT/"
cp "$COMPONENTS_DIR/components.js" "$UI_OUT/"
cp "$COMPONENTS_DIR/components.js.map" "$UI_OUT/"

echo "=== Done — static assets assembled to $OUTPUT_DIR/META-INF/resources/pages/ ==="
```

```bash
chmod +x static-assets/assembly.sh
```

- [ ] **Step 2: Create pom.xml**

Create `static-assets/pom.xml`:
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
        <relativePath/>
    </parent>

    <artifactId>casehub-pages-ui-static</artifactId>
    <version>0.2-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>CaseHub Pages UI Static Assets</name>
    <description>Pre-built design system assets — theme CSS and component ESM bundle served from META-INF/resources/pages/</description>

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

    <build>
        <resources>
            <resource>
                <directory>${project.build.directory}/static</directory>
            </resource>
        </resources>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.5.0</version>
                <executions>
                    <execution>
                        <id>assemble-static-assets</id>
                        <phase>generate-resources</phase>
                        <goals>
                            <goal>exec</goal>
                        </goals>
                        <configuration>
                            <executable>bash</executable>
                            <arguments>
                                <argument>${project.basedir}/assembly.sh</argument>
                                <argument>${project.build.directory}/static</argument>
                            </arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <skipMain>true</skipMain>
                    <skip>true</skip>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Test the full assembly locally**

Requires `yarn build` to have run first (tsc output for both packages):

```bash
yarn install && yarn build:packages
mvn -f static-assets/pom.xml --batch-mode install
```

Expected: BUILD SUCCESS. Verify the JAR contains the right structure:

```bash
jar tf static-assets/target/casehub-pages-ui-static-0.2-SNAPSHOT.jar | grep pages/
```

Expected output:
```
META-INF/resources/pages/tokens/casehub-dark.css
META-INF/resources/pages/tokens/casehub-light.css
META-INF/resources/pages/tokens/default-dark.css
META-INF/resources/pages/tokens/default-light.css
META-INF/resources/pages/ui/components.js
META-INF/resources/pages/ui/components.js.map
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add static-assets/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#247): add static-assets Maven artifact with assembly script

New casehub-pages-ui-static artifact packages pre-built theme CSS and
component ESM bundle into META-INF/resources/pages/ for classpath serving.
assembly.sh orchestrates build:tokens, build:bundle, validation, and copy.

Refs #247"
```

---

### Task 3: Update CI workflows

**Files:**
- Modify: `.github/workflows/maven-publish.yml`
- Modify: `.github/workflows/pr-validation.yml`

**Interfaces:**
- Consumes: `static-assets/pom.xml` (Task 2)
- Produces: CI builds and publishes `casehub-pages-ui-static` alongside existing artifacts

- [ ] **Step 1: Update maven-publish.yml path triggers**

Add `'static-assets/**'` to both `on.push.paths` and `on.pull_request.paths` arrays in `.github/workflows/maven-publish.yml`:

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'backend/**'
      - 'packages/**'
      - 'components/**'
      - 'webapp/**'
      - 'npm-packages/**'
      - 'static-assets/**'
      - '.github/workflows/maven-publish.yml'
  pull_request:
    branches: [main]
    paths:
      - 'backend/**'
      - 'packages/**'
      - 'components/**'
      - 'webapp/**'
      - 'npm-packages/**'
      - 'static-assets/**'
      - '.github/workflows/maven-publish.yml'
```

- [ ] **Step 2: Add static-assets install step to maven-publish.yml**

Add after the "Install npm-packages artifact" step (line 110) and before "Publish frontend artifacts":

```yaml
      - name: Install static-assets artifact
        run: mvn -f static-assets/pom.xml --batch-mode install
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 3: Add static-assets to the deploy step in maven-publish.yml**

In the "Publish frontend artifacts to GitHub Packages" step, add a third line:

```yaml
      - name: Publish frontend artifacts to GitHub Packages
        if: github.event_name != 'pull_request'
        run: |
          mvn -f webapp/pom.xml --batch-mode deploy -DaltDeploymentRepository=github::https://maven.pkg.github.com/casehubio/casehub-pages -DaltSnapshotDeploymentRepository=github::https://maven.pkg.github.com/casehubio/casehub-pages
          mvn -f npm-packages/pom.xml --batch-mode deploy -DaltDeploymentRepository=github::https://maven.pkg.github.com/casehubio/casehub-pages -DaltSnapshotDeploymentRepository=github::https://maven.pkg.github.com/casehubio/casehub-pages
          mvn -f static-assets/pom.xml --batch-mode deploy -DaltDeploymentRepository=github::https://maven.pkg.github.com/casehubio/casehub-pages -DaltSnapshotDeploymentRepository=github::https://maven.pkg.github.com/casehubio/casehub-pages
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 4: Update pr-validation.yml path filter**

Add `'static-assets/**'` to the `js-changes` filter in `.github/workflows/pr-validation.yml`:

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

- [ ] **Step 5: Add static-assets build step to pr-validation.yml**

Add after the "Build JavaScript packages" step (js-build, line 99) and before "Test JavaScript packages":

```yaml
      - name: Build static-assets artifact
        id: static-assets-build
        if: steps.js-changes.outputs.js == 'true'
        continue-on-error: true
        run: mvn -f static-assets/pom.xml --batch-mode install
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Add `static-assets-build` to the failure check at the bottom:

```yaml
      - name: Fail workflow if tests failed
        if: |
          (steps.java-build.outcome == 'failure') ||
          (steps.java-test.outcome == 'failure') ||
          (steps.js-install.outcome == 'failure') ||
          (steps.js-build.outcome == 'failure') ||
          (steps.js-test.outcome == 'failure') ||
          (steps.static-assets-build.outcome == 'failure')
        run: exit 1
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add .github/workflows/maven-publish.yml .github/workflows/pr-validation.yml
git -C /Users/mdproctor/claude/casehub/pages commit -m "ci(#247): add static-assets to Maven publish and PR validation workflows

Adds casehub-pages-ui-static build/publish to maven-publish.yml and
pr-validation.yml. Triggered by changes to static-assets/, packages/,
or components/ directories.

Refs #247"
```
