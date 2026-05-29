# Design: ConnectorChannelBackend — Inbound Message Bridge via HumanParticipatingChannelBackend

**Date:** 2026-05-29  
**Issue:** casehubio/connectors#6  
**Repos affected:** casehub-qhorus (primary), casehub-connectors (no changes)

---

## Problem

`InboundConnectorService` fires a synchronous `Event<InboundMessage>` when any connector receives a
message. Nothing currently consumes that event — received messages go nowhere. connectors#6 wires
inbound connector traffic into Qhorus channels so agents can process it.

Two design gaps were identified before this spec could be written:

1. **No API-level entry point** — `ChannelGateway.receiveHumanMessage()` is the correct Qhorus
   entry point for human messages, but it lives in qhorus-runtime. A bridge depending only on
   qhorus-api has no way to call it.

2. **No channel-to-connector mapping** — the `Channel` entity has no concept of which external
   connector it is bound to, or what external key identifies the conversation.

The `HumanParticipatingChannelBackend` SPI is the correct architectural home for this bridge. It
handles both inbound delivery (`receiveHumanMessage`) and outbound fan-out (`post`), and is the
model already used by ClaudonyChannelBackend and OpenClawChannelBackend.

---

## Key Decisions

### D1: Channel granularity — per-conversation (not per-endpoint)

For SMS, WhatsApp, and Email the `InboundMessage.externalChannelRef` is our own endpoint (our
Twilio number, our inbox). It does NOT identify the conversation partner. If channels were
per-endpoint, all messages from all senders would land in one channel, and the `post()` callback
would have no way to know which human to reply to without a database query.

**Decision:** one Qhorus channel per conversation partner. For Slack, the conversation is the Slack
channel (multi-participant); for SMS/WhatsApp/Email the conversation is with one sender.

### D2: Pre-provisioned channels only in v1

Channels with connector bindings must be created by an operator before conversations begin. If an
`InboundMessage` arrives and no channel is found for the binding key, a WARN is logged and the
message is silently discarded. Auto-creation on first contact is deferred to v2 (requires
governance decisions on semantic, ACL, and rate limit defaults).

### D3: Bridge lives in qhorus repo, new module

The bridge must call `ChannelGateway.registerBackend()`, `ChannelGateway.receiveHumanMessage()`,
and `ChannelService.findByConnectorKey()` — all qhorus-runtime classes. Placing it in the
connectors repo would introduce a library-to-runtime cross-repo dependency. Instead: a new module
`casehub-qhorus-connector-backend` in the qhorus repo, listed after `runtime` in the root pom
(dependency order constraint).

The `cross-foundation-bridge-module-placement` protocol is updated to note that bidirectional SPI
bridges requiring runtime access are placed in the repo that owns the runtime, not necessarily the
event-source repo.

### D4: No changes to existing inbound connectors

`externalChannelRef` and `externalSenderId` in `InboundMessage` remain as-is. The bridge derives
the lookup key per connector type internally (see Inbound Key Derivation below).

---

## Data Model Changes (casehub-qhorus)

### Channel entity — four new nullable columns

```java
// Channel.java — four new nullable fields
@Column(name = "inbound_connector_id")
public String inboundConnectorId;   // "slack-inbound", "twilio-sms-inbound", etc.

@Column(name = "external_key")
public String externalKey;          // conversation identifier; see Key Derivation table

@Column(name = "outbound_connector_id")
public String outboundConnectorId;  // "slack", "twilio-sms", etc.

@Column(name = "outbound_destination")
public String outboundDestination;  // Slack webhook URL, E.164 phone number, email address
```

All four must be set together or all null. A channel with `inboundConnectorId` set but
`outboundConnectorId` null is invalid (enforced in ChannelService at create/update time, not at
the DB level). A channel with no binding behaves identically to today — the backend does not
register for it.

### Flyway migration

```sql
ALTER TABLE channel ADD COLUMN inbound_connector_id VARCHAR(64);
ALTER TABLE channel ADD COLUMN external_key          VARCHAR(255);
ALTER TABLE channel ADD COLUMN outbound_connector_id VARCHAR(64);
ALTER TABLE channel ADD COLUMN outbound_destination  VARCHAR(512);
```

### ChannelService — new lookup method

```java
public Optional<Channel> findByConnectorKey(String inboundConnectorId, String externalKey) {
    return channelStore.findByConnectorKey(inboundConnectorId, externalKey);
}
```

Both `ChannelStore` (interface) and `JpaChannelStore` / `InMemoryChannelStore` (implementations)
gain `findByConnectorKey`. The JPA query is a single indexed lookup; a composite index on
`(inbound_connector_id, external_key)` is added in the same migration for performance.

### ChannelDetail API type

`ChannelDetail` gains the four new fields as nullable strings, for MCP tool exposure.

---

## Inbound Key Derivation

The bridge determines the lookup key from the `InboundMessage` fields. No connector changes needed.

| `InboundMessage.connectorId` | Lookup key source | Rationale |
|------------------------------|-------------------|-----------|
| `"slack-inbound"` | `externalChannelRef` | Slack channel ID (C123456) IS the conversation space |
| `"twilio-sms-inbound"` | `externalSenderId` | `externalChannelRef` = our Twilio number; the sender IS the conversation |
| `"whatsapp-inbound"` | `externalSenderId` | `externalChannelRef` = our phone number ID; sender IS the conversation |
| `"email-inbound"` | `externalSenderId` | `externalChannelRef` = our inbox; sender IS the conversation |
| any other | `externalChannelRef` | Safe default for future connectors |

This mapping is a small static table in the bridge (`ConnectorKeyStrategy`), not reflection or
configuration. Unknown connector IDs use `externalChannelRef` as a safe fallback.

---

## ConnectorChannelBackend

Single `@ApplicationScoped` bean. Implements `HumanParticipatingChannelBackend`. Self-registers
per channel using the `ChannelInitialisedEvent` pattern established by `ClaudonyChannelBackend`.

### `backendId()`

Returns `"connector-human"`. Constant. Used by `deregisterBackend()` for idempotent re-registration.

### `actorType()`

Returns `ActorType.HUMAN`. All connector-sourced messages are human speech acts.

### Registration — `@Observes ChannelInitialisedEvent`

```
onChannelInitialised(event):
  channel = channelService.findById(event.channelId())   // null if deleted mid-init
  if channel == null or channel.inboundConnectorId == null: return
  cache.put(event.channelId(), new Binding(
      channel.inboundConnectorId, channel.externalKey,
      channel.outboundConnectorId, channel.outboundDestination))
  gateway.deregisterBackend(event.channelId(), BACKEND_ID)   // idempotent
  gateway.registerBackend(event.channelId(), this, "human_participating")
```

Fires at startup (all persisted channels) and on channel creation. The deregister-then-register
pattern matches `ClaudonyChannelBackend` and is safe for concurrent restarts.

The in-memory `cache` is a `ConcurrentHashMap<UUID, Binding>`. It is the sole source of truth for
`post()` — no database access during fan-out.

### Inbound routing — `@ObservesAsync InboundMessage`

```
onInboundMessage(msg):
  key = ConnectorKeyStrategy.deriveKey(msg)           // per-connector-type lookup
  channel = channelService.findByConnectorKey(msg.connectorId(), key)
  if channel == null:
    log.warn("No channel bound to {} / {} — message discarded", msg.connectorId(), key)
    return
  gateway.receiveHumanMessage(
      new ChannelRef(channel.id, channel.name),
      new InboundHumanMessage(
          msg.externalSenderId(),
          msg.content(),
          msg.receivedAt(),
          msg.metadata(),
          null,   // correlationId — none on raw inbound
          null))  // inReplyTo — none on raw inbound
```

`@ObservesAsync` ensures this runs off the webhook thread. The `InboundConnectorService` Javadoc
already requires async observation for Qhorus bridge usage (Slack's 3-second retry window).

The `NormalisedMessage` produced by `receiveHumanMessage` defaults to `DefaultInboundNormaliser`
(QUERY type, sender = `"human:" + externalSenderId`). The backend does not override `normaliser()`
in v1. Per-connector normaliser customisation is deferred.

### Outbound delivery — `post(ChannelRef, OutboundMessage)`

```
post(channelRef, message):
  binding = cache.get(channelRef.id())
  if binding == null:
    log.error("No binding cache entry for channel {} — outbound dropped", channelRef.id())
    return                                              // must not throw
  connectorService.send(
      binding.outboundConnectorId(),
      new ConnectorMessage(binding.outboundDestination(), null, message.content()))
```

No database access. The cache is always populated before `registerBackend()` is called, so a
channel with a registered backend always has a cache entry.

### `open()` / `close()`

`open()` is a no-op. Registration lifecycle is driven by `ChannelInitialisedEvent`, not `open()`.

`close()` removes the channel's binding from the in-memory cache. This is called by
`ChannelGateway.closeChannel()` when a channel is deleted, ensuring the cache stays consistent with
the registry.

```
close(channelRef):
  cache.remove(channelRef.id())
```

---

## Module Structure (casehub-qhorus)

### New module: `casehub-qhorus-connector-backend`

Location: `connector-backend/` sibling of `connectors/`, `runtime/`, etc.

```xml
<!-- connector-backend/pom.xml -->
<artifactId>casehub-qhorus-connector-backend</artifactId>
<description>Optional bridge — ConnectorChannelBackend routes InboundMessage to Qhorus channels
via HumanParticipatingChannelBackend. Activates by classpath presence.</description>

<dependencies>
  <dependency>casehub-qhorus-api</dependency>
  <dependency>casehub-qhorus-runtime</dependency>    <!-- ChannelGateway, ChannelService -->
  <dependency>casehub-connectors-core</dependency>   <!-- InboundMessage, ConnectorService -->
  <dependency>jakarta.enterprise.cdi-api (provided)</dependency>
  <dependency>jboss-logging (provided)</dependency>
  <dependency>quarkus-junit5 (test)</dependency>
  <dependency>casehub-qhorus-testing (test)</dependency>
</dependencies>
```

Root `pom.xml` module order:

```xml
<modules>
  <module>api</module>
  <module>connectors</module>    <!-- api-level bridge: WatchdogAlertEvent → ConnectorService -->
  <module>runtime</module>
  <module>connector-backend</module>   <!-- runtime-aware bridge: InboundMessage ↔ ChannelGateway -->
  <module>deployment</module>
  <module>testing</module>
  <module>examples</module>
</modules>
```

`connector-backend` must be after `runtime` because it depends on it. Per the
`cross-repo-optional-dep-table-registration` protocol, the dependency on `casehub-connectors-core`
is registered in both the build order and the Cross-Repo Dependency Map in `parent/docs/PLATFORM.md`.

---

## Example: Binding a Twilio SMS channel

```
// Operator creates channel with binding
ChannelService.create(
    name:               "sms-alice",
    description:        "Alice's SMS conversation",
    semantic:           STANDARD,
    inboundConnectorId: "twilio-sms-inbound",
    externalKey:        "+15551110000",    // Alice's phone number (externalSenderId)
    outboundConnectorId: "twilio-sms",
    outboundDestination: "+15551110000")   // same phone number for replies
```

When Alice texts our Twilio number, `TwilioSmsInboundConnector` fires:
```
InboundMessage(
    connectorId:        "twilio-sms-inbound",
    externalSenderId:   "+15551110000",    // Alice's From number
    externalChannelRef: "+14155552671",    // our Twilio number (To)
    content:            "I need help",
    ...)
```

The bridge:
1. Derives key = `externalSenderId` = `"+15551110000"` (Twilio-type rule)
2. Looks up `findByConnectorKey("twilio-sms-inbound", "+15551110000")` → channel "sms-alice"
3. Calls `gateway.receiveHumanMessage(...)` → dispatched as QUERY, sender `"human:+15551110000"`

When qhorus replies, `fanOut()` calls `post()`:
- Cache hit: `outboundConnectorId = "twilio-sms"`, `outboundDestination = "+15551110000"`
- Calls `connectorService.send("twilio-sms", new ConnectorMessage("+15551110000", null, "We can help"))`

## Example: Binding a Slack channel

```
ChannelService.create(
    name:               "slack-support",
    inboundConnectorId: "slack-inbound",
    externalKey:        "C123456",         // Slack channel ID (externalChannelRef)
    outboundConnectorId: "slack",
    outboundDestination: "https://hooks.slack.com/services/...")
```

Inbound: `externalChannelRef` = "C123456" → key = `externalChannelRef` (Slack rule) → channel found.
Outbound: POST to the incoming webhook URL.

---

## Testing

### Unit tests (no Quarkus, `ChannelGatewayTest` pattern)

- `ConnectorChannelBackendTest` — mocks ChannelGateway, ChannelService, ConnectorService
  - `onChannelInitialised_registersBackend_whenBindingPresent`
  - `onChannelInitialised_skips_whenNoBinding`
  - `onChannelInitialised_isIdempotent` (second event for same channel: deregister+register, cache updated)
  - `onInboundMessage_routesToReceiveHumanMessage_whenChannelFound`
  - `onInboundMessage_logsWarn_whenNoChannelFound` (no exception thrown)
  - `post_sendsViaConnectorService_fromCache`
  - `post_logsError_whenNoCacheEntry` (no exception thrown)
  - `close_removesCacheEntry`
- `ConnectorKeyStrategyTest`
  - One test per connector type asserting correct field selection
  - Fallback test for unknown connector ID

### Integration test (`@QuarkusTest`, in-memory stores)

- `ConnectorChannelBackendIntegrationTest`
  - Full round-trip: create channel with binding → simulate `ChannelInitialisedEvent` → fire `InboundMessage` → assert `receiveHumanMessage` was called → simulate `fanOut` → assert `ConnectorService.send()` called with correct destination
  - Multi-channel: two channels with different bindings → correct routing for each
  - Unknown sender: `InboundMessage` arrives with no matching channel → no exception, warn logged

---

## Scope

### In v1

- Flyway migration for 4 new Channel columns + composite index
- `ChannelStore.findByConnectorKey()` interface + JPA + InMemory implementations
- `ChannelService.findByConnectorKey()`
- `ChannelDetail` updated with 4 new fields
- `casehub-qhorus-connector-backend` module
- `ConnectorChannelBackend` (registration, inbound, post, close)
- `ConnectorKeyStrategy` (static per-connector-type table)
- `create()` / `update()` overloads in `ChannelService` accepting connector binding fields
- All unit and integration tests above
- Protocol update: `cross-foundation-bridge-module-placement` — add note about runtime-requiring bridges
- Cross-Repo Dependency Map update: `casehub-qhorus-connector-backend → casehub-connectors-core`

### Deferred

- Auto-channel creation on first contact (v2, requires governance config for semantic/ACL/rate limits)
- Per-connector `InboundNormaliser` override (e.g., email subject → correlationId)
- IMAP IDLE integration (#9 — `open()` would start an IDLE connection per channel; `close()` would
  tear it down in addition to cache removal)
- MCP tools to create/update channels with connector bindings (currently via ChannelService only)
- Teams inbound connector binding support (connector not yet implemented)
