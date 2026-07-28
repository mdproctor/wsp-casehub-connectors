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

Default `PER_USER` in the compact constructor provides a convenient default
for callers that don't care about scope — passing `null` yields `PER_USER`.
All existing 7-argument call sites must be updated to pass the 8th argument.

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
`ResolvedChannel`. Two per-tenant overrides:

1. **Digest prevention**: `digested` is always `false` when
   `destinationScope == PER_TENANT`. Digest is a per-user concept
   (batch *my* notifications) — a shared webhook has no per-user schedule.

2. **Quiet hours buffering gate**: `quietHoursBuffering` is `false` when
   `destinationScope == PER_TENANT`. Without this, the "don't suppress
   because we'll digest instead" logic still fires even though digest is
   disabled, causing per-tenant channels to deliver immediately during
   quiet hours. With the gate, per-tenant channels follow the normal
   suppression path:

   ```java
   final boolean quietHoursBuffering = suppressionResult.quietHoursActive()
           && quietHoursAction == QuietHoursAction.BUFFER_FOR_DIGEST
           && effectiveDigest != null
           && descriptor.destinationScope() != DestinationScope.PER_TENANT;
   ```

**Modified class** `NotificationDispatcher`:

In `onMatch()`, create `Map<String, DeliveryResult> perTenantResults` and
pass it through the per-user loop.

In `dispatchToUser()`, the dedup check runs **after** digest/suppress
routing but **before** delivery. This placement ensures per-user routing
decisions (snooze, quiet hours, enabled/disabled) are respected: a user
who has the channel suppressed skips it entirely and neither contributes
to nor consumes from the dedup map.

```java
for (final ResolvedChannel channel : channels) {
    if (channel.digested()) {
        digestBuffer.add(new DigestBufferKey(userId, tenancyId, channel.channelId()),
                notificationInput);
        continue;
    }
    if (channel.suppressed()) {
        continue;
    }

    // Per-tenant dedup — after routing, before delivery
    if (channel.destinationScope() == DestinationScope.PER_TENANT) {
        DeliveryResult previous = perTenantResults.get(channel.channelId());
        if (previous != null) {
            LOG.debugf("Per-tenant dedup: channel '%s' already delivered for tenancy '%s', "
                    + "propagating %s to user '%s'",
                    channel.channelId(), tenancyId,
                    previous.success() ? "success" : "failure", userId);
            if (previous.success()) {
                deliveryTracker.recordSuccess(channel.channelId(), notificationInput,
                        null, DeliverySourceType.NOTIFICATION);
            } else {
                // null guaranteedMinSeverity → FAILED, not RETRYING
                deliveryTracker.recordFailure(channel.channelId(), notificationInput,
                        null, DeliverySourceType.NOTIFICATION,
                        null, previous.failureReason());
            }
            continue;
        }
    }

    try {
        final DeliveryResult result = channel.deliverer().deliver(notificationInput);
        if (channel.destinationScope() == DestinationScope.PER_TENANT) {
            perTenantResults.put(channel.channelId(), result);
        }
        if (result.success()) {
            deliveryTracker.recordSuccess(channel.channelId(), notificationInput,
                    null, DeliverySourceType.NOTIFICATION);
        } else {
            LOG.warnf("Delivery failed for channel '%s', user '%s': %s",
                    channel.channelId(), userId, result.failureReason());
            deliveryTracker.recordFailure(channel.channelId(), notificationInput,
                    null, DeliverySourceType.NOTIFICATION,
                    channel.guaranteedMinSeverity(), result.failureReason());
        }
    } catch (Exception e) {
        var failedResult = new DeliveryResult(false, e.getMessage());
        if (channel.destinationScope() == DestinationScope.PER_TENANT) {
            perTenantResults.put(channel.channelId(), failedResult);
        }
        LOG.warnf(e, "Delivery error for channel '%s', user '%s'",
                channel.channelId(), userId);
        deliveryTracker.recordFailure(channel.channelId(), notificationInput,
                null, DeliverySourceType.NOTIFICATION,
                channel.guaranteedMinSeverity(), e.getMessage());
    }
}
```

The dedup ensures:

- **Immediate delivery**: first non-suppressed user delivers; subsequent
  non-suppressed users get the propagated result.
- **Digest**: per-tenant channels are never digested (enforced by
  `ChannelRouter`), so the digest path is never reached.
- **Retry**: only the first user's failure record is retry-eligible (has
  `guaranteedMinSeverity`). Propagated failures pass `null` for
  `guaranteedMinSeverity`, creating `FAILED` (not `RETRYING`) records.
  One broken webhook produces one retry chain, not N.
- **Exception path**: caught exceptions store the failure in
  `perTenantResults`, preventing subsequent users from re-attempting
  delivery to the same broken endpoint.

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
- `dispatch_perTenantChannel_failurePropagation_notRetryEligible`:
  first user's delivery fails with `guaranteedMinSeverity` → first user's
  record is `RETRYING`, propagated users' records are `FAILED`.
- `dispatch_perTenantChannel_exceptionCaptured_subsequentUsersSkipDelivery`:
  first user's delivery throws → exception stored in `perTenantResults` →
  subsequent users skip delivery and record failure.
- `dispatch_perTenantChannel_suppressedUserSkipped_nextUserDelivers`:
  first user has channel snoozed → skipped (no dedup entry) → second user
  delivers normally.

### Platform — ChannelRouterTest

- `ResolvedChannel.destinationScope()` propagates from descriptor.
- `route_perTenantChannel_neverDigested`: register `PER_TENANT` channel
  with digest schedule, assert `digested` is `false`.
- `route_perTenantChannel_quietHoursBuffering_stillSuppressed`: register
  `PER_TENANT` channel with digest schedule, route with quiet hours active
  and `BUFFER_FOR_DIGEST`, assert channel is `suppressed` (not delivered
  immediately).

### Connectors — NotificationBridgeStartupTest

- Slack/Teams register with `PER_TENANT`.
- Email/SMS/WhatsApp register with `PER_USER`.

### Connectors — SlackConnector / TeamsConnector

- `channelType()` returns `"slack"` / `"teams"` (no longer null).

## Cross-Repo Execution Order

1. Platform API + implementation changes → `mvn install` to local repo
2. Connectors changes (depend on updated platform-api SNAPSHOT)
3. Blocks-UI: update TypeScript `DeliveryChannelDescriptor` interface in
   `casehub-blocks-ui-npm` to include `destinationScope` field (additive —
   existing UI code is unaffected, field is informational)
4. Push platform first, then connectors, then blocks-ui
