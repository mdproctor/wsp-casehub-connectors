# Notification Delivery Bridge — Design Spec

**Issue:** casehubio/connectors#86
**Date:** 2026-07-20
**Status:** Approved

## Problem

Platform's notification system (`NotificationDeliverer`, `DeliveryChannelRegistry`,
`NotificationDispatcher`) supports pluggable delivery channels, but the only
implementation is `InAppNotificationDeliverer`. The `DeliveryChannels` constants
define `EMAIL`, `SMS`, `PUSH` alongside `IN_APP`, but none have deliverers.

Connectors already has the outbound SPI: `Connector.id()` + `send(ConnectorMessage)`,
CDI auto-discovered, with built-in implementations for `slack`, `teams`, `twilio-sms`,
`whatsapp`, `email`.

These two systems need a bridge so any connector automatically becomes a notification
delivery channel.

## Industry Context

Notification infrastructure systems (Knock, MagicBell, Novu) converge on a
three-concept model:

1. **Channel** — delivery medium type (email, SMS, push, chat)
2. **Provider** — swappable implementation (SendGrid, Twilio, FCM)
3. **Subscriber profile** — user record with structured contact fields (email, phone,
   device tokens) stored separately from notification preferences

Contact attributes answer "where can I be reached?" — notification preferences answer
"how do I want to be notified?" These are separate concerns with separate stores.
Novu's subscriber model is the most explicit: structured fields (`email`, `phone`) on
the subscriber entity, resolved at delivery time.

Our platform lacks a user directory with contact attributes. The `DestinationResolver`
SPI introduced here is the seam for this — a future user profile store implements it.

## Design Decisions

### Channel type vs connector implementation (two-level model)

`Connector.id()` is implementation-specific (`"twilio-sms"`). Notification channels are
delivery medium types (`"sms"`). The bridge registers channels by type, not by
connector id.

`Connector` gains a `channelType()` default method returning `id()`. Connectors where
the implementation id differs from the channel type override it (e.g.,
`TwilioSmsConnector` returns `"sms"`). Returning `null` opts out of notification
bridging.

Payoff: swap Twilio for Vonage (same channelType `"sms"`, different connector id) —
user preferences and destination resolvers remain unchanged.

### DestinationResolver in platform-api

The resolver SPI lives in `casehub-platform-api` alongside the notification SPIs, not
in the bridge module. "Resolve a user's delivery destination" is a platform-level
concern — any `NotificationDeliverer` (not just connector-backed) may need it.

Per-channel CDI beans with `channelId()`, matching the `NotificationDeliverer` pattern.
Returns `Optional<String>` — empty means no known destination, delivery skips.

### Bridge-wide descriptor defaults

All bridged connectors use the same `DeliveryChannelDescriptor` defaults. Connectors
do not declare notification metadata — that's the bridge's concern.

- `external = true`
- `defaultEnabled = false` (users must explicitly enable)
- `defaultMinSeverity = WARNING`
- `defaultDigestSchedule = null` (immediate delivery)
- `guaranteedMinSeverity = null` (no retry)

### connectors-core stays independent of platform-api

`TwilioSmsConnector.channelType()` returns the string literal `"sms"`, not
`DeliveryChannels.SMS`. This avoids creating a dependency from connectors-core to
platform-api. The bridge module is where the two worlds meet.

## Changes

### casehub-platform-api

**New: `DestinationResolver`** in `io.casehub.platform.api.delivery`:

```java
public interface DestinationResolver {
    String channelId();
    Optional<String> resolve(String userId, String tenancyId);
}
```

**Modified: `DeliveryChannels`** — add constants:

```java
public static final String SLACK = "slack";
public static final String TEAMS = "teams";
public static final String WHATSAPP = "whatsapp";
```

### casehub-connectors-core

**Modified: `Connector`** — add default method:

```java
default String channelType() { return id(); }
```

**Modified: `TwilioSmsConnector`** — override:

```java
@Override
public String channelType() { return "sms"; }
```

### casehub-connectors — new `notification-bridge` module

**Dependencies:**
- `casehub-platform-api` — notification SPIs + DestinationResolver
- `casehub-connectors-core` — Connector SPI

**`ConnectorNotificationDeliverer`** — one instance per bridged connector:

```java
class ConnectorNotificationDeliverer implements NotificationDeliverer {
    private final Connector connector;
    private final String channelType;
    private final DestinationResolver resolver; // nullable

    @Override
    public String channelId() { return channelType; }

    @Override
    public DeliveryResult deliver(NotificationInput notification) {
        if (resolver == null) {
            return new DeliveryResult(false, "no destination resolver for " + channelType);
        }
        Optional<String> destination = resolver.resolve(
                notification.userId(), notification.tenancyId());
        if (destination.isEmpty()) {
            return new DeliveryResult(false, "no destination for user " + notification.userId());
        }
        try {
            String body = notification.body() != null
                    ? notification.body() : notification.title();
            connector.send(new ConnectorMessage(
                    destination.get(), notification.title(), body));
            return new DeliveryResult(true, null);
        } catch (Exception e) {
            return new DeliveryResult(false, e.getMessage());
        }
    }
}
```

**`NotificationBridgeStartup`** — `@ApplicationScoped`, wires everything at
`@PostConstruct`:

```java
1. Inject @All List<Connector> — all CDI connector beans
2. Inject @All List<DestinationResolver> — index by channelId()
3. Inject DeliveryChannelRegistry

4. For each connector where channelType() != null:
   a. Duplicate channelType check → startup error
   b. Match DestinationResolver by channelId (may be null)
   c. Create ConnectorNotificationDeliverer
   d. Build DeliveryChannelDescriptor with bridge-wide defaults:
      - channelId = channelType
      - displayName = channelType with first letter capitalised (e.g., "Email", "Sms", "Slack")
      - external = true
      - defaultEnabled = false
      - defaultMinSeverity = WARNING
      - defaultDigestSchedule = null
      - guaranteedMinSeverity = null
   e. registry.register(descriptor, deliverer)
```

**`NotificationInput` → `ConnectorMessage` mapping:**

| NotificationInput | ConnectorMessage |
|-------------------|------------------|
| (from resolver)   | destination      |
| title()           | title            |
| body() ?: title() | body             |
| (none)            | attributes       |

### Parent pom.xml

Add `<module>notification-bridge</module>` to the modules list.

## Dependency Direction

```
notification-bridge → platform-api     (notification SPIs)
notification-bridge → connectors-core  (Connector SPI)
connectors-core    → (no platform dependency)
platform           → (no connectors dependency)
```

The bridge is the only place the two worlds meet.

## What Does Not Change

- `NotificationDispatcher`, `ChannelRouter`, `InAppNotificationDeliverer` — work as-is
- `ConnectorService`, `ConnectorDiscovery` — untouched
- `NotificationPreferences`, `ChannelPreference` — user preferences key on channelType,
  which the existing model already supports

## Testing Strategy

- **Unit tests for `ConnectorNotificationDeliverer`**: resolver present/absent, destination
  found/not found, connector send success/failure, null body fallback
- **Unit tests for `NotificationBridgeStartup`**: auto-discovery of connectors, resolver
  matching, duplicate channelType detection, null channelType opt-out
- **Contract test for `DestinationResolver`**: follows `NotificationDelivererContractTest`
  pattern in platform-api
- **Integration test**: bridge startup with real CDI container, verify channels appear in
  `DeliveryChannelRegistry.discover()`
