## D1: Overall implementation architecture

**Choice:** Extend in-place — add new decorators to DecoratorChain, new structural step to Walker + StructuralEvaluator, extend ParallelStep with policy fields
**Alternatives:**
- New evaluation layer — separate evaluator for event-sequence features. Unnecessary abstraction boundary when the existing dispatch pattern handles this cleanly.
**Rationale:** The Walker → StructuralEvaluator → DecoratorChain architecture is designed for exactly this kind of extension. Each new feature maps to a specific extension point: `from:`/`cancel:`/`background:` are decorators, `race:` is structural, parallel policies are fields on the existing ParallelStep.
**Trade-offs:** No isolation between old and new features — a regression in DecoratorChain affects everything. Acceptable because the chain is well-tested and decorators are independent.
**Sources:** walker.ts, structural-evaluator.ts, decorator-chain.ts
**Exploration:** quick
**Status:** captured

## D2: Race step cancellation mechanism

**Choice:** AbortSignal on Context — add `signal?: AbortSignal` to the Context interface. Race evaluator creates an AbortController per branch, aborts losers when the winner completes. Decorators that do async waits check `signal.aborted` before blocking.
**Alternatives:**
- AbortSignal on ScenarioScope — scope is shared across branches, context is per-evaluation-path. Wrong granularity.
- Promise.race only (no cancel) — simplest but leaks resources, doesn't match platform cancel semantics.
**Rationale:** Context is the per-step execution environment. AbortSignal is per-branch state in a race. AbortController is the standard Web API for cooperative cancellation, integrates with fetch/streams natively.
**Trade-offs:** Existing decorators (wait, delay, timeout, semaphore) must be updated to check `signal.aborted` — adds a conditional to each but ensures clean cancellation.
**Sources:** decorator-chain.ts Context interface, platform #603 race semantics
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (extends Context, which is part of the existing architecture)

## D3: ParallelCompletionTracker design

**Choice:** Two-latch pattern matching platform — advanceAfter triggers advancement latch (step Promise resolves), remaining branches continue toward completion latch. cancelRemaining aborts remaining branches after advancement. collectionTimeout sets a deadline on the completion latch.
**Alternatives:**
- Single Promise.race with N — resolve when Nth completes. Simpler but can't collect remaining results after advancement threshold.
**Rationale:** Matches platform semantics exactly. Handles all three policies: barrier (wait all, current default), discriminator (advance on N, collect rest), race-as-policy (advance on first, cancel rest).
**Trade-offs:** More complex than current Promise.allSettled. The two-latch pattern is inherently more stateful. Worth it because discriminator is the primary use case for parallel extensions.
**Sources:** platform #392, current evaluateParallel in structural-evaluator.ts
**Exploration:** quick
**Depends on:** D2 (cancelRemaining uses AbortController from D2)
**Status:** captured

## D4: from: decorator — variable injection mechanism

**Choice:** VariableResolver scope push — same pattern as ForEachDecorator which pushes `each.*` variables via `resolver.withObjectScope('from', source)`. The from: decorator receives from the channel, creates an ObjectVariableSource, and pushes it onto the resolver.
**Alternatives:**
- Direct context params merge — breaks the resolver abstraction and doesn't support nested field access (`${from.reading.value}`).
**Rationale:** Consistent with existing patterns (ForEachDecorator, WaitDecorator both use resolver scoping). Supports nested field access naturally.
**Trade-offs:** None significant — this is the established pattern.
**Sources:** decorator-chain.ts ForEachDecorator, VariableResolver.withObjectScope
**Exploration:** quick
**Depends on:** D1 (extends DecoratorChain)
**Status:** captured

## D5: loop: continuous — implementation

**Choice:** New LoopDirective variant `{ type: 'continuous' }`. LoopDecorator treats this as `maxIterations = Infinity` with no until condition. Termination comes from cancel: decorator or AbortSignal. `parseLoopDirective` recognizes `continuous` as a string literal.
**Alternatives:**
- Sentinel count value (`loop: 999999`) — fragile, doesn't express intent, platform has an explicit Continuous variant.
**Rationale:** Matches platform's `LoopDirective.Continuous`. Clean discriminated union extension.
**Trade-offs:** LoopDecorator must check AbortSignal on each iteration to avoid infinite loops when cancel/race aren't present. Safety cap of 100000 iterations for non-cancelled continuous loops prevents runaway.
**Sources:** orchestration/directives.ts LoopDirective union, platform #563
**Exploration:** quick
**Depends on:** D1 (extends existing LoopDirective), D2 (uses AbortSignal for termination)
**Status:** captured

## D6: background: decorator — spawn semantics

**Choice:** Scope.spawn() integration — `background: true` wraps the inner execution in `scope.spawn()`, returning a handle. The decorator resolves immediately (spawn-and-continue). Spawned execution is scope-bounded — `scope.close()` cancels it via AbortSignal.
**Alternatives:**
- Detached Promise — fire-and-forget with no handle, cancellation, or scope bounding. Leaks tasks.
**Rationale:** ScenarioScope already has `spawn()`. Background execution must be scope-bounded so that scope teardown (dispose, runTo) cleanly cancels background work.
**Trade-offs:** Background failures are silent unless explicitly checked via the spawn handle. This matches platform behavior — background tasks report failure through the scope, not the step chain.
**Sources:** orchestration/types.ts ScenarioScope.spawn(), platform #563 background semantics
**Exploration:** quick
**Depends on:** D1 (extends DecoratorChain), D2 (uses AbortSignal for scope-bounded cancellation)
**Status:** captured

## D7: cancel: decorator — signal-triggered cancellation

**Choice:** Signal listener + AbortController — `cancel: signal-name` creates an AbortController, registers a listener on the named signal. When the signal fires, the controller aborts. The AbortSignal is merged into the context for the inner step. Composable with `loop: continuous` — the loop checks `signal.aborted` each iteration.
**Alternatives:**
- Polling-based — check signal state at start of each decorator layer. Misses mid-step cancellation and doesn't compose with async waits (semaphore.acquire, channel.receive).
**Rationale:** Reactive cancellation via AbortController is the standard pattern. The signal listener fires asynchronously and propagates cancellation through the entire decorator chain below cancel: in the stack.
**Trade-offs:** Signal listener must be cleaned up on normal completion (not just cancellation). Cleanup in finally block.
**Sources:** platform #563 cancel: semantics, AbortController Web API
**Exploration:** quick
**Depends on:** D2 (AbortSignal on Context), D1 (extends DecoratorChain)
**Status:** captured

## D8: from: hardening — ConsumerHandle lifecycle

**Choice:** Named consumer as scope primitive — `from: { channel: events, name: enricher }` registers a ConsumerHandle in the scope. `stop: enricher` / `start: enricher` are new step-level actions (plugin steps registered in the catalog, not decorators). ConsumerHandle wraps an AbortController for stop, with a promise-based pause/resume mechanism.
**Alternatives:**
- Decorator-level pause/resume — pause/resume as decorators on subsequent steps. Awkward because pause/resume are imperative actions, not step modifiers.
**Rationale:** Matches platform's ConsumerHandle pattern. Stop/start/pause/resume are actions performed on a named consumer — they are imperative commands, not step decorations.
**Trade-offs:** Requires catalog registration for stop/start/pause/resume actions. Small overhead but consistent with the invoke/plugin step model.
**Sources:** platform #602 named consumers, ScenarioScope primitive registry
**Exploration:** quick
**Depends on:** D4 (from: decorator), D2 (AbortController for stop)
**Status:** captured
