# agent-service-spring: Spring Boot Auto-Configuration for @AgentService Proxies

**Issue:** casehubio/platform#996
**Branch:** issue-994-pojo-list-return-types
**Date:** 2026-11-26

## Summary

Create a standalone `agent-service-spring` module that auto-discovers `@AgentService`-annotated interfaces on the classpath and registers their APT-generated proxy classes as Spring beans. Zero consumer boilerplate — adding the starter is sufficient.

## Architecture

### Module: `agent-service-spring`

Two production classes:

**`AgentServiceAutoConfiguration`** (`@AutoConfiguration`)
- `@ConditionalOnClass(AgentServiceBeans.class)` — only activates when agent-service is on the classpath
- `@AutoConfigureAfter` the agent-spring auto-configuration (ensures `AgentProvider` and `AgentResolver` beans exist first)
- Registers `AgentServiceBeans` as a `@Bean` (constructor-injected with `AgentProvider` and `AgentResolver`)
- Registers a static `AgentServiceBeanRegistrar` bean (the post-processor)

**`AgentServiceBeanRegistrar`** (`BeanDefinitionRegistryPostProcessor`)
- Holds a reference to the `BeanFactory` (implements `BeanFactoryAware`)
- In `postProcessBeanDefinitionRegistry()`:
  1. Retrieves base packages from `AutoConfigurationPackages.get(beanFactory)`
  2. Creates a `ClassPathScanningCandidateComponentProvider(false)` with `AnnotationTypeFilter(AgentService.class)` and `includeFilters` set to include interfaces
  3. Scans each base package for `@AgentService` interfaces
  4. For each discovered interface, registers a `GenericBeanDefinition` with:
     - Bean class set to the interface type
     - Instance supplier that lazily resolves `AgentServiceBeans` from the `BeanFactory` and calls `createProxy(interfaceClass)` — post-processors run before regular beans are created, so the supplier defers resolution to bean instantiation time
     - Scope: singleton
     - Lazy-init: false (proxies are lightweight — just constructor injection)

### Dependencies (pom.xml)

| Dependency | Scope | Purpose |
|---|---|---|
| `casehub-platform-agent-api` | compile | `@AgentService`, `AgentProvider`, `AgentResolver` |
| `casehub-platform-agent-service` | compile | `AgentServiceBeans.createProxy()` |
| `spring-boot-autoconfigure` | compile | `@AutoConfiguration`, `AutoConfigurationPackages` |
| `casehub-platform-agent-service-generator` | test (annotation processor) | Generate test proxy classes |
| `spring-boot-test-autoconfigure` | test | `@SpringBootTest` |
| `spring-test` | test | `ApplicationContext` injection |
| `junit-jupiter` | test | Test framework |
| `assertj-core` | test | Assertions |

### Auto-configuration registration

`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:
```
io.casehub.platform.agent.service.spring.AgentServiceAutoConfiguration
```

### Parent pom integration

Add `<module>agent-service-spring</module>` after `agent-service` in the reactor (line ~192).

### Starter integration

Add to `spring-boot-starter-agent/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-service-spring</artifactId>
    <version>${project.version}</version>
</dependency>
```

## Consumer experience

A Spring Boot app using the agent starter:

```java
@AgentService(model = "fast", descriptor = "coach")
public interface Coach {
    @Query("Break ${goal} into tasks")
    String decompose(@Param("goal") String goal);
}

@RestController
class CoachController {
    @Autowired Coach coach;  // auto-discovered and injected

    @PostMapping("/decompose")
    String decompose(@RequestBody String goal) {
        return coach.decompose(goal);
    }
}
```

No `@Bean` method, no `@EnableAgentServices` annotation. The APT processor generates `CoachProxy` at compile time; the auto-configuration discovers `Coach` and wires `CoachProxy` at runtime.

## Testing

**`AgentServiceAutoConfigurationTest`** (`@SpringBootTest`):
1. Define a test `@AgentService` interface in the test sources
2. APT processor generates the proxy at test compile time (via `maven-compiler-plugin` annotation processor path)
3. Provide stub `AgentProvider` and `AgentResolver` as `@TestConfiguration` beans
4. Assert:
   - The `@AgentService` interface bean exists in the context
   - Calling a method on it delegates to the stub `AgentProvider`
   - `AgentServiceBeans` is registered as a bean

## Error handling

- If no `AgentProvider` or `AgentResolver` bean exists, the auto-configuration doesn't activate (`@ConditionalOnClass` + `@AutoConfigureAfter` ordering)
- If an `@AgentService` interface is found but its proxy class is missing (APT processor not configured), `AgentServiceBeans.createProxy()` throws `IllegalStateException` with a clear message pointing to the annotation processor path
- If `AutoConfigurationPackages` is empty (unusual — means no `@SpringBootApplication`), the registrar logs a warning and skips scanning

## Decisions

- D1: Standalone module (not folded into agent-spring)
- D2: `BeanDefinitionRegistryPostProcessor` + `AutoConfigurationPackages` auto-scan (not `@EnableAgentServices` or manual `@Bean`)

See [agent-service-spring-decisions.md](agent-service-spring-decisions.md) for full rationale.

## References

- `agent-service/src/main/java/.../AgentServiceBeans.java` — the generic proxy factory
- `agent-api/src/main/java/.../AgentService.java` — `@Retention(RUNTIME)` annotation
- `agent-spring/pom.xml` — consolidated spring module pattern (spring-generator + hand-written)
- `guardrail-spring/pom.xml` — standalone spring concern module pattern
- `agent-spring/.../AgentGateSpringAutoConfiguration.java` — hand-written `@AutoConfiguration` reference
- `spring-boot-starter-agent/pom.xml` — starter that pulls in agent spring modules
- `spring-integration-test/.../SpringBootCompositionTest.java` — integration test pattern
- `agent-service/target/.../TestCoachProxy.java` — APT-generated proxy structure
