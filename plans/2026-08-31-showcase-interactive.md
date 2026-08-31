# Showcase Gallery Interactive Controls — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #352 — Showcase gallery: interactive controls for data component samples
**Issue group:** #352

**Goal:** Add interactive property controls (dropdowns, checkboxes) to the gallery config bar, driven by `samples.json` metadata, and investigate/fix the data-component stack overflow in pills/tabs.

**Architecture:** Extend `samples.json` entries with an optional `config` array declaring control metadata. Extend the existing `renderConfigBar` function to render dropdown/checkbox controls that immediately reload the sample on change. Investigate the pills/tabs stack overflow by creating a test sample.

**Tech Stack:** Vanilla JS, CSS, YAML

## Global Constraints

- Gallery code is vanilla JS (`examples/src/app.js`), not TypeScript — no compilation step
- Samples are YAML files loaded via `loadSite()` from `@casehubio/pages-runtime`
- The `renderConfigBar` function already handles URL text-input overrides with Apply button
- Config controls (enum/boolean) apply immediately on change, no Apply button needed

---

## Batch 1: Config UI and initial sample

### Task 1: Add config-driven interactive controls to the gallery

**Files:**
- Modify: `examples/src/app.js:264-314` — extend `renderConfigBar` for `config` array
- Modify: `examples/src/styles.css` — add styles for `select` and checkbox controls
- Modify: `examples/samples.json` — add `config` to Event Timeline entry
- Modify: `examples/samples/Charts/Event Timeline.dash.yaml` — restructure to single component with configurable layout

**Interfaces:**
- Consumes: `currentSample.config` array from `samples.json`, existing `propertyOverrides` map, `loadSampleInTarget()`
- Produces: extended `renderConfigBar` function, restructured Event Timeline sample

- [ ] **Step 1: Restructure Event Timeline sample to single configurable component**

The current Event Timeline shows all three layouts stacked. Change it to a single component with a `layout` property that can be toggled:

Replace `examples/samples/Charts/Event Timeline.dash.yaml` with:

```yaml
datasets:
  - uuid: events
    content: >-
      [
        ["1", "Case Started", "completed", "2026-01-15T09:00:00Z", "System", "lifecycle"],
        ["2", "Task Created", "completed", "2026-01-15T09:05:00Z", "Alice", "task"],
        ["3", "Processing", "active", "2026-01-15T09:30:00Z", "Bob", "task"],
        ["4", "Review Pending", "pending", "", "", "gate"]
      ]
    columns:
      - id: key
        type: LABEL
      - id: label
        type: LABEL
      - id: status
        type: LABEL
      - id: timestamp
        type: TEXT
      - id: actor
        type: LABEL
      - id: category
        type: LABEL

pages:
  - components:
      - type: event-timeline
        properties:
          layout: vertical
          lookup:
            uuid: events
```

- [ ] **Step 2: Add config metadata to samples.json for Event Timeline**

Find the Event Timeline entry in `examples/samples.json` and add a `config` array:

```json
{
  "name": "Event Timeline",
  "path": "Charts/Event Timeline.dash.yaml",
  "category": "Charts",
  "file": "Event Timeline.dash.yaml",
  "config": [
    {
      "prop": "layout",
      "label": "Layout",
      "type": "enum",
      "options": ["vertical", "horizontal", "compact"],
      "default": "vertical"
    }
  ]
}
```

- [ ] **Step 3: Extend renderConfigBar to handle config array**

In `examples/src/app.js`, modify `renderConfigBar` (currently at line 279). The function currently receives `urlProps` and `samplePath`. Change the signature to also accept the sample's `config` array, and render controls for each config entry:

Replace the `renderConfigBar` function with:

```javascript
function renderConfigBar(urlProps, samplePath, config) {
    const configBar = document.getElementById('config-bar');
    const urlKeys = Object.keys(urlProps);
    const hasConfig = config && config.length > 0;

    if (urlKeys.length === 0 && !hasConfig) {
        configBar.style.display = 'none';
        return;
    }
    configBar.style.display = 'flex';
    configBar.innerHTML = '';

    // Render config-driven controls (enum dropdowns, boolean checkboxes)
    if (hasConfig) {
        for (const entry of config) {
            const field = document.createElement('div');
            field.className = 'config-field';

            if (entry.type === 'enum') {
                const currentVal = propertyOverrides[entry.prop] || entry.default || entry.options[0];
                field.innerHTML = `<label>${entry.label}</label>`;
                const select = document.createElement('select');
                select.dataset.prop = entry.prop;
                select.className = 'config-select';
                for (const opt of entry.options) {
                    const option = document.createElement('option');
                    option.value = opt;
                    option.textContent = opt;
                    option.selected = opt === currentVal;
                    select.appendChild(option);
                }
                select.addEventListener('change', () => {
                    propertyOverrides[entry.prop] = select.value;
                    loadSampleInTarget(samplePath);
                });
                field.appendChild(select);
            } else if (entry.type === 'boolean') {
                const currentVal = propertyOverrides[entry.prop] !== undefined
                    ? propertyOverrides[entry.prop] === 'true'
                    : (entry.default || false);
                const label = document.createElement('label');
                label.className = 'config-checkbox-label';
                const checkbox = document.createElement('input');
                checkbox.type = 'checkbox';
                checkbox.checked = currentVal;
                checkbox.addEventListener('change', () => {
                    propertyOverrides[entry.prop] = checkbox.checked ? 'true' : 'false';
                    loadSampleInTarget(samplePath);
                });
                label.appendChild(checkbox);
                label.appendChild(document.createTextNode(` ${entry.label}`));
                field.appendChild(label);
            }

            configBar.appendChild(field);
        }
    }

    // Render URL text inputs (existing behavior)
    for (const key of urlKeys) {
        const defaultVal = urlProps[key];
        const override = propertyOverrides[key] || '';
        const field = document.createElement('div');
        field.className = 'config-field';
        field.innerHTML = `<label>${key}</label><input type="text" data-prop="${key}" value="${override.replace(/"/g, '&quot;')}" placeholder="${defaultVal.replace(/"/g, '&quot;')}" />`;
        configBar.appendChild(field);
    }

    // Apply button (only for URL text inputs)
    if (urlKeys.length > 0) {
        const applyBtn = document.createElement('button');
        applyBtn.className = 'config-apply';
        applyBtn.textContent = 'Apply';
        applyBtn.addEventListener('click', () => {
            for (const input of configBar.querySelectorAll('input[type="text"][data-prop]')) {
                const val = input.value.trim();
                if (val) propertyOverrides[input.dataset.prop] = val;
                else delete propertyOverrides[input.dataset.prop];
            }
            loadSampleInTarget(samplePath);
        });
        configBar.appendChild(applyBtn);
    }
}
```

- [ ] **Step 4: Update the renderConfigBar call site to pass config**

In `loadSampleInTarget` (line 331-332), update the call to pass the sample's config:

Change:
```javascript
renderConfigBar(urlProps, samplePath);
```
To:
```javascript
renderConfigBar(urlProps, samplePath, currentSample?.config);
```

- [ ] **Step 5: Add CSS styles for select and checkbox controls**

Add these styles after the existing `.config-bar .config-status` rule (after line 577) in `examples/src/styles.css`:

```css
.config-bar select,
.config-bar .config-select {
    padding: var(--pages-space-1-5) var(--pages-space-3);
    border: 1px solid var(--pages-border-default);
    border-radius: var(--pages-radius-md);
    font-size: var(--pages-font-size-sm);
    color: var(--pages-text-primary);
    background: var(--pages-surface-primary);
    cursor: pointer;
    transition: border-color var(--pages-duration-fast) var(--pages-ease-out);
}

.config-bar select:focus {
    outline: none;
    border-color: var(--pages-interactive);
    box-shadow: 0 0 0 2px var(--pages-focus-ring);
}

.config-bar .config-checkbox-label {
    display: flex;
    align-items: center;
    gap: var(--pages-space-2);
    cursor: pointer;
    font-weight: var(--pages-font-weight-medium);
    color: var(--pages-text-primary);
    padding: var(--pages-space-1-5) 0;
}

.config-bar .config-checkbox-label input[type="checkbox"] {
    width: 16px;
    height: 16px;
    cursor: pointer;
    accent-color: var(--pages-interactive);
}
```

- [ ] **Step 6: Verify in browser**

Run: `yarn workspace @casehubio/pages-examples run serve`

Open http://localhost:8080, navigate to Charts > Event Timeline. Verify:
1. A "Layout" dropdown appears in the config bar with vertical/horizontal/compact
2. Changing the dropdown reloads the sample with the selected layout
3. Other samples (without config) continue to work — no config bar shown unless URL properties exist

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/src/app.js examples/src/styles.css examples/samples.json examples/samples/Charts/Event\ Timeline.dash.yaml
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#352): interactive config controls in gallery — enum dropdowns and boolean checkboxes

Extend samples.json with optional config array for per-sample property
controls. renderConfigBar renders dropdowns and checkboxes that
immediately reload the sample on change. Event Timeline is the first
sample with a layout toggle (vertical/horizontal/compact).

Refs #352"
```

## Batch 2: Stack overflow investigation

### Task 2: Investigate and fix data-component stack overflow in pills/tabs

**Files:**
- Modify: depends on investigation — may be in `packages/pages-runtime/src/` or `packages/pages-component/src/`

**Interfaces:**
- Consumes: `loadSite()` from pages-runtime
- Produces: fix for the recursion path (if still present)

- [ ] **Step 1: Create a test sample with data-table inside tabs**

Create `examples/samples/Tables/Tabbed Table.dash.yaml`:

```yaml
datasets:
  - uuid: team
    content: >-
      [
        ["Alice", "Engineering", "Senior"],
        ["Bob", "Marketing", "Manager"],
        ["Charlie", "Engineering", "Staff"]
      ]
    columns:
      - id: name
        type: TEXT
      - id: department
        type: LABEL
      - id: role
        type: LABEL

pages:
  - tabs:
      - name: Team
        components:
          - type: data-table
            properties:
              lookup:
                uuid: team
      - name: Summary
        components:
          - html: "<div style='padding: 20px'>Summary view placeholder</div>"
```

- [ ] **Step 2: Add the sample to samples.json**

Add to the Tables category:
```json
{
  "name": "Tabbed Table",
  "path": "Tables/Tabbed Table.dash.yaml",
  "category": "Tables",
  "file": "Tabbed Table.dash.yaml"
}
```

- [ ] **Step 3: Test in browser**

Run gallery, navigate to Tables > Tabbed Table. Three outcomes:

**A. No stack overflow:** The bug was fixed by other runtime work. Add the sample to the gallery as a regression guard. Commit and close this gap.

**B. Stack overflow occurs:** Open browser DevTools, capture the call stack. Identify the recursive cycle in pages-runtime or pages-component. The recursion likely involves the tab renderer re-triggering data resolution, which re-renders the tab. Fix the recursion guard in the identified file.

**C. Component doesn't render but no crash:** Likely a data-request event not bubbling through the tab boundary. Fix the event propagation in the tab/pills container.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/samples/Tables/Tabbed\ Table.dash.yaml examples/samples.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#352): add tabbed table sample — regression guard for pills/tabs data components

Closes #352"
```

If a runtime fix was needed, include those files in the commit and adjust the message.

## References

- [2026-08-31-showcase-interactive-design.md] — design spec
- `examples/src/app.js:264-323` — existing renderConfigBar and propertyOverrides
- `examples/src/styles.css:506-577` — existing config bar styles
- `examples/samples.json` — sample catalog schema
- `examples/samples/Charts/Event Timeline.dash.yaml` — first config-enabled sample
- `767f391a` — scroll fix already landed (gap 3 done)
- [GitHub #352] — issue with three-gap analysis
