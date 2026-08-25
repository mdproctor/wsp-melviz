# Session Handover — casehub-pages

**Branch:** `issue-334-dsl-and-scenario`
**Slot:** 152 (pages + examples/helpdesk)
**Date:** 2026-08-25

## Last Session

Implemented #365 show-markdown end-to-end, then built the tabbed viewer (Source + Guide tabs), modal slide deck with progressive loading, and helpdesk tutorial content. Extensive debugging of three root causes: (1) `executeSequence` bricking from stale push events after reload (fixed with try-finally), (2) dual WebSocket connections (fixed by reusing controller's connection via poll), (3) modal auto-pause racing with server Resume command in Play mode (fixed by removing auto-pause and using activeDeck guard). All verified in Playwright across multiple Reset cycles.

## Immediate Next Step

Continue iterating on the helpdesk demo. Open items: outline type icons need server-side `action` field in `/scenario/outline` endpoint (Java change), "run through" auto-advance mode for slides (word-count timer), and blocks-ui case viewer diagram (replacing the hand-coded SVG).

## Known Issues

- **Work item → case context feedback broken** — completing a work item via REST (`PUT /workitems/{id}/complete`) does not apply the humanTask `outputMapping` back to the case context. The case stays at `TRIAGED` instead of advancing to `RESOLVED`, so the `notify-resolution` binding never fires and the Notifications panel stays empty. Filed as casehubio/engine#988. The helpdesk demo fails at "Spotlight the resolution" because of this.
- **ES module caching** — `controller.js` import now uses `import('...?_=' + Date.now())` for cache busting. Works but re-parses the module on every load. A content-hash filename would be better long-term.
- **Controller push state** — the controller's `ScenarioConnectionController` creates its own connection before the module script runs. The module script polls for `_conn._ownConnection` and creates the handler on it. Works reliably but is fragile — accessing internal properties.
- **Helpdesk YAML duplication** — two copies at `src/main/resources/scenarios/` and `src/main/resources/META-INF/resources/scenarios/`. Both must be kept in sync.
