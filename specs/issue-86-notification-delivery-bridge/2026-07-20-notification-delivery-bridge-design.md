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

1. **Channel** — delivery platform or medium (email, SMS, push, Slack, Teams)
2. **Provider** — swappable implementation within a channel (SendGrid, Twilio, FCM)
3. **Subscriber profile** — user record with structured contact fields (email, phone,
   device tokens) stored separately from notification preferences

Channels are defined at the level where users need independent preference control.
SMS and email are medium types — swapping Twilio for Vonage within SMS is transparent
to users. Slack and Teams are distinct channels: different platforms, different
audiences, different capabilities, independent user preferences.

Contact attributes answer "where can I be reached?" — notification preferences answer
"how do I want to be notified?" These are separate concerns with separate stores.
Novu's subscriber model is the most explicit: structured fields (`email`, `phone`) on
the subscriber entity, resolved at delivery time.

Our platform lacks a user directory with contact attributes. The `DestinationResolver`
SPI introduced here is the seam for this — a future user profile store implements it.

## Design Decisions

### Channel type vs connector implementation

`Connector.id()` is implementation-specific (`"twilio-sms"`). Notification channels are
delivery platforms or medium types (`"sms"`, `"slack"`). The bridge registers channels
by channel type, not by connector id.

Channels are defined at the level where users need independent preference control:
- **Medium-type channels** (SMS, email): multiple providers may share one channel type.
  Swap Twilio for Vonage — user preferences and destination resolvers remain unchanged.
- **Platform channels** (Slack, Teams, WhatsApp): each platform is its own channel type.
  Different audiences, capabilities, and user preferences.

`Connector` gains a `channelType()` default method returning `id()`. Connectors where
the implementation id differs from the channel type override it (e.g.,
`TwilioSmsConnector` returns `"sms"`). Returning `null` opts out of notification
bridging.

### DestinationResolver in platform-api

The resolver SPI lives in `casehub-platform-api` alongside the notification SPIs, not
in the bridge module. "Resolve a user's delivery destination" is a platform-level
concern — any `NotificationDeliverer` (not just connector-backed) may need it.

Per-channel CDI beans with `channelId()`, matching the `NotificationDeliverer` pattern.
Returns `Optional<String>` — empty means no known destination, delivery skips.

#### Per-user vs per-tenant destinations

Destinations follow two resolution models:
- **Per-user** (email, SMS, WhatsApp): each user has their own destination (email
  address, phone number). The resolver looks up the user's contact attribute.
- **Per-tenant** (Slack, Teams): the destination is a shared webhook URL configured
  per tenant or integration. All users in a tenant resolve to the same URL.

Both models use the same `DestinationResolver` SPI — the implementation determines
the resolution strategy.

#### Initial bridge scope: per-user channels only

Per-tenant destinations have a deduplication problem: when a subscription matches
N users, the dispatcher calls `deliver()` N times per channel. For per-user channels
(email, SMS), each call delivers to a different destination — correct. For per-tenant
channels (Slack, Teams), all N calls deliver to the same webhook URL, producing N
identical messages in the same channel. A team of 20 people generates 20 copies of
every notification.

Deduplication cannot happen in the bridge — `deliver()` is called per-user with no
visibility into other users' destinations. It must happen in the dispatcher, which
sees all recipients but currently resolves destinations inside deliverers (#2).

Until #2 lands, Slack and Teams connectors opt out of notification bridging by
returning `null` from `channelType()`. The initial bridge covers **email, SMS, and
WhatsApp** — all per-user destination channels. Slack and Teams remain fully
functional as connectors (MCP tools, direct `ConnectorService` use) but are not
registered as notification delivery channels.

Future per-user Slack/Teams connectors (e.g., bot-based DMs) would return a non-null
`channelType()` and bridge correctly without deduplication.

#### Relationship to ConnectorDiscovery

`ConnectorDiscovery` in connectors-core answers "what delivery targets are reachable?"
for MCP tool use (e.g., listing available Slack channels for an agent).
`DestinationResolver` answers "where does this user's notification go?" for automated
delivery routing. These are distinct concerns at different layers:
- `ConnectorDiscovery` — interactive, exploratory, agent-driven
- `DestinationResolver` — automated, deterministic, system-driven

They operate independently and are not substitutable.

### Bridge-wide descriptor defaults

All bridged connectors share common descriptor defaults with per-channel-type
overrides where reliability profiles differ.

Common defaults:
- `external = true`
- `defaultEnabled = false` (users must explicitly enable)
- `defaultMinSeverity = WARNING`
- `defaultDigestSchedule = null` (immediate delivery; digest not yet supported — #3)

Per-channel-type retry policy (`guaranteedMinSeverity`):
- Email, SMS: `WARNING` (retry on transient SMTP/carrier failures)
- WhatsApp: `null` (no retry — API failures are typically configuration errors)

Slack and Teams are not bridged initially (see §Initial bridge scope).

Display names are explicit per connector, not algorithmically derived:

| channelType | displayName |
|---|---|
| `email` | "Email" |
| `sms` | "SMS" |
| `whatsapp` | "WhatsApp" |
| (unknown) | channelType as-is |

### Channel enablement

Bridged channels register in `DeliveryChannelRegistry` at startup with
`defaultEnabled = false`. Users enable channels through the existing
`NotificationPreferenceStore` (supports in-memory and JPA backends). The
`ChannelPreference` model already includes `enabled`, `minSeverity`, and
`digestSchedule` — no new preference model is needed.

A REST API for notification preference management is not yet available (#4).

### Known connector reliability gaps

Some connectors have known limitations that affect delivery result accuracy:
- WhatsApp: Meta may return HTTP 200 with an error body for invalid template names,
  which `HttpHelper` treats as success (#5). The bridge will report
  `DeliveryResult(true)` for these silent failures until the connector is fixed.

These are pre-existing connector issues, not bridge design problems, but they affect
the bridge's end-to-end reliability.

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

**Modified: `DeliveryChannels`** — add constant:

```java
public static final String WHATSAPP = "whatsapp";
```

`SLACK` and `TEAMS` constants are deferred until per-tenant deduplication lands (#2).

### casehub-connectors-core

**Modified: `Connector`** — change `send()` return type and add `channelType()`:

```java
/**
 * Send a message via this connector.
 *
 * @param message the message to deliver; must not be null
 * @return true if delivery succeeded, false on failure
 */
boolean send(ConnectorMessage message);

default String channelType() { return id(); }
```

All built-in connectors already track success/failure internally (HTTP status checks,
try/catch). The change from `void` to `boolean` makes failure reporting explicit
rather than swallowed. The "must not throw" contract is unchanged — connectors catch
exceptions internally and return `false`.

**Modified: `ConnectorService.send()`** — return type changes from `void` to `boolean`
to propagate the connector's delivery result. Existing callers that ignore the return
value compile without changes.

**Modified: `TwilioSmsConnector`** — override:

```java
@Override
public String channelType() { return "sms"; }
```

**Modified: `SlackConnector`** — opt out of notification bridging (per-tenant
webhook destinations produce N duplicates; see §Initial bridge scope):

```java
@Override
public String channelType() { return null; }
```

**Modified: `TeamsConnector`** — same reason:

```java
@Override
public String channelType() { return null; }
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
            var attributes = new java.util.HashMap<String, String>();
            attributes.put("category", notification.category());
            attributes.put("severity", notification.severity().name());
            if (notification.actionUrl() != null) {
                attributes.put("actionUrl", notification.actionUrl());
            }
            boolean success = connector.send(new ConnectorMessage(
                    destination.get(), notification.title(), body, attributes));
            return new DeliveryResult(success,
                    success ? null : "connector reported delivery failure");
        } catch (Exception e) {
            return new DeliveryResult(false, e.getMessage());
        }
    }

    @Override
    public DeliveryResult deliverDigest(DigestSummary summary) {
        return new DeliveryResult(false,
                "digest delivery not yet supported for bridged channels");
    }
}
```

**`NotificationBridgeStartup`** — `@Startup @ApplicationScoped`, wires everything at
`@PostConstruct`:

```java
1. Inject @All List<Connector> — all CDI connector beans
2. Inject @All List<DestinationResolver> — index by channelId()
3. Inject DeliveryChannelRegistry

4. For each connector where channelType() != null:
   a. Duplicate channelType check → startup error
   b. Match DestinationResolver by channelId (may be null)
   c. Create ConnectorNotificationDeliverer
   d. Build DeliveryChannelDescriptor:
      - channelId = channelType
      - displayName = from explicit display name map (see §Bridge-wide descriptor defaults)
      - external = true
      - defaultEnabled = false
      - defaultMinSeverity = WARNING
      - defaultDigestSchedule = null
      - guaranteedMinSeverity = per-channel-type (see §Bridge-wide descriptor defaults)
   e. registry.register(descriptor, deliverer)
```

**`NotificationInput` → `ConnectorMessage` mapping:**

| NotificationInput | ConnectorMessage |
|-------------------|------------------|
| (from resolver)   | destination      |
| title()           | title            |
| body() ?: title() | body             |
| category, severity, actionUrl | attributes |

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
- `ConnectorDiscovery` — untouched; distinct concern from `DestinationResolver`
  (see §DestinationResolver in platform-api)
- `NotificationPreferences`, `ChannelPreference` — user preferences key on channelType,
  which the existing model already supports

## Testing Strategy

- **Unit tests for `ConnectorNotificationDeliverer`**: resolver present/absent, destination
  found/not found, connector send success/failure (boolean return), null body fallback,
  metadata attribute mapping, digest rejection
- **Unit tests for `NotificationBridgeStartup`**: auto-discovery of connectors, resolver
  matching, duplicate channelType detection, null channelType opt-out, per-channel-type
  display names and retry policies
- **Contract test for `DestinationResolver`**: follows `NotificationDelivererContractTest`
  pattern in platform-api
- **Integration test**: bridge startup with real CDI container, verify channels appear in
  `DeliveryChannelRegistry.discover()`
