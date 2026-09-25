## D1: Webhook and polling consistency model

**Choice:** Webhook fires CDI event for real-time notification; polling stays stateless against TrueLayer API. Both paths are consistent because TrueLayer is the single source of truth.
**Alternatives:**
- Local payment status cache updated by webhook, consulted by paymentStatus() — adds complexity with no benefit since the TrueLayer API is always authoritative.
- CDI event only, no consideration of polling — functionally identical but doesn't make the consistency model explicit.
**Rationale:** TrueLayerPaymentInitiation.paymentStatus() calls TrueLayer directly each time — no local state. Introducing a cache just for webhook consistency adds complexity without value. The webhook's purpose is real-time notification, not replacing the query channel.
**Trade-offs:** Polling callers don't see webhook-delivered status faster — they still poll at their own cadence. Callers wanting real-time must observe the CDI event.
**Sources:** bank-truelayer/TrueLayerPaymentInitiation.java:59-62 (stateless polling), core/ConnectorService.java (SentMessage CDI event pattern)
**Exploration:** quick
**Status:** captured

## D2: PaymentStatusChanged event location

**Choice:** bank-spi — the event record lives on the SPI, not in bank-truelayer
**Alternatives:**
- bank-truelayer — YAGNI, keep it provider-specific until a second provider needs it. Couples callers to TrueLayer module.
**Rationale:** Any future provider with webhook support fires the same event type. Callers observe PaymentStatusChanged without knowing which provider sent it. Follows the ConnectorService.send() → Event<SentMessage> pattern where the event is on the SPI, not on individual connectors.
**Trade-offs:** Slightly enlarges the SPI surface for a feature only one provider currently uses.
**Sources:** core/ConnectorService.java (SentMessage pattern), bank-spi/BankPlatform.java (SPI surface)
**Exploration:** quick
**Status:** captured

## D3: Standalone JAX-RS endpoint over WebhookInboundConnector

**Choice:** Dedicated JAX-RS endpoint at `/webhooks/truelayer/payments`
**Alternatives:**
- WebhookInboundConnector extending the platform's inbound connector infrastructure — wrong semantic model (payment status != inbound message), pulls in InboundMessage/WebhookResult types, routes through WebhookRouter at /connectors/{id}/webhook.
**Rationale:** Payment webhook events are provider-internal status updates, not inbound messages. The CDI event type is PaymentStatusChanged, not InboundMessage. D4 from #106 established precedent for standalone JAX-RS endpoints in bank-truelayer (consent callback). The semantic difference justifies a separate path.
**Trade-offs:** Two webhook patterns in the platform — /connectors/{id}/webhook for messaging, /webhooks/truelayer/payments for banking. Future banking providers with webhooks would follow this pattern.
**Sources:** webhook/WebhookRouter.java (messaging webhook infrastructure), bank-truelayer/TrueLayerAuthCallback.java (D4 precedent), ARC42STORIES.MD L2 (WebhookInboundConnector architecture)
**Exploration:** quick
**Status:** captured
