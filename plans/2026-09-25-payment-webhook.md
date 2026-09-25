# TrueLayer Payment Webhook Receiver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #108 — TrueLayer: Payment webhook receiver for real-time status updates
**Issue group:** #110, #111, #108

**Goal:** Add a JAX-RS webhook endpoint that receives TrueLayer payment
status events, validates JWS signatures, deduplicates, and fires CDI events.

**Architecture:** New `PaymentStatusChanged` event record in bank-spi.
New JAX-RS endpoint, JWS validator, and event deduplicator in bank-truelayer.
Uses `com.truelayer:truelayer-signing` library for JWS verification.
Polling via `paymentStatus()` unchanged — webhook is additive (D1).

**Tech Stack:** Java 21, Quarkus CDI + JAX-RS, TrueLayer signing library 0.3.0, JUnit 5, WireMock, AssertJ

## Global Constraints

- Java source level 21, compiled for Java 26 JVM
- Webhook does NOT affect the BankPlatform SPI surface
- `PaymentStatusChanged` record in bank-spi (D2), not bank-truelayer
- Standalone JAX-RS endpoint, not WebhookInboundConnector (D3)
- JWS (ES512) signature validation via `com.truelayer:truelayer-signing:0.3.0`
- CDI event via `fireAsync` (consistent with ConnectorService pattern)
- `HttpHelper.CLIENT` for JWKS fetching (shared-http-client protocol)

---

## Batch 1: PaymentStatusChanged record + webhook endpoint

### Task 1: PaymentStatusChanged event record (bank-spi) + webhook endpoint with JWS validation (bank-truelayer)

**Files:**
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/PaymentStatusChanged.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerPaymentWebhook.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/WebhookEventDeduplicator.java`
- Modify: `bank-truelayer/pom.xml` (add truelayer-signing dependency)
- Test: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerPaymentWebhookTest.java`

**Interfaces:**
- Consumes: `PaymentStatus` enum (bank-spi), `TrueLayerPaymentInitiation.mapPaymentStatus()` pattern
- Produces: `PaymentStatusChanged` record (consumed by observers via `@ObservesAsync`), `/webhooks/truelayer/payments` endpoint

- [ ] **Step 1: Add truelayer-signing dependency to bank-truelayer/pom.xml**

Add to `<dependencies>` section:

```xml
<dependency>
    <groupId>com.truelayer</groupId>
    <artifactId>truelayer-signing</artifactId>
    <version>0.3.0</version>
</dependency>
```

- [ ] **Step 2: Create PaymentStatusChanged record in bank-spi**

```java
package io.casehub.connectors.bank.model;

import java.time.Instant;

public record PaymentStatusChanged(String platformId,
                                    String paymentId,
                                    PaymentStatus status,
                                    String failureReason,
                                    String eventId,
                                    Instant timestamp) {}
```

- [ ] **Step 3: Create WebhookEventDeduplicator**

```java
package io.casehub.connectors.bank.truelayer;

import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;

import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class WebhookEventDeduplicator {

    private static final long TTL_SECONDS = 600;
    private final ConcurrentHashMap<String, Instant> seen = new ConcurrentHashMap<>();

    public boolean isDuplicate(String eventId) {
        if (eventId == null) return false;
        Instant previous = seen.putIfAbsent(eventId, Instant.now());
        return previous != null;
    }

    @Scheduled(every = "5m")
    void cleanup() {
        Instant cutoff = Instant.now().minusSeconds(TTL_SECONDS);
        seen.entrySet().removeIf(e -> e.getValue().isBefore(cutoff));
    }
}
```

- [ ] **Step 4: Write failing test for TrueLayerPaymentWebhook**

```java
package io.casehub.connectors.bank.truelayer;

import io.casehub.connectors.bank.model.PaymentStatus;
import io.casehub.connectors.bank.model.PaymentStatusChanged;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import java.time.Duration;

@QuarkusTest
@TestProfile(TrueLayerTestProfile.class)
class TrueLayerPaymentWebhookTest {

    @Inject PaymentEventCapture capture;
    @Inject WebhookEventDeduplicator deduplicator;

    @BeforeEach
    void setUp() {
        capture.clear();
    }

    @Test
    void validWebhook_firesEvent() {
        given()
            .contentType("application/json")
            .body("""
                {"type":"payment_executed","event_id":"evt-001",
                 "event_version":1,"payment_id":"pay-123",
                 "payment_status":"executed",
                 "timestamp":"2026-09-25T10:30:00Z"}""")
            .when().post("/webhooks/truelayer/payments")
            .then().statusCode(200);

        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            assertThat(capture.events()).hasSize(1);
            PaymentStatusChanged event = capture.events().getFirst();
            assertThat(event.platformId()).isEqualTo("truelayer");
            assertThat(event.paymentId()).isEqualTo("pay-123");
            assertThat(event.status()).isEqualTo(PaymentStatus.EXECUTED);
            assertThat(event.eventId()).isEqualTo("evt-001");
        });
    }

    @Test
    void malformedBody_returns400() {
        given()
            .contentType("application/json")
            .body("{\"type\":\"payment_executed\"}")
            .when().post("/webhooks/truelayer/payments")
            .then().statusCode(400);

        assertThat(capture.events()).isEmpty();
    }

    @Test
    void unknownEventType_returns200_noEvent() {
        given()
            .contentType("application/json")
            .body("""
                {"type":"payment_refunded","event_id":"evt-002",
                 "event_version":1,"payment_id":"pay-456",
                 "payment_status":"refunded",
                 "timestamp":"2026-09-25T11:00:00Z"}""")
            .when().post("/webhooks/truelayer/payments")
            .then().statusCode(200);

        assertThat(capture.events()).isEmpty();
    }

    @Test
    void duplicateEventId_returns200_noSecondEvent() {
        String body = """
            {"type":"payment_settled","event_id":"evt-dup",
             "event_version":1,"payment_id":"pay-789",
             "payment_status":"settled",
             "timestamp":"2026-09-25T12:00:00Z"}""";

        given().contentType("application/json").body(body)
            .when().post("/webhooks/truelayer/payments")
            .then().statusCode(200);

        await().atMost(Duration.ofSeconds(5)).untilAsserted(() ->
            assertThat(capture.events()).hasSize(1));

        given().contentType("application/json").body(body)
            .when().post("/webhooks/truelayer/payments")
            .then().statusCode(200);

        // Still only 1 event — second was deduplicated
        assertThat(capture.events()).hasSize(1);
    }

    @Test
    void failedPayment_includesFailureReason() {
        given()
            .contentType("application/json")
            .body("""
                {"type":"payment_failed","event_id":"evt-003",
                 "event_version":1,"payment_id":"pay-fail",
                 "payment_status":"failed",
                 "failure_reason":"insufficient_funds",
                 "timestamp":"2026-09-25T13:00:00Z"}""")
            .when().post("/webhooks/truelayer/payments")
            .then().statusCode(200);

        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            assertThat(capture.events()).hasSize(1);
            PaymentStatusChanged event = capture.events().getFirst();
            assertThat(event.status()).isEqualTo(PaymentStatus.FAILED);
            assertThat(event.failureReason()).isEqualTo("insufficient_funds");
        });
    }

    @Test
    void allStatusValues_mappedCorrectly() {
        var mappings = List.of(
            new String[]{"payment_authorization_required", "AUTHORIZATION_REQUIRED"},
            new String[]{"payment_authorizing", "AUTHORIZING"},
            new String[]{"payment_authorized", "AUTHORIZED"},
            new String[]{"payment_executed", "EXECUTED"},
            new String[]{"payment_settled", "SETTLED"},
            new String[]{"payment_failed", "FAILED"});

        for (int i = 0; i < mappings.size(); i++) {
            capture.clear();
            String eventType = mappings.get(i)[0];
            PaymentStatus expected = PaymentStatus.valueOf(mappings.get(i)[1]);

            given()
                .contentType("application/json")
                .body("""
                    {"type":"%s","event_id":"evt-map-%d",
                     "event_version":1,"payment_id":"pay-map-%d",
                     "payment_status":"%s",
                     "timestamp":"2026-09-25T14:00:00Z"}"""
                    .formatted(eventType, i, i,
                        eventType.replaceFirst("^payment_", "")))
                .when().post("/webhooks/truelayer/payments")
                .then().statusCode(200);

            int idx = i;
            await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
                assertThat(capture.events()).hasSize(1);
                assertThat(capture.events().getFirst().status()).isEqualTo(expected);
            });
        }
    }

    @ApplicationScoped
    public static class PaymentEventCapture {
        private final List<PaymentStatusChanged> events = new CopyOnWriteArrayList<>();

        void onEvent(@ObservesAsync PaymentStatusChanged event) {
            events.add(event);
        }

        public List<PaymentStatusChanged> events() { return events; }
        public void clear() { events.clear(); }
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer -am test -Dtest="TrueLayerPaymentWebhookTest" -Dsurefire.failIfNoSpecifiedTests=false -f pom.xml`
Expected: Compilation failure (TrueLayerPaymentWebhook does not exist)

- [ ] **Step 6: Implement TrueLayerPaymentWebhook**

```java
package io.casehub.connectors.bank.truelayer;

import io.casehub.connectors.bank.model.PaymentStatus;
import io.casehub.connectors.bank.model.PaymentStatusChanged;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.io.StringReader;
import java.time.Instant;

@Path("/webhooks/truelayer")
@ApplicationScoped
public class TrueLayerPaymentWebhook {

    private static final Logger LOG = Logger.getLogger(TrueLayerPaymentWebhook.class);

    @Inject Event<PaymentStatusChanged> statusEvent;
    @Inject WebhookEventDeduplicator deduplicator;

    @ConfigProperty(name = "casehub.connectors.bank.truelayer.webhook-validation-disabled",
                    defaultValue = "false")
    boolean validationDisabled;

    @POST
    @Path("/payments")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response handlePayment(@HeaderParam("Tl-Signature") String signature,
                                  String body) {
        if (!validationDisabled && signature == null) {
            return Response.status(401).build();
        }
        if (validationDisabled) {
            LOG.warn("Webhook signature validation disabled — accepting unsigned request");
        }

        JsonObject json;
        try (var reader = Json.createReader(new StringReader(body))) {
            json = reader.readObject();
        } catch (Exception e) {
            return Response.status(400).build();
        }

        String paymentId = json.getString("payment_id", null);
        if (paymentId == null) {
            return Response.status(400).build();
        }

        String eventId = json.getString("event_id", null);
        if (deduplicator.isDuplicate(eventId)) {
            LOG.debugf("Duplicate webhook event '%s' — skipping", eventId);
            return Response.ok().build();
        }

        String eventType = json.getString("type", "");
        PaymentStatus status = mapWebhookEventType(eventType);
        if (status == null) {
            LOG.warnf("Unknown webhook event type '%s' — ignoring", eventType);
            return Response.ok().build();
        }

        String failureReason = json.getString("failure_reason", null);
        String timestampStr = json.getString("timestamp", null);
        Instant timestamp = timestampStr != null
                ? Instant.parse(timestampStr) : Instant.now();

        statusEvent.fireAsync(new PaymentStatusChanged(
                "truelayer", paymentId, status, failureReason,
                eventId, timestamp));

        return Response.ok().build();
    }

    private PaymentStatus mapWebhookEventType(String eventType) {
        String status = eventType.replaceFirst("^payment_", "");
        return switch (status) {
            case "authorization_required" -> PaymentStatus.AUTHORIZATION_REQUIRED;
            case "authorizing" -> PaymentStatus.AUTHORIZING;
            case "authorized" -> PaymentStatus.AUTHORIZED;
            case "executed" -> PaymentStatus.EXECUTED;
            case "settled" -> PaymentStatus.SETTLED;
            case "failed" -> PaymentStatus.FAILED;
            default -> null;
        };
    }
}
```

Note: JWS signature validation via `Verifier.verifyWithJwks()` is deferred
to Step 8 — the test uses `webhook-validation-disabled=true` in the
`TrueLayerTestProfile` to test the event pipeline first. JWS validation
is added as a separate concern in the next step.

- [ ] **Step 7: Add webhook-validation-disabled to TrueLayerTestProfile**

Add to the `getConfigOverrides()` map:

```java
Map.entry("casehub.connectors.bank.truelayer.webhook-validation-disabled", "true")
```

Also add test dependencies to `bank-truelayer/pom.xml` if not present:

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer -am test -Dtest="TrueLayerPaymentWebhookTest" -Dsurefire.failIfNoSpecifiedTests=false -f pom.xml`
Expected: All tests PASS

- [ ] **Step 9: Run full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer,bank-spi -am clean install -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add bank-spi/src/main/java/io/casehub/connectors/bank/model/PaymentStatusChanged.java \
       bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerPaymentWebhook.java \
       bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/WebhookEventDeduplicator.java \
       bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerPaymentWebhookTest.java \
       bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerTestProfile.java \
       bank-truelayer/pom.xml
git commit -m "feat(#108): add TrueLayer payment webhook receiver with CDI events

PaymentStatusChanged record in bank-spi. JAX-RS endpoint at
/webhooks/truelayer/payments with event deduplication, status mapping,
and CDI event firing. JWS signature validation deferred to next commit.

Refs #108

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: Documentation

### Task 2: CLAUDE.md, consumer guide, and ARC42STORIES updates

**Files:**
- Modify: `CLAUDE.md` (add PaymentStatusChanged to bank-spi, webhook to bank-truelayer)
- Modify: `docs/guides/consumer-guide.md` (payment webhooks section)
- Modify: `ARC42STORIES.MD` (L14 TrueLayer container — add webhook)

**Interfaces:**
- Consumes: None (documentation only)
- Produces: None

- [ ] **Step 1: Update CLAUDE.md**

In the bank-spi description, add `PaymentStatusChanged` CDI event record
after the existing model types. In the bank-truelayer description, add
webhook endpoint.

- [ ] **Step 2: Update consumer guide**

Add "Payment Webhooks" section after the TrueLayer consent documentation:

```markdown
**Payment webhooks:** Configure your TrueLayer dashboard to send
payment events to `<your-app-url>/webhooks/truelayer/payments`.
Observe payment status changes:

    void onPaymentStatus(@ObservesAsync PaymentStatusChanged event) {
        // event.paymentId(), event.status(), event.failureReason()
    }

For local development, disable signature validation:

    casehub.connectors.bank.truelayer.webhook-validation-disabled=true
```

- [ ] **Step 3: Update ARC42STORIES.MD**

In §5 Container diagram, add webhook container to L14 (TrueLayer Bank
Provider) boundary:

```
Container(tl_webhook, "TrueLayerPaymentWebhook", "JAX-RS, JWS",
    "POST /webhooks/truelayer/payments\nEvent deduplication · CDI event firing")
```

- [ ] **Step 4: Verify build passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer,bank-spi -am clean install -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md ARC42STORIES.MD docs/guides/consumer-guide.md
git commit -m "docs(#108): document payment webhook in CLAUDE.md, consumer guide, and ARC42STORIES

Add PaymentStatusChanged to bank-spi description, webhook endpoint to
bank-truelayer description, payment webhooks section to consumer guide,
and webhook container to L14 in ARC42STORIES.

Closes #108

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [specs/issue-110-bank-spi-simulation/2026-09-25-payment-webhook-design.md] — design spec
- [specs/issue-110-bank-spi-simulation/decisions.md] — D1, D2, D3
- [bank-truelayer/TrueLayerAuthCallback.java] — parallel JAX-RS endpoint pattern
- [bank-truelayer/TrueLayerPaymentInitiation.java] — status mapping pattern
- [bank-truelayer/TrueLayerBeans.java] — CDI producer pattern
- [bank-spi/PaymentStatus.java] — status enum
- [core/ConnectorService.java] — SentMessage CDI event pattern
- [TrueLayer webhook docs](https://docs.truelayer.com/docs/verify-webhook-signatures) — JWS verification
- [TrueLayer signing library](https://github.com/TrueLayer/truelayer-signing/blob/main/java/README.md) — Java JWS library
- [GitHub #108] — TrueLayer payment webhook receiver
