# agent-service-spring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/platform#996 — agent-service-spring: Spring Boot auto-configuration for @AgentService proxies
**Issue group:** #996

**Goal:** Create a standalone `agent-service-spring` module that auto-discovers `@AgentService` interfaces on the classpath and registers their APT-generated proxies as Spring beans with zero consumer boilerplate.

**Architecture:** A `BeanDefinitionRegistryPostProcessor` scans `AutoConfigurationPackages` for `@AgentService`-annotated interfaces at startup and registers `GenericBeanDefinition`s whose suppliers lazily resolve `AgentServiceBeans.createProxy()` from the `BeanFactory`. An `@AutoConfiguration` class registers both `AgentServiceBeans` and the registrar.

**Tech Stack:** Java 21, Spring Boot 3.x auto-configuration, Maven

## Global Constraints

- Parent version: `0.4-SNAPSHOT`
- Spring Boot version: managed by parent BOM
- Package: `io.casehub.platform.agent.service.spring`
- Artifact ID: `casehub-platform-agent-service-spring`
- All production code is framework-neutral where possible — the registrar and auto-config are the only Spring-coupled classes

---

## Batch 1: Create agent-service-spring module

### Task 1: Module scaffold, production code, and test

**Files:**
- Modify: `platform/agent-service/src/main/java/io/casehub/platform/agent/service/AgentServiceBeans.java` (add 2-arg constructor)
- Modify: `platform/pom.xml:192` (add module entry)
- Create: `platform/agent-service-spring/pom.xml`
- Create: `platform/agent-service-spring/src/main/java/io/casehub/platform/agent/service/spring/AgentServiceBeanRegistrar.java`
- Create: `platform/agent-service-spring/src/main/java/io/casehub/platform/agent/service/spring/AgentServiceAutoConfiguration.java`
- Create: `platform/agent-service-spring/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- Create: `platform/agent-service-spring/src/test/java/io/casehub/platform/agent/service/spring/TestAdvisor.java`
- Create: `platform/agent-service-spring/src/test/java/io/casehub/platform/agent/service/spring/AgentServiceAutoConfigurationTest.java`

**Interfaces:**
- Consumes: `AgentServiceBeans.createProxy(Class<T>)` from `agent-service`, `@AgentService` from `agent-api`, `AgentProvider` and `AgentResolver` from `agent-api`
- Produces: Spring auto-configuration that registers `@AgentService` interface proxies as Spring beans

- [ ] **Step 1: Add 2-arg constructor to AgentServiceBeans**

`AgentServiceBeans` uses CDI field injection (`@Inject`). Spring needs constructor-based wiring. Add an explicit 2-arg constructor alongside the implicit no-arg one.

Modify `platform/agent-service/src/main/java/io/casehub/platform/agent/service/AgentServiceBeans.java`:

```java
package io.casehub.platform.agent.service;

import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentResolver;
import io.casehub.platform.agent.AgentService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.enterprise.inject.spi.InjectionPoint;
import jakarta.inject.Inject;

import java.lang.reflect.Constructor;
import java.lang.reflect.ParameterizedType;

@ApplicationScoped
public class AgentServiceBeans {

    @Inject AgentProvider agentProvider;
    @Inject AgentResolver agentResolver;

    public AgentServiceBeans() {}

    public AgentServiceBeans(AgentProvider agentProvider, AgentResolver agentResolver) {
        this.agentProvider = agentProvider;
        this.agentResolver = agentResolver;
    }

    public <T> T createProxy(Class<T> serviceInterface) {
        String proxyClassName = serviceInterface.getName() + "Proxy";
        try {
            @SuppressWarnings("unchecked")
            Class<T> proxyClass = (Class<T>) serviceInterface.getClassLoader().loadClass(proxyClassName);
            Constructor<T> constructor = proxyClass.getConstructor(AgentProvider.class, AgentResolver.class);
            return constructor.newInstance(agentProvider, agentResolver);
        } catch (ReflectiveOperationException e) {
            throw new IllegalStateException(
                "Failed to create proxy for @AgentService interface " + serviceInterface.getName()
                + ". Ensure agent-service-generator is on the annotation processor path.", e);
        }
    }
}
```

- [ ] **Step 2: Run existing agent-service tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl agent-service test -f platform/pom.xml`
Expected: All existing tests pass — the no-arg constructor preserves CDI compatibility.

- [ ] **Step 3: Create module pom.xml**

Create `platform/agent-service-spring/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-platform-agent-service-spring</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform Agent Service Spring</name>
    <description>Spring Boot auto-configuration for @AgentService proxies — discovers
        annotated interfaces on the classpath and registers their APT-generated proxy
        classes as Spring beans.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-service</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test-autoconfigure</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <executions>
                    <execution>
                        <id>default-testCompile</id>
                        <configuration>
                            <annotationProcessorPaths>
                                <path>
                                    <groupId>io.casehub</groupId>
                                    <artifactId>casehub-platform-agent-service-generator</artifactId>
                                    <version>${project.version}</version>
                                </path>
                                <path>
                                    <groupId>io.casehub</groupId>
                                    <artifactId>casehub-platform-agent-api</artifactId>
                                    <version>${project.version}</version>
                                </path>
                            </annotationProcessorPaths>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 4: Add module to parent pom**

Modify `platform/pom.xml` — add `<module>agent-service-spring</module>` after line 192 (`agent-service`):

```xml
        <module>agent-service-generator</module>
        <module>agent-service</module>
        <module>agent-service-spring</module>
        <module>guardrail-spring</module>
```

- [ ] **Step 5: Write the test @AgentService interface**

Create `platform/agent-service-spring/src/test/java/io/casehub/platform/agent/service/spring/TestAdvisor.java`:

```java
package io.casehub.platform.agent.service.spring;

import io.casehub.platform.agent.AgentService;
import io.casehub.platform.agent.Param;
import io.casehub.platform.agent.Query;

@AgentService(descriptor = "advisor")
public interface TestAdvisor {
    @Query("Advise on ${topic}")
    String advise(@Param("topic") String topic);
}
```

- [ ] **Step 6: Write the failing test**

Create `platform/agent-service-spring/src/test/java/io/casehub/platform/agent/service/spring/AgentServiceAutoConfigurationTest.java`:

```java
package io.casehub.platform.agent.service.spring;

import io.casehub.platform.agent.AgentDescriptor;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentResolver;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.platform.agent.ResolvedAgent;
import io.casehub.platform.agent.service.AgentServiceBeans;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurationPackage;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class AgentServiceAutoConfigurationTest {

    @AutoConfigurationPackage
    static class TestConfig {}

    private final ApplicationContextRunner runner = new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(AgentServiceAutoConfiguration.class))
            .withUserConfiguration(TestConfig.class)
            .withBean(AgentProvider.class, StubAgentProvider::new)
            .withBean(AgentResolver.class, StubAgentResolver::new);

    @Test
    void discoversAndRegistersAgentServiceProxy() {
        runner.run(context -> {
            assertThat(context).hasSingleBean(TestAdvisor.class);
            TestAdvisor advisor = context.getBean(TestAdvisor.class);
            assertThat(advisor.advise("Java")).isEqualTo("stub response");
        });
    }

    @Test
    void registersAgentServiceBeansBean() {
        runner.run(context ->
            assertThat(context).hasSingleBean(AgentServiceBeans.class));
    }

    @Test
    void skipsWhenNoAutoConfigurationPackages() {
        new ApplicationContextRunner()
                .withConfiguration(AutoConfigurations.of(AgentServiceAutoConfiguration.class))
                .withBean(AgentProvider.class, StubAgentProvider::new)
                .withBean(AgentResolver.class, StubAgentResolver::new)
                .run(context ->
                    assertThat(context).doesNotHaveBean(TestAdvisor.class));
    }

    static class StubAgentProvider implements AgentProvider {
        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            return Multi.createFrom().items(
                new AgentEvent.TextDelta("stub response"),
                new AgentEvent.InvocationComplete(5, 3, 0, 0, 0, 0.001, 50, 40, null, 1, false, "stub", null)
            );
        }
        @Override
        public AgentSession openSession(AgentSessionInit init) {
            return null;
        }
    }

    static class StubAgentResolver implements AgentResolver {
        @Override
        public Optional<ResolvedAgent> resolve(String agentId) {
            return Optional.of(new ResolvedAgent(agentId, "You are an advisor.", null, null, null));
        }
        @Override
        public List<AgentDescriptor> discover() {
            return List.of();
        }
    }
}
```

- [ ] **Step 7: Run the test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl agent-service-spring test -f platform/pom.xml`
Expected: Compilation fails — `AgentServiceAutoConfiguration` class not found.

- [ ] **Step 8: Write AgentServiceBeanRegistrar**

Create `platform/agent-service-spring/src/main/java/io/casehub/platform/agent/service/spring/AgentServiceBeanRegistrar.java`:

```java
package io.casehub.platform.agent.service.spring;

import io.casehub.platform.agent.AgentService;
import io.casehub.platform.agent.service.AgentServiceBeans;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.beans.factory.support.GenericBeanDefinition;
import org.springframework.boot.autoconfigure.AutoConfigurationPackages;
import org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider;
import org.springframework.core.type.filter.AnnotationTypeFilter;
import org.springframework.beans.factory.annotation.AnnotatedBeanDefinition;

import java.beans.Introspector;
import java.util.List;
import java.util.logging.Logger;

class AgentServiceBeanRegistrar implements BeanDefinitionRegistryPostProcessor, BeanFactoryAware {

    private static final Logger LOG = Logger.getLogger(
            AgentServiceBeanRegistrar.class.getName());

    private ConfigurableListableBeanFactory beanFactory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        List<String> packages;
        try {
            packages = AutoConfigurationPackages.get(beanFactory);
        } catch (IllegalStateException e) {
            LOG.warning("No auto-configuration packages registered — skipping @AgentService scan");
            return;
        }

        var scanner = new ClassPathScanningCandidateComponentProvider(false) {
            @Override
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                return beanDefinition.getMetadata().isInterface();
            }
        };
        scanner.addIncludeFilter(new AnnotationTypeFilter(AgentService.class));

        for (String basePackage : packages) {
            for (BeanDefinition candidate : scanner.findCandidateComponents(basePackage)) {
                String className = candidate.getBeanClassName();
                try {
                    Class<?> serviceInterface = Class.forName(className);
                    var bd = new GenericBeanDefinition();
                    bd.setBeanClass(serviceInterface);
                    bd.setInstanceSupplier(() ->
                        beanFactory.getBean(AgentServiceBeans.class).createProxy(serviceInterface));
                    String beanName = Introspector.decapitalize(serviceInterface.getSimpleName());
                    registry.registerBeanDefinition(beanName, bd);
                    LOG.info("Registered @AgentService proxy bean: " + beanName
                            + " (" + className + ")");
                } catch (ClassNotFoundException e) {
                    throw new IllegalStateException(
                        "Cannot load @AgentService interface: " + className, e);
                }
            }
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // no-op — all work done in postProcessBeanDefinitionRegistry
    }
}
```

- [ ] **Step 9: Write AgentServiceAutoConfiguration**

Create `platform/agent-service-spring/src/main/java/io/casehub/platform/agent/service/spring/AgentServiceAutoConfiguration.java`:

```java
package io.casehub.platform.agent.service.spring;

import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentResolver;
import io.casehub.platform.agent.service.AgentServiceBeans;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(AgentServiceBeans.class)
@AutoConfigureAfter(name = "io.casehub.platform.agent.gate.spring.AgentGateSpringAutoConfiguration")
public class AgentServiceAutoConfiguration {

    @Bean
    AgentServiceBeans agentServiceBeans(AgentProvider agentProvider, AgentResolver agentResolver) {
        return new AgentServiceBeans(agentProvider, agentResolver);
    }

    @Bean
    static AgentServiceBeanRegistrar agentServiceBeanRegistrar() {
        return new AgentServiceBeanRegistrar();
    }
}
```

- [ ] **Step 10: Create auto-configuration imports file**

Create `platform/agent-service-spring/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
io.casehub.platform.agent.service.spring.AgentServiceAutoConfiguration
```

- [ ] **Step 11: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl agent-service-spring test -f platform/pom.xml`
Expected: All 3 tests pass — proxy is auto-discovered, `AgentServiceBeans` is registered, graceful skip when no packages.

- [ ] **Step 12: Add to spring-boot-starter-agent**

Modify `platform/spring-boot-starter-agent/pom.xml` — add dependency after `agent-spring`:

```xml
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-service-spring</artifactId>
            <version>${project.version}</version>
        </dependency>
```

- [ ] **Step 13: Verify full reactor build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl agent-service,agent-service-spring,spring-boot-starter-agent install -f platform/pom.xml`
Expected: All modules build and install successfully.

- [ ] **Step 14: Commit**

```bash
git add agent-service/src/main/java/io/casehub/platform/agent/service/AgentServiceBeans.java \
       agent-service-spring/ \
       pom.xml \
       spring-boot-starter-agent/pom.xml
git commit -m "feat(#996): agent-service-spring — Spring Boot auto-configuration for @AgentService proxies

Add BeanDefinitionRegistryPostProcessor that scans AutoConfigurationPackages
for @AgentService interfaces and registers their APT-generated proxies as
Spring beans. Zero consumer boilerplate — just add the starter.

Refs casehubio/platform#996"
```

## References

- [2026-11-26-agent-service-spring-design.md](../specs/issue-994-pojo-list-return-types/2026-11-26-agent-service-spring-design.md) — design spec this plan implements
- `agent-service/src/main/java/.../AgentServiceBeans.java` — proxy factory (modified: add 2-arg constructor)
- `agent-api/src/main/java/.../AgentService.java` — `@Retention(RUNTIME)` annotation
- `agent-spring/.../AgentGateSpringAutoConfiguration.java` — `@AutoConfiguration` reference pattern
- `agent-spring/.../AgentGateBeanPostProcessorTest.java` — `ApplicationContextRunner` test pattern
- `agent-service/.../AgentServiceBeansTest.java` — CDI test with stub provider/resolver
- `spring-boot-starter-agent/pom.xml` — starter integration point
- casehubio/platform#996 — focal issue
