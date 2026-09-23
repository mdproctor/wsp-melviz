## D1: Concurrency model — cooperative round-robin, not Web Workers

**Choice:** Cooperative round-robin scheduler on the main thread with Promise-based coordination
**Alternatives:**
- Web Workers with SharedArrayBuffer/Atomics — heavyweight, no shared memory, would need to rebuild coordination primitives across worker boundaries
- Raw Promise.all with natural event-loop interleaving — non-deterministic ordering, no virtual time control
**Rationale:** Single-threaded cooperative model eliminates all Atomics/SharedArrayBuffer complexity. All primitives become plain TS objects with Promise callbacks. Round-robin gives deterministic step interleaving needed for tutorials and simulation replay. Web Workers can't share objects and would require a coordinator pattern that rebuilds the same primitives — complexity explosion for no benefit.
**Trade-offs:** No true CPU parallelism — but the use case (tutorials, simulation) doesn't need it. Steps are I/O-bound (DOM updates, network, timers), not compute-bound.
**Sources:** Discussion with user, browser API constraints (Atomics.wait() blocked on main thread)
**Exploration:** deep-analysis
**Status:** captured

## D2: Virtual time — scheduler owns all clocks

**Choice:** Scheduler maintains virtual clock. Timers, delays, and wait states are scheduler-managed, not browser timers.
**Alternatives:**
- Real setTimeout/setInterval — can't pause, can't speed-control, non-deterministic
**Rationale:** Tutorial/simulation engine needs pause/resume, speed multiplier, and deterministic replay. All three require virtual time. Every real-time simulation system ends up here.
**Trade-offs:** Building a lightweight cooperative runtime adds complexity. Worth it for the pause/speed/replay capabilities.
**Sources:** User discussion, SpeedMultiplier interface in Java yaml-core
**Exploration:** quick
**Status:** captured

## D3: Main execution is just another queue

**Choice:** No privileged "main thread" — all execution paths (including the primary sequence) are queues managed by the scheduler.
**Alternatives:**
- Special-case main execution with concurrent branches as secondary — creates two code paths, nesting is harder
**Rationale:** Uniform queue model means concurrent blocks, fork/join, and nesting all work the same way. A concurrent block in a branch just adds sub-queues — the scheduler doesn't care about depth. Also enables simulation data feeds as peer queues alongside scenario queues.
**Trade-offs:** Slightly more abstract mental model for simple linear scenarios. Mitigated by the fact that linear scenarios are just a single queue — the abstraction is invisible until you need concurrency.
**Sources:** User insight during design discussion
**Exploration:** quick
**Status:** captured

## D4: Primitives are scheduler-agnostic (#462), scheduling lives in runner (#461/#463)

**Choice:** Orchestration primitives are pure Promise-based types with no knowledge of the scheduler, virtual time, or step queues. The scenario runner owns scheduling, queue management, and virtual time.
**Alternatives:**
- Primitives aware of scheduler (coupled) — harder to test, can't use primitives outside scenario context
**Rationale:** Clean separation. Primitives can be unit-tested without a scheduler. The scheduler uses primitives for inter-queue coordination but the primitives don't know they're being used in a round-robin context. This matches Java's design where primitives are general-purpose and the orchestrator is the scheduling layer.
**Trade-offs:** Scheduler must wrap primitive timeouts in virtual-time-aware wrappers rather than primitives handling timeouts natively. Small code cost, significant design clarity gain.
**Sources:** Java yaml-core package structure (orchestration vs runtime separation)
**Exploration:** quick
**Status:** captured

## D5: TS API surface — async equivalents of Java blocking calls

**Choice:** Every Java blocking method becomes an async method returning Promise. Same method names, same semantics, different programming model.
**Alternatives:**
- Synchronous API with callback registration — awkward in modern TS, can't use async/await
- Different method names to signal async nature — unnecessary divergence from Java API
**Rationale:** TypeScript's async/await is the natural equivalent of Java's virtual thread blocking. Same caller experience (write sequential-looking code), same semantics (wait for resource). Keeping method names identical to Java reduces cognitive load when reading both codebases.
**Trade-offs:** Java's timeout overloads use `(long timeout, TimeUnit unit)` — TS uses `(timeoutMs?: number)` returning `Promise<boolean>` or `Promise<T | undefined>`. Slight API divergence on timeout signatures.
**Sources:** Java yaml-core API survey, TS async/await conventions
**Exploration:** quick
**Status:** captured

## D6: Discriminated unions for Java sealed types

**Choice:** Java sealed interfaces (LoopDirective, RetryDirective) become TypeScript discriminated unions with a `type` field.
**Alternatives:**
- Class hierarchies with instanceof — verbose, doesn't leverage TS's type narrowing
- Enum + separate data objects — splits definition across two locations
**Rationale:** Discriminated unions are idiomatic TS and give exhaustive switch checking. `if (directive.type === 'count')` narrows the type automatically. Maps cleanly to YAML parsing where `type` is inferred from the structure.
**Trade-offs:** No methods on variants (functions take the union type as argument). Minor style difference from Java.
**Sources:** TypeScript handbook (discriminated unions), existing yaml-core TS patterns
**Exploration:** quick
**Status:** captured
