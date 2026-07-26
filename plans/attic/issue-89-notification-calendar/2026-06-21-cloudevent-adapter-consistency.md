# CloudEvent Adapter Consistency Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement ConnectorCloudEventAdapter (connectors#20) and fix the existing IoT/Qhorus adapters to match the canonical CloudEvent adapter pattern across all three repos.

**Architecture:** New `casehub-connectors-cloud-events` submodule observes `@ObservesAsync InboundMessage` and fires `Event<CloudEvent>.fireAsync()`. `InboundMessage` gains `connectorType` and `tenancyId` fields. IoT adapter gets 4 fixes; Qhorus adapter gets 1 fix + ~14 test site migrations.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, CloudEvents Java SDK (`cloudevents-core` via `casehub-platform-api`), CDI async events

**Spec:** `specs/2026-06-21-cloudevent-adapter-consistency-design.md`

**Repos:**
- connectors: `/Users/mdproctor/claude/casehub/connectors` (branch: `issue-20-cloudEvent-adapter`)
- iot: `/Users/mdproctor/claude/casehub/iot` (main)
- qhorus: `/Users/mdproctor/claude/casehub/qhorus` (main)

**Build command (all repos):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

### Task 1: File IoT and Qhorus issues

**Files:** None — GitHub only

- [ ] **Step 1: File IoT issue**

```bash
gh issue create --repo casehubio/iot \
  --title "fix: IoTCloudEventAdapter — inject ObjectMapper, null-safe tenancyId, handle serialisation + fireAsync" \
  --body "Four fixes to align with canonical CloudEvent adapter pattern (spec: casehubio/connectors#20).

1. **Inject ObjectMapper** — static instance isolates serialisation from app's shared mapper
2. **Null-safe tenancyId** — defence in depth (DeviceEntity enforces non-null, but adapter should not rely on domain invariant)
3. **Serialisation error** — \`throw new UncheckedIOException\` from \`@ObservesAsync\` is silently swallowed by CDI; catch at WARN, fire with empty data
4. **fireAsync CompletionStage** — unhandled; add \`.exceptionally()\` handler
5. **Add Logger** — class has no logging infrastructure; add \`org.jboss.logging.Logger\`

Refs casehubio/connectors#20" \
  --label "bug"
```

Record the returned issue number as `IOT_ISSUE`.

- [ ] **Step 2: File Qhorus issue**

```bash
gh issue create --repo casehubio/qhorus \
  --title "fix: QhorusCloudEventAdapter fireAsync + InboundMessage constructor migration" \
  --body "1. **fireAsync CompletionStage** — unhandled; add \`.exceptionally()\` handler
2. **~14 test construction sites** — \`InboundMessage\` gains \`connectorType\` and \`tenancyId\` fields (casehubio/connectors#20); all test sites must supply them
3. **Protocol violation** — \`ConfiguredAutoChannelPolicyTest:113\` uses raw string \`\"slack-inbound\"\` instead of \`InboundConnectorIds.SLACK_INBOUND\`

Refs casehubio/connectors#20" \
  --label "bug"
```

Record the returned issue number as `QHORUS_ISSUE`.

---

### Task 2: New constants and WebhookRequest accessor

**Files:**
- Create: `core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java`
- Modify: `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java`
- Modify: `core/src/main/java/io/casehub/connectors/WebhookRequest.java`
- Test: `core/src/test/java/io/casehub/connectors/InboundConnectorTypesTest.java`

- [ ] **Step 1: Write tests for InboundConnectorTypes**

```java
package io.casehub.connectors;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class InboundConnectorTypesTest {

    @Test
    void constants_areStableSemanticLabels() {
        assertThat(InboundConnectorTypes.SLACK).isEqualTo("slack");
        assertThat(InboundConnectorTypes.EMAIL).isEqualTo("email");
        assertThat(InboundConnectorTypes.SMS).isEqualTo("sms");
        assertThat(InboundConnectorTypes.WHATSAPP).isEqualTo("whatsapp");
        assertThat(InboundConnectorTypes.TEAMS).isEqualTo("teams");
    }

    @Test
    void types_doNotContainProviderOrDirectionSuffix() {
        assertThat(InboundConnectorTypes.SMS).doesNotContain("twilio");
        assertThat(InboundConnectorTypes.SMS).doesNotContain("inbound");
        assertThat(InboundConnectorTypes.SLACK).doesNotContain("inbound");
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=InboundConnectorTypesTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Compilation failure — `InboundConnectorTypes` does not exist

- [ ] **Step 3: Create InboundConnectorTypes**

Create `core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java`:

```java
package io.casehub.connectors;

public final class InboundConnectorTypes {

    public static final String SLACK    = "slack";
    public static final String EMAIL    = "email";
    public static final String SMS      = "sms";
    public static final String WHATSAPP = "whatsapp";
    public static final String TEAMS    = "teams";

    private InboundConnectorTypes() {}
}
```

- [ ] **Step 4: Add TEAMS_INBOUND to InboundConnectorIds**

In `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java`, add after the WHATSAPP line:

```java
public static final String TEAMS_INBOUND = "teams-inbound";
```

- [ ] **Step 5: Add tenancyId() accessor to WebhookRequest**

In `core/src/main/java/io/casehub/connectors/WebhookRequest.java`, add after the existing `header()` method:

```java
public String tenancyId() {
    return header("x-tenancy-id");
}
```

- [ ] **Step 6: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests pass

- [ ] **Step 7: Commit**

```
git add core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java \
        core/src/main/java/io/casehub/connectors/InboundConnectorIds.java \
        core/src/main/java/io/casehub/connectors/WebhookRequest.java \
        core/src/test/java/io/casehub/connectors/InboundConnectorTypesTest.java
git commit -m "feat(core): add InboundConnectorTypes, TEAMS_INBOUND, WebhookRequest.tenancyId() — Refs #20"
```

---

### Task 3: EmailInboundAccount tenancyId field

**Files:**
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccount.java`
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProvider.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProviderTest.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

- [ ] **Step 1: Add tenancyId to EmailInboundAccount**

In `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccount.java`, add `tenancyId` as the last field:

```java
public record EmailInboundAccount(
        String id,
        String host,
        int port,
        boolean tls,
        String username,
        String password,
        String folder,
        int reconnectDelaySeconds,
        String tenancyId) {
}
```

- [ ] **Step 2: Update DefaultEmailInboundAccountProvider**

Add the config property field:

```java
@ConfigProperty(name = "casehub.connectors.email-inbound.tenancy-id", defaultValue = "")
String tenancyId;
```

Update the package-private test constructor to accept `tenancyId`:

```java
DefaultEmailInboundAccountProvider(final String host, final int port, final boolean tls,
                                   final String username, final String password,
                                   final String folder, final int reconnectDelaySeconds,
                                   final String tenancyId) {
    this.host = host;
    this.port = port;
    this.tls = tls;
    this.username = username;
    this.password = password;
    this.folder = folder;
    this.reconnectDelaySeconds = reconnectDelaySeconds;
    this.tenancyId = tenancyId;
}
```

Update `accounts()` to normalise empty to null:

```java
@Override
public List<EmailInboundAccount> accounts() {
    if (host == null || host.isBlank()) {
        return List.of();
    }
    return List.of(new EmailInboundAccount(
            EmailInboundConnector.ID, host, port, tls, username, password,
            folder, reconnectDelaySeconds,
            tenancyId == null || tenancyId.isBlank() ? null : tenancyId));
}
```

- [ ] **Step 3: Fix DefaultEmailInboundAccountProviderTest**

Add `null` as the last arg to all 4 test constructor calls (lines 14, 21, 28–29, 48–49):

```java
// Each constructor call gains a trailing null:
new DefaultEmailInboundAccountProvider("", 993, true, "user", "pass", "INBOX", 60, null)
new DefaultEmailInboundAccountProvider(null, 993, true, "user", "pass", "INBOX", 60, null)
new DefaultEmailInboundAccountProvider("imap.example.com", 993, true, "user@example.com", "secret", "INBOX", 60, null)
new DefaultEmailInboundAccountProvider("imap.example.com", 143, false, "user", "pass", "Support", 30, null)
```

- [ ] **Step 4: Fix EmailInboundConnectorTest**

Update `testAccount()` (~line 62):

```java
private EmailInboundAccount testAccount() {
    return new EmailInboundAccount(
            "email-inbound", "localhost", GREEN_MAIL.getImap().getPort(),
            false, "inbox@example.com", "password", "INBOX", 60, null);
}
```

Update connection failure test (~line 249):

```java
new EmailInboundAccount("bad", "localhost", 19999, false,
        "u", "p", "INBOX", 1, null)
```

- [ ] **Step 5: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests pass

- [ ] **Step 6: Commit**

```
git add email-inbound/
git commit -m "feat(email-inbound): add tenancyId to EmailInboundAccount + MP Config property — Refs #20"
```

---

### Task 4: InboundMessage record changes + all connectors test migrations

This task changes the `InboundMessage` record and fixes every construction site in the connectors repo. Must be atomic — nothing compiles between the record change and the call site fixes.

**Files:**
- Modify: `core/src/main/java/io/casehub/connectors/InboundMessage.java`
- Modify: `core/src/test/java/io/casehub/connectors/InboundMessageTest.java`
- Modify: `core/src/test/java/io/casehub/connectors/InboundConnectorServiceTest.java`
- Modify: `webhook/src/test/java/io/casehub/connectors/webhook/InboundMessageCapture.java` (no changes needed — observes, doesn't construct)
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/InboundMessageCapture.java` (no changes needed)

- [ ] **Step 1: Rewrite InboundMessage record**

Replace the entire `InboundMessage.java` with:

```java
package io.casehub.connectors;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public record InboundMessage(
        String connectorId,
        String connectorType,
        String externalSenderId,
        String externalChannelRef,
        String content,
        List<Attachment> attachments,
        Instant receivedAt,
        Map<String, String> metadata,
        String tenancyId) {

    public InboundMessage {
        Objects.requireNonNull(connectorType, "connectorType");
        attachments = List.copyOf(attachments);
    }
}
```

Remove all convenience constructors.

- [ ] **Step 2: Rewrite InboundMessageTest**

Replace with tests for the new canonical constructor and validation:

```java
package io.casehub.connectors;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

class InboundMessageTest {

    @Test
    void canonicalConstructor_allFieldsSet() {
        final List<Attachment> atts = List.of(
                new Attachment("f.pdf", "application/pdf", new byte[]{1}));
        final Instant now = Instant.now();
        final InboundMessage msg = new InboundMessage(
                "email-inbound", "email", "sender@example.com", "inbox@example.com",
                "body", atts, now, Map.of("k", "v"), "tenant-1");

        assertThat(msg.connectorId()).isEqualTo("email-inbound");
        assertThat(msg.connectorType()).isEqualTo("email");
        assertThat(msg.externalSenderId()).isEqualTo("sender@example.com");
        assertThat(msg.externalChannelRef()).isEqualTo("inbox@example.com");
        assertThat(msg.content()).isEqualTo("body");
        assertThat(msg.attachments()).hasSize(1);
        assertThat(msg.receivedAt()).isEqualTo(now);
        assertThat(msg.metadata()).containsEntry("k", "v");
        assertThat(msg.tenancyId()).isEqualTo("tenant-1");
    }

    @Test
    void nullTenancyId_isAllowed() {
        final InboundMessage msg = new InboundMessage(
                "slack-inbound", "slack", "U123", "C456",
                "hello", List.of(), Instant.now(), Map.of(), null);
        assertThat(msg.tenancyId()).isNull();
    }

    @Test
    void nullConnectorType_throwsNPE() {
        assertThatNullPointerException().isThrownBy(() ->
                new InboundMessage("slack-inbound", null, "U123", "C456",
                        "hello", List.of(), Instant.now(), Map.of(), null))
                .withMessageContaining("connectorType");
    }

    @Test
    void attachments_defensivelyCopied() {
        final List<Attachment> mutable = new ArrayList<>();
        mutable.add(new Attachment("f.pdf", "application/pdf", new byte[]{1}));
        final InboundMessage msg = new InboundMessage(
                "email-inbound", "email", "s", "c",
                "body", mutable, Instant.now(), Map.of(), null);

        mutable.clear();
        assertThat(msg.attachments()).hasSize(1);
    }
}
```

- [ ] **Step 3: Fix InboundConnectorServiceTest**

Update `sampleMessage()` helper (~line 46):

```java
private static InboundMessage sampleMessage(final String connectorId) {
    return new InboundMessage(connectorId, "slack", "sender-1", "channel-1",
            "hello", List.of(), Instant.now(), Map.of(), null);
}
```

Add `List.of()` import if missing, and ensure every `new InboundMessage(...)` call in this file uses the 9-arg constructor.

- [ ] **Step 4: Fix webhook connector production sites**

**SlackInboundConnector.java** (~line 168) — the `parseMessages` method. The `new InboundMessage(...)` call changes from 6-arg to 9-arg:

```java
messages.add(new InboundMessage(ID, InboundConnectorTypes.SLACK,
        user, channel, text, List.of(), Instant.now(), meta, request.tenancyId()));
```

Note: `parseMessages` does not currently have access to the `WebhookRequest`. The `request` must be threaded through from `doHandle()`. Change `parseMessages(String body)` to `parseMessages(String body, WebhookRequest request)` and pass `request` from the call site in `doHandle()`. Alternatively, extract `tenancyId` once in `doHandle()` and pass it as a `String` parameter: `parseMessages(String body, String tenancyId)`.

The cleaner approach: `parseMessages(final String body, final String tenancyId)`. In `doHandle()`:

```java
final String tenancyId = request.tenancyId();
// ...
final List<InboundMessage> messages = parseMessages(request.body(), tenancyId);
```

And in `parseMessages`, the construction becomes:

```java
messages.add(new InboundMessage(ID, InboundConnectorTypes.SLACK,
        user, channel, text, List.of(), Instant.now(), meta, tenancyId));
```

Add import: `import io.casehub.connectors.InboundConnectorTypes;`

**TeamsInboundConnector.java** (~line 108) — update `parseMessage()` similarly. Thread `tenancyId` through:

```java
private WebhookResult parseMessage(final String body, final String tenancyId) {
```

In `doHandle()`:

```java
return parseMessage(request.body(), request.tenancyId());
```

Construction:

```java
List.of(new InboundMessage(InboundConnectorIds.TEAMS_INBOUND, InboundConnectorTypes.TEAMS,
        senderId, channelId, text, List.of(), Instant.now(), Map.of(), tenancyId))
```

Replace `ID` with `InboundConnectorIds.TEAMS_INBOUND` in the constructor call.
Add import: `import io.casehub.connectors.InboundConnectorIds;` and `import io.casehub.connectors.InboundConnectorTypes;`

**TwilioSmsInboundConnector.java** (~line 96):

```java
return new WebhookResult.Delivered(
        List.of(new InboundMessage(ID, InboundConnectorTypes.SMS,
                from, to, body, List.of(), Instant.now(), meta, request.tenancyId())));
```

The `request` variable is already in scope in `doHandle()`. Add import for `InboundConnectorTypes`.

**WhatsAppInboundConnector.java** (~line 156) — `extractMessages` needs `tenancyId` threaded through. In `doHandle()`:

```java
final List<InboundMessage> messages = parseMessages(request.body(), request.tenancyId());
```

Change `parseMessages(String body)` to `parseMessages(String body, String tenancyId)`.
Change `extractMessages(JsonObject value, List<InboundMessage> out)` to `extractMessages(JsonObject value, List<InboundMessage> out, String tenancyId)`.

Construction in `extractMessages`:

```java
out.add(new InboundMessage(ID, InboundConnectorTypes.WHATSAPP,
        from, phoneNumberId, content, List.of(), Instant.now(), meta, tenancyId));
```

Add import for `InboundConnectorTypes`.

- [ ] **Step 5: Fix EmailInboundConnector production sites**

In `toInboundMessage()` (~line 208), happy path:

```java
return new InboundMessage(
        ID,
        InboundConnectorTypes.EMAIL,
        extractSenderId(msg),
        extractChannelRef(msg, account),
        extracted.content(),
        extracted.attachments(),
        resolveReceivedAt(msg),
        buildMetadata(account, msg, extracted.attachments().size()),
        account.tenancyId());
```

Error fallback (~line 218):

```java
return new InboundMessage(ID, InboundConnectorTypes.EMAIL,
        "", account.username(), "",
        List.of(), Instant.now(), Map.of("account-id", account.id()),
        account.tenancyId());
```

Add import for `InboundConnectorTypes`.

- [ ] **Step 6: Run full build — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All modules compile and all tests pass

- [ ] **Step 7: Commit**

```
git add core/ webhook/ email-inbound/
git commit -m "feat(core): add connectorType + tenancyId to InboundMessage, migrate all call sites — Refs #20"
```

---

### Task 5: cloud-events submodule — ConnectorCloudEventAdapter (TDD)

**Files:**
- Create: `cloud-events/pom.xml`
- Create: `cloud-events/src/main/java/io/casehub/connectors/cloudevents/ConnectorCloudEventAdapter.java`
- Create: `cloud-events/src/test/java/io/casehub/connectors/cloudevents/ConnectorCloudEventAdapterTest.java`
- Modify: `pom.xml` (parent — add module)

- [ ] **Step 1: Create cloud-events/pom.xml**

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

  <artifactId>casehub-connectors-cloud-events</artifactId>
  <name>CaseHub Connectors — CloudEvents Adapter</name>
  <description>Optional CDI adapter: observes InboundMessage and fires CloudEvent.
Activates by classpath presence. Depends on casehub-platform-api for cloudevents-core.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
      <version>0.2-SNAPSHOT</version>
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

In the root `pom.xml`, add `<module>cloud-events</module>` after `<module>slack-bot</module>`.

- [ ] **Step 3: Write the adapter test**

Create `cloud-events/src/test/java/io/casehub/connectors/cloudevents/ConnectorCloudEventAdapterTest.java`:

```java
package io.casehub.connectors.cloudevents;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.cloudevents.CloudEvent;

class ConnectorCloudEventAdapterTest {

    private final List<CloudEvent> fired = new ArrayList<>();
    private ConnectorCloudEventAdapter adapter;

    @BeforeEach
    void setUp() {
        jakarta.enterprise.event.Event<CloudEvent> mockEvent = new jakarta.enterprise.event.Event<>() {
            @Override public void fire(CloudEvent event) { fired.add(event); }
            @Override public CompletionStage<CloudEvent> fireAsync(CloudEvent event) {
                fired.add(event);
                return CompletableFuture.completedFuture(event);
            }
            @Override public CompletionStage<CloudEvent> fireAsync(CloudEvent event,
                    jakarta.enterprise.event.NotificationOptions options) {
                return fireAsync(event);
            }
            @Override public jakarta.enterprise.event.Event<CloudEvent> select(
                    java.lang.annotation.Annotation... qualifiers) { return this; }
            @Override public <U extends CloudEvent> jakarta.enterprise.event.Event<U> select(
                    Class<U> subtype, java.lang.annotation.Annotation... qualifiers) { throw new UnsupportedOperationException(); }
            @Override public <U extends CloudEvent> jakarta.enterprise.event.Event<U> select(
                    jakarta.enterprise.util.TypeLiteral<U> subtype, java.lang.annotation.Annotation... qualifiers) { throw new UnsupportedOperationException(); }
        };
        adapter = new ConnectorCloudEventAdapter(mockEvent, new ObjectMapper());
    }

    @Test
    void onMessage_producesCloudEventWithCorrectType() {
        InboundMessage msg = slackMessage("tenant-1");
        adapter.onMessage(msg);

        assertThat(fired).hasSize(1);
        CloudEvent ce = fired.get(0);
        assertThat(ce.getType()).isEqualTo("io.casehub.connectors.inbound.slack");
    }

    @Test
    void onMessage_sourceContainsConnectorId() {
        adapter.onMessage(slackMessage(null));

        CloudEvent ce = fired.get(0);
        assertThat(ce.getSource().toString()).isEqualTo("/casehub-connectors/slack-inbound");
    }

    @Test
    void onMessage_subjectContainsChannelRef() {
        adapter.onMessage(slackMessage(null));

        CloudEvent ce = fired.get(0);
        assertThat(ce.getSubject()).isEqualTo("channel/C456");
    }

    @Test
    void onMessage_tenancyIdSetWhenPresent() {
        adapter.onMessage(slackMessage("tenant-42"));

        CloudEvent ce = fired.get(0);
        assertThat(ce.getExtension("tenancyid")).isEqualTo("tenant-42");
    }

    @Test
    void onMessage_tenancyIdOmittedWhenNull() {
        adapter.onMessage(slackMessage(null));

        CloudEvent ce = fired.get(0);
        assertThat(ce.getExtension("tenancyid")).isNull();
    }

    @Test
    void onMessage_dataIsJsonSerialised() {
        adapter.onMessage(slackMessage(null));

        CloudEvent ce = fired.get(0);
        assertThat(ce.getData()).isNotNull();
        assertThat(ce.getData().toBytes().length).isGreaterThan(0);
        assertThat(ce.getDataContentType()).isEqualTo("application/json");
    }

    @Test
    void onMessage_timeFromReceivedAt() {
        Instant now = Instant.now();
        InboundMessage msg = new InboundMessage(
                InboundConnectorIds.SLACK_INBOUND, InboundConnectorTypes.SLACK,
                "U123", "C456", "hello", List.of(), now, Map.of(), null);
        adapter.onMessage(msg);

        CloudEvent ce = fired.get(0);
        assertThat(ce.getTime()).isNotNull();
        assertThat(ce.getTime().toInstant()).isEqualTo(now);
    }

    @Test
    void onMessage_idIsUUID() {
        adapter.onMessage(slackMessage(null));

        CloudEvent ce = fired.get(0);
        assertThat(ce.getId()).matches("[0-9a-f\\-]{36}");
    }

    private static InboundMessage slackMessage(String tenancyId) {
        return new InboundMessage(
                InboundConnectorIds.SLACK_INBOUND, InboundConnectorTypes.SLACK,
                "U123", "C456", "hello", List.of(), Instant.now(), Map.of(), tenancyId);
    }
}
```

- [ ] **Step 4: Run test — verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cloud-events -Dtest=ConnectorCloudEventAdapterTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Compilation failure — `ConnectorCloudEventAdapter` does not exist

- [ ] **Step 5: Implement ConnectorCloudEventAdapter**

Create `cloud-events/src/main/java/io/casehub/connectors/cloudevents/ConnectorCloudEventAdapter.java`:

```java
package io.casehub.connectors.cloudevents;

import java.net.URI;
import java.time.ZoneOffset;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.connectors.InboundMessage;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;

@ApplicationScoped
public class ConnectorCloudEventAdapter {

    private static final Logger LOG = Logger.getLogger(ConnectorCloudEventAdapter.class);

    private final Event<CloudEvent> cloudEventBus;
    private final ObjectMapper objectMapper;

    @Inject
    public ConnectorCloudEventAdapter(Event<CloudEvent> cloudEventBus, ObjectMapper objectMapper) {
        this.cloudEventBus = cloudEventBus;
        this.objectMapper = objectMapper;
    }

    public void onMessage(@ObservesAsync InboundMessage message) {
        byte[] data;
        try {
            data = objectMapper.writeValueAsBytes(message);
        } catch (JsonProcessingException e) {
            LOG.warnf("Failed to serialise InboundMessage for CloudEvent — connector=%s: %s",
                    message.connectorId(), e.getMessage());
            data = new byte[0];
        }

        CloudEventBuilder builder = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType("io.casehub.connectors.inbound." + message.connectorType())
                .withSource(URI.create("/casehub-connectors/" + message.connectorId()))
                .withSubject("channel/" + message.externalChannelRef())
                .withTime(message.receivedAt().atOffset(ZoneOffset.UTC))
                .withDataContentType("application/json")
                .withData(data);

        if (message.tenancyId() != null) {
            builder = builder.withExtension("tenancyid", message.tenancyId());
        }

        cloudEventBus.fireAsync(builder.build())
                .exceptionally(ex -> {
                    LOG.warnf(ex, "CloudEvent dispatch failed for connector=%s",
                            message.connectorId());
                    return null;
                });
    }
}
```

- [ ] **Step 6: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cloud-events -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests pass

- [ ] **Step 7: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All modules compile and pass

- [ ] **Step 8: Commit**

```
git add cloud-events/ pom.xml
git commit -m "feat: add casehub-connectors-cloud-events submodule with ConnectorCloudEventAdapter — Refs #20"
```

---

### Task 6: Fix IoTCloudEventAdapter (on iot main)

**Files:**
- Modify: `api/src/main/java/io/casehub/iot/api/IoTCloudEventAdapter.java`

- [ ] **Step 1: Rewrite IoTCloudEventAdapter**

Replace the entire class body of `api/src/main/java/io/casehub/iot/api/IoTCloudEventAdapter.java`:

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.net.URI;
import java.time.ZoneOffset;
import java.util.UUID;

@ApplicationScoped
public class IoTCloudEventAdapter {

    private static final Logger LOG = Logger.getLogger(IoTCloudEventAdapter.class);
    private static final URI SOURCE = URI.create("/casehub-iot");
    private static final String TYPE_PREFIX = "io.casehub.iot.state_change.";

    private final Event<CloudEvent> cloudEvents;
    private final ObjectMapper objectMapper;

    @Inject
    public IoTCloudEventAdapter(Event<CloudEvent> cloudEvents, ObjectMapper objectMapper) {
        this.cloudEvents = cloudEvents;
        this.objectMapper = objectMapper;
    }

    void onStateChange(@ObservesAsync StateChangeEvent event) {
        String deviceClass = event.after().deviceClass().name().toLowerCase();
        byte[] data;
        try {
            data = objectMapper.writeValueAsBytes(event);
        } catch (JsonProcessingException e) {
            LOG.warnf("Failed to serialise StateChangeEvent for CloudEvent — device=%s: %s",
                    event.after().deviceId(), e.getMessage());
            data = new byte[0];
        }

        CloudEventBuilder builder = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(TYPE_PREFIX + deviceClass)
                .withSource(SOURCE)
                .withSubject("device/" + event.after().deviceId())
                .withTime(event.occurredAt().atOffset(ZoneOffset.UTC))
                .withData("application/json", data)
                .withExtension("providerid", event.providerId());

        if (event.after().tenancyId() != null) {
            builder = builder.withExtension("tenancyid", event.after().tenancyId());
        }

        cloudEvents.fireAsync(builder.build())
                .exceptionally(ex -> {
                    LOG.warnf(ex, "CloudEvent dispatch failed for device=%s",
                            event.after().deviceId());
                    return null;
                });
    }
}
```

- [ ] **Step 2: Build and test IoT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: All tests pass

- [ ] **Step 3: Commit on iot main**

```
git -C /Users/mdproctor/claude/casehub/iot add api/src/main/java/io/casehub/iot/api/IoTCloudEventAdapter.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "fix: IoTCloudEventAdapter — inject ObjectMapper, null-safe tenancyId, handle serialisation + fireAsync — Closes #IOT_ISSUE

Four fixes to align with canonical CloudEvent adapter pattern:
1. Inject ObjectMapper (was private static — isolated from app config)
2. Null-safe tenancyId extension (defence in depth)
3. Serialisation error: catch at WARN + empty data (was throw from @ObservesAsync — silently swallowed)
4. fireAsync: add .exceptionally() handler (was unhandled CompletionStage)

Refs casehubio/connectors#20"
```

Replace `#IOT_ISSUE` with the actual issue number from Task 1.

---

### Task 7: Fix QhorusCloudEventAdapter + InboundMessage test migrations (on qhorus main)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapter.java`
- Modify: 7 test files in `connector-backend/src/test/` and `slack-channel/src/test/`

- [ ] **Step 1: Install connectors SNAPSHOT to local Maven cache**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/connectors/pom.xml -DskipTests`
Expected: `casehub-connectors-core-0.2-SNAPSHOT.jar` installed to `~/.m2/repository/io/casehub/casehub-connectors-core/0.2-SNAPSHOT/`

- [ ] **Step 2: Fix QhorusCloudEventAdapter**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapter.java`, change `onMessageReceived`:

```java
public void onMessageReceived(@ObservesAsync MessageReceivedEvent event) {
    cloudEventBus.fireAsync(toCloudEvent(event))
            .exceptionally(ex -> {
                LOG.warnf(ex, "CloudEvent dispatch failed for channel=%s type=%s",
                        event.channelId(), event.messageType());
                return null;
            });
}
```

- [ ] **Step 3: Migrate test construction sites**

All 14 sites need `connectorType` (2nd arg) and `tenancyId` (last arg). The import for `InboundConnectorTypes` is needed in each file.

**ConnectorAutoChannelBackendTest.java** — update `smsMsg()` helper:

```java
private InboundMessage smsMsg(String sender, String content) {
    return new InboundMessage(CONNECTOR, InboundConnectorTypes.SMS,
            sender, "+14155550000", content, List.of(), Instant.now(), Map.of(), null);
}
```

Add import: `import io.casehub.connectors.InboundConnectorTypes;`

**ConfiguredAutoChannelPolicyTest.java** — update `smsMsg()`, `emailMsg()`, and inline constructions:

```java
private InboundMessage smsMsg(String sender) {
    return new InboundMessage(InboundConnectorIds.TWILIO_SMS, InboundConnectorTypes.SMS,
            sender, "+14155550000", "hello", List.of(), Instant.now(), Map.of(), null);
}

private InboundMessage emailMsg(String sender) {
    return new InboundMessage(InboundConnectorIds.EMAIL, InboundConnectorTypes.EMAIL,
            sender, "support@company.com", "hello", List.of(), Instant.now(), Map.of(), null);
}
```

Line 113 — fix protocol violation AND add new fields:

```java
InboundMessage slackMsg = new InboundMessage(InboundConnectorIds.SLACK_INBOUND,
        InboundConnectorTypes.SLACK, "U12345",
        "C67890", "hi", List.of(), Instant.now(), Map.of(), null);
```

Line 160 — WhatsApp:

```java
InboundMessage waMsg = new InboundMessage(InboundConnectorIds.WHATSAPP,
        InboundConnectorTypes.WHATSAPP, "+44791100001",
        "+14155550000", "hi", List.of(), Instant.now(), Map.of(), null);
```

Add imports: `import io.casehub.connectors.InboundConnectorTypes;` and `import java.util.List;`

**ConcurrentAutoChannelTest.java** — both inline constructions:

```java
InboundMessage msg1 = new InboundMessage(CONNECTOR, InboundConnectorTypes.SMS,
        SENDER, "+14155550000", "first", List.of(), Instant.now(), Map.of(), null);
InboundMessage msg2 = new InboundMessage(CONNECTOR, InboundConnectorTypes.SMS,
        SENDER, "+14155550000", "second", List.of(), Instant.now(), Map.of(), null);
```

Add import: `import io.casehub.connectors.InboundConnectorTypes;`

**ConnectorKeyStrategyTest.java** — update `msg()` helper:

```java
private InboundMessage msg(String connectorId, String senderId, String channelRef) {
    return new InboundMessage(connectorId, "test-type",
            senderId, channelRef, "content", List.of(), Instant.now(), Map.of(), null);
}
```

**ConnectorChannelBackendTest.java** — both inline constructions:

```java
InboundMessage msg = new InboundMessage(InboundConnectorIds.TWILIO_SMS,
        InboundConnectorTypes.SMS, "+1234", "+9999",
        "hello", List.of(), Instant.now(), Map.of(), null);
```

(Apply same pattern to both sites.) Add import: `import io.casehub.connectors.InboundConnectorTypes;`

**ConnectorChannelBackendIntegrationTest.java** — all 3 inline constructions:

```java
InboundMessage msg = new InboundMessage(InboundConnectorIds.TWILIO_SMS,
        InboundConnectorTypes.SMS, "+15551110000",
        "+14155550100", "hello", List.of(), Instant.now(), Map.of(), null);
```

(Apply same pattern to all 3 sites.) Add import: `import io.casehub.connectors.InboundConnectorTypes;`

**SlackChannelBackendTest.java** — line 319:

```java
io.casehub.connectors.InboundMessage msg = new io.casehub.connectors.InboundMessage(
        io.casehub.connectors.InboundConnectorIds.SLACK_INBOUND,
        io.casehub.connectors.InboundConnectorTypes.SLACK,
        "U123", slackChannelId, "hello", java.util.List.of(),
        java.time.Instant.now(), meta, null);
```

- [ ] **Step 4: Build and test qhorus**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: All tests pass

- [ ] **Step 5: Commit on qhorus main**

```
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapter.java connector-backend/ slack-channel/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix: QhorusCloudEventAdapter fireAsync + InboundMessage constructor migration — Closes #QHORUS_ISSUE

1. Handle fireAsync CompletionStage (was silently swallowed)
2. Migrate ~14 test sites to 9-arg InboundMessage constructor (connectorType + tenancyId)
3. Fix ConfiguredAutoChannelPolicyTest:113 — use InboundConnectorIds.SLACK_INBOUND constant

Refs casehubio/connectors#20"
```

Replace `#QHORUS_ISSUE` with the actual issue number from Task 1.

---

### Task 8: Garden protocol + PLATFORM.md

**Files:**
- Create: garden protocol entry (in `~/.hortora/garden`)
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`

- [ ] **Step 1: Create garden protocol entry**

Invoke `forage` CAPTURE with this content:

**Title:** Canonical CloudEvent adapter pattern — 6 rules for CDI async adapters
**Category:** technique
**Domain:** quarkus, cdi, cloudevents

Content: Six rules every CloudEvent adapter in the platform must satisfy:
1. ObjectMapper injected — static instance isolates from app's shared mapper
2. tenancyId null-safe — extension omitted when null, SDK serialises null as literal `"null"` or throws NPE
3. fireAsync always `.exceptionally()` — unhandled CompletionStage swallows downstream failures
4. Serialisation error: catch at WARN, fire with `new byte[0]` — never throw from `@ObservesAsync` (CDI swallows the exception silently on managed executor thread)
5. type from stable semantic field, not instance identifier — routing rules must survive renames
6. Severity WARN for degraded paths — SEVERE causes alert fatigue for non-catastrophic failures

tenancyId always from event record, never `CurrentPrincipal` (inactive in async observers). Reference implementations: `QhorusCloudEventAdapter`, `ConnectorCloudEventAdapter`.

- [ ] **Step 2: Update PLATFORM.md Capability Ownership table**

In the row for "Inbound message reception (webhook push + IMAP pull)", verify connectors CloudEvent adapter is mentioned. If not already present in the `CloudEvent` typed event envelope row, add `casehub-connectors` to the list of producers:

Find the line:
```
Produced by: `platform-streams-*` modules (external transports), `casehub-iot` (StateChangeEvent adapter), `casehub-qhorus` (MessageReceivedEvent adapter)
```

Add `casehub-connectors` (InboundMessage adapter) to this list.

- [ ] **Step 3: Commit PLATFORM.md**

```
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add casehub-connectors to CloudEvent adapter producers — Refs casehubio/connectors#20"
```

---

### Task 9: Update repo handoffs, ARC42STORIES, and cross-repo artifacts

**Files:**
- IoT: `ARC42STORIES.MD` or equivalent design doc (if exists)
- Qhorus: `ARC42STORIES.MD` (if adapter section exists)
- Connectors: `ARC42STORIES.MD` (update for new cloud-events submodule)

- [ ] **Step 1: Update connectors ARC42STORIES.MD**

Read `ARC42STORIES.MD` and update:
- §5 Building Block View: add `casehub-connectors-cloud-events` submodule
- §9 Journeys: update CloudEvent adapter chapter if exists, or add one
- Any pending/open status references that this work closes

- [ ] **Step 2: Update IoT handoff**

Update `/Users/mdproctor/claude/casehub/iot/HANDOFF.md` (or workspace equivalent) noting the IoT adapter fix commit.

- [ ] **Step 3: Update Qhorus handoff**

Update qhorus workspace handoff noting the Qhorus adapter fix + test migration commit.

- [ ] **Step 4: Commit all handoff updates**

Commit each to its respective workspace on main.
