# CloudEvent Adapter Consistency — connectors, iot, qhorus

**Date:** 2026-06-21
**Status:** Approved
**Issues:** casehubio/connectors#20 (primary), new iot issue (TBD), new qhorus issue (TBD)
**Architectural context:** `docs/superpowers/specs/2026-06-13-p0-layering-decisions-design.md` in casehubio/parent

---

## Scope

connectors#20 (implement `InboundMessage → CloudEvent` adapter) revealed that the two existing
adapters — `IoTCloudEventAdapter` (iot#19, closed) and `QhorusCloudEventAdapter` (qhorus#279,
closed) — contain two latent bugs, one silent failure path, and one pattern inconsistency.
All three repos are fixed together so the platform has one coherent CloudEvent adapter pattern.

---

## Canonical CloudEvent Adapter Pattern

All adapters in the platform must satisfy these rules:

| Rule | Requirement | Rationale |
|---|---|---|
| ObjectMapper | Injected — never a private static instance | Static instance isolates serialisation config from the app's shared mapper; custom serializers registered by the application are invisible |
| tenancyId | Null-safe — extension omitted when null, never set to string `"null"` | SDK serialises null extension as literal `"null"` or throws NPE; downstream consumers receive malformed data |
| fireAsync | Always `.exceptionally(ex -> { LOG.warn(...); return null; })` | Unhandled CompletionStage swallows downstream observer failures silently on the managed executor thread |
| Serialisation error | Catch at WARN, fire with `new byte[0]` data — never throw from `@ObservesAsync` | Unchecked exceptions from async observers propagate into CDI's managed executor and are silently swallowed; the observer method returns normally from CDI's perspective |
| type derivation | From a stable semantic field on the event record — not from an instance identifier | Instance IDs are mutable; CloudEvent `type` is used for routing rules that must be stable across renames |
| Severity | WARN for all degraded-but-non-fatal paths (serialisation failure, fireAsync failure) | SEVERE causes alert fatigue; a single event's serialisation failure is degraded, not catastrophic; inbound message processing is unaffected |

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

### 1c. InboundConnectorIds — add missing TEAMS constant

Pre-existing gap: `TeamsInboundConnector` uses a package-private `static final String ID = "teams-inbound"`
rather than a constant in `InboundConnectorIds`. Since we're adding the parallel `InboundConnectorTypes`
class, add `TEAMS_INBOUND = "teams-inbound"` to `InboundConnectorIds` and update
`TeamsInboundConnector` to reference the constant. Enforces the `inbound-connector-id-constants` protocol.

### 1d. tenancyId at production sites

**Webhook connectors** (Slack, Teams, Twilio SMS, WhatsApp): read from
`request.header("x-tenancy-id")`. `WebhookRequest` already carries lower-cased headers; no new
dependency required.

**Note on `x-tenancy-id` origin:** External platforms (Slack, Twilio, Meta, Teams) do not set this
header. It is injected by infrastructure (reverse proxy / API gateway) in multi-tenant deployments.
When absent (single-tenant), `tenancyId` is null, and the CloudEvent `tenancyid` extension is
omitted. This is correct — single-tenant deployments have no tenant routing requirement.

**Email inbound** (`EmailInboundConnector`): `EmailInboundAccount` gains a `tenancyId` String
field (nullable). `toInboundMessage()` reads it directly.

### 1e. Production construction site migrations

| File | connectorType | tenancyId source |
|---|---|---|
| `SlackInboundConnector.java:168` | `InboundConnectorTypes.SLACK` | `request.header("x-tenancy-id")` |
| `TeamsInboundConnector.java:108` | `InboundConnectorTypes.TEAMS` | `request.header("x-tenancy-id")` |
| `TwilioSmsInboundConnector.java:96` | `InboundConnectorTypes.SMS` | `request.header("x-tenancy-id")` |
| `WhatsAppInboundConnector.java:156` | `InboundConnectorTypes.WHATSAPP` | `request.header("x-tenancy-id")` |
| `EmailInboundConnector.java:208,218` | `InboundConnectorTypes.EMAIL` | `account.tenancyId()` |

### 1f. New submodule — casehub-connectors-cloud-events

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

### 1g. Parent pom.xml

Add `<module>cloud-events</module>` to `casehub-connectors-parent`.

---

## Repo 2 — casehub-iot (main, new issue)

Four fixes to `IoTCloudEventAdapter`:

**Fix 1 — Inject ObjectMapper (bug)**

Static ObjectMapper isolates serialisation config from the application's shared mapper.
Custom serializers registered by the application are invisible.

```java
// Before
private static final ObjectMapper MAPPER = new ObjectMapper()
    .registerModule(new JavaTimeModule())
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

// After
private final ObjectMapper objectMapper;  // injected via constructor
```

**Fix 2 — Null-safe tenancyId (pattern conformance, not a bug)**

`DeviceEntity` enforces `Objects.requireNonNull(builder.tenancyId, "tenancyId")` in its
constructor, so null cannot currently reach the adapter. This guard is defence in depth —
if the domain model changes or a new `DeviceEntity` subclass is introduced, the adapter
must not break.

```java
// Before
.withExtension("tenancyid", event.after().tenancyId())

// After
CloudEventBuilder builder = CloudEventBuilder.v1()...;
if (event.after().tenancyId() != null) {
    builder = builder.withExtension("tenancyid", event.after().tenancyId());
}
```

**Fix 3 — Serialisation error handling (silent failure path)**

`throw new UncheckedIOException(e)` fires from inside an `@ObservesAsync` observer.
CDI catches the exception on the managed executor thread. The exception never reaches
`fireAsync()` — it happens before `CloudEventBuilder` is even called. The observer
method fails silently from the application's perspective: no log, no alert, the
`StateChangeEvent` is simply lost.

```java
// Before
} catch (JsonProcessingException e) {
    throw new UncheckedIOException(e);
}

// After
} catch (JsonProcessingException e) {
    LOG.warnf("Failed to serialise StateChangeEvent for CloudEvent — device=%s: %s",
            event.after().deviceId(), e.getMessage());
    data = new byte[0];
}
```

**Fix 4 — Handle fireAsync CompletionStage (bug)**

```java
// Before
cloudEvents.fireAsync(ce);

// After
cloudEvents.fireAsync(ce)
    .exceptionally(ex -> {
        LOG.warnf(ex, "CloudEvent dispatch failed for device=%s",
                event.after().deviceId());
        return null;
    });
```

**Summary:** Two bugs (static ObjectMapper, unhandled fireAsync), one silent failure path
(thrown exception from async observer), one pattern conformance fix (null tenancyId guard).

---

## Repo 3 — casehub-qhorus (main, new issue)

**Scope: 1 adapter fix + ~14 test construction site migrations.**

### Adapter fix

One fix to `QhorusCloudEventAdapter` — ObjectMapper already injected, tenancyId already null-safe,
serialisation errors already handled with log-and-empty-data:

**Fix — Handle fireAsync CompletionStage (bug)**
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

### Test construction site migrations

Adding `connectorType` and `tenancyId` to `InboundMessage` and removing convenience constructors
breaks every `new InboundMessage(...)` call in qhorus. 14 construction sites across 7 test files:

| File | Sites |
|---|---|
| `ConnectorAutoChannelBackendTest.java` | 1 |
| `ConfiguredAutoChannelPolicyTest.java` | 4 (line 113 also fixes pre-existing protocol violation: uses raw `"slack-inbound"` string instead of `InboundConnectorIds.SLACK_INBOUND`) |
| `ConcurrentAutoChannelTest.java` | 2 |
| `ConnectorKeyStrategyTest.java` | 1 |
| `ConnectorChannelBackendTest.java` | 2 |
| `ConnectorChannelBackendIntegrationTest.java` | 3 |
| `SlackChannelBackendTest.java` | 1 |

All mechanical: add `connectorType` and `tenancyId` args to each `new InboundMessage(...)`.
qhorus production code (`ConnectorChannelBackend`, `SlackChannelBackend`, `ConnectorKeyStrategy`,
`ConfiguredAutoChannelPolicy`, `AutoChannelPolicy`) consumes `InboundMessage` via method calls
but does not construct instances — no production code changes beyond the adapter.

---

## Garden Protocol

New entry documenting the canonical CloudEvent adapter pattern — the six rules from the table
above, with rationale for each. Captures: why static ObjectMapper is wrong, why null tenancyId
corrupts extensions, why fireAsync needs `.exceptionally()`, why exceptions thrown from
`@ObservesAsync` are silently swallowed, why type must use a semantic field, why WARN not SEVERE.

---

## Issues to File Before Implementation

| Repo | Title | Scope |
|---|---|---|
| casehubio/iot | fix: IoTCloudEventAdapter — inject ObjectMapper, null-safe tenancyId, handle serialisation + fireAsync | 4 fixes to one class |
| casehubio/qhorus | fix: QhorusCloudEventAdapter fireAsync + InboundMessage constructor migration | 1 adapter fix + ~14 test site migrations |

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
- [ ] `InboundMessage` has `connectorType` and `tenancyId` fields; convenience constructors removed
- [ ] `InboundConnectorTypes` constants class exists with SLACK, EMAIL, SMS, WHATSAPP, TEAMS
- [ ] `InboundConnectorIds` gains `TEAMS_INBOUND`; `TeamsInboundConnector` references it
- [ ] All 5 production construction sites supply both new fields
- [ ] All test call sites constructing `InboundMessage` updated to canonical constructor
- [ ] `casehub-connectors-cloud-events` submodule exists in parent pom
- [ ] `ConnectorCloudEventAdapter` observes `@ObservesAsync InboundMessage` and fires `Event<CloudEvent>.fireAsync()`
- [ ] CloudEvent `type` = `io.casehub.connectors.inbound.<connectorType>`
- [ ] `tenancyid` extension conditionally set (omitted if null)
- [ ] Serialisation error: log at WARN, fire with empty data
- [ ] `fireAsync` failure: log at WARN via `.exceptionally()`
- [ ] Unit test: fire `InboundMessage` → assert `CloudEvent` with correct `type`, `source`, `tenancyid`
- [ ] `EmailInboundAccount` has nullable `tenancyId` field

### iot fix
- [ ] `IoTCloudEventAdapter` injects `ObjectMapper` (bug fix)
- [ ] `tenancyid` extension omitted when `tenancyId()` is null (pattern conformance)
- [ ] Serialisation error: catch at WARN, empty data (silent failure path fix)
- [ ] `fireAsync` failure: `.exceptionally()` handler (bug fix)

### qhorus fix
- [ ] `QhorusCloudEventAdapter` handles `fireAsync` CompletionStage failure (bug fix)
- [ ] ~14 test construction sites migrated to 9-arg canonical `InboundMessage` constructor
- [ ] `ConfiguredAutoChannelPolicyTest:113` uses `InboundConnectorIds.SLACK_INBOUND` (protocol violation fix)

### cross-cutting
- [ ] Garden protocol entry committed — canonical CloudEvent adapter pattern with 6 rules
- [ ] PLATFORM.md Capability Ownership row for CloudEvent adapter updated (connectors added)
