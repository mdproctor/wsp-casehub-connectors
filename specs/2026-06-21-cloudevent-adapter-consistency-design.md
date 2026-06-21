# CloudEvent Adapter Consistency — connectors, iot, qhorus

**Date:** 2026-06-21
**Status:** Approved
**Issues:** casehubio/connectors#20 (primary), new iot issue (TBD), new qhorus issue (TBD)
**Architectural context:** `docs/superpowers/specs/2026-06-13-p0-layering-decisions-design.md` in casehubio/parent

---

## Scope

connectors#20 (implement `InboundMessage → CloudEvent` adapter) revealed that the two existing
adapters — `IoTCloudEventAdapter` (iot#19, closed) and `QhorusCloudEventAdapter` (qhorus#279,
closed) — contain three latent bugs and one design inconsistency. All three repos are fixed
together so the platform has one coherent CloudEvent adapter pattern.

---

## Canonical CloudEvent Adapter Pattern

All adapters in the platform must satisfy these four rules:

| Rule | Requirement | Rationale |
|---|---|---|
| ObjectMapper | Injected — never a private static instance | Static instance isolates serialisation config from the app's shared mapper |
| tenancyId | Null-safe — extension omitted when null, never set to string `"null"` | SDK serialises null extension as literal `"null"`; breaks consumers in single-tenant deployments |
| fireAsync | Always `.exceptionally(ex -> { LOG.severe(...); return null; })` | Unhandled CompletionStage swallows downstream failures silently |
| type derivation | From a stable semantic field on the event record — not from an instance identifier | Instance IDs are mutable; CloudEvent `type` is used for routing rules that must be stable |

tenancyId is **always sourced from the event record** — never from `CurrentPrincipal`, which is
request-scoped and inactive in `@ObservesAsync` observers.

---

## Repo 1 — casehub-connectors (branch: issue-20-cloudEvent-adapter)

### 1a. InboundMessage — two new fields

```java
public record InboundMessage(
    String connectorId,       // existing — instance identifier, used in source URI
    String connectorType,     // NEW — stable semantic label, used in CloudEvent type
    String externalSenderId,
    String externalChannelRef,
    String content,
    List<Attachment> attachments,
    Instant receivedAt,
    Map<String, String> metadata,
    String tenancyId)         // NEW — nullable; omitted from CloudEvent extension if null
```

Existing convenience constructors are removed — callers must use the canonical constructor.
The compiler enforces that every call site supplies both new fields.

### 1b. InboundConnectorTypes — new constants class

Parallel to `InboundConnectorIds`. Semantic transport categories, not provider specifics:

```java
public final class InboundConnectorTypes {
    public static final String SLACK    = "slack";
    public static final String EMAIL    = "email";
    public static final String SMS      = "sms";      // "sms" not "twilio-sms" — type is transport, not provider
    public static final String WHATSAPP = "whatsapp";
    public static final String TEAMS    = "teams";
    private InboundConnectorTypes() {}
}
```

### 1c. tenancyId at production sites

**Webhook connectors** (Slack, Teams, Twilio SMS, WhatsApp): read from
`request.header("x-tenancy-id")`. `WebhookRequest` already carries lower-cased headers; no new
dependency required.

**Email inbound** (`EmailInboundConnector`): `EmailInboundAccount` gains a `tenancyId` String
field (nullable). `toInboundMessage()` reads it directly.

### 1d. Production construction site migrations

| File | connectorType | tenancyId source |
|---|---|---|
| `SlackInboundConnector.java:168` | `InboundConnectorTypes.SLACK` | `request.header("x-tenancy-id")` |
| `TeamsInboundConnector.java:108` | `InboundConnectorTypes.TEAMS` | `request.header("x-tenancy-id")` |
| `TwilioSmsInboundConnector.java:96` | `InboundConnectorTypes.SMS` | `request.header("x-tenancy-id")` |
| `WhatsAppInboundConnector.java:156` | `InboundConnectorTypes.WHATSAPP` | `request.header("x-tenancy-id")` |
| `EmailInboundConnector.java:208,218` | `InboundConnectorTypes.EMAIL` | `account.tenancyId()` |

### 1e. New submodule — casehub-connectors-cloud-events

**Why a submodule, not in core:** `casehub-connectors-core` carries zero external deps beyond CDI
and `java.net.http`. Adding `casehub-platform-api` (which carries `cloudevents-core`) would
permanently break that invariant and force the CloudEvents SDK onto every consumer of core.
The new submodule activates by classpath presence — exactly the optional-module-pattern used by
IoT and Qhorus.

**pom.xml dependencies:**
- `casehub-connectors-core`
- `casehub-platform-api` (carries `cloudevents-core` transitively)
- `quarkus-arc`

**`ConnectorCloudEventAdapter`:**

```java
@ApplicationScoped
public class ConnectorCloudEventAdapter {

    private final Event<CloudEvent> cloudEventBus;
    private final ObjectMapper objectMapper;

    @Inject
    public ConnectorCloudEventAdapter(Event<CloudEvent> cloudEventBus, ObjectMapper objectMapper) {
        this.cloudEventBus = cloudEventBus;
        this.objectMapper = objectMapper;
    }

    public void onMessage(@ObservesAsync InboundMessage message) {
        cloudEventBus.fireAsync(toCloudEvent(message))
            .exceptionally(ex -> {
                LOG.severe("CloudEvent dispatch failed for connector=" + message.connectorId()
                    + ": " + ex.getMessage());
                return null;
            });
    }
}
```

**CloudEvent field mapping:**

| CloudEvent field | Value |
|---|---|
| `type` | `"io.casehub.connectors.inbound." + message.connectorType()` |
| `source` | `URI.create("/casehub-connectors/" + message.connectorId())` |
| `subject` | `"channel/" + message.externalChannelRef()` |
| `id` | `UUID.randomUUID().toString()` |
| `time` | `message.receivedAt().atOffset(ZoneOffset.UTC)` |
| `datacontenttype` | `"application/json"` |
| `data` | JSON-serialised `InboundMessage` via injected `ObjectMapper` |
| `tenancyid` extension | Conditionally set — omitted if `message.tenancyId()` is null |

**Serialisation error handling:** log at SEVERE, fire CloudEvent with empty data (`new byte[0]`) —
matches Qhorus's pattern. Inbound connector processing is unaffected.

### 1f. Parent pom.xml

Add `<module>cloud-events</module>` to `casehub-connectors-parent`.

---

## Repo 2 — casehub-iot (main, new issue)

Three fixes to `IoTCloudEventAdapter`:

**Fix 1 — Inject ObjectMapper**
```java
// Before
private static final ObjectMapper MAPPER = new ObjectMapper()
    .registerModule(new JavaTimeModule())
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

// After
private final ObjectMapper objectMapper;  // injected
```

**Fix 2 — Null-safe tenancyId**
```java
// Before
.withExtension("tenancyid", event.after().tenancyId())

// After
CloudEventBuilder builder = CloudEventBuilder.v1()...;
if (event.after().tenancyId() != null) {
    builder = builder.withExtension("tenancyid", event.after().tenancyId());
}
```

**Fix 3 — Handle fireAsync**
```java
// Before
cloudEvents.fireAsync(ce);

// After
cloudEvents.fireAsync(ce)
    .exceptionally(ex -> {
        LOG.severe("CloudEvent dispatch failed for device=" + event.after().deviceId()
            + ": " + ex.getMessage());
        return null;
    });
```

---

## Repo 3 — casehub-qhorus (main, new issue)

One fix to `QhorusCloudEventAdapter` — ObjectMapper already injected, tenancyId already null-safe:

**Fix — Handle fireAsync**
```java
// Before
cloudEventBus.fireAsync(toCloudEvent(event));

// After
cloudEventBus.fireAsync(toCloudEvent(event))
    .exceptionally(ex -> {
        LOG.warnf(ex, "CloudEvent dispatch failed for channel=%s type=%s",
            event.channelId(), event.messageType());
        return null;
    });
```

---

## Garden Protocol

New entry documenting the canonical CloudEvent adapter pattern — the four rules above, with
rationale for each. Captures: why static ObjectMapper is wrong, why null tenancyId corrupts
extensions, why fireAsync needs `.exceptionally()`, why type must use a semantic field.

---

## Issues to File Before Implementation

| Repo | Title | Scope |
|---|---|---|
| casehubio/iot | fix: IoTCloudEventAdapter — inject ObjectMapper, null-safe tenancyId, handle fireAsync | 3 changes to one class |
| casehubio/qhorus | fix: QhorusCloudEventAdapter — handle fireAsync CompletionStage | 1 change to one class |

---

## What This Does Not Change

- `InboundConnectorService` event dispatch — unchanged
- `WebhookRouter` — unchanged (tenancyId flows through `WebhookRequest` headers, read by each connector)
- All existing `@ObservesAsync InboundMessage` observers — unaffected; they do not receive CloudEvents
- IoT's CloudEvent `type` format (`io.casehub.iot.state_change.<deviceClass>`) — unchanged; altering it would break existing RAS observers
- Qhorus's CloudEvent field mapping — unchanged beyond fireAsync

---

## Acceptance Criteria

### connectors#20
- [ ] `InboundMessage` has `connectorType` and `tenancyId` fields
- [ ] `InboundConnectorTypes` constants class exists with SLACK, EMAIL, SMS, WHATSAPP, TEAMS
- [ ] All 5 production construction sites supply both new fields
- [ ] `casehub-connectors-cloud-events` submodule exists in parent pom
- [ ] `ConnectorCloudEventAdapter` observes `@ObservesAsync InboundMessage` and fires `Event<CloudEvent>.fireAsync()`
- [ ] CloudEvent `type` = `io.casehub.connectors.inbound.<connectorType>`
- [ ] `tenancyid` extension conditionally set (omitted if null)
- [ ] `fireAsync` failure logged at SEVERE
- [ ] Unit test: fire `InboundMessage` → assert `CloudEvent` with correct `type`, `source`, `tenancyid`
- [ ] `EmailInboundAccount` has nullable `tenancyId` field
- [ ] All test call sites constructing `InboundMessage` updated to canonical constructor

### iot fix
- [ ] `IoTCloudEventAdapter` injects `ObjectMapper`
- [ ] `tenancyid` extension omitted when `tenancyId()` is null
- [ ] `fireAsync` failure logged at SEVERE

### qhorus fix
- [ ] `QhorusCloudEventAdapter` handles `fireAsync` CompletionStage failure

### cross-cutting
- [ ] Garden protocol entry committed
- [ ] PLATFORM.md Capability Ownership row for CloudEvent adapter updated (connectors added)
