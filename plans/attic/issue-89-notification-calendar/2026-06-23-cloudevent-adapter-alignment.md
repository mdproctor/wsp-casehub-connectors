# CloudEvent Adapter Alignment — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move `ConnectorCloudEventAdapter` from the `cloud-events` submodule into `casehub-connectors-core`, rename to `ConnectorsCloudEventAdapter`, align style with IoT reference, and update all documentation.

**Architecture:** Direct `cloudevents-core` dependency on core (not via `casehub-platform-api`). The `cloud-events` submodule is deleted. The adapter activates by CDI discovery — a no-op when no `CloudEvent` observer exists.

**Tech Stack:** Java 21, Quarkus 3.32.2, CloudEvents SDK 4.0.1, Jackson

## Global Constraints

- Java 21 source, Quarkus 3.32.2, `casehub-parent` BOM manages `cloudevents-core` version
- Zero casehubio dependencies in `casehub-connectors-core` — `cloudevents-core` and `jackson-databind` are external libraries
- All 7 rules from GE-20260621-629712 must be satisfied
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- Every commit references issue #23

---

### Task 1: Add dependencies to core and delete cloud-events submodule

**Files:**
- Modify: `core/pom.xml` — add 3 dependencies, update `<description>`
- Modify: `pom.xml` (parent) — remove `<module>cloud-events</module>`
- Delete: `cloud-events/` directory (entire submodule)

**Interfaces:**
- Consumes: nothing
- Produces: core module with `cloudevents-core`, `jackson-databind` on compile classpath and `jackson-datatype-jsr310` on test classpath

- [ ] **Step 1: Add dependencies to `core/pom.xml`**

Add these three dependencies after the existing `quarkus-config-yaml` dependency and before the `<!-- Testing -->` comment:

```xml
    <dependency>
      <groupId>io.cloudevents</groupId>
      <artifactId>cloudevents-core</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
    </dependency>
```

Add this test dependency after the existing `wiremock-standalone` test dependency:

```xml
    <dependency>
      <groupId>com.fasterxml.jackson.datatype</groupId>
      <artifactId>jackson-datatype-jsr310</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Update core `pom.xml` `<description>`**

Replace the `<description>` element:

Old:
```xml
  <description>HTTP-based outbound connectors: Slack, Teams, Twilio SMS, WhatsApp.
No dependencies beyond CDI and java.net.http — usable in any Quarkus application.</description>
```

New:
```xml
  <description>Outbound and inbound message connectors: Slack, Teams, Twilio SMS, WhatsApp.
CDI + external standards (CloudEvents SDK, Jackson). No casehubio dependencies.</description>
```

- [ ] **Step 3: Remove `cloud-events` module from parent `pom.xml`**

Delete this line from the `<modules>` block in the root `pom.xml`:

```xml
    <module>cloud-events</module>
```

- [ ] **Step 4: Delete `cloud-events/` directory**

Delete the entire `cloud-events/` directory.

- [ ] **Step 5: Build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS. All existing tests pass. No module depends on `casehub-connectors-cloud-events`.

- [ ] **Step 6: Commit**

```bash
git add core/pom.xml pom.xml
git rm -r cloud-events/
git commit -m "feat(core): add cloudevents-core + jackson-databind, delete cloud-events submodule — Refs #23"
```

---

### Task 2: Move adapter and test to core

**Files:**
- Create: `core/src/main/java/io/casehub/connectors/ConnectorsCloudEventAdapter.java`
- Create: `core/src/test/java/io/casehub/connectors/ConnectorsCloudEventAdapterTest.java`

**Interfaces:**
- Consumes: `InboundMessage`, `InboundConnectorIds`, `InboundConnectorTypes` from `io.casehub.connectors` (same package)
- Produces: `ConnectorsCloudEventAdapter` CDI bean; fires `Event<CloudEvent>.fireAsync()` when `InboundMessage` is observed

- [ ] **Step 1: Write the test**

Create `core/src/test/java/io/casehub/connectors/ConnectorsCloudEventAdapterTest.java`:

```java
package io.casehub.connectors;

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
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import io.cloudevents.CloudEvent;

class ConnectorsCloudEventAdapterTest {

    private final List<CloudEvent> fired = new ArrayList<>();
    private ConnectorsCloudEventAdapter adapter;

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
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        adapter = new ConnectorsCloudEventAdapter(mockEvent, mapper);
    }

    @Test
    void onMessage_producesCloudEventWithCorrectType() {
        adapter.onMessage(slackMessage("tenant-1"));

        assertThat(fired).hasSize(1);
        assertThat(fired.get(0).getType()).isEqualTo("io.casehub.connectors.inbound.slack");
    }

    @Test
    void onMessage_sourceContainsConnectorId() {
        adapter.onMessage(slackMessage(null));

        assertThat(fired.get(0).getSource().toString()).isEqualTo("/casehub-connectors/slack-inbound");
    }

    @Test
    void onMessage_subjectContainsChannelRef() {
        adapter.onMessage(slackMessage(null));

        assertThat(fired.get(0).getSubject()).isEqualTo("channel/C456");
    }

    @Test
    void onMessage_tenancyIdSetWhenPresent() {
        adapter.onMessage(slackMessage("tenant-42"));

        assertThat(fired.get(0).getExtension("tenancyid")).isEqualTo("tenant-42");
    }

    @Test
    void onMessage_tenancyIdOmittedWhenNull() {
        adapter.onMessage(slackMessage(null));

        assertThat(fired.get(0).getExtension("tenancyid")).isNull();
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

        assertThat(fired.get(0).getId()).matches("[0-9a-f\\-]{36}");
    }

    private static InboundMessage slackMessage(String tenancyId) {
        return new InboundMessage(
                InboundConnectorIds.SLACK_INBOUND, InboundConnectorTypes.SLACK,
                "U123", "C456", "hello", List.of(), Instant.now(), Map.of(), tenancyId);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=ConnectorsCloudEventAdapterTest`

Expected: FAIL — `ConnectorsCloudEventAdapter` class does not exist.

- [ ] **Step 3: Write the adapter**

Create `core/src/main/java/io/casehub/connectors/ConnectorsCloudEventAdapter.java`:

```java
package io.casehub.connectors;

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

import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;

@ApplicationScoped
public class ConnectorsCloudEventAdapter {

    private static final Logger LOG = Logger.getLogger(ConnectorsCloudEventAdapter.class);
    private static final String TYPE_PREFIX = "io.casehub.connectors.inbound.";

    private final Event<CloudEvent> cloudEventBus;
    private final ObjectMapper objectMapper;

    @Inject
    public ConnectorsCloudEventAdapter(Event<CloudEvent> cloudEventBus, ObjectMapper objectMapper) {
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
                .withType(TYPE_PREFIX + message.connectorType())
                .withSource(URI.create("/casehub-connectors/" + message.connectorId()))
                .withSubject("channel/" + message.externalChannelRef())
                .withTime(message.receivedAt().atOffset(ZoneOffset.UTC))
                .withData("application/json", data);

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

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=ConnectorsCloudEventAdapterTest`

Expected: all 8 tests PASS.

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 6: Commit**

```bash
git add core/src/main/java/io/casehub/connectors/ConnectorsCloudEventAdapter.java core/src/test/java/io/casehub/connectors/ConnectorsCloudEventAdapterTest.java
git commit -m "feat(core): add ConnectorsCloudEventAdapter — IoT pattern alignment — Refs #23

Moved from cloud-events submodule, renamed from ConnectorCloudEventAdapter.
Direct cloudevents-core dependency (not via casehub-platform-api).
Style aligned: TYPE_PREFIX constant, 2-arg withData()."
```

---

### Task 3: Update ARC42STORIES.MD

**Files:**
- Modify: `ARC42STORIES.MD`

**Interfaces:**
- Consumes: nothing
- Produces: accurate documentation reflecting the module elimination

- [ ] **Step 1: Update §1 — Top Quality Goals**

In the quality goals table, replace the "Zero-dep core" row:

Old:
```
| Zero-dep core | `java.net.http.HttpClient` only; no framework dep in `core` or `slack-bot` |
```

New:
```
| Lightweight foundation | CDI + external standards only (CloudEvents SDK, Jackson); no casehubio dependencies in `core` or `slack-bot` |
```

- [ ] **Step 2: Update §2 — Constraints**

Replace line 8 header:

Old:
```
**Depends on:** Quarkus BOM + JDK only (core module: JDK only)
```

New:
```
**Depends on:** Quarkus BOM + external standards (core: CDI, cloudevents-core, jackson-databind; zero casehubio deps)
```

Replace the constraint row in the table:

Old:
```
| `core` and `slack-bot` modules: zero non-JDK runtime dependencies | Design principle — library embeds without pulling a framework |
```

New:
```
| `core` and `slack-bot` modules: no casehubio runtime dependencies; external libraries only (Quarkus CDI, CloudEvents SDK, Jackson) | Design principle — foundation tier embeds without coupling to platform |
```

- [ ] **Step 3: Update §4 — Layer Taxonomy L6 row**

Old:
```
| L6 — CloudEvents Adapter | `ConnectorCloudEventAdapter`, `InboundConnectorTypes` — optional submodule; observes `InboundMessage`, fires `CloudEvent` |
```

New:
```
| L6 — CloudEvents Adapter | `ConnectorsCloudEventAdapter`, `InboundConnectorTypes` — in core; observes `InboundMessage`, fires `CloudEvent` |
```

- [ ] **Step 4: Update §5 — Module Structure table and text**

Remove the `cloud-events` row from the module table:

```
| `cloud-events` | `casehub-connectors-cloud-events` | `core`, `casehub-platform-api` |
```

Update the `core` row "Depends on" column:

Old:
```
| `core` | `casehub-connectors-core` | JDK only |
```

New:
```
| `core` | `casehub-connectors-core` | CDI, `cloudevents-core`, `jackson-databind` |
```

Remove the sentence at the end of the module list that explains why `cloud-events` is separate:

```
`cloud-events` is separate because `core` carries zero external deps; adding `casehub-platform-api` (which carries `cloudevents-core`) would break that invariant.
```

- [ ] **Step 5: Update §7 — Deployment View**

Remove the `cloud-events` row from the deployment table:

```
| `cloud-events` | CloudEvent adapter — bridges `InboundMessage` to `Event<CloudEvent>.fireAsync()` for casehub-ras |
```

- [ ] **Step 6: Update §13 — Glossary**

Replace the `ConnectorCloudEventAdapter` glossary entry:

Old:
```
| `ConnectorCloudEventAdapter` | CDI adapter in `cloud-events` module: observes `@ObservesAsync InboundMessage`, fires `Event<CloudEvent>.fireAsync()`. Follows canonical CloudEvent adapter pattern (GE-20260621-629712). |
```

New:
```
| `ConnectorsCloudEventAdapter` | CDI adapter in `core`: observes `@ObservesAsync InboundMessage`, fires `Event<CloudEvent>.fireAsync()`. Follows canonical CloudEvent adapter pattern (GE-20260621-629712). |
```

- [ ] **Step 7: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs: update ARC42STORIES.MD for cloud-events module elimination — Refs #23

§1: rename Zero-dep core → Lightweight foundation
§2: rewrite constraint to name real invariant (no casehubio deps)
§4: L6 now in core, not optional submodule
§5: remove cloud-events row, update core deps
§7: remove cloud-events deployment row
§13: rename glossary entry"
```

---

### Task 4: Update platform docs (casehub-parent)

**Files:**
- Modify: `../parent/docs/repos/casehub-connectors.md` — update core row description and "Depends On"

**Interfaces:**
- Consumes: nothing
- Produces: accurate platform documentation

- [ ] **Step 1: Update core row in module table**

In the module table in `docs/repos/casehub-connectors.md`, append to the core row description. After the existing text ending with `...connectorId() + discover() → List<DiscoveredTarget>.`, add:

```
`ConnectorsCloudEventAdapter` — CDI adapter observing `@ObservesAsync InboundMessage`, fires `Event<CloudEvent>.fireAsync()` with type `io.casehub.connectors.inbound.<connectorType>`. Follows canonical CloudEvent adapter pattern (GE-20260621-629712).
```

- [ ] **Step 2: Update "Depends On" section**

Replace the "Depends On" section content:

Old:
```
Nothing in the casehubio ecosystem. Pure Java (`java.net.http.HttpClient`) + optional `quarkus-mailer` for email outbound + standard IMAP (`jakarta.mail`) for email inbound.
```

New:
```
Nothing in the casehubio ecosystem. Core module: `java.net.http.HttpClient`, `cloudevents-core` (CNCF CloudEvents SDK), `jackson-databind`. Optional modules: `quarkus-mailer` (email outbound), `jakarta.mail` (email inbound).
```

- [ ] **Step 3: Update GE-20260621-629712 reference**

In `~/.hortora/garden/jvm/GE-20260621-629712.md`, in the "Reference implementations" list, replace:

```
- `ConnectorCloudEventAdapter` (casehub-connectors)
```

with:

```
- `ConnectorsCloudEventAdapter` (casehub-connectors)
```

- [ ] **Step 4: Commit platform doc changes**

Commit to casehub-parent repo:

```bash
git -C ../parent add docs/repos/casehub-connectors.md
git -C ../parent commit -m "docs: update casehub-connectors deep-dive for cloud-events module elimination — Refs casehubio/connectors#23"
```

Commit garden entry (separate repo):

```bash
git -C ~/.hortora/garden add jvm/GE-20260621-629712.md
git -C ~/.hortora/garden commit -m "fix: update ConnectorsCloudEventAdapter class name reference"
```
