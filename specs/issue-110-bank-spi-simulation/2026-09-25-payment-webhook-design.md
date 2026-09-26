# TrueLayer Payment Webhook Receiver — Design Spec

> **Issue:** casehubio/connectors#108
> **Date:** 2026-09-25
> **Status:** Draft (revised after light design review)
> **Decisions:** [decisions.md](decisions.md)

## Overview

Add a payment webhook receiver to `bank-truelayer` for real-time payment
status notifications. TrueLayer sends webhook POST requests signed with
JWS (ES512) when payment status changes. The receiver validates the
signature, deduplicates by event ID, maps the event to `PaymentStatus`,
and fires a CDI event.

Polling via `paymentStatus()` continues to work unchanged — the webhook
is additive. Both paths are consistent because TrueLayer is the single
source of truth (D1).

Two modules affected: `bank-spi` (new `PaymentStatusChanged` event
record, D2) and `bank-truelayer` (webhook endpoint, JWS signature
validation, event deduplication).

## PaymentStatusChanged event record (bank-spi)

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

Lives in `bank-spi` alongside the other model records (D2). Any
provider with webhook support fires the same event type. `platformId`
identifies the source provider (`"truelayer"`). Callers observe
`@ObservesAsync PaymentStatusChanged` without coupling to a specific
provider module.

`failureReason` is nullable — non-null only for `FAILED` status, carrying
the TrueLayer failure reason (e.g., insufficient funds, bank rejection).
Without it, observers would need to poll `paymentStatus()` just to get the
reason, defeating real-time notification.

`eventId` is nullable — carries the TrueLayer `event_id` for audit trail
and observer-level deduplication. Non-TrueLayer providers may not have
equivalent identifiers.

## TrueLayerPaymentWebhook (bank-truelayer)

JAX-RS endpoint receiving TrueLayer payment event webhooks. Uses a
standalone endpoint rather than `WebhookInboundConnector` (D3) — payment
status events are not inbound messages and don't fit the `InboundMessage`
/ `WebhookResult` model.

```java
@Path("/webhooks/truelayer")
@ApplicationScoped
public class TrueLayerPaymentWebhook {

    @Inject Event<PaymentStatusChanged> statusEvent;

    @POST
    @Path("/payments")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response handlePayment(@HeaderParam("Tl-Signature") String signature,
                                  String body) {
        // 1. Validate JWS signature (unless validation explicitly disabled)
        // 2. Parse payment event (paymentId, status, eventId, failureReason)
        // 3. Deduplicate by eventId
        // 4. Map webhook event type to PaymentStatus
        // 5. Fire CDI event async
        // 6. Return 200 OK
    }
}
```

### Endpoint path

`POST /webhooks/truelayer/payments` — follows the consent callback
pattern (`/auth/truelayer/callback`) with a parallel path structure.

### Request format

TrueLayer sends a JSON payload:

```json
{
  "type": "payment_executed",
  "event_id": "evt-xxx",
  "event_version": 1,
  "payment_id": "pay-xxx",
  "payment_status": "executed",
  "timestamp": "2026-09-25T10:30:00Z",
  "failure_reason": null
}
```

Event types: `payment_authorization_required`, `payment_authorizing`,
`payment_authorized`, `payment_executed`, `payment_settled`,
`payment_failed`.

### Signature validation (JWS, not HMAC)

TrueLayer signs webhooks with JWS (ES512) using the `Tl-Signature`
header. The signature is a JWS with detached content — verified against
TrueLayer's public key fetched via JKU.

Uses the official `com.truelayer:truelayer-signing` Java library:

```java
String jku = Verifier.extractJku(signature);
if (!isAllowedJku(jku)) {
    return Response.status(401).build();
}
String jwks = fetchJwks(jku);

Verifier.verifyWithJwks(jwks)
    .method("POST")
    .path(path)
    .headers(allHeaders)
    .body(body)
    .verify(signature);
```

**JKU allowlist:** Only TrueLayer production and sandbox JKU URLs are
accepted. Hardcoded allowlist — not configurable.

**JWKS caching:** The JWKS response is cached for 1 hour to avoid
per-request HTTP calls. Cache invalidated when verification fails
(key rotation).

**New dependency:** `com.truelayer:truelayer-signing` added to
`bank-truelayer/pom.xml`.

### Dev/test mode

Signature validation can be disabled for local development via an
explicit boolean flag:

```properties
casehub.connectors.bank.truelayer.webhook-validation-disabled=false
```

Defaults to `false` (validation always on). When `true`, every request
logs at WARNING:

```
WARN Webhook signature validation disabled — accepting unsigned request
```

This is safer than the blank-key bypass pattern — it requires explicit
opt-in and produces visible warnings.

### Event deduplication

TrueLayer retries on non-2xx responses. The same `event_id` can arrive
multiple times. A bounded `ConcurrentHashMap<String, Instant>` tracks
recently-seen event IDs with a 10-minute TTL. Duplicate event IDs are
logged at DEBUG and return 200 OK without firing a CDI event.

A scheduled cleanup runs every 5 minutes to evict expired entries.

### Status mapping

Webhook event types use a `payment_` prefix that the polling API does
not. A dedicated `mapWebhookEventType()` strips the prefix and delegates
to the shared mapping:

```java
private PaymentStatus mapWebhookEventType(String eventType) {
    String status = eventType.replaceFirst("^payment_", "");
    return mapPaymentStatus(status);
}
```

**Unknown event types:** log at WARNING and return `null`. The webhook
handler checks for null and skips CDI event firing — never fires a
`PaymentStatusChanged` with an incorrect status. Returns 200 OK to
TrueLayer (prevent retries).

The existing `mapPaymentStatus()` default case should also be hardened:
replace `default -> PaymentStatus.AUTHORIZATION_REQUIRED` with
`default -> null` (or throw), and guard callers. This is a pre-existing
issue in `TrueLayerPaymentInitiation` but fixing it here prevents the
webhook from inheriting the same silent-default bug.

### Event version handling

The handler validates `event_version`. Currently only version 1 is
supported. Unknown versions log at WARNING and are processed best-effort
(do not reject — TrueLayer's contract is forward-compatible payloads).

### CDI event firing

```java
statusEvent.fireAsync(new PaymentStatusChanged(
        "truelayer", paymentId, status, failureReason, eventId, timestamp));
```

Uses `fireAsync` (consistent with `ConnectorService.send()` pattern).
Observers receive the event without blocking the webhook response.

### Response contract

| Condition | Response |
|-----------|----------|
| Valid signature, event processed | 200 OK |
| Valid signature, duplicate event_id | 200 OK (no CDI event) |
| Invalid or missing signature | 401 Unauthorized |
| Malformed body (missing payment_id) | 400 Bad Request |
| Unknown event type | 200 OK (log WARNING, no CDI event) |

### Configuration

```properties
casehub.connectors.bank.truelayer.webhook-validation-disabled=false
```

No signing key config needed — JWS uses TrueLayer's public key fetched
via JKU from the signature header.

## Testing

### TrueLayerPaymentWebhookTest (WireMock + QuarkusTest)

- Valid webhook → 200, CDI event fired with correct fields
- Invalid JWS signature → 401, no CDI event
- Missing Tl-Signature header → 401
- Malformed body (no payment_id) → 400
- Unknown event type → 200, WARNING logged, no CDI event
- Duplicate event_id → 200, no CDI event (deduplication)
- Each PaymentStatus value mapped correctly
- payment_ prefix stripped correctly
- failureReason populated for FAILED events
- eventId included in CDI event
- Validation disabled → 200 with WARNING log

CDI event verification via `@ObservesAsync` test observer that captures
fired events into a list (same pattern as `SentMessageCapture`).

### Note on ref/simulation support

`PaymentStatusChanged` is provider-emitted — `RefBankPlatform` has no
webhook infrastructure and cannot fire these events. Tests requiring
payment status change observations should inject the CDI event directly
(same approach as `InboundMessage` testing).

## Deliverables beyond code

### Consumer guide update

Add "Payment Webhooks" section:

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

### CLAUDE.md update

Add `PaymentStatusChanged` to the bank-spi description. Add webhook
endpoint to bank-truelayer description.

## References

- [TrueLayer webhook verification docs](https://docs.truelayer.com/docs/verify-webhook-signatures)
- [TrueLayer signing library (Java)](https://github.com/TrueLayer/truelayer-signing/blob/main/java/README.md)
- [TrueLayer payment webhooks](https://docs.truelayer.com/docs/payment-webhooks)
- [TrueLayer payments webhook reference](https://docs.truelayer.com/docs/payments-api-webhook-reference)
- `bank-truelayer/TrueLayerAuthCallback.java` — parallel JAX-RS endpoint pattern
- `bank-truelayer/TrueLayerPaymentInitiation.java:59-62` — stateless polling
- `bank-truelayer/TrueLayerPaymentInitiation.java:74-83` — status mapping
- `bank-spi/PaymentStatus.java` — status enum
- `core/ConnectorService.java` — `Event<SentMessage>.fireAsync()` CDI event pattern
- D1 (decisions.md) — webhook fires CDI event, polling stays stateless
- D2 (decisions.md) — PaymentStatusChanged lives in bank-spi
- D3 (decisions.md) — standalone JAX-RS endpoint, not WebhookInboundConnector
- Design review R1-02 — JWS not HMAC (showstopper caught)
- Design review R1-03/R1-04 — replay prevention and idempotency
- Design review R1-05/R1-06 — status format mismatch and silent default
- Design review R1-07/R1-08 — failureReason and eventId on record
- Design review R1-13 — explicit validation disable flag
