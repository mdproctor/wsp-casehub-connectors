# Per-Tenant Destination Deduplication — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #90 — per-tenant destination deduplication in notification dispatcher
**Issue group:** #90

**Goal:** Add `DestinationScope` to the delivery channel model so the
dispatcher deduplicates per-tenant channel deliveries, then enable
Slack/Teams notification bridging.

**Architecture:** New `DestinationScope` enum (`PER_USER`, `PER_TENANT`)
on `DeliveryChannelDescriptor`. `ChannelRouter` propagates scope to
`ResolvedChannel` and prevents per-tenant channels from being digested.
`NotificationDispatcher` tracks `Map<String, DeliveryResult>` across
the per-user loop, skipping delivery for per-tenant channels already
delivered while still recording tracking for all users.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api,
casehub-platform (notification-dispatch module), casehub-connectors
(notification-bridge, core)

## Global Constraints

- All casehubio artifacts are `0.2-SNAPSHOT`
- Platform changes must `mvn install` to local repo before connectors
  can compile against them
- Use `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn` for all builds
- IntelliJ MCP mandatory for all `.java` edits — use `ide_edit_member`,
  `ide_insert_member`, `ide_replace_member`, `ide_create_file`
- Project paths: platform at `/Users/mdproctor/claude/casehub/platform`,
  connectors at `/Users/mdproctor/claude/casehub/connectors`
- Pass `project_path` to every `mcp__intellij-index__*` call

---

### Task 1: Platform API — DestinationScope enum + DeliveryChannelDescriptor field

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/delivery/DestinationScope.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/delivery/DeliveryChannelDescriptor.java`
- Modify: `platform-api/src/test/java/io/casehub/platform/api/delivery/DeliveryChannelDescriptorTest.java`

**Interfaces:**
- Produces: `DestinationScope.PER_USER`, `DestinationScope.PER_TENANT`
- Produces: `DeliveryChannelDescriptor.destinationScope()` — 8th record field

- [ ] **Step 1: Write failing tests for DestinationScope on DeliveryChannelDescriptor**

Add two tests to `DeliveryChannelDescriptorTest`:

```java
@Test
void defaultDestinationScopeIsPerUser() {
    var desc = new DeliveryChannelDescriptor(
            "test", "Test", true, false,
            NotificationSeverity.WARNING, null, null, null);
    assertThat(desc.destinationScope()).isEqualTo(DestinationScope.PER_USER);
}

@Test
void carriesDestinationScope() {
    var desc = new DeliveryChannelDescriptor(
            "test", "Test", true, false,
            NotificationSeverity.WARNING, null, null, DestinationScope.PER_TENANT);
    assertThat(desc.destinationScope()).isEqualTo(DestinationScope.PER_TENANT);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -Dtest=DeliveryChannelDescriptorTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: compilation failure — `DestinationScope` does not exist

- [ ] **Step 3: Create DestinationScope enum**

Use `ide_create_file` in platform:

```java
package io.casehub.platform.api.delivery;

public enum DestinationScope {
    PER_USER,
    PER_TENANT
}
```

- [ ] **Step 4: Add destinationScope field to DeliveryChannelDescriptor**

Use `ide_edit_member` on `DeliveryChannelDescriptor` (member = `DeliveryChannelDescriptor`):

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

- [ ] **Step 5: Update all existing constructor call sites in platform**

Every existing 7-arg `new DeliveryChannelDescriptor(...)` call needs `null`
(or `DestinationScope.PER_USER`) as the 8th arg. Files to update
(use `ide_replace_member` or `ide_edit_member` for each):

Production code:
- `InAppNotificationDeliverer.java:45` — register() method

Test code (all pass `null` for PER_USER default):
- `DeliveryChannelDescriptorTest.java:12` — acceptsNullGuaranteedMinSeverity
- `DeliveryChannelDescriptorTest.java:20` — carriesGuaranteedMinSeverity
- `ChannelRouterTest.java:38,44,51` — setUp()
- `ChannelRouterTest.java:207` — route_externalChannelWithDigest_markedDigested
- `ChannelRouterTest.java:229` — route_urgentSeverity_bypassesDigest
- `ChannelRouterTest.java:251` — route_internalChannel_neverDigested
- `ChannelRouterTest.java:272` — route_userPreferenceOverridesDefaultDigest
- `ChannelRouterTest.java:314` — route_populatesGuaranteedMinSeverity
- `NotificationDispatcherTest.java:75` — setUp()
- `NotificationDispatcherTest.java:233` — dispatch_channelDeliveryFailure
- `NotificationDispatcherTest.java:252` — dispatch_digestedChannel
- `NotificationDispatcherTest.java:276` — dispatch_urgentSeverity
- `DeliveryRetryProcessorTest.java:204` — emailDescriptor()
- `DigestFlushSchedulerTest.java:67` — setUp()
- `PreferenceValidatorTest.java:69` — bufferForDigest test
- `DeliveryChannelResourceTest.java:37` — setUp()

Use `ide_search_text` with query `new DeliveryChannelDescriptor(` to find
all sites, then `ide_replace_text_in_file` to append `, null)` to each
(replacing the final `)` of the 7th arg).

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -Dtest=DeliveryChannelDescriptorTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `DeliveryChannelDescriptor.java` and
`DestinationScope.java` — expect no errors.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add \
  platform-api/src/main/java/io/casehub/platform/api/delivery/DestinationScope.java \
  platform-api/src/main/java/io/casehub/platform/api/delivery/DeliveryChannelDescriptor.java \
  platform-api/src/test/java/io/casehub/platform/api/delivery/DeliveryChannelDescriptorTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(api): DestinationScope enum + DeliveryChannelDescriptor field — #90"
```

Note: the call-site updates in Step 5 affect many files — stage them
in this commit too so the build stays green.

---

### Task 2: Platform — ResolvedChannel + ChannelRouter propagation + tests

**Files:**
- Modify: `notification-dispatch/src/main/java/io/casehub/platform/notification/dispatch/ResolvedChannel.java`
- Modify: `notification-dispatch/src/main/java/io/casehub/platform/notification/dispatch/ChannelRouter.java`
- Modify: `notification-dispatch/src/test/java/io/casehub/platform/notification/dispatch/ChannelRouterTest.java`

**Interfaces:**
- Consumes: `DestinationScope` enum, `DeliveryChannelDescriptor.destinationScope()`
- Produces: `ResolvedChannel.destinationScope()` — used by dispatcher for dedup

- [ ] **Step 1: Write failing tests for ChannelRouter scope propagation**

Add to `ChannelRouterTest`:

```java
@Test
void route_propagatesDestinationScope() {
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO, null, null,
                    DestinationScope.PER_TENANT),
            new CapturingDeliverer("slack", new ArrayList<>()));

    Set<ResolvedChannel> result = channelRouter.route(
            Map.of(), noSuppression(), NotificationSeverity.INFO, null);

    assertThat(result).filteredOn(rc -> rc.channelId().equals("slack"))
            .extracting(ResolvedChannel::destinationScope)
            .containsExactly(DestinationScope.PER_TENANT);
}

@Test
void route_perTenantChannel_neverDigested() {
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO,
                    new DigestSchedule(java.time.Duration.ofHours(1)),
                    null, DestinationScope.PER_TENANT),
            new CapturingDeliverer("slack", new ArrayList<>()));

    Set<ResolvedChannel> result = channelRouter.route(
            Map.of(), noSuppression(), NotificationSeverity.INFO, null);

    assertThat(result).filteredOn(rc -> rc.channelId().equals("slack"))
            .extracting(ResolvedChannel::digested)
            .containsExactly(false);
}

@Test
void route_perTenantChannel_quietHoursBuffering_stillSuppressed() {
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO,
                    new DigestSchedule(java.time.Duration.ofHours(1)),
                    null, DestinationScope.PER_TENANT),
            new CapturingDeliverer("slack", new ArrayList<>()));

    Set<ResolvedChannel> result = channelRouter.route(
            Map.of(), quietHoursActive(), NotificationSeverity.INFO,
            QuietHoursAction.BUFFER_FOR_DIGEST);

    assertThat(result).filteredOn(rc -> rc.channelId().equals("slack"))
            .extracting(ResolvedChannel::suppressed)
            .containsExactly(true);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-dispatch -Dtest=ChannelRouterTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: compilation failure — `ResolvedChannel` doesn't have `destinationScope`

- [ ] **Step 3: Add destinationScope to ResolvedChannel**

Use `ide_edit_member` on `ResolvedChannel` (member = `ResolvedChannel`):

```java
public record ResolvedChannel(
        String channelId,
        NotificationDeliverer deliverer,
        boolean suppressed,
        boolean digested,
        NotificationSeverity guaranteedMinSeverity,
        DestinationScope destinationScope
) {
    public ResolvedChannel {
        Objects.requireNonNull(channelId, "channelId");
        Objects.requireNonNull(deliverer, "deliverer");
    }
}
```

- [ ] **Step 4: Update ChannelRouter.route() — propagate scope + digest prevention + quiet hours gate**

Use `ide_replace_member` on `ChannelRouter.route()`:

Key changes to the method body:
1. After computing `effectiveDigest`, add quiet hours buffering gate:
```java
final boolean quietHoursBuffering = suppressionResult.quietHoursActive()
        && quietHoursAction == QuietHoursAction.BUFFER_FOR_DIGEST
        && effectiveDigest != null
        && descriptor.destinationScope() != DestinationScope.PER_TENANT;
```

2. After computing `digested`, add per-tenant override:
```java
final boolean digested = descriptor.external()
        && effectiveDigest != null
        && (!severity.isAtLeast(NotificationSeverity.URGENT) || quietHoursBuffering)
        && descriptor.destinationScope() != DestinationScope.PER_TENANT;
```

3. Pass scope to `ResolvedChannel` constructor:
```java
result.add(new ResolvedChannel(channelId, deliverer, suppressed, digested,
        descriptor.guaranteedMinSeverity(), descriptor.destinationScope()));
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-dispatch -Dtest=ChannelRouterTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `ChannelRouter.java` and `ResolvedChannel.java`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add \
  notification-dispatch/src/main/java/io/casehub/platform/notification/dispatch/ResolvedChannel.java \
  notification-dispatch/src/main/java/io/casehub/platform/notification/dispatch/ChannelRouter.java \
  notification-dispatch/src/test/java/io/casehub/platform/notification/dispatch/ChannelRouterTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(dispatch): ResolvedChannel scope propagation + digest/quiet-hours gate — #90"
```

---

### Task 3: Platform — NotificationDispatcher dedup logic + tests

**Files:**
- Modify: `notification-dispatch/src/main/java/io/casehub/platform/notification/dispatch/NotificationDispatcher.java`
- Modify: `notification-dispatch/src/test/java/io/casehub/platform/notification/dispatch/NotificationDispatcherTest.java`

**Interfaces:**
- Consumes: `ResolvedChannel.destinationScope()`, `DestinationScope.PER_TENANT`,
  `DeliveryResult`

- [ ] **Step 1: Write failing tests**

Add to `NotificationDispatcherTest`:

```java
@Test
void dispatch_perTenantChannel_deliversOnceForMultipleRecipients() {
    // Register a PER_TENANT channel
    var captured = new ArrayList<NotificationInput>();
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO, null, null,
                    DestinationScope.PER_TENANT),
            new CapturingDeliverer("slack", captured));

    // 3 users targeted
    var targets = List.of(
            new NotificationTarget(NotificationTarget.Type.USER, "user1"),
            new NotificationTarget(NotificationTarget.Type.USER, "user2"),
            new NotificationTarget(NotificationTarget.Type.USER, "user3"));
    dispatcher.onMatch(new SubscriptionMatched(subscription(targets, true),
            new TestEvent("E1", "actor1", "open")));

    // deliver() called exactly once
    assertThat(captured).hasSize(1);
}

@Test
void dispatch_perTenantChannel_mixedWithPerUser_bothWork() {
    var slackCaptured = new ArrayList<NotificationInput>();
    var emailCaptured = new ArrayList<NotificationInput>();
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO, null, null,
                    DestinationScope.PER_TENANT),
            new CapturingDeliverer("slack", slackCaptured));
    channelRegistry.register(
            new DeliveryChannelDescriptor("email", "Email", true, true,
                    NotificationSeverity.INFO, null, null, null),
            new CapturingDeliverer("email", emailCaptured));

    var targets = List.of(
            new NotificationTarget(NotificationTarget.Type.USER, "user1"),
            new NotificationTarget(NotificationTarget.Type.USER, "user2"));
    dispatcher.onMatch(new SubscriptionMatched(subscription(targets, true),
            new TestEvent("E1", "actor1", "open")));

    assertThat(slackCaptured).hasSize(1);
    assertThat(emailCaptured).hasSize(2);
}

@Test
void dispatch_perTenantChannel_deliveryFailure_recordsFailureForAllUsers() {
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO, null,
                    NotificationSeverity.WARNING,
                    DestinationScope.PER_TENANT),
            new FailingDeliverer("slack"));

    var targets = List.of(
            new NotificationTarget(NotificationTarget.Type.USER, "user1"),
            new NotificationTarget(NotificationTarget.Type.USER, "user2"));
    dispatcher.onMatch(new SubscriptionMatched(subscription(targets, true),
            new TestEvent("E1", "actor1", "open")));

    // First user: RETRYING (has guaranteedMinSeverity)
    // Second user: FAILED (propagated, null guaranteedMinSeverity)
    assertThat(deliveryTracker.records()).hasSize(2);
}

@Test
void dispatch_perTenantChannel_suppressedUserSkipped_nextUserDelivers() {
    var captured = new ArrayList<NotificationInput>();
    channelRegistry.register(
            new DeliveryChannelDescriptor("slack", "Slack", true, true,
                    NotificationSeverity.INFO, null, null,
                    DestinationScope.PER_TENANT),
            new CapturingDeliverer("slack", captured));

    // User1 has channel snoozed — should skip, not populate dedup map
    suppressionStore.snooze = new Snooze("s1", "user1", TENANT,
            NOW.minusSeconds(60), NOW.plusSeconds(3600));

    var targets = List.of(
            new NotificationTarget(NotificationTarget.Type.USER, "user1"),
            new NotificationTarget(NotificationTarget.Type.USER, "user2"));
    dispatcher.onMatch(new SubscriptionMatched(subscription(targets, true),
            new TestEvent("E1", "actor1", "open")));

    // user1 suppressed, user2 delivers — exactly 1 delivery
    assertThat(captured).hasSize(1);
    assertThat(captured.getFirst().userId()).isEqualTo("user2");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-dispatch -Dtest=NotificationDispatcherTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL — dedup not implemented, slack delivers N times

- [ ] **Step 3: Modify NotificationDispatcher.onMatch() — add perTenantResults map**

Use `ide_replace_member` on `NotificationDispatcher.onMatch()`. Add
`Map<String, DeliveryResult> perTenantResults = new HashMap<>()` before
the per-user loop, pass it to `dispatchToUser()`.

- [ ] **Step 4: Modify NotificationDispatcher.dispatchToUser() — add dedup logic**

Use `ide_replace_member` on `dispatchToUser()`. Add dedup check after
digest/suppress routing, before delivery. Full implementation per the
spec's pseudocode (lines 127-185 of the reviewed spec).

Key additions:
1. Add `Map<String, DeliveryResult> perTenantResults` parameter
2. Before delivery: check `perTenantResults.get(channel.channelId())`
3. After delivery: `perTenantResults.put(channel.channelId(), result)`
4. After exception: `perTenantResults.put(channel.channelId(), failedResult)`
5. Add `LOG.debugf` for dedup skip path

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-dispatch -Dtest=NotificationDispatcherTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 6: Run full platform test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS — no regressions

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `NotificationDispatcher.java`.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add \
  notification-dispatch/src/main/java/io/casehub/platform/notification/dispatch/NotificationDispatcher.java \
  notification-dispatch/src/test/java/io/casehub/platform/notification/dispatch/NotificationDispatcherTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(dispatch): per-tenant destination deduplication in NotificationDispatcher — #90"
```

---

### Task 4: Platform build + install to local repo

**Files:** None — build only

- [ ] **Step 1: Install platform to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: BUILD SUCCESS

This makes the updated `casehub-platform-api` SNAPSHOT available for
connectors to compile against.

---

### Task 5: Connectors — bridge scope + connector opt-in + tests

**Files:**
- Modify: `notification-bridge/src/main/java/io/casehub/connectors/notification/NotificationBridgeStartup.java`
- Modify: `core/src/main/java/io/casehub/connectors/slack/SlackConnector.java`
- Modify: `core/src/main/java/io/casehub/connectors/teams/TeamsConnector.java`
- Modify: `notification-bridge/src/test/java/io/casehub/connectors/notification/NotificationBridgeStartupTest.java`

**Interfaces:**
- Consumes: `DestinationScope.PER_TENANT`, `DeliveryChannelDescriptor` 8-arg constructor

- [ ] **Step 1: Write failing tests for bridge scope registration**

Add to `NotificationBridgeStartupTest`:

```java
@Test
void slackConnector_registersWithPerTenantScope() {
    // Verify the registered descriptor has PER_TENANT scope
    var descriptor = channelRegistry.resolve("slack");
    assertThat(descriptor).isPresent();
    assertThat(descriptor.get().destinationScope())
            .isEqualTo(DestinationScope.PER_TENANT);
}

@Test
void teamsConnector_registersWithPerTenantScope() {
    var descriptor = channelRegistry.resolve("teams");
    assertThat(descriptor).isPresent();
    assertThat(descriptor.get().destinationScope())
            .isEqualTo(DestinationScope.PER_TENANT);
}

@Test
void emailConnector_registersWithPerUserScope() {
    var descriptor = channelRegistry.resolve("email");
    assertThat(descriptor).isPresent();
    assertThat(descriptor.get().destinationScope())
            .isEqualTo(DestinationScope.PER_USER);
}
```

Also add tests for connector opt-in (channelType no longer null):

```java
@Test
void slackConnector_channelType_returnsSlack() {
    assertThat(new SlackConnector().channelType()).isEqualTo("slack");
}

@Test
void teamsConnector_channelType_returnsTeams() {
    assertThat(new TeamsConnector().channelType()).isEqualTo("teams");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dtest=NotificationBridgeStartupTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — Slack/Teams not registered (channelType returns null)

- [ ] **Step 3: Remove channelType() → null from SlackConnector**

Use `ide_refactor_safe_delete` or `ide_edit_member` to remove the
`channelType()` override from `SlackConnector`. The method falls back
to `Connector.default` which returns `id()` → `"slack"`.

- [ ] **Step 4: Remove channelType() → null from TeamsConnector**

Same as Step 3 for `TeamsConnector` — remove `channelType()` override.

- [ ] **Step 5: Add PER_TENANT_CHANNELS to NotificationBridgeStartup**

Use `ide_insert_member` to add the field:

```java
private static final Set<String> PER_TENANT_CHANNELS = Set.of("slack", "teams");
```

- [ ] **Step 6: Update descriptor construction in registerBridgedChannels()**

Use `ide_replace_member` on `registerBridgedChannels()`. Change the
`DeliveryChannelDescriptor` constructor call to pass the 8th arg:

```java
var descriptor = new DeliveryChannelDescriptor(
        channelType,
        DISPLAY_NAMES.getOrDefault(channelType, channelType),
        true,
        false,
        NotificationSeverity.WARNING,
        null,
        RETRY_POLICIES.get(channelType),
        PER_TENANT_CHANNELS.contains(channelType)
                ? DestinationScope.PER_TENANT
                : DestinationScope.PER_USER);
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge,core -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 8: Run full connectors test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on modified files.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  core/src/main/java/io/casehub/connectors/slack/SlackConnector.java \
  core/src/main/java/io/casehub/connectors/teams/TeamsConnector.java \
  notification-bridge/src/main/java/io/casehub/connectors/notification/NotificationBridgeStartup.java \
  notification-bridge/src/test/java/io/casehub/connectors/notification/NotificationBridgeStartupTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(bridge): per-tenant scope for Slack/Teams + enable notification bridging — #90"
```
