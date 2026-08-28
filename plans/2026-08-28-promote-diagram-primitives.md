# Promote Diagram Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #389 — Promote DiagramBaseMixin to pages-diagram-core package
**Issue group:** #389, blocks-ui#TBD (re-export compat issue, created in Batch 3)

**Goal:** Migrate six categories of generic primitives from blocks-ui to casehub-pages so any pages app can reach for them without a blocks-ui dependency.

**Architecture:** Create a new `pages-diagram-core` package for diagram editing infrastructure (DiagramBaseMixin, toolbar, export, schema-registry, editors, GitHubBackend, YAML primitives). Add status-badge/registry SPI, split-workbench, and event-trail to the existing `pages-ui-components` package via sub-path exports. blocks-ui then re-exports from pages for zero downstream impact.

**Tech Stack:** TypeScript 5, Lit 3, yaml (CST), html-to-image, vitest

## Global Constraints

- All web components use `pages-` prefix (`pages-diagram-toolbar`, `pages-status-badge`, etc.)
- Guarded manual registration (no `@customElement` decorator) — enables blocks-ui compat re-registration under old names
- Sub-path exports for side-effect isolation per web-component-strategy protocol
- `yaml` package for CST-preserving edits (already in monorepo via pages-ui)
- No domain-specific imports — the filter is "would a non-CaseHub pages app use this?"
- Element registration pattern:
  ```ts
  export class PagesStatusBadge extends LitElement { ... }
  if (!customElements.get('pages-status-badge')) {
    customElements.define('pages-status-badge', PagesStatusBadge);
  }
  ```

---

## Batch 1: pages-diagram-core package

After this batch: a new `@casehubio/pages-diagram-core` package exists in pages with all diagram editing infrastructure, YAML primitives, and GitHubBackend. Builds and tests pass.

### Task 1: Package scaffold + YAML primitives + schema-registry + editors

**Files:**
- Create: `packages/pages-diagram-core/package.json`
- Create: `packages/pages-diagram-core/tsconfig.json`
- Create: `packages/pages-diagram-core/tsconfig.build.json`
- Create: `packages/pages-diagram-core/vitest.config.ts`
- Create: `packages/pages-diagram-core/src/index.ts`
- Create: `packages/pages-diagram-core/src/yaml-primitives.ts`
- Create: `packages/pages-diagram-core/src/yaml-primitives.test.ts`
- Create: `packages/pages-diagram-core/src/schema-registry.ts`
- Create: `packages/pages-diagram-core/src/schema-registry.test.ts`
- Create: `packages/pages-diagram-core/src/editors/pages-prompt-editor.ts`
- Create: `packages/pages-diagram-core/src/editors/pages-json-viewer.ts`
- Create: `packages/pages-diagram-core/src/editors/index.ts`
- Create: `packages/pages-diagram-core/src/editors/editors.test.ts`
- Modify: `package.json` (root — add workspace entry)
- Modify: `tsconfig.json` (root — add project reference)

**Interfaces:**
- Consumes: `yaml` package (parseDocument, CST manipulation)
- Produces:
  - `yamlSetField(yaml: string, path: readonly (string | number)[], value: unknown): string`
  - `yamlDeleteField(yaml: string, path: readonly (string | number)[]): string`
  - `registerPropertySchema(nodeType: string, schema: Record<string, unknown>): void`
  - `getPropertySchema(nodeType: string): Record<string, unknown> | undefined`
  - `PagesPromptEditor` class (element: `pages-prompt-editor`)
  - `PagesJsonViewer` class (element: `pages-json-viewer`)

- [ ] **Step 1: Create package scaffold**

`packages/pages-diagram-core/package.json`:
```json
{
  "name": "@casehubio/pages-diagram-core",
  "version": "0.1.0",
  "description": "Diagram editing infrastructure — DiagramBaseMixin, toolbar, export, YAML primitives",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/casehub-pages.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/graph-core": "workspace:*",
    "@casehubio/graph-renderer": "workspace:*",
    "@casehubio/pages-diagram-palette": "workspace:*",
    "@casehubio/pages-property-palette": "workspace:*",
    "html-to-image": "1.11.11",
    "lit": "^3.3.3",
    "yaml": "^2.7.0"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "jsdom": "^29.1.1",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

Create `tsconfig.json`, `tsconfig.build.json`, and `vitest.config.ts` following the pattern from `packages/pages-diagram-palette/`.

Add `"packages/pages-diagram-core"` to root `package.json` workspaces array.

Add `{ "path": "packages/pages-diagram-core" }` to root `tsconfig.json` references array.

- [ ] **Step 2: Write failing tests for YAML primitives**

`packages/pages-diagram-core/src/yaml-primitives.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { yamlSetField, yamlDeleteField } from './yaml-primitives.js';

describe('yamlSetField', () => {
  it('sets a top-level field', () => {
    const yaml = 'name: hello\n';
    const result = yamlSetField(yaml, ['name'], 'world');
    expect(result).toContain('name: world');
  });

  it('sets a nested field', () => {
    const yaml = 'spec:\n  name: hello\n';
    const result = yamlSetField(yaml, ['spec', 'name'], 'world');
    expect(result).toContain('name: world');
  });

  it('creates intermediate paths', () => {
    const yaml = 'name: hello\n';
    const result = yamlSetField(yaml, ['spec', 'nested', 'value'], 42);
    expect(result).toContain('value: 42');
  });

  it('preserves CST formatting of untouched fields', () => {
    const yaml = 'name: hello  # important comment\nage: 30\n';
    const result = yamlSetField(yaml, ['age'], 31);
    expect(result).toContain('# important comment');
    expect(result).toContain('age: 31');
  });

  it('handles array index paths', () => {
    const yaml = 'items:\n  - name: first\n  - name: second\n';
    const result = yamlSetField(yaml, ['items', 1, 'name'], 'updated');
    expect(result).toContain('name: updated');
    expect(result).toContain('name: first');
  });
});

describe('yamlDeleteField', () => {
  it('removes a field', () => {
    const yaml = 'name: hello\nage: 30\n';
    const result = yamlDeleteField(yaml, ['age']);
    expect(result).not.toContain('age');
    expect(result).toContain('name: hello');
  });

  it('removes a nested field', () => {
    const yaml = 'spec:\n  name: hello\n  age: 30\n';
    const result = yamlDeleteField(yaml, ['spec', 'age']);
    expect(result).not.toContain('age');
    expect(result).toContain('name: hello');
  });

  it('removes an array element', () => {
    const yaml = 'items:\n  - first\n  - second\n  - third\n';
    const result = yamlDeleteField(yaml, ['items', 1]);
    expect(result).toContain('first');
    expect(result).not.toContain('second');
    expect(result).toContain('third');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: FAIL — module not found

- [ ] **Step 4: Implement YAML primitives**

`packages/pages-diagram-core/src/yaml-primitives.ts`:
```ts
import { parseDocument } from 'yaml';

export function yamlSetField(
  yaml: string,
  path: readonly (string | number)[],
  value: unknown,
): string {
  const doc = parseDocument(yaml);
  doc.setIn([...path], value);
  return doc.toString();
}

export function yamlDeleteField(
  yaml: string,
  path: readonly (string | number)[],
): string {
  const doc = parseDocument(yaml);
  doc.deleteIn([...path]);
  return doc.toString();
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: PASS

- [ ] **Step 6: Write failing tests for schema-registry**

`packages/pages-diagram-core/src/schema-registry.test.ts`:
```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { registerPropertySchema, getPropertySchema, clearPropertySchemas } from './schema-registry.js';

describe('schema-registry', () => {
  beforeEach(() => clearPropertySchemas());

  it('returns undefined for unregistered type', () => {
    expect(getPropertySchema('unknown')).toBeUndefined();
  });

  it('registers and retrieves a schema', () => {
    const schema = { type: 'object', properties: { name: { type: 'string' } } };
    registerPropertySchema('myNode', schema);
    expect(getPropertySchema('myNode')).toEqual(schema);
  });

  it('overwrites on re-register', () => {
    registerPropertySchema('myNode', { v: 1 });
    registerPropertySchema('myNode', { v: 2 });
    expect(getPropertySchema('myNode')).toEqual({ v: 2 });
  });
});
```

- [ ] **Step 7: Implement schema-registry**

`packages/pages-diagram-core/src/schema-registry.ts`:
```ts
const schemas = new Map<string, Record<string, unknown>>();

export function registerPropertySchema(nodeType: string, schema: Record<string, unknown>): void {
  schemas.set(nodeType, schema);
}

export function getPropertySchema(nodeType: string): Record<string, unknown> | undefined {
  return schemas.get(nodeType);
}

export function clearPropertySchemas(): void {
  schemas.clear();
}
```

- [ ] **Step 8: Run tests to verify schema-registry passes**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: PASS (both yaml-primitives and schema-registry)

- [ ] **Step 9: Create editor components**

`packages/pages-diagram-core/src/editors/pages-prompt-editor.ts`:
```ts
import { LitElement, html, css } from 'lit';
import { property } from 'lit/decorators.js';

export class PagesPromptEditor extends LitElement {
  @property() value = '';

  static override styles = css`
    :host { display: block; }
    textarea { width: 100%; min-height: 80px; font-family: monospace; font-size: 13px; resize: vertical; }
  `;

  override render() {
    return html`
      <textarea
        .value=${this.value}
        aria-label="Prompt editor"
        @input=${this._onInput}></textarea>
    `;
  }

  private _onInput(e: Event) {
    this.value = (e.target as HTMLTextAreaElement).value;
    this.dispatchEvent(new CustomEvent('change', { detail: { value: this.value }, bubbles: true, composed: true }));
  }
}

if (!customElements.get('pages-prompt-editor')) {
  customElements.define('pages-prompt-editor', PagesPromptEditor);
}
```

`packages/pages-diagram-core/src/editors/pages-json-viewer.ts`:
```ts
import { LitElement, html, css } from 'lit';
import { property } from 'lit/decorators.js';

export class PagesJsonViewer extends LitElement {
  @property({ type: Object }) value: unknown = {};

  static override styles = css`
    :host { display: block; }
    pre { font-family: monospace; font-size: 12px; white-space: pre-wrap; word-break: break-all; margin: 0; }
  `;

  override render() {
    return html`<pre role="region" aria-label="JSON viewer">${JSON.stringify(this.value, null, 2)}</pre>`;
  }
}

if (!customElements.get('pages-json-viewer')) {
  customElements.define('pages-json-viewer', PagesJsonViewer);
}
```

`packages/pages-diagram-core/src/editors/index.ts`:
```ts
export { PagesPromptEditor } from './pages-prompt-editor.js';
export { PagesJsonViewer } from './pages-json-viewer.js';
```

- [ ] **Step 10: Write editor tests**

`packages/pages-diagram-core/src/editors/editors.test.ts`:
```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

describe('PagesPromptEditor', () => {
  it('module exports the class', async () => {
    const mod = await import('./pages-prompt-editor.js');
    expect(mod.PagesPromptEditor).toBeDefined();
    expect(mod.PagesPromptEditor.prototype).toBeInstanceOf(Object);
  });
});

describe('PagesJsonViewer', () => {
  it('module exports the class', async () => {
    const mod = await import('./pages-json-viewer.js');
    expect(mod.PagesJsonViewer).toBeDefined();
    expect(mod.PagesJsonViewer.prototype).toBeInstanceOf(Object);
  });
});
```

- [ ] **Step 11: Write barrel export**

`packages/pages-diagram-core/src/index.ts`:
```ts
export { yamlSetField, yamlDeleteField } from './yaml-primitives.js';
export { registerPropertySchema, getPropertySchema, clearPropertySchemas } from './schema-registry.js';
export { PagesPromptEditor } from './editors/pages-prompt-editor.js';
export { PagesJsonViewer } from './editors/pages-json-viewer.js';
```

- [ ] **Step 12: Run all tests, typecheck, commit**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Run: `yarn workspace @casehubio/pages-diagram-core run typecheck`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-diagram-core/ package.json tsconfig.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-diagram-core): scaffold package with YAML primitives, schema-registry, editors Refs #389"
```

---

### Task 2: diagram-export + DiagramToolbar + GitHubBackend

**Files:**
- Create: `packages/pages-diagram-core/src/diagram-export.ts`
- Create: `packages/pages-diagram-core/src/diagram-export.test.ts`
- Create: `packages/pages-diagram-core/src/diagram-toolbar.ts`
- Create: `packages/pages-diagram-core/src/github-backend.ts`
- Create: `packages/pages-diagram-core/src/github-backend.test.ts`
- Modify: `packages/pages-diagram-core/src/index.ts` (add exports)

**Interfaces:**
- Consumes: `html-to-image` (toSvg, toPng), `@casehubio/graph-core` (PersistenceBackend, ReadResult, WriteResult)
- Produces:
  - `exportDiagram(canvas: HTMLElement, nodes: ReadonlyArray<NodeLike>, format: ExportFormat, filename?: string, pixelRatio?: number): Promise<void>`
  - `computeNodeBounds(nodes: ReadonlyArray<NodeLike>): ExportBounds`
  - `computeExportViewport(bounds: ExportBounds, targetWidth: number, targetHeight: number, padding?: number): ExportViewport`
  - `type ExportFormat = 'svg' | 'png'`
  - `ExportBounds`, `ExportViewport` interfaces
  - `PagesDiagramToolbar` class (element: `pages-diagram-toolbar`)
  - `GitHubBackend` class implementing `PersistenceBackend`
  - `GitHubBackendConfig` interface

- [ ] **Step 1: Write failing tests for diagram-export pure functions**

`packages/pages-diagram-core/src/diagram-export.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { computeNodeBounds, computeExportViewport } from './diagram-export.js';

describe('computeNodeBounds', () => {
  it('returns zero bounds for empty array', () => {
    const bounds = computeNodeBounds([]);
    expect(bounds).toEqual({ x: 0, y: 0, width: 0, height: 0 });
  });

  it('computes bounds from node positions and sizes', () => {
    const nodes = [
      { position: { x: 10, y: 20 }, width: 100, height: 50 },
      { position: { x: 200, y: 30 }, width: 80, height: 40 },
    ];
    const bounds = computeNodeBounds(nodes);
    expect(bounds.x).toBe(10);
    expect(bounds.y).toBe(20);
    expect(bounds.width).toBe(270);
    expect(bounds.height).toBe(50);
  });

  it('uses measured dimensions when available', () => {
    const nodes = [
      { position: { x: 0, y: 0 }, measured: { width: 200, height: 100 } },
    ];
    const bounds = computeNodeBounds(nodes);
    expect(bounds.width).toBe(200);
    expect(bounds.height).toBe(100);
  });
});

describe('computeExportViewport', () => {
  it('returns identity for zero bounds', () => {
    const vp = computeExportViewport({ x: 0, y: 0, width: 0, height: 0 }, 1920, 1080);
    expect(vp).toEqual({ x: 0, y: 0, zoom: 1 });
  });

  it('zooms to fit content', () => {
    const bounds = { x: 0, y: 0, width: 500, height: 300 };
    const vp = computeExportViewport(bounds, 1920, 1080);
    expect(vp.zoom).toBeGreaterThan(0);
    expect(vp.zoom).toBeLessThanOrEqual(1920 / (500 + 40));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement diagram-export**

Copy `diagram-export.ts` from blocks-ui `packages/diagram-core/src/diagram-export.ts` verbatim — the code is already generic with zero domain imports. No changes needed.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: PASS

- [ ] **Step 5: Create DiagramToolbar**

`packages/pages-diagram-core/src/diagram-toolbar.ts`:
```ts
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';

export class PagesDiagramToolbar extends LitElement {
  @property({ type: Boolean }) dirty = false;
  @property({ type: Boolean }) saving = false;
  @property({ type: Boolean }) hasBackend = false;
  @property({ type: Boolean }) hasNodes = false;

  static override styles = css`
    :host { display: flex; align-items: center; gap: 8px; padding: 4px 12px; border-bottom: 1px solid var(--pages-border-color, #ddd); height: 32px; box-sizing: border-box; font-family: var(--pages-font-family, system-ui, sans-serif); }
    button {
      border: 1px solid var(--pages-border-color, #ccc); border-radius: 4px;
      background: var(--pages-surface-color, #fff); cursor: pointer;
      padding: 2px 10px; font-size: 12px; color: var(--pages-text-color, #333);
      display: flex; align-items: center; gap: 4px;
    }
    button:hover:not(:disabled) { background: var(--pages-surface-raised, #f5f5f5); }
    button:disabled { opacity: 0.4; cursor: default; }
    .dirty-dot { width: 6px; height: 6px; border-radius: 50%; background: var(--pages-warning-color, #f59e0b); }
    .spacer { flex: 1; }
  `;

  override render() {
    const saveSection = this.hasBackend ? html`
      <button ?disabled=${!this.dirty || this.saving} @click=${this._save}>
        ${this.saving ? 'Saving…' : 'Save'}
      </button>
      ${this.dirty ? html`<span class="dirty-dot"></span>` : nothing}
    ` : nothing;

    return html`
      ${saveSection}
      <span class="spacer"></span>
      <button ?disabled=${!this.hasNodes} @click=${() => this._export('svg')}>Export SVG</button>
      <button ?disabled=${!this.hasNodes} @click=${() => this._export('png')}>Export PNG</button>
    `;
  }

  private _save(): void {
    this.dispatchEvent(new CustomEvent('toolbar-save', { bubbles: true, composed: true }));
  }

  private _export(format: 'svg' | 'png'): void {
    this.dispatchEvent(new CustomEvent('toolbar-export', {
      detail: { format },
      bubbles: true,
      composed: true,
    }));
  }
}

if (!customElements.get('pages-diagram-toolbar')) {
  customElements.define('pages-diagram-toolbar', PagesDiagramToolbar);
}
```

- [ ] **Step 6: Write failing tests for GitHubBackend**

`packages/pages-diagram-core/src/github-backend.test.ts`:
```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { GitHubBackend } from './github-backend.js';

describe('GitHubBackend', () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it('reads a file via GitHub API', async () => {
    const yamlContent = btoa('name: test\n');
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(
      new Response(JSON.stringify({ content: yamlContent, sha: 'abc123' }), { status: 200 }),
    );

    const backend = new GitHubBackend({ token: 'tok', owner: 'org', repo: 'repo' });
    const result = await backend.read('path/file.yaml');

    expect(result.status).toBe('ok');
    if (result.status === 'ok') {
      expect(result.yaml).toBe('name: test\n');
      expect(result.version).toBe('abc123');
    }
  });

  it('returns not_found for 404', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(new Response('', { status: 404 }));

    const backend = new GitHubBackend({ token: 'tok', owner: 'org', repo: 'repo' });
    const result = await backend.read('missing.yaml');
    expect(result.status).toBe('not_found');
  });

  it('writes a file via GitHub API', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(
      new Response(JSON.stringify({ content: { sha: 'new-sha' } }), { status: 200 }),
    );

    const backend = new GitHubBackend({ token: 'tok', owner: 'org', repo: 'repo' });
    const result = await backend.write('path/file.yaml', 'name: updated\n', 'old-sha');

    expect(result.status).toBe('ok');
    if (result.status === 'ok') {
      expect(result.version).toBe('new-sha');
    }
  });

  it('uses "Update document" as default commit message', async () => {
    const fetchSpy = vi.spyOn(globalThis, 'fetch').mockResolvedValue(
      new Response(JSON.stringify({ content: { sha: 'sha' } }), { status: 200 }),
    );

    const backend = new GitHubBackend({ token: 'tok', owner: 'org', repo: 'repo' });
    await backend.write('f.yaml', 'content', '');

    const body = JSON.parse(fetchSpy.mock.calls[0]![1]!.body as string);
    expect(body.message).toBe('Update document');
  });

  it('detects conflict on 409', async () => {
    vi.spyOn(globalThis, 'fetch')
      .mockResolvedValueOnce(new Response('', { status: 409 }))
      .mockResolvedValueOnce(new Response(JSON.stringify({ sha: 'conflict-sha' }), { status: 200 }));

    const backend = new GitHubBackend({ token: 'tok', owner: 'org', repo: 'repo' });
    const result = await backend.write('f.yaml', 'content', 'old-sha');
    expect(result.status).toBe('conflict');
  });
});
```

- [ ] **Step 7: Implement GitHubBackend**

`packages/pages-diagram-core/src/github-backend.ts`:
Port from blocks-ui `packages/graph-stencil-case/src/persistence/github-backend.ts` with one change: default commit message from `'Update case definition'` to `'Update document'`.

```ts
import type { PersistenceBackend, ReadResult, WriteResult } from '@casehubio/graph-core';

export interface GitHubBackendConfig {
  readonly token: string;
  readonly owner: string;
  readonly repo: string;
  readonly branch?: string;
  readonly commitMessage?: string;
}

export class GitHubBackend implements PersistenceBackend {
  private readonly token: string;
  private readonly owner: string;
  private readonly repo: string;
  private readonly branch: string;
  private readonly commitMessage: string;

  constructor(config: GitHubBackendConfig) {
    this.token = config.token;
    this.owner = config.owner;
    this.repo = config.repo;
    this.branch = config.branch ?? 'main';
    this.commitMessage = config.commitMessage ?? 'Update document';
  }

  async read(uri: string): Promise<ReadResult> {
    const url = `https://api.github.com/repos/${this.owner}/${this.repo}/contents/${uri}?ref=${this.branch}`;
    const res = await fetch(url, {
      headers: { Authorization: `Bearer ${this.token}`, Accept: 'application/vnd.github.v3+json' },
    });

    if (res.status === 404) return { status: 'not_found', uri };
    if (!res.ok) throw new Error('GitHub API error: ' + String(res.status));

    const data = (await res.json()) as { content: string; sha: string };
    const yaml = atob(data.content.split(String.fromCharCode(10)).join(''));
    return { status: 'ok', yaml, version: data.sha };
  }

  async write(uri: string, yaml: string, expectedVersion: string): Promise<WriteResult> {
    const url = `https://api.github.com/repos/${this.owner}/${this.repo}/contents/${uri}`;
    const body: Record<string, unknown> = {
      message: this.commitMessage,
      content: btoa(yaml),
      branch: this.branch,
    };
    if (expectedVersion) body.sha = expectedVersion;

    const res = await fetch(url, {
      method: 'PUT',
      headers: {
        Authorization: `Bearer ${this.token}`,
        Accept: 'application/vnd.github.v3+json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    });

    if (res.status === 409) {
      const current = await fetch(`${url}?ref=${this.branch}`, {
        headers: { Authorization: `Bearer ${this.token}`, Accept: 'application/vnd.github.v3+json' },
      });
      const currentData = (await current.json()) as { sha: string };
      return { status: 'conflict', currentVersion: currentData.sha };
    }

    const data = (await res.json()) as { content: { sha: string } };
    return { status: 'ok', version: data.content.sha };
  }
}
```

- [ ] **Step 8: Run tests to verify GitHubBackend passes**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: PASS

- [ ] **Step 9: Update barrel export**

Add to `packages/pages-diagram-core/src/index.ts`:
```ts
export { exportDiagram, computeNodeBounds, computeExportViewport } from './diagram-export.js';
export type { ExportFormat, ExportBounds, ExportViewport } from './diagram-export.js';
export { PagesDiagramToolbar } from './diagram-toolbar.js';
export { GitHubBackend } from './github-backend.js';
export type { GitHubBackendConfig } from './github-backend.js';
```

- [ ] **Step 10: Typecheck and commit**

Run: `yarn workspace @casehubio/pages-diagram-core run typecheck`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-diagram-core/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-diagram-core): add diagram-export, toolbar, GitHubBackend Refs #389"
```

---

### Task 3: DiagramBaseMixin

**Files:**
- Create: `packages/pages-diagram-core/src/diagram-base-mixin.ts`
- Create: `packages/pages-diagram-core/src/diagram-base-mixin.test.ts`
- Modify: `packages/pages-diagram-core/src/index.ts` (add exports)

**Interfaces:**
- Consumes:
  - `computeElkLayout`, `toReactFlowGraph` from `@casehubio/graph-renderer`
  - `ElkLayoutOptions`, `ElkLayoutResult`, `EditPolicy`, `GraphEdit` types from `@casehubio/graph-renderer`
  - `PersistenceBackend`, `GraphModel`, `NodeDecoration` from `@casehubio/graph-core`
  - `Node`, `Edge` types from `@xyflow/react`
  - `PropertyPaletteSource`, `EditorResolver` from `@casehubio/pages-property-palette`
  - `PaletteItem`, `PaletteSelectDetail` from `@casehubio/pages-diagram-palette`
  - `getPropertySchema` from `./schema-registry.js`
  - `exportDiagram`, `ExportFormat` from `./diagram-export.js`
- Produces:
  - `DiagramBaseMixin<T>(Base: T)` — mixin function
  - `DiagramBaseInterface` — interface declaration
  - `AdapterResult` interface

- [ ] **Step 1: Write failing test for DiagramBaseMixin**

`packages/pages-diagram-core/src/diagram-base-mixin.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { DiagramBaseMixin } from './diagram-base-mixin.js';

describe('DiagramBaseMixin', () => {
  it('exports the mixin function', () => {
    expect(typeof DiagramBaseMixin).toBe('function');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement DiagramBaseMixin**

Port from blocks-ui `packages/diagram-core/src/diagram-base-mixin.ts` verbatim. The code is already generic — all imports resolve to pages packages (`@casehubio/graph-renderer`, `@casehubio/graph-core`, `@casehubio/pages-property-palette`, `@casehubio/pages-diagram-palette`).

The only change: internal imports switch from `./diagram-export.js` and `./schema-registry.js` (same as source since they're now co-located in the same package).

Copy the file. Verify all imports are correct for the pages package structure.

- [ ] **Step 4: Run tests and typecheck**

Run: `yarn workspace @casehubio/pages-diagram-core run test`
Run: `yarn workspace @casehubio/pages-diagram-core run typecheck`
Expected: PASS

- [ ] **Step 5: Update barrel export**

Add to `packages/pages-diagram-core/src/index.ts`:
```ts
export { DiagramBaseMixin } from './diagram-base-mixin.js';
export type { AdapterResult, DiagramBaseInterface } from './diagram-base-mixin.js';
```

- [ ] **Step 6: Full build verification**

Run: `yarn build:packages`
Expected: PASS — all packages including pages-diagram-core build successfully.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-diagram-core/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-diagram-core): add DiagramBaseMixin — completes package Refs #389"
```

---

## Batch 2: pages-ui-components extensions

After this batch: status-badge + registry SPI, split-workbench, and event-trail are available in `@casehubio/pages-ui-components` via sub-path exports. Full build passes.

### Task 4: Status registry SPI + StatusBadge component

**Files:**
- Create: `packages/pages-ui-components/src/status-badge/status-registry.ts`
- Create: `packages/pages-ui-components/src/status-badge/status-registry.test.ts`
- Create: `packages/pages-ui-components/src/status-badge/category-styles.ts`
- Create: `packages/pages-ui-components/src/status-badge/pages-status-badge.ts`
- Create: `packages/pages-ui-components/src/status-badge/pages-status-badge.test.ts`
- Create: `packages/pages-ui-components/src/status-badge/index.ts`
- Modify: `packages/pages-ui-components/package.json` (add `./status-badge` sub-path export)
- Modify: `packages/pages-ui-components/src/index.ts` (add re-export)

**Interfaces:**
- Consumes: nothing external (pure types + lit)
- Produces:
  - `type StateCategory = 'active' | 'info' | 'success' | 'danger' | 'neutral' | 'transfer' | 'warning'`
  - `interface StatusDescriptor { readonly category: StateCategory; readonly icon: string; readonly label?: string; readonly pulse?: boolean; readonly border?: boolean; }`
  - `const FALLBACK_DESCRIPTOR: StatusDescriptor`
  - `registerStatus(domain: string, state: string, descriptor: StatusDescriptor): void`
  - `lookupStatus(domain: string | undefined, state: string): StatusDescriptor`
  - `interface CategoryStyle { readonly background: string; readonly color: string; }`
  - `stateCategoryStyles(category: StateCategory): CategoryStyle`
  - `PagesStatusBadge` class (element: `pages-status-badge`)

- [ ] **Step 1: Write failing tests for status-registry**

`packages/pages-ui-components/src/status-badge/status-registry.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { registerStatus, lookupStatus, FALLBACK_DESCRIPTOR } from './status-registry.js';
import type { StatusDescriptor } from './status-registry.js';

describe('status-registry', () => {
  it('returns fallback for unknown domain/state', () => {
    expect(lookupStatus('unknown', 'UNKNOWN')).toBe(FALLBACK_DESCRIPTOR);
  });

  it('returns wildcard match when no domain-specific entry exists', () => {
    const result = lookupStatus('anything', 'RUNNING');
    expect(result.category).toBe('success');
    expect(result.icon).toBe('▶');
  });

  it('returns domain-specific match when registered', () => {
    const descriptor: StatusDescriptor = { category: 'danger', icon: '💥' };
    registerStatus('myDomain', 'EXPLODED', descriptor);
    expect(lookupStatus('myDomain', 'EXPLODED')).toBe(descriptor);
  });

  it('falls back to wildcard when domain has no match', () => {
    registerStatus('myDomain', 'CUSTOM', { category: 'info', icon: '★' });
    expect(lookupStatus('myDomain', 'PENDING').category).toBe('neutral');
  });

  it('returns wildcard when domain is undefined', () => {
    const result = lookupStatus(undefined, 'COMPLETED');
    expect(result.category).toBe('success');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement status-registry**

`packages/pages-ui-components/src/status-badge/status-registry.ts`:

Extract only the generic SPI + wildcard `*:` registrations from blocks-ui `status.ts`. Domain registrations (case, task, work, commitment, execution, agent, pattern, etc.) stay in blocks-ui.

```ts
export type StateCategory = 'active' | 'info' | 'success' | 'danger'
  | 'neutral' | 'transfer' | 'warning';

export interface StatusDescriptor {
  readonly category: StateCategory;
  readonly icon: string;
  readonly label?: string;
  readonly pulse?: boolean;
  readonly border?: boolean;
}

export const FALLBACK_DESCRIPTOR: StatusDescriptor = { category: 'neutral', icon: '?' };

const REGISTRY = new Map<string, StatusDescriptor>([
  ['*:PENDING',    { category: 'neutral', icon: '○' }],
  ['*:RUNNING',    { category: 'success', icon: '▶', pulse: true, border: true }],
  ['*:COMPLETED',  { category: 'success', icon: '✓' }],
  ['*:FAULTED',    { category: 'danger',  icon: '!' }],
  ['*:CANCELLED',  { category: 'neutral', icon: '/' }],
  ['*:SUSPENDED',  { category: 'warning', icon: '⏸', border: true }],
]);

export function registerStatus(domain: string, state: string, descriptor: StatusDescriptor): void {
  REGISTRY.set(`${domain}:${state}`, descriptor);
}

export function lookupStatus(domain: string | undefined, state: string): StatusDescriptor {
  if (domain) {
    const exact = REGISTRY.get(`${domain}:${state}`);
    if (exact) return exact;
  }
  return REGISTRY.get(`*:${state}`) ?? FALLBACK_DESCRIPTOR;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: PASS

- [ ] **Step 5: Create category-styles**

`packages/pages-ui-components/src/status-badge/category-styles.ts`:
```ts
import type { StateCategory } from './status-registry.js';

export interface CategoryStyle {
  readonly background: string;
  readonly color: string;
}

const CATEGORY_STYLES: Record<StateCategory, CategoryStyle> = {
  active:   { background: 'var(--pages-accent-3, #e0e7ff)',  color: 'var(--pages-accent-11, #3730a3)' },
  info:     { background: 'var(--pages-info-3, #dbeafe)',    color: 'var(--pages-info-11, #1e40af)' },
  success:  { background: 'var(--pages-success-3, #d1fae5)', color: 'var(--pages-success-11, #065f46)' },
  danger:   { background: 'var(--pages-danger-3, #fee2e2)',  color: 'var(--pages-danger-11, #991b1b)' },
  neutral:  { background: 'var(--pages-neutral-3, #e5e5e5)', color: 'var(--pages-neutral-9, #737373)' },
  transfer: { background: 'var(--pages-info-3, #dbeafe)',    color: 'var(--pages-info-11, #1e40af)' },
  warning:  { background: 'var(--pages-warning-3, #fef3c7)', color: 'var(--pages-warning-11, #92400e)' },
};

export function stateCategoryStyles(category: StateCategory): CategoryStyle {
  return CATEGORY_STYLES[category];
}
```

- [ ] **Step 6: Write StatusBadge test**

`packages/pages-ui-components/src/status-badge/pages-status-badge.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { PagesStatusBadge } from './pages-status-badge.js';

describe('PagesStatusBadge', () => {
  it('exports the class', () => {
    expect(PagesStatusBadge).toBeDefined();
    expect(typeof PagesStatusBadge).toBe('function');
  });
});
```

- [ ] **Step 7: Implement PagesStatusBadge**

`packages/pages-ui-components/src/status-badge/pages-status-badge.ts`:
```ts
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import { styleMap } from 'lit/directives/style-map.js';
import { lookupStatus } from './status-registry.js';
import { stateCategoryStyles } from './category-styles.js';

export class PagesStatusBadge extends LitElement {
  @property({ type: String }) state?: string;
  @property({ type: String }) domain?: string;
  @property({ type: String }) size: 'sm' | 'md' = 'sm';
  @property({ type: Boolean }) showIcon = false;

  static override styles = css`
    :host { display: inline-block; }
  `;

  override render() {
    if (!this.state) return nothing;
    const descriptor = lookupStatus(this.domain, this.state);
    const colors = stateCategoryStyles(descriptor.category);
    const fontSize = this.size === 'md' ? '12px' : '10px';
    const padding = this.size === 'md' ? '2px 8px' : '1px 6px';
    const displayLabel = descriptor.label ?? this.state;

    const styles = {
      display: 'inline-flex',
      alignItems: 'center',
      gap: '3px',
      fontSize,
      fontWeight: '500',
      padding,
      borderRadius: '9999px',
      textTransform: 'uppercase',
      letterSpacing: '0.5px',
      lineHeight: '1.4',
      background: colors.background,
      color: colors.color,
    };

    return html`
      <span class="pill" style=${styleMap(styles)} aria-label="Status: ${displayLabel}">
        ${this.showIcon ? html`<span class="icon">${descriptor.icon}</span>` : nothing}
        ${displayLabel}
      </span>
    `;
  }
}

if (!customElements.get('pages-status-badge')) {
  customElements.define('pages-status-badge', PagesStatusBadge);
}
```

- [ ] **Step 8: Create barrel export and update package.json**

`packages/pages-ui-components/src/status-badge/index.ts`:
```ts
export { PagesStatusBadge } from './pages-status-badge.js';
export { registerStatus, lookupStatus, FALLBACK_DESCRIPTOR } from './status-registry.js';
export type { StateCategory, StatusDescriptor } from './status-registry.js';
export { stateCategoryStyles } from './category-styles.js';
export type { CategoryStyle } from './category-styles.js';
```

Add to `packages/pages-ui-components/package.json` exports map:
```json
"./status-badge": {
  "types": "./dist/status-badge/index.d.ts",
  "default": "./dist/status-badge/index.js"
}
```

Add `"./dist/status-badge/index.js"` and `"./src/status-badge/index.ts"` to the `sideEffects` array.

Add to `packages/pages-ui-components/src/index.ts`:
```ts
export * from './status-badge/index.js';
```

- [ ] **Step 9: Run tests, typecheck, commit**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Run: `yarn workspace @casehubio/pages-ui-components run typecheck`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui-components/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-ui-components): add status-badge + registry SPI Refs #389"
```

---

### Task 5: SplitWorkbench + EventTrail

**Files:**
- Create: `packages/pages-ui-components/src/split-workbench/pages-split-workbench.ts`
- Create: `packages/pages-ui-components/src/split-workbench/pages-split-workbench.test.ts`
- Create: `packages/pages-ui-components/src/split-workbench/index.ts`
- Create: `packages/pages-ui-components/src/event-trail/pages-event-trail.ts`
- Create: `packages/pages-ui-components/src/event-trail/pages-event-trail.test.ts`
- Create: `packages/pages-ui-components/src/event-trail/index.ts`
- Modify: `packages/pages-ui-components/package.json` (add sub-path exports + dependencies)
- Modify: `packages/pages-ui-components/src/index.ts` (add re-exports)

**Interfaces:**
- Consumes:
  - `onPagesEvent`, `emitPagesEvent`, `DataSourceMixin` from `@casehubio/pages-component`
  - `LiveRegionMixin` from `@casehubio/pages-primitives/a11y`
  - `fromRows`, `TypedDataSet`, `TypedRow`, `ColumnId`, `ColumnType`, `DataSource`, `DataSink` from `@casehubio/pages-data`
  - `TableColumnConfig`, `ColumnRenderer` from `@casehubio/pages-table`
  - `EMPTY_FILTER_STATE`, `FilterState` from `@casehubio/pages-filter-bar`
- Produces:
  - `PagesSplitWorkbench` class (element: `pages-split-workbench`)
  - `PagesEventTrail` class (element: `pages-event-trail`)

- [ ] **Step 1: Add dependencies to package.json**

Add to `packages/pages-ui-components/package.json` dependencies:
```json
"@casehubio/pages-component": "workspace:*",
"@casehubio/pages-data": "workspace:*",
"@casehubio/pages-filter-bar": "workspace:*",
"@casehubio/pages-primitives": "workspace:*",
"@casehubio/pages-table": "workspace:*"
```

These are needed for split-workbench (pages-component, pages-primitives) and event-trail (all five). Sub-path exports ensure consumers who only import `./checkbox` or `./input` don't execute this code — tree-shaking handles the rest.

- [ ] **Step 2: Write failing test for SplitWorkbench**

`packages/pages-ui-components/src/split-workbench/pages-split-workbench.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { PagesSplitWorkbench } from './pages-split-workbench.js';

describe('PagesSplitWorkbench', () => {
  it('exports the class', () => {
    expect(PagesSplitWorkbench).toBeDefined();
    expect(typeof PagesSplitWorkbench).toBe('function');
  });
});
```

- [ ] **Step 3: Implement PagesSplitWorkbench**

`packages/pages-ui-components/src/split-workbench/pages-split-workbench.ts`:

Port from blocks-ui `components/split-workbench/src/split-workbench.ts`. Changes:
- Class name: `SplitWorkbench` → `PagesSplitWorkbench`
- Element name: `blocks-split-workbench` → `pages-split-workbench`
- Use guarded manual registration (no `@customElement` decorator)
- Remove `blocks-ui-core` dependency (it was stale — split-workbench doesn't use it)
- Imports stay the same (`@casehubio/pages-component`, `@casehubio/pages-primitives/a11y`)

```ts
import { LitElement, html, css, nothing } from 'lit';
import { property, state } from 'lit/decorators.js';
import { onPagesEvent, emitPagesEvent } from '@casehubio/pages-component';
import { LiveRegionMixin } from '@casehubio/pages-primitives/a11y';

export class PagesSplitWorkbench extends LiveRegionMixin(LitElement) {
  // ... (port entire class body from blocks-ui, unchanged)
}

if (!customElements.get('pages-split-workbench')) {
  customElements.define('pages-split-workbench', PagesSplitWorkbench);
}

declare global {
  interface HTMLElementTagNameMap {
    'pages-split-workbench': PagesSplitWorkbench;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: PASS

- [ ] **Step 5: Write failing test for EventTrail**

`packages/pages-ui-components/src/event-trail/pages-event-trail.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { PagesEventTrail } from './pages-event-trail.js';

describe('PagesEventTrail', () => {
  it('exports the class', () => {
    expect(PagesEventTrail).toBeDefined();
    expect(typeof PagesEventTrail).toBe('function');
  });
});
```

- [ ] **Step 6: Implement PagesEventTrail**

`packages/pages-ui-components/src/event-trail/pages-event-trail.ts`:

Port from blocks-ui `components/event-trail/src/event-trail.ts`. Changes:
- Class name: `BlocksEventTrail` → `PagesEventTrail`
- Element name: `blocks-event-trail` → `pages-event-trail`
- Use guarded manual registration
- All imports unchanged — they already point to pages packages

```ts
import { LitElement, html, css, nothing, type PropertyValues, type TemplateResult } from 'lit';
import { property, state } from 'lit/decorators.js';
import { DataSourceMixin } from '@casehubio/pages-component';
import { LiveRegionMixin } from '@casehubio/pages-primitives/a11y';
import { fromRows } from '@casehubio/pages-data';
import type { TypedDataSet, TypedRow, ColumnId, ColumnType } from '@casehubio/pages-data';
import type { TableColumnConfig, ColumnRenderer } from '@casehubio/pages-table';
import { EMPTY_FILTER_STATE, type FilterState } from '@casehubio/pages-filter-bar';
import type { DataSource, DataSink } from '@casehubio/pages-data';
import '@casehubio/pages-table';
import '@casehubio/pages-filter-bar';

// ... (port entire class body from blocks-ui, unchanged)

export class PagesEventTrail extends DataSourceMixin(LiveRegionMixin(LitElement)) {
  // ... full implementation
}

if (!customElements.get('pages-event-trail')) {
  customElements.define('pages-event-trail', PagesEventTrail);
}
```

- [ ] **Step 7: Create barrel exports**

`packages/pages-ui-components/src/split-workbench/index.ts`:
```ts
export { PagesSplitWorkbench } from './pages-split-workbench.js';
```

`packages/pages-ui-components/src/event-trail/index.ts`:
```ts
export { PagesEventTrail } from './pages-event-trail.js';
```

- [ ] **Step 8: Update package.json exports map**

Add to `packages/pages-ui-components/package.json`:

Exports:
```json
"./split-workbench": {
  "types": "./dist/split-workbench/index.d.ts",
  "default": "./dist/split-workbench/index.js"
},
"./event-trail": {
  "types": "./dist/event-trail/index.d.ts",
  "default": "./dist/event-trail/index.js"
}
```

Add to sideEffects array:
```json
"./dist/split-workbench/index.js",
"./src/split-workbench/index.ts",
"./dist/event-trail/index.js",
"./src/event-trail/index.ts"
```

- [ ] **Step 9: Update main barrel export**

Add to `packages/pages-ui-components/src/index.ts`:
```ts
export * from './split-workbench/index.js';
export * from './event-trail/index.js';
```

- [ ] **Step 10: Full build verification and commit**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Run: `yarn workspace @casehubio/pages-ui-components run typecheck`
Run: `yarn build:packages`
Expected: PASS — all packages build, no circular dependencies

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui-components/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-ui-components): add split-workbench, event-trail Refs #389"
```

---

## Batch 3: blocks-ui full migration

After this batch: blocks-ui's diagram-core, status-badge, split-workbench, and event-trail source code is deleted — replaced by direct imports from pages. All consumers updated. No re-exports anywhere.

**NOTE:** This batch operates on the blocks-ui repo (`/Users/mdproctor/claude/casehub/blocks-ui/`). A separate blocks-ui issue must be created first, linked to casehub-pages#389.

### Task 6: Delete diagram-core source, update diagram consumers

**Files (all in blocks-ui repo):**
- Delete: `packages/diagram-core/src/diagram-base-mixin.ts`
- Delete: `packages/diagram-core/src/diagram-base-mixin.test.ts`
- Delete: `packages/diagram-core/src/diagram-toolbar.ts`
- Delete: `packages/diagram-core/src/diagram-export.ts`
- Delete: `packages/diagram-core/src/diagram-export.test.ts`
- Delete: `packages/diagram-core/src/schema-registry.ts`
- Delete: `packages/diagram-core/src/schema-registry.test.ts`
- Delete: `packages/diagram-core/src/editors/blocks-prompt-editor.ts`
- Delete: `packages/diagram-core/src/editors/blocks-json-editor.ts`
- Delete: `packages/diagram-core/src/editors/editors.test.ts`
- Delete: `packages/diagram-core/src/editors/index.ts`
- Delete: `packages/diagram-core/src/index.ts`
- Delete: `packages/graph-stencil-case/src/persistence/github-backend.ts`
- Delete: `packages/graph-stencil-case/src/persistence/github-backend.test.ts`
- Modify: `packages/diagram-core/package.json` (remove all deps, mark deprecated or remove package entirely)
- Modify: `components/casehub-diagram/package.json` (dep `@casehubio/diagram-core` → `@casehubio/pages-diagram-core`)
- Modify: `components/casehub-diagram/src/casehub-diagram.ts` (imports from `@casehubio/pages-diagram-core`, element names `pages-diagram-toolbar`/`pages-prompt-editor`/`pages-json-viewer` in templates)
- Modify: `components/swf-diagram/package.json` (same dep switch)
- Modify: `components/swf-diagram/src/swf-diagram.ts` (same import switch)
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.ts` (use pages YAML primitives)
- Modify: `packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts` (use pages YAML primitives)
- Modify: `packages/graph-stencil-case/package.json` (add `@casehubio/pages-diagram-core` dep, remove `yaml` if only used by the deleted generic function)
- Modify: `packages/graph-stencil-swf/package.json` (add `@casehubio/pages-diagram-core` dep)
- Modify: `packages/graph-stencil-case/src/index.ts` (remove GitHubBackend re-export)

**Interfaces:**
- Consumes: all exports from `@casehubio/pages-diagram-core`
- Produces: nothing new — deletes source, updates imports

- [ ] **Step 1: Create blocks-ui issue**

```bash
gh issue create --repo casehubio/blocks-ui \
  --title "Migrate primitives to casehub-pages — delete source, update imports" \
  --body "Full migration of diagram-core, status-badge, split-workbench, and event-trail to casehub-pages. Source is deleted from blocks-ui, all consumers updated to import directly from pages packages. No re-exports.

Linked to: casehubio/casehub-pages#389"
```

Record the issue number for commit references.

- [ ] **Step 2: Delete diagram-core package source**

Delete all source files from `packages/diagram-core/src/`. The entire package is being replaced by `@casehubio/pages-diagram-core`.

Either remove the package from the workspace entirely, or leave a minimal `package.json` with a deprecation notice pointing to `@casehubio/pages-diagram-core`.

- [ ] **Step 3: Update casehub-diagram imports**

`components/casehub-diagram/package.json` — replace dep:
```diff
- "@casehubio/diagram-core": "workspace:*",
+ "@casehubio/pages-diagram-core": "*",
```

`components/casehub-diagram/src/casehub-diagram.ts` — update all imports:
```ts
// Before:
import { DiagramBaseMixin, type AdapterResult } from '@casehubio/diagram-core';
// After:
import { DiagramBaseMixin, type AdapterResult } from '@casehubio/pages-diagram-core';
```

Update template element names:
- `<diagram-toolbar` → `<pages-diagram-toolbar`
- `<blocks-prompt-editor` → `<pages-prompt-editor`
- `<blocks-json-editor` → `<pages-json-viewer`
- Any `toolbar-save`/`toolbar-export` event listeners stay unchanged (event names don't change)

- [ ] **Step 4: Update swf-diagram imports**

Same pattern as Step 3 for `components/swf-diagram/`.

- [ ] **Step 5: Switch stencil YAML functions to pages primitives**

`packages/graph-stencil-case/src/adapter/yaml-editor.ts` — replace the generic `applyPropertyEdit` body:
```ts
import { yamlSetField, yamlDeleteField } from '@casehubio/pages-diagram-core';

export function applyPropertyEdit(
  yaml: string,
  nodePath: readonly (string | number)[],
  field: readonly (string | number)[],
  value: unknown,
): string {
  const fullPath = [...nodePath, ...field];
  return value === undefined
    ? yamlDeleteField(yaml, fullPath)
    : yamlSetField(yaml, fullPath, value);
}
```

The function signature stays the same — it wraps the pages primitives. Domain-specific functions (`addElement`, `removeElement`, `switchBindingTarget`, etc.) stay unchanged — they are case-domain logic, not generic primitives.

Apply same pattern to `packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts`:
```ts
import { yamlSetField, yamlDeleteField } from '@casehubio/pages-diagram-core';

export function applySwfPropertyEdit(
  yaml: string,
  nodePath: readonly (string | number)[],
  field: (string | number)[],
  value: unknown,
): string {
  const fullPath = [...nodePath, ...field];
  return value === undefined
    ? yamlDeleteField(yaml, fullPath)
    : yamlSetField(yaml, fullPath, value);
}
```

- [ ] **Step 6: Delete GitHubBackend from graph-stencil-case**

Delete `packages/graph-stencil-case/src/persistence/github-backend.ts` and its test.

Update `packages/graph-stencil-case/src/index.ts` — remove the GitHubBackend export line. Consumers import `GitHubBackend` from `@casehubio/pages-diagram-core` directly.

Add `@casehubio/pages-diagram-core` dep to `packages/graph-stencil-case/package.json` and `packages/graph-stencil-swf/package.json`.

- [ ] **Step 7: Run blocks-ui build and tests**

Run: full blocks-ui build and test suite
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add .
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: delete diagram-core, migrate to pages-diagram-core

Source deleted from packages/diagram-core/. casehub-diagram and
swf-diagram now import directly from @casehubio/pages-diagram-core.
Stencil YAML functions delegate to pages yamlSetField/yamlDeleteField.
GitHubBackend deleted — consumers import from pages-diagram-core.

Refs casehubio/casehub-pages#389
Refs blocks-ui#<N>"
```

---

### Task 7: Delete status-badge, split-workbench, event-trail — update all consumers

**Files (all in blocks-ui repo):**
- Delete: `packages/blocks-ui-core/src/status-badge/status-badge.ts`
- Delete: `packages/blocks-ui-core/src/status-badge/status-badge.test.ts`
- Delete: `packages/blocks-ui-core/src/status-badge/index.ts`
- Delete: `packages/blocks-ui-core/src/styles/category.ts`
- Modify: `packages/blocks-ui-core/src/types/status.ts` (keep domain registrations, import SPI from pages)
- Modify: `packages/blocks-ui-core/src/index.ts` (remove status-badge and category exports)
- Modify: `packages/blocks-ui-core/package.json` (add `@casehubio/pages-ui-components` dep)
- Delete: `components/split-workbench/src/split-workbench.ts` (entire source)
- Delete: `components/split-workbench/src/split-workbench.test.ts`
- Delete: `components/split-workbench/src/index.ts`
- Delete: `components/event-trail/src/event-trail.ts` (entire source)
- Delete: `components/event-trail/src/event-trail.test.ts`
- Delete: `components/event-trail/src/index.ts`
- Modify: 9 split-workbench consumers (update import + element name)
- Modify: 1 event-trail consumer (update import + element name)
- Modify: 1 status-badge consumer (update import + element name)

**Interfaces:**
- Consumes: `@casehubio/pages-ui-components/status-badge`, `@casehubio/pages-ui-components/split-workbench`, `@casehubio/pages-ui-components/event-trail`
- Produces: nothing new — deletes source, updates imports

- [ ] **Step 1: Update blocks-ui-core status types**

`packages/blocks-ui-core/src/types/status.ts` — import SPI from pages, keep only domain registrations:
```ts
import { registerStatus } from '@casehubio/pages-ui-components/status-badge';
export type { StateCategory, StatusDescriptor } from '@casehubio/pages-ui-components/status-badge';

// Domain registrations — CaseHub business state mappings
registerStatus('case', 'STARTING', { category: 'info',    icon: '◐' });
registerStatus('case', 'WAITING',  { category: 'warning', icon: '⏳' });
registerStatus('task', 'DELEGATED', { category: 'info',    icon: '→', border: true });
registerStatus('task', 'REJECTED',  { category: 'warning', icon: '✕' });
registerStatus('task', 'OBSOLETE',  { category: 'neutral', icon: '—' });
// ... all remaining domain-specific entries (work, workitem, milestone,
// outcome, group, sla, node, session, commitment, execution, agent, pattern)
```

The `lookupStatus`, `registerStatus`, `FALLBACK_DESCRIPTOR`, `stateCategoryStyles` functions are no longer exported from blocks-ui-core. Consumers import directly from `@casehubio/pages-ui-components/status-badge`.

Delete `packages/blocks-ui-core/src/status-badge/` directory entirely.
Delete `packages/blocks-ui-core/src/styles/category.ts`.

Update `packages/blocks-ui-core/src/index.ts`:
```diff
- export * from './commitment-pill/index.js';
- export * from './status-badge/index.js';
- export { stateCategoryStyles, type CategoryStyle } from './styles/category.js';
+ export * from './commitment-pill/index.js';
```

(types/index.ts barrel still exports domain types — `StateCategory` and `StatusDescriptor` are now type re-exports from pages via status.ts, which is acceptable for types.)

- [ ] **Step 2: Update status-badge consumer**

`components/session-list/src/session-list.ts`:
```diff
- import '@casehubio/blocks-ui-core/status-badge/status-badge.js';
+ import '@casehubio/pages-ui-components/status-badge';
```

Update template: `<status-badge` → `<pages-status-badge`.

Update `components/session-list/package.json` — add `@casehubio/pages-ui-components` dep.

- [ ] **Step 3: Delete split-workbench source and update 9 consumers**

Delete `components/split-workbench/src/` directory. Either remove the package from the workspace or mark deprecated.

Update each consumer (trust-workbench, work-item-workbench, session-workbench, case-explorer, diagram-workbench, contributor-workbench, orchestration-workbench, channel-activity, conversation-viewer):

For each:
1. `package.json`: replace `@casehubio/blocks-ui-split-workbench` dep with `@casehubio/pages-ui-components`
2. Source file: update import:
   ```diff
   - import '@casehubio/blocks-ui-split-workbench';
   + import '@casehubio/pages-ui-components/split-workbench';
   ```
3. Template: `<blocks-split-workbench` → `<pages-split-workbench`

The split-workbench API (selection-topic, title, storage-key attributes, list/detail/header slots) is unchanged — only the element name and import path change.

- [ ] **Step 4: Delete event-trail source and update consumer**

Delete `components/event-trail/src/` directory. Either remove the package or mark deprecated.

`components/audit-trail-viewer/`:
1. `package.json`: replace `@casehubio/blocks-ui-event-trail` dep with `@casehubio/pages-ui-components`
2. Source: update import:
   ```diff
   - import '@casehubio/blocks-ui-event-trail';
   + import '@casehubio/pages-ui-components/event-trail';
   ```
3. Template: `<blocks-event-trail` → `<pages-event-trail`

- [ ] **Step 5: Run blocks-ui build and tests**

Run: full blocks-ui build and test suite
Expected: PASS — all tests pass with new imports and element names

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add .
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: delete status-badge, split-workbench, event-trail — import from pages

Source deleted. All consumers updated to import directly from
@casehubio/pages-ui-components sub-paths. Element names updated to
pages- prefix. Domain status registrations stay in blocks-ui-core.

Refs casehubio/casehub-pages#389
Refs blocks-ui#<N>"
```

---

## Batch 4: pages internal re-export cleanup

After this batch: no package in pages re-exports another package's API in a way that confuses what lives where. Every import points to the actual owning package.

### Task 8: Remove confusing re-exports within pages

**Files (all in casehub-pages repo):**
- Modify: `packages/pages-component/src/events.ts` (delete file — entire contents are re-exports from pages-data)
- Modify: `packages/pages-component/src/index.ts` (remove events.ts re-export, replace with direct pages-data imports where needed)
- Modify: `packages/graph-renderer/src/index.ts` (remove `emitPagesEvent`/`PagesEventDetail` re-export from pages-data, remove `GraphModel`/`GraphNode`/`GraphEdge`/`NodeDecoration` re-export from graph-core)
- Modify: `packages/pages-ui/src/index.ts` (remove `renderComponent`/`RenderOptions` re-export from pages-component)
- Modify: `packages/pages-viz/src/index.ts` (remove type re-exports from pages-component and pages-ui-components)
- Modify: `packages/pages-runtime/src/index.ts` (remove type re-exports from pages-component)
- Modify: all consumers that imported the re-exported symbols from the wrong package

**Approach:** For each re-export removed, find all consumers that import the symbol from the re-exporting package and update them to import from the actual owning package. Use IntelliJ `ide_find_references` to locate all usages before removing.

- [ ] **Step 1: Remove pages-component/events.ts re-export**

This file re-exports `emitPagesEvent`, `onPagesEvent`, `PagesEventDetail` from `@casehubio/pages-data`. These functions live in pages-data — that's where consumers should import them.

1. Use `ide_find_references` on `emitPagesEvent` from `pages-component` to find all consumers
2. Update each consumer to import from `@casehubio/pages-data` instead
3. Delete `packages/pages-component/src/events.ts`
4. Remove the events barrel from `packages/pages-component/src/index.ts`

Note: the newly migrated `PagesSplitWorkbench` (Task 5) imports `onPagesEvent`/`emitPagesEvent` from `@casehubio/pages-component`. Update it to import from `@casehubio/pages-data` as part of this step.

- [ ] **Step 2: Remove graph-renderer re-exports**

`packages/graph-renderer/src/index.ts` re-exports:
- `emitPagesEvent` from `@casehubio/pages-data`
- `PagesEventDetail` type from `@casehubio/pages-data`
- `GraphModel`, `GraphNode`, `GraphEdge`, `NodeDecoration` types from `@casehubio/graph-core`

1. Use `ide_find_references` on each re-exported symbol from `graph-renderer`
2. Update consumers to import from the actual owning package (`pages-data` or `graph-core`)
3. Remove the re-export lines from `packages/graph-renderer/src/index.ts`

- [ ] **Step 3: Remove pages-ui re-exports**

`packages/pages-ui/src/index.ts` re-exports `renderComponent` and `RenderOptions` from `@casehubio/pages-component`.

1. Find references, update consumers, remove re-export lines

- [ ] **Step 4: Remove pages-viz type re-exports**

`packages/pages-viz/src/index.ts` re-exports types from `@casehubio/pages-component` (`FieldSchema`, `SchemaFormProps`, `EventTimelineLayout`) and from `@casehubio/pages-ui-components` (`PagesNumberInput`, `PagesDateInput`).

1. Find references, update consumers, remove re-export lines

- [ ] **Step 5: Remove pages-runtime type re-exports**

`packages/pages-runtime/src/index.ts` re-exports `DataReceiver`, `LayoutState`, `PanelEntry` from `@casehubio/pages-component`.

1. Find references, update consumers, remove re-export lines

- [ ] **Step 6: Typecheck and full build**

Run: `yarn typecheck`
Run: `yarn build`
Expected: PASS — all packages compile, no broken imports

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add .
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: remove internal re-exports — every import points to the actual owner

Removed re-export chains: pages-component/events.ts shim deleted,
graph-renderer no longer re-exports pages-data events or graph-core
types, pages-ui/pages-viz/pages-runtime no longer re-export pages-component
symbols. All consumers updated to import from the owning package.

Refs #389"
```

---

## Post-Batch: CLAUDE.md + ARC42STORIES.MD update

After all four batches:

- [ ] **Update CLAUDE.md package overview:** Add `pages-diagram-core` to the package list with its description. Update `pages-ui-components` description to include status-badge, split-workbench, event-trail.
- [ ] **Update ARC42STORIES.MD:** Reflect the new package in the architecture diagram (§9–10) — pages-diagram-core sits alongside graph-renderer in the framework tier.

---

## References

- [casehub-pages#389] — focal issue with full migration spec
- [blocks-ui packages/diagram-core/src/] — source of DiagramBaseMixin, toolbar, export, schema-registry, editors
- [blocks-ui packages/graph-stencil-case/src/persistence/github-backend.ts] — GitHubBackend source
- [blocks-ui packages/graph-stencil-case/src/adapter/yaml-editor.ts] — applyPropertyEdit (YAML primitives)
- [blocks-ui packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts] — applySwfPropertyEdit (duplicate)
- [blocks-ui packages/blocks-ui-core/src/status-badge/] — StatusBadge + registry
- [blocks-ui packages/blocks-ui-core/src/types/status.ts] — StatusDescriptor, registerStatus, lookupStatus
- [blocks-ui packages/blocks-ui-core/src/styles/category.ts] — stateCategoryStyles
- [blocks-ui components/split-workbench/src/split-workbench.ts] — SplitWorkbench source
- [blocks-ui components/event-trail/src/event-trail.ts] — EventTrail source
- [blocks-ui consumer scan] — casehub-diagram, swf-diagram, 9 workbench consumers, session-list, audit-trail-viewer
- [pages re-export scan] — pages-component/events.ts, graph-renderer, pages-ui, pages-viz, pages-runtime
- [PP-20260705-c7687d] — web-component-strategy protocol (Lit conventions, pages- prefix, sub-path exports)
- [PP-20260810-cdcc8f] — content-agnostic-workbench protocol
- [PP-20260826-507928] — graph-core-pure-data protocol
- [PP-20260826-6e9569] — per-instance-spi-registration protocol
- [docs/specs/diagram-editing-infrastructure/] — diagram editing infrastructure spec
