## D1: Simulation processor handling of as() delegation

**Choice:** Detect `PlatformExtensible` super-interface by name and generate a simple `as()` delegation override in the decorator
**Alternatives:**
- Traverse all super-interfaces for all inherited default methods — too broad, risks conflicting with simulation wrapping logic
- Skip processor change, require explicit `as()` on each SPI — defeats the purpose of a shared base interface
**Rationale:** Minimal targeted change with precedent — the processor already has special-case handling for `supports()`. Forward-compatible for future `PlatformExtensible` methods.
**Trade-offs:** Requires updating the processor each time a new method is added to `PlatformExtensible` (unlikely to be frequent)
**Sources:** SimulationDecoratorProcessor.java:216-219 (supports() special case), ChatPlatform.java (SPI pattern), issue #225
**Exploration:** quick
**Status:** captured
