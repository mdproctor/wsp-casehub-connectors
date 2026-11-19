# MCP Server Packaging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #324 — MCP server packaging — standalone server for LLM agent frameworks
**Issue group:** #324, #323

**Goal:** Package the platform's @McpDomain tools as a standalone MCP server consumable by any MCP client, starting with connectors' 90+ tools.

**Architecture:** Single Quarkus app reusing the existing platform `mcp` module (`GraphQLModelScanner`, `DynamicToolRegistrar`, `ReflectiveOperationDispatcher`). Two build outputs: JVM JAR (all providers) and GraalVM native image (native-ready providers only). `quarkus-mcp-server` handles MCP protocol. Each domain project contributes a thin config module — no new platform modules beyond `mcp-server`.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-mcp-server 1.11.1, GraalVM (optional native build)

## Global Constraints

- Java 21 source level (`--release 21`). GraalVM 25 for optional native build only.
- All casehubio artifacts are `0.2-SNAPSHOT` (connectors) / `0.4-SNAPSHOT` (platform) resolved from GitHub Packages.
- `@Blocking` required on every `@Tool` method calling `HttpHelper.CLIENT.send()` (protocol: `mcp-tool-blocking-annotation.md`).
- All outbound HTTP via `HttpHelper.CLIENT` singleton (protocol: `shared-http-client.md`).
- Do not add business logic to connectors — pure delivery infrastructure.
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and structural editing.

---

## Batch 1: Platform mcp-server module + flat tool registration

### Task 1: Add flat tool registration mode to DynamicToolRegistrar

The existing `DynamicToolRegistrar` registers `casehub_action` (meta-dispatch) and `casehub_activate` (on-demand individual tools). The standalone server needs all tools registered individually at startup — "flat mode." This task adds a config property to control the registration mode.

**Files:**
- Modify: `platform/mcp/src/main/java/io/casehub/platform/mcp/DynamicToolRegistrar.java`
- Create: `platform/mcp/src/test/java/io/casehub/platform/mcp/DynamicToolRegistrarFlatModeTest.java`

**Interfaces:**
- Consumes: `DomainModelRegistry.getDomains()`, `OperationDescriptor`, `registerOperationTool()` (all existing)
- Produces: Config property `casehub.mcp.tool-exposure` (values: `dispatch` (default), `flat`). When `flat`, all domain operations are registered as individual tools at startup.

- [ ] **Step 1: Write failing test for flat mode**

```java
package io.casehub.platform.mcp;

import io.quarkiverse.mcp.server.ToolManager;
import io.quarkus.test.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(DynamicToolRegistrarFlatModeTest.FlatProfile.class)
class DynamicToolRegistrarFlatModeTest {

    public static class FlatProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("casehub.mcp.tool-exposure", "flat");
        }
    }

    @Inject
    ToolManager toolManager;

    @Test
    void flatModeRegistersIndividualToolsNotCasehubAction() {
        // casehub_action and casehub_activate should NOT be registered
        assertThat(toolManager.getTool("casehub_action")).isNull();
        assertThat(toolManager.getTool("casehub_activate")).isNull();

        // Individual domain tools SHOULD be registered (domains from test classpath)
        // The exact tool names depend on what @McpDomain beans are on the test classpath
        assertThat(toolManager.getTools()).isNotEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=DynamicToolRegistrarFlatModeTest -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: compilation error — config property not recognised / test assertion fails

- [ ] **Step 3: Implement flat mode in DynamicToolRegistrar**

In `DynamicToolRegistrar.java`, add a config property and branch the `onScanComplete` method:

```java
@ConfigProperty(name = "casehub.mcp.tool-exposure", defaultValue = "dispatch")
String toolExposure;
```

In `onScanComplete(@Observes ModelScanComplete event)`, after `registry.discoverAppCapabilities()`:

```java
if ("flat".equals(toolExposure)) {
    registerFlatTools();
    return;
}
// existing casehub_action / casehub_activate registration...
```

Add `registerFlatTools()` method:

```java
private void registerFlatTools() {
    int count = 0;
    for (DomainModel domain : registry.getDomains()) {
        for (OperationDescriptor op : domain.operations()) {
            if (op.type() == OperationDescriptor.OperationType.STREAM
                || op.type() == OperationDescriptor.OperationType.STREAMING_MUTATION) continue;
            String toolName = domain.domain().replace("/", "_") + "_" + op.name();
            registerOperationTool(domain.domain(), op, toolName);
            count++;
        }
    }
    LOG.infof("Flat mode: registered %d individual tools across %d domains",
              count, registry.getDomains().size());
}
```

Use `ide_edit_member` / `ide_insert_member` for the structural edits.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=DynamicToolRegistrarFlatModeTest -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: PASS

- [ ] **Step 5: Verify existing dispatch mode still works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: All existing tests PASS (dispatch is still the default)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/platform add mcp/
git -C /Users/mdproctor/claude/casehub/slots/225/platform commit -m "feat(mcp): add flat tool registration mode for standalone MCP server Refs casehubio/connectors#324"
```

### Task 2: Create platform mcp-server module

This module provides the standalone MCP server runtime: an anonymous `CurrentPrincipal` for standalone use, domain filtering, and default configuration.

**Files:**
- Create: `platform/mcp-server/pom.xml`
- Create: `platform/mcp-server/src/main/java/io/casehub/platform/mcp/server/AnonymousCurrentPrincipal.java`
- Create: `platform/mcp-server/src/main/java/io/casehub/platform/mcp/server/DomainFilter.java`
- Create: `platform/mcp-server/src/main/resources/application.properties`
- Create: `platform/mcp-server/src/test/java/io/casehub/platform/mcp/server/AnonymousCurrentPrincipalTest.java`
- Create: `platform/mcp-server/src/test/java/io/casehub/platform/mcp/server/DomainFilterTest.java`
- Modify: `platform/pom.xml` (add module to reactor)

**Interfaces:**
- Consumes: `CurrentPrincipal` SPI (`io.casehub.platform.api.identity.CurrentPrincipal`), `DynamicToolRegistrar` (via flat mode config)
- Produces: `AnonymousCurrentPrincipal` (`@DefaultBean @ApplicationScoped`) — yields to any real principal provider. `DomainFilter` CDI bean that reads `casehub.mcp.domains` and filters tool registration.

- [ ] **Step 1: Write failing test for AnonymousCurrentPrincipal**

```java
package io.casehub.platform.mcp.server;

import io.casehub.platform.api.identity.CurrentPrincipal;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class AnonymousCurrentPrincipalTest {

    @Test
    void providesAnonymousDefaults() {
        CurrentPrincipal principal = new AnonymousCurrentPrincipal();
        assertThat(principal.actorId()).isEqualTo("anonymous");
        assertThat(principal.groups()).isEmpty();
        assertThat(principal.tenancyId()).isEqualTo("default");
        assertThat(principal.isCrossTenantAdmin()).isFalse();
        assertThat(principal.isAuthenticated()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: compilation error — `AnonymousCurrentPrincipal` doesn't exist

- [ ] **Step 3: Create the pom.xml**

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
    <artifactId>casehub-platform-mcp-server</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform :: MCP Server</name>
    <description>Standalone MCP server runtime — anonymous principal,
        domain filtering, flat tool registration defaults.
        No quarkus:build goal (library module).</description>
    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-mcp</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.eclipse.microprofile.config</groupId>
            <artifactId>microprofile-config-api</artifactId>
            <scope>provided</scope>
        </dependency>
        <!-- Test -->
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
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 4: Implement AnonymousCurrentPrincipal**

```java
package io.casehub.platform.mcp.server;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Set;

@DefaultBean
@ApplicationScoped
public class AnonymousCurrentPrincipal implements CurrentPrincipal {

    @Override public String actorId() { return "anonymous"; }
    @Override public Set<String> groups() { return Set.of(); }
    @Override public String tenancyId() { return TenancyConstants.DEFAULT_TENANT_ID; }
    @Override public boolean isCrossTenantAdmin() { return false; }
}
```

- [ ] **Step 5: Create default application.properties**

```properties
# MCP Server defaults — standalone mode
casehub.mcp.tool-exposure=flat
casehub.mcp.mode=discover
quarkus.http.host=127.0.0.1
```

- [ ] **Step 6: Write failing test for DomainFilter**

```java
package io.casehub.platform.mcp.server;

import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class DomainFilterTest {

    @Test
    void discoverModeAllowsAllDomains() {
        var filter = new DomainFilter("discover", Optional.empty());
        assertThat(filter.isAllowed("connectors/chat")).isTrue();
        assertThat(filter.isAllowed("connectors/bank")).isTrue();
        assertThat(filter.isAllowed("anything")).isTrue();
    }

    @Test
    void explicitModeAllowsOnlyListedDomains() {
        var filter = new DomainFilter("explicit",
                Optional.of("chat,calendar,document"));
        assertThat(filter.isAllowed("connectors/chat")).isTrue();
        assertThat(filter.isAllowed("connectors/calendar")).isTrue();
        assertThat(filter.isAllowed("connectors/document")).isTrue();
        assertThat(filter.isAllowed("connectors/bank")).isFalse();
        assertThat(filter.isAllowed("connectors/health")).isFalse();
    }

    @Test
    void explicitModeMatchesLastSegment() {
        var filter = new DomainFilter("explicit", Optional.of("chat"));
        // "connectors/chat" should match on "chat" (last segment)
        assertThat(filter.isAllowed("connectors/chat")).isTrue();
        assertThat(filter.isAllowed("neocortex/chat")).isTrue();
        assertThat(filter.isAllowed("chat")).isTrue();
    }
}
```

- [ ] **Step 7: Implement DomainFilter**

```java
package io.casehub.platform.mcp.server;

import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.Arrays;
import java.util.Optional;
import java.util.Set;
import java.util.stream.Collectors;

@ApplicationScoped
public class DomainFilter {

    private final boolean discoverMode;
    private final Set<String> allowedDomains;

    public DomainFilter(
            @ConfigProperty(name = "casehub.mcp.mode", defaultValue = "discover") String mode,
            @ConfigProperty(name = "casehub.mcp.domains") Optional<String> domains) {
        this.discoverMode = !"explicit".equalsIgnoreCase(mode);
        this.allowedDomains = domains
                .map(d -> Arrays.stream(d.split(","))
                        .map(String::trim)
                        .filter(s -> !s.isEmpty())
                        .collect(Collectors.toUnmodifiableSet()))
                .orElse(Set.of());
    }

    public boolean isAllowed(String domain) {
        if (discoverMode) return true;
        if (allowedDomains.contains(domain)) return true;
        // Match on last segment: "connectors/chat" matches "chat"
        int slash = domain.lastIndexOf('/');
        if (slash >= 0) {
            return allowedDomains.contains(domain.substring(slash + 1));
        }
        return false;
    }
}
```

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: All PASS

- [ ] **Step 9: Add module to platform reactor pom.xml**

Add `<module>mcp-server</module>` to `platform/pom.xml` `<modules>` section.

- [ ] **Step 10: Wire DomainFilter into DynamicToolRegistrar**

In `DynamicToolRegistrar.java`, inject `DomainFilter` as optional (not all deployments will have mcp-server on classpath):

```java
@Inject
Instance<DomainFilter> domainFilter;
```

In `registerFlatTools()`, filter domains:

```java
private void registerFlatTools() {
    DomainFilter filter = domainFilter.isResolvable() ? domainFilter.get() : null;
    int count = 0;
    for (DomainModel domain : registry.getDomains()) {
        if (filter != null && !filter.isAllowed(domain.domain())) {
            LOG.infof("Skipping domain '%s' — excluded by domain filter", domain.domain());
            continue;
        }
        // ... rest of registration
    }
}
```

- [ ] **Step 11: Run full platform mcp tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp,mcp-server -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: All PASS

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/platform add mcp-server/ mcp/ pom.xml
git -C /Users/mdproctor/claude/casehub/slots/225/platform commit -m "feat(mcp-server): add standalone MCP server module with anonymous principal and domain filtering Refs casehubio/connectors#324"
```

---

## Batch 2: Connectors mcp-server-connectors module

### Task 3: Add @ApplicationScoped to ConnectorOperationsImpl

`ConnectorOperationsImpl` has `@McpDomain` but no CDI scope annotation. It needs `@ApplicationScoped` for the standalone server's CDI container to discover it.

**Files:**
- Modify: `connectors/graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java:65-66`

**Interfaces:**
- Consumes: all 16 constructor parameters (existing — CDI injection)
- Produces: CDI-discoverable `ConnectorOperationsImpl` bean

- [ ] **Step 1: Verify current annotation**

Use `ide_file_structure` on `ConnectorOperationsImpl.java` to confirm it has `@McpDomain` but no `@ApplicationScoped`.

- [ ] **Step 2: Add @ApplicationScoped annotation**

Use `ide_edit_member` to add `@ApplicationScoped` to the class declaration:

```java
@ApplicationScoped
@McpDomain(value = "connectors/operations", app = "connectors", summary = "Connector operational status and management")
public class ConnectorOperationsImpl {
```

Also add the import:
```java
import jakarta.enterprise.context.ApplicationScoped;
```

- [ ] **Step 3: Run existing tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: All PASS — existing tests construct the class manually, not via CDI

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/connectors add graphql/
git -C /Users/mdproctor/claude/casehub/slots/225/connectors commit -m "fix(graphql): add @ApplicationScoped to ConnectorOperationsImpl for CDI discovery Refs #324"
```

### Task 4: Create mcp-server-connectors config module

The thin config module that assembles the standalone MCP server for connectors. This is the module users build and run.

**Files:**
- Create: `connectors/mcp-server-connectors/pom.xml`
- Create: `connectors/mcp-server-connectors/src/main/resources/application.properties`
- Create: `connectors/mcp-server-connectors/src/test/java/io/casehub/connectors/mcp/McpServerStartupTest.java`
- Modify: `connectors/pom.xml` (add module to reactor)

**Interfaces:**
- Consumes: platform `mcp-server` module, platform `mcp` module, connectors `graphql` module, all 13 `*-ref` modules, all 13 `*-spi` modules, `connectors-api`, `core`, `notification-bridge`
- Produces: runnable Quarkus uber-jar (`java -jar casehub-mcp-server-connectors-runner.jar`)

- [ ] **Step 1: Create pom.xml with full dependency tree**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-mcp-server-connectors</artifactId>
    <name>CaseHub MCP Server :: Connectors</name>
    <description>Standalone MCP server exposing connectors' 90+ tools.
        Run with: java -jar target/quarkus-app/quarkus-run.jar</description>

    <dependencies>
        <!-- Platform MCP runtime -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-mcp-server</artifactId>
            <version>0.4-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-mcp</artifactId>
            <version>0.4-SNAPSHOT</version>
        </dependency>

        <!-- Connectors domain APIs (all @McpDomain classes) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-graphql</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- Connector core -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-notification-bridge</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- All 13 ref implementations -->
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-chat-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-calendar-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-bank-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-email-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-document-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-contacts-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-project-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-location-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-commerce-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-travel-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-health-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-voice-ref</artifactId><version>${project.version}</version></dependency>
        <dependency><groupId>io.casehub</groupId><artifactId>casehub-connectors-mobile-ref</artifactId><version>${project.version}</version></dependency>

        <!-- Email connector (required by ConnectorService) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-email</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- Quarkus runtime -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
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
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>native</id>
            <activation>
                <property><name>native</name></property>
            </activation>
            <properties>
                <quarkus.native.enabled>true</quarkus.native.enabled>
                <quarkus.native.resources.includes>seed/*.yaml,seed/*.json</quarkus.native.resources.includes>
            </properties>
        </profile>
    </profiles>
</project>
```

- [ ] **Step 2: Create application.properties**

```properties
# CaseHub MCP Server — Connectors
# Run: java -jar target/quarkus-app/quarkus-run.jar

# MCP transport: stdio (default) or http
quarkus.mcp-server.transport=stdio

# Tool exposure: flat (individual tools) for external MCP clients
casehub.mcp.tool-exposure=flat

# Domain mode: discover (all on classpath) or explicit (filtered)
casehub.mcp.mode=discover
# casehub.mcp.domains=chat,calendar,document,location

# HTTP transport binds to localhost only by default
quarkus.http.host=127.0.0.1
quarkus.http.port=8080

# Disable banner for clean stdio
quarkus.banner.enabled=false
quarkus.log.console.enable=false
```

- [ ] **Step 3: Write startup integration test**

```java
package io.casehub.connectors.mcp;

import io.quarkiverse.mcp.server.ToolManager;
import io.quarkus.test.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class McpServerStartupTest {

    @Inject
    ToolManager toolManager;

    @Test
    void serverStartsAndRegistersConnectorTools() {
        // Flat mode should register individual tools, not casehub_action
        assertThat(toolManager.getTool("casehub_action")).isNull();

        // Verify representative tools from different domains are registered
        // Tool names follow pattern: domain_operationName
        var tools = toolManager.getTools();
        assertThat(tools).isNotEmpty();

        // Should have tools from multiple connector domains
        // Exact names depend on @McpDomain class operation names
        assertThat(tools.size()).isGreaterThan(10);
    }

    @Test
    void refProvidersReturnSeedData() {
        // Call a tool and verify it returns ref data (not an error)
        var calendarTools = toolManager.getTools().stream()
                .filter(t -> t.name().startsWith("connectors_calendar"))
                .findFirst();
        assertThat(calendarTools).isPresent();
    }
}
```

- [ ] **Step 4: Add module to connectors reactor pom.xml**

Add `<module>mcp-server-connectors</module>` to `connectors/pom.xml` `<modules>` section.

- [ ] **Step 5: Build and run the startup test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: Quarkus starts, discovers all @McpDomain beans from graphql module, registers flat tools, test PASSES.

This step will likely require iterating on the dependency tree — missing beans, unsatisfied injection points. Fix each one by adding the required dependency to the pom.xml. The key CDI beans that must be satisfied:
- All 13 `*PlatformService` beans (provided by ref modules)
- `ConnectorService` (from `core`)
- `InboundConnectorService` (from `core`)
- `CurrentPrincipal` (from `mcp-server` AnonymousCurrentPrincipal)
- `SentMessageCapture` (Optional — `@UnlessBuildProfile("prod")`)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/connectors add mcp-server-connectors/ pom.xml
git -C /Users/mdproctor/claude/casehub/slots/225/connectors commit -m "feat(mcp-server): add standalone MCP server config module for connectors Refs #324"
```

---

## Batch 3: Verification and documentation

### Task 5: End-to-end verification — JVM mode

Verify the complete MCP server works as a standalone JAR with stdio transport.

**Files:**
- Modify: `connectors/mcp-server-connectors/src/test/java/io/casehub/connectors/mcp/McpServerStartupTest.java` (add more assertions)

**Interfaces:**
- Consumes: built uber-jar from Task 4
- Produces: verified working MCP server

- [ ] **Step 1: Build the uber-jar**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -pl mcp-server-connectors -DskipTests -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: `target/quarkus-app/quarkus-run.jar` exists

- [ ] **Step 2: Verify the jar starts and responds to tools/list**

Run the jar with a JSON-RPC `tools/list` request piped to stdin:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1"}}}
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | JAVA_HOME=$(/usr/libexec/java_home -v 26) timeout 30 java -jar /Users/mdproctor/claude/casehub/slots/225/connectors/mcp-server-connectors/target/quarkus-app/quarkus-run.jar 2>/dev/null
```

Expected: JSON response containing tool names from all connector domains.

- [ ] **Step 3: Verify tools/call with a ref provider**

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1"}}}
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"connectors_calendar_listCalendars","arguments":{}}}' | JAVA_HOME=$(/usr/libexec/java_home -v 26) timeout 30 java -jar /Users/mdproctor/claude/casehub/slots/225/connectors/mcp-server-connectors/target/quarkus-app/quarkus-run.jar 2>/dev/null
```

Expected: JSON response with ref calendar data (seed data from calendar-ref).

- [ ] **Step 4: Add domain filtering test**

Add a test with explicit mode to verify filtering:

```java
@QuarkusTest
@TestProfile(McpServerDomainFilterTest.ExplicitProfile.class)
class McpServerDomainFilterTest {

    public static class ExplicitProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                "casehub.mcp.mode", "explicit",
                "casehub.mcp.domains", "chat,calendar"
            );
        }
    }

    @Inject
    ToolManager toolManager;

    @Test
    void explicitModeFiltersTools() {
        var tools = toolManager.getTools();
        // Chat and calendar tools should be present
        assertThat(tools.stream().anyMatch(t -> t.name().contains("chat"))).isTrue();
        assertThat(tools.stream().anyMatch(t -> t.name().contains("calendar"))).isTrue();
        // Bank and health tools should NOT be present
        assertThat(tools.stream().noneMatch(t -> t.name().contains("bank"))).isTrue();
        assertThat(tools.stream().noneMatch(t -> t.name().contains("health"))).isTrue();
    }
}
```

Create this as `connectors/mcp-server-connectors/src/test/java/io/casehub/connectors/mcp/McpServerDomainFilterTest.java`.

- [ ] **Step 5: Run all mcp-server-connectors tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/connectors add mcp-server-connectors/
git -C /Users/mdproctor/claude/casehub/slots/225/connectors commit -m "test(mcp-server): add e2e verification and domain filtering tests Refs #324"
```

## References

- [specs/issue-324-mcp-server-packaging/2026-11-19-mcp-server-packaging-design.md] — design spec this plan implements
- [specs/issue-324-mcp-server-packaging/decisions.md] — 10 decisions (D2,D3,D5,D9 superseded)
- [platform/mcp/src/main/java/io/casehub/platform/mcp/DynamicToolRegistrar.java:24] — existing tool registration
- [platform/mcp/src/main/java/io/casehub/platform/mcp/GraphQLModelScanner.java] — existing @McpDomain discovery
- [platform/platform-core/src/main/java/io/casehub/platform/mock/MockCurrentPrincipal.java:11] — existing mock principal (pattern for anonymous principal)
- [connectors/graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java:66] — needs @ApplicationScoped
- [docs/audits/2026-11-19-native-image-readiness.md] — native image readiness audit
- [docs/protocols/connectors/mcp-tool-blocking-annotation.md] — @Blocking protocol
- [docs/protocols/connectors/shared-http-client.md] — HttpHelper.CLIENT protocol
- [GitHub #324] — epic issue
- [GitHub #323] — implementation issue
