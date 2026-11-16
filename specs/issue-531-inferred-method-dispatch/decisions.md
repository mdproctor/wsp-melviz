# Decisions — issue-531-inferred-method-dispatch

## D1: Walker insertion point for inferred dispatch

**Choice:** `Catalog.resolveInferred()` — optional method on the Catalog interface. Walker calls `catalog.resolve(key) ?? catalog.resolveInferred?.(key)`. Returns a normal `CatalogEntry` with a deferred `Action`, producing a standard `PluginStep`.
**Alternatives:**
- New `InferredStep` variant in `ResolvedStep` — semantically cleaner but touches every consumer of the union type (evaluator, viewer, validator, diff tools)
- Dynamic catalog extension (scan DOM before parsing, register entries) — requires DOM availability at parse time, wrong lifecycle
**Rationale:** Minimal Walker change (one line). Reuses `PluginStep` so all downstream infrastructure (StructuralEvaluator, DecoratorChain, step viewer, portability validator) works unchanged. The trade-off is that the `PluginStep`'s `Definition` has permissive inputs — parameter validation is deferred to runtime, which is inherent to the problem since we don't know the target component at parse time.
**Trade-offs:** Parse-time parameter validation is lost for inferred steps. Portability can only be fully resolved at runtime. The PluginStep conflates two resolution modes (explicit catalog vs inferred).
**Sources:** `yaml-core/src/step/walker.ts:320-328` (current unknown-key error path), `yaml-core/src/step/walker.ts:66-79` (ResolvedStep union)
**Exploration:** deep-analysis
**Status:** captured

## D2: TS discovery protocol — SPI in pages-primitives

**Choice:** `PlaybookCapability` type + `PLAYBOOK_ACTIONS` Symbol (`Symbol.for('playbook-actions')`) in `pages-primitives`. Components return a lightweight `Record<string, PlaybookMethodDescriptor>` — no yaml-core dependency. The platform (pages-aria) bridges to `Definition[]` internally via an adapter.
**Alternatives:**
- Return `Definition[]` directly — reuses the existing type but creates wrong-way dependency from blocks-tier to yaml-core (infrastructure)
- Separate `pages-playbook-spi` package — cleanest separation but adds package management overhead for one Symbol + one interface
- Convention-only (no typed SPI) — maximum flexibility, no type safety for component authors
**Rationale:** pages-primitives is already the "minimal component contracts" package (AriaTarget, AriaState). Adding a Symbol + lightweight descriptor type is natural there. Components stay decoupled from catalog internals. Third-party components can even use `Symbol.for('playbook-actions')` directly without any CaseHub import. The platform bridges to `Definition` at the pages-aria layer — the only place that needs to know both types.
**Trade-offs:** Extra conversion step (PlaybookCapability → Definition) in pages-aria. Component authors don't get auto-complete for Definition fields — but they shouldn't need to, since PlaybookCapability is simpler.
**Sources:** `pages-editor-core/src/discovery.ts` (EDITABLE_TEXT pattern — same SPI approach), `pages-primitives/src/a11y/aria-contract.ts` (existing lightweight contracts)
**Depends on:** D1 (Catalog.resolveInferred consumes the bridged Definitions)
**Exploration:** deep-analysis
**Status:** captured

## D3: Java discovery protocol — @PlaybookInterface on interface type

**Choice:** New `@PlaybookInterface` annotation at the interface level. All methods on an annotated interface become callable via inferred dispatch. `ActionRegistry.invoke()` falls back to scanning the target bean's `@PlaybookInterface`-annotated interfaces when no `@PlaybookAction` handler matches.
**Alternatives:**
- `@PlaybookAction` on interface methods — consistent annotation reuse but conflates explicit registration with interface-level opt-in. No way to opt-in an entire interface at once.
- `PlaybookCallable` marker interface (like Serializable) — too coarse, ALL public methods callable with no per-interface control.
**Rationale:** Clean separation of concerns: `@PlaybookAction("name")` for explicit per-method registration on beans (existing pattern), `@PlaybookInterface` for interface-level opt-in (new pattern). The interface IS the security boundary — all methods on it are callable by method name. Parameter mapping uses Java `-parameters` compile flag for name-based arg resolution from `ActionContext`.
**Trade-offs:** Two annotation types to understand. An interface might expose methods the author didn't intend as playbook-callable — but annotating the interface is already a deliberate opt-in.
**Sources:** `playbook-client/ActionRegistry.java:20-36` (current @PlaybookAction scan), `playbook-client/PlaybookAction.java` (existing annotation)
**Depends on:** D1 (ActionRegistry fallback parallels Catalog.resolveInferred)
**Exploration:** quick
**Status:** captured

## D4: Naming convention — uniform kebab-case with auto-conversion

**Choice:** All step names use kebab-case, whether registered or inferred: `load-content`, `next-diff`, `set-view-mode`. The executor auto-converts to camelCase for method lookup (`load-content` → `loadContent`).
**Alternatives:**
- Exact method name (camelCase) — creates a visual dispatch-mechanism signal but leaks resolution mechanism into YAML syntax. If a method is promoted from inferred to registered, its YAML name changes, breaking playbooks.
**Rationale:** Uniform naming means promotion/demotion between registered and inferred is an internal implementation change, not a YAML-breaking change. The dispatch mechanism is an implementation detail — YAML authors care about WHAT a step does, not HOW it resolves. Consistent with existing step naming.
**Trade-offs:** No visual distinction between registered and inferred steps in YAML. Requires kebab→camel conversion on both TS and Java sides (trivial).
**Sources:** Design review round 1 — reviewer identified camelCase as leaky abstraction
**Exploration:** deep-analysis
**Status:** revised (was: exact camelCase; revised after design review)

## D5: Parameter mapping — descriptor-driven named→positional

**Choice:** The executor uses the `PlaybookCapability` descriptor's `params` key order to map named YAML parameters to positional method arguments. No method signature changes needed.
**Alternatives:**
- Single params object — methods accept `{panel, content, label}` as one arg. Type-safe but requires changing existing method signatures across all components.
**Rationale:** Existing component APIs (loadContent(panel, content, label), setViewMode(mode)) work as-is. The descriptor declares parameter names and order. The executor extracts named values from YAML in descriptor order and passes positionally. JavaScript object key order is insertion order for string keys, making this deterministic.
**Trade-offs:** Fragile if descriptor key order doesn't match method signature order — but since the component author writes both, they control the alignment. No runtime type checking beyond what the descriptor declares.
**Sources:** `command-executor.ts:108-115` (current extractArgs positional pattern), `pages-document-diff.ts:549` (loadContent signature)
**Depends on:** D2 (PlaybookCapability provides the param descriptors)
**Exploration:** quick
**Status:** captured

## D6: playbook-handler.ts — extend with fallback, defer convergence

**Choice:** Add inferred dispatch to `playbook-handler.ts`'s default case with generic visual feedback (highlight target, show method+params, execute). Leave the two switches (command-executor.ts and playbook-handler.ts) separate. Convergence is a separate future refactor.
**Alternatives:**
- Converge the switches — unify into single dispatch path. Eliminates duplication but much larger scope.
- Defer entirely — inferred dispatch only in client-side executor. Server-dispatched scenarios can't use inferred methods. Creates confusing inconsistency.
**Rationale:** Server-dispatched scenarios should support inferred methods (otherwise `loadContent` works in client mode but not server mode — confusing). But converging the two switches is a large refactoring orthogonal to the security/allowlist design. Extend with fallback keeps scope focused.
**Trade-offs:** Duplication persists between the two dispatch paths. The inferred path is implemented twice (once in command-executor.ts, once in playbook-handler.ts) — but both call the same `PLAYBOOK_ACTIONS`-based resolution, so the logic is shared.
**Sources:** `pages-aria/src/server/playbook-handler.ts:562-666` (server switch), `pages-aria/src/executor/command-executor.ts:77-106` (client switch)
**Exploration:** quick
**Status:** captured
