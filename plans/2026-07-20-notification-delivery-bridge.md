# Notification Delivery Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #86 — feat: notification delivery bridge
**Issue group:** #86

**Goal:** Bridge the platform notification system to the connectors SPI so any
Connector CDI bean automatically becomes a notification delivery channel.

**Architecture:** Three changes across two repos: (1) add `DestinationResolver`
SPI and `WHATSAPP` constant to `casehub-platform-api`, (2) change `Connector.send()`
from void to boolean and add `channelType()` in `connectors-core`, (3) new
`notification-bridge` module that wires them together at startup.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`@ApplicationScoped`, `@Startup`,
`@All`), JUnit 5, AssertJ, WireMock

## Global Constraints

- Java source level: 21 (running on Java 26 JVM)
- All casehubio artifacts: `0.2-SNAPSHOT`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- HTTP clients: `HttpHelper.CLIENT` singleton (protocol: `shared-http-client.md`)
- connectors-core has no platform dependency — string literals, not `DeliveryChannels` constants
- platform-api is zero-dependency (no Quarkus, no JPA, no casehubio deps — pure Java + mutiny provided)

---

### Task 1: Platform-api — DestinationResolver SPI + WHATSAPP constant

**Repo:** `casehub-platform` (`/Users/mdproctor/claude/casehub/platform`)

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/delivery/DestinationResolver.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/delivery/DeliveryChannels.java`
- Create: `platform-api/src/test/java/io/casehub/platform/api/delivery/DestinationResolverContractTest.java`

**Interfaces:**
- Produces: `DestinationResolver { String channelId(); Optional<String> resolve(String userId, String tenancyId); }` — consumed by Task 4 (notification-bridge)
- Produces: `DeliveryChannels.WHATSAPP = "whatsapp"` — consumed by Task 4

- [ ] **Step 1: Write the DestinationResolver contract test**

```java
package io.casehub.platform.api.delivery;

import org.junit.jupiter.api.Test;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

class DestinationResolverContractTest {

    @Test
    void resolve_returnsDestinationForKnownUser() {
        DestinationResolver resolver = new DestinationResolver() {
            @Override
            public String channelId() { return "email"; }

            @Override
            public Optional<String> resolve(String userId, String tenancyId) {
                if ("user-1".equals(userId)) return Optional.of("user1@example.com");
                return Optional.empty();
            }
        };

        assertThat(resolver.channelId()).isEqualTo("email");
        assertThat(resolver.resolve("user-1", "tenant-1")).contains("user1@example.com");
    }

    @Test
    void resolve_returnsEmptyForUnknownUser() {
        DestinationResolver resolver = new DestinationResolver() {
            @Override
            public String channelId() { return "sms"; }

            @Override
            public Optional<String> resolve(String userId, String tenancyId) {
                return Optional.empty();
            }
        };

        assertThat(resolver.resolve("unknown", "tenant-1")).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -Dtest=DestinationResolverContractTest -f /Users/mdproctor/claude/casehub/platform/pom.xml
```

Expected: compilation failure — `DestinationResolver` not found

- [ ] **Step 3: Create DestinationResolver interface**

File: `platform-api/src/main/java/io/casehub/platform/api/delivery/DestinationResolver.java`

```java
package io.casehub.platform.api.delivery;

import java.util.Optional;

/**
 * SPI for resolving a user's delivery destination for a specific channel.
 *
 * <p>Implementations are {@code @ApplicationScoped} CDI beans discovered automatically,
 * one per channel type. The bridge matches resolvers to deliverers by {@link #channelId()}.
 *
 * <p>Resolution models vary by channel:
 * <ul>
 *   <li><b>Per-user</b> (email, SMS, WhatsApp): resolves to the user's contact attribute</li>
 *   <li><b>Per-tenant</b> (future — Slack, Teams): resolves to a shared webhook URL</li>
 * </ul>
 */
public interface DestinationResolver {

    /**
     * Channel type this resolver handles.
     *
     * @return channel type identifier; never null or blank
     */
    String channelId();

    /**
     * Resolve the delivery destination for a user on this channel.
     *
     * @param userId    notification recipient
     * @param tenancyId tenant isolation
     * @return destination string if known, empty otherwise
     */
    Optional<String> resolve(String userId, String tenancyId);
}
```

- [ ] **Step 4: Add WHATSAPP constant to DeliveryChannels**

Use `ide_edit_member` to add the constant to `DeliveryChannels`:

```java
public static final String WHATSAPP = "whatsapp";
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -Dtest=DestinationResolverContractTest -f /Users/mdproctor/claude/casehub/platform/pom.xml
```

Expected: PASS

- [ ] **Step 6: Run full platform-api test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -f /Users/mdproctor/claude/casehub/platform/pom.xml
```

Expected: all tests pass (DeliveryChannelDescriptorTest, NotificationDelivererContractTest, etc.)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/main/java/io/casehub/platform/api/delivery/DestinationResolver.java platform-api/src/main/java/io/casehub/platform/api/delivery/DeliveryChannels.java platform-api/src/test/java/io/casehub/platform/api/delivery/DestinationResolverContractTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(api): add DestinationResolver SPI and WHATSAPP channel constant — #86"
```

- [ ] **Step 8: Install to local Maven repo**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl platform-api -f /Users/mdproctor/claude/casehub/platform/pom.xml -DskipTests
```

This makes the updated `casehub-platform-api` available for Task 4's dependency resolution.

---

### Task 2: Connector SPI changes — send() → boolean + channelType()

**Repo:** `casehub-connectors` (`/Users/mdproctor/claude/casehub/connectors`)

**Files:**
- Modify: `core/src/main/java/io/casehub/connectors/Connector.java` — `send()` return type + `channelType()`
- Modify: `core/src/main/java/io/casehub/connectors/ConnectorService.java` — `send()` return type
- Modify: `core/src/main/java/io/casehub/connectors/slack/SlackConnector.java` — boolean return + `channelType() → null`
- Modify: `core/src/main/java/io/casehub/connectors/teams/TeamsConnector.java` — boolean return + `channelType() → null`
- Modify: `core/src/main/java/io/casehub/connectors/twilio/TwilioSmsConnector.java` — boolean return + `channelType() → "sms"`
- Modify: `core/src/main/java/io/casehub/connectors/whatsapp/WhatsAppConnector.java` — boolean return
- Modify: `email/src/main/java/io/casehub/connectors/email/EmailConnector.java` — boolean return
- Modify: `core/src/test/java/io/casehub/connectors/ConnectorServiceTest.java` — RecordingConnector + tests
- Modify: `mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java` — RecordingConnector

**Interfaces:**
- Consumes: nothing (SPI-level change)
- Produces: `Connector.send()` returns `boolean`; `Connector.channelType()` returns `String` (nullable); `ConnectorService.send()` returns `boolean` — all consumed by Task 4

- [ ] **Step 1: Write tests for channelType() default and send() boolean return**

Add to `ConnectorTest.java` using `ide_insert_member`:

```java
@Test
void channelType_defaultsToId() {
    Connector connector = new Connector() {
        @Override public String id() { return "my-connector"; }
        @Override public boolean send(ConnectorMessage message) { return true; }
    };
    assertThat(connector.channelType()).isEqualTo("my-connector");
}

@Test
void channelType_canBeOverriddenToNull() {
    Connector connector = new Connector() {
        @Override public String id() { return "slack"; }
        @Override public boolean send(ConnectorMessage message) { return true; }
        @Override public String channelType() { return null; }
    };
    assertThat(connector.channelType()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=ConnectorTest#channelType_defaultsToId+channelType_canBeOverriddenToNull -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: compilation failure — `channelType()` not defined, `send()` return type mismatch

- [ ] **Step 3: Change Connector interface**

Use `ide_edit_member` on `Connector.java`:

Change `send()`:
```java
/**
 * Send a message via this connector.
 *
 * @param message the message to deliver; must not be null
 * @return true if delivery succeeded, false on failure
 */
boolean send(ConnectorMessage message);
```

Add `channelType()` using `ide_insert_member` after `send`:
```java
/**
 * Notification channel type this connector implements.
 *
 * <p>Defaults to {@link #id()}. Override to map to a different channel type
 * (e.g., {@code "sms"} for a Twilio connector with id {@code "twilio-sms"}).
 * Return {@code null} to opt out of notification bridging.
 *
 * @return channel type string, or null to opt out
 */
default String channelType() { return id(); }
```

- [ ] **Step 4: Update ConnectorService.send() return type**

Use `ide_edit_member` on `ConnectorService.send()`:

```java
/**
 * Send a message via the named connector.
 *
 * @param connectorId id of the connector to use (e.g. {@code "slack"})
 * @param message     the message to deliver
 * @return true if the connector reported success
 * @throws IllegalArgumentException if no connector is registered for {@code connectorId}
 */
public boolean send(final String connectorId, final ConnectorMessage message) {
    final Connector connector = registry.get(connectorId);
    if (connector == null) {
        throw new IllegalArgumentException(
                "No connector registered for id '" + connectorId
                + "'. Available: " + registry.keySet());
    }
    return connector.send(message);
}
```

- [ ] **Step 5: Update SlackConnector — boolean return + channelType null**

Use `ide_edit_member` on `SlackConnector.send()`:

```java
@Override
public boolean send(final ConnectorMessage message) {
    final String json = buildPayload(message.title(), message.body());
    final boolean ok = HttpHelper.postJson(message.destination(), json);
    if (!ok) {
        LOG.warning("Slack connector failed for destination: " + message.destination());
    }
    return ok;
}
```

Add `channelType()` using `ide_insert_member`:
```java
@Override
public String channelType() { return null; }
```

- [ ] **Step 6: Update TeamsConnector — boolean return + channelType null**

Use `ide_edit_member` on `TeamsConnector.send()`:

```java
@Override
public boolean send(final ConnectorMessage message) {
    final String json = buildPayload(message.title(), message.body());
    final boolean ok = HttpHelper.postJson(message.destination(), json);
    if (!ok) {
        LOG.warning("Teams connector failed for destination: " + message.destination());
    }
    return ok;
}
```

Add `channelType()` using `ide_insert_member`:
```java
@Override
public String channelType() { return null; }
```

- [ ] **Step 7: Update TwilioSmsConnector — boolean return + channelType "sms"**

Use `ide_edit_member` on `TwilioSmsConnector.send()`:

```java
@Override
public boolean send(final ConnectorMessage message) {
    if (accountSid.isBlank() || authToken.isBlank() || from.isBlank()) {
        LOG.warning("TwilioSmsConnector: casehub.connectors.twilio.* not configured — message not sent");
        return false;
    }

    final String url = TWILIO_API + accountSid + "/Messages.json";
    final String body = "To=" + encode(message.destination())
            + "&From=" + encode(from)
            + "&Body=" + encode(message.body() != null ? message.body() : "");

    final String credentials = Base64.getEncoder()
            .encodeToString((accountSid + ":" + authToken).getBytes(StandardCharsets.UTF_8));

    try {
        final HttpResponse<String> response = HttpHelper.CLIENT.send(
                HttpRequest.newBuilder()
                        .uri(URI.create(url))
                        .timeout(Duration.ofSeconds(10))
                        .header("Content-Type", "application/x-www-form-urlencoded")
                        .header("Authorization", "Basic " + credentials)
                        .POST(HttpRequest.BodyPublishers.ofString(body, StandardCharsets.UTF_8))
                        .build(),
                HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() < 200 || response.statusCode() >= 300) {
            LOG.warning("Twilio SMS failed to " + message.destination()
                    + " status=" + response.statusCode());
            return false;
        }
        return true;
    } catch (final Exception e) {
        LOG.warning("Twilio SMS error to " + message.destination() + ": " + e.getMessage());
        return false;
    }
}
```

Add `channelType()` using `ide_insert_member`:
```java
@Override
public String channelType() { return "sms"; }
```

- [ ] **Step 8: Update WhatsAppConnector — boolean return**

Use `ide_edit_member` on `WhatsAppConnector.send()`:

```java
@Override
public boolean send(final ConnectorMessage message) {
    if (apiToken.isBlank() || phoneNumberId.isBlank()) {
        LOG.warning("WhatsAppConnector: casehub.connectors.whatsapp.* not configured — message not sent");
        return false;
    }

    final String url = "https://graph.facebook.com/v18.0/" + phoneNumberId + "/messages";
    final String to = message.destination().replaceAll("[^0-9+]", "");
    final String templateName = message.attributes() != null
            ? message.attributes().get("templateName") : null;
    final String templateLanguage = (message.attributes() != null
            && message.attributes().get("templateLanguage") != null)
            ? message.attributes().get("templateLanguage") : "en_US";

    final String json = buildPayload(to, message.body(), templateName, templateLanguage);
    final boolean ok = HttpHelper.postJson(url, json, "Authorization", "Bearer " + apiToken);
    if (!ok) {
        LOG.warning("WhatsApp connector failed to: " + to);
    }
    return ok;
}
```

- [ ] **Step 9: Update EmailConnector — boolean return**

Use `ide_edit_member` on `EmailConnector.send()`:

```java
@Override
public boolean send(final ConnectorMessage message) {
    if (message.destination() == null || message.destination().isBlank()) {
        LOG.warning("EmailConnector: destination (email address) is blank — message not sent");
        return false;
    }

    final String subject = message.title() != null && !message.title().isBlank()
            ? message.title()
            : "Notification";
    final String body = message.body() != null ? message.body() : "";

    try {
        mailer.send(Mail.withText(message.destination(), subject, body));
        return true;
    } catch (final Exception e) {
        LOG.warning("EmailConnector: failed to send to " + message.destination()
                + ": " + e.getMessage());
        return false;
    }
}
```

- [ ] **Step 10: Update test RecordingConnectors**

`ConnectorServiceTest.RecordingConnector` — use `ide_edit_member`:
```java
@Override
public boolean send(final ConnectorMessage msg) {
    this.received = msg;
    return true;
}
```

`McpToolTestSupport.RecordingConnector` — use `ide_edit_member`:
```java
@Override
public boolean send(final ConnectorMessage message) {
    this.lastMessage = message;
    return true;
}
```

- [ ] **Step 11: Verify compilation and run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: all tests pass. The MCP tool callers (`EmailMcpTool`, `SlackMcpTool`, etc.) discard the return value — binary compatible.

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add core/ email/ mcp/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(core)!: Connector.send() returns boolean + add channelType() — #86

BREAKING: Connector.send() return type changes from void to boolean.
All built-in connectors updated. Callers that ignore the return value
compile without changes.

SlackConnector and TeamsConnector opt out of notification bridging
(channelType → null) until per-tenant deduplication lands.
TwilioSmsConnector maps to channel type 'sms'."
```

---

### Task 3: New notification-bridge module

**Repo:** `casehub-connectors` (`/Users/mdproctor/claude/casehub/connectors`)

**Files:**
- Create: `notification-bridge/pom.xml`
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/ConnectorNotificationDeliverer.java`
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/NotificationBridgeStartup.java`
- Create: `notification-bridge/src/test/java/io/casehub/connectors/notification/ConnectorNotificationDelivererTest.java`
- Create: `notification-bridge/src/test/java/io/casehub/connectors/notification/NotificationBridgeStartupTest.java`
- Modify: `pom.xml` (parent) — add `<module>notification-bridge</module>`

**Interfaces:**
- Consumes: `Connector` (Task 2), `ConnectorMessage` (Task 2), `NotificationDeliverer`, `DeliveryChannelRegistry`, `DeliveryChannelDescriptor`, `DeliveryResult`, `NotificationInput`, `DestinationResolver` (Task 1), `DeliveryChannels` (Task 1)
- Produces: registered `NotificationDeliverer` instances in `DeliveryChannelRegistry` at startup

- [ ] **Step 1: Create notification-bridge/pom.xml**

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

  <artifactId>casehub-connectors-notification-bridge</artifactId>
  <name>CaseHub Connectors — Notification Bridge</name>
  <description>Bridges the platform notification delivery system to the connector SPI.
Each Connector with a non-null channelType() automatically registers as a
NotificationDeliverer in the DeliveryChannelRegistry at startup.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
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

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>notification-bridge</module>` after `<module>chat-slack</module>` in the parent pom.xml.

- [ ] **Step 3: Write ConnectorNotificationDeliverer tests**

File: `notification-bridge/src/test/java/io/casehub/connectors/notification/ConnectorNotificationDelivererTest.java`

```java
package io.casehub.connectors.notification;

import java.util.Map;
import java.util.Optional;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DeliveryResult;
import io.casehub.platform.api.delivery.DestinationResolver;
import io.casehub.platform.api.notification.NotificationInput;
import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.notification.NotificationSource;

import static org.assertj.core.api.Assertions.assertThat;

class ConnectorNotificationDelivererTest {

    private static final NotificationSource SOURCE = new NotificationSource(
            "evt-1", "work-item", "wi-1", "actor-1");

    // ── Stub implementations ────────────────────────────────────────────────

    static class StubConnector implements Connector {
        ConnectorMessage lastMessage;
        boolean shouldSucceed = true;

        @Override public String id() { return "email"; }
        @Override public boolean send(ConnectorMessage message) {
            lastMessage = message;
            return shouldSucceed;
        }
    }

    static class StubResolver implements DestinationResolver {
        private final String channelId;
        private final Map<String, String> destinations;

        StubResolver(String channelId, Map<String, String> destinations) {
            this.channelId = channelId;
            this.destinations = destinations;
        }

        @Override public String channelId() { return channelId; }
        @Override public Optional<String> resolve(String userId, String tenancyId) {
            return Optional.ofNullable(destinations.get(userId));
        }
    }

    // ── Tests ───────────────────────────────────────────────────────────────

    @Test
    void deliver_withResolverAndDestination_callsConnectorAndReturnsSuccess() {
        var connector = new StubConnector();
        var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver);

        var input = new NotificationInput("user-1", "tenant-1", "Alert",
                "Something happened", "incident", NotificationSeverity.WARNING,
                "https://app/incidents/1", SOURCE);

        DeliveryResult result = deliverer.deliver(input);

        assertThat(result.success()).isTrue();
        assertThat(connector.lastMessage.destination()).isEqualTo("user1@example.com");
        assertThat(connector.lastMessage.title()).isEqualTo("Alert");
        assertThat(connector.lastMessage.body()).isEqualTo("Something happened");
    }

    @Test
    void deliver_passesMetadataAsAttributes() {
        var connector = new StubConnector();
        var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver);

        var input = new NotificationInput("user-1", "tenant-1", "Alert",
                "Body", "sla.breached", NotificationSeverity.URGENT,
                "https://app/sla/1", SOURCE);

        deliverer.deliver(input);

        assertThat(connector.lastMessage.attributes())
                .containsEntry("category", "sla.breached")
                .containsEntry("severity", "URGENT")
                .containsEntry("actionUrl", "https://app/sla/1");
    }

    @Test
    void deliver_nullBody_fallsBackToTitle() {
        var connector = new StubConnector();
        var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver);

        var input = new NotificationInput("user-1", "tenant-1", "Alert title",
                null, "test", NotificationSeverity.INFO, null, SOURCE);

        deliverer.deliver(input);

        assertThat(connector.lastMessage.body()).isEqualTo("Alert title");
    }

    @Test
    void deliver_nullActionUrl_omitsFromAttributes() {
        var connector = new StubConnector();
        var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver);

        var input = new NotificationInput("user-1", "tenant-1", "Alert",
                "Body", "test", NotificationSeverity.INFO, null, SOURCE);

        deliverer.deliver(input);

        assertThat(connector.lastMessage.attributes()).doesNotContainKey("actionUrl");
    }

    @Test
    void deliver_noResolver_returnsFailure() {
        var connector = new StubConnector();
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", null);

        var input = new NotificationInput("user-1", "tenant-1", "Alert",
                "Body", "test", NotificationSeverity.INFO, null, SOURCE);

        DeliveryResult result = deliverer.deliver(input);

        assertThat(result.success()).isFalse();
        assertThat(result.failureReason()).contains("no destination resolver");
        assertThat(connector.lastMessage).isNull();
    }

    @Test
    void deliver_noDestinationForUser_returnsFailure() {
        var connector = new StubConnector();
        var resolver = new StubResolver("email", Map.of());
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver);

        var input = new NotificationInput("unknown-user", "tenant-1", "Alert",
                "Body", "test", NotificationSeverity.INFO, null, SOURCE);

        DeliveryResult result = deliverer.deliver(input);

        assertThat(result.success()).isFalse();
        assertThat(result.failureReason()).contains("no destination");
    }

    @Test
    void deliver_connectorReturnsFalse_returnsFailure() {
        var connector = new StubConnector();
        connector.shouldSucceed = false;
        var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver);

        var input = new NotificationInput("user-1", "tenant-1", "Alert",
                "Body", "test", NotificationSeverity.INFO, null, SOURCE);

        DeliveryResult result = deliverer.deliver(input);

        assertThat(result.success()).isFalse();
        assertThat(result.failureReason()).contains("delivery failure");
    }

    @Test
    void channelId_returnsChannelType() {
        var connector = new StubConnector();
        var deliverer = new ConnectorNotificationDeliverer(connector, "sms", null);

        assertThat(deliverer.channelId()).isEqualTo("sms");
    }

    @Test
    void deliverDigest_returnsFailure() {
        var connector = new StubConnector();
        var deliverer = new ConnectorNotificationDeliverer(connector, "email", null);

        var input = new NotificationInput("user-1", "tenant-1", "Alert",
                null, "test", NotificationSeverity.INFO, null, SOURCE);
        var summary = new io.casehub.platform.api.notification.DigestSummary(
                "user-1", "tenant-1", "email",
                java.util.List.of(input),
                java.time.Instant.now().minusSeconds(3600),
                java.time.Instant.now(), null);

        DeliveryResult result = deliverer.deliverDigest(summary);

        assertThat(result.success()).isFalse();
        assertThat(result.failureReason()).contains("digest delivery not yet supported");
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=ConnectorNotificationDelivererTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: compilation failure — `ConnectorNotificationDeliverer` not found

- [ ] **Step 5: Implement ConnectorNotificationDeliverer**

File: `notification-bridge/src/main/java/io/casehub/connectors/notification/ConnectorNotificationDeliverer.java`

```java
package io.casehub.connectors.notification;

import java.util.HashMap;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DeliveryResult;
import io.casehub.platform.api.delivery.DestinationResolver;
import io.casehub.platform.api.delivery.NotificationDeliverer;
import io.casehub.platform.api.notification.DigestSummary;
import io.casehub.platform.api.notification.NotificationInput;

class ConnectorNotificationDeliverer implements NotificationDeliverer {

    private final Connector connector;
    private final String channelType;
    private final DestinationResolver resolver;

    ConnectorNotificationDeliverer(Connector connector, String channelType,
                                   DestinationResolver resolver) {
        this.connector = connector;
        this.channelType = channelType;
        this.resolver = resolver;
    }

    @Override
    public String channelId() {
        return channelType;
    }

    @Override
    public DeliveryResult deliver(NotificationInput notification) {
        if (resolver == null) {
            return new DeliveryResult(false,
                    "no destination resolver for " + channelType);
        }
        var destination = resolver.resolve(
                notification.userId(), notification.tenancyId());
        if (destination.isEmpty()) {
            return new DeliveryResult(false,
                    "no destination for user " + notification.userId());
        }

        String body = notification.body() != null
                ? notification.body() : notification.title();
        var attributes = new HashMap<String, String>();
        attributes.put("category", notification.category());
        attributes.put("severity", notification.severity().name());
        if (notification.actionUrl() != null) {
            attributes.put("actionUrl", notification.actionUrl());
        }

        boolean success = connector.send(new ConnectorMessage(
                destination.get(), notification.title(), body, attributes));
        return new DeliveryResult(success,
                success ? null : "connector reported delivery failure");
    }

    @Override
    public DeliveryResult deliverDigest(DigestSummary summary) {
        return new DeliveryResult(false,
                "digest delivery not yet supported for bridged channels");
    }
}
```

- [ ] **Step 6: Run ConnectorNotificationDeliverer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=ConnectorNotificationDelivererTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: all 9 tests PASS

- [ ] **Step 7: Write NotificationBridgeStartup tests**

File: `notification-bridge/src/test/java/io/casehub/connectors/notification/NotificationBridgeStartupTest.java`

```java
package io.casehub.connectors.notification;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DeliveryChannelDescriptor;
import io.casehub.platform.api.delivery.DeliveryChannelRegistry;
import io.casehub.platform.api.delivery.DestinationResolver;
import io.casehub.platform.api.delivery.NotificationDeliverer;
import io.casehub.platform.api.notification.NotificationSeverity;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class NotificationBridgeStartupTest {

    // ── Stubs ────────────────────────────────────────────────────────────────

    static class StubConnector implements Connector {
        private final String id;
        private final String channelType;

        StubConnector(String id, String channelType) {
            this.id = id;
            this.channelType = channelType;
        }

        StubConnector(String id) { this(id, id); }

        @Override public String id() { return id; }
        @Override public boolean send(ConnectorMessage message) { return true; }
        @Override public String channelType() { return channelType; }
    }

    static class StubResolver implements DestinationResolver {
        private final String channelId;
        StubResolver(String channelId) { this.channelId = channelId; }
        @Override public String channelId() { return channelId; }
        @Override public Optional<String> resolve(String userId, String tenancyId) {
            return Optional.of("resolved");
        }
    }

    static class RecordingRegistry implements DeliveryChannelRegistry {
        final java.util.LinkedHashMap<String, DeliveryChannelDescriptor> descriptors = new java.util.LinkedHashMap<>();
        final java.util.LinkedHashMap<String, NotificationDeliverer> deliverers = new java.util.LinkedHashMap<>();

        @Override
        public void register(DeliveryChannelDescriptor descriptor, NotificationDeliverer deliverer) {
            descriptors.put(descriptor.channelId(), descriptor);
            deliverers.put(descriptor.channelId(), deliverer);
        }

        @Override
        public Optional<DeliveryChannelDescriptor> resolve(String channelId) {
            return Optional.ofNullable(descriptors.get(channelId));
        }

        @Override
        public Optional<NotificationDeliverer> resolveDeliverer(String channelId) {
            return Optional.ofNullable(deliverers.get(channelId));
        }

        @Override
        public Set<DeliveryChannelDescriptor> discover() {
            return Set.copyOf(descriptors.values());
        }
    }

    // ── Tests ────────────────────────────────────────────────────────────────

    @Test
    void registers_connectorWithMatchingResolver() {
        var registry = new RecordingRegistry();
        var connector = new StubConnector("email");
        var resolver = new StubResolver("email");

        var startup = new NotificationBridgeStartup(
                List.of(connector), List.of(resolver), registry);
        startup.registerBridgedChannels();

        assertThat(registry.descriptors).containsKey("email");
        assertThat(registry.deliverers).containsKey("email");
    }

    @Test
    void registers_connectorWithoutResolver() {
        var registry = new RecordingRegistry();
        var connector = new StubConnector("email");

        var startup = new NotificationBridgeStartup(
                List.of(connector), List.of(), registry);
        startup.registerBridgedChannels();

        assertThat(registry.descriptors).containsKey("email");
        assertThat(registry.deliverers).containsKey("email");
    }

    @Test
    void skips_connectorWithNullChannelType() {
        var registry = new RecordingRegistry();
        var connector = new StubConnector("slack", null);

        var startup = new NotificationBridgeStartup(
                List.of(connector), List.of(), registry);
        startup.registerBridgedChannels();

        assertThat(registry.descriptors).isEmpty();
    }

    @Test
    void usesChannelType_notConnectorId() {
        var registry = new RecordingRegistry();
        var connector = new StubConnector("twilio-sms", "sms");
        var resolver = new StubResolver("sms");

        var startup = new NotificationBridgeStartup(
                List.of(connector), List.of(resolver), registry);
        startup.registerBridgedChannels();

        assertThat(registry.descriptors).containsKey("sms");
        assertThat(registry.descriptors).doesNotContainKey("twilio-sms");
    }

    @Test
    void duplicateChannelType_throwsAtStartup() {
        var registry = new RecordingRegistry();
        var c1 = new StubConnector("email-sendgrid", "email");
        var c2 = new StubConnector("email-ses", "email");

        var startup = new NotificationBridgeStartup(
                List.of(c1, c2), List.of(), registry);

        assertThatThrownBy(startup::registerBridgedChannels)
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("email");
    }

    @Test
    void descriptorDefaults_email() {
        var registry = new RecordingRegistry();
        var startup = new NotificationBridgeStartup(
                List.of(new StubConnector("email")), List.of(), registry);
        startup.registerBridgedChannels();

        var desc = registry.descriptors.get("email");
        assertThat(desc.displayName()).isEqualTo("Email");
        assertThat(desc.external()).isTrue();
        assertThat(desc.defaultEnabled()).isFalse();
        assertThat(desc.defaultMinSeverity()).isEqualTo(NotificationSeverity.WARNING);
        assertThat(desc.defaultDigestSchedule()).isNull();
        assertThat(desc.guaranteedMinSeverity()).isEqualTo(NotificationSeverity.WARNING);
    }

    @Test
    void descriptorDefaults_sms() {
        var registry = new RecordingRegistry();
        var startup = new NotificationBridgeStartup(
                List.of(new StubConnector("twilio-sms", "sms")), List.of(), registry);
        startup.registerBridgedChannels();

        var desc = registry.descriptors.get("sms");
        assertThat(desc.displayName()).isEqualTo("SMS");
        assertThat(desc.guaranteedMinSeverity()).isEqualTo(NotificationSeverity.WARNING);
    }

    @Test
    void descriptorDefaults_whatsapp() {
        var registry = new RecordingRegistry();
        var startup = new NotificationBridgeStartup(
                List.of(new StubConnector("whatsapp")), List.of(), registry);
        startup.registerBridgedChannels();

        var desc = registry.descriptors.get("whatsapp");
        assertThat(desc.displayName()).isEqualTo("WhatsApp");
        assertThat(desc.guaranteedMinSeverity()).isNull();
    }

    @Test
    void descriptorDefaults_unknownChannelType_usesChannelTypeAsDisplayName() {
        var registry = new RecordingRegistry();
        var startup = new NotificationBridgeStartup(
                List.of(new StubConnector("custom-channel")), List.of(), registry);
        startup.registerBridgedChannels();

        var desc = registry.descriptors.get("custom-channel");
        assertThat(desc.displayName()).isEqualTo("custom-channel");
        assertThat(desc.guaranteedMinSeverity()).isNull();
    }

    @Test
    void multipleConnectors_registersAll() {
        var registry = new RecordingRegistry();
        var startup = new NotificationBridgeStartup(
                List.of(
                        new StubConnector("email"),
                        new StubConnector("twilio-sms", "sms"),
                        new StubConnector("whatsapp"),
                        new StubConnector("slack", null)),
                List.of(new StubResolver("email"), new StubResolver("sms")),
                registry);
        startup.registerBridgedChannels();

        assertThat(registry.descriptors.keySet())
                .containsExactlyInAnyOrder("email", "sms", "whatsapp");
    }
}
```

- [ ] **Step 8: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=NotificationBridgeStartupTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: compilation failure — `NotificationBridgeStartup` not found

- [ ] **Step 9: Implement NotificationBridgeStartup**

File: `notification-bridge/src/main/java/io/casehub/connectors/notification/NotificationBridgeStartup.java`

```java
package io.casehub.connectors.notification;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkus.arc.All;
import io.quarkus.runtime.Startup;

import org.jboss.logging.Logger;

import io.casehub.connectors.Connector;
import io.casehub.platform.api.delivery.DeliveryChannelDescriptor;
import io.casehub.platform.api.delivery.DeliveryChannelRegistry;
import io.casehub.platform.api.delivery.DestinationResolver;
import io.casehub.platform.api.notification.NotificationSeverity;

@Startup
@ApplicationScoped
public class NotificationBridgeStartup {

    private static final Logger LOG = Logger.getLogger(NotificationBridgeStartup.class);

    private static final Map<String, String> DISPLAY_NAMES = Map.of(
            "email", "Email",
            "sms", "SMS",
            "slack", "Slack",
            "teams", "Teams",
            "whatsapp", "WhatsApp");

    private static final Map<String, NotificationSeverity> RETRY_POLICIES = Map.of(
            "email", NotificationSeverity.WARNING,
            "sms", NotificationSeverity.WARNING);

    private final List<Connector> connectors;
    private final List<DestinationResolver> resolvers;
    private final DeliveryChannelRegistry channelRegistry;

    @Inject
    public NotificationBridgeStartup(@All List<Connector> connectors,
                                     @All List<DestinationResolver> resolvers,
                                     DeliveryChannelRegistry channelRegistry) {
        this.connectors = connectors;
        this.resolvers = resolvers;
        this.channelRegistry = channelRegistry;
    }

    @PostConstruct
    void registerBridgedChannels() {
        Map<String, DestinationResolver> resolverIndex = new HashMap<>();
        for (DestinationResolver r : resolvers) {
            resolverIndex.put(r.channelId(), r);
        }

        Map<String, String> seenTypes = new HashMap<>();
        for (Connector connector : connectors) {
            String channelType = connector.channelType();
            if (channelType == null) {
                continue;
            }

            String previous = seenTypes.put(channelType, connector.id());
            if (previous != null) {
                throw new IllegalStateException(
                        "Duplicate channel type '" + channelType
                        + "' — connectors '" + previous + "' and '"
                        + connector.id() + "' both declare it");
            }

            DestinationResolver resolver = resolverIndex.get(channelType);
            var deliverer = new ConnectorNotificationDeliverer(
                    connector, channelType, resolver);

            var descriptor = new DeliveryChannelDescriptor(
                    channelType,
                    DISPLAY_NAMES.getOrDefault(channelType, channelType),
                    true,
                    false,
                    NotificationSeverity.WARNING,
                    null,
                    RETRY_POLICIES.get(channelType));

            channelRegistry.register(descriptor, deliverer);
            LOG.infof("Bridged connector '%s' as notification channel '%s'%s",
                    connector.id(), channelType,
                    resolver != null ? " (resolver: " + resolver.getClass().getSimpleName() + ")"
                            : " (no resolver)");
        }
    }
}
```

- [ ] **Step 10: Run all notification-bridge tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: all tests PASS

- [ ] **Step 11: Run full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS — all modules compile and all tests pass

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add notification-bridge/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat: notification-bridge module — auto-discovers connectors as delivery channels — #86

New module bridges platform NotificationDeliverer SPI to Connector SPI.
At startup, each Connector with non-null channelType() registers as a
notification delivery channel in DeliveryChannelRegistry.

DestinationResolver provides userId → destination resolution per channel.
Initial scope: email, SMS, WhatsApp (per-user destinations).
Slack and Teams deferred until per-tenant deduplication lands."
```
