# Session Handover — casehub-pages

**Branch:** `issue-334-dsl-and-scenario`
**Slot:** 152 (pages + examples/helpdesk)
**Date:** 2026-08-27

## Last Session

Improved the helpdesk demo presentation: reframed slides for choreography vs orchestration (hybrid execution model with 6-model table), added tabbed Diagram/YAML views in modal slides, syntax highlighting for YAML code blocks, markdown table rendering, outline step icons (unicode outline-weight), and Prev/Next navigation fix. Exported case architecture SVG from blocks-ui export.html (fixed React dedup, style bloat, viewport scaling). Filed engine bugs #988/#989/#990 (moved misfiled pages issues). Added action field to outline endpoint (Java). Updated CLAUDE.md with demo profile requirement.

## Immediate Next Step

Verify end-to-end demo once engine JAR is deployed (engine had build issue preventing install). The demo fails at "Spotlight the resolution" because engine#988 fix (outputMapping feedback) hasn't been picked up. Once deployed, reset and run the full demo — notification should fire this time.

## Blocked On

- **Engine JAR deployment** — engine had a build issue preventing `mvn install`. Once fixed, restart helpdesk Quarkus (`-Dquarkus.profile=demo`) to pick up the new engine.
- **Work-end incomplete** — review gates all passed, findings filed (#379). Squash/merge/close deferred until demo is verified end-to-end.

## Known Issues

- **Work item → case context feedback broken** — engine#988. Work item completion via REST doesn't apply outputMapping. Case stays TRIAGED. Filed, reportedly fixed but JAR not deployed.
- **Duplicate work items** — engine#989. contextChange binding fires twice for single case. Filed, reportedly fixed but JAR not deployed.
- **Engine reset API** — engine#990. No clean reset without JVM restart.
- **Export viewport scaling** — blocks-ui#138. Fixed (MAX_ZOOM clamp removed). Re-exported architecture SVG.
- **ES module caching** — controller.js uses `import('...?_=' + Date.now())` for cache busting.
- **Helpdesk YAML duplication** — two copies at `src/main/resources/scenarios/` and `META-INF/resources/scenarios/`. Both must be kept in sync.
