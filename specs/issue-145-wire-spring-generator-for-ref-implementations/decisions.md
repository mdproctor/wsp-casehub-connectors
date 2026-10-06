# Decisions — #145 Wire spring-generator for ref implementations

## D1: Module structure

**Choice:** Extend the existing `connectors-spring` module — add the generator plugin and optional ref module dependencies
**Alternatives:**
- New `connectors-ref-spring` module — cleaner separation but adds a module for no consumer benefit
- Per-SPI Spring modules (chat-ref-spring, etc.) — maximum granularity but 9 new modules is excessive
**Rationale:** Platform repos use one `-spring` module per repo. `@ConditionalOnClass` on generated auto-configs handles selectivity — consumers include whichever ref modules they want, auto-configs activate only when the implementation class is on the classpath
**Trade-offs:** `connectors-spring` gains optional dependencies on all 9 ref modules; slightly larger module scope
**Sources:** parent#469 (dual-framework epic), platform-spring module pattern, existing connectors-spring module
**Exploration:** quick
**Status:** captured

## D2: Verification approach

**Choice:** SpringVerifyMojo only — the verify goal compares Quarkus `@Produces` return types against Spring `@Bean` return types, failing the build on drift
**Alternatives:**
- Verify + Spring integration test — higher confidence but requires spring-boot-test dependency and test application context
- Full per-SPI context test — 9 test classes, maximum confidence but excessive for auto-generated beans
**Rationale:** The verify goal catches regressions automatically when new producers are added. The generated beans are mechanical mappings — runtime verification adds weight without proportional value
**Trade-offs:** No runtime proof that beans actually resolve in a Spring context; relies on generator correctness
**Sources:** SpringVerifyMojo in platform spring-generator, platform-spring verify configuration
**Exploration:** quick
**Status:** captured
