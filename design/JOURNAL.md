# Design Journal — issue-4-inbound-connector-spi

### 2026-05-29 · §Module Structure

Added `casehub-connectors-webhook` as a third Maven module. It depends on `core` and `quarkus-rest` — the REST dependency is the reason for the split, mirroring the same rationale that made `email` a separate module (quarkus-mailer). Deployments that only need outbound delivery pull in `core` only; the webhook transport is opt-in by classpath presence.

### 2026-05-29 · §SPI

Inbound transport uses two distinct types rather than a unified SPI:

- `InboundConnector` — for pull-based connectors (e.g. IMAP). Has `start(InboundMessageSink)/stop()` lifecycle managed by `InboundConnectorService` at Quarkus startup/shutdown.
- `WebhookInboundConnector` — for push-based (webhook) connectors. No lifecycle methods; the JAX-RS endpoint IS the lifecycle. Does not implement `InboundConnector` — the two are CDI-discovered from separate `@All List<>` injection points.

The unified-SPI approach was considered and rejected: it would have required `final` no-op `start()/stop()` methods on every webhook connector, making the interface contract misleading and `InboundConnectorService.onStart()` call lifecycle methods that have no effect.

### 2026-05-29 · §Data Model

Three new types in `core`:

- `InboundMessage` record — transport metadata only (`connectorId`, `externalSenderId`, `externalChannelRef`, `content`, `receivedAt`, `metadata`). No domain semantics — the connector does not interpret the message. Text-only in v1; media messages yield `content` = media URL or empty string.
- `WebhookRequest` record — normalised HTTP request for connector `handle()`. Header keys lower-cased at the JAX-RS boundary so connectors use simple `Map.get("x-slack-signature")`. Includes `requestUrl` (required for Twilio HMAC-SHA1 which signs the full URL).
- `WebhookResult` sealed interface — `Delivered`, `Challenged`, `Ignored`, `Unauthorized`. Exhaustive pattern match in the router prevents missing a case. `Unauthorized` maps to HTTP 200 for POST (suppress retry storms) and HTTP 403 for GET (admin console setup needs a visible failure signal).

### 2026-05-29 · §Design Principles

Inbound transport is decoupled from processing via synchronous CDI `Event<InboundMessage>`. Connectors deliver a message and return immediately — they have no knowledge of what happens next. Observers (the Qhorus bridge in connectors#6, WorkItem routing in work#234) own their dispatch strategy and must not block the CDI fire for longer than Slack's 3-second retry deadline. The `InboundConnectorService.receive()` method is the single CDI event bus for all inbound transports, regardless of whether the message arrived via pull polling or webhook.
