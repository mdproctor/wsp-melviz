## D1: Replace existing runners entirely

**Choice:** Delete runner.ts and sectioned-runner.ts. The DES scheduler is the only execution engine. The public API retains the TutorialRunner shape (play/pause/step/setSpeed/dispose) but the implementation is the scheduler.
**Alternatives:**
- Layer underneath — scheduler under existing runners as facades. Preserves backward compat but adds an unnecessary abstraction layer that's pure tech debt.
- Compose alongside — scheduler as a peer, runners delegate for orchestration only. Two execution paths, harder to reason about.
**Rationale:** Pre-release project with no external consumers. The old runners are sequential-only with no orchestration support. Keeping them as wrappers around the scheduler adds complexity for zero benefit. Clean replacement gives one code path, one execution model, no legacy baggage.
**Trade-offs:** Tests for the old runners need rewriting against the new API. Worth it — those tests verify sequential execution which the scheduler handles as a single-queue degenerate case.
**Sources:** Survey of runner.ts, sectioned-runner.ts, types.ts in pages-aria/src/scenario/
**Exploration:** quick
**Status:** captured

## D2: Strategy pattern for step dispatch

**Choice:** Scheduler calls a `StepExecutor` interface per delivery type. `AriaExecutor`, `GraphqlExecutor`, `SimulatedExecutor` each implement `execute(step): Promise<void>`. Scheduler is delivery-agnostic — it manages queues and timing, executors handle domain-specific step logic.
**Alternatives:**
- Switch in scheduler — couples scheduler to every delivery type, grows as types are added
- Event-based dispatch — fully decoupled but needs Promise-based ack for completion tracking, overengineered for 3 known types
**Rationale:** Clean separation. The scheduler manages concurrency, virtual time, and queue ordering. Executors handle DOM (aria), network (graphql), or data injection (simulated). Adding a new delivery type is one new class, zero scheduler changes.
**Trade-offs:** One extra interface. Negligible cost for the extensibility it provides.
**Sources:** Existing command-executor.ts (aria dispatch), ScenarioStep discriminated union in types.ts
**Exploration:** quick
**Depends on:** D1 (scheduler is the only runner)
**Status:** captured
