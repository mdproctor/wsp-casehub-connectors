## D1: Simulation processor handling of as() delegation

**Choice:** Use existing `collectAllMethods(spiClass, index)` to traverse the full super-interface hierarchy, with method-name deduplication — inherited methods like `as()` get standard `generateSimulatedMethod()` wrapping, whose fallthrough path calls `delegate.as(extensionType)` where `this` is the real bean
**Alternatives:**
- Detect `PlatformExtensible` by qualified name string and generate pure delegation — fragile cross-repo coupling, not forward-compatible, the `supports()` "precedent" is architecturally different (active simulation integration vs pure delegation)
- Skip processor change, require explicit `as()` on each SPI — defeats the purpose of a shared base interface
**Rationale:** `collectAllMethods()` already exists in the processor for capability wrapper inner classes. The standard simulation fallthrough path naturally fixes the `this` identity problem. No name-based string detection needed. Handles any future inherited default methods automatically.
**Trade-offs:** Inherited methods become simulation-configurable (via `strategyFor()`), which is a feature not a limitation — consistent with how all other methods work
**Sources:** SimulationDecoratorProcessor.java:594-615 (collectAllMethods/collectMethodsRecursive), SimulationDecoratorProcessor.java:362-457 (generateSimulatedMethod with generic type parameter support), decision review R1-02, issue #225
**Exploration:** quick
**Status:** revised (from name-based detection, after decision review R1-02)
