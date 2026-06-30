# Design Journal — issue-64-workbench-primitives

## §1 Framework Positioning

**2026-06-30** — Before implementation, audited all documentation across the CaseHub ecosystem and found "dashboard rendering runtime" embedded in every layer — README, PLATFORM.md, presentation slides, app CLAUDE.md files. Downstream apps (Claudony, DraftHouse, DevTown) were dismissing casehub-pages for anything beyond data visualization.

Repositioned as "web application framework" across 7 repos in one pass. Key additions: five-category capabilities table (Architecture, Application Shell, Data, Components, Developer Experience), Getting Started guidance (Quinoa, TypeScript DSL, hostPanel, recursive composition, drive requirements upstream). Filed #78 for cross-repo tracking, #79 for examples gallery code rename.

This reframing is prerequisite to the workbench primitives implementation — apps need to understand they can host custom Web Components via `hostPanel()` and drive requirements upstream before they'll adopt the new primitives.
