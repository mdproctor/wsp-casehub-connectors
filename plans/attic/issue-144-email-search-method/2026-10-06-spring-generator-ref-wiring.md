# Spring Generator Ref Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #145 — Wire spring-generator for platform SPI ref implementations
**Issue group:** #145

**Goal:** Generate Spring `@AutoConfiguration` classes for all 9 platform SPI ref implementations so Spring consumers get the same ref platform beans that Quarkus consumers get via CDI.

**Architecture:** Extend the existing `connectors-spring` module with the `spring-generator` Maven plugin. The plugin scans Jandex indexes from the 9 ref modules and generates auto-configuration classes with `@ConditionalOnClass` for selective activation. One manual config class handles the CDI `Instance<T>` dependency in `ProjectRefBeans`.

**Tech Stack:** Spring Boot 4.1.0, `casehub-platform-spring-generator` Maven plugin, Spring Boot auto-configuration (`@AutoConfiguration`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`)

## Global Constraints

- Spring Boot version: 4.1.0 (matches existing `connectors-spring` module)
- Generator plugin: `io.casehub:casehub-platform-spring-generator:${project.version}`
- Quarkus modules use `<scope>provided</scope>` — compile-time availability without transitive pollution
- Auto-configuration registration via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- The generator merges manual entries from `src/main/resources/` with generated entries in `target/`

---

## Batch 1: Wire spring-generator and verify duality

### Task 1: Add ref module dependencies and spring-generator plugin to connectors-spring

**Files:**
- Modify: `connectors-spring/pom.xml`

**Interfaces:**
- Consumes: existing pom.xml structure, existing `spring-boot.version` property
- Produces: ref module classes available at compile time for generated auto-configs

- [ ] **Step 1: Add provided-scope dependencies for all 9 SPI + ref module pairs**

Add after the existing `spring-boot-starter-test` dependency, before `</dependencies>`:

```xml
    <!-- Platform SPI refs — provided scope for generator scanning + @ConditionalOnClass -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-bank-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-bank-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-calendar-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-calendar-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-commerce-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-commerce-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-contacts-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-contacts-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-document-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-document-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-email-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-email-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-location-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-location-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-project-spi</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-project-ref</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
```

- [ ] **Step 2: Add spring-generator plugin to the build section**

Add after the existing `jandex-maven-plugin` block, before `</plugins>`:

```xml
      <plugin>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-spring-generator</artifactId>
        <version>${project.version}</version>
        <executions>
          <execution>
            <id>generate</id>
            <goals><goal>generate</goal></goals>
            <configuration>
              <quarkusModules>
                <quarkusModule>${project.basedir}/../chat-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../bank-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../calendar-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../commerce-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../contacts-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../document-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../email-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../location-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../project-ref</quarkusModule>
              </quarkusModules>
            </configuration>
          </execution>
          <execution>
            <id>verify-drift</id>
            <goals><goal>verify</goal></goals>
            <phase>verify</phase>
            <configuration>
              <quarkusModules>
                <quarkusModule>${project.basedir}/../chat-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../bank-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../calendar-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../commerce-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../contacts-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../document-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../email-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../location-ref</quarkusModule>
                <quarkusModule>${project.basedir}/../project-ref</quarkusModule>
              </quarkusModules>
            </configuration>
          </execution>
        </executions>
      </plugin>
```

- [ ] **Step 3: Run `mvn compile` on connectors-spring to verify dependencies resolve**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-spring/pom.xml compile`
Expected: BUILD SUCCESS — all 18 new dependencies resolve from GitHub Packages

- [ ] **Step 4: Commit**

```bash
git add connectors-spring/pom.xml
git commit -m "feat(#145): add ref module deps and spring-generator plugin to connectors-spring"
```

### Task 2: Write ProjectRefManualConfig for the CDI-dependent bean

**Files:**
- Create: `connectors-spring/src/main/java/io/casehub/connectors/spring/ProjectRefManualConfig.java`
- Modify: `connectors-spring/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

**Interfaces:**
- Consumes: `ProjectPlatformService(List<ProjectPlatform>)` constructor, `ProjectPlatform` interface from project-spi
- Produces: Spring `@Bean` for `ProjectPlatformService` — the verify goal checks this against the Quarkus `@Produces` method

- [ ] **Step 1: Create ProjectRefManualConfig.java**

Use `ide_create_file` to create:

```java
package io.casehub.connectors.spring;

import io.casehub.connectors.project.spi.ProjectPlatform;
import io.casehub.connectors.project.spi.ProjectPlatformService;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(ProjectPlatformService.class)
public class ProjectRefManualConfig {

    @Bean
    @ConditionalOnMissingBean
    ProjectPlatformService projectPlatformService(ObjectProvider<ProjectPlatform> platforms) {
        return new ProjectPlatformService(platforms.orderedStream().toList());
    }
}
```

- [ ] **Step 2: Add manual config to AutoConfiguration.imports**

Append to `connectors-spring/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
io.casehub.connectors.spring.ProjectRefManualConfig
```

The file should now contain:
```
io.casehub.connectors.spring.ConnectorsAutoConfiguration
io.casehub.connectors.spring.ProjectRefManualConfig
```

The generator merges these manual entries with generated entries in the output.

- [ ] **Step 3: Commit**

```bash
git add connectors-spring/src/main/java/io/casehub/connectors/spring/ProjectRefManualConfig.java
git add connectors-spring/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
git commit -m "feat(#145): add ProjectRefManualConfig for Instance<T> → ObjectProvider mapping"
```

### Task 3: Generate auto-configs and verify duality

**Files:**
- Generated (verify only): `connectors-spring/target/generated-sources/spring-generator/`

**Interfaces:**
- Consumes: Jandex indexes from all 9 ref modules, manual config entries from Task 2
- Produces: Complete Spring auto-configuration — all Quarkus producers have Spring equivalents

- [ ] **Step 1: Run the generator**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-spring/pom.xml generate-sources`
Expected: BUILD SUCCESS — generated auto-configuration classes appear in `target/generated-sources/spring-generator/`

- [ ] **Step 2: Verify generated output**

Check that auto-configuration classes were generated for each ref module:

```bash
find connectors-spring/target/generated-sources/spring-generator -name "*AutoConfiguration.java" -type f
```

Expected: one `*AutoConfiguration.java` per ref module package (8 generated + 0 for project-ref since its generatable bean is in the same scan as the manual one).

- [ ] **Step 3: Inspect one generated class to verify the pattern**

Read one generated auto-config (e.g., the chat-ref one) and verify it contains:
- `@AutoConfiguration`
- `@ConditionalOnClass(RefChatPlatform.class)` (or similar)
- `@Bean @ConditionalOnMissingBean` methods matching the Quarkus `@Produces` methods

- [ ] **Step 4: Run the verify goal**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-spring/pom.xml verify`
Expected: BUILD SUCCESS — `SpringVerifyMojo` confirms all Quarkus `@Produces` return types have matching Spring `@Bean` return types

- [ ] **Step 5: Run full project build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS — all existing tests pass, verify goal passes

- [ ] **Step 6: Commit any adjustments**

If Steps 2-4 revealed issues requiring manual fixes (e.g., additional manual configs, import adjustments), commit those fixes:

```bash
git add connectors-spring/
git commit -m "feat(#145): verify spring-generator output and fix any drift"
```

## References

- [2026-10-06-spring-generator-ref-wiring-design.md] — design spec this plan implements
- [connectors-spring/pom.xml] — existing Spring module pom
- [connectors-spring/ConnectorsAutoConfiguration.java:28] — existing hand-written auto-config
- [platform-spring/pom.xml] — spring-generator plugin configuration pattern
- [SpringGeneratorMojo / SpringVerifyMojo] — generator and verify implementations
- [GitHub #145] — focal issue
- [GitHub parent#469] — dual-framework support epic
- [GitHub #141] — ref CDI normalisation (prerequisite)
