# TrueLayer Payment Webhook Receiver — Design Spec

> **Issue:** casehubio/connectors#108
> **Date:** 2026-09-25
> **Status:** Draft
> **Decisions:** [decisions.md](decisions.md)

## Overview

Add a payment webhook receiver to `bank-truelayer` for real-time payment
status notifications. TrueLayer sends webhook POST requests when payment
status changes (authorized → executed → settled, or → failed). The
receiver validates the HMAC signature, maps the event to `PaymentStatus`,
and fires a CDI event.

Polling via `paymentStatus()` continues to work unchanged — the webhook
is additive. Both paths are consistent because TrueLayer is the single
source of truth (D1).

Two modules affected: `bank-spi` (new `PaymentStatusChanged` event
record, D2) and `bank-truelayer` (webhook endpoint, signature
validation).

## PaymentStatusChanged event record (bank-spi)

```java
package io.casehub.connectors.bank.model;

import java.time.Instant;

public record PaymentStatusChanged(String platformId,
                                    String paymentId,
                                    PaymentStatus status,
                                    Instant timestamp) {}
```

Lives in `bank-spi` alongside the other model records (D2). Any
provider with webhook support fires the same event type. `platformId`
identifies the source provider (`"truelayer"`). Callers observe
`@ObservesAsync PaymentStatusChanged` without coupling to a specific
provider module.

## TrueLayerPaymentWebhook (bank-truelayer)

JAX-RS endpoint receiving TrueLayer payment event webhooks.

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
        // 1. Validate HMAC signature
        // 2. Parse payment event
        // 3. Map to PaymentStatus
        // 4. Fire CDI event
        // 5. Return 200 OK
    }
}
```

### Endpoint path

`POST /webhooks/truelayer/payments` — follows the consent callback
pattern (`/auth/truelayer/callback`) with a parallel path structure.
Webhook endpoints use `/webhooks/` prefix to distinguish from auth
callbacks.

### Request format

TrueLayer sends a JSON payload:

```json
{
  "type": "payment_executed",
  "event_id": "evt-xxx",
  "event_version": 1,
  "payment_id": "pay-xxx",
  "payment_status": "executed",
  "timestamp": "2026-09-25T10:30:00Z"
}
```

Event types: `payment_authorization_required`, `payment_authorizing`,
`payment_authorized`, `payment_executed`, `payment_settled`,
`payment_failed`.

### Signature validation

TrueLayer signs webhooks with HMAC-SHA512 using a signing key from
the TrueLayer console. The signature is sent in the `Tl-Signature`
header.

```java
class TrueLayerWebhookValidator {

    private final String signingKey;

    boolean isValid(String signature, String body) {
        Mac mac = Mac.getInstance("HmacSHA512");
        mac.init(new SecretKeySpec(
                signingKey.getBytes(StandardCharsets.UTF_8), "HmacSHA512"));
        byte[] computed = mac.doFinal(body.getBytes(StandardCharsets.UTF_8));
        String expected = HexFormat.of().formatHex(computed);
        return MessageDigest.isEqual(
                expected.getBytes(StandardCharsets.UTF_8),
                signature.getBytes(StandardCharsets.UTF_8));
    }
}
```

Constant-time comparison via `MessageDigest.isEqual()` to prevent
timing attacks. The signing key is a config property:
`casehub.connectors.bank.truelayer.webhook-signing-key`.

### Status mapping

Reuses the existing `mapPaymentStatus()` pattern from
`TrueLayerPaymentInitiation`:

```
payment_authorization_required → AUTHORIZATION_REQUIRED
payment_authorizing            → AUTHORIZING
payment_authorized             → AUTHORIZED
payment_executed               → EXECUTED
payment_settled                → SETTLED
payment_failed                 → FAILED
```

### CDI event firing

```java
statusEvent.fireAsync(new PaymentStatusChanged(
        "truelayer", paymentId, status, timestamp));
```

Uses `fireAsync` (consistent with `ConnectorService.send()` pattern).
Observers receive the event without blocking the webhook response.

### Response contract

| Condition | Response |
|-----------|----------|
| Valid signature, event processed | 200 OK |
| Invalid or missing signature | 401 Unauthorized |
| Malformed body (missing paymentId) | 400 Bad Request |
| Unknown event type | 200 OK (log + ignore) |

Unknown event types return 200 to prevent TrueLayer from retrying.
TrueLayer retries on non-2xx responses.

### Configuration

```properties
casehub.connectors.bank.truelayer.webhook-signing-key=${TRUELAYER_WEBHOOK_KEY:}
```

Empty default — webhook validation is skipped when the key is blank
(dev/test mode). When set, all webhooks must pass HMAC validation.

## Testing

### TrueLayerWebhookValidatorTest (unit)

- Valid signature accepted
- Invalid signature rejected
- Constant-time comparison (no timing leak — structural test)
- Empty signing key skips validation

### TrueLayerPaymentWebhookTest (WireMock + QuarkusTest)

- Valid webhook → 200, CDI event fired with correct paymentId and status
- Invalid signature → 401, no CDI event
- Missing signature header → 401
- Malformed body (no payment_id) → 400
- Unknown event type → 200, no CDI event
- Each PaymentStatus value mapped correctly

CDI event verification via `@ObservesAsync` test observer that
captures fired events into a list (same pattern as
`SentMessageCapture`).

## Deliverables beyond code

### Consumer guide update

Add a "Payment Webhooks" section after the existing TrueLayer consent
documentation:

```markdown
**Payment webhooks:** Configure your TrueLayer dashboard to send
payment events to `<your-app-url>/webhooks/truelayer/payments`.
Set the signing key:

    casehub.connectors.bank.truelayer.webhook-signing-key=${TRUELAYER_WEBHOOK_KEY}

Observe payment status changes:

    void onPaymentStatus(@ObservesAsync PaymentStatusChanged event) {
        // event.paymentId(), event.status(), event.platformId()
    }
```

### CLAUDE.md update

Add `PaymentStatusChanged` to the bank-spi description.

## References

- `bank-truelayer/TrueLayerAuthCallback.java` — parallel JAX-RS endpoint pattern (consent callback)
- `bank-truelayer/TrueLayerPaymentInitiation.java:59-62` — existing stateless polling
- `bank-truelayer/TrueLayerPaymentInitiation.java:74-83` — status mapping pattern
- `bank-spi/PaymentStatus.java` — status enum
- `core/ConnectorService.java` — `Event<SentMessage>.fireAsync()` CDI event pattern
- `bank-truelayer/TrueLayerConsentService.java` — HMAC/crypto patterns in the module
- #106 design spec — deferred webhooks to this issue (§PaymentInitiation, line 126-129)
- D1 (decisions.md) — webhook fires CDI event, polling stays stateless
- D2 (decisions.md) — PaymentStatusChanged lives in bank-spi
