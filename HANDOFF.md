# HANDOFF — casehub-pages

## Last Session

Landed #749 (PlaybookSplitView → ThreeWayViewer consumer, -103 lines) and #750 (PanelController for floating panel state in playbook controller). Brainstormed #747 (conditional @WebSocket endpoint) — design spec committed, implementation not started.

## Immediate Next Step

Invoke `writing-plans` for #747 using the spec below. The brainstorm pipeline completed through spec writing; the work pipeline stopped at `brainstorm_offer` — skip it (spec exists). Then `executing-plans`.

## #747 Implementation Context

**What:** Default `@WebSocket` push endpoint in `backend/push-runtime` with `@UnlessBuildProperty` for conditionality, delegating to the existing `PushRequestHandler` chain.

**Why `@UnlessBuildProperty` not `@DefaultBean`:** `@DefaultBean` resolves CDI injection conflicts, but `@WebSocket` endpoints aren't injected — they're discovered by `WebSocketProcessor.collectEndpoints` iterating `beanStream().classBeans()`. Two `@WebSocket` at the same path = build failure. `@UnlessBuildProperty` excludes the bean from Arc before endpoint scanning sees it.

**Key existing files:**
- `backend/push/src/main/java/io/casehub/pages/push/PushRequestHandler.java` — SPI interface (`handles()` + `handle()`), already exists
- `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/PlaybookPushHandler.java` — existing handler impl (ExecutorRegister, StepResult)
- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/` — existing `@DefaultBean` producers (WebSocketSessionSender, ConnectionRegistry, PushProducers, CdiCommandResultHandler)
- `examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java` — the boilerplate this eliminates (87 lines of standard protocol handling + app-specific Subscribe/Unsubscribe)
- `examples/server/src/main/java/io/casehub/pages/examples/WebSocketSessionSender.java` — duplicate of push-runtime's `@DefaultBean` version, can be deleted

**New files needed:**
- `push-runtime/.../DefaultPushWebSocket.java` — `@WebSocket(path="/ws/push")` + `@UnlessBuildProperty(name="casehub.push.custom-endpoint", stringValue="true")`, injects `Instance<PushRequestHandler>`, handles listen/unlisten directly, delegates rest to handler chain
- `push/.../PushConnectionListener.java` — new SPI interface (`onConnect`/`onDisconnect` default methods) for app cleanup on WebSocket close
- Example app: `DemoDatasetHandler.java` replacing inline Subscribe/Unsubscribe in PushWebSocket

**Three customization levels:** zero-config (default endpoint), add handlers (implement PushRequestHandler), full override (set build property + own @WebSocket).

## References

- `specs/issue-747-conditional-websocket-endpoint/2026-11-26-conditional-websocket-endpoint-design.md` — full spec
- `specs/issue-747-conditional-websocket-endpoint/decisions.md` — D1 approach decision
- `blog/2026-11-26-mdp01-the-controller-pattern-pays-off.md` — session diary (#749/#750)
