# Design Journal — issue-408-scenario-engine

## 2026-08-11 — Spec session: format and demo SPI convention

### §Scenario Format (#409)

Formalized the cross-platform scenario YAML format as a platform protocol
document (`parent/docs/platform/scenario-format.md`). Derived from the
design spec's §3 with 9 amendments from review feedback:

- **ControlChannel resilience** — reconnect-resets-state model, per-step ack
  timeout, no clock sync needed
- **DataTrigger** — collapsed to server-side polling (convergent with await
  syntax), not client-side DataSet evaluation
- **`fill: { from: data }`** — resolution convention via `data-field`
  attributes, element-type dispatch
- **`on-error: continue|stop|pause`** — scenario-level failure policy
- **File distribution** — executor pushes data via `POST /scenario/bootstrap`
  (Option B — no shared resource module needed)
- **DemoCurrentPrincipal** — shared in platform-api, not per-app

### §Demo SPI Convention (#410)

Wrote the demo SPI convention protocol (`parent/docs/platform/demo-spi-convention.md`):
`@Alternative @Priority(300) @IfBuildProfile("demo")`. Pull/push mode templates.
Priority allocation table (300=demo connector, 200=demo identity, 100=OIDC).
Profile convention: demo (synthetic) vs dev (live dev creds) vs prod.

Noted discrepancy: clinical's existing `DemoCurrentPrincipal` uses
`@IfBuildProfile("dev")` with fixed identity — needs migration to
shared `@IfBuildProfile("demo")` version.

## 2026-08-20 — Distributed executor protocol design (#418)

### §Executor Protocol (#418)

Designed the distributed executor protocol — how the orchestrator
dispatches ordered step sequences to executors (browser and services)
via push wire WebSocket. Six key decisions:

- **D8: Ordered step sequences** as the dispatch unit (not single-step
  RPC or full sub-scenarios). Reduces round-trips while keeping
  executors simple.
- **D9: Push wire WebSocket** as universal transport for all executor
  types. Browser executors already use it; service executors connect
  as WebSocket clients.
- **D10: New PushRequest/PushMessage op types** — ExecutorRegister,
  StepResult, dispatch-sequence, executor-control. Pre-release, so
  sealed interface changes are free.
- **D11: CDI @ScenarioAction annotation** for service executor
  contract. Shared library dispatches to annotated handlers.
- **D12: Optional nested YAML hierarchy** — chapters → sections →
  steps → commands. All levels optional. Steps are the demo-meaningful
  unit; commands execute without pausing.
- **D13: Separable controller** — REST/GraphQL/MCP API on the
  orchestrator, with state broadcast on `scenario:state` topic.
  Controller UI can run on a separate device (phone as presenter
  remote).

### §Format Evolution

This spec explicitly evolves the cross-platform design spec's format.
The `delivery` modes (rest/ui-form/simulated) are replaced by `target`
(executor name). `ControlChannel` is superseded by the executor
protocol. `GraphQLDispatcher` retained as fallback for services without
local executors.

### §Implementation Started

Implemented Task 1 of the plan: PushRequest/PushMessage protocol
extensions. Added ExecutorRegister, StepResult records and
dispatchSequence, executorControl static methods. 123 tests pass.

Discovered IntelliJ MCP gotcha: ide_import_modules with duplicate
Maven artifactIds routes edits to the first-registered module.
Worked around with git patch transfer. Captured as GE-20260821-8ada11.
