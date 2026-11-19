# Design Journal — issue-707-playbook-sync

## 2026-11-19 — Playbook sync landed

Synced 5 platform changes to TS yaml-core: trust model (4 risk-bearing caps + ASSISTED schema), canonical form (forEach → for-each across 17 files), scalar subjects (graphql/rest/mcp), tutorial headers, and cross-parser fixtures. Removed stale TRIGGERS capability. Code review found one missing export — fixed before landing.

Decision: kebab-case keys throughout TS types (`'for-each'` with bracket notation) rather than a normalization layer. Noisier code, but every bracket access maps directly to a YAML key — no translation ambiguity.

Filed 3 follow-up issues: #714 (test describe string cleanup, XS), #715 (LSP completion verification, XS), #716 (category execution tests, M).
