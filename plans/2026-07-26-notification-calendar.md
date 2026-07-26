# Notification Bridge + CalendarPlatform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #89 — config-based DestinationResolver
**Issue group:** #89, #91, #88

**Goal:** Make the notification bridge functional (config resolver + digest delivery) and add CalendarPlatform SPI with Google Calendar provider and MCP tools.

**Architecture:** Three independent pieces: (1) config-based resolver in notification-bridge as fallback when no CDI resolver exists, (2) CDI-based DigestFormatter SPI with channel-specific formatters + EmailConnector HTML support, (3) CalendarPlatform SPI following the ChatPlatform module pattern (calendar-spi, calendar-ref, calendar-google) with MCP tools.

**Tech Stack:** Java 21, Quarkus 3.32.2, google-api-services-calendar v3, google-api-client 2.4.0, google-auth-library-oauth2-http

## Global Constraints

- **Java source level:** 21 (on Java 26 JVM)
- **Quarkus:** 3.32.2 — all modules use `casehub-parent` BOM
- **Artifact version:** `0.2-SNAPSHOT` throughout
- **SPI id method:** `id()` not `connectorId()` or `typeId()` (protocol PP-20260609-e3a2bd)
- **MCP tools:** every `@Tool` method calling blocking HTTP must have `@Blocking` (protocol PP-20260609-0625c9)
- **HTTP clients:** Google Calendar uses its own `NetHttpTransport`, not `HttpHelper.CLIENT` — different HTTP stack
- **Pagination:** `listEvents` must paginate with fail-soft partial return (protocol PP-20260610-83747b)
- **Build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- **IntelliJ MCP:** mandatory for all .java editing — use `ide_create_file`, `ide_insert_member`, `ide_edit_member`, `ide_replace_member`. Never use bash Edit/Write on existing Java files.

---

### Task 1: Config-based DestinationResolver (#89)

**Files:**
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/ConfigDestinationResolver.java`
- Modify: `notification-bridge/src/main/java/io/casehub/connectors/notification/NotificationBridgeStartup.java`
- Create: `notification-bridge/src/test/java/io/casehub/connectors/notification/ConfigDestinationResolverTest.java`
- Modify: `notification-bridge/src/test/java/io/casehub/connectors/notification/NotificationBridgeStartupTest.java`

**Interfaces:**
- Consumes: `DestinationResolver` SPI from `casehub-platform-api` (`channelId()`, `resolve(userId, tenancyId)`)
- Produces: `ConfigDestinationResolver(String channelType, Map<String, String> destinations)` — used only by `NotificationBridgeStartup`

- [ ] **Step 1: Write ConfigDestinationResolver unit tests**

```java
// ConfigDestinationResolverTest.java
package io.casehub.connectors.notification;

import java.util.Map;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ConfigDestinationResolverTest {

    @Test
    void resolve_knownUser_returnsDestination() {
        var resolver = new ConfigDestinationResolver("email",
                Map.of("user-1", "user1@example.com"));
        assertThat(resolver.resolve("user-1", "tenant-1"))
                .hasValue("user1@example.com");
    }

    @Test
    void resolve_unknownUser_returnsEmpty() {
        var resolver = new ConfigDestinationResolver("email",
                Map.of("user-1", "user1@example.com"));
        assertThat(resolver.resolve("unknown", "tenant-1")).isEmpty();
    }

    @Test
    void resolve_ignoresTenancyId() {
        var resolver = new ConfigDestinationResolver("sms",
                Map.of("user-1", "+447700900000"));
        assertThat(resolver.resolve("user-1", "tenant-A"))
                .hasValue("+447700900000");
        assertThat(resolver.resolve("user-1", "tenant-B"))
                .hasValue("+447700900000");
    }

    @Test
    void channelId_returnsConstructorValue() {
        var resolver = new ConfigDestinationResolver("whatsapp", Map.of());
        assertThat(resolver.channelId()).isEqualTo("whatsapp");
    }

    @Test
    void resolve_emptyMap_alwaysReturnsEmpty() {
        var resolver = new ConfigDestinationResolver("email", Map.of());
        assertThat(resolver.resolve("user-1", "tenant-1")).isEmpty();
    }

    @Test
    void hasEntries_withEntries_returnsTrue() {
        var resolver = new ConfigDestinationResolver("email",
                Map.of("user-1", "user1@example.com"));
        assertThat(resolver.hasEntries()).isTrue();
    }

    @Test
    void hasEntries_emptyMap_returnsFalse() {
        var resolver = new ConfigDestinationResolver("email", Map.of());
        assertThat(resolver.hasEntries()).isFalse();
    }
}
```

- [ ] **Step 2: Run tests — expect FAIL (class does not exist)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=ConfigDestinationResolverTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 3: Implement ConfigDestinationResolver**

```java
// ConfigDestinationResolver.java
package io.casehub.connectors.notification;

import java.util.Map;
import java.util.Optional;
import io.casehub.platform.api.delivery.DestinationResolver;

class ConfigDestinationResolver implements DestinationResolver {

    private final String channelType;
    private final Map<String, String> destinations;

    ConfigDestinationResolver(String channelType, Map<String, String> destinations) {
        this.channelType = channelType;
        this.destinations = Map.copyOf(destinations);
    }

    @Override
    public String channelId() {
        return channelType;
    }

    @Override
    public Optional<String> resolve(String userId, String tenancyId) {
        return Optional.ofNullable(destinations.get(userId));
    }

    boolean hasEntries() {
        return !destinations.isEmpty();
    }
}
```

- [ ] **Step 4: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=ConfigDestinationResolverTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 5: Write NotificationBridgeStartup integration tests**

Add these tests to `NotificationBridgeStartupTest.java`:

```java
// Add to existing test class — need a Config stub for testing
static class StubConfig implements org.eclipse.microprofile.config.Config {
    private final java.util.Map<String, String> values;
    StubConfig(java.util.Map<String, String> values) { this.values = values; }
    @Override public <T> T getValue(String propertyName, Class<T> propertyType) {
        return propertyType.cast(values.get(propertyName));
    }
    @Override public org.eclipse.microprofile.config.ConfigValue getConfigValue(String propertyName) { return null; }
    @Override public <T> java.util.Optional<T> getOptionalValue(String propertyName, Class<T> propertyType) {
        return java.util.Optional.ofNullable(values.get(propertyName)).map(propertyType::cast);
    }
    @Override public Iterable<String> getPropertyNames() { return values.keySet(); }
    @Override public Iterable<org.eclipse.microprofile.config.spi.ConfigSource> getConfigSources() { return java.util.List.of(); }
}

@Test
void configResolver_fallsBackWhenNoCdiResolver() {
    var registry = new RecordingRegistry();
    var connector = new StubConnector("email");
    var config = new StubConfig(java.util.Map.of(
            "casehub.notification.destinations.email.user-1", "user1@example.com"));

    var startup = new NotificationBridgeStartup(
            List.of(connector), List.of(), registry, config);
    startup.registerBridgedChannels();

    assertThat(registry.deliverers).containsKey("email");
    // Verify the config resolver actually works by delivering
    var deliverer = registry.deliverers.get("email");
    var input = new io.casehub.platform.api.notification.NotificationInput(
            "user-1", "tenant-1", "Test", null, "test",
            io.casehub.platform.api.notification.NotificationSeverity.INFO, null,
            new io.casehub.platform.api.notification.NotificationSource("e1", "wi", "w1", "a1"));
    var result = deliverer.deliver(input);
    assertThat(result.success()).isTrue();
}

@Test
void configResolver_cdiResolverTakesPrecedence() {
    var registry = new RecordingRegistry();
    var connector = new StubConnector("email");
    var cdiResolver = new StubResolver("email");
    var config = new StubConfig(java.util.Map.of(
            "casehub.notification.destinations.email.user-1", "should-not-be-used@example.com"));

    var startup = new NotificationBridgeStartup(
            List.of(connector), List.of(cdiResolver), registry, config);
    startup.registerBridgedChannels();

    // CDI resolver takes precedence — delivers via CDI resolver's "resolved" destination
    assertThat(registry.deliverers).containsKey("email");
}

@Test
void configResolver_noConfigNoResolver_resolverStaysNull() {
    var registry = new RecordingRegistry();
    var connector = new StubConnector("email");
    var config = new StubConfig(java.util.Map.of());

    var startup = new NotificationBridgeStartup(
            List.of(connector), List.of(), registry, config);
    startup.registerBridgedChannels();

    assertThat(registry.deliverers).containsKey("email");
    // Delivery should fail — no resolver
    var input = new io.casehub.platform.api.notification.NotificationInput(
            "user-1", "tenant-1", "Test", null, "test",
            io.casehub.platform.api.notification.NotificationSeverity.INFO, null,
            new io.casehub.platform.api.notification.NotificationSource("e1", "wi", "w1", "a1"));
    var result = registry.deliverers.get("email").deliver(input);
    assertThat(result.success()).isFalse();
    assertThat(result.failureReason()).contains("no destination resolver");
}

@Test
void configResolver_dynamicChannelTypeDiscovery() {
    var registry = new RecordingRegistry();
    var connector = new StubConnector("custom-channel");
    var config = new StubConfig(java.util.Map.of(
            "casehub.notification.destinations.custom-channel.user-1", "custom-dest"));

    var startup = new NotificationBridgeStartup(
            List.of(connector), List.of(), registry, config);
    startup.registerBridgedChannels();

    assertThat(registry.deliverers).containsKey("custom-channel");
}
```

- [ ] **Step 6: Run tests — expect FAIL (constructor signature mismatch)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=NotificationBridgeStartupTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 7: Update NotificationBridgeStartup to accept Config and create config fallback**

Modify `NotificationBridgeStartup`:
- Add `Config` as constructor parameter (injected via CDI)
- In `registerBridgedChannels()`, after checking `resolverIndex`, scan config for fallback
- Add private helper `scanConfigDestinations(Config, String channelType)` → `Map<String, String>`

Updated constructor:
```java
@Inject
public NotificationBridgeStartup(@All List<Connector> connectors,
                                 @All List<DestinationResolver> resolvers,
                                 DeliveryChannelRegistry channelRegistry,
                                 Config config) {
    this.connectors = connectors;
    this.resolvers = resolvers;
    this.channelRegistry = channelRegistry;
    this.config = config;
}
```

Updated `registerBridgedChannels()` — after `DestinationResolver resolver = resolverIndex.get(channelType);`:
```java
if (resolver == null) {
    Map<String, String> configDests = scanConfigDestinations(channelType);
    if (!configDests.isEmpty()) {
        resolver = new ConfigDestinationResolver(channelType, configDests);
    }
}
```

New private method:
```java
private Map<String, String> scanConfigDestinations(String channelType) {
    String prefix = "casehub.notification.destinations." + channelType + ".";
    Map<String, String> destinations = new HashMap<>();
    for (String name : config.getPropertyNames()) {
        if (name.startsWith(prefix)) {
            destinations.put(name.substring(prefix.length()),
                    config.getValue(name, String.class));
        }
    }
    return destinations;
}
```

Also update the existing test constructor calls to pass a `StubConfig(Map.of())` as the fourth parameter so existing tests still compile.

- [ ] **Step 8: Run all notification-bridge tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add notification-bridge/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(notification-bridge): config-based DestinationResolver fallback — #89

NotificationBridgeStartup scans config for userId→destination mappings
when no CDI-provided resolver exists for a channel type. CDI resolvers
take precedence. Config format: casehub.notification.destinations.<channel>.<userId>=<dest>"
```

---

### Task 2: DigestFormatter SPI + channel formatters (#91)

**Files:**
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/DigestFormatter.java`
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/EmailDigestFormatter.java`
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/SmsDigestFormatter.java`
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/WhatsAppDigestFormatter.java`
- Create: `notification-bridge/src/main/java/io/casehub/connectors/notification/DefaultDigestFormat.java`
- Modify: `notification-bridge/src/main/java/io/casehub/connectors/notification/ConnectorNotificationDeliverer.java`
- Modify: `notification-bridge/src/main/java/io/casehub/connectors/notification/NotificationBridgeStartup.java`
- Create: `notification-bridge/src/test/java/io/casehub/connectors/notification/DigestFormatterTest.java`
- Modify: `notification-bridge/src/test/java/io/casehub/connectors/notification/ConnectorNotificationDelivererTest.java`

**Interfaces:**
- Consumes: `DigestSummary`, `ConnectorMessage`, `Connector.send()`, `DestinationResolver.resolve()`
- Produces: `DigestFormatter` SPI interface — `channelId(): String`, `format(DigestSummary, String destination): ConnectorMessage`

- [ ] **Step 1: Write DigestFormatter interface and default format utility**

```java
// DigestFormatter.java
package io.casehub.connectors.notification;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DigestSummary;

public interface DigestFormatter {
    String channelId();
    ConnectorMessage format(DigestSummary summary, String destination);
}
```

```java
// DefaultDigestFormat.java — static utility, not CDI
package io.casehub.connectors.notification;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DigestSummary;

final class DefaultDigestFormat {
    private DefaultDigestFormat() {}

    static ConnectorMessage format(DigestSummary summary, String destination) {
        String body = "You have " + summary.notifications().size()
                + " notifications from " + summary.periodStart()
                + " to " + summary.periodEnd();
        return new ConnectorMessage(destination, "Notification digest", body);
    }
}
```

- [ ] **Step 2: Write formatter tests**

```java
// DigestFormatterTest.java
package io.casehub.connectors.notification;

import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.Test;
import io.casehub.platform.api.delivery.DigestGroupBy;
import io.casehub.platform.api.delivery.DigestSummary;
import io.casehub.platform.api.notification.NotificationInput;
import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.notification.NotificationSource;
import static org.assertj.core.api.Assertions.assertThat;

class DigestFormatterTest {

    private static final NotificationSource SOURCE =
            new NotificationSource("e1", "work-item", "wi-1", "actor-1");

    private static final Instant PERIOD_START = Instant.parse("2026-07-26T08:00:00Z");
    private static final Instant PERIOD_END = Instant.parse("2026-07-26T12:00:00Z");

    private DigestSummary summary(int count, DigestGroupBy groupBy) {
        var notifications = new java.util.ArrayList<NotificationInput>();
        for (int i = 0; i < count; i++) {
            notifications.add(new NotificationInput(
                    "user-1", "tenant-1", "Alert " + (i + 1),
                    "Body " + (i + 1), i % 2 == 0 ? "sla.breached" : "work-item.created",
                    i == 0 ? NotificationSeverity.URGENT : NotificationSeverity.INFO,
                    "https://app/item/" + (i + 1), SOURCE));
        }
        return new DigestSummary("user-1", "tenant-1", "email",
                notifications, PERIOD_START, PERIOD_END, groupBy);
    }

    @Test
    void emailFormatter_htmlBody() {
        var formatter = new EmailDigestFormatter();
        var msg = formatter.format(summary(3, null), "user1@example.com");
        assertThat(msg.destination()).isEqualTo("user1@example.com");
        assertThat(msg.title()).contains("3 notifications");
        assertThat(msg.body()).contains("<html>");
        assertThat(msg.body()).contains("Alert 1");
        assertThat(msg.body()).contains("Alert 2");
        assertThat(msg.body()).contains("Alert 3");
        assertThat(msg.attributes()).containsEntry("format", "html");
    }

    @Test
    void emailFormatter_categoryGrouping() {
        var formatter = new EmailDigestFormatter();
        var msg = formatter.format(summary(4, DigestGroupBy.CATEGORY), "user1@example.com");
        assertThat(msg.body()).contains("sla.breached");
        assertThat(msg.body()).contains("work-item.created");
    }

    @Test
    void emailFormatter_entityGrouping_treatedAsFlat() {
        var formatter = new EmailDigestFormatter();
        var msg = formatter.format(summary(2, DigestGroupBy.ENTITY), "user1@example.com");
        assertThat(msg.body()).contains("Alert 1");
        assertThat(msg.body()).contains("Alert 2");
    }

    @Test
    void smsFormatter_shortText() {
        var formatter = new SmsDigestFormatter();
        var msg = formatter.format(summary(5, null), "+447700900000");
        assertThat(msg.destination()).isEqualTo("+447700900000");
        assertThat(msg.body()).contains("5 notifications");
        assertThat(msg.body()).contains("Alert 1"); // most urgent (URGENT severity)
        assertThat(msg.body().length()).isLessThan(160);
    }

    @Test
    void whatsappFormatter_countAndCategories() {
        var formatter = new WhatsAppDigestFormatter();
        var msg = formatter.format(summary(4, null), "+447700900000");
        assertThat(msg.destination()).isEqualTo("+447700900000");
        assertThat(msg.body()).contains("4 notifications");
        assertThat(msg.body()).contains("sla.breached");
        assertThat(msg.body()).contains("Alert 1"); // most urgent
    }

    @Test
    void defaultFormat_plainText() {
        var msg = DefaultDigestFormat.format(summary(3, null), "dest");
        assertThat(msg.body()).contains("3 notifications");
        assertThat(msg.body()).contains(PERIOD_START.toString());
    }

    @Test
    void emailFormatter_channelId() {
        assertThat(new EmailDigestFormatter().channelId()).isEqualTo("email");
    }

    @Test
    void smsFormatter_channelId() {
        assertThat(new SmsDigestFormatter().channelId()).isEqualTo("sms");
    }

    @Test
    void whatsappFormatter_channelId() {
        assertThat(new WhatsAppDigestFormatter().channelId()).isEqualTo("whatsapp");
    }
}
```

- [ ] **Step 3: Run tests — expect FAIL**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=DigestFormatterTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 4: Implement channel formatters**

```java
// EmailDigestFormatter.java
package io.casehub.connectors.notification;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DigestGroupBy;
import io.casehub.platform.api.delivery.DigestSummary;
import io.casehub.platform.api.notification.NotificationInput;

@ApplicationScoped
public class EmailDigestFormatter implements DigestFormatter {

    @Override
    public String channelId() {
        return "email";
    }

    @Override
    public ConnectorMessage format(DigestSummary summary, String destination) {
        String title = summary.notifications().size() + " notifications ("
                + summary.periodStart() + " — " + summary.periodEnd() + ")";

        StringBuilder html = new StringBuilder("<html><body>");
        html.append("<h2>").append(title).append("</h2>");

        if (summary.groupBy() == DigestGroupBy.CATEGORY) {
            Map<String, List<NotificationInput>> grouped = summary.notifications().stream()
                    .collect(Collectors.groupingBy(NotificationInput::category));
            for (var entry : grouped.entrySet()) {
                html.append("<h3>").append(entry.getKey()).append("</h3><ul>");
                for (var n : entry.getValue()) {
                    appendNotification(html, n);
                }
                html.append("</ul>");
            }
        } else {
            html.append("<ul>");
            for (var n : summary.notifications()) {
                appendNotification(html, n);
            }
            html.append("</ul>");
        }

        html.append("</body></html>");
        var attributes = new HashMap<String, String>();
        attributes.put("format", "html");
        return new ConnectorMessage(destination, title, html.toString(), attributes);
    }

    private static void appendNotification(StringBuilder html, NotificationInput n) {
        html.append("<li><strong>").append(n.title()).append("</strong>");
        html.append(" [").append(n.severity()).append("] ");
        html.append(n.category());
        if (n.actionUrl() != null) {
            html.append(" — <a href=\"").append(n.actionUrl()).append("\">View</a>");
        }
        html.append("</li>");
    }
}
```

```java
// SmsDigestFormatter.java
package io.casehub.connectors.notification;

import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DigestSummary;
import io.casehub.platform.api.notification.NotificationInput;
import io.casehub.platform.api.notification.NotificationSeverity;
import java.util.Comparator;

@ApplicationScoped
public class SmsDigestFormatter implements DigestFormatter {

    private static final Comparator<NotificationInput> BY_SEVERITY =
            Comparator.comparing(NotificationInput::severity);

    @Override
    public String channelId() {
        return "sms";
    }

    @Override
    public ConnectorMessage format(DigestSummary summary, String destination) {
        var mostUrgent = summary.notifications().stream()
                .min(BY_SEVERITY)
                .orElseThrow();
        String body = summary.notifications().size() + " notifications. Most urgent: "
                + mostUrgent.title();
        return new ConnectorMessage(destination, body);
    }
}
```

```java
// WhatsAppDigestFormatter.java
package io.casehub.connectors.notification;

import java.util.Map;
import java.util.Comparator;
import java.util.stream.Collectors;
import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.platform.api.delivery.DigestSummary;
import io.casehub.platform.api.notification.NotificationInput;

@ApplicationScoped
public class WhatsAppDigestFormatter implements DigestFormatter {

    @Override
    public String channelId() {
        return "whatsapp";
    }

    @Override
    public ConnectorMessage format(DigestSummary summary, String destination) {
        var mostUrgent = summary.notifications().stream()
                .min(Comparator.comparing(NotificationInput::severity))
                .orElseThrow();

        Map<String, Long> categories = summary.notifications().stream()
                .collect(Collectors.groupingBy(NotificationInput::category, Collectors.counting()));

        StringBuilder body = new StringBuilder();
        body.append(summary.notifications().size()).append(" notifications\n");
        categories.forEach((cat, count) -> body.append("• ").append(cat).append(": ").append(count).append("\n"));
        body.append("\nMost urgent: ").append(mostUrgent.title());
        if (mostUrgent.actionUrl() != null) {
            body.append("\n").append(mostUrgent.actionUrl());
        }
        return new ConnectorMessage(destination, body.toString());
    }
}
```

- [ ] **Step 5: Run formatter tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=DigestFormatterTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 6: Write deliverDigest integration tests**

Add to `ConnectorNotificationDelivererTest.java`:

```java
// Replace existing deliverDigest_returnsFailure test with these:

@Test
void deliverDigest_withFormatter_formatsAndSends() {
    var connector = new StubConnector();
    var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
    var formatter = new EmailDigestFormatter();
    var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver, formatter);

    var input = new NotificationInput("user-1", "tenant-1", "Alert",
            null, "test", NotificationSeverity.INFO, null, SOURCE);
    var summary = new DigestSummary("user-1", "tenant-1", "email",
            List.of(input), Instant.now().minusSeconds(3600), Instant.now(), null);

    DeliveryResult result = deliverer.deliverDigest(summary);

    assertThat(result.success()).isTrue();
    assertThat(connector.lastMessage.destination()).isEqualTo("user1@example.com");
    assertThat(connector.lastMessage.body()).contains("<html>");
}

@Test
void deliverDigest_noFormatter_usesDefault() {
    var connector = new StubConnector();
    var resolver = new StubResolver("email", Map.of("user-1", "user1@example.com"));
    var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver, null);

    var input = new NotificationInput("user-1", "tenant-1", "Alert",
            null, "test", NotificationSeverity.INFO, null, SOURCE);
    var summary = new DigestSummary("user-1", "tenant-1", "email",
            List.of(input), Instant.now().minusSeconds(3600), Instant.now(), null);

    DeliveryResult result = deliverer.deliverDigest(summary);

    assertThat(result.success()).isTrue();
    assertThat(connector.lastMessage.body()).contains("1 notifications");
    assertThat(connector.lastMessage.body()).doesNotContain("<html>");
}

@Test
void deliverDigest_noResolver_returnsFailure() {
    var connector = new StubConnector();
    var deliverer = new ConnectorNotificationDeliverer(connector, "email", null, null);

    var input = new NotificationInput("user-1", "tenant-1", "Alert",
            null, "test", NotificationSeverity.INFO, null, SOURCE);
    var summary = new DigestSummary("user-1", "tenant-1", "email",
            List.of(input), Instant.now().minusSeconds(3600), Instant.now(), null);

    DeliveryResult result = deliverer.deliverDigest(summary);
    assertThat(result.success()).isFalse();
    assertThat(result.failureReason()).contains("no destination resolver");
}

@Test
void deliverDigest_noDestination_returnsFailure() {
    var connector = new StubConnector();
    var resolver = new StubResolver("email", Map.of());
    var deliverer = new ConnectorNotificationDeliverer(connector, "email", resolver, null);

    var input = new NotificationInput("user-1", "tenant-1", "Alert",
            null, "test", NotificationSeverity.INFO, null, SOURCE);
    var summary = new DigestSummary("user-1", "tenant-1", "email",
            List.of(input), Instant.now().minusSeconds(3600), Instant.now(), null);

    DeliveryResult result = deliverer.deliverDigest(summary);
    assertThat(result.success()).isFalse();
    assertThat(result.failureReason()).contains("no destination");
}
```

- [ ] **Step 7: Run tests — expect FAIL (constructor mismatch)**

- [ ] **Step 8: Update ConnectorNotificationDeliverer**

Add `digestFormatter` field and update constructor to accept it. Update `deliverDigest()`:

```java
private final DigestFormatter digestFormatter;

ConnectorNotificationDeliverer(Connector connector, String channelType,
                               DestinationResolver resolver, DigestFormatter digestFormatter) {
    this.connector = connector;
    this.channelType = channelType;
    this.resolver = resolver;
    this.digestFormatter = digestFormatter;
}

@Override
public DeliveryResult deliverDigest(DigestSummary summary) {
    if (resolver == null) {
        return new DeliveryResult(false,
                "no destination resolver for " + channelType);
    }
    var destination = resolver.resolve(summary.userId(), summary.tenancyId());
    if (destination.isEmpty()) {
        return new DeliveryResult(false,
                "no destination for user " + summary.userId());
    }
    ConnectorMessage msg = digestFormatter != null
            ? digestFormatter.format(summary, destination.get())
            : DefaultDigestFormat.format(summary, destination.get());
    boolean success = connector.send(msg);
    return new DeliveryResult(success,
            success ? null : "connector reported delivery failure");
}
```

Update `NotificationBridgeStartup` to inject `@All List<DigestFormatter>`, index them, and pass to deliverer:

```java
// New field
private final List<DigestFormatter> digestFormatters;

// Updated constructor — add parameter
@Inject
public NotificationBridgeStartup(@All List<Connector> connectors,
                                 @All List<DestinationResolver> resolvers,
                                 @All List<DigestFormatter> digestFormatters,
                                 DeliveryChannelRegistry channelRegistry,
                                 Config config) {
    // ... existing assignments ...
    this.digestFormatters = digestFormatters;
}

// In registerBridgedChannels() — build formatter index at start:
Map<String, DigestFormatter> formatterIndex = new HashMap<>();
for (DigestFormatter f : digestFormatters) {
    formatterIndex.put(f.channelId(), f);
}

// When creating deliverer — pass the formatter:
DigestFormatter formatter = formatterIndex.get(channelType);
var deliverer = new ConnectorNotificationDeliverer(
        connector, channelType, resolver, formatter);
```

Update existing test constructor calls (add `List.of()` for digestFormatters parameter, `null` for fourth parameter in ConnectorNotificationDeliverer where it was using the 3-arg constructor).

- [ ] **Step 9: Run all notification-bridge tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add notification-bridge/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(notification-bridge): DigestFormatter SPI + channel-aware digest delivery — #91

CDI-based DigestFormatter SPI with email (HTML), SMS, WhatsApp formatters.
ConnectorNotificationDeliverer.deliverDigest() now resolves destination and
formats per channel type. DefaultDigestFormat provides plain-text fallback."
```

---

### Task 3: EmailConnector HTML support (#91)

**Files:**
- Modify: `email/src/main/java/io/casehub/connectors/email/EmailConnector.java`
- Modify: `email/src/test/java/io/casehub/connectors/email/EmailConnectorTest.java`

**Interfaces:**
- Consumes: `ConnectorMessage.attributes()` — checks for `format=html`
- Produces: backward-compatible `EmailConnector.send()` — default remains plain text

- [ ] **Step 1: Write HTML rendering test**

Add to `EmailConnectorTest.java`:

```java
@Test
void send_htmlFormat_usesMailWithHtml() {
    mailbox.clear();
    var attributes = java.util.Map.of("format", "html");
    connector.send(new ConnectorMessage("alice@example.com", "Digest",
            "<html><body><h1>Report</h1></body></html>", attributes));

    final var messages = mailbox.getMailMessagesSentTo("alice@example.com");
    assertThat(messages).hasSize(1);
    assertThat(messages.get(0).getHtml()).contains("<h1>Report</h1>");
}

@Test
void send_noFormatAttribute_usesPlainText() {
    mailbox.clear();
    connector.send(new ConnectorMessage("bob@example.com", "Alert", "Plain text body"));

    final var messages = mailbox.getMailMessagesSentTo("bob@example.com");
    assertThat(messages).hasSize(1);
    assertThat(messages.get(0).getText()).isEqualTo("Plain text body");
    assertThat(messages.get(0).getHtml()).isNull();
}

@Test
void send_textFormatAttribute_usesPlainText() {
    mailbox.clear();
    var attributes = java.util.Map.of("format", "text");
    connector.send(new ConnectorMessage("bob@example.com", "Alert", "Plain body", attributes));

    final var messages = mailbox.getMailMessagesSentTo("bob@example.com");
    assertThat(messages).hasSize(1);
    assertThat(messages.get(0).getText()).isEqualTo("Plain body");
}
```

- [ ] **Step 2: Run tests — expect FAIL (HTML test fails — always sends plain text)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email -Dtest=EmailConnectorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 3: Update EmailConnector.send() to check format attribute**

Replace the `mailer.send(...)` call in `EmailConnector.send()` with:

```java
String format = message.attributes() != null
        ? message.attributes().getOrDefault("format", "text") : "text";
if ("html".equals(format)) {
    mailer.send(Mail.withHtml(message.destination(), subject, body));
} else {
    mailer.send(Mail.withText(message.destination(), subject, body));
}
```

- [ ] **Step 4: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email -Dtest=EmailConnectorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(email): HTML rendering via format=html attribute — #91

EmailConnector.send() checks attributes.get(\"format\"). When \"html\",
uses Mail.withHtml(). Default remains Mail.withText() — backward-compatible."
```

---

### Task 4: calendar-spi module — SPI + models + service (#88)

**Files:**
- Create: `calendar-spi/pom.xml`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/CalendarPlatform.java`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/EventTiming.java`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/model/CalendarInfo.java`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/model/CalendarEvent.java`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/model/EventDetails.java`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/CalendarPlatformService.java`
- Create: `calendar-spi/src/test/java/io/casehub/connectors/calendar/CalendarPlatformServiceTest.java`
- Modify: `pom.xml` (parent — add `calendar-spi` module)

**Interfaces:**
- Consumes: nothing — self-contained SPI
- Produces: `CalendarPlatform` interface, `CalendarPlatformService`, model records (`CalendarEvent`, `CalendarInfo`, `EventDetails`, `EventTiming`)

- [ ] **Step 1: Create pom.xml**

Create `calendar-spi/pom.xml` following `chat-spi/pom.xml` pattern but without `casehub-connectors-core` dependency (calendar-spi is self-contained):

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

  <artifactId>casehub-connectors-calendar-spi</artifactId>
  <name>CaseHub Connectors — Calendar SPI</name>
  <description>Calendar Platform SPI: CalendarPlatform interface, model types,
and CalendarPlatformService routing.</description>

  <dependencies>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
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

Add `<module>calendar-spi</module>` to parent `pom.xml` modules list.

- [ ] **Step 2: Create model records**

```java
// EventTiming.java
package io.casehub.connectors.calendar.spi;

import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneId;

public sealed interface EventTiming {
    record Timed(Instant start, Instant end, ZoneId timeZone) implements EventTiming {
        public Timed {
            java.util.Objects.requireNonNull(start, "start");
            java.util.Objects.requireNonNull(end, "end");
            java.util.Objects.requireNonNull(timeZone, "timeZone");
        }
    }
    record AllDay(LocalDate start, LocalDate end) implements EventTiming {
        public AllDay {
            java.util.Objects.requireNonNull(start, "start");
            java.util.Objects.requireNonNull(end, "end");
        }
    }
}
```

```java
// CalendarInfo.java
package io.casehub.connectors.calendar.model;

public record CalendarInfo(String id, String summary, String description, boolean primary) {}
```

```java
// CalendarEvent.java
package io.casehub.connectors.calendar.model;

import java.util.List;
import io.casehub.connectors.calendar.spi.EventTiming;

public record CalendarEvent(
        String id, String calendarId, String summary, String description,
        String location, EventTiming timing,
        List<String> attendees, String recurringEventId) {
    public CalendarEvent {
        java.util.Objects.requireNonNull(id, "id");
        java.util.Objects.requireNonNull(calendarId, "calendarId");
        java.util.Objects.requireNonNull(timing, "timing");
        attendees = attendees != null ? List.copyOf(attendees) : List.of();
    }
}
```

```java
// EventDetails.java
package io.casehub.connectors.calendar.model;

import java.util.List;
import io.casehub.connectors.calendar.spi.EventTiming;

public record EventDetails(
        String summary, String description, String location,
        EventTiming timing, List<String> attendees) {
    public EventDetails {
        java.util.Objects.requireNonNull(timing, "timing");
        attendees = attendees != null ? List.copyOf(attendees) : List.of();
    }
}
```

- [ ] **Step 3: Create CalendarPlatform interface**

```java
// CalendarPlatform.java
package io.casehub.connectors.calendar.spi;

import java.time.Instant;
import java.util.List;
import io.casehub.connectors.calendar.model.CalendarEvent;
import io.casehub.connectors.calendar.model.CalendarInfo;
import io.casehub.connectors.calendar.model.EventDetails;

public interface CalendarPlatform {
    String id();
    List<CalendarInfo> listCalendars();
    List<CalendarEvent> listEvents(String calendarId, Instant from, Instant to);
    CalendarEvent getEvent(String calendarId, String eventId);
    CalendarEvent createEvent(String calendarId, EventDetails details);
    CalendarEvent updateEvent(String calendarId, String eventId, EventDetails details);
    void deleteEvent(String calendarId, String eventId);
}
```

- [ ] **Step 4: Write CalendarPlatformService test**

```java
// CalendarPlatformServiceTest.java
package io.casehub.connectors.calendar;

import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.Test;
import io.casehub.connectors.calendar.model.CalendarEvent;
import io.casehub.connectors.calendar.model.CalendarInfo;
import io.casehub.connectors.calendar.model.EventDetails;
import io.casehub.connectors.calendar.spi.CalendarPlatform;
import io.casehub.connectors.calendar.spi.EventTiming;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class CalendarPlatformServiceTest {

    static class StubPlatform implements CalendarPlatform {
        private final String platformId;
        StubPlatform(String id) { this.platformId = id; }
        @Override public String id() { return platformId; }
        @Override public List<CalendarInfo> listCalendars() { return List.of(); }
        @Override public List<CalendarEvent> listEvents(String c, Instant f, Instant t) { return List.of(); }
        @Override public CalendarEvent getEvent(String c, String e) { return null; }
        @Override public CalendarEvent createEvent(String c, EventDetails d) { return null; }
        @Override public CalendarEvent updateEvent(String c, String e, EventDetails d) { return null; }
        @Override public void deleteEvent(String c, String e) {}
    }

    @Test
    void platform_knownId_returnsPlatform() {
        var service = new CalendarPlatformService(List.of(new StubPlatform("google")));
        assertThat(service.platform("google").id()).isEqualTo("google");
    }

    @Test
    void platform_unknownId_throws() {
        var service = new CalendarPlatformService(List.of(new StubPlatform("google")));
        assertThatThrownBy(() -> service.platform("outlook"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("outlook")
                .hasMessageContaining("google");
    }

    @Test
    void supports_knownId_returnsTrue() {
        var service = new CalendarPlatformService(List.of(new StubPlatform("ref")));
        assertThat(service.supports("ref")).isTrue();
    }

    @Test
    void supports_unknownId_returnsFalse() {
        var service = new CalendarPlatformService(List.of(new StubPlatform("ref")));
        assertThat(service.supports("outlook")).isFalse();
    }

    @Test
    void ids_returnsAllRegistered() {
        var service = new CalendarPlatformService(
                List.of(new StubPlatform("google"), new StubPlatform("ref")));
        assertThat(service.ids()).containsExactlyInAnyOrder("google", "ref");
    }

    @Test
    void duplicateId_throwsAtConstruction() {
        assertThatThrownBy(() -> new CalendarPlatformService(
                List.of(new StubPlatform("ref"), new StubPlatform("ref"))))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("ref");
    }
}
```

- [ ] **Step 5: Run test — expect FAIL (class does not exist)**

- [ ] **Step 6: Implement CalendarPlatformService**

```java
// CalendarPlatformService.java
package io.casehub.connectors.calendar;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;
import jakarta.enterprise.context.ApplicationScoped;
import io.quarkus.arc.All;
import io.casehub.connectors.calendar.spi.CalendarPlatform;

@ApplicationScoped
public class CalendarPlatformService {

    private final Map<String, CalendarPlatform> registry;

    public CalendarPlatformService(@All final List<CalendarPlatform> platforms) {
        this.registry = platforms.stream()
                .collect(Collectors.toMap(
                        CalendarPlatform::id,
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate calendar platform id: '" + a.id() + "'");
                        }));
    }

    public CalendarPlatform platform(final String id) {
        final CalendarPlatform platform = registry.get(id);
        if (platform == null) {
            throw new IllegalArgumentException(
                    "No calendar platform registered for id '" + id
                    + "'. Available: " + registry.keySet());
        }
        return platform;
    }

    public boolean supports(final String id) {
        return registry.containsKey(id);
    }

    public Set<String> ids() {
        return Set.copyOf(registry.keySet());
    }
}
```

- [ ] **Step 7: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add calendar-spi/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat: CalendarPlatform SPI — interface, model records, service routing — #88

CalendarPlatform with id(), listCalendars(), listEvents(), getEvent(),
createEvent(), updateEvent(), deleteEvent(). Sealed EventTiming (Timed/AllDay).
CalendarPlatformService routes by id(). No core dependency — self-contained."
```

---

### Task 5: calendar-ref module — reference implementation (#88)

**Files:**
- Create: `calendar-ref/pom.xml`
- Create: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/CalendarBackend.java`
- Create: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/InMemoryCalendarBackend.java`
- Create: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/RefCalendarPlatform.java`
- Create: `calendar-ref/src/test/java/io/casehub/connectors/calendar/ref/RefCalendarPlatformTest.java`
- Modify: `pom.xml` (parent — add `calendar-ref` module)

**Interfaces:**
- Consumes: `CalendarPlatform`, model records from `calendar-spi`
- Produces: `RefCalendarPlatform` — `@ApplicationScoped`, `id() = "ref"`. `InMemoryCalendarBackend` for test support.

- [ ] **Step 1: Create pom.xml**

Follow `chat-ref/pom.xml` pattern. Depend on `casehub-connectors-calendar-spi`. Include `test-jar` execution for reuse.

Add `<module>calendar-ref</module>` to parent `pom.xml`.

- [ ] **Step 2: Write RefCalendarPlatform test**

```java
// RefCalendarPlatformTest.java
package io.casehub.connectors.calendar.ref;

import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.List;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import io.casehub.connectors.calendar.model.EventDetails;
import io.casehub.connectors.calendar.spi.EventTiming;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RefCalendarPlatformTest {

    private RefCalendarPlatform platform;
    private InMemoryCalendarBackend backend;

    @BeforeEach
    void setUp() {
        backend = new InMemoryCalendarBackend();
        platform = new RefCalendarPlatform(backend);
    }

    @Test
    void id_isRef() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void listCalendars_returnsDefault() {
        assertThat(platform.listCalendars()).isNotEmpty();
        assertThat(platform.listCalendars().getFirst().primary()).isTrue();
    }

    @Test
    void createEvent_timedEvent_returnsWithId() {
        var details = new EventDetails("Meeting", "Standup", "Room 1",
                new EventTiming.Timed(
                        Instant.parse("2026-07-26T10:00:00Z"),
                        Instant.parse("2026-07-26T11:00:00Z"),
                        ZoneId.of("Europe/London")),
                List.of("alice@example.com"));

        var event = platform.createEvent("primary", details);

        assertThat(event.id()).isNotNull();
        assertThat(event.calendarId()).isEqualTo("primary");
        assertThat(event.summary()).isEqualTo("Meeting");
        assertThat(event.attendees()).containsExactly("alice@example.com");
        assertThat(event.timing()).isInstanceOf(EventTiming.Timed.class);
    }

    @Test
    void createEvent_allDayEvent() {
        var details = new EventDetails("Holiday", null, null,
                new EventTiming.AllDay(
                        LocalDate.of(2026, 7, 27),
                        LocalDate.of(2026, 7, 28)),
                List.of());

        var event = platform.createEvent("primary", details);

        assertThat(event.timing()).isInstanceOf(EventTiming.AllDay.class);
        var allDay = (EventTiming.AllDay) event.timing();
        assertThat(allDay.start()).isEqualTo(LocalDate.of(2026, 7, 27));
    }

    @Test
    void listEvents_returnsCreatedEvents() {
        var details = new EventDetails("Test", null, null,
                new EventTiming.Timed(
                        Instant.parse("2026-07-26T10:00:00Z"),
                        Instant.parse("2026-07-26T11:00:00Z"),
                        ZoneId.of("UTC")),
                List.of());
        platform.createEvent("primary", details);

        var events = platform.listEvents("primary",
                Instant.parse("2026-07-26T00:00:00Z"),
                Instant.parse("2026-07-27T00:00:00Z"));

        assertThat(events).hasSize(1);
        assertThat(events.getFirst().summary()).isEqualTo("Test");
    }

    @Test
    void listEvents_filtersOutsideRange() {
        var details = new EventDetails("Outside", null, null,
                new EventTiming.Timed(
                        Instant.parse("2026-07-28T10:00:00Z"),
                        Instant.parse("2026-07-28T11:00:00Z"),
                        ZoneId.of("UTC")),
                List.of());
        platform.createEvent("primary", details);

        var events = platform.listEvents("primary",
                Instant.parse("2026-07-26T00:00:00Z"),
                Instant.parse("2026-07-27T00:00:00Z"));

        assertThat(events).isEmpty();
    }

    @Test
    void getEvent_existing_returnsEvent() {
        var details = new EventDetails("Find me", null, null,
                new EventTiming.Timed(
                        Instant.parse("2026-07-26T10:00:00Z"),
                        Instant.parse("2026-07-26T11:00:00Z"),
                        ZoneId.of("UTC")),
                List.of());
        var created = platform.createEvent("primary", details);

        var found = platform.getEvent("primary", created.id());
        assertThat(found.summary()).isEqualTo("Find me");
    }

    @Test
    void getEvent_nonExistent_throws() {
        assertThatThrownBy(() -> platform.getEvent("primary", "no-such-id"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void updateEvent_replacesSummary() {
        var details = new EventDetails("Original", null, null,
                new EventTiming.Timed(
                        Instant.parse("2026-07-26T10:00:00Z"),
                        Instant.parse("2026-07-26T11:00:00Z"),
                        ZoneId.of("UTC")),
                List.of());
        var created = platform.createEvent("primary", details);

        var updated = platform.updateEvent("primary", created.id(),
                new EventDetails("Updated", "new desc", "Room 2",
                        details.timing(), List.of("bob@example.com")));

        assertThat(updated.summary()).isEqualTo("Updated");
        assertThat(updated.description()).isEqualTo("new desc");
        assertThat(updated.attendees()).containsExactly("bob@example.com");
    }

    @Test
    void deleteEvent_removesEvent() {
        var details = new EventDetails("Delete me", null, null,
                new EventTiming.Timed(
                        Instant.parse("2026-07-26T10:00:00Z"),
                        Instant.parse("2026-07-26T11:00:00Z"),
                        ZoneId.of("UTC")),
                List.of());
        var created = platform.createEvent("primary", details);

        platform.deleteEvent("primary", created.id());

        assertThatThrownBy(() -> platform.getEvent("primary", created.id()))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 3: Run test — expect FAIL**

- [ ] **Step 4: Implement CalendarBackend, InMemoryCalendarBackend, RefCalendarPlatform**

`CalendarBackend` is an interface following `ChatBackend` pattern. `InMemoryCalendarBackend` stores events in a `ConcurrentHashMap`. `RefCalendarPlatform` delegates to the backend.

The key implementation detail for `listEvents` time filtering: for `Timed` events, check overlap with `[from, to)`. For `AllDay` events, convert dates to instants at UTC midnight for comparison.

`RefCalendarPlatform` is `@ApplicationScoped` with `id() = "ref"`.

Event IDs are generated via `UUID.randomUUID().toString()`.

- [ ] **Step 5: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add calendar-ref/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat: RefCalendarPlatform — in-memory reference implementation — #88

InMemoryCalendarBackend with CRUD + time-range filtering. RefCalendarPlatform
delegates to backend, id()=\"ref\". Full test coverage for all SPI methods."
```

---

### Task 6: calendar-google module — Google Calendar provider (#88)

**Files:**
- Create: `calendar-google/pom.xml`
- Create: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java`
- Create: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleEventMapper.java`
- Create: `calendar-google/src/test/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatformTest.java`
- Create: `calendar-google/src/test/java/io/casehub/connectors/calendar/google/GoogleEventMapperTest.java`
- Modify: `pom.xml` (parent — add `calendar-google` module)

**Interfaces:**
- Consumes: `CalendarPlatform` from `calendar-spi`, Google Calendar API types
- Produces: `GoogleCalendarPlatform` — `@ApplicationScoped`, `id() = "google"`, fail-soft on missing credentials

- [ ] **Step 1: Create pom.xml**

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

  <artifactId>casehub-connectors-calendar-google</artifactId>
  <name>CaseHub Connectors — Google Calendar</name>
  <description>Google Calendar provider for the CalendarPlatform SPI.
Uses google-api-services-calendar with OAuth2 refresh token authentication.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-calendar-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>com.google.api-client</groupId>
      <artifactId>google-api-client</artifactId>
      <version>2.4.0</version>
    </dependency>
    <dependency>
      <groupId>com.google.apis</groupId>
      <artifactId>google-api-services-calendar</artifactId>
      <version>v3-rev20250404-2.0.0</version>
    </dependency>
    <dependency>
      <groupId>com.google.auth</groupId>
      <artifactId>google-auth-library-oauth2-http</artifactId>
      <version>1.30.1</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
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

Add `<module>calendar-google</module>` to parent pom.xml.

- [ ] **Step 2: Write GoogleEventMapper tests**

Test mapping between Google `Event` ↔ `CalendarEvent` and `EventDetails` → Google `Event`. Cover timed events, all-day events, attendees, recurring event IDs.

- [ ] **Step 3: Implement GoogleEventMapper**

Static utility class. Maps `com.google.api.services.calendar.model.Event` → `CalendarEvent` and `EventDetails` → `com.google.api.services.calendar.model.Event`.

Key mapping decisions:
- `EventDateTime.getDateTime()` non-null → `Timed` event; `EventDateTime.getDate()` non-null → `AllDay` event
- Timezone: `EventDateTime.getTimeZone()` or fall back to `event.getStart().getTimeZone()`
- Attendees: `event.getAttendees()` → `List<String>` of emails
- `recurringEventId`: direct mapping from Google's `event.getRecurringEventId()`

- [ ] **Step 4: Write GoogleCalendarPlatform test with WireMock**

Test `listEvents` (including pagination), `createEvent`, `getEvent`, `updateEvent`, `deleteEvent`. Verify fail-soft: blank credentials → no API client built, `CalendarPlatformService` simply excludes `"google"`.

For WireMock: mock the Google Calendar API endpoints. `GoogleCalendarPlatform` needs a constructor that accepts a pre-built `Calendar` client for testing (no need for real OAuth).

- [ ] **Step 5: Implement GoogleCalendarPlatform**

`@ApplicationScoped` bean. At `@PostConstruct`, read config properties. If any credential is blank, log WARNING and skip client construction. All methods throw `IllegalStateException` if client not initialised (fail-soft — the service won't include this platform if it's not operational).

`listEvents` must paginate per protocol — loop on `nextPageToken`, fail-soft on mid-loop error (return partial results with WARNING).

- [ ] **Step 6: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add calendar-google/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat: GoogleCalendarPlatform — Google Calendar API provider — #88

OAuth2 refresh token auth via config. Paginated listEvents with fail-soft
partial return. GoogleEventMapper translates between platform model and
Google API types. Fail-soft on missing credentials."
```

---

### Task 7: CalendarMcpTool in mcp module (#88)

**Files:**
- Modify: `mcp/pom.xml` (add `calendar-spi` dependency)
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/CalendarMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/CalendarMcpToolTest.java`

**Interfaces:**
- Consumes: `CalendarPlatformService.platform(id)`, `CalendarPlatform` methods
- Produces: MCP `@Tool` methods: `calendar_list_calendars`, `calendar_list_events`, `calendar_get_event`, `calendar_create_event`, `calendar_update_event`, `calendar_delete_event`

- [ ] **Step 1: Add dependency to mcp/pom.xml**

Add `casehub-connectors-calendar-spi` compile dependency and `casehub-connectors-calendar-ref` test dependency.

- [ ] **Step 2: Write CalendarMcpTool tests**

Test each tool method using `RefCalendarPlatform` + `CalendarPlatformService` (same pattern as `ChatPlatformMcpToolTest`):
- `listCalendars` — returns ref calendars
- `listCalendarEvents` — returns created events
- `getCalendarEvent` — returns specific event
- `createCalendarEvent` — timed event with all params
- `createCalendarEvent` — all-day event
- `createCalendarEvent` — both timed and allDay params → `"Failed: ..."`
- `createCalendarEvent` — start without timeZone → `"Failed: ..."`
- `updateCalendarEvent` — PATCH merge behavior (only provided fields change)
- `deleteCalendarEvent` — removes event
- Unknown platform → `"Failed: ..."`

- [ ] **Step 3: Run tests — expect FAIL**

- [ ] **Step 4: Implement CalendarMcpTool**

```java
@ApplicationScoped
public class CalendarMcpTool {

    private final CalendarPlatformService platformService;

    @Inject
    public CalendarMcpTool(CalendarPlatformService platformService) {
        this.platformService = platformService;
    }

    @Tool(description = "List available calendars")
    @Blocking
    public String listCalendars(String platform) { ... }

    @Tool(description = "List calendar events in a time range")
    @Blocking
    public String listCalendarEvents(String platform, String calendarId,
                                      String from, String to) { ... }

    @Tool(description = "Get a specific calendar event by ID")
    @Blocking
    public String getCalendarEvent(String platform, String calendarId,
                                    String eventId) { ... }

    @Tool(description = "Create a calendar event")
    @Blocking
    public String createCalendarEvent(String platform, String calendarId,
            String summary, String description, String location,
            String start, String end, String timeZone,
            String startDate, String endDate, String attendees) { ... }

    @Tool(description = "Update a calendar event")
    @Blocking
    public String updateCalendarEvent(String platform, String calendarId,
            String eventId, String summary, String description, String location,
            String start, String end, String timeZone,
            String startDate, String endDate, String attendees) { ... }

    @Tool(description = "Delete a calendar event")
    @Blocking
    public String deleteCalendarEvent(String platform, String calendarId,
                                       String eventId) { ... }
}
```

Key implementation details:
- `calendarId` defaults to `"primary"` when null/blank
- Timing parameter validation per spec table (both timed+allDay → fail, start without timeZone → fail, etc.)
- `updateCalendarEvent` does PATCH: `getEvent()` → merge non-null params → `updateEvent()` with full `EventDetails`
- All methods catch exceptions and return `"Failed: ..."` (fail-soft MCP pattern)
- `attendees` param is comma-separated string, split into `List<String>`

- [ ] **Step 5: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 6: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add mcp/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): CalendarMcpTool — calendar operations for LLM agents — #88

Six @Tool methods: list_calendars, list_events, get_event, create_event,
update_event, delete_event. Timing param validation (timed vs all-day).
Update uses PATCH merge. All @Blocking per protocol."
```
