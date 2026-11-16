# Inferred Method-Call Dispatch for Playbook Steps

**Issue:** casehubio/casehub-pages#531
**Branch:** issue-531-inferred-method-dispatch
**Date:** 2026-10-20
**Status:** Draft

## 1. Overview

Add a generic method-call dispatch mechanism so that unrecognised step
names are resolved as method calls on the target element (TS) or bean
(Java), replacing the need for per-method step definitions.

```yaml
# Registered step — keeps special behaviour (progressive typing)
- editor-insert:
    role: textbox
    name: "Editor"
    value: "hello"
    typing: progressive

# Inferred method call — no registration needed
- load-content:
    role: region
    name: "Document diff"
    panel: a
    content: "# Hello"
    label: "v1"
```

### 1.1 Problem

Adding playbook support for a new component today requires:
1. YAML step definitions for every method (`definitions.ts`)
2. Switch cases in the executor (`command-executor.ts`)
3. Corresponding `@PlaybookAction` handlers on the Java side

The Java backend already has reflective dispatch via `ActionRegistry` +
`@PlaybookAction`, but neither runtime supports inferred dispatch — the
TS executor has a raw `invokeMethod` fallback with no security, and the
Java side throws on unknown actions.

### 1.2 Design principles

- **Catalog first** — registered steps always take priority (D0)
- **Interface-based allowlisting** — only methods the component author explicitly declared are callable
- **Plugin extensible** — new components opt in without platform code changes
- **SPI layering** — components depend on primitives, not on catalog internals
- **Cross-runtime parity** — same YAML syntax, same resolution chain, same security principle (explicit opt-in; mechanism differs by language idiom — see §8.1)

## 2. Resolution Chain

A single ordered resolution path for all step names:

| Priority | Resolver | When | Example |
|----------|----------|------|---------|
| 1 | Catalog (explicit) | `catalog.resolve(key)` returns entry | `editor-insert`, `click` |
| 2 | Inferred (component-declared) | `catalog.resolveInferred(key)` returns entry | `load-content`, `next-diff` |
| 3 | Denied | Both return undefined | Hard error |

### 2.1 Catalog interface extension

```typescript
// yaml-core/src/step/walker.ts (Catalog interface lives alongside CatalogEntry, Action, etc.)
interface Catalog {
  resolve(actionName: string): CatalogEntry | undefined;
  resolveInferred?(actionName: string): CatalogEntry | undefined;
  availableActions(): Set<string>;
}
```

`resolveInferred` is optional — catalogs that don't support inferred
dispatch (standalone YAML-only mode) omit it. The Walker falls back
gracefully.

`CompositeCatalog` and `ImportScopedCatalog` must delegate
`resolveInferred` to their underlying sources/delegate. Without this,
inferred dispatch is silently suppressed when these wrappers are in the
resolution path.

`ImportScopedCatalog` delegates to its wrapped catalog:

```typescript
// yaml-core/src/step/catalog.ts — ImportScopedCatalog addition
resolveInferred(actionName: string): CatalogEntry | undefined {
  return this.delegate.resolveInferred?.(actionName);
}
```

`CompositeCatalog` needs a different approach — its sources are
`CatalogSource` (which populate a static map), not `Catalog` objects.
Add an optional `inferredResolver` constructor parameter:

```typescript
// yaml-core/src/step/catalog.ts — CompositeCatalog addition
constructor(
  sources: CatalogSource[],
  private readonly inferredResolver?: (name: string) => CatalogEntry | undefined,
) { /* existing source population */ }

resolveInferred(actionName: string): CatalogEntry | undefined {
  return this.inferredResolver?.(actionName);
}
```

Call sites that build a `CompositeCatalog` pass the scenario catalog's
`resolveInferred` as the second argument. This keeps `CatalogSource`
focused on static entry population.

### 2.2 Walker change

One line in `Walker.resolveOne()`:

```typescript
// yaml-core/src/step/walker.ts, within the key classification loop
} else if (!RESERVED_KEYS.has(key)) {
  const entry = catalog.resolve(key) ?? catalog.resolveInferred?.(key);
  if (entry) {
    actionKey = key;
    actionValue = value;
  } else {
    throw new Error(`unknown step key '${key}' — not a reserved keyword and not found in catalog`);
  }
}
```

The resolved entry produces a standard `PluginStep` — no new
`ResolvedStep` variant. All downstream infrastructure (StructuralEvaluator,
DecoratorChain, step viewer, portability validator) works unchanged.

Note: parameter validation for inferred steps is deferred to runtime
(the `Definition` has permissive inputs). The Walker cannot validate
parameters because the target component — and therefore its declared
parameter schema — is not known at parse time. Typos in step names
are caught at runtime with a clear error ("method X not declared as
playbook-callable"), not at parse time.

### 2.3 InvokeBinding kind

New binding kind for the catalog type system:

```typescript
// yaml-core/src/step/types.ts
interface ElementMethodBinding {
  kind: 'element-method';
  method: string;
}
```

Added to the `InvokeBinding` union. The `AriaInvokeHandler` does not
handle this binding — a new `InferredMethodHandler` does (see §4.2.1).

## 3. TS Discovery Protocol — SPI in pages-primitives

### 3.1 Symbol and types

```typescript
// pages-primitives/src/playbook/playbook-actions.ts

export const PLAYBOOK_ACTIONS: unique symbol =
  Symbol.for('playbook-actions') as any;

export interface PlaybookMethodDescriptor {
  params?: Record<string, string | { type: string; required?: boolean }>;
  returns?: string;
  description?: string;
  portability?: 'universal' | 'java' | 'ts' | 'both';
}

export type PlaybookCapability = Record<string, PlaybookMethodDescriptor>;
```

`Symbol.for('playbook-actions')` is a globally shared symbol. Third-party
web components can implement it without any CaseHub import.

### 3.2 Component implementation

Components declare callable methods as a static getter:

```typescript
// pages-document-diff/src/pages-document-diff.ts
import { PLAYBOOK_ACTIONS, type PlaybookCapability } from '@casehubio/pages-primitives';

@customElement('pages-document-diff')
export class PagesDocumentDiff extends LitElement {
  static get [PLAYBOOK_ACTIONS](): PlaybookCapability {
    return {
      loadContent: {
        params: {
          panel: { type: 'string', required: true },
          content: { type: 'string', required: true },
          label: 'string',
        },
        description: 'Load content into a diff panel',
      },
      nextDiff: { description: 'Navigate to next diff chunk' },
      prevDiff: { description: 'Navigate to previous diff chunk' },
      setViewMode: {
        params: { mode: { type: 'string', required: true } },
        description: 'Switch between split and unified view',
      },
      toggleSync: { description: 'Toggle scroll synchronisation' },
      swapPanels: { description: 'Swap panel A and B' },
      getDiffSummary: { returns: 'object', description: 'Get current diff summary' },
    };
  }

  // ... existing methods unchanged
}
```

Static (class-level) is the default — enables catalog discovery, LSP
completions, and emulator introspection without element instantiation.
Instance-level override is possible for components with dynamic
capabilities.

### 3.3 Dependency flow

```
pages-primitives  ← pages-document-diff (blocks tier)
       ↑
pages-aria (platform) → yaml-core (infrastructure)
```

No upward dependency. Platform does not import from blocks. Components
depend only on primitives for the SPI.

### 3.4 Shorthand params syntax

The `params` record supports both shorthand and detailed forms:

```typescript
// Shorthand — type name only, required defaults to false
loadContent: { params: { panel: 'string', content: 'string' } }

// Detailed — explicit required, type object
loadContent: { params: { panel: { type: 'string', required: true } } }

// Optional suffix shorthand
loadContent: { params: { label: 'string?' } }  // '?' suffix = not required
```

The adapter in pages-aria normalises all forms to the detailed shape
before converting to `Definition`. Normalisation preserves insertion
order (key order IS parameter order — changing it breaks positional
mapping).

### 3.5 Runtime descriptor validation

When `resolvePlaybookCapability` first reads a component's
`[PLAYBOOK_ACTIONS]`, the adapter validates each descriptor:

1. Descriptor param count matches `method.length` on the element
2. All declared method names exist as functions on the element

Mismatches produce a console warning (not a hard error — components
may have optional trailing parameters where `.length` differs).
This catches the most common class of descriptor/signature drift.

## 4. TS Executor Changes

There are two dispatch paths that need inferred method support:

1. **Catalog path** (primary): Walker → `resolveInferred` → PluginStep →
   StructuralEvaluator → `InferredMethodAction.execute()` (§4.2, §4.3).
   This is the normal flow for inferred steps parsed from YAML.

2. **Direct executeStep path** (defence in depth): Code that calls
   `executeStep()` directly (e.g., legacy callers, tests) hits the switch
   default. This must be secure too (§4.1).

All dispatch paths share `callPlaybookMethod` (§4.2) as the single
security-critical function. `resolvePlaybookCapability`, `mapParams`,
and the actual method invocation live in `executor/playbook-resolve.ts`.
Each call site adapts its inputs and wraps the result for its own
return type — but the security check and invocation happen exactly once.

### 4.1 Secure switch default (command-executor.ts)

Replace the current raw `invokeMethod` at line 104 with a secure path.
This is defence in depth — the primary inferred dispatch goes through
the catalog path (§4.3), but any code calling `executeStep()` directly
must also be safe:

```typescript
// command-executor.ts
import { callPlaybookMethod } from './playbook-resolve.js';

default: return invokePlaybookMethod(step);
```

```typescript
async function invokePlaybookMethod(
  step: { action?: string; target?: AriaTarget; [key: string]: unknown }
): Promise<void> {
  if (!step.target) {
    throw new Error(`Inferred method '${step.action}' requires a target`);
  }
  if (!step.action) return;

  const el = resolveTarget(step.target);
  const methodName = kebabToCamel(step.action);
  await callPlaybookMethod(el, methodName, step);
}
```

The function is a thin adapter: resolve target, convert kebab-case,
delegate to `callPlaybookMethod`. No security logic here — it all
lives in the shared core (§4.2).

### 4.2 Shared dispatch core (playbook-resolve.ts)

All dispatch paths converge on a single function that performs the
security check and method invocation. This is the security-critical
code — one implementation, used everywhere:

```typescript
// pages-aria/src/executor/playbook-resolve.ts
import { PLAYBOOK_ACTIONS, type PlaybookCapability } from '@casehubio/pages-primitives';

export function resolvePlaybookCapability(el: Element): PlaybookCapability | null {
  const ctor = el.constructor as any;
  if (PLAYBOOK_ACTIONS in el) return (el as any)[PLAYBOOK_ACTIONS];
  if (ctor && PLAYBOOK_ACTIONS in ctor) return ctor[PLAYBOOK_ACTIONS];
  return null;
}

export async function callPlaybookMethod(
  el: Element,
  methodName: string,
  namedParams: Record<string, unknown>,
): Promise<unknown> {
  const cap = resolvePlaybookCapability(el);
  if (!cap) {
    throw new Error(
      `Target is not playbook-callable — no [PLAYBOOK_ACTIONS] declared`
    );
  }
  const desc = cap[methodName];
  if (!desc) {
    throw new Error(
      `Method '${methodName}' is not declared as playbook-callable. ` +
      `Available: ${Object.keys(cap).join(', ')}`
    );
  }
  const args = mapParams(namedParams, desc);
  const method = (el as any)[methodName];
  if (typeof method !== 'function') {
    throw new Error(
      `Target declares '${methodName}' as playbook-callable but has no such method`
    );
  }
  const result = method.call(el, ...args);
  if (result instanceof Promise) return await result;
  return result;
}

const TARGET_KEYS = new Set(['role', 'name', 'index', 'within']);

export function extractAriaTarget(
  params: Record<string, unknown>,
): AriaTarget | undefined {
  if (!params['role']) return undefined;
  const target: AriaTarget = {
    role: params['role'] as string,
    name: params['name'] as string,
  };
  if (params['index'] != null) target.index = params['index'] as string;
  if (params['within'] != null) target.within = params['within'] as AriaTarget;
  return target;
}

export function mapParams(
  step: Record<string, unknown>,
  desc: PlaybookMethodDescriptor,
): unknown[] {
  if (!desc.params) return [];
  const args: unknown[] = [];
  for (const [key, spec] of Object.entries(desc.params)) {
    if (TARGET_KEYS.has(key)) {
      throw new Error(
        `Parameter '${key}' collides with ARIA target key — ` +
        `use a different name in the descriptor`
      );
    }
    const value = step[key];
    const isRequired = typeof spec === 'object' ? spec.required : !spec?.endsWith?.('?');
    if (isRequired && value === undefined) {
      throw new Error(`Required parameter '${key}' is missing`);
    }
    args.push(value);
  }
  return args;
}

export function wrapResult(result: unknown): Record<string, unknown> {
  if (result == null) return {};
  if (typeof result === 'object' && !Array.isArray(result)) {
    return result as Record<string, unknown>;
  }
  return { result };
}
```

Primitives and arrays are wrapped as `{ result: value }`, matching
the Java-side pattern (`Map.of("result", result)`). Objects pass
through directly.

`callPlaybookMethod` is the single point that performs PLAYBOOK_ACTIONS
resolution, descriptor validation, parameter mapping, and method
invocation. Every dispatch path — catalog action, executeStep default,
server handler default — calls this function. Security fixes apply
once.

### 4.2.1 InferredMethodHandler (InvokeHandler)

`InvokeHandler` for explicit `invoke: { kind: 'element-method' }` specs
in YAML. This is NOT the catalog path — it handles the case where a
YAML author writes an explicit invoke binding rather than using the
inferred step name shorthand:

```typescript
// pages-aria/src/playbook/invoke/inferred-method-handler.ts
export class InferredMethodHandler implements InvokeHandler {
  supports(binding: InvokeBinding): boolean {
    return binding.kind === 'element-method';
  }

  create(_definition: Definition, binding: ElementMethodBinding): Action {
    const methodName = binding.method;
    return {
      async execute(params, services) {
        const target = extractAriaTarget(params);
        if (!target) {
          return stepFailure(`Inferred method '${methodName}' requires a target (role/name)`);
        }
        const el = resolveTarget(target);
        try {
          const result = await callPlaybookMethod(el, methodName, params);
          return stepSuccess(wrapResult(result));
        } catch (err) {
          return stepFailure((err as Error).message);
        }
      }
    };
  }
}
```

### 4.3 InferredMethodAction and catalog factory

`InferredMethodAction` is the `Action` placed on the `CatalogEntry`
returned by `resolveInferred`. The `StructuralEvaluator.evaluatePlugin`
calls `step.entry.action.execute()` — this class is that action.

```typescript
// pages-aria/src/playbook/invoke/inferred-method-action.ts
import { callPlaybookMethod, extractAriaTarget } from '../../executor/playbook-resolve.js';
import { resolveTarget } from '../../executor/command-executor.js';
import type { Action, Result, ServiceRegistry } from '@casehubio/yaml-core/step';
import { stepSuccess, stepFailure } from '@casehubio/yaml-core/step';

export class InferredMethodAction implements Action {
  constructor(private readonly methodName: string) {}

  async execute(params: Record<string, unknown>, _services: ServiceRegistry): Promise<Result> {
    const target = extractAriaTarget(params);
    if (!target) {
      return stepFailure(`Inferred method '${this.methodName}' requires a target (role/name)`);
    }
    const el = resolveTarget(target);
    try {
      const result = await callPlaybookMethod(el, this.methodName, params);
      return stepSuccess(wrapResult(result));
    } catch (err) {
      return stepFailure((err as Error).message);
    }
  }
}
```

The Walker passes YAML step values as flat params — `{ role: 'region',
name: 'Document diff', panel: 'a' }` — not as a pre-built `target`
object. `extractAriaTarget` (shared in playbook-resolve.ts, same pattern
as `AriaInvokeHandler`'s `extractAriaTarget` and `playbook-handler`'s
`buildAriaTarget`) constructs the `AriaTarget` from individual keys.

The catalog factory creates entries using `InferredMethodAction`:

```typescript
// pages-aria/src/playbook/catalog-factory.ts
import { InferredMethodAction } from './invoke/inferred-method-action.js';

export function createScenarioCatalog(): Catalog {
  // ... existing static catalog building ...

  return {
    resolve(name) { return staticEntries.get(name); },
    resolveInferred(name) {
      if (!/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/.test(name)) return undefined;

      const camelName = kebabToCamel(name);
      return {
        qualifiedName: name,
        definition: {
          name,
          inputs: {},   // Permissive — validation at runtime
          outputs: {},
          invoke: { kind: 'element-method', method: camelName },
          portability: 'ts',
        },
        action: new InferredMethodAction(camelName),
      };
    },
    availableActions() { return new Set(staticEntries.keys()); },
  };
}
```

`resolveInferred` accepts kebab-case identifiers only (e.g.,
`load-content`, `next-diff`). Names that don't match this pattern
are rejected, letting the Walker produce a parse-time error for typos
of registered kebab-case actions.

**Parse-time validation tradeoff:** The catalog accepts any
well-formed kebab-case name without checking whether the target
component actually declares that method. This is by design — the
target element is not instantiated during parsing, and scanning the
custom element registry would couple `yaml-core` to the DOM. Typos
in step names (e.g., `lood-content`) pass parsing but fail at runtime
with a clear error: "method 'loadContent' not playbook-callable on
region 'Document diff'". A follow-up validation pass using static
PLAYBOOK_ACTIONS descriptors from registered elements is tracked as
a future enhancement (see §11.5).

### 4.4 playbook-handler.ts extension

The server-dispatched path (`executeAriaAction` switch) gets the same
fallback in its default case. Like §4.1, this is a thin adapter around
the shared core:

```typescript
// playbook-handler.ts, in executeAriaAction()
// executeAriaAction returns void | Promise<void> (NOT async)
import { callPlaybookMethod } from '../executor/playbook-resolve.js';

default: {
  const el = resolveTarget(target!);
  highlightTarget(el);
  const methodName = kebabToCamel(action);
  return callPlaybookMethod(el, methodName, params).then(() => {});
}
```

## 5. Java Discovery Protocol

### 5.1 @PlaybookInterface annotation

```java
// playbook-client/.../PlaybookInterface.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface PlaybookInterface {
}
```

Applied to an interface to mark all its methods as playbook-callable:

```java
@PlaybookInterface
public interface DiffViewer {
    void loadContent(String panel, String content, String label);
    DiffSummary nextDiff();
    void setViewMode(String mode);
    boolean toggleSync();
}
```

### 5.2 ActionRegistry extension

```java
// ActionRegistry.java
public Map<String, Object> invoke(String action, ActionContext ctx) throws Exception {
    // Priority 1: explicit @PlaybookAction handler (registered by kebab-case name)
    ActionHandler handler = handlers.get(action);
    if (handler != null) {
        return handler.invoke(ctx);
    }

    // Priority 2: @PlaybookInterface method (match by camelCase method name)
    String camelAction = kebabToCamel(action);
    return invokeInterfaceMethod(camelAction, action, ctx);
}

private Map<String, Object> invokeInterfaceMethod(
        String camelAction, String originalAction, ActionContext ctx) throws Exception {
    for (Object bean : registeredBeans) {
        for (Class<?> iface : getAllInterfaces(bean.getClass())) {
            if (!iface.isAnnotationPresent(PlaybookInterface.class)) continue;
            for (Method m : iface.getDeclaredMethods()) {
                if (m.isDefault() || Modifier.isStatic(m.getModifiers())) continue;
                if (m.getName().equals(camelAction)) {
                    return invokeReflective(m, bean, ctx);
                }
            }
        }
    }
    throw new IllegalArgumentException(
        "No @PlaybookAction handler or @PlaybookInterface method for: " + originalAction);
}

static String kebabToCamel(String name) {
    if (!name.contains("-")) return name;
    var sb = new StringBuilder();
    boolean upper = false;
    for (int i = 0; i < name.length(); i++) {
        char c = name.charAt(i);
        if (c == '-') { upper = true; continue; }
        sb.append(upper ? Character.toUpperCase(c) : c);
        upper = false;
    }
    return sb.toString();
}

static String camelToKebab(String name) {
    return name.replaceAll("([a-z0-9])([A-Z])", "$1-$2").toLowerCase();
}

private static Set<Class<?>> getAllInterfaces(Class<?> clazz) {
    Set<Class<?>> result = new LinkedHashSet<>();
    for (Class<?> c = clazz; c != null; c = c.getSuperclass()) {
        collectInterfaces(c, result);
    }
    return result;
}

private static void collectInterfaces(Class<?> clazz, Set<Class<?>> result) {
    for (Class<?> iface : clazz.getInterfaces()) {
        if (result.add(iface)) {
            collectInterfaces(iface, result);
        }
    }
}

private Map<String, Object> invokeReflective(
        Method method, Object bean, ActionContext ctx) throws Exception {
    Object[] args = new Object[method.getParameterCount()];
    Parameter[] params = method.getParameters();
    for (int i = 0; i < params.length; i++) {
        args[i] = ctx.data(params[i].getName(), params[i].getType());
    }
    try {
        Object result = method.invoke(bean, args);
        if (result instanceof Map<?, ?> map) {
            return (Map<String, Object>) map;
        }
        return result != null ? Map.of("result", result) : Map.of();
    } catch (InvocationTargetException e) {
        if (e.getCause() instanceof Exception ex) throw ex;
        throw e;
    }
}
```

### 5.2.1 Startup validation

`ActionRegistry.scan()` validates at startup that the `-parameters`
javac flag is active by checking that at least one `@PlaybookInterface`
method has real parameter names (not `arg0`). If validation fails,
throw `IllegalStateException` with a message pointing to the required
compiler flag. This prevents silent data-mapping failures in production.

### 5.3 ActionRegistry.scan() extension

Add `registeredBeans` as an immutable field populated during
construction. The existing `ActionRegistry` is immutable
(`Map.copyOf(handlers)` in the constructor) — the bean list must
follow the same pattern:

```java
public class ActionRegistry {
    private final Map<String, ActionHandler> handlers;
    private final List<Object> registeredBeans;

    private ActionRegistry(Map<String, ActionHandler> handlers, List<Object> registeredBeans) {
        this.handlers = Map.copyOf(handlers);
        this.registeredBeans = List.copyOf(registeredBeans);
    }

    public static ActionRegistry scan(List<Object> beans) {
        var handlers = new HashMap<String, ActionHandler>();
        // ... existing @PlaybookAction scanning ...
        return new ActionRegistry(handlers, beans);
    }

    public List<Object> registeredBeans() {
        return registeredBeans;
    }
}
```

No mutable fields added. The bean list is set once at construction
via the existing factory pattern.

### 5.4 Executor registration

`PlaybookExecutorClient.sendRegister()` includes interface-declared
methods in the action list:

```java
private List<String> discoverActions() {
    List<String> actions = new ArrayList<>(actionRegistry.actions());

    for (Object bean : actionRegistry.registeredBeans()) {
        for (Class<?> iface : getAllInterfaces(bean.getClass())) {
            if (!iface.isAnnotationPresent(PlaybookInterface.class)) continue;
            for (Method m : iface.getDeclaredMethods()) {
                if (m.isDefault() || Modifier.isStatic(m.getModifiers())) continue;
                actions.add(camelToKebab(m.getName()));
            }
        }
    }
    return actions;
}
```

Note: `discoverActions` registers kebab-case action names (converted
from camelCase method names) since the orchestrator dispatches by
kebab-case name. Uses `getAllInterfaces` for consistency with
`invokeInterfaceMethod`, and skips default/static methods.

## 6. Naming Convention

All step names use **kebab-case**, whether registered or inferred:

```yaml
# Registered action
- editor-insert:
    role: textbox
    name: "Editor"
    value: "hello"

# Inferred method call — same naming convention
- load-content:
    role: region
    name: "Document diff"
    panel: a
    content: "# Hello"
    label: "v1"
```

The executor auto-converts kebab-case to camelCase for method lookup:

```typescript
function kebabToCamel(name: string): string {
  return name.replace(/-([a-z0-9])/g, (_, c) => c.toUpperCase());
}
// 'load-content' → 'loadContent'
// 'next-diff' → 'nextDiff'
// 'load-2x' → 'load2x' (digit after hyphen — hyphen removed)
// 'toggleSync' → 'toggleSync' (no-op for names without hyphens)
```

**Why uniform naming:** If an inferred method is later promoted to a
registered action (for better tooling, parameter validation, or special
behaviour), the YAML name stays the same. Promotion/demotion is an
internal implementation change, not a YAML-breaking change.

**Java side:** The same conversion applies. `ActionRegistry` converts
kebab-case action names to camelCase before matching against
`@PlaybookInterface` method names.

**Reserved keyword precedence:** Walker reserved keys always take
precedence over inferred dispatch. The Walker consumes these keys
before catalog resolution — they never reach `resolveInferred`.
Component authors must not name playbook-callable methods after
reserved keywords. If a component has a method called `select`, it
cannot be called via inferred dispatch — it would need an explicit
catalog registration with a different name.

The full reserved set (from `walker.ts:191-211`):

- **Structural:** `step`, `invoke`, `block`, `parallel`, `try`,
  `catch`, `finally`, `select`, `if`, `then`, `else`, `match`,
  `cases`, `barrier`, `quorum`, `race`, `emit-event`, `emit-to`
- **Decorators:** `on-success`, `on-failure`, `forEach`, `loop`,
  `retry`, `timeout`, `delay`, `on-error`, `trigger`, `transform`,
  `signal`, `publish`, `transition`, `semaphore`, `mutex`
- **Target/context:** `label`, `target`, `actor`, `when`, `speed`,
  `content`, `await`, `mode`, `wait`, `from`, `cancel`, `background`
- **Parallel modifiers:** `advance-after`, `cancel-remaining`,
  `collection-timeout`

Of these, the following are plausible method names that component
authors must avoid: `target`, `from`, `wait`, `mode`, `label`,
`delay`, `signal`, `cancel`, `transform`, `loop`, `retry`, `timeout`,
`background`, `content`, `speed`.

## 7. Parameter Mapping

Named YAML parameters are mapped to positional method arguments via the
descriptor's `params` key order.

```yaml
- load-content:
    role: region           # target resolution (stripped)
    name: "Document diff"  # target resolution (stripped)
    panel: a               # → arg 0
    content: "# Hello"     # → arg 1
    label: "v1"            # → arg 2
```

The executor:
1. Converts the step name: `load-content` → `loadContent`
2. Strips target keys (`role`, `name`, `index`, `within`)
3. Iterates the descriptor's `params` keys in declaration order
4. Extracts the corresponding value from the YAML step
5. Passes positionally: `el.loadContent('a', '# Hello', 'v1')`

**Reserved parameter names:** The ARIA target keys `role`, `name`,
`index`, and `within` are present in the params alongside method
parameters (the Walker passes ALL YAML keys under the step). The
`mapParams` function rejects descriptors that declare any of these
as parameter names — a collision would silently pick the ARIA target
value instead of the intended method argument. YAML forbids duplicate
keys, so there is no way to disambiguate. Component authors must
choose different parameter names (e.g., `title` instead of `name`).
The `mapParams` validation throws at dispatch time if a descriptor
violates this constraint.

On Java, `Method.getParameters()` with `-parameters` compile flag
provides the name→position mapping. `ActionContext.data(paramName, paramType)`
extracts typed values.

## 8. Security Model

### 8.1 Contract

Nothing is callable unless explicitly declared by the component author.

| Runtime | Mechanism | Granularity |
|---------|-----------|-------------|
| TS | `[PLAYBOOK_ACTIONS]` Symbol on element class | Per-method in descriptor |
| Java | `@PlaybookInterface` annotation on interface | All methods on interface |

**Granularity difference rationale:** The TS and Java mechanisms have
different granularity, by design. In TS, the `PlaybookCapability`
descriptor explicitly lists each callable method — the element class
has many methods (DOM, LitElement lifecycle, private helpers) and the
descriptor is the filter. In Java, the `@PlaybookInterface` annotation
goes on a purpose-built interface that defines ONLY the playbook
contract. The interface IS the filter — if a method shouldn't be
callable, it doesn't belong on that interface. This is standard Java
idiom (ISP — Interface Segregation Principle). A `DiffViewer` interface
has `loadContent`, `nextDiff`, `setViewMode` — not `hashCode()` or
`toString()`. The bean can have additional methods on other interfaces
or its concrete class; only `@PlaybookInterface`-annotated interface
methods are callable.

Both share the same security principle: nothing is callable unless the
component author explicitly opted it in. The mechanism differs because
TS web components and Java beans have different idioms for expressing
capability boundaries.

### 8.2 What is NOT callable

- **TS:** DOM methods (`remove`, `setAttribute`, `innerHTML`), LitElement
  lifecycle (`connectedCallback`, `requestUpdate`), private/protected
  methods, any method not in the `PlaybookCapability` descriptor.
- **Java:** Methods not on a `@PlaybookInterface`-annotated interface and
  not annotated with `@PlaybookAction`. CDI lifecycle methods, framework
  methods.

### 8.3 Existing invokeMethod replacement

The current `invokeMethod` at `command-executor.ts:117-126` calls
`(el as any)[step.action!]` with no checks. This is replaced by
`invokePlaybookMethod` (§4.1) which enforces the `PLAYBOOK_ACTIONS`
check. The raw path is removed entirely — no code path allows
arbitrary method invocation.

## 9. Portability and Emulator Support

### 9.1 Portability

Inferred TS methods default to `portability: 'ts'`. Inferred Java
methods default to `portability: 'java'`. The `validatePortability()`
function from #506 works unchanged — the `Definition` on the inferred
`CatalogEntry` carries the portability value.

Components can override portability per-method if both runtimes
implement the same interface:

```typescript
static get [PLAYBOOK_ACTIONS](): PlaybookCapability {
  return {
    loadContent: { params: { ... }, portability: 'both' },
  };
}
```

This requires adding an optional `portability` field to
`PlaybookMethodDescriptor`.

### 9.2 Emulator support

The structured descriptor (method name + typed named params + component
identity from ARIA target) gives the emulator enough to simulate.
The emulator looks up a simulation by component type + method name:

```typescript
// Emulator registration
emulator.register('pages-document-diff', 'loadContent', (params) => {
  return { simulated: true, panel: params.panel, status: 'loaded' };
});
```

No special emulator changes needed — the existing structured step
format is sufficient.

## 10. Files Changed

### New files

| File | Package | Description |
|------|---------|-------------|
| `playbook/playbook-actions.ts` | pages-primitives | `PLAYBOOK_ACTIONS` Symbol, `PlaybookCapability`, `PlaybookMethodDescriptor` types |
| `playbook/index.ts` | pages-primitives | Re-export |
| `invoke/inferred-method-handler.ts` | pages-aria | `InferredMethodHandler` — InvokeHandler for explicit `element-method` invoke bindings |
| `invoke/inferred-method-action.ts` | pages-aria | `InferredMethodAction` — Action for catalog-resolved inferred steps |
| `executor/playbook-resolve.ts` | pages-aria | Shared core: `callPlaybookMethod`, `resolvePlaybookCapability`, `mapParams` |
| `PlaybookInterface.java` | playbook-client | `@PlaybookInterface` annotation |

### Modified files

| File | Package | Change |
|------|---------|--------|
| `step/types.ts` | yaml-core | Add `ElementMethodBinding` to `InvokeBinding` union |
| `step/walker.ts` | yaml-core | Add optional `resolveInferred` to `Catalog` interface; use `catalog.resolve(key) ?? catalog.resolveInferred?.(key)` in resolution |
| `step/catalog.ts` | yaml-core | Add `resolveInferred` delegation to `CompositeCatalog` and `ImportScopedCatalog` |
| `executor/command-executor.ts` | pages-aria | Replace `invokeMethod` with `invokePlaybookMethod` (secure) |
| `playbook/catalog-factory.ts` | pages-aria | Implement `resolveInferred` on scenario catalog |
| `playbook/invoke/definitions.ts` | pages-aria | No change (existing actions unaffected) |
| `server/playbook-handler.ts` | pages-aria | Add inferred dispatch to default case |
| `pages-document-diff.ts` | pages-document-diff | Add `[PLAYBOOK_ACTIONS]` static getter (first adopter) |
| `ActionRegistry.java` | playbook-client | Add interface-scanning fallback in `invoke()` |
| `PlaybookExecutorClient.java` | playbook-client | Include interface methods in `executor-register` action list |

### Unchanged packages

**playbook-runtime** (listed in issue #531 scope): No changes required.
`AriaDeliveryHandler.toCompactStep()` constructs the action name from
`data.getOrDefault("action", stepName)` and sends it to the browser.
For inferred steps, the server passes the kebab-case action name
through to the TS `executeAriaAction()` switch, which handles it in
the new default case (§4.4). The runtime is a transparent pass-through
— it doesn't validate or transform action names.

## 11. Non-Goals

1. **Converging the two TS dispatch switches** — command-executor.ts and
   playbook-handler.ts remain separate. Convergence is a future refactor.
2. **Migrating existing editor-* steps** — they keep their explicit
   definitions and special behaviour (progressive typing). They could
   eventually migrate but it's not required.
3. **Dynamic capability changes** — a component's `PLAYBOOK_ACTIONS`
   is read once per dispatch, not observed. Hot-swapping capabilities
   at runtime is not supported.
4. **Automatic catalog population from components** — the catalog does
   NOT pre-scan the DOM for component capabilities. Discovery is lazy
   (at dispatch time via `resolveInferred`).

### 11.5 Future: tooling and static discovery

Inferred actions are invisible to `availableActions()`, the step
catalog browser, LSP completions, and portability pre-flight checks.
This is a deliberate consequence of lazy discovery (§11.4) — the
catalog doesn't pre-scan the DOM for component capabilities.

A future enhancement could add static discovery by scanning registered
custom elements for their `PLAYBOOK_ACTIONS` static getters at catalog
build time, populating `availableActions()` with inferred entries.
This would enable completion, browsing, and pre-flight validation.
Tracked as a follow-up issue.

## 12. Testing Strategy

### 12.1 Unit tests

- **`resolvePlaybookCapability`**: static getter, instance override,
  missing symbol, null constructor, prototype chain
- **`callPlaybookMethod`**: valid method, method not in descriptor,
  no PLAYBOOK_ACTIONS, missing required param, method not a function,
  async method, return value propagation
- **`mapParams`**: all params present, missing optional, missing
  required (throws), empty params, shorthand vs detailed descriptor
- **`kebabToCamel`**: single word, multi-segment, already camelCase

### 12.2 Security tests

- Attempt to call DOM methods (`remove`, `setAttribute`,
  `innerHTML`) — must fail with "not playbook-callable"
- Attempt to call LitElement lifecycle methods
  (`connectedCallback`, `requestUpdate`) — must fail
- Attempt to call methods not in `PLAYBOOK_ACTIONS` descriptor — must fail
- Confirm that removing a method from the descriptor revokes access

### 12.3 Integration tests

- Full dispatch chain: Walker → catalog → InferredMethodAction →
  element method → result propagation via DecoratorChain
- Same step through all three dispatch paths produces consistent
  results (catalog path, executeStep default, playbook-handler default)
- `CompositeCatalog` with `resolveInferred` delegation
- `ImportScopedCatalog` with `resolveInferred` delegation
- Registered action takes priority over inferred method of same name

### 12.4 Java tests

- `ActionRegistry.invoke()` priority: `@PlaybookAction` before
  `@PlaybookInterface`
- Interface method invocation with named parameters
- `-parameters` flag validation at startup (synthetic names → error)
- Method on non-annotated interface → rejection

## References

- `pages-aria/src/executor/command-executor.ts:77-126` — current TS dispatch with raw invokeMethod
- `pages-aria/src/playbook/invoke/definitions.ts` — registered ARIA action definitions
- `pages-aria/src/playbook/catalog-factory.ts` — catalog construction
- `pages-aria/src/server/playbook-handler.ts:562-666` — server dispatch switch
- `pages-editor-core/src/discovery.ts` — EDITABLE_TEXT Symbol pattern (precedent)
- `pages-editor-core/src/types.ts:38-78` — EditableText interface
- `pages-primitives/src/a11y/aria-contract.ts` — AriaTarget type
- `yaml-core/src/step/walker.ts:320-328` — Walker unknown-key error path
- `yaml-core/src/step/walker.ts:49-52` — Catalog interface
- `yaml-core/src/step/catalog.ts` — CompositeCatalog, ImportScopedCatalog
- `yaml-core/src/step/types.ts` — Definition, InvokeBinding types
- `yaml-core/src/step/structural-evaluator.ts` — StructuralEvaluator dispatch
- `playbook-client/ActionRegistry.java` — Java action registry
- `playbook-client/PlaybookAction.java` — existing annotation
- `playbook-client/PlaybookExecutorClient.java` — executor client registration
- `pages-document-diff/src/pages-document-diff.ts:306-549` — motivating component API
- Spec: 2026-09-29-unified-catalog-portability-design.md — portability model
- Spec: 2026-08-20-distributed-executor-protocol-design.md — dispatch protocol
- casehubio/casehub-pages#506 — unified step catalog
- casehubio/casehub-pages#530 — Milkdown editor (motivation)
