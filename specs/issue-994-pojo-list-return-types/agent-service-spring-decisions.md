## D1: Module structure

**Choice:** Standalone `agent-service-spring` module
**Alternatives:**
- Folded into `agent-spring` — fewer modules but mixes generated backend configs with hand-written service discovery
**Rationale:** Keeps the auto-discovery concern isolated from the generated agent backend wiring. Follows the precedent set by `guardrail-spring` (separate module for a distinct concern). Added to `spring-boot-starter-agent` for automatic inclusion.
**Trade-offs:** One more module in the reactor
**Sources:** agent-spring/pom.xml (consolidated backend module), guardrail-spring/pom.xml (separate concern module), spring-boot-starter-agent/pom.xml
**Exploration:** quick
**Status:** captured

## D2: Discovery mechanism

**Choice:** `BeanDefinitionRegistryPostProcessor` with `AutoConfigurationPackages`-based classpath scanning
**Alternatives:**
- `@EnableAgentServices` annotation with `ImportBeanDefinitionRegistrar` — more explicit but requires consumer annotation, breaks "just add the starter" pattern
- Manual `@Bean` methods per interface — simplest module but maximum consumer boilerplate (exactly what CDI requires, which we're trying to eliminate)
**Rationale:** Matches Spring Boot's "auto-configure everything" philosophy. `AutoConfigurationPackages` is the standard mechanism (used by JPA auto-config, Spring Data repos). Zero consumer boilerplate — `@AgentService` interfaces on the classpath are automatically wired as Spring beans via their APT-generated proxy classes.
**Trade-offs:** Classpath scanning at startup (negligible — single annotation filter, runs once). First use of `AutoConfigurationPackages` in this codebase — new pattern for contributors to learn.
**Sources:** AgentServiceBeans.java (generic createProxy factory), AgentService.java (@Retention(RUNTIME)), spring-integration-test TestApplication.java
**Exploration:** quick
**Status:** captured
