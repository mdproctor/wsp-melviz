# Tabbed Viewer and Modal Slides

**Branch:** issue-334-dsl-and-scenario
**Issue:** casehubio/casehub-pages#365
**Date:** 2026-08-25

## Summary

Extend the scenario YAML viewer into a tabbed panel (Source + Guide) and
add a full-screen modal slide deck mode for presentation-format markdown.
Add type-specific icons to the scenario outline.

## YAML Syntax

The `show-markdown` action gains a `display` property:

```yaml
# Panel mode (default) — fire-and-forget into Guide tab
- show-markdown:
    content: |
      ## Introduction
      Tutorial text here.

# Modal mode — blocking full-screen slide
- show-markdown:
    display: modal
    content: |
      ## Welcome
      Full-screen presentation slide.

# File reference with section extraction
- show-markdown:
    file: tutorial/guide.md
    section: "REST API"

# Modal with file reference
- show-markdown:
    display: modal
    file: tutorial/guide.md
    section: "Overview"
```

`display` defaults to `panel`. Consecutive `display: modal` steps auto-group
into a slide deck.

## Component Architecture

### Tabbed Viewer

`PagesScenarioYamlViewer` gains a tab bar and Guide tab. No new components.

```
PagesScenarioYamlViewer
├── Tab bar: [Source] [Guide]
├── Source tab: existing YAML rendering (unchanged)
├── Guide tab: markdown rendering (from PagesScenarioNarrative)
└── Drag, dock, undock, detach, resize — carry both tabs
```

**Guide tab behavior:**
- Listens for `scenario-narrative` events on `eventTarget` (reactive binding
  via `willUpdate`, same pattern as the narrative component fix)
- Content persists until the next `show-markdown` replaces it
- No auto-open, no auto-switch — user controls the viewer
- Empty state: "No guide content" (italic, muted)

**Markdown rendering:** Lift `_renderMarkdown`, `_fetchTemplate`, and
`_extractSection` from `PagesScenarioNarrative` into the viewer. These
are pure rendering methods with no component-specific dependencies.

**`PagesScenarioNarrative` stays unchanged** — it's still used by the
connection-based content path. The viewer is a second consumer of
`scenario-narrative` events.

### Modal Slide Deck

When `display: modal`, the scenario handler creates a full-screen overlay
managed entirely in-browser.

**Overlay structure:**

```
┌──────────────────────────────────────────────────┐
│  ← Back                            Slide 2 of 4 │
│──────────────────────────────────────────────────│
│                                                  │
│  ## REST API Integration                         │
│                                                  │
│  Tickets don't only come from the web form...    │
│                                                  │
│──────────────────────────────────────────────────│
│  ○ Intro  ● REST API  ○ Routing  ○ Summary       │
│                                    [Next →]       │
└──────────────────────────────────────────────────┘
```

**Mini controller:** Shows slide names (from step labels), current slide
indicator, dot navigation, Next button.

**Flow:**

1. First modal step arrives in `executeAriaCommand`
2. Handler checks `stepQueue` for consecutive modal steps and pulls them
   all (deck look-ahead from the queue, not the outline)
3. All pulled steps' step-results are sent back to the server immediately
   (the steps completed from the server's perspective)
4. Handler creates the overlay, renders slide 1
5. **Next** advances to next slide (client-side, no server roundtrip)
6. Last slide's Next dismisses the overlay
7. **Escape** or **Back on slide 1** dismisses the entire deck

The modal returns void (non-blocking from `executeSequence`'s perspective).
The overlay manages its own lifecycle. When dismissed, it dispatches
`scenario-narrative-dismiss` for consistency.

**Deck look-ahead:** The handler peeks at `stepQueue` for consecutive
show-markdown commands with `display: modal`. It shifts them all out of
the queue, builds the deck, and sends step-results for each. This is
simpler than outline-based look-ahead — the queue already has the steps.

**Correction from D7:** The outline look-ahead was the original decision,
but queue-based look-ahead is simpler — the dispatch already groups steps
into the queue. The outline approach is fallback if steps aren't pre-queued.

### Outline Type Icons

The controller's `_renderNode` adds a type icon between the status
indicator and the label.

**Icon mapping:**

| Action | Icon |
|--------|------|
| `show-markdown` | 📝 |
| `spotlight` | 🔆 |
| `click` | 👆 |
| `fill` | ✍ |
| `navigate` | ➜ |
| unmapped | ○ |

Icons show for pending (○) and current (●) steps. Completed steps keep ✓.

**Data source:** Extend the server's `/scenario/outline` response to
include an `action` field on leaf nodes. The controller reads it and maps
to an icon. Unknown actions fall back to ○.

## Data Flow

### Panel mode

```
YAML step (display: panel)
  → scenario-handler.ts executeAriaCommand
  → dispatch CustomEvent('scenario-narrative', { type, markdown, path, section })
  → return void (non-blocking)

PagesScenarioYamlViewer (Guide tab)
  → willUpdate binds eventTarget listeners
  → _onNarrative stores content in _guideContent state
  → Guide tab renders markdown
  → Content persists until next scenario-narrative event
```

### Modal mode

```
YAML step (display: modal)
  → scenario-handler.ts executeAriaCommand
  → peek stepQueue for consecutive modal steps
  → shift all modal steps, send step-results for each
  → create full-screen overlay with deck
  → user clicks Next → advance slide
  → last Next or Escape → dismiss overlay
  → return void (non-blocking)
```

## Files Changed

| File | Change |
|------|--------|
| `packages/pages-aria/src/controller/scenario-yaml-viewer.ts` | Add tab bar, Guide tab, markdown rendering, event listener |
| `packages/pages-aria/src/server/scenario-handler.ts` | Branch on `display` property, modal overlay creation, deck look-ahead |
| `packages/pages-aria/src/controller/scenario-controller.ts` | Type icons in `_renderNode`, icon mapping |
| `packages/pages-aria/src/scenario/parser.ts` | Pass `display` property through parser |
| Server: scenario outline endpoint | Add `action` field to leaf nodes |

## Testing

- **Unit:** YAML viewer renders Guide tab with markdown content
- **Unit:** Modal deck groups consecutive modal steps from queue
- **Unit:** Outline icon mapping for all action types
- **E2E:** Panel mode — show-markdown content appears in Guide tab
- **E2E:** Modal mode — full-screen deck with Next/Back/Escape
- **E2E:** Mixed — panel and modal steps in same scenario

## References

- `packages/pages-aria/src/controller/scenario-yaml-viewer.ts` — existing viewer
- `packages/pages-aria/src/controller/scenario-narrative.ts` — markdown rendering source
- `packages/pages-aria/src/server/scenario-handler.ts:222` — show-markdown handler
- `packages/pages-aria/src/controller/scenario-controller.ts:270` — outline rendering
- Issue casehubio/casehub-pages#365 — show-markdown action
- decisions.md D1-D8 — all design decisions
