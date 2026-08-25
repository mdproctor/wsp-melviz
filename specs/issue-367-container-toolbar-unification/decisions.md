## D1: Free-layout strategy absorbs all frame interaction

**Choice:** The free-layout strategy owns DnD, edge splits, tab drag, z-order, snap zones, and frame chrome — everything the backend currently does for free-mode frames. The backend becomes a thin adapter. No coordinator boundary.
**Alternatives:**
- Strategy renders, backend coordinates — keeps DnD state machine separate. Creates the same coordination boundary we're eliminating.
**Rationale:** One system instead of two. DnD bugs are in one place. Every free-layout container at every depth gets the same capabilities automatically.
**Trade-offs:** Larger migration — ~220 lines of DnD state machine move into the strategy. Strategy file grows significantly.
**Exploration:** quick
**Status:** captured

## D2: FloatingFrameBackend becomes a facade

**Choice:** FloatingFrameBackend wraps the root Container, translating the existing API (`renderFrame`, `addTab`, `removeTab`) into Container operations. Zero breaking changes for consumers.
**Alternatives:**
- Replace with Container directly — cleaner but breaks existing API surface in activation.ts and wireFloatingWorkspace
**Rationale:** Backward compatibility. Deprecation tracked as #370.
**Trade-offs:** Facade is temporary code. Must be removed once consumers migrate.
**Depends on:** D1 (strategy owns everything the backend used to)
**Exploration:** quick
**Status:** captured

## D3: DnD absorbed into free-layout strategy as internal coordinator

**Choice:** The free-layout strategy creates an internal DnD handler. Since the strategy already knows all entry positions and sizes, it can detect targets. Edge splits use `refreshEntry` for surgical replant.
**Alternatives:**
- Pluggable DnD via callbacks — strategy fires events, callback implementation handles detection. Creates the coordination boundary we're eliminating.
**Rationale:** The strategy has all the spatial information. A callback interface adds indirection without value.
**Trade-offs:** Strategy grows in complexity. DnD logic is coupled to rendering.
**Depends on:** D1 (strategy absorbs all interaction)
**Exploration:** quick
**Status:** captured

## D4: Full DnD at every depth

**Choice:** Any free-layout container, root or nested, supports tab drag between entries, edge splits, and cross-entry drops. No gating flag.
**Alternatives:**
- Root only for now, nested later via `policy.allowDnd` flag. Reduces scope.
**Rationale:** Uniform behavior root to leaf. The DnD mechanism is depth-agnostic once it's in the strategy.
**Trade-offs:** Must handle edge cases like drag from nested free into parent free. Larger scope.
**Depends on:** D3 (DnD in strategy)
**Exploration:** quick
**Status:** captured

## D5: Eliminate FloatingFrameEngine, extract math to layout-math.ts

**Choice:** Eliminate the engine as a class. Extract its pure functions (zone preset math, proportional scaling) into `layout-math.ts`. The free-layout strategy calls these functions. Engine's 58 tests become: pure function tests for layout math + strategy-level tests for DOM + container-level integration tests.
**Alternatives:**
- Keep engine as persistence layer — strategy renders, engine handles state capture/restore. Preserves dual state.
- Keep engine as strategy's internal state model — cleaner than persistence-only but still two objects for one concern.
**Rationale:** The engine's capabilities are already duplicated across Container + strategy. Every engine capability maps to an existing Container/strategy capability. Triple-bookkeeping (engine + backend + strategy) is the root cause of mode-switching complexity.
**Trade-offs:** 58 engine tests need migration. Pure layout math extracted to utility module preserves testability without DOM.
**Sources:** Current overlap analysis — entryState, zOrder, arrange(), captureContainerState() all duplicate engine capabilities.
**Exploration:** deep-analysis
**Status:** captured

## D6: Tab drag propagation via DOM event bubbling

**Choice:** Child container fires `pages-tab-drag-start` custom DOM event. Parent free-layout strategy listens on its host element. Zero coupling between container instances.
**Alternatives:**
- Content factory bridge — factory wraps child callbacks to relay drag events upward. Couples factory to DnD.
- Container-level `onChildEvent` callback — adds upward event path to Container interface. Over-general.
**Rationale:** DOM bubbling is the web platform's native parent-child communication. Works at any depth. Simple to test with `dispatchEvent`. No coupling.
**Trade-offs:** DOM events are stringly-typed. Must document the event contract.
**Depends on:** D3, D4 (DnD at every depth)
**Exploration:** quick
**Status:** captured

## D7: wireFloatingWorkspace remains as thin initializer

**Choice:** wireFloatingWorkspace stays but shrinks from ~360 lines to ~80. Creates root Container from saved layout config, passes to `restoreState`, wires external event dispatch (`pages-frame-move`, etc.), handles detach.
**Alternatives:**
- Eliminate entirely — activation.ts creates root Container directly. Loses useful abstraction boundary.
**Rationale:** Useful abstraction between "YAML config → running workspace" and Container internals. Activation doesn't need to know how containers work.
**Trade-offs:** One more module to maintain, but dramatically simpler than current.
**Depends on:** D1, D5 (no engine, no backend coordination)
**Exploration:** quick
**Status:** captured

## D8: ContainerState persistence — new format only, clean migration

**Choice:** `ContainerState` is the sole persistence format. A migration function converts `FrameLayout[]` → `ContainerState` at load time in wireFloatingWorkspace. No dual-format support.
**Alternatives:**
- Dual read, single write — restore accepts both formats by shape detection. No migration needed but carries format debt permanently.
**Rationale:** No technical debt from day one. Migration function is straightforward (FrameLayout maps directly to ContainerState). Function can be deleted once all persisted layouts are re-saved.
**Trade-offs:** Existing saved layouts must pass through the migration function. If migration has bugs, old layouts break on first load.
**Depends on:** D5 (engine eliminated, ContainerState is the model)
**Exploration:** quick
**Status:** captured
