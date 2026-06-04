# Connector MCP Tools Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `mcp` submodule that exposes five per-connector MCP tools (Slack, Teams, SMS, WhatsApp, Email) behind a `ConnectorMeshBridge` SPI so any Quarkus app can let LLM agents send notifications directly.

**Architecture:** `ConnectorMeshBridge` SPI + no-op `@DefaultBean` lives in `core` (no new deps). A new `mcp` Maven submodule depends on `core` + `email` + `quarkus-mcp-server-core` and provides five `@ApplicationScoped @Tool` beans, one per connector. Each tool calls `ConnectorService.send()` then `ConnectorMeshBridge.notifyDelivered()` with sanitized content. `WhatsAppConnector` is extended to support template messages via `ConnectorMessage.attributes("templateName")`.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-mcp-server-core 1.11.1, JUnit 5, AssertJ.

---

## File Map

**Modified:**
- `pom.xml` — add `<module>mcp</module>`, add `quarkus-mcp-server.version=1.11.1` property
- `core/src/main/java/io/casehub/connectors/ConnectorService.java:30` — make constructor `public`
- `core/src/main/java/io/casehub/connectors/whatsapp/WhatsAppConnector.java:57` — add template branch in `send()`

**New (core):**
- `core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java`
- `core/src/main/java/io/casehub/connectors/NoOpConnectorMeshBridge.java`
- `core/src/test/java/io/casehub/connectors/NoOpConnectorMeshBridgeTest.java`
- `core/src/test/java/io/casehub/connectors/whatsapp/WhatsAppConnectorTemplateTest.java`

**New (mcp submodule):**
- `mcp/pom.xml`
- `mcp/src/main/java/io/casehub/connectors/mcp/McpContentSanitizer.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/SlackMcpTool.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/TeamsMcpTool.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/TwilioSmsMcpTool.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/WhatsAppMcpTool.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/EmailMcpTool.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/SlackMcpToolTest.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/TeamsMcpToolTest.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/TwilioSmsMcpToolTest.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/WhatsAppMcpToolTest.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/EmailMcpToolTest.java`

---

## Task 1: ConnectorMeshBridge SPI + NoOp + ConnectorService public constructor

**Files:**
- Create: `core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java`
- Create: `core/src/main/java/io/casehub/connectors/NoOpConnectorMeshBridge.java`
- Create: `core/src/test/java/io/casehub/connectors/NoOpConnectorMeshBridgeTest.java`
- Modify: `core/src/main/java/io/casehub/connectors/ConnectorService.java` (constructor visibility)

- [ ] **Step 1: Write the failing test for NoOpConnectorMeshBridge**

`core/src/test/java/io/casehub/connectors/NoOpConnectorMeshBridgeTest.java`:
```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;

class NoOpConnectorMeshBridgeTest {

    private final ConnectorMeshBridge bridge = new NoOpConnectorMeshBridge();

    @Test
    void notifyDelivered_doesNotThrow_forNormalInput() {
        bridge.notifyDelivered("slack", "https://hooks.slack.com/T/B/X", "hello");
    }

    @Test
    void notifyDelivered_doesNotThrow_forNullContent() {
        bridge.notifyDelivered("email", "user@example.com", null);
    }

    @Test
    void notifyDelivered_doesNotThrow_forBlankInputs() {
        bridge.notifyDelivered("", "", "");
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure (classes do not exist yet)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=NoOpConnectorMeshBridgeTest 2>&1 | tail -20
```
Expected: compilation error — `ConnectorMeshBridge` and `NoOpConnectorMeshBridge` not found.

- [ ] **Step 3: Create ConnectorMeshBridge**

`core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java`:
```java
package io.casehub.connectors;

/**
 * SPI — notifies the active mesh implementation that a connector delivery has been
 * dispatched via an MCP tool call.
 *
 * <p>The default implementation ({@link NoOpConnectorMeshBridge}) does nothing.
 * When {@code qhorus/connector-backend} is on the classpath, its implementation
 * activates by classpath presence and posts an {@code EVENT} to the active Qhorus
 * observe channel for the current case session (casehubio/qhorus#249).
 *
 * <h2>Contract for implementations</h2>
 * <ul>
 * <li>Must return quickly — no blocking network I/O on the calling thread.</li>
 * <li>Must tolerate absent case context without throwing — silently no-op when
 *     no active case session is resolvable.</li>
 * <li>Must never throw — exceptions propagate to the MCP tool caller.</li>
 * </ul>
 */
public interface ConnectorMeshBridge {

    /**
     * Called after each MCP-initiated connector dispatch.
     *
     * @param connectorId  connector type id — use the connector's {@code ID} constant,
     *                     e.g. {@link io.casehub.connectors.slack.SlackConnector#ID}
     * @param destination  delivery target: webhook URL, E.164 number, or email address
     * @param content      message body, pre-sanitized and truncated to 500 chars
     */
    void notifyDelivered(String connectorId, String destination, String content);
}
```

- [ ] **Step 4: Create NoOpConnectorMeshBridge**

`core/src/main/java/io/casehub/connectors/NoOpConnectorMeshBridge.java`:
```java
package io.casehub.connectors;

import io.quarkus.arc.DefaultBean;
import io.quarkus.arc.Unremovable;
import jakarta.enterprise.context.ApplicationScoped;

/**
 * Default no-op {@link ConnectorMeshBridge}. Active when no other implementation is on
 * the classpath. Displaced automatically by any {@code @ApplicationScoped} (non-{@code
 * @DefaultBean}) implementation — specifically {@code qhorus/connector-backend} once
 * casehubio/qhorus#249 lands.
 *
 * <p>{@code @Unremovable}: ARC sees no injection point within {@code core} itself;
 * the injection point lives in the {@code mcp} module. Without this annotation, ARC
 * eliminates the bean at augmentation time when core is used without mcp.
 */
@DefaultBean
@Unremovable
@ApplicationScoped
public class NoOpConnectorMeshBridge implements ConnectorMeshBridge {

    @Override
    public void notifyDelivered(final String connectorId,
                                final String destination,
                                final String content) {
        // intentional no-op — Qhorus bridge activates by classpath presence (qhorus#249)
    }
}
```

- [ ] **Step 5: Make ConnectorService constructor public**

In `core/src/main/java/io/casehub/connectors/ConnectorService.java`, change line 30:
```java
// Before:
ConnectorService(@All final List<Connector> connectors) {
// After:
public ConnectorService(@All final List<Connector> connectors) {
```

- [ ] **Step 6: Run tests — all must pass**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`, all tests green including `NoOpConnectorMeshBridgeTest`.

- [ ] **Step 7: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java core/src/main/java/io/casehub/connectors/NoOpConnectorMeshBridge.java core/src/main/java/io/casehub/connectors/ConnectorService.java core/src/test/java/io/casehub/connectors/NoOpConnectorMeshBridgeTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(core): add ConnectorMeshBridge SPI + NoOp default — closes #1 (partial)

- ConnectorMeshBridge SPI for Qhorus bridge activation (qhorus#249 tracks implementation)
- NoOpConnectorMeshBridge @DefaultBean @Unremovable — activates when no bridge present
- ConnectorService constructor made public to enable direct construction in tests

Refs casehubio/connectors#1"
```

---

## Task 2: WhatsApp template support

**Files:**
- Modify: `core/src/main/java/io/casehub/connectors/whatsapp/WhatsAppConnector.java`
- Create: `core/src/test/java/io/casehub/connectors/whatsapp/WhatsAppConnectorTemplateTest.java`

- [ ] **Step 1: Write failing tests**

`core/src/test/java/io/casehub/connectors/whatsapp/WhatsAppConnectorTemplateTest.java`:
```java
package io.casehub.connectors.whatsapp;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.ConnectorMessage;

class WhatsAppConnectorTemplateTest {

    // buildPayload is tested directly without HTTP
    @Test
    void buildTextPayload_noTemplate_producesTextType() {
        String json = WhatsAppConnector.buildPayload("+447700900000", "Hello world", null);
        assertThat(json).contains("\"type\":\"text\"");
        assertThat(json).contains("\"body\":\"Hello world\"");
        assertThat(json).doesNotContain("template");
    }

    @Test
    void buildTemplatePayload_withTemplateName_producesTemplateType() {
        String json = WhatsAppConnector.buildPayload("+447700900000", null, "hello_world");
        assertThat(json).contains("\"type\":\"template\"");
        assertThat(json).contains("\"name\":\"hello_world\"");
        assertThat(json).contains("en_US");
        assertThat(json).doesNotContain("\"type\":\"text\"");
    }

    @Test
    void buildTemplatePayload_blankTemplateName_producesTextType() {
        String json = WhatsAppConnector.buildPayload("+447700900000", "hi", "");
        assertThat(json).contains("\"type\":\"text\"");
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure (buildPayload does not exist yet)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=WhatsAppConnectorTemplateTest 2>&1 | tail -20
```
Expected: compilation error — `buildPayload` not found.

- [ ] **Step 3: Extract buildPayload and add template branch in WhatsAppConnector**

Replace `send()` in `core/src/main/java/io/casehub/connectors/whatsapp/WhatsAppConnector.java`:
```java
@Override
public void send(final ConnectorMessage message) {
    if (apiToken.isBlank() || phoneNumberId.isBlank()) {
        LOG.warning("WhatsAppConnector: casehub.connectors.whatsapp.* not configured — message not sent");
        return;
    }

    final String url = "https://graph.facebook.com/v18.0/" + phoneNumberId + "/messages";
    final String to = message.destination().replaceAll("[^0-9+]", "");
    final String templateName = message.attributes() != null
            ? message.attributes().get("templateName") : null;

    final String json = buildPayload(to, message.body(), templateName);
    final boolean ok = HttpHelper.postJson(url, json, "Authorization", "Bearer " + apiToken);
    if (!ok) {
        LOG.warning("WhatsApp connector failed to: " + to);
    }
}

/**
 * Package-private for unit testing.
 *
 * @param to           E.164 recipient number
 * @param body         message body (used for text messages only)
 * @param templateName if non-blank, produces a template message; otherwise text
 */
static String buildPayload(final String to, final String body, final String templateName) {
    if (templateName != null && !templateName.isBlank()) {
        return "{"
                + "\"messaging_product\":\"whatsapp\","
                + "\"to\":" + HttpHelper.jsonQuote(to) + ","
                + "\"type\":\"template\","
                + "\"template\":{"
                + "\"name\":" + HttpHelper.jsonQuote(templateName) + ","
                + "\"language\":{\"code\":\"en_US\"}"
                + "}"
                + "}";
    }
    final String text = body != null ? body : "";
    return "{"
            + "\"messaging_product\":\"whatsapp\","
            + "\"to\":" + HttpHelper.jsonQuote(to) + ","
            + "\"type\":\"text\","
            + "\"text\":{\"body\":" + HttpHelper.jsonQuote(text) + "}"
            + "}";
}
```

- [ ] **Step 4: Run all core tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add core/src/main/java/io/casehub/connectors/whatsapp/WhatsAppConnector.java core/src/test/java/io/casehub/connectors/whatsapp/WhatsAppConnectorTemplateTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(core): WhatsApp template message support via attributes(templateName)

Adds buildPayload() static method (testable without HTTP) that switches
between text and template JSON based on ConnectorMessage.attributes.
Template messages use en_US language by default.

Refs casehubio/connectors#1"
```

---

## Task 3: mcp submodule scaffold

**Files:**
- Modify: `pom.xml` (root)
- Create: `mcp/pom.xml`

- [ ] **Step 1: Add mcp module to root pom.xml**

In `pom.xml`, add to `<properties>`:
```xml
<quarkus-mcp-server.version>1.11.1</quarkus-mcp-server.version>
```

Add to `<modules>` (after `email-inbound`):
```xml
<module>mcp</module>
```

- [ ] **Step 2: Create mcp/pom.xml**

`mcp/pom.xml`:
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

  <artifactId>casehub-connectors-mcp</artifactId>
  <name>CaseHub Connectors — MCP Tools</name>
  <description>MCP tool surface for casehub-connectors. Exposes send_slack, send_teams,
send_sms, send_whatsapp, and send_email as Quarkus MCP server tools. Add this module and
quarkus-mcp-server-http to any Quarkus app to let LLM agents send notifications directly.
Integrates with Qhorus observe channel when casehub-qhorus-connector-backend is on the
classpath (casehubio/qhorus#249).</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-email</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>
    <dependency>
      <groupId>io.quarkiverse.mcp</groupId>
      <artifactId>quarkus-mcp-server-core</artifactId>
      <version>${quarkus-mcp-server.version}</version>
    </dependency>

    <!-- Testing -->
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
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 3: Verify build resolves**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl mcp --also-make -DskipTests 2>&1 | tail -20
```
Expected: `BUILD SUCCESS` (no sources yet, but pom resolves).

- [ ] **Step 4: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add pom.xml mcp/pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "build(mcp): scaffold casehub-connectors-mcp submodule

Adds mcp/pom.xml with deps on core, email, and quarkus-mcp-server-core 1.11.1.
Consuming apps must add quarkus-mcp-server-http for the MCP transport.

Refs casehubio/connectors#1"
```

---

## Task 4: McpContentSanitizer + shared test support

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/McpContentSanitizer.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java`

- [ ] **Step 1: Write failing tests for McpContentSanitizer**

`mcp/src/test/java/io/casehub/connectors/mcp/McpContentSanitizerTest.java`:
```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class McpContentSanitizerTest {

    @Test
    void sanitize_null_returnsEmpty() {
        assertThat(McpContentSanitizer.sanitize(null)).isEmpty();
    }

    @Test
    void sanitize_stripsNewlines() {
        assertThat(McpContentSanitizer.sanitize("line1\nline2\r\nline3"))
                .isEqualTo("line1 line2  line3");
    }

    @Test
    void sanitize_stripsTabs() {
        assertThat(McpContentSanitizer.sanitize("col1\tcol2")).isEqualTo("col1 col2");
    }

    @Test
    void sanitize_truncatesAt500() {
        String input = "x".repeat(600);
        assertThat(McpContentSanitizer.sanitize(input)).hasSize(500);
    }

    @Test
    void sanitize_under500_noTruncation() {
        assertThat(McpContentSanitizer.sanitize("hello")).isEqualTo("hello");
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=McpContentSanitizerTest 2>&1 | tail -20
```
Expected: compilation error.

- [ ] **Step 3: Create McpContentSanitizer**

`mcp/src/main/java/io/casehub/connectors/mcp/McpContentSanitizer.java`:
```java
package io.casehub.connectors.mcp;

/** Sanitizes content before passing it to {@link io.casehub.connectors.ConnectorMeshBridge}. */
final class McpContentSanitizer {

    private static final int MAX_LENGTH = 500;

    private McpContentSanitizer() {}

    /**
     * Strips control characters that enable log injection and truncates to 500 chars.
     * Newlines, carriage returns, and tabs are replaced with spaces.
     */
    static String sanitize(final String content) {
        if (content == null) return "";
        final String stripped = content
                .replace('\n', ' ')
                .replace('\r', ' ')
                .replace('\t', ' ');
        return stripped.length() > MAX_LENGTH ? stripped.substring(0, MAX_LENGTH) : stripped;
    }
}
```

- [ ] **Step 4: Create shared test support**

`mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java`:
```java
package io.casehub.connectors.mcp;

import java.util.ArrayList;
import java.util.List;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;

/** Shared test doubles for MCP tool unit tests. */
final class McpToolTestSupport {

    private McpToolTestSupport() {}

    /** Records the last call to {@link Connector#send(ConnectorMessage)}. */
    static final class RecordingConnector implements Connector {

        private final String id;
        String lastId;
        ConnectorMessage lastMessage;

        RecordingConnector(final String id) {
            this.id = id;
        }

        @Override
        public String id() {
            return id;
        }

        @Override
        public void send(final ConnectorMessage message) {
            this.lastId = id;
            this.lastMessage = message;
        }

        void reset() {
            lastId = null;
            lastMessage = null;
        }
    }

    /** Records all calls to {@link ConnectorMeshBridge#notifyDelivered}. */
    static final class RecordingBridge implements ConnectorMeshBridge {

        String lastConnectorId;
        String lastDestination;
        String lastContent;

        @Override
        public void notifyDelivered(final String connectorId,
                                    final String destination,
                                    final String content) {
            this.lastConnectorId = connectorId;
            this.lastDestination = destination;
            this.lastContent = content;
        }

        void reset() {
            lastConnectorId = lastDestination = lastContent = null;
        }
    }

    /** Builds a ConnectorService backed only by the given connectors. */
    static ConnectorService serviceWith(final Connector... connectors) {
        return new ConnectorService(List.of(connectors));
    }
}
```

- [ ] **Step 5: Run tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`, `McpContentSanitizerTest` all green.

- [ ] **Step 6: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): McpContentSanitizer + shared test support

Sanitizer strips newlines/tabs (log injection prevention) and truncates
content to 500 chars before passing to ConnectorMeshBridge.
McpToolTestSupport provides RecordingConnector and RecordingBridge for
plain-Java unit tests across all tool classes.

Refs casehubio/connectors#1"
```

---

## Task 5: SlackMcpTool

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/SlackMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/SlackMcpToolTest.java`

- [ ] **Step 1: Write failing tests**

`mcp/src/test/java/io/casehub/connectors/mcp/SlackMcpToolTest.java`:
```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.slack.SlackConnector;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingBridge;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingConnector;

class SlackMcpToolTest {

    private final RecordingConnector connector = new RecordingConnector(SlackConnector.ID);
    private final RecordingBridge bridge = new RecordingBridge();
    private final SlackMcpTool tool = new SlackMcpTool(
            McpToolTestSupport.serviceWith(connector), bridge);

    @BeforeEach
    void reset() {
        connector.reset();
        bridge.reset();
    }

    @Test
    void sendSlack_dispatches_toSlackConnectorWithCorrectMessage() {
        String result = tool.sendSlack(
                "https://hooks.slack.com/services/T/B/X", "Alert", "Server is down");

        assertThat(result).isEqualTo("Dispatched to https://hooks.slack.com/services/T/B/X");
        assertThat(connector.lastMessage.destination())
                .isEqualTo("https://hooks.slack.com/services/T/B/X");
        assertThat(connector.lastMessage.title()).isEqualTo("Alert");
        assertThat(connector.lastMessage.body()).isEqualTo("Server is down");
    }

    @Test
    void sendSlack_callsBridge_withConnectorIdDestinationAndSanitizedContent() {
        tool.sendSlack("https://hooks.slack.com/services/T/B/X", "T", "line1\nline2");

        assertThat(bridge.lastConnectorId).isEqualTo(SlackConnector.ID);
        assertThat(bridge.lastDestination).isEqualTo("https://hooks.slack.com/services/T/B/X");
        assertThat(bridge.lastContent).isEqualTo("line1 line2");
    }

    @Test
    void sendSlack_connectorNotRegistered_returnsFailedString() {
        var emptyService = McpToolTestSupport.serviceWith(); // no connectors
        var failTool = new SlackMcpTool(emptyService, bridge);

        String result = failTool.sendSlack("https://hooks.slack.com/services/T/B/X", "T", "B");

        assertThat(result).startsWith("Failed:");
        assertThat(bridge.lastConnectorId).isNull(); // bridge not called on failure
    }

    @Test
    void sendSlack_longBody_contentTruncatedTo500InBridge() {
        String longBody = "x".repeat(600);
        tool.sendSlack("https://hooks.slack.com/services/T/B/X", "T", longBody);

        assertThat(bridge.lastContent).hasSize(500);
        assertThat(connector.lastMessage.body()).hasSize(600); // original body passed to connector
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=SlackMcpToolTest 2>&1 | tail -20
```
Expected: compilation error — `SlackMcpTool` not found.

- [ ] **Step 3: Create SlackMcpTool**

`mcp/src/main/java/io/casehub/connectors/mcp/SlackMcpTool.java`:
```java
package io.casehub.connectors.mcp;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.slack.SlackConnector;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SlackMcpTool {

    private static final Logger LOG = Logger.getLogger(SlackMcpTool.class);

    private final ConnectorService connectorService;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    SlackMcpTool(final ConnectorService connectorService, final ConnectorMeshBridge meshBridge) {
        this.connectorService = connectorService;
        this.meshBridge = meshBridge;
    }

    @Tool(name = "send_slack",
          description = "Posts a message to a Slack channel via an incoming webhook URL. "
                      + "Returns 'Dispatched to <url>' on success or 'Failed: <reason>' on error. "
                      + "'Dispatched' means the request was sent — not that Slack confirmed delivery.")
    public String sendSlack(
            @ToolArg(description = "Full Slack incoming webhook URL — "
                                 + "starts with https://hooks.slack.com/services/. "
                                 + "This URL is the credential; keep it confidential.")
            final String webhookUrl,
            @ToolArg(description = "Card header / bold title. Use empty string if not needed.")
            final String title,
            @ToolArg(description = "Message text body.")
            final String body) {
        try {
            connectorService.send(SlackConnector.ID, new ConnectorMessage(webhookUrl, title, body));
            meshBridge.notifyDelivered(SlackConnector.ID, webhookUrl,
                    McpContentSanitizer.sanitize(body));
            return "Dispatched to " + webhookUrl;
        } catch (final IllegalArgumentException e) {
            LOG.warnf("send_slack failed: %s", e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 4: Run tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`, all `SlackMcpToolTest` tests green.

- [ ] **Step 5: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): SlackMcpTool — send_slack MCP tool

Dispatches to SlackConnector, sanitizes body before bridge notification.
Connector not registered → returns 'Failed:' string, bridge not called.

Refs casehubio/connectors#1"
```

---

## Task 6: TeamsMcpTool

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/TeamsMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/TeamsMcpToolTest.java`

- [ ] **Step 1: Write failing tests**

`mcp/src/test/java/io/casehub/connectors/mcp/TeamsMcpToolTest.java`:
```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.teams.TeamsConnector;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingBridge;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingConnector;

class TeamsMcpToolTest {

    private final RecordingConnector connector = new RecordingConnector(TeamsConnector.ID);
    private final RecordingBridge bridge = new RecordingBridge();
    private final TeamsMcpTool tool = new TeamsMcpTool(
            McpToolTestSupport.serviceWith(connector), bridge);

    @BeforeEach void reset() { connector.reset(); bridge.reset(); }

    @Test
    void sendTeams_dispatches_withCorrectMessage() {
        String result = tool.sendTeams(
                "https://company.webhook.office.com/webhookb2/...", "Deploy", "v2.3 deployed");

        assertThat(result).startsWith("Dispatched to");
        assertThat(connector.lastMessage.destination())
                .isEqualTo("https://company.webhook.office.com/webhookb2/...");
        assertThat(connector.lastMessage.title()).isEqualTo("Deploy");
        assertThat(connector.lastMessage.body()).isEqualTo("v2.3 deployed");
    }

    @Test
    void sendTeams_callsBridge_withTeamsConnectorId() {
        tool.sendTeams("https://company.webhook.office.com/x", "T", "B");
        assertThat(bridge.lastConnectorId).isEqualTo(TeamsConnector.ID);
    }

    @Test
    void sendTeams_notRegistered_returnsFailedString() {
        var failTool = new TeamsMcpTool(McpToolTestSupport.serviceWith(), bridge);
        assertThat(failTool.sendTeams("https://x.com", "T", "B")).startsWith("Failed:");
        assertThat(bridge.lastConnectorId).isNull();
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=TeamsMcpToolTest 2>&1 | tail -20
```

- [ ] **Step 3: Create TeamsMcpTool**

`mcp/src/main/java/io/casehub/connectors/mcp/TeamsMcpTool.java`:
```java
package io.casehub.connectors.mcp;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.teams.TeamsConnector;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class TeamsMcpTool {

    private static final Logger LOG = Logger.getLogger(TeamsMcpTool.class);

    private final ConnectorService connectorService;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    TeamsMcpTool(final ConnectorService connectorService, final ConnectorMeshBridge meshBridge) {
        this.connectorService = connectorService;
        this.meshBridge = meshBridge;
    }

    @Tool(name = "send_teams",
          description = "Posts an adaptive card message to a Microsoft Teams channel "
                      + "via an incoming webhook URL. "
                      + "Returns 'Dispatched to <url>' on success or 'Failed: <reason>' on error.")
    public String sendTeams(
            @ToolArg(description = "Teams incoming webhook URL — "
                                 + "format: https://<org>.webhook.office.com/webhookb2/...")
            final String webhookUrl,
            @ToolArg(description = "Card title displayed at the top of the message.")
            final String title,
            @ToolArg(description = "Message text body.")
            final String body) {
        try {
            connectorService.send(TeamsConnector.ID, new ConnectorMessage(webhookUrl, title, body));
            meshBridge.notifyDelivered(TeamsConnector.ID, webhookUrl,
                    McpContentSanitizer.sanitize(body));
            return "Dispatched to " + webhookUrl;
        } catch (final IllegalArgumentException e) {
            LOG.warnf("send_teams failed: %s", e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 4: Run tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): TeamsMcpTool — send_teams MCP tool

Refs casehubio/connectors#1"
```

---

## Task 7: TwilioSmsMcpTool

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/TwilioSmsMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/TwilioSmsMcpToolTest.java`

- [ ] **Step 1: Write failing tests**

`mcp/src/test/java/io/casehub/connectors/mcp/TwilioSmsMcpToolTest.java`:
```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.twilio.TwilioSmsConnector;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingBridge;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingConnector;

class TwilioSmsMcpToolTest {

    private final RecordingConnector connector = new RecordingConnector(TwilioSmsConnector.ID);
    private final RecordingBridge bridge = new RecordingBridge();
    private final TwilioSmsMcpTool tool = new TwilioSmsMcpTool(
            McpToolTestSupport.serviceWith(connector), bridge);

    @BeforeEach void reset() { connector.reset(); bridge.reset(); }

    @Test
    void sendSms_dispatches_withE164NumberAsDestination() {
        String result = tool.sendSms("+447700900000", "Your code is 123456");

        assertThat(result).isEqualTo("Dispatched to +447700900000");
        assertThat(connector.lastMessage.destination()).isEqualTo("+447700900000");
        assertThat(connector.lastMessage.body()).isEqualTo("Your code is 123456");
        assertThat(connector.lastMessage.title()).isNull();
    }

    @Test
    void sendSms_callsBridge_withTwilioConnectorIdAndPhoneNumber() {
        tool.sendSms("+447700900000", "Hello");
        assertThat(bridge.lastConnectorId).isEqualTo(TwilioSmsConnector.ID);
        assertThat(bridge.lastDestination).isEqualTo("+447700900000");
    }

    @Test
    void sendSms_notRegistered_returnsFailedString() {
        var failTool = new TwilioSmsMcpTool(McpToolTestSupport.serviceWith(), bridge);
        assertThat(failTool.sendSms("+447700900000", "Hi")).startsWith("Failed:");
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=TwilioSmsMcpToolTest 2>&1 | tail -20
```

- [ ] **Step 3: Create TwilioSmsMcpTool**

`mcp/src/main/java/io/casehub/connectors/mcp/TwilioSmsMcpTool.java`:
```java
package io.casehub.connectors.mcp;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.twilio.TwilioSmsConnector;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class TwilioSmsMcpTool {

    private static final Logger LOG = Logger.getLogger(TwilioSmsMcpTool.class);

    private final ConnectorService connectorService;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    TwilioSmsMcpTool(final ConnectorService connectorService, final ConnectorMeshBridge meshBridge) {
        this.connectorService = connectorService;
        this.meshBridge = meshBridge;
    }

    @Tool(name = "send_sms",
          description = "Sends an SMS message via Twilio. "
                      + "Requires Twilio credentials configured on the server "
                      + "(casehub.connectors.twilio.*). "
                      + "Returns 'Dispatched to <number>' on success or 'Failed: <reason>' on error. "
                      + "Note: a Dispatched response means the request reached Twilio, "
                      + "not that the SMS was delivered to the handset.")
    public String sendSms(
            @ToolArg(description = "Recipient phone number in E.164 format — "
                                 + "must include country code, e.g. +447700900000 or +12125551234.")
            final String to,
            @ToolArg(description = "SMS message body. Max 1600 characters; "
                                 + "longer messages are split by Twilio into concatenated segments.")
            final String body) {
        try {
            connectorService.send(TwilioSmsConnector.ID, new ConnectorMessage(to, body));
            meshBridge.notifyDelivered(TwilioSmsConnector.ID, to,
                    McpContentSanitizer.sanitize(body));
            return "Dispatched to " + to;
        } catch (final IllegalArgumentException e) {
            LOG.warnf("send_sms failed: %s", e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 4: Run tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp 2>&1 | tail -20
```

- [ ] **Step 5: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): TwilioSmsMcpTool — send_sms MCP tool

Refs casehubio/connectors#1"
```

---

## Task 8: WhatsAppMcpTool

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/WhatsAppMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/WhatsAppMcpToolTest.java`

- [ ] **Step 1: Write failing tests**

`mcp/src/test/java/io/casehub/connectors/mcp/WhatsAppMcpToolTest.java`:
```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.whatsapp.WhatsAppConnector;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingBridge;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingConnector;

class WhatsAppMcpToolTest {

    private final RecordingConnector connector = new RecordingConnector(WhatsAppConnector.ID);
    private final RecordingBridge bridge = new RecordingBridge();
    private final WhatsAppMcpTool tool = new WhatsAppMcpTool(
            McpToolTestSupport.serviceWith(connector), bridge);

    @BeforeEach void reset() { connector.reset(); bridge.reset(); }

    @Test
    void sendWhatsApp_noTemplate_sendsTextMessage() {
        String result = tool.sendWhatsApp("+447700900000", "Hello!", null);

        assertThat(result).isEqualTo("Dispatched to +447700900000");
        assertThat(connector.lastMessage.destination()).isEqualTo("+447700900000");
        assertThat(connector.lastMessage.body()).isEqualTo("Hello!");
        assertThat(connector.lastMessage.attributes()).doesNotContainKey("templateName");
    }

    @Test
    void sendWhatsApp_withTemplate_passesTemplateNameInAttributes() {
        tool.sendWhatsApp("+447700900000", null, "hello_world");

        assertThat(connector.lastMessage.attributes()).containsEntry("templateName", "hello_world");
    }

    @Test
    void sendWhatsApp_blankTemplate_treatedAsNoTemplate() {
        tool.sendWhatsApp("+447700900000", "hi", "");

        assertThat(connector.lastMessage.attributes()).doesNotContainKey("templateName");
    }

    @Test
    void sendWhatsApp_callsBridge_withWhatsAppConnectorId() {
        tool.sendWhatsApp("+447700900000", "hi", null);
        assertThat(bridge.lastConnectorId).isEqualTo(WhatsAppConnector.ID);
        assertThat(bridge.lastDestination).isEqualTo("+447700900000");
    }

    @Test
    void sendWhatsApp_notRegistered_returnsFailedString() {
        var failTool = new WhatsAppMcpTool(McpToolTestSupport.serviceWith(), bridge);
        assertThat(failTool.sendWhatsApp("+447700900000", "hi", null)).startsWith("Failed:");
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=WhatsAppMcpToolTest 2>&1 | tail -20
```

- [ ] **Step 3: Create WhatsAppMcpTool**

`mcp/src/main/java/io/casehub/connectors/mcp/WhatsAppMcpTool.java`:
```java
package io.casehub.connectors.mcp;

import java.util.Map;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.whatsapp.WhatsAppConnector;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class WhatsAppMcpTool {

    private static final Logger LOG = Logger.getLogger(WhatsAppMcpTool.class);

    private final ConnectorService connectorService;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    WhatsAppMcpTool(final ConnectorService connectorService, final ConnectorMeshBridge meshBridge) {
        this.connectorService = connectorService;
        this.meshBridge = meshBridge;
    }

    @Tool(name = "send_whatsapp",
          description = "Sends a WhatsApp message via the Meta Cloud API. "
                      + "Requires WhatsApp Business credentials configured on the server "
                      + "(casehub.connectors.whatsapp.*). "
                      + "For recipients outside the 24-hour engagement window, provide templateName "
                      + "with a pre-approved Meta Business Manager template name. "
                      + "Returns 'Dispatched to <number>' on success or 'Failed: <reason>' on error.")
    public String sendWhatsApp(
            @ToolArg(description = "Recipient phone number in E.164 format, e.g. +447700900000.")
            final String to,
            @ToolArg(description = "Message body text. Ignored when templateName is provided.")
            final String body,
            @ToolArg(description = "Optional WhatsApp template name (e.g. 'hello_world'). "
                                 + "Required for first-contact messages or outside the 24-hour window. "
                                 + "Template must be pre-approved in Meta Business Manager. "
                                 + "Language defaults to en_US.",
                     required = false)
            final String templateName) {
        try {
            final Map<String, String> attrs = (templateName != null && !templateName.isBlank())
                    ? Map.of("templateName", templateName)
                    : Map.of();
            connectorService.send(WhatsAppConnector.ID,
                    new ConnectorMessage(to, null, body, attrs));
            meshBridge.notifyDelivered(WhatsAppConnector.ID, to,
                    McpContentSanitizer.sanitize(body));
            return "Dispatched to " + to;
        } catch (final IllegalArgumentException e) {
            LOG.warnf("send_whatsapp failed: %s", e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 4: Run tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp 2>&1 | tail -20
```

- [ ] **Step 5: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): WhatsAppMcpTool — send_whatsapp MCP tool with template support

Passes templateName via ConnectorMessage.attributes when non-blank,
routing to WhatsAppConnector's template branch (added in Task 2).

Refs casehubio/connectors#1"
```

---

## Task 9: EmailMcpTool

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/EmailMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/EmailMcpToolTest.java`

- [ ] **Step 1: Write failing tests**

`mcp/src/test/java/io/casehub/connectors/mcp/EmailMcpToolTest.java`:
```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.email.EmailConnector;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingBridge;
import io.casehub.connectors.mcp.McpToolTestSupport.RecordingConnector;

class EmailMcpToolTest {

    private final RecordingConnector connector = new RecordingConnector(EmailConnector.ID);
    private final RecordingBridge bridge = new RecordingBridge();
    private final EmailMcpTool tool = new EmailMcpTool(
            McpToolTestSupport.serviceWith(connector), bridge);

    @BeforeEach void reset() { connector.reset(); bridge.reset(); }

    @Test
    void sendEmail_dispatches_withCorrectFields() {
        String result = tool.sendEmail("user@example.com", "Deploy complete", "v2.3 is live");

        assertThat(result).isEqualTo("Dispatched to user@example.com");
        assertThat(connector.lastMessage.destination()).isEqualTo("user@example.com");
        assertThat(connector.lastMessage.title()).isEqualTo("Deploy complete");
        assertThat(connector.lastMessage.body()).isEqualTo("v2.3 is live");
    }

    @Test
    void sendEmail_callsBridge_withEmailConnectorId() {
        tool.sendEmail("user@example.com", "S", "B");
        assertThat(bridge.lastConnectorId).isEqualTo(EmailConnector.ID);
        assertThat(bridge.lastDestination).isEqualTo("user@example.com");
    }

    @Test
    void sendEmail_notRegistered_returnsFailedString() {
        var failTool = new EmailMcpTool(McpToolTestSupport.serviceWith(), bridge);
        assertThat(failTool.sendEmail("user@example.com", "S", "B")).startsWith("Failed:");
        assertThat(bridge.lastConnectorId).isNull();
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=EmailMcpToolTest 2>&1 | tail -20
```

- [ ] **Step 3: Create EmailMcpTool**

`mcp/src/main/java/io/casehub/connectors/mcp/EmailMcpTool.java`:
```java
package io.casehub.connectors.mcp;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.email.EmailConnector;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class EmailMcpTool {

    private static final Logger LOG = Logger.getLogger(EmailMcpTool.class);

    private final ConnectorService connectorService;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    EmailMcpTool(final ConnectorService connectorService, final ConnectorMeshBridge meshBridge) {
        this.connectorService = connectorService;
        this.meshBridge = meshBridge;
    }

    @Tool(name = "send_email",
          description = "Sends an email via the SMTP server configured on this app "
                      + "(quarkus.mailer.*). "
                      + "Returns 'Dispatched to <address>' on success or 'Failed: <reason>' on error.")
    public String sendEmail(
            @ToolArg(description = "Recipient email address, e.g. user@example.com.")
            final String to,
            @ToolArg(description = "Email subject line.")
            final String subject,
            @ToolArg(description = "Plain-text email body.")
            final String body) {
        try {
            connectorService.send(EmailConnector.ID, new ConnectorMessage(to, subject, body));
            meshBridge.notifyDelivered(EmailConnector.ID, to,
                    McpContentSanitizer.sanitize(body));
            return "Dispatched to " + to;
        } catch (final IllegalArgumentException e) {
            LOG.warnf("send_email failed: %s", e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 4: Run full build**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -30
```
Expected: `BUILD SUCCESS`, all modules build, all tests green.

- [ ] **Step 5: Commit**
```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): EmailMcpTool — send_email MCP tool

Completes the five-tool MCP surface for casehub-connectors-mcp.

Refs casehubio/connectors#1"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| `ConnectorMeshBridge` SPI in `core` | Task 1 |
| `NoOpConnectorMeshBridge @DefaultBean @Unremovable` | Task 1 |
| `ConnectorService` constructor public (enables testing) | Task 1 |
| WhatsApp template support | Task 2 |
| `mcp` submodule pom | Task 3 |
| `McpContentSanitizer` — strip control chars + truncate 500 | Task 4 |
| `send_slack` with `@Tool` description, bridge call | Task 5 |
| `send_teams` | Task 6 |
| `send_sms` | Task 7 |
| `send_whatsapp` with optional `templateName` | Task 8 |
| `send_email` | Task 9 |
| Security model — documented in tool descriptions | Tasks 5-9 |
| Bridge not called on failure | Tasks 5-9 (tested) |
| Content truncated to 500 before bridge | Task 4 (tested) |
| Original body passed to connector untruncated | Task 5 (tested) |
| qhorus#249 filed (deferred bridge) | Done before planning |

**No placeholders found.** All code is complete.

**Type consistency:** `McpContentSanitizer.sanitize()` used consistently in all tools. `RecordingConnector`/`RecordingBridge` names consistent across all tests. `ConnectorMeshBridge.notifyDelivered(connectorId, destination, content)` signature matches SPI definition throughout.
