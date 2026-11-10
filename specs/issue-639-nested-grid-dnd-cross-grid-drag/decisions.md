## D1: Controller Architecture

**Choice:** Controller Tree with Coordinator
**Alternatives:**
- Single Unified Controller — one controller manages all grid levels; requires full rewrite of existing 877-line controller, high risk
- Event Delegation Only — independent controllers communicate via DOM events; cross-grid state is split with no shared state, hard to keep consistent at arbitrary depth
**Rationale:** Preserves the existing well-tested single-grid controller. Adds cross-grid awareness as a separate concern (coordinator). Naturally supports arbitrary depth via tree traversal. Clean bridge routing via `gridPath` property on events.
**Trade-offs:** Additional module (coordinator) to maintain. Controllers need lifecycle hooks for registration/deregistration.
**Sources:** grid-edit-controller.ts (existing controller architecture), activation.ts:948-951 (attachment callsite), grid-edit-bridge.ts (bridge routing)
**Exploration:** quick
**Status:** captured
