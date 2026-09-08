# Theme Designer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #415 — Theme designer component — browser-based theme generation and loading
**Issue group:** #415

**Goal:** Build a `<pages-theme-designer>` Lit web component that lets end users generate and persist custom themes using the existing pipeline.

**Architecture:** ThemeStorage SPI with localStorage default → designer component with simple/advanced modes → picker integration via customize button → server endpoint for file-backed persistence → showcase sample.

**Tech Stack:** Lit 3, TypeScript, Vitest (jsdom), Quarkus (server), OKLCH colour pipeline

## Global Constraints

- All new TS code in `packages/pages-ui-tokens/src/`
- Tests use Vitest with jsdom (follow `theme-picker.test.ts` patterns)
- Lit web components — no React
- Server code in `examples/server/` (Quarkus, Java)
- Existing test command: `vitest run` from package root
- No new runtime dependencies beyond `lit` (already present)

---

## Batch 1: Storage SPI + auto-restore

### Task 1: ThemeStorage SPI and LocalStorageThemeStorage

**Files:**
- Create: `packages/pages-ui-tokens/src/theme-storage.ts`
- Create: `packages/pages-ui-tokens/src/theme-storage.test.ts`

**Interfaces:**
- Consumes: `PresetConfig` from `./types.js`
- Produces: `ThemeStorage` interface, `LocalStorageThemeStorage` class, `ServerThemeStorage` class, `detectStorage()` function — used by Task 3 (`<pages-theme-designer>`)

- [ ] **Step 1: Write the failing tests**

```typescript
// theme-storage.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import type { PresetConfig } from './types.js';

const TEST_PRESET: PresetConfig = {
  $name: 'test-dark',
  $description: 'Test theme',
  pipeline: [
    { transform: 'dark-mode' },
    { transform: 'oklch-scale', params: { hues: { accent: 200, neutral: 220 }, chroma: 0.15, contrast: 0.5 } },
    { transform: 'semantic-map' },
    { transform: 'gamut-clamp' },
  ],
};

describe('LocalStorageThemeStorage', () => {
  let storage: import('./theme-storage.js').LocalStorageThemeStorage;

  beforeEach(async () => {
    localStorage.clear();
    const mod = await import('./theme-storage.js');
    storage = new mod.LocalStorageThemeStorage();
  });

  it('save and load round-trips a PresetConfig', async () => {
    await storage.save('test-dark', TEST_PRESET);
    const loaded = await storage.load('test-dark');
    expect(loaded).toEqual(TEST_PRESET);
  });

  it('load returns undefined for unknown name', async () => {
    const result = await storage.load('nonexistent');
    expect(result).toBeUndefined();
  });

  it('list returns saved theme names', async () => {
    await storage.save('alpha', TEST_PRESET);
    await storage.save('beta', { ...TEST_PRESET, $name: 'beta' });
    const names = await storage.list();
    expect(names).toContain('alpha');
    expect(names).toContain('beta');
    expect(names).toHaveLength(2);
  });

  it('list returns empty array when nothing saved', async () => {
    const names = await storage.list();
    expect(names).toEqual([]);
  });

  it('remove deletes a saved theme', async () => {
    await storage.save('to-remove', TEST_PRESET);
    await storage.remove('to-remove');
    const loaded = await storage.load('to-remove');
    expect(loaded).toBeUndefined();
    const names = await storage.list();
    expect(names).not.toContain('to-remove');
  });

  it('remove is a no-op for unknown name', async () => {
    await expect(storage.remove('nonexistent')).resolves.toBeUndefined();
  });

  it('does not collide with non-theme localStorage keys', async () => {
    localStorage.setItem('unrelated-key', 'value');
    await storage.save('my-theme', TEST_PRESET);
    const names = await storage.list();
    expect(names).toEqual(['my-theme']);
    expect(localStorage.getItem('unrelated-key')).toBe('value');
  });
});

describe('ServerThemeStorage', () => {
  let storage: import('./theme-storage.js').ServerThemeStorage;

  beforeEach(async () => {
    const mod = await import('./theme-storage.js');
    storage = new mod.ServerThemeStorage('/api/themes');
    vi.restoreAllMocks();
  });

  it('list calls GET and returns names', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValueOnce(
      new Response(JSON.stringify(['alpha', 'beta']), { status: 200 })
    );
    const names = await storage.list();
    expect(names).toEqual(['alpha', 'beta']);
  });

  it('load calls GET with name and returns config', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValueOnce(
      new Response(JSON.stringify(TEST_PRESET), { status: 200 })
    );
    const config = await storage.load('test-dark');
    expect(config).toEqual(TEST_PRESET);
  });

  it('load returns undefined on 404', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValueOnce(
      new Response('', { status: 404 })
    );
    const config = await storage.load('missing');
    expect(config).toBeUndefined();
  });

  it('save calls PUT with config body', async () => {
    const spy = vi.spyOn(globalThis, 'fetch').mockResolvedValueOnce(
      new Response('', { status: 204 })
    );
    await storage.save('test-dark', TEST_PRESET);
    expect(spy).toHaveBeenCalledWith('/api/themes/test-dark', expect.objectContaining({
      method: 'PUT',
      body: JSON.stringify(TEST_PRESET),
    }));
  });

  it('remove calls DELETE', async () => {
    const spy = vi.spyOn(globalThis, 'fetch').mockResolvedValueOnce(
      new Response('', { status: 204 })
    );
    await storage.remove('test-dark');
    expect(spy).toHaveBeenCalledWith('/api/themes/test-dark', expect.objectContaining({
      method: 'DELETE',
    }));
  });
});

describe('detectStorage', () => {
  it('returns ServerThemeStorage when server responds', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValueOnce(
      new Response(JSON.stringify([]), { status: 200 })
    );
    const mod = await import('./theme-storage.js');
    const storage = await mod.detectStorage();
    expect(storage).toBeInstanceOf(mod.ServerThemeStorage);
  });

  it('returns LocalStorageThemeStorage when server unreachable', async () => {
    vi.spyOn(globalThis, 'fetch').mockRejectedValueOnce(new Error('network'));
    const mod = await import('./theme-storage.js');
    const storage = await mod.detectStorage();
    expect(storage).toBeInstanceOf(mod.LocalStorageThemeStorage);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-storage.test.ts`
Expected: FAIL — module `./theme-storage.js` not found

- [ ] **Step 3: Implement ThemeStorage SPI**

```typescript
// theme-storage.ts
import type { PresetConfig } from './types.js';

export interface ThemeStorage {
  save(name: string, config: PresetConfig): Promise<void>;
  load(name: string): Promise<PresetConfig | undefined>;
  list(): Promise<string[]>;
  remove(name: string): Promise<void>;
}

const KEY_PREFIX = 'pages-theme:';

export class LocalStorageThemeStorage implements ThemeStorage {
  async save(name: string, config: PresetConfig): Promise<void> {
    localStorage.setItem(KEY_PREFIX + name, JSON.stringify(config));
  }

  async load(name: string): Promise<PresetConfig | undefined> {
    const raw = localStorage.getItem(KEY_PREFIX + name);
    if (!raw) return undefined;
    return JSON.parse(raw) as PresetConfig;
  }

  async list(): Promise<string[]> {
    const names: string[] = [];
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i);
      if (key?.startsWith(KEY_PREFIX)) {
        names.push(key.slice(KEY_PREFIX.length));
      }
    }
    return names;
  }

  async remove(name: string): Promise<void> {
    localStorage.removeItem(KEY_PREFIX + name);
  }
}

export class ServerThemeStorage implements ThemeStorage {
  constructor(private readonly baseUrl: string = '/api/themes') {}

  async save(name: string, config: PresetConfig): Promise<void> {
    await fetch(`${this.baseUrl}/${name}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(config),
    });
  }

  async load(name: string): Promise<PresetConfig | undefined> {
    const res = await fetch(`${this.baseUrl}/${name}`);
    if (!res.ok) return undefined;
    return (await res.json()) as PresetConfig;
  }

  async list(): Promise<string[]> {
    const res = await fetch(this.baseUrl);
    return (await res.json()) as string[];
  }

  async remove(name: string): Promise<void> {
    await fetch(`${this.baseUrl}/${name}`, { method: 'DELETE' });
  }
}

export async function detectStorage(baseUrl = '/api/themes'): Promise<ThemeStorage> {
  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 2000);
    const res = await fetch(baseUrl, { signal: controller.signal });
    clearTimeout(timeout);
    if (res.ok) return new ServerThemeStorage(baseUrl);
  } catch { /* server unavailable */ }
  return new LocalStorageThemeStorage();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-storage.test.ts`
Expected: all PASS

- [ ] **Step 5: Add exports to index.ts and build.ts**

Add to `packages/pages-ui-tokens/src/index.ts`:
```typescript
export { type ThemeStorage, LocalStorageThemeStorage, ServerThemeStorage, detectStorage } from './theme-storage.js';
```

Add to `packages/pages-ui-tokens/src/build.ts`:
```typescript
export { type ThemeStorage, LocalStorageThemeStorage, ServerThemeStorage, detectStorage } from './theme-storage.js';
```

- [ ] **Step 6: Run full test suite**

Run: `npx vitest run --project pages-ui-tokens`
Expected: all existing tests still pass

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui-tokens/src/theme-storage.ts packages/pages-ui-tokens/src/theme-storage.test.ts packages/pages-ui-tokens/src/index.ts packages/pages-ui-tokens/src/build.ts
git commit -m "feat(ui-tokens): ThemeStorage SPI with localStorage and server backends Refs #415"
```

### Task 2: Auto-restore custom themes on page load

**Files:**
- Modify: `packages/pages-ui-tokens/src/init.ts`

**Interfaces:**
- Consumes: `LocalStorageThemeStorage` from Task 1, `runPipeline` from `./pipeline.js`, `generateCSS` from `./output.js`, `registerTheme` from `./runtime.js`
- Produces: on module import, any custom themes in localStorage are registered and available in `listThemes()`

- [ ] **Step 1: Write the failing test**

Add a test to `packages/pages-ui-tokens/src/theme-storage.test.ts`:

```typescript
describe('auto-restore on init', () => {
  it('custom themes in localStorage are registered after init import', async () => {
    const { initPresets } = await import('./transforms/index.js');
    const { runPipeline } = await import('./pipeline.js');
    const { generateCSS } = await import('./output.js');
    const { _resetThemeRegistry, listThemes, registerTheme } = await import('./runtime.js');

    _resetThemeRegistry();
    initPresets();

    const preset: PresetConfig = {
      $name: 'custom-dark',
      pipeline: [
        { transform: 'dark-mode' },
        { transform: 'oklch-scale', params: { hues: { accent: 200, neutral: 220 }, chroma: 0.12, contrast: 0.5 } },
        { transform: 'semantic-map' },
        { transform: 'gamut-clamp' },
      ],
    };
    localStorage.setItem('pages-theme:custom-dark', JSON.stringify(preset));

    // Manually invoke the restore function
    const { restoreCustomThemes } = await import('./theme-storage.js');
    await restoreCustomThemes();

    expect(listThemes()).toContain('custom-dark');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-storage.test.ts -t "auto-restore"`
Expected: FAIL — `restoreCustomThemes` not exported

- [ ] **Step 3: Implement restoreCustomThemes and wire into init.ts**

Add to `theme-storage.ts`:

```typescript
import { registerTheme } from './runtime.js';

export async function restoreCustomThemes(): Promise<void> {
  const storage = new LocalStorageThemeStorage();
  const names = await storage.list();
  // Dynamic import to avoid circular dependency — pipeline imports are heavy
  const [{ initPresets }, { runPipeline }, { generateCSS, generateDensityCSS }] = await Promise.all([
    import('./transforms/index.js'),
    import('./pipeline.js'),
    import('./output.js'),
  ]);
  initPresets();
  const densityCSS = generateDensityCSS();
  for (const name of names) {
    const config = await storage.load(name);
    if (!config) continue;
    try {
      const tokens = runPipeline(config);
      const css = generateCSS(tokens, config.$name) + '\n\n' + densityCSS;
      registerTheme(config.$name, css);
    } catch {
      // Invalid preset — skip silently, don't block app startup
    }
  }
}
```

Add to end of `init.ts`:

```typescript
import { restoreCustomThemes } from './theme-storage.js';
restoreCustomThemes();
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-storage.test.ts`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-tokens/src/theme-storage.ts packages/pages-ui-tokens/src/theme-storage.test.ts packages/pages-ui-tokens/src/init.ts
git commit -m "feat(ui-tokens): auto-restore custom themes from localStorage on init Refs #415"
```

## Batch 2: Theme designer component — simple mode

### Task 3: `<pages-theme-designer>` core component with simple mode

**Files:**
- Create: `packages/pages-ui-tokens/src/theme-designer.ts`
- Create: `packages/pages-ui-tokens/src/theme-designer.test.ts`
- Modify: `packages/pages-ui-tokens/src/index.ts`

**Interfaces:**
- Consumes: `ThemeStorage`, `LocalStorageThemeStorage`, `detectStorage` from Task 1; `generateThemeCSS`, `ThemeConfig` from `./themes.js`; `registerTheme` from `./runtime.js`; `PresetConfig` from `./types.js`; `initPresets` from `./transforms/index.js`; `runPipeline` from `./pipeline.js`; `generateCSS`, `generateDensityCSS` from `./output.js`
- Produces: `PagesThemeDesignerElement` custom element (`<pages-theme-designer>`), events `pages-theme-created`, `pages-theme-updated`, `pages-theme-deleted`, `pages-designer-closed`

- [ ] **Step 1: Write the failing tests**

```typescript
// theme-designer.test.ts
import { describe, it, expect, beforeEach, beforeAll, vi } from 'vitest';
import { registerTheme, listThemes, _resetThemeRegistry, _resetAppliedThemes, applyTheme } from './runtime.js';
import { initPresets } from './transforms/index.js';
import type { PresetConfig } from './types.js';

beforeAll(async () => {
  initPresets();
  registerTheme('default-light', '.pages-theme-default-light {}');
  registerTheme('default-dark', '.pages-theme-default-dark {}');
  applyTheme('default-dark');
  await import('./theme-designer.js');
});

describe('pages-theme-designer', () => {
  let designer: HTMLElement & { open: boolean; _accentHue: number; _neutralHue: number; _chroma: number; _contrast: number; _themeName: string; updateComplete: Promise<boolean> };

  beforeEach(async () => {
    document.body.innerHTML = '';
    localStorage.clear();
    designer = document.createElement('pages-theme-designer') as any;
    document.body.appendChild(designer);
    await designer.updateComplete;
  });

  it('is a defined custom element', () => {
    expect(customElements.get('pages-theme-designer')).toBeDefined();
  });

  it('renders a shadow root', () => {
    expect(designer.shadowRoot).not.toBeNull();
  });

  it('is hidden when open is false', () => {
    designer.open = false;
    expect(designer.shadowRoot?.querySelector('.designer-overlay')).toBeNull();
  });

  it('shows overlay when open is true', async () => {
    designer.open = true;
    await designer.updateComplete;
    expect(designer.shadowRoot?.querySelector('.designer-overlay')).not.toBeNull();
  });

  it('renders hue sliders for accent and neutral', async () => {
    designer.open = true;
    await designer.updateComplete;
    const sliders = designer.shadowRoot?.querySelectorAll('input[type="range"]');
    expect(sliders?.length).toBeGreaterThanOrEqual(4); // accent hue, neutral hue, chroma, contrast
  });

  it('renders colour preview swatches', async () => {
    designer.open = true;
    await designer.updateComplete;
    const swatches = designer.shadowRoot?.querySelectorAll('.swatch-row');
    expect(swatches?.length).toBeGreaterThanOrEqual(2); // at least accent + neutral
  });

  it('renders a name input', async () => {
    designer.open = true;
    await designer.updateComplete;
    const nameInput = designer.shadowRoot?.querySelector('input[type="text"]');
    expect(nameInput).not.toBeNull();
  });

  it('renders save, load, import, export, close buttons', async () => {
    designer.open = true;
    await designer.updateComplete;
    const buttons = Array.from(designer.shadowRoot?.querySelectorAll('button') ?? []);
    const labels = buttons.map(b => b.textContent?.trim().toLowerCase());
    expect(labels).toContain('save');
    expect(labels).toContain('close');
  });

  it('dispatches pages-designer-closed on close button click', async () => {
    designer.open = true;
    await designer.updateComplete;
    const handler = vi.fn();
    designer.addEventListener('pages-designer-closed', handler);
    const closeBtn = Array.from(designer.shadowRoot?.querySelectorAll('button') ?? [])
      .find(b => b.textContent?.trim().toLowerCase() === 'close');
    closeBtn?.click();
    expect(handler).toHaveBeenCalledOnce();
  });

  it('dispatches pages-theme-created on save', async () => {
    designer.open = true;
    designer._themeName = 'my-custom';
    designer._accentHue = 200;
    designer._neutralHue = 220;
    designer._chroma = 0.15;
    designer._contrast = 0.5;
    await designer.updateComplete;

    const handler = vi.fn();
    designer.addEventListener('pages-theme-created', handler);

    const saveBtn = Array.from(designer.shadowRoot?.querySelectorAll('button') ?? [])
      .find(b => b.textContent?.trim().toLowerCase() === 'save');
    saveBtn?.click();
    await designer.updateComplete;

    expect(handler).toHaveBeenCalledOnce();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.name).toMatch(/my-custom/);
  });

  it('registers theme after save so it appears in listThemes', async () => {
    designer.open = true;
    designer._themeName = 'brand-new';
    designer._accentHue = 180;
    await designer.updateComplete;

    const saveBtn = Array.from(designer.shadowRoot?.querySelectorAll('button') ?? [])
      .find(b => b.textContent?.trim().toLowerCase() === 'save');
    saveBtn?.click();
    await designer.updateComplete;

    const themes = listThemes();
    expect(themes.some(t => t.includes('brand-new'))).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-designer.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement `<pages-theme-designer>` component**

Create `packages/pages-ui-tokens/src/theme-designer.ts`. This is a large component — key sections:

1. **Lit element with properties**: `open`, `storage`, `target`, `preset`, and internal state for `_accentHue`, `_neutralHue`, `_chroma`, `_contrast`, `_themeName`, `_advancedMode`, `_previewCSS`
2. **Simple mode render**: four range sliders (accent hue 0-360, neutral hue 0-360, chroma 0-0.4 step 0.01, contrast 0-1 step 0.01), name input, toolbar buttons
3. **Preview generation**: on any slider change, build a `PresetConfig` with `light-mode`/`dark-mode` + `oklch-scale` + `semantic-map` + `gamut-clamp`, run `runPipeline`, generate CSS, inject into preview panel
4. **Colour swatches**: for each semantic group (accent, neutral, success, warning, danger, info), render 12 colour boxes from the generated tokens
5. **Save flow**: generate both light and dark presets, `registerTheme()` for each, `storage.save()`, dispatch event
6. **Load flow**: `storage.list()` populates dropdown, selecting loads `PresetConfig` and extracts params back to sliders
7. **Import/Export**: file input → JSON parse → load config; download button → JSON blob → file download
8. **Modal overlay**: fixed position, backdrop, close on button or Escape key

The component should:
- Use `generateScale()` from `./colours.js` for swatch preview (lightweight, no full pipeline)
- Use `runPipeline()` for the actual save (full pipeline with all transforms)
- Apply preview CSS to `target` via a `<style>` element for live preview
- Auto-detect storage via `detectStorage()` in `connectedCallback` (unless `storage` property is set)

```typescript
// theme-designer.ts
import { LitElement, html, css, nothing } from 'lit';
import { registerTheme } from './runtime.js';
import { generateScale } from './colours.js';
import { initPresets } from './transforms/index.js';
import { runPipeline } from './pipeline.js';
import { generateCSS, generateDensityCSS } from './output.js';
import type { ThemeStorage } from './theme-storage.js';
import { LocalStorageThemeStorage, detectStorage } from './theme-storage.js';
import type { PresetConfig } from './types.js';

const SEMANTIC_HUES: Record<string, (accentHue: number, neutralHue: number) => number> = {
  accent: (a) => a,
  neutral: (_, n) => n,
  success: () => 145,
  warning: () => 55,
  danger: () => 25,
  info: () => 210,
};

export class PagesThemeDesignerElement extends LitElement {
  // ... full implementation with styles, properties, render methods
  // See Step 3 body below for the complete implementation
}

customElements.define('pages-theme-designer', PagesThemeDesignerElement);
```

The full implementation will include:
- `static styles` with modal overlay, slider, swatch grid, toolbar, and preview panel CSS
- `static properties` for all reactive props and internal state
- `connectedCallback` that calls `detectStorage()` and `_loadSavedList()`
- `_buildPresetConfig(mode: 'light' | 'dark')` that constructs a `PresetConfig` from current slider values
- `_generatePreview()` that uses `generateScale()` for fast swatch updates
- `_onSave()` that builds both light/dark presets, runs pipeline, registers, persists, dispatches event
- `_onLoad(name: string)` that loads a preset and extracts hue/chroma/contrast back to sliders
- `_onImport()` / `_onExport()` for file I/O
- `_onClose()` that sets `open = false` and dispatches `pages-designer-closed`
- `_onKeyDown(e: KeyboardEvent)` for Escape to close
- `render()` returning the modal overlay with conditional simple/advanced views

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-designer.test.ts`
Expected: all PASS

- [ ] **Step 5: Add export to index.ts**

Add to `packages/pages-ui-tokens/src/index.ts`:
```typescript
export { PagesThemeDesignerElement } from './theme-designer.js';
```

- [ ] **Step 6: Run full test suite**

Run: `npx vitest run --project pages-ui-tokens`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui-tokens/src/theme-designer.ts packages/pages-ui-tokens/src/theme-designer.test.ts packages/pages-ui-tokens/src/index.ts
git commit -m "feat(ui-tokens): <pages-theme-designer> component with simple mode Refs #415"
```

## Batch 3: Picker integration + advanced mode

### Task 4: Add "customize" button to `<pages-theme-picker>`

**Files:**
- Modify: `packages/pages-ui-tokens/src/theme-picker.ts`
- Modify: `packages/pages-ui-tokens/src/theme-picker.test.ts`

**Interfaces:**
- Consumes: `PagesThemeDesignerElement` from Task 3 (lazy-imports `./theme-designer.js`)
- Produces: "customize" button in the picker that opens a `<pages-theme-designer>` modal

- [ ] **Step 1: Write the failing tests**

Add to `theme-picker.test.ts`:

```typescript
describe('customize button', () => {
  let picker: HTMLElement;

  beforeEach(async () => {
    _resetThemeRegistry();
    _resetAppliedThemes();
    document.body.innerHTML = '';
    registerTheme('default-light', '.pages-theme-default-light {}');
    registerTheme('default-dark', '.pages-theme-default-dark {}');
    applyTheme('default-dark');
    picker = document.createElement('pages-theme-picker');
    document.body.appendChild(picker);
    await (picker as any).updateComplete;
  });

  it('renders a customize button in full mode', () => {
    const btn = picker.shadowRoot?.querySelector('.customize-btn');
    expect(btn).not.toBeNull();
  });

  it('customize button has aria-label', () => {
    const btn = picker.shadowRoot?.querySelector('.customize-btn');
    expect(btn?.getAttribute('aria-label')).toBe('Customize theme');
  });

  it('clicking customize creates and opens a theme designer', async () => {
    const btn = picker.shadowRoot?.querySelector('.customize-btn') as HTMLButtonElement;
    btn.click();
    await (picker as any).updateComplete;
    const designer = document.querySelector('pages-theme-designer');
    expect(designer).not.toBeNull();
    expect((designer as any).open).toBe(true);
  });
});

describe('customize button in compact mode', () => {
  let picker: HTMLElement;

  beforeEach(async () => {
    _resetThemeRegistry();
    _resetAppliedThemes();
    document.body.innerHTML = '';
    registerTheme('default-light', '.pages-theme-default-light {}');
    registerTheme('default-dark', '.pages-theme-default-dark {}');
    applyTheme('default-dark');
    picker = document.createElement('pages-theme-picker');
    (picker as any).compact = true;
    document.body.appendChild(picker);
    await (picker as any).updateComplete;
  });

  it('renders a customize button in compact mode popover', () => {
    const popover = picker.shadowRoot?.querySelector('[popover]');
    const btn = popover?.querySelector('.customize-btn');
    expect(btn).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-picker.test.ts -t "customize"`
Expected: FAIL — `.customize-btn` not found

- [ ] **Step 3: Add customize button to theme-picker.ts**

Add CSS for the customize button to `static styles`:
```css
.customize-btn {
  background: var(--pages-surface-secondary, #222);
  color: var(--pages-text-secondary, #ccc);
  border: 1px solid var(--pages-border-default, #444);
  border-radius: var(--pages-radius-sm, 4px);
  padding: 4px 8px;
  cursor: pointer;
  font: inherit;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}
.customize-btn:hover { background: var(--pages-surface-hover, #333); }
```

Add a `_openDesigner()` method to `PagesThemePickerElement`:
```typescript
private _openDesigner(): void {
  let designer = document.querySelector('pages-theme-designer') as any;
  if (!designer) {
    import('./theme-designer.js');
    designer = document.createElement('pages-theme-designer');
    designer.target = this.target;
    document.body.appendChild(designer);
  }
  designer.open = true;
}
```

Add the button to both `render()` (full mode) and `_renderCompact()` (in popover):
```typescript
// In render() — after the mode toggle div:
<button class="customize-btn" aria-label="Customize theme" @click=${() => { this._openDesigner(); }}>
  <svg viewBox="0 0 16 16" width="14" height="14" fill="none" aria-hidden="true">
    <path d="M13.5 2.5l-1-1-9 9-.5 2 2-.5 9-9z" stroke="currentColor" stroke-width="1.2" stroke-linejoin="round"/>
  </svg>
</button>

// In _renderCompact() — inside the popover, after the mode toggle:
<button class="customize-btn" aria-label="Customize theme" @click=${() => { this._openDesigner(); }}>
  Customize...
</button>
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-picker.test.ts`
Expected: all PASS (existing + new)

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-tokens/src/theme-picker.ts packages/pages-ui-tokens/src/theme-picker.test.ts
git commit -m "feat(ui-tokens): add customize button to theme picker Refs #415"
```

### Task 5: Advanced mode — pipeline editor

**Files:**
- Modify: `packages/pages-ui-tokens/src/theme-designer.ts`
- Modify: `packages/pages-ui-tokens/src/theme-designer.test.ts`

**Interfaces:**
- Consumes: `listTransforms()` from `./registry.js`, `PresetConfig` / `TransformDef` from `./types.js`
- Produces: advanced mode UI in the designer — toggle between simple/advanced, pipeline stage cards with param editing

- [ ] **Step 1: Write the failing tests**

Add to `theme-designer.test.ts`:

```typescript
describe('advanced mode', () => {
  let designer: any;

  beforeEach(async () => {
    document.body.innerHTML = '';
    localStorage.clear();
    designer = document.createElement('pages-theme-designer');
    designer.open = true;
    document.body.appendChild(designer);
    await designer.updateComplete;
  });

  it('has an advanced mode toggle', () => {
    const toggle = designer.shadowRoot?.querySelector('.advanced-toggle');
    expect(toggle).not.toBeNull();
  });

  it('shows simple mode controls by default', () => {
    const simplePanel = designer.shadowRoot?.querySelector('.simple-controls');
    expect(simplePanel).not.toBeNull();
    const pipelinePanel = designer.shadowRoot?.querySelector('.pipeline-editor');
    expect(pipelinePanel).toBeNull();
  });

  it('switches to pipeline editor when toggled', async () => {
    designer._advancedMode = true;
    await designer.updateComplete;
    const pipelinePanel = designer.shadowRoot?.querySelector('.pipeline-editor');
    expect(pipelinePanel).not.toBeNull();
    const simplePanel = designer.shadowRoot?.querySelector('.simple-controls');
    expect(simplePanel).toBeNull();
  });

  it('pipeline editor shows transform stages', async () => {
    designer._advancedMode = true;
    await designer.updateComplete;
    const stages = designer.shadowRoot?.querySelectorAll('.pipeline-stage');
    expect(stages?.length).toBeGreaterThan(0);
  });

  it('has add-stage button listing available transforms', async () => {
    designer._advancedMode = true;
    await designer.updateComplete;
    const addBtn = designer.shadowRoot?.querySelector('.add-stage-btn');
    expect(addBtn).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-designer.test.ts -t "advanced"`
Expected: FAIL — `.advanced-toggle` not found

- [ ] **Step 3: Implement advanced mode**

Add to the designer component:
- `_advancedMode` boolean state property
- Toggle button/checkbox at the top of the editor
- When `_advancedMode` is true, render a `_renderPipelineEditor()` method that shows:
  - Each `TransformDef` in the current `_pipeline` array as a collapsible `.pipeline-stage` card
  - Stage name, remove button, and param inputs (JSON key-value editor)
  - An `.add-stage-btn` dropdown populated from `listTransforms()`
  - Reorder buttons (move up/move down) on each stage
- When pipeline changes, run `runPipeline()` and update the preview
- When switching from advanced to simple, attempt to extract hue/chroma/contrast from the pipeline's `oklch-scale` params

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-ui-tokens/src/theme-designer.test.ts`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-tokens/src/theme-designer.ts packages/pages-ui-tokens/src/theme-designer.test.ts
git commit -m "feat(ui-tokens): advanced pipeline editor mode in theme designer Refs #415"
```

## Batch 4: Server endpoint + showcase

### Task 6: ThemeResource Quarkus endpoint

**Files:**
- Create: `examples/server/src/main/java/io/casehub/pages/examples/ThemeResource.java`
- Create: `examples/server/src/test/java/io/casehub/pages/examples/ThemeResourceTest.java`
- Modify: `examples/server/src/main/resources/application.properties`

**Interfaces:**
- Consumes: nothing from TS — standalone REST resource
- Produces: REST API at `/api/themes` with CRUD for theme JSON files

- [ ] **Step 1: Write the failing test**

```java
// ThemeResourceTest.java
package io.casehub.pages.examples;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ThemeResourceTest {

    @Test
    void listReturnsEmptyArrayWhenNoThemes() {
        given()
            .when().get("/api/themes")
            .then()
            .statusCode(200)
            .body("$", hasSize(0));
    }

    @Test
    void putAndGetRoundTrip() {
        String config = "{\"$name\":\"test-dark\",\"pipeline\":[{\"transform\":\"dark-mode\"}]}";
        given()
            .contentType(ContentType.JSON)
            .body(config)
            .when().put("/api/themes/test-dark")
            .then()
            .statusCode(204);

        given()
            .when().get("/api/themes/test-dark")
            .then()
            .statusCode(200)
            .body("$name", equalTo("test-dark"));
    }

    @Test
    void getReturns404ForUnknownTheme() {
        given()
            .when().get("/api/themes/nonexistent")
            .then()
            .statusCode(404);
    }

    @Test
    void deleteRemovesTheme() {
        String config = "{\"$name\":\"to-delete\",\"pipeline\":[]}";
        given()
            .contentType(ContentType.JSON)
            .body(config)
            .when().put("/api/themes/to-delete");

        given()
            .when().delete("/api/themes/to-delete")
            .then()
            .statusCode(204);

        given()
            .when().get("/api/themes/to-delete")
            .then()
            .statusCode(404);
    }

    @Test
    void listIncludesSavedThemes() {
        String config = "{\"$name\":\"listed\",\"pipeline\":[]}";
        given()
            .contentType(ContentType.JSON)
            .body(config)
            .when().put("/api/themes/listed");

        given()
            .when().get("/api/themes")
            .then()
            .statusCode(200)
            .body("$", hasItem("listed"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl examples/server -Dtest=ThemeResourceTest`
Expected: FAIL — ThemeResource class not found / 404 on all endpoints

- [ ] **Step 3: Implement ThemeResource**

```java
// ThemeResource.java
package io.casehub.pages.examples;

import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.stream.Stream;

@Path("/api/themes")
@Produces(MediaType.APPLICATION_JSON)
public class ThemeResource {

    @ConfigProperty(name = "casehub.pages.themes.directory",
                    defaultValue = "${user.home}/.casehub/themes")
    String themesDirectory;

    private Path themesDir() {
        return Path.of(themesDirectory);
    }

    @GET
    public List<String> list() throws IOException {
        Path dir = themesDir();
        if (!Files.isDirectory(dir)) return List.of();
        try (Stream<Path> files = Files.list(dir)) {
            return files
                .filter(p -> p.toString().endsWith(".json"))
                .map(p -> p.getFileName().toString().replace(".json", ""))
                .sorted()
                .toList();
        }
    }

    @GET
    @Path("/{name}")
    public Response get(@PathParam("name") String name) throws IOException {
        Path file = themesDir().resolve(name + ".json");
        if (!Files.exists(file)) return Response.status(404).build();
        String json = Files.readString(file);
        return Response.ok(json).build();
    }

    @PUT
    @Path("/{name}")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response put(@PathParam("name") String name, String body) throws IOException {
        Path dir = themesDir();
        Files.createDirectories(dir);
        Files.writeString(dir.resolve(name + ".json"), body);
        return Response.noContent().build();
    }

    @DELETE
    @Path("/{name}")
    public Response delete(@PathParam("name") String name) throws IOException {
        Path file = themesDir().resolve(name + ".json");
        Files.deleteIfExists(file);
        return Response.noContent().build();
    }
}
```

Add to `application.properties`:
```properties
casehub.pages.themes.directory=${java.io.tmpdir}/casehub-themes
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl examples/server -Dtest=ThemeResourceTest`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add examples/server/src/main/java/io/casehub/pages/examples/ThemeResource.java examples/server/src/test/java/io/casehub/pages/examples/ThemeResourceTest.java examples/server/src/main/resources/application.properties
git commit -m "feat(server): ThemeResource CRUD endpoint for theme persistence Refs #415"
```

### Task 7: Showcase sample + visual verification

**Files:**
- Create: `examples/samples/Theming/Theme Designer.dash.yaml`

**Interfaces:**
- Consumes: `<pages-theme-designer>` from Task 3, `<pages-theme-picker>` from Task 4
- Produces: showcase gallery entry for the theme designer

- [ ] **Step 1: Create the showcase sample**

```yaml
# Theme Designer.dash.yaml
title: Theme Designer
description: Create and customize themes with live preview. Pick brand colours, adjust chroma and contrast, or dive into the advanced pipeline editor for full control.
displayers:
  - type: html
    html: |
      <div style="padding: 24px;">
        <h2 style="margin: 0 0 16px;">Theme Designer</h2>
        <p style="margin: 0 0 16px; color: var(--pages-text-secondary);">
          Click the button below to open the theme designer. Changes preview live
          and saved themes appear in the theme picker above.
        </p>
        <button id="open-designer"
          style="background: var(--pages-interactive); color: var(--pages-surface-primary);
                 border: none; border-radius: var(--pages-radius-sm); padding: 8px 16px;
                 cursor: pointer; font: inherit;">
          Open Theme Designer
        </button>
        <pages-theme-designer id="demo-designer"></pages-theme-designer>
      </div>
    scripts:
      - Theme Designer.ts
```

Create companion `examples/samples/Theming/Theme Designer.ts`:
```typescript
const btn = document.getElementById('open-designer');
const designer = document.getElementById('demo-designer') as any;
btn?.addEventListener('click', () => { designer.open = true; });
designer?.addEventListener('pages-theme-created', (e: CustomEvent) => {
  console.log('Theme created:', e.detail.name);
});
```

- [ ] **Step 2: Start dev server and verify**

Run: `npm run dev` from `examples/`
Navigate to Theming → Theme Designer in the gallery.
Verify:
- Button opens the designer modal
- Sliders change colour swatches in real time
- Save creates a theme that appears in the header's theme picker
- Close dismisses the modal
- Reload page → saved theme still in picker

- [ ] **Step 3: Commit**

```bash
git add "examples/samples/Theming/Theme Designer.dash.yaml" "examples/samples/Theming/Theme Designer.ts"
git commit -m "feat(examples): Theme Designer showcase sample Refs #415"
```

## References

- [2026-09-07-theme-designer-design.md] — design spec this plan implements
- `packages/pages-ui-tokens/src/themes.ts` — existing generateThemeCSS, ThemeConfig
- `packages/pages-ui-tokens/src/pipeline.ts` — runPipeline, buildInitialTokenMap
- `packages/pages-ui-tokens/src/output.ts` — generateCSS, generateDTCG
- `packages/pages-ui-tokens/src/transforms/index.ts` — preset definitions
- `packages/pages-ui-tokens/src/theme-picker.ts` — existing picker component
- `packages/pages-ui-tokens/src/theme-picker.test.ts` — test patterns
- `packages/pages-ui-tokens/src/runtime.ts` — registerTheme, applyTheme, listThemes
- `packages/pages-ui-tokens/src/colours.ts` — generateScale
- `examples/server/src/main/java/io/casehub/pages/examples/DemoResource.java` — server pattern
- [GitHub #415] — focal issue
