# HANDOFF — casehub-pages

## Last Session

Landed #749 (PlaybookSplitView → ThreeWayViewer consumer, -103 lines) and #750 (PanelController for floating panel state in playbook controller). Both reuse the ViewerController architectural pattern. Brainstormed #747 (conditional @WebSocket endpoint) — design spec committed, implementation not started. Research confirmed `@UnlessBuildProperty` works for conditional `@WebSocket` classes because Arc excludes the bean before endpoint scanning.

## Immediate Next Step

Run `writing-plans` for #747, then `executing-plans`. The spec is at `specs/issue-747-conditional-websocket-endpoint/2026-11-26-conditional-websocket-endpoint-design.md`. The work is Java/Quarkus in `backend/push-runtime`.

## References

- `specs/issue-747-conditional-websocket-endpoint/2026-11-26-conditional-websocket-endpoint-design.md`
- `specs/issue-747-conditional-websocket-endpoint/decisions.md`
- `blog/2026-11-26-mdp01-the-controller-pattern-pays-off.md`
