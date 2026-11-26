# Decisions — #747 Conditional @WebSocket endpoint mechanism

## D1: How to provide a conditional default WebSocket endpoint

**Choice:** `@UnlessBuildProperty` + `PushRequestHandler` chain
**Alternatives:**
- Quarkus extension with path-conflict resolution — heavyweight; requires custom build step, testing Quarkus deployment pipeline. Overkill for one endpoint path
- Abstract base class with no default endpoint — doesn't achieve zero-Java; apps still need a one-line Java class for the `@WebSocket` annotation
**Rationale:** `@UnlessBuildProperty` is the Quarkus-idiomatic mechanism for conditional beans. Arc excludes the bean before WebSocket endpoint scanning sees it. The PushRequestHandler chain gives three levels of customization: zero-config default, add custom handlers, or full endpoint override.
**Trade-offs:** Apps that need full endpoint override must set a build property (`casehub.push.custom-endpoint=true`). Handler chain adds one level of indirection vs direct method calls in the endpoint.
**Sources:** Quarkus WebSocket Next `WebSocketProcessor.collectEndpoints` — iterates `beanStream().classBeans()` which respects `@UnlessBuildProperty`. PushWebSocket in examples/server shows the standard protocol boilerplate. Research confirmed `@DefaultBean` does not work for `@WebSocket` classes.
**Exploration:** deep-analysis
**Status:** captured
