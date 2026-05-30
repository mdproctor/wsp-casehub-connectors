# ConnectorChannelBackend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `InboundMessage` CDI events into Qhorus channels via `HumanParticipatingChannelBackend` so messages received by any connector reach Qhorus agents.

**Architecture:** Two-repo implementation. (1) `casehub-connectors`: switch `InboundConnectorService` from `Event.fire()` to `Event.fireAsync()` and update test capture beans. (2) `casehub-qhorus`: add `ChannelConnectorBinding` JPA entity + `ChannelBindingStore` SPI, extend `ChannelService` and `ChannelDetail`, and ship a new `connector-backend` module containing `ConnectorChannelBackend` — a single `@ApplicationScoped` `HumanParticipatingChannelBackend` bean that self-registers per bound channel, routes inbound async events, and delivers outbound via `ConnectorService`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI Events (`@ObservesAsync`), JPA/Panache, Flyway (V14), Micrometer, Mockito (`@InjectMock`, `timeout()`), AssertJ, JUnit 5

---

## File Map

### casehub-connectors (branch: `issue-6-connector-channel-backend`)

| Action | File |
|--------|------|
| Modify | `core/src/main/java/io/casehub/connectors/InboundConnectorService.java` |
| Modify | `webhook/src/test/java/io/casehub/connectors/webhook/InboundMessageCapture.java` |
| Modify | `webhook/src/test/java/io/casehub/connectors/webhook/WebhookRouterTest.java` |
| Modify | `email-inbound/src/test/java/io/casehub/connectors/email/inbound/InboundMessageCapture.java` |
| Modify | `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java` |

### casehub-qhorus (branch: `issue-219-connector-channel-backend`)

| Action | File |
|--------|------|
| Create | `runtime/src/main/resources/db/qhorus/migration/V14__channel_connector_binding.sql` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConnectorBinding.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/store/ChannelBindingStore.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelBindingStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java` |
| Modify | `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java` |
| Create | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java` |
| Create | `testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java` |
| Create | `testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java` |
| Modify | `pom.xml` (root — add connector-backend module) |
| Create | `connector-backend/pom.xml` |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorKeyStrategy.java` |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/OutboundTitle.java` |
| Create | `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java` |
| Create | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorKeyStrategyTest.java` |
| Create | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/OutboundTitleTest.java` |
| Create | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendTest.java` |
| Create | `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendIntegrationTest.java` |
| Create | `connector-backend/src/test/resources/application.properties` |

### casehub-parent (direct on main)
| Action | File |
|--------|------|
| Modify | `docs/PLATFORM.md` |
| Modify | `docs/protocols/casehub/cross-foundation-bridge-module-placement.md` |

---

## PART 1 — casehub-connectors

*Working directory: `/Users/mdproctor/claude/casehub/connectors`*  
*Branch: `issue-6-connector-channel-backend`*

---

### Task 1: Update webhook InboundMessageCapture to @ObservesAsync

`@ObservesAsync` only fires when the event is delivered via `Event.fireAsync()`. Changing to `@ObservesAsync` now — with ICS still using `fire()` — will make the webhook tests fail, proving the change is needed.

**Files:**
- Modify: `webhook/src/test/java/io/casehub/connectors/webhook/InboundMessageCapture.java`
- Modify: `webhook/src/test/java/io/casehub/connectors/webhook/WebhookRouterTest.java`

- [ ] **Step 1: Rewrite InboundMessageCapture (webhook)**

```java
package io.casehub.connectors.webhook;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.ArrayList;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;

import io.casehub.connectors.InboundMessage;

@ApplicationScoped
public class InboundMessageCapture {

    private final BlockingQueue<InboundMessage> queue = new LinkedBlockingQueue<>();

    public void observe(@ObservesAsync InboundMessage message) {
        queue.offer(message);
    }

    /** Blocks until one message arrives or timeout elapses. Returns null on timeout. */
    public InboundMessage poll(long timeout, TimeUnit unit) throws InterruptedException {
        return queue.poll(timeout, unit);
    }

    /** Drains all messages currently in the queue (non-blocking). */
    public List<InboundMessage> drainAll() {
        List<InboundMessage> result = new ArrayList<>();
        queue.drainTo(result);
        return result;
    }

    public void clear() {
        queue.clear();
    }
}
```

- [ ] **Step 2: Update WebhookRouterTest — async assertions**

Replace every `capture.received()` call with the blocking `poll()` pattern. The full test file:

```java
package io.casehub.connectors.webhook;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.equalTo;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Base64;
import java.util.HexFormat;
import java.util.TreeMap;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import io.casehub.connectors.InboundMessage;

@QuarkusTest
class WebhookRouterTest {

    @Inject
    InboundMessageCapture capture;

    @BeforeEach
    void reset() {
        capture.clear();
    }

    // ── Unknown connector ─────────────────────────────────────────────────────

    @Test
    void post_unknownConnectorId_returns404() {
        given()
            .contentType("application/json")
            .body("{}")
            .when().post("/connectors/nonexistent-connector/webhook")
            .then().statusCode(404);
    }

    // ── Slack — POST ──────────────────────────────────────────────────────────

    @Test
    void post_slackValid_returns200AndFiresEvent() throws InterruptedException {
        final String ts = nowTs();
        final String body = slackMessageEvent("U123", "C456", "hello from router test");
        final String sig = slackSig(body, ts, "test-slack-secret");

        given()
            .contentType("application/json")
            .header("X-Slack-Signature", sig)
            .header("X-Slack-Request-Timestamp", ts)
            .body(body)
            .when().post("/connectors/slack-inbound/webhook")
            .then().statusCode(200);

        InboundMessage msg = capture.poll(2, TimeUnit.SECONDS);
        assertThat(msg).isNotNull();
        assertThat(msg.content()).isEqualTo("hello from router test");
    }

    @Test
    void post_slackInvalidSignature_returns200WithNoEvent() throws InterruptedException {
        given()
            .contentType("application/json")
            .header("X-Slack-Signature", "v0=badhash")
            .header("X-Slack-Request-Timestamp", nowTs())
            .body(slackMessageEvent("U1", "C1", "hi"))
            .when().post("/connectors/slack-inbound/webhook")
            .then().statusCode(200);

        InboundMessage msg = capture.poll(200, TimeUnit.MILLISECONDS);
        assertThat(msg).isNull();
    }

    @Test
    void post_slackUrlVerification_returnsChallengeJson() throws InterruptedException {
        final String body = "{\"type\":\"url_verification\",\"challenge\":\"test-tok\"}";

        given()
            .contentType("application/json")
            .body(body)
            .when().post("/connectors/slack-inbound/webhook")
            .then().statusCode(200)
            .body("challenge", equalTo("test-tok"));

        InboundMessage msg = capture.poll(200, TimeUnit.MILLISECONDS);
        assertThat(msg).isNull();
    }
```

Continue applying the same pattern for the remaining tests in the file — any `capture.received()` call becomes `capture.poll(2, TimeUnit.SECONDS)`, any "empty" assertion becomes `capture.poll(200, TimeUnit.MILLISECONDS)` asserting null. Keep the rest of the test class (helper methods like `slackSig`, `nowTs`, `slackMessageEvent`, `buildTwilioBody`, etc.) unchanged — copy them from the original file.

- [ ] **Step 3: Run webhook tests — confirm they fail with timeout**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook -Dtest=WebhookRouterTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: tests that wait for messages time out (2s). Tests that assert "no message" pass immediately (they poll 200ms and get null — which is correct since ICS fires synchronously, so @ObservesAsync receives nothing). The "event expected" tests should FAIL.

---

### Task 2: Update email-inbound InboundMessageCapture to @ObservesAsync

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/InboundMessageCapture.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java`

- [ ] **Step 1: Rewrite email-inbound InboundMessageCapture**

```java
package io.casehub.connectors.email.inbound;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;

import io.casehub.connectors.InboundMessage;

@ApplicationScoped
public class InboundMessageCapture {

    private final BlockingQueue<InboundMessage> queue = new LinkedBlockingQueue<>();

    public void observe(@ObservesAsync InboundMessage message) {
        queue.offer(message);
    }

    public InboundMessage poll(long timeout, TimeUnit unit) throws InterruptedException {
        return queue.poll(timeout, unit);
    }

    public void clear() {
        queue.clear();
    }
}
```

- [ ] **Step 2: Update EmailInboundConnectorQuarkusTest — async assertions**

Read the current `EmailInboundConnectorQuarkusTest` to see exact assertion patterns. Replace each `capture.messages().get(0)` / `capture.messages()` pattern with `capture.poll(2, TimeUnit.SECONDS)`. Any assertion on the list size for zero messages becomes `assertThat(capture.poll(200, TimeUnit.MILLISECONDS)).isNull()`.

Open the file to read its current form:
```bash
cat /Users/mdproctor/claude/casehub/connectors/email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java
```

Then apply the same blocking-queue pattern to every assertion site.

---

### Task 3: Switch ICS to fireAsync() — confirm all tests pass — commit

**Files:**
- Modify: `core/src/main/java/io/casehub/connectors/InboundConnectorService.java`

- [ ] **Step 1: Change the CDI constructor in InboundConnectorService**

In `InboundConnectorService.java`, locate the CDI constructor (the one annotated `@Inject` that takes `Event<InboundMessage> messageEvent`). Replace `messageEvent::fire` with an async-dispatching lambda:

Change:
```java
@Inject
InboundConnectorService(@All final List<InboundConnector> pullConnectors,
                        final Event<InboundMessage> messageEvent) {
    this(pullConnectors, messageEvent::fire);
}
```

To:
```java
@Inject
InboundConnectorService(@All final List<InboundConnector> pullConnectors,
                        final Event<InboundMessage> messageEvent) {
    this(pullConnectors, msg -> messageEvent.fireAsync(msg)
            .exceptionally(ex -> {
                LOG.severe("Async InboundMessage dispatch failed: " + ex.getMessage());
                return null;
            }));
}
```

The `LOG` field is already present (`java.util.logging.Logger`). The `Consumer<InboundMessage>` lambda compiles because the block has void return type.

Also update the class-level Javadoc comment that currently says "fires a synchronous `Event<InboundMessage>`" — change to "fires an asynchronous `Event<InboundMessage>` via `fireAsync()`" and add: "`@ObservesAsync InboundMessage` observers are required. `@Observes` observers will not receive events."

- [ ] **Step 2: Run full connector test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: all tests pass including `WebhookRouterTest`, `EmailInboundConnectorQuarkusTest`. The `InboundConnectorServiceTest` uses the package-private constructor (direct `Consumer<InboundMessage>`), bypasses CDI, and is unaffected — it should also pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
    core/src/main/java/io/casehub/connectors/InboundConnectorService.java \
    webhook/src/test/java/io/casehub/connectors/webhook/InboundMessageCapture.java \
    webhook/src/test/java/io/casehub/connectors/webhook/WebhookRouterTest.java \
    email-inbound/src/test/java/io/casehub/connectors/email/inbound/InboundMessageCapture.java \
    email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#6): switch InboundConnectorService to fireAsync(); update test captures to @ObservesAsync"
```

- [ ] **Step 4: Publish connectors SNAPSHOT to local Maven repo**

qhorus needs the updated connectors SNAPSHOT to compile against `@ObservesAsync InboundMessage` in its integration test.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/connectors/pom.xml -DskipTests
```

---

## PART 2 — casehub-qhorus

*Working directory: `/Users/mdproctor/claude/casehub/qhorus`*  
*Create branch: `issue-219-connector-channel-backend`*

---

### Task 4: Create qhorus branch

- [ ] **Step 1: Create branch in qhorus project repo**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus checkout -b issue-219-connector-channel-backend
```

- [ ] **Step 2: Create matching workspace branch**

```bash
git -C /Users/mdproctor/claude/public/casehub/connectors checkout -b issue-219-connector-channel-backend
```

Note: the qhorus workspace (if configured) would be at `/Users/mdproctor/claude/public/casehub/qhorus`. Check whether it exists and create the branch there too if so.

---

### Task 5: Flyway V14 migration + ChannelConnectorBinding entity

The migration and entity are co-authored: write the SQL first, then the entity that maps to it.

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V14__channel_connector_binding.sql`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConnectorBinding.java`

- [ ] **Step 1: Write V14 migration SQL**

```sql
-- V14__channel_connector_binding.sql
-- Binds a Qhorus channel to an external connector for bi-directional message routing.
-- Exactly one binding per channel (channel_id is PK). One binding per connector+key pair
-- (uq_binding_key). external_key holds the per-conversation identifier: Slack channel ID
-- for Slack, sender's phone/email for SMS/WhatsApp/Email.
CREATE TABLE channel_connector_binding (
    channel_id            UUID         NOT NULL,
    inbound_connector_id  VARCHAR(64)  NOT NULL,
    external_key          VARCHAR(255) NOT NULL,   -- 255 covers RFC 5321 email (practical max 254)
    outbound_connector_id VARCHAR(64)  NOT NULL,
    outbound_destination  VARCHAR(512) NOT NULL,

    CONSTRAINT pk_channel_connector_binding
        PRIMARY KEY (channel_id),
    CONSTRAINT fk_binding_channel
        FOREIGN KEY (channel_id) REFERENCES channel(id),
    CONSTRAINT uq_binding_key
        UNIQUE (inbound_connector_id, external_key)
);
```

- [ ] **Step 2: Write ChannelConnectorBinding entity**

```java
package io.casehub.qhorus.runtime.channel;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity
@Table(name = "channel_connector_binding",
       uniqueConstraints = @UniqueConstraint(name = "uq_binding_key",
           columnNames = {"inbound_connector_id", "external_key"}))
public class ChannelConnectorBinding extends PanacheEntityBase {

    @Id
    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(name = "inbound_connector_id", nullable = false, length = 64)
    public String inboundConnectorId;

    @Column(name = "external_key", nullable = false, length = 255)
    public String externalKey;

    @Column(name = "outbound_connector_id", nullable = false, length = 64)
    public String outboundConnectorId;

    @Column(name = "outbound_destination", nullable = false, length = 512)
    public String outboundDestination;
}
```

- [ ] **Step 3: Verify migration path — do not run yet, just verify file is in place**

```bash
ls /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/qhorus/migration/
```

Expected: V1–V13 plus the new V14 file.

---

### Task 6: ChannelBindingStore interface + JpaChannelBindingStore

TDD: write the interface first, then the JPA implementation. The InMemory implementation (for tests) comes in Task 7.

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/ChannelBindingStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelBindingStore.java`

- [ ] **Step 1: Write ChannelBindingStore interface**

```java
package io.casehub.qhorus.runtime.store;

import java.util.Optional;
import java.util.UUID;

import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;

public interface ChannelBindingStore {

    Optional<ChannelConnectorBinding> findByChannelId(UUID channelId);

    Optional<ChannelConnectorBinding> findByKey(String inboundConnectorId, String externalKey);

    void put(ChannelConnectorBinding binding);

    void delete(UUID channelId);
}
```

- [ ] **Step 2: Write JpaChannelBindingStore**

```java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;

@ApplicationScoped
public class JpaChannelBindingStore implements ChannelBindingStore {

    @Override
    public Optional<ChannelConnectorBinding> findByChannelId(UUID channelId) {
        return ChannelConnectorBinding.findByIdOptional(channelId);
    }

    @Override
    public Optional<ChannelConnectorBinding> findByKey(String inboundConnectorId, String externalKey) {
        return ChannelConnectorBinding
                .find("inboundConnectorId = ?1 AND externalKey = ?2",
                        inboundConnectorId, externalKey)
                .firstResultOptional();
    }

    @Override
    @Transactional
    public void put(ChannelConnectorBinding binding) {
        binding.persistAndFlush();
    }

    @Override
    @Transactional
    public void delete(UUID channelId) {
        ChannelConnectorBinding.deleteById(channelId);
    }
}
```

---

### Task 7: InMemoryChannelBindingStore + contract test

**Files:**
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java`
- Create: `testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java`
- Create: `testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java`

- [ ] **Step 1: Write the contract test (abstract, runs against any implementation)**

```java
package io.casehub.qhorus.testing.contract;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;

public abstract class ChannelBindingStoreContractTest {

    protected abstract ChannelBindingStore store();

    @BeforeEach
    void reset() {
        // Subclasses may override to clear state
    }

    private ChannelConnectorBinding binding(UUID channelId, String connectorId,
                                            String externalKey, String outConnectorId,
                                            String destination) {
        ChannelConnectorBinding b = new ChannelConnectorBinding();
        b.channelId = channelId;
        b.inboundConnectorId = connectorId;
        b.externalKey = externalKey;
        b.outboundConnectorId = outConnectorId;
        b.outboundDestination = destination;
        return b;
    }

    @Test
    void put_andFindByChannelId_returnsBinding() {
        UUID id = UUID.randomUUID();
        store().put(binding(id, "twilio-sms-inbound", "+15551110000", "twilio-sms", "+15551110000"));
        assertThat(store().findByChannelId(id)).isPresent()
                .get().extracting(b -> b.externalKey).isEqualTo("+15551110000");
    }

    @Test
    void put_andFindByKey_returnsBinding() {
        UUID id = UUID.randomUUID();
        store().put(binding(id, "slack-inbound", "C123456", "slack", "https://hooks.slack.com/x"));
        assertThat(store().findByKey("slack-inbound", "C123456")).isPresent()
                .get().extracting(b -> b.channelId).isEqualTo(id);
    }

    @Test
    void findByChannelId_absentChannel_returnsEmpty() {
        assertThat(store().findByChannelId(UUID.randomUUID())).isEmpty();
    }

    @Test
    void findByKey_absentKey_returnsEmpty() {
        assertThat(store().findByKey("slack-inbound", "C_NONE")).isEmpty();
    }

    @Test
    void delete_removesBinding() {
        UUID id = UUID.randomUUID();
        store().put(binding(id, "email-inbound", "user@example.com", "email", "user@example.com"));
        store().delete(id);
        assertThat(store().findByChannelId(id)).isEmpty();
        assertThat(store().findByKey("email-inbound", "user@example.com")).isEmpty();
    }

    @Test
    void put_overwritesExistingBinding() {
        UUID id = UUID.randomUUID();
        store().put(binding(id, "twilio-sms-inbound", "+1111", "twilio-sms", "+1111"));
        store().put(binding(id, "twilio-sms-inbound", "+1111", "twilio-sms", "+2222"));
        assertThat(store().findByChannelId(id)).isPresent()
                .get().extracting(b -> b.outboundDestination).isEqualTo("+2222");
    }
}
```

- [ ] **Step 2: Write InMemoryChannelBindingStore**

```java
package io.casehub.qhorus.testing;

import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryChannelBindingStore implements ChannelBindingStore {

    private final ConcurrentHashMap<UUID, ChannelConnectorBinding> byChannelId = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ChannelConnectorBinding> byKey = new ConcurrentHashMap<>();

    private String key(String connectorId, String externalKey) {
        return connectorId + " " + externalKey;
    }

    @Override
    public Optional<ChannelConnectorBinding> findByChannelId(UUID channelId) {
        return Optional.ofNullable(byChannelId.get(channelId));
    }

    @Override
    public Optional<ChannelConnectorBinding> findByKey(String inboundConnectorId, String externalKey) {
        return Optional.ofNullable(byKey.get(key(inboundConnectorId, externalKey)));
    }

    @Override
    public void put(ChannelConnectorBinding binding) {
        // Remove old key mapping if channelId already had a different binding
        ChannelConnectorBinding old = byChannelId.get(binding.channelId);
        if (old != null) {
            byKey.remove(key(old.inboundConnectorId, old.externalKey));
        }
        byChannelId.put(binding.channelId, binding);
        byKey.put(key(binding.inboundConnectorId, binding.externalKey), binding);
    }

    @Override
    public void delete(UUID channelId) {
        ChannelConnectorBinding removed = byChannelId.remove(channelId);
        if (removed != null) {
            byKey.remove(key(removed.inboundConnectorId, removed.externalKey));
        }
    }

    /** For test setup — clears all state. */
    public void clear() {
        byChannelId.clear();
        byKey.clear();
    }
}
```

- [ ] **Step 3: Write InMemoryChannelBindingStoreTest**

```java
package io.casehub.qhorus.testing;

import io.casehub.qhorus.testing.contract.ChannelBindingStoreContractTest;
import org.junit.jupiter.api.BeforeEach;

class InMemoryChannelBindingStoreTest extends ChannelBindingStoreContractTest {

    private final InMemoryChannelBindingStore subject = new InMemoryChannelBindingStore();

    @Override
    protected InMemoryChannelBindingStore store() {
        return subject;
    }

    @Override
    @BeforeEach
    void reset() {
        subject.clear();
    }
}
```

- [ ] **Step 4: Run testing module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all `InMemoryChannelBindingStoreTest` tests pass.

---

### Task 8: ChannelCreateRequest + ChannelService additions

TDD: write unit tests for the new ChannelService methods using the InMemory store, then implement.

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`

- [ ] **Step 1: Write ChannelCreateRequest record**

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.ChannelSemantic;

/**
 * Request record for creating a Qhorus channel with an optional connector binding.
 * If {@code inboundConnectorId} is non-null, all four binding fields must be non-null.
 */
public record ChannelCreateRequest(
        String name,
        String description,
        ChannelSemantic semantic,
        String barrierContributors,
        String allowedWriters,
        String adminInstances,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        String allowedTypes,
        // Connector binding — all four non-null together, or all null
        String inboundConnectorId,
        String externalKey,
        String outboundConnectorId,
        String outboundDestination
) {
    public ChannelCreateRequest {
        if (inboundConnectorId != null) {
            if (externalKey == null || outboundConnectorId == null || outboundDestination == null) {
                throw new IllegalArgumentException(
                        "Connector binding requires all four fields: inboundConnectorId, " +
                        "externalKey, outboundConnectorId, outboundDestination");
            }
        }
    }

    public boolean hasConnectorBinding() {
        return inboundConnectorId != null;
    }

    /** Convenience factory — no connector binding. */
    public static ChannelCreateRequest simple(String name, ChannelSemantic semantic) {
        return new ChannelCreateRequest(name, null, semantic, null, null, null,
                null, null, null, null, null, null, null);
    }
}
```

- [ ] **Step 2: Add ChannelService methods — write the tests first**

In the existing `ChannelService`, add two methods. But first: is there an existing `ChannelServiceTest`? Check:

```bash
find /Users/mdproctor/claude/casehub/qhorus -name "ChannelServiceTest.java" | grep -v target
```

If it exists, add the new test cases to it. If not, the tests for ChannelService are likely in the integration tests. For now, add the following test cases directly to the connector-backend integration test (Task 15) which covers `findByConnectorKey`. 

The `create(ChannelCreateRequest)` method is exercised by the integration test as well — it's the setup step. The duplicate-binding validation test is in Task 13 (unit test).

- [ ] **Step 3: Implement ChannelService additions**

Add `@Inject ChannelBindingStore bindingStore;` field to ChannelService.

Add `findByConnectorKey` method:
```java
public Optional<Channel> findByConnectorKey(String inboundConnectorId, String externalKey) {
    return bindingStore.findByKey(inboundConnectorId, externalKey)
            .flatMap(b -> channelStore.find(b.channelId));
}
```

Add `create(ChannelCreateRequest)` method:
```java
@Transactional
public Channel create(final ChannelCreateRequest req) {
    if (req.hasConnectorBinding()) {
        bindingStore.findByKey(req.inboundConnectorId(), req.externalKey()).ifPresent(existing -> {
            throw new IllegalStateException(
                    "Connector binding already exists: " + req.inboundConnectorId()
                    + " / " + req.externalKey() + " → channel " + existing.channelId);
        });
    }
    // Reuse the existing create chain
    Channel channel = create(req.name(), req.description(), req.semantic(),
            req.barrierContributors(), req.allowedWriters(), req.adminInstances(),
            req.rateLimitPerChannel(), req.rateLimitPerInstance(), req.allowedTypes());

    if (req.hasConnectorBinding()) {
        final ChannelConnectorBinding binding = new ChannelConnectorBinding();
        binding.channelId = channel.id;
        binding.inboundConnectorId = req.inboundConnectorId();
        binding.externalKey = req.externalKey();
        binding.outboundConnectorId = req.outboundConnectorId();
        binding.outboundDestination = req.outboundDestination();
        bindingStore.put(binding);
    }
    return channel;
}
```

---

### Task 9: Update ChannelDetail + QhorusEntityMapper + QhorusDashboardService

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`

- [ ] **Step 1: Add ConnectorBinding nested record to ChannelDetail**

The existing `ChannelDetail` record has 13 fields. Add a 14th nullable field `connectorBinding`:

```java
package io.casehub.qhorus.api.channel;

import java.util.UUID;

public record ChannelDetail(
        UUID channelId,
        String name,
        String description,
        String semantic,
        String barrierContributors,
        long messageCount,
        String lastActivityAt,
        boolean paused,
        String allowedWriters,
        String adminInstances,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        String allowedTypes,
        ConnectorBinding connectorBinding   // null when channel has no connector binding
) {
    /** API-level connector binding descriptor — subset of ChannelConnectorBinding entity. */
    public record ConnectorBinding(
            String inboundConnectorId,
            String externalKey,
            String outboundConnectorId,
            String outboundDestination) {}
}
```

This is a binary-incompatible change to the record's canonical constructor. All call sites that construct `new ChannelDetail(...)` directly will fail to compile. They all go through `QhorusEntityMapper.toChannelDetail()`.

- [ ] **Step 2: Update QhorusEntityMapper**

`QhorusEntityMapper` needs to inject `ChannelBindingStore` to populate the binding field. Add the injection and update `toChannelDetail`:

```java
// Add field:
@Inject
ChannelBindingStore bindingStore;

// Update method:
public ChannelDetail toChannelDetail(Channel ch, long messageCount) {
    ChannelDetail.ConnectorBinding binding = bindingStore.findByChannelId(ch.id)
            .map(b -> new ChannelDetail.ConnectorBinding(
                    b.inboundConnectorId, b.externalKey,
                    b.outboundConnectorId, b.outboundDestination))
            .orElse(null);
    return new ChannelDetail(
            ch.id, ch.name, ch.description,
            ch.semantic != null ? ch.semantic.name() : null,
            ch.barrierContributors, messageCount,
            ch.lastActivityAt != null ? ch.lastActivityAt.toString() : null,
            ch.paused, ch.allowedWriters, ch.adminInstances,
            ch.rateLimitPerChannel, ch.rateLimitPerInstance, ch.allowedTypes,
            binding);
}
```

- [ ] **Step 3: Update QhorusDashboardService.toChannelDetail()**

The `QhorusDashboardService` has its own `toChannelDetail()` (line ~145) that constructs `ChannelDetail` directly. It also needs `ChannelBindingStore`. However, since `QhorusMcpToolsBase.toChannelDetail()` already delegates to `QhorusEntityMapper`, and the dashboard has its own implementation, we have two paths to fix.

In `QhorusDashboardService.java`:

Add `@Inject ChannelBindingStore bindingStore;` field.

Update `toChannelDetail(Channel ch, int count)`:
```java
private ChannelDetail toChannelDetail(Channel ch, int count) {
    ChannelDetail.ConnectorBinding binding = bindingStore.findByChannelId(ch.id)
            .map(b -> new ChannelDetail.ConnectorBinding(
                    b.inboundConnectorId, b.externalKey,
                    b.outboundConnectorId, b.outboundDestination))
            .orElse(null);
    return new ChannelDetail(
            ch.id, ch.name, ch.description,
            ch.semantic != null ? ch.semantic.name() : null,
            ch.barrierContributors, count,
            ch.lastActivityAt != null ? ch.lastActivityAt.toString() : null,
            ch.paused, ch.allowedWriters, ch.adminInstances,
            ch.rateLimitPerChannel, ch.rateLimitPerInstance, ch.allowedTypes,
            binding);
}
```

- [ ] **Step 4: Run runtime build to confirm binary break is fully resolved**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: `BUILD SUCCESS`. If any `ChannelDetail(...)` constructor call sites are still failing, find and fix them now.

- [ ] **Step 5: Run qhorus test suite (all modules except connector-backend which doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,testing -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all existing tests pass.

- [ ] **Step 6: Commit data layer**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    runtime/src/main/resources/db/qhorus/migration/V14__channel_connector_binding.sql \
    runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConnectorBinding.java \
    runtime/src/main/java/io/casehub/qhorus/runtime/store/ChannelBindingStore.java \
    runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelBindingStore.java \
    runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java \
    runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java \
    api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java \
    runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java \
    runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java \
    testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java \
    testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java \
    testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#219): ChannelConnectorBinding entity, ChannelBindingStore SPI, V14 migration, ChannelCreateRequest, ChannelDetail.ConnectorBinding"
```

---

### Task 10: Create connector-backend Maven module

**Files:**
- Create: `connector-backend/pom.xml`
- Modify: `pom.xml` (root)

- [ ] **Step 1: Create connector-backend/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-qhorus-connector-backend</artifactId>
  <name>Quarkus Qhorus - Connector Backend Bridge</name>
  <description>Optional bridge — ConnectorChannelBackend routes InboundMessage CDI events to Qhorus channels via HumanParticipatingChannelBackend. Activates by classpath presence.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-runtime</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- InboundMessage, ConnectorService, ConnectorMessage -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>

    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-core</artifactId>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>org.jboss.logging</groupId>
      <artifactId>jboss-logging</artifactId>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5-mockito</artifactId>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-testing</artifactId>
      <version>${project.version}</version>
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

- [ ] **Step 2: Add connector-backend to root pom.xml module list**

Find the `<modules>` block in the root `pom.xml`. It currently reads:
```xml
<modules>
  <module>api</module>
  <module>connectors</module>
  <module>runtime</module>
  <module>deployment</module>
  <module>testing</module>
  <module>examples</module>
</modules>
```

Add `connector-backend` after `runtime`:
```xml
<modules>
  <module>api</module>
  <module>connectors</module>
  <module>runtime</module>
  <module>connector-backend</module>
  <module>deployment</module>
  <module>testing</module>
  <module>examples</module>
</modules>
```

- [ ] **Step 3: Create required directory structure**

```bash
mkdir -p /Users/mdproctor/claude/casehub/qhorus/connector-backend/src/main/java/io/casehub/qhorus/connector/backend
mkdir -p /Users/mdproctor/claude/casehub/qhorus/connector-backend/src/test/java/io/casehub/qhorus/connector/backend
mkdir -p /Users/mdproctor/claude/casehub/qhorus/connector-backend/src/test/resources
```

---

### Task 11: ConnectorKeyStrategy (TDD)

**Files:**
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorKeyStrategy.java`
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorKeyStrategyTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundMessage;

class ConnectorKeyStrategyTest {

    private InboundMessage msg(String connectorId, String senderId, String channelRef) {
        return new InboundMessage(connectorId, senderId, channelRef, "content", Instant.now(), Map.of());
    }

    @Test
    void slackInbound_usesExternalChannelRef() {
        InboundMessage m = msg("slack-inbound", "U123", "C456");
        assertThat(ConnectorKeyStrategy.deriveKey(m)).isEqualTo("C456");
    }

    @Test
    void twilioSmsInbound_usesExternalSenderId() {
        InboundMessage m = msg("twilio-sms-inbound", "+15551110000", "+14155552671");
        assertThat(ConnectorKeyStrategy.deriveKey(m)).isEqualTo("+15551110000");
    }

    @Test
    void whatsappInbound_usesExternalSenderId() {
        InboundMessage m = msg("whatsapp-inbound", "+44700000000", "phone-number-id");
        assertThat(ConnectorKeyStrategy.deriveKey(m)).isEqualTo("+44700000000");
    }

    @Test
    void emailInbound_usesExternalSenderId() {
        InboundMessage m = msg("email-inbound", "alice@example.com", "inbox@casehub.io");
        assertThat(ConnectorKeyStrategy.deriveKey(m)).isEqualTo("alice@example.com");
    }

    @Test
    void unknownConnector_fallsBackToExternalChannelRef() {
        InboundMessage m = msg("custom-inbound", "sender-id", "channel-ref");
        assertThat(ConnectorKeyStrategy.deriveKey(m)).isEqualTo("channel-ref");
    }
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend -Dtest=ConnectorKeyStrategyTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: compilation error — `ConnectorKeyStrategy` does not exist yet.

- [ ] **Step 3: Write ConnectorKeyStrategy**

```java
package io.casehub.qhorus.connector.backend;

import java.util.Set;

import io.casehub.connectors.InboundMessage;

/**
 * Derives the per-conversation lookup key from an InboundMessage.
 *
 * <p>For Slack, the externalChannelRef IS the conversation space (Slack channel ID).
 * For SMS/WhatsApp/Email, externalChannelRef is our own endpoint; the conversation
 * is with the sender — so externalSenderId is the correct key.
 */
final class ConnectorKeyStrategy {

    private static final Set<String> SENDER_KEYED = Set.of(
            "twilio-sms-inbound",
            "whatsapp-inbound",
            "email-inbound"
    );

    private ConnectorKeyStrategy() {}

    static String deriveKey(final InboundMessage msg) {
        if (SENDER_KEYED.contains(msg.connectorId())) {
            return msg.externalSenderId();
        }
        return msg.externalChannelRef();
    }
}
```

- [ ] **Step 4: Run test — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend -Dtest=ConnectorKeyStrategyTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: 5 tests pass.

---

### Task 12: OutboundTitle (TDD)

**Files:**
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/OutboundTitle.java`
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/OutboundTitleTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.ChannelRef;

class OutboundTitleTest {

    private final ChannelRef channel = new ChannelRef(UUID.randomUUID(), "support-channel");

    @Test
    void email_returnsReChannelName() {
        assertThat(OutboundTitle.forConnector("email", channel))
                .isEqualTo("Re: support-channel");
    }

    @Test
    void slack_returnsNull() {
        assertThat(OutboundTitle.forConnector("slack", channel)).isNull();
    }

    @Test
    void twilioSms_returnsNull() {
        assertThat(OutboundTitle.forConnector("twilio-sms", channel)).isNull();
    }

    @Test
    void whatsapp_returnsNull() {
        assertThat(OutboundTitle.forConnector("whatsapp", channel)).isNull();
    }

    @Test
    void teams_returnsNull() {
        assertThat(OutboundTitle.forConnector("teams", channel)).isNull();
    }

    @Test
    void unknownConnector_returnsNull() {
        assertThat(OutboundTitle.forConnector("custom-outbound", channel)).isNull();
    }
}
```

- [ ] **Step 2: Run to confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend -Dtest=OutboundTitleTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: compilation error.

- [ ] **Step 3: Write OutboundTitle**

```java
package io.casehub.qhorus.connector.backend;

import io.casehub.qhorus.api.gateway.ChannelRef;

/**
 * Per-connector outbound title strategy.
 *
 * <p>TODO(v2, qhorus#216): email threading — proper subjects should carry the original
 * In-Reply-To Message-ID header. Implement when per-connector normaliser work lands.
 */
final class OutboundTitle {

    private OutboundTitle() {}

    static String forConnector(final String outboundConnectorId, final ChannelRef channel) {
        if ("email".equals(outboundConnectorId)) {
            return "Re: " + channel.name();
        }
        return null;
    }
}
```

- [ ] **Step 4: Run test — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend -Dtest=OutboundTitleTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: 6 tests pass.

---

### Task 13: ConnectorChannelBackend unit tests (TDD — write all tests, then implement)

Write the full unit test class. The class under test does not exist yet — tests will fail to compile until Task 14.

**Files:**
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendTest.java`

- [ ] **Step 1: Write ConnectorChannelBackendTest**

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Disabled;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.InboundMessage;
import io.casehub.qhorus.api.gateway.ChannelInitialisedEvent;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;

class ConnectorChannelBackendTest {

    private ChannelGateway gateway;
    private ChannelService channelService;
    private ChannelBindingStore bindingStore;
    private ConnectorService connectorService;
    private ConnectorChannelBackend backend;

    @BeforeEach
    void setUp() {
        gateway = mock(ChannelGateway.class);
        channelService = mock(ChannelService.class);
        bindingStore = mock(ChannelBindingStore.class);
        connectorService = mock(ConnectorService.class);
        backend = new ConnectorChannelBackend(gateway, channelService, bindingStore,
                connectorService, new SimpleMeterRegistry());
    }

    // ── Registration ──────────────────────────────────────────────────────────

    @Test
    void onChannelInitialised_registersBackend_whenBindingPresent() {
        UUID channelId = UUID.randomUUID();
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "twilio-sms-inbound", "+1111",
                        "twilio-sms", "+1111")));

        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-alice"));

        verify(gateway).deregisterBackend(channelId, "connector-human");
        verify(gateway).registerBackend(eq(channelId), eq(backend), eq("human_participating"));
    }

    @Test
    void onChannelInitialised_skips_whenNoBinding() {
        UUID channelId = UUID.randomUUID();
        when(bindingStore.findByChannelId(channelId)).thenReturn(Optional.empty());

        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "plain-channel"));

        verifyNoInteractions(gateway);
    }

    @Test
    void onChannelInitialised_isIdempotent_secondEventOverwritesCache() {
        UUID channelId = UUID.randomUUID();
        ChannelConnectorBinding b1 = binding(channelId, "twilio-sms-inbound", "+1111", "twilio-sms", "+1111");
        ChannelConnectorBinding b2 = binding(channelId, "twilio-sms-inbound", "+1111", "twilio-sms", "+2222");
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(b1))
                .thenReturn(Optional.of(b2));

        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-alice"));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-alice"));

        // Registered twice (idempotent deregister+register each time)
        verify(gateway, times(2)).deregisterBackend(channelId, "connector-human");
        verify(gateway, times(2)).registerBackend(any(), any(), any());

        // Cache has the latest destination — verify by calling post()
        OutboundMessage outbound = new OutboundMessage(UUID.randomUUID(), "agent",
                MessageType.RESPONSE, "hi", null, null, ActorType.AGENT);
        backend.post(new ChannelRef(channelId, "sms-alice"), outbound);
        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService).send(eq("twilio-sms"), captor.capture());
        assertThat(captor.getValue().destination()).isEqualTo("+2222");
    }

    // ── Inbound routing ───────────────────────────────────────────────────────

    @Test
    void onInboundMessage_routesToReceiveHumanMessage_whenChannelFound() {
        UUID channelId = UUID.randomUUID();
        // Pre-populate the cache by firing ChannelInitialisedEvent
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "twilio-sms-inbound", "+1111",
                        "twilio-sms", "+1111")));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-alice"));

        Channel ch = channel(channelId, "sms-alice");
        when(channelService.findByConnectorKey("twilio-sms-inbound", "+1111"))
                .thenReturn(Optional.of(ch));

        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+1111",
                "+14155552671", "hello", Instant.now(), Map.of());
        backend.onInboundMessage(msg);

        ArgumentCaptor<InboundHumanMessage> captor = ArgumentCaptor.forClass(InboundHumanMessage.class);
        verify(gateway).receiveHumanMessage(eq(new ChannelRef(channelId, "sms-alice")), captor.capture());
        assertThat(captor.getValue().externalSenderId()).isEqualTo("+1111");
        assertThat(captor.getValue().content()).isEqualTo("hello");
    }

    @Test
    void onInboundMessage_noChannelFound_doesNotThrow_andIncrementsCounter() {
        when(channelService.findByConnectorKey(any(), any())).thenReturn(Optional.empty());

        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+9999",
                "+14155552671", "hello", Instant.now(), Map.of());
        assertThat(backend.discardedCount("twilio-sms-inbound")).isZero();

        backend.onInboundMessage(msg);

        verifyNoInteractions(gateway);
        assertThat(backend.discardedCount("twilio-sms-inbound")).isOne();
    }

    // ── Outbound ─────────────────────────────────────────────────────────────

    @Test
    void post_sendsViaConnectorService_fromCache() {
        UUID channelId = UUID.randomUUID();
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "slack-inbound", "C123",
                        "slack", "https://hooks.slack.com/x")));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "slack-support"));

        OutboundMessage outbound = new OutboundMessage(UUID.randomUUID(), "agent",
                MessageType.RESPONSE, "We can help", null, null, ActorType.AGENT);
        backend.post(new ChannelRef(channelId, "slack-support"), outbound);

        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService).send(eq("slack"), captor.capture());
        assertThat(captor.getValue().destination()).isEqualTo("https://hooks.slack.com/x");
        assertThat(captor.getValue().body()).isEqualTo("We can help");
        assertThat(captor.getValue().title()).isNull(); // Slack gets no title
    }

    @Test
    void post_emailConnector_includesReSubjectTitle() {
        UUID channelId = UUID.randomUUID();
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "email-inbound", "alice@ex.com",
                        "email", "alice@ex.com")));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "support-email"));

        backend.post(new ChannelRef(channelId, "support-email"),
                new OutboundMessage(UUID.randomUUID(), "agent", MessageType.RESPONSE,
                        "Got your message", null, null, ActorType.AGENT));

        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService).send(eq("email"), captor.capture());
        assertThat(captor.getValue().title()).isEqualTo("Re: support-email");
    }

    @Test
    void post_noCacheEntry_doesNotThrow_logsError() {
        // No ChannelInitialisedEvent fired — cache is empty
        UUID channelId = UUID.randomUUID();
        backend.post(new ChannelRef(channelId, "unknown"),
                new OutboundMessage(UUID.randomUUID(), "agent", MessageType.RESPONSE,
                        "content", null, null, ActorType.AGENT));
        verifyNoInteractions(connectorService);
    }

    @Test
    void post_connectorServiceThrows_doesNotPropagate() {
        UUID channelId = UUID.randomUUID();
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "twilio-sms-inbound", "+1",
                        "nonexistent-connector", "+1")));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "ch"));
        doThrow(new IllegalArgumentException("No connector 'nonexistent-connector'"))
                .when(connectorService).send(any(), any());

        // Must not throw
        backend.post(new ChannelRef(channelId, "ch"),
                new OutboundMessage(UUID.randomUUID(), "agent", MessageType.RESPONSE,
                        "hi", null, null, ActorType.AGENT));
    }

    // ── Close ─────────────────────────────────────────────────────────────────

    @Test
    void close_removesCacheEntry_outboundDropsAfterClose() {
        UUID channelId = UUID.randomUUID();
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "twilio-sms-inbound", "+1111",
                        "twilio-sms", "+1111")));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-alice"));

        backend.close(new ChannelRef(channelId, "sms-alice"));

        // post() after close — cache miss, must not throw, must not send
        backend.post(new ChannelRef(channelId, "sms-alice"),
                new OutboundMessage(UUID.randomUUID(), "agent", MessageType.RESPONSE,
                        "hi", null, null, ActorType.AGENT));
        verifyNoInteractions(connectorService);
    }

    // ── Staleness documentation ───────────────────────────────────────────────

    @Disabled("v2: cache not invalidated on binding update — enable when ChannelInitialisedEvent " +
              "fires on binding update (qhorus#215)")
    @Test
    void cacheRefreshesAfterBindingUpdate() {
        UUID channelId = UUID.randomUUID();
        // Initial binding: destination = "+1111"
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "twilio-sms-inbound", "+1111",
                        "twilio-sms", "+1111")));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-alice"));

        // Binding updated externally: destination = "+2222"
        when(bindingStore.findByChannelId(channelId))
                .thenReturn(Optional.of(binding(channelId, "twilio-sms-inbound", "+1111",
                        "twilio-sms", "+2222")));
        // No ChannelInitialisedEvent fired — cache not refreshed

        backend.post(new ChannelRef(channelId, "sms-alice"),
                new OutboundMessage(UUID.randomUUID(), "agent", MessageType.RESPONSE,
                        "hi", null, null, ActorType.AGENT));
        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService).send(eq("twilio-sms"), captor.capture());
        // This fails in v1 — cache still has "+1111"
        assertThat(captor.getValue().destination()).isEqualTo("+2222");
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private ChannelConnectorBinding binding(UUID channelId, String inConnId, String extKey,
                                             String outConnId, String dest) {
        ChannelConnectorBinding b = new ChannelConnectorBinding();
        b.channelId = channelId;
        b.inboundConnectorId = inConnId;
        b.externalKey = extKey;
        b.outboundConnectorId = outConnId;
        b.outboundDestination = dest;
        return b;
    }

    private Channel channel(UUID id, String name) {
        Channel ch = new Channel();
        ch.id = id;
        ch.name = name;
        return ch;
    }
}
```

- [ ] **Step 2: Run test — confirm compile failure (ConnectorChannelBackend missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl connector-backend -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep "error:"
```

Expected: compiler errors about missing `ConnectorChannelBackend` class and the `discardedCount` method.

---

### Task 14: Implement ConnectorChannelBackend

**Files:**
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java`

- [ ] **Step 1: Write ConnectorChannelBackend**

```java
package io.casehub.qhorus.connector.backend;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.InboundMessage;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.ChannelInitialisedEvent;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.HumanParticipatingChannelBackend;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;
import io.micrometer.core.instrument.MeterRegistry;
import org.jboss.logging.Logger;

/**
 * Routes {@link InboundMessage} CDI events to Qhorus channels and delivers outbound
 * messages back via {@link ConnectorService}.
 *
 * <p>Self-registers for channels that have a {@link ChannelConnectorBinding} via
 * {@link ChannelInitialisedEvent}. Activates by classpath presence — no configuration needed.
 *
 * <p><strong>v1 constraint:</strong> channels must be pre-provisioned with a binding before
 * the first inbound message arrives. Auto-creation on first contact is tracked in qhorus#214.
 * Changing binding fields on an existing channel requires a service restart to refresh the
 * in-memory cache (qhorus#215).
 */
@ApplicationScoped
public class ConnectorChannelBackend implements HumanParticipatingChannelBackend {

    static final String BACKEND_ID = "connector-human";

    private static final Logger LOG = Logger.getLogger(ConnectorChannelBackend.class);

    private final ConcurrentHashMap<UUID, CacheEntry> bindingCache = new ConcurrentHashMap<>();

    private final ChannelGateway gateway;
    private final ChannelService channelService;
    private final ChannelBindingStore bindingStore;
    private final ConnectorService connectorService;
    private final MeterRegistry meterRegistry;

    @Inject
    public ConnectorChannelBackend(final ChannelGateway gateway,
                                   final ChannelService channelService,
                                   final ChannelBindingStore bindingStore,
                                   final ConnectorService connectorService,
                                   final MeterRegistry meterRegistry) {
        this.gateway = gateway;
        this.channelService = channelService;
        this.bindingStore = bindingStore;
        this.connectorService = connectorService;
        this.meterRegistry = meterRegistry;
    }

    @Override public String backendId() { return BACKEND_ID; }
    @Override public ActorType actorType() { return ActorType.HUMAN; }
    @Override public void open(final ChannelRef channel, final Map<String, String> metadata) {} // no-op

    @Override
    public void close(final ChannelRef channel) {
        bindingCache.remove(channel.id());
    }

    void onChannelInitialised(@Observes final ChannelInitialisedEvent event) {
        bindingStore.findByChannelId(event.channelId()).ifPresent(binding -> {
            bindingCache.put(event.channelId(), new CacheEntry(
                    binding.inboundConnectorId, binding.externalKey,
                    binding.outboundConnectorId, binding.outboundDestination));
            gateway.deregisterBackend(event.channelId(), BACKEND_ID);
            gateway.registerBackend(event.channelId(), this, "human_participating");
        });
    }

    void onInboundMessage(@ObservesAsync final InboundMessage msg) {
        final String key = ConnectorKeyStrategy.deriveKey(msg);
        channelService.findByConnectorKey(msg.connectorId(), key).ifPresentOrElse(
                channel -> gateway.receiveHumanMessage(
                        new ChannelRef(channel.id, channel.name),
                        new InboundHumanMessage(msg.externalSenderId(), msg.content(),
                                msg.receivedAt(), msg.metadata(), null, null)),
                () -> {
                    LOG.warnf("No channel bound to %s / %s — discarding", msg.connectorId(), key);
                    meterRegistry.counter("inbound_messages_discarded_total",
                            "connector_id", msg.connectorId()).increment();
                });
    }

    @Override
    public void post(final ChannelRef channel, final OutboundMessage message) {
        final CacheEntry entry = bindingCache.get(channel.id());
        if (entry == null) {
            LOG.errorf("No cache entry for channel %s — outbound dropped", channel.id());
            return;
        }
        final String title = OutboundTitle.forConnector(entry.outboundConnectorId(), channel);
        try {
            connectorService.send(entry.outboundConnectorId(),
                    new ConnectorMessage(entry.outboundDestination(), title, message.content()));
        } catch (final IllegalArgumentException ex) {
            LOG.errorf("Unknown outbound connector '%s' for channel %s — outbound dropped",
                    entry.outboundConnectorId(), channel.id());
        }
    }

    /** Exposed for testing — returns discard counter value for the given connector. */
    double discardedCount(final String connectorId) {
        return meterRegistry.counter("inbound_messages_discarded_total",
                "connector_id", connectorId).count();
    }

    private record CacheEntry(String inboundConnectorId, String externalKey,
                               String outboundConnectorId, String outboundDestination) {}
}
```

- [ ] **Step 2: Run unit tests — confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend -Dtest="ConnectorKeyStrategyTest,OutboundTitleTest,ConnectorChannelBackendTest" -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all pass; `cacheRefreshesAfterBindingUpdate` is skipped (disabled).

---

### Task 15: ConnectorChannelBackendIntegrationTest

**Files:**
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendIntegrationTest.java`
- Create: `connector-backend/src/test/resources/application.properties`

- [ ] **Step 1: Write application.properties for the test module**

```properties
# connector-backend test application.properties
# Use H2 in-memory for Flyway; InMemory*Store beans replace JPA at test time
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:qhorus_test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.flyway.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
quarkus.flyway.migrate-at-start=true

# Keep Quarkus from trying to connect to the qhorus named datasource
quarkus.datasource."qhorus".db-kind=h2
quarkus.datasource."qhorus".jdbc.url=jdbc:h2:mem:qhorus_test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Step 2: Write integration test**

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.InboundConnectorService;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ConnectorChannelBackendIntegrationTest {

    @Inject ConnectorChannelBackend backend;
    @Inject ChannelService channelService;
    @Inject ChannelGateway gateway;
    @Inject InboundConnectorService inboundConnectorService;

    @InjectMock MessageService messageService;
    @InjectMock ConnectorService connectorService;

    private UUID channelId;

    @BeforeEach
    void setUp() {
        // Create a connector-bound channel
        Channel ch = channelService.create(new ChannelCreateRequest(
                "sms-alice", "Alice's SMS conversation", ChannelSemantic.STANDARD,
                null, null, null, null, null, null,
                "twilio-sms-inbound", "+15551110000", "twilio-sms", "+15551110000"));
        channelId = ch.id;
        // initChannel fires ChannelInitialisedEvent → ConnectorChannelBackend.onChannelInitialised()
        gateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name));
    }

    @Test
    void inboundMessage_routesToReceiveHumanMessage_andDispatchesToMessageService() {
        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+15551110000",
                "+14155552671", "I need help", Instant.now(), Map.of());

        inboundConnectorService.receive(msg);

        // @ObservesAsync — wait up to 2s for async delivery
        verify(messageService, timeout(2000)).dispatch(argThat(d ->
                d.channelId().equals(channelId)
                && "human:+15551110000".equals(d.sender())
                && d.actorType() == ActorType.HUMAN
                && "I need help".equals(d.content())));
    }

    @Test
    void outboundFanOut_sendsViaConnectorService() throws InterruptedException {
        // First ensure the backend is registered by triggering an inbound route
        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+15551110000",
                "+14155552671", "ping", Instant.now(), Map.of());
        inboundConnectorService.receive(msg);
        verify(messageService, timeout(2000)).dispatch(any());

        // Simulate fanOut (normally triggered by MessageService after persistence)
        OutboundMessage outbound = new OutboundMessage(UUID.randomUUID(), "agent",
                MessageType.RESPONSE, "We can help", null, null, ActorType.AGENT);
        gateway.fanOut(channelId, "sms-alice", outbound);

        // fanOut is async (virtual thread) — wait for ConnectorService.send()
        Thread.sleep(300);
        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService, atLeastOnce()).send(eq("twilio-sms"), captor.capture());
        assertThat(captor.getValue().destination()).isEqualTo("+15551110000");
        assertThat(captor.getValue().body()).isEqualTo("We can help");
    }

    @Test
    void unknownSender_noChannelBound_noDispatch_counterIncremented() {
        double before = backend.discardedCount("twilio-sms-inbound");

        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+99999",
                "+14155552671", "hello", Instant.now(), Map.of());
        inboundConnectorService.receive(msg);

        // Give async observer time to run
        verify(messageService, after(500).never()).dispatch(any());
        assertThat(backend.discardedCount("twilio-sms-inbound")).isGreaterThan(before);
    }

    @Test
    void duplicateBinding_createChannelWithSameKey_throws() {
        assertThatThrownBy(() ->
            channelService.create(new ChannelCreateRequest(
                "sms-bob", "Bob's channel", ChannelSemantic.STANDARD,
                null, null, null, null, null, null,
                "twilio-sms-inbound", "+15551110000",  // same key as alice!
                "twilio-sms", "+15551110000"))
        ).isInstanceOf(IllegalStateException.class)
         .hasMessageContaining("Connector binding already exists");
    }
}
```

Note: `assertThatThrownBy` needs the AssertJ static import `import static org.assertj.core.api.Assertions.assertThatThrownBy;`. Add it.

- [ ] **Step 3: Run the integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend -Dtest=ConnectorChannelBackendIntegrationTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all 4 tests pass.

---

### Task 16: Run full qhorus build + commit

- [ ] **Step 1: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all tests pass across api, connectors, runtime, testing, connector-backend modules.

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
    pom.xml \
    connector-backend/ \
    testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java \
    testing/src/test/java/io/casehub/qhorus/testing/InMemoryChannelBindingStoreTest.java \
    testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#219): casehub-qhorus-connector-backend — ConnectorChannelBackend, ConnectorKeyStrategy, OutboundTitle, InMemoryChannelBindingStore"
```

---

### Task 17: Update PLATFORM.md

**Files:**
- Modify: `docs/PLATFORM.md` in casehub-parent repo

- [ ] **Step 1: Fix stale Flyway version**

Find the line that says `next domain migration: V11` in the Qhorus tables section and change to `V15` (V14 is now used by this feature).

Find and update: `Qhorus tables | casehub-qhorus | Flyway V1–V10, V2000 (... next domain migration: V11)`

Correct to: `Flyway V1–V14, V2000 (... next domain migration: V15)`

- [ ] **Step 2: Add cross-repo dependency entry**

In the Cross-Repo Dependency Map table, add after the existing `casehub-connectors-core | casehub-qhorus | connectors` row:

```
| `casehub-connectors-core` | `casehub-qhorus` | `connector-backend` | optional — `InboundMessage → ConnectorChannelBackend` bridge; activates by classpath presence |
```

Also add to the build order comment after `casehub-qhorus`:

```
  casehub-qhorus            (depends on casehub-ledger; connector-backend module optionally depends on casehub-connectors-core)
```

- [ ] **Step 3: Commit parent update**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: update PLATFORM.md — qhorus Flyway V14, connector-backend cross-repo dep"
```

---

### Task 18: Update cross-foundation-bridge-module-placement protocol

**Files:**
- Modify: `docs/protocols/casehub/cross-foundation-bridge-module-placement.md` in casehub-parent

- [ ] **Step 1: Amend the protocol**

The existing protocol says bridges live in the event-source repo. Add a note for the bidirectional case:

Append to the existing rule body:

```
**Exception — bidirectional bridges requiring runtime access:** when a bridge reacts to events from BOTH repos (e.g. ChannelInitialisedEvent from qhorus AND InboundMessage from connectors) AND requires runtime-tier classes (ChannelGateway, ChannelService) that are only available in one repo's runtime module, the bridge lives in the repo that owns the runtime classes. The protocol's goal — avoiding circular Maven dependencies — is still satisfied, because the bridge depends on the other repo's core module (not its runtime), preventing cycles. Precedent: `casehub-qhorus-connector-backend` (in qhorus repo) depends on `casehub-connectors-core` (connectors repo) — not connectors runtime.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/protocols/casehub/cross-foundation-bridge-module-placement.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: extend cross-foundation-bridge-module-placement — bidirectional bridge exception"
```

---

## Self-Review Against Spec

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| ICS switches to fireAsync() (D6) | Task 3 |
| InboundMessageCapture → @ObservesAsync | Tasks 1, 2 |
| ChannelConnectorBinding entity + V14 migration | Task 5 |
| ChannelBindingStore interface | Task 6 |
| JpaChannelBindingStore | Task 6 |
| InMemoryChannelBindingStore + contract test | Task 7 |
| ChannelCreateRequest record | Task 8 |
| ChannelService.findByConnectorKey() | Task 8 |
| ChannelService.create(ChannelCreateRequest) + duplicate check | Task 8 |
| ChannelDetail.ConnectorBinding nested record | Task 9 |
| QhorusEntityMapper updated | Task 9 |
| QhorusDashboardService updated | Task 9 |
| connector-backend module created | Task 10 |
| ConnectorKeyStrategy | Task 11 |
| OutboundTitle + email "Re:" subject | Task 12 |
| ConnectorChannelBackend (registration, inbound, outbound, close) | Task 14 |
| Micrometer discard counter | Task 14 |
| Unit tests for all components | Tasks 11–13 |
| @Disabled staleness regression test | Task 13 |
| Integration test (round-trip, counter, duplicate) | Task 15 |
| PLATFORM.md dep map + Flyway version | Task 17 |
| bridge placement protocol update | Task 18 |
| @BeforeEach @Transactional not needed here (InMemory store, no JPA entity lookups in setup) | ✅ N/A |

**Gaps:** None found.
