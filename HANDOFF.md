# Session Handover — casehub-pages

**Branch:** `issue-312-workspace-compositor`
**Issue:** #312 — workspace compositor: splits, tabs, accordion view
**Date:** 2026-08-13

## Last Session

Implemented all 7 plan tasks for the workspace compositor (types, persistence,
renderer, drag, transfer, activation integration, examples gallery) across 11
commits. The pure-state modules (Tasks 1–5) are solid with full test coverage
(741/741 tests pass). The integration layer (Task 6 — activation.ts, site.ts)
has an **architectural flaw** that must be fixed before this branch can merge:
the compositor path reimplements the floating workspace wiring instead of
composing above it, losing live drag preview and tab group dragging.

Also fixed a pre-existing dark mode test failure in `site.test.ts` (stale
theme class on dispose).

## Immediate Next Step

**Re-layer the compositor activation to compose above the existing engine,
not replace it.** The pure-state modules are done — the problem is entirely
in `activation.ts` lines 760–870 (the compositor path). See the architecture
section below for the exact fix.

## Architecture — What Went Wrong and How to Fix It

### The existing layers (working beautifully before this branch)

```
Layer 0: FloatingFrameEngine     — frame state (position, z-order, snap, drag)
Layer 1: wireFloatingWorkspace() — connects engine to backend, creates overlay,
                                    zone picker, detach, keyboard — all live
                                    interaction behaviors
Layer 2: activation.ts           — creates the workspace: centre content,
                                    overlay container, calls wire, toolbar,
                                    restores frames
Layer 3: dockview-backend        — Dockview rendering: frame chrome, titlebar
                                    drag, live preview, tab group operations
```

**The key invariant:** the overlay container (`data-floating-workspace-overlay`)
is the single DOM node that bridges all layers. It is:
- The `container` arg to `wireFloatingWorkspace()` (event dispatch target)
- The `el` arg to `backend.attach()` (dockview mount point)
- The reference for `clientWidth/Height` in toolbar and zone picker
- The parent for zone picker dropdown elements

### What the compositor SHOULD be

**Layer 4: WorkspaceCompositor** — manages regions and tabs. Each tab creates
a **complete, unmodified** instance of the Layer 2 activation flow. The
compositor never reaches into how frames drag, how tabs preview, how the
overlay works.

### What I actually built (the bug)

I wrote a `wireTab()` function inside the compositor activation path that
cherry-picked bits from Layers 1–3 — calling `wireFloatingWorkspace()`,
`backend.attach()`, `createOrganiserToolbar()` etc. piecemeal. This:
1. Lost the dockview live drag preview (dockview manages this internally)
2. Lost tab group dragging (dockview feature, not wired)
3. Created a separate overlay per tab instead of letting dockview manage the
   floating group lifecycle
4. Broke the "centre content underneath, frames overlaying" layout

### The correct fix

The compositor activation path should call the **exact same code** as the
non-compositor path, just scoped per tab. Concretely:

```
For each compositor tab:
  1. Create a container div (the tab's workspace)
  2. Call renderCentreInto(tabContainer)      — same function
  3. Create overlayContainer                   — same pattern
  4. wireFloatingWorkspace(backend, overlay)   — same call
  5. backend.attach(overlay, factory, extras)  — same call
  6. createOrganiserToolbar(engine, overlay, tabContainer, signal)
  7. createFrameKeyboardHandler(engine, overlay, signal)
  8. Restore frames from compositor state
```

The compositor adds ONLY:
- Tab bar UI (already built in compositor-renderer.ts)
- Tab switching (show/hide tab containers)
- Tab lifecycle callbacks (create/close/rename)
- Cross-tab frame transfer coordination
- Split/collapse region management

It does NOT:
- Create its own overlay containers with different structure
- Skip `backend.attach()` or call it with different arguments
- Reimplement frame chrome, drag handlers, or keyboard shortcuts
- Change the DOM hierarchy that dockview expects

### The non-compositor activation flow (reference — do not change this)

```
activation.ts lines 709-764 (the !useCompositor path):

1.  renderCentreInto(el)
2.  await createDockviewBackend()
3.  define defaultFactory (ContentFactory)
4.  create overlayContainer [position:absolute;inset:0;pointer-events:none]
5.  el.appendChild(overlayContainer)
6.  wireFloatingWorkspace(backend, overlayContainer, undefined,
      {detachEnabled, contentFactory, signal})
      → creates engine
      → wires backend callbacks (onFrameMove → engine.updatePosition, etc.)
      → injects animation CSS
      → creates detach handler + detach button
      → creates zone picker + zone picker button + ResizeObserver
      → returns WireHandle {engine, detachHandler, detachButton, zonePickerButton}
7.  collect extraButtons from handle (zonePickerButton, detachButton)
8.  backend.attach(overlayContainer, defaultFactory, {extraButtons})
      → creates DockviewComponent inside overlay
      → stores factory + extraButtons for renderFrame
9.  createOrganiserToolbar(engine, overlayContainer, el, signal)
10. el.insertBefore(toolbar, overlayContainer)
11. createFrameKeyboardHandler(engine, overlayContainer, signal)
12. For each frame: engine.createFrame(config)
      → engine adds to frames map
      → backend.renderFrame() called
         → dockview.addPanel(..., floating:{...})
         → subscribeOverlayEvents(key) [position/resize tracking]
         → injectFrameChrome(group, key) [close dot, pin btn, extra btns]
```

### DOM tree after activation (reference)

```
el [position:relative; display:flex; flex-direction:column]
  ├── centreContainer [data-floating-workspace-centre, flex:1]
  │     └── (rendered dashboard components)
  ├── toolbar [data-floating-workspace-toolbar, display:none initially]
  └── overlayContainer [data-floating-workspace-overlay,
  │     position:absolute; inset:0; pointer-events:none]
        └── dockview instance (floating groups render here)
```

The overlay uses `position:absolute; inset:0` to cover the entire `el`,
including the centre content. Individual frames re-enable `pointer-events`
on their own elements. This is what creates the "floating above" visual.

### The `compositor: true` prop flow

YAML → `component-desugar.ts` (must explicitly extract the field) →
`floatingWorkspace()` builder in `builders.ts` → `FloatingWorkspaceProps`
and `FloatingWorkspaceConfig` in `component-props.ts` / `types.ts` →
`activation.ts` reads `props.compositor`. All four places must be updated
for any new prop (see GE-20260813-674be0 — silent drop gotcha).

### Files to change (next session)

Only `activation.ts` needs rework (lines ~760-870). The pure-state modules
are correct and tested:
- `workspace-compositor.ts` — state management ✅
- `compositor-persistence.ts` — capture/restore ✅
- `compositor-renderer.ts` — tab bar UI ✅
- `compositor-drag.ts` — edge detection ✅
- `compositor-transfer.ts` — cross-tab frame transfer ✅
- `site.ts` — layout save/restore ✅
- `index.ts` — exports ✅

## References

| Artifact | Path |
|----------|------|
| Spec | `docs/specs/workspace-compositor/2026-08-13-workspace-compositor-design.md` |
| Plan | `plans/2026-08-13-workspace-compositor.md` (workspace) |
| Event contract | `docs/protocols/casehub/pages-event-contract.md` |
| Example | `examples/samples/Layout/Workspace Compositor.dash.yaml` |
