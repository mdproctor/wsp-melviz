## D1: Implementation order

**Choice:** Bottom-up by independence: H3 → M4a → H1 → M6 → H4
**Alternatives:**
- Risk-first (H1 first) — tackle hardest piece when context is cleanest, but delays quick wins and risks blocking smaller items if annotations hits complexity
- Test-first (M6→H4 first) — builds validation infrastructure early, but test framework can't test error-aware or annotated playbooks until those land
**Rationale:** Quick wins (H3, M4a) build momentum and are independently valuable. H1 is the most complex piece — mid-sequence gives maximum context freshness without blocking smaller items. M6→H4 forms a natural pair at the end, and H4 as capstone exercises all earlier work.
**Trade-offs:** If annotations (H1) proves harder than expected, M6/H4 get delayed. Mitigated by H1 being independent — we can always skip decorator migration and still have parse+extract+map.
**Sources:** docs/specs/2026-11-07-ts-platform-parity-audit.md (Phase 2 table), structural-evaluator.ts (dispatch switch — no control case), walker.ts (annotations always [])
**Exploration:** quick
**Status:** captured

## D2: Error scope injection strategy (H3)

**Choice:** Derive error scope from Result.failure via ObjectVariableSource — supports `${error.type}`, `${error.message}`, `${error.step}`, and deep drilling `${error.detail.someField}`
**Alternatives:**
- Mirror Java's sealed hierarchy (YamlError discriminated union + YamlErrorMapper) — over-engineered for TS where failures propagate as Result, not thrown exceptions
- Simple string source only — works for basic cases but blocks `${error.detail.someField}` access patterns
**Rationale:** TS Result.failure already carries `message`, `failureCategory`, and `detail`. No new error type hierarchy needed — just wire existing data to the resolver. ObjectVariableSource enables deep access to structured error detail.
**Trade-offs:** No category validation — any string passes as failureCategory. Mitigated by defining ErrorCategory as a const of the 20 Java values for documentation.
**Sources:** structural-evaluator.ts:170-191 (evaluateTryCatchFinally), variable-resolver.ts:54 (withObjectScope), walker.ts:16 (Result type with detail)
**Exploration:** quick
**Status:** captured

## D3: Annotation parser architecture (H1)

**Choice:** Pre-parse raw YAML text for `#@Name(args)` patterns, build positional map, correlate to steps after YAML parsing — matches Java's AnnotationParser approach
**Alternatives:**
- Custom YAML loader with comment preservation — fragile, parser-dependent, breaks on upgrades
- Embed annotations in YAML values (_annotations: key) — diverges from Java syntax, makes playbooks non-portable
**Rationale:** Annotation syntax is a shared language feature. Both runtimes must parse `#@Name(args)` from YAML comments. Pre-parse scan is simple, parser-independent, and matches the proven Java approach.
**Trade-offs:** Two-pass parsing (text scan + YAML parse). Position correlation requires care with multi-document YAML. Mitigated by Java having already solved the addressing scheme (hierarchical dotted-key: 0, 0.1, 1.0.2).
**Sources:** docs/specs/2026-11-07-ts-platform-parity-audit.md (H1 section), Java AnnotationParser/AnnotationExtractor docs
**Exploration:** quick
**Depends on:** D1 (order — H1 is third in sequence)
**Status:** captured

## D4: Annotation-to-decorator integration (H1)

**Choice:** Annotations resolve to decorator map entries — AnnotationExtractor merges annotation values into the step's `decorators` map before DecoratorChain.build() runs. DecoratorChain itself doesn't change.
**Alternatives:**
- Separate annotation chain (parallel AnnotationChain alongside DecoratorChain) — duplicates logic, risks behaviour divergence between inline and annotation decorators
- DecoratorChain reads annotations directly (modify build() to accept both) — tightly couples chain to annotation system, requires modifying every caller
**Rationale:** Annotations are syntactic sugar for decorators. #@Timeout("5s") should behave identically to `timeout: 5s`. Merging into the existing map guarantees identical execution. InjectionResolver similarly merges @Inject results into params.
**Trade-offs:** Inline decorator values take precedence over annotations (if both specify timeout, inline wins). This matches Java's behaviour and is the expected semantics.
**Sources:** decorator-chain.ts (DecoratorChain.build), walker.ts (step resolution — decorators map populated during resolveOne)
**Exploration:** quick
**Depends on:** D3 (annotation parser produces the annotations that this step consumes)
**Status:** captured

## D5: Test plugin architecture (H4)

**Choice:** Test plugins (assert, assertThat, verify-simulation, run-playbook, validate-playbook) are standard CatalogEntry definitions with Definition + Action. A TestCatalog composes with the main catalog via CompositeCatalog during test execution. assertThat uses absorbSiblings:true so matchers are absorbed into params.
**Alternatives:**
- Structural keywords (add test steps to Walker's RESERVED_KEYS as structural types) — wrong level of abstraction, bloats Walker with test concerns, test actions are plugin-level not control-flow-level
**Rationale:** Test plugins are actions — they take inputs and produce results. The plugin/catalog system already handles this perfectly. absorbSiblings (added in Phase 1 for exactly this use case) enables `assertThat: ${expr}` + `contains: "expected"` without modifying the Walker.
**Trade-offs:** TestCatalog must be explicitly composed during test execution — test assertions in regular playbooks won't resolve. This is correct behaviour (test assertions shouldn't appear in production playbooks).
**Sources:** types.ts:112-120 (Definition with absorbSiblings), catalog.ts (CompositeCatalog pattern), docs/specs audit (H4 — assert, assertThat with 12 matchers)
**Exploration:** quick
**Depends on:** D1 (order — H4 is last in sequence)
**Status:** captured
