# Per-Tenant Destination Deduplication — Design Spec

**Issue:** casehubio/connectors#90
**Date:** 2026-07-28
**Status:** approved

## Problem

The notification dispatch loop in `NotificationDispatcher` iterates per-user,
then per-channel. For per-user channels (email, SMS), each user resolves to a
distinct destination — N users produce N distinct deliveries. For per-tenant
channels (Slack webhook, Teams webhook), all users in the same tenancy resolve
to the same destination, producing N identical deliveries to the same webhook URL.

Slack and Teams connectors currently opt out of notification bridging
(`channelType() → null`) to avoid this. Unblocking them requires
dispatcher-level deduplication.

## Root Cause

The dispatch loop has no concept of channels whose destination is shared across
users within a tenancy. `deliver()` is called per-user with no visibility into
other users' destinations, so dedup cannot happen at the delivery layer.

## Design Decision

The addressing model (per-user vs per-tenant) is a static, declarative property
of the channel type — known at registration time, immutable, not dependent on
notifications or user preferences.

The right place for this metadata is `DeliveryChannelDescriptor`, not the
`NotificationDeliverer` SPI:

- The descriptor already holds channel metadata (`external`, `defaultEnabled`,
  `defaultMinSeverity`, etc.). Scope is the same category.
- Adding `resolveDestination()` to the deliverer SPI would leak destination
  resolution into the dispatcher, breaking the black-box delivery contract.
- `InAppNotificationDeliverer` doesn't use destination resolution — forcing
  a scope method on the SPI burdens unrelated implementations.

## Changes

### Platform API (`casehub-platform-api`)

**New enum** `DestinationScope` in `io.casehub.platform.api.delivery`:

```java
public enum DestinationScope {
    PER_USER,
    PER_TENANT
}
```

**Modified record** `DeliveryChannelDescriptor` — add 8th field:

```java
public record DeliveryChannelDescriptor(
        String channelId,
        String displayName,
        boolean external,
        boolean defaultEnabled,
        NotificationSeverity defaultMinSeverity,
        DigestSchedule defaultDigestSchedule,
        NotificationSeverity guaranteedMinSeverity,
        DestinationScope destinationScope
) {
    public DeliveryChannelDescriptor {
        Objects.requireNonNull(channelId, "channelId");
        Objects.requireNonNull(displayName, "displayName");
        Objects.requireNonNull(defaultMinSeverity, "defaultMinSeverity");
        if (destinationScope == null) destinationScope = DestinationScope.PER_USER;
    }
}
```

Default `PER_USER` in the compact constructor ensures backward compatibility for
existing call sites that pass `null`.

### Platform Implementation (`casehub-platform`)

**Modified record** `ResolvedChannel` — add `destinationScope` field:

```java
public record ResolvedChannel(
        String channelId,
        NotificationDeliverer deliverer,
        boolean suppressed,
        boolean digested,
        NotificationSeverity guaranteedMinSeverity,
        DestinationScope destinationScope
) { ... }
```

**Modified class** `ChannelRouter` — propagate scope from descriptor to
`ResolvedChannel`.

**Modified class** `NotificationDispatcher`:

In `onMatch()`, create `Set<String> deliveredPerTenantChannels` and pass it
through the per-user loop.

In `dispatchToUser()`, use `Map<String, DeliveryResult> perTenantResults`
(passed from `onMatch()`) to track the outcome of the first delivery per
per-tenant channel:

```java
if (channel.destinationScope() == DestinationScope.PER_TENANT) {
    DeliveryResult previous = perTenantResults.get(channel.channelId());
    if (previous != null) {
        // Already delivered — propagate the same result for this user
        if (previous.success()) {
            deliveryTracker.recordSuccess(...);
        } else {
            deliveryTracker.recordFailure(..., previous.failureReason());
        }
        continue;
    }
}

// ... deliver ...
// After delivery, if PER_TENANT: perTenantResults.put(channelId, result)
```

First user delivers and records. Subsequent users skip delivery and record the
same outcome (success or failure) — per-user tracking accountability preserved
for shared channels without retrying a failed shared destination per-user.

### Connectors (`casehub-connectors`)

**Modified class** `NotificationBridgeStartup` — add scope map and pass to
descriptor construction:

```java
private static final Set<String> PER_TENANT_CHANNELS = Set.of("slack", "teams");
```

The scope is set by the bridge (which wires channels into the notification
system), not by the `Connector` SPI (which is a pure send-message interface).
This follows the existing pattern for `DISPLAY_NAMES` and `RETRY_POLICIES`.

**Modified classes** `SlackConnector`, `TeamsConnector` — remove
`channelType() → null` override. With the override removed, `channelType()`
falls back to the `Connector` default returning `id()` ("slack" / "teams").

## Testing

### Platform API

- `DeliveryChannelDescriptorTest`: default scope is `PER_USER`; `PER_TENANT`
  preserved when set.

### Platform — NotificationDispatcherTest

- `dispatch_perTenantChannel_deliversOnceForMultipleRecipients`: register a
  `PER_TENANT` channel, dispatch to 3 users, assert `deliver()` called once,
  `deliveryTracker` records success for all 3.
- `dispatch_perTenantChannel_mixedWithPerUser_bothWork`: register `PER_USER`
  (email) and `PER_TENANT` (slack) channels, dispatch to 2 users, assert
  email delivers twice and slack delivers once.
- `dispatch_perTenantChannel_deliveryFailure_recordsFailureForAllUsers`:
  single delivery fails → all users get failure tracking.

### Platform — ChannelRouterTest

- `ResolvedChannel.destinationScope()` propagates from descriptor.

### Connectors — NotificationBridgeStartupTest

- Slack/Teams register with `PER_TENANT`.
- Email/SMS/WhatsApp register with `PER_USER`.

### Connectors — SlackConnector / TeamsConnector

- `channelType()` returns `"slack"` / `"teams"` (no longer null).

## Cross-Repo Execution Order

1. Platform API + implementation changes → `mvn install` to local repo
2. Connectors changes (depend on updated platform-api SNAPSHOT)
3. Push platform first, then connectors
