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

## D3: Dual-level YAML — inline for small, top-level for complex

**Choice:** Orchestration constructs can be expressed both ways: small coordination (mutex, retry, signal, wait, concurrent) inline as step fields or standalone step entries; complex structures (state machines, named barriers with specific counts, typed channels) declared in a top-level `orchestration:` block and referenced by name from steps.
**Alternatives:**
- Inline only — forces complex state machine definitions into step arrays, cluttered and hard to read
- Top-level only — verbose for simple cases like `retry: 3` or `signal: go`, everything needs a name
- Step decorators only — can't express standalone signals or concurrent blocks
**Rationale:** Matches existing YAML patterns in the codebase (inline parameters vs named module references). Small things stay local and readable. Complex things get named, documented, and reusable. Parser handles both: inline constructs are syntactic sugar that create anonymous ScenarioScope entries; top-level constructs are named entries created at parse time.
**Trade-offs:** Parser handles two paths for the same constructs. Worth it — the YAML authoring experience is significantly better.
**Sources:** Existing yaml-core module/parameter patterns, YAML binding table in #463 issue body
**Exploration:** quick
**Depends on:** D1 (scheduler is the only runner)
**Status:** captured
