# Slack ChannelBackend — Design Spec

**Date:** 2026-06-06 (revised post-review)
**Issue:** casehubio/connectors#2
**Cross-repo:** casehubio/qhorus (new module)
**Branch:** issue-2-slack-channel-backend

---

## Context

The Qhorus `ChannelBackend` SPI defines how external systems participate in Qhorus channels as human participants or agent endpoints. The generic `ConnectorChannelBackend` (in `qhorus/connector-backend`) already bridges Slack ↔ Qhorus via `SlackConnector` (outbound webhook) and `SlackInboundConnector` (inbound Events API). That bridge works but is deliberately generic — it does not understand Slack-native semantics.

This spec covers a dedicated `SlackChannelBackend` that brings three capabilities the generic bridge cannot provide:

1. **Thread support** — agent replies are posted into the Slack thread started by the human's message, not to the channel root
2. **Typed slash-command-pattern messages** — messages starting with `/` are classified as `COMMAND` type in Qhorus rather than `QUERY`
3. **Workspace-level bot token** — one `xoxb-…` token covers all Slack channels; no per-channel webhook URL configuration

The existing `SlackConnector` (Incoming Webhooks) is **not deprecated**. It serves fire-and-forget notification use cases (work escalation alerts, status pings). The new backend serves bidirectional conversation channels.

---

## Clarifications on What V1 Actually Delivers

### Thread support — approximate, not ledger-correlated

V1 implements *approximate threading*: the most-recent inbound `slack-ts` per channel is cached, and every outbound post to that channel uses it as `thread_ts`. This is correct for linear Q/A conversations (the dominant use case) and breaks only for concurrent multi-threaded channels.

Precise `inReplyTo` → Slack `ts` correlation (mapping a Qhorus ledger message ID to the specific Slack message that prompted it) is infeasible in v1: `MessageReceivedEvent` does not carry a ledger message ID, and the `OutboundMessage` carries only a delivery UUID, not the ledger ID. This is a known v1.5 design item that requires a `MessageReceivedEvent` API change or an alternative approach.

### Slash commands — Events API text pattern only

V1 detects text that *starts with `/`* in regular Events API message events and classifies it as `COMMAND`. This covers users who type `/approve` or `/escalate` as a message in a Slack channel.

Native Slack registered slash commands (which Slack intercepts before the Events API and POSTs as URL-encoded form data to a separate endpoint) are **not** supported in v1. They require a dedicated `WebhookInboundConnector` implementation and are deferred.

---

## Out of Scope (V1)

- Native Slack slash commands (see above)
- Interactive Block Kit elements (button clicks, modal submissions) — needs a separate action-callback handler
- Multi-workspace OAuth bot installation flow — v1 assumes one manually-provisioned bot token
- Rich Block Kit outbound formatting — v1 uses type-prefixed plain text; Block Kit is a v2 concern
- Precise ledger-ID ↔ Slack-ts thread correlation — deferred; requires `MessageReceivedEvent` API change
- Quarkiverse `quarkus-slack` revival — noted as a future contribution once this implementation is proven

---

## Open-Source Alternatives Evaluated

No viable lift exists. See [connectors#2 comment](https://github.com/casehubio/connectors/issues/2#issuecomment-4639585964) for full diligence. Summary: `slack-api-client` pulls in OkHttp + Kotlin stdlib with known Quarkus BOM conflicts; all community alternatives are abandoned or wrong-fit; `quarkus-slack` is stale at 0.0.5. Rolling our own `SlackBotClient` (~250 lines, `java.net.http`) is the correct call.

---

## Architecture

### Two-repo split

**casehub-connectors (connectors#2):**
1. New submodule `casehub-connectors-slack-bot` — `SlackBotClient` (pure HTTP client for Slack Web API). New submodule rather than adding to `core/` avoids future breaking change when the bot client grows beyond `chat.postMessage` (Block Kit, reactions, file uploads). Precedent: `email-inbound/` for heavier inbound concerns.
2. `InboundConnectorIds` (connectors-core) — add `SLACK_INBOUND = "slack-inbound"` constant. Update `SlackInboundConnector` to reference it (same pattern as `WhatsAppInboundConnector`, `TwilioSmsInboundConnector`). `qhorus-slack-channel` references this constant — no hardcoded strings.
3. `SlackInboundConnector.parseMessages()` — add `"slack-ts"` and `"slack-thread-ts"` fields to emitted `InboundMessage.metadata()`. Without this, thread support ships as dead code.

**casehub-qhorus (new issue):**
New module `casehub-qhorus-slack-channel` — `SlackBotBinding`, `SlackBotBindingStore`, `SlackThreadCache`, `SlackInboundNormaliser`, `SlackChannelBackend`, admin provisioning endpoint.

### Dependency direction

```
connectors-core / connectors-slack-bot  (SlackBotClient, SlackInboundConnector)
      ↑
qhorus-slack-channel  (SlackChannelBackend — depends on connectors-slack-bot + qhorus-api + qhorus-runtime)
```

Same pattern as the existing `connector-backend` module. No new dependency cycles.

### No new inbound connector

The existing `SlackInboundConnector` handles the Slack Events API for both generic connector channels and Slack Bot channels. `SlackChannelBackend` discriminates at the CDI routing layer via `SlackBotBindingStore`.

---

## Module Structure

```
casehub-connectors/
  slack-bot/                             ← NEW submodule
    pom.xml
    src/main/java/io/casehub/connectors/slack/bot/
      SlackBotClient.java
  webhook/
    SlackInboundConnector.java           ← MODIFIED: add slack-ts, slack-thread-ts to metadata

casehub-qhorus/
  slack-channel/                         ← NEW module
    pom.xml
    src/main/java/io/casehub/qhorus/slack/
      SlackBotBinding.java
      SlackBotBindingStore.java
      SlackThreadCache.java
      SlackInboundNormaliser.java
      SlackChannelBackend.java
      SlackBindingResource.java          ← admin provisioning REST endpoint
    src/main/resources/db/slack-channel/migration/
      V1__slack_bot_binding.sql
```

---

## Class Design

### `SlackBotClient` (connectors-slack-bot)

```java
@ApplicationScoped
public class SlackBotClient {

    @ConfigProperty(name = "casehub.connectors.slack-bot.api-base-url",
                    defaultValue = "https://slack.com")
    String apiBaseUrl;

    /** Posts a message. threadTs null = new message; non-null = thread reply. */
    public PostResult postMessage(String token, String channelId,
                                  String text, String threadTs);

    public record PostResult(boolean ok, String ts, String error) {}
}
```

- JSON built manually; `Authorization: Bearer {token}` header
- On HTTP 429: reads `Retry-After`, sleeps that many seconds, retries once. Sleep is unbounded (safe on virtual threads — no carrier-thread starvation). Slack's `Retry-After` is typically 1s; cap at 60s if rate-limit events become frequent in practice.
- `apiBaseUrl` injected via config — tests point it at WireMock

### `SlackInboundConnector` change (connectors/webhook) — mandatory for threading

Current `parseMessages()` emits only `"workspace-id"`. Revised:

```java
final String slackTs    = event.getString("ts", null);
final String threadTs   = event.getString("thread_ts", null);

final Map<String, String> meta = new HashMap<>();
if (teamId   != null) meta.put("workspace-id",      teamId);
if (slackTs  != null) meta.put("slack-ts",           slackTs);
if (threadTs != null) meta.put("slack-thread-ts",    threadTs);
```

Without this change, `SlackInboundNormaliser` always receives null `thread_ts`, thread support ships as dead code, and the normaliser can never resolve `inReplyTo`.

### `SlackBotBinding` entity (qhorus-slack-channel)

```java
@Entity
@Table(name = "slack_bot_binding",
       uniqueConstraints = {
           @UniqueConstraint(name = "uq_slack_bot_channel_id",  columnNames = "channel_id"),
           @UniqueConstraint(name = "uq_slack_bot_slack_id",    columnNames = "slack_channel_id")
       })
public class SlackBotBinding {
    @Id @Column(name = "channel_id")                        public UUID   channelId;
    @Column(name = "slack_channel_id", nullable = false)    public String slackChannelId;   // "C123ABC"
    @Column(name = "workspace_id",     nullable = false)    public String workspaceId;      // "T456DEF"
}
```

`workspaceId` is stored now to support per-workspace tokens in v1.5 (see Config section) without a schema migration.

### `SlackBotBindingStore` (qhorus-slack-channel)

```java
@ApplicationScoped
public class SlackBotBindingStore {
    List<SlackBotBinding>     findAll();
    Optional<SlackBotBinding> findByChannelId(UUID channelId);
    Optional<SlackBotBinding> findBySlackChannelId(String slackChannelId);
    void                      save(SlackBotBinding binding);
    void                      deleteByChannelId(UUID channelId);
}
```

### `SlackThreadCache` (CDI, in-memory, qhorus-slack-channel)

Approximate threading: cache most-recent inbound `slack-ts` per channel. Bounded map, 24-hour TTL. Lost on restart — threads break only for pre-restart messages, which is a known v1 limitation.

Slack `ts` values are globally unique per workspace (Unix epoch with microsecond precision), so no additional scoping is needed.

```java
@ApplicationScoped
class SlackThreadCache {
    /** Called on inbound: records the most recent Slack ts for this channel. */
    void recordInbound(UUID channelId, String slackTs);

    /** Returns the most recent inbound slack-ts for a channel. Used as thread_ts on outbound. */
    Optional<String> getLatestSlackTs(UUID channelId);
}
```

This gives: agent replies go into the thread started by the most recent human message. Correct for linear Q/A (dominant case). Breaks for concurrent multi-threaded channels (rare, documented v1 limitation).

### `SlackInboundNormaliser` (qhorus-slack-channel)

The `InboundNormaliser` SPI signature (verified against source):
```java
NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw);
```

`NormalisedMessage` (verified, 7 fields in order): `type, content, senderInstanceId, correlationId, inReplyTo, artefactRefs, target`.

The normaliser is a pure function in v1 — no injected state, no dependencies. `inReplyTo` is always null until the v1.5 design is resolved (see Known Limitations). A singleton instance returned from `normaliser()` is correct.

```java
class SlackInboundNormaliser implements InboundNormaliser {

    @Override
    public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
        String content = raw.content();

        // slash-command-pattern text → COMMAND; everything else → QUERY
        MessageType type = content.startsWith("/") ? MessageType.COMMAND : MessageType.QUERY;

        // Precise inReplyTo (Qhorus ledger ID ↔ Slack ts) deferred to v1.5 — see Known Limitations.
        Long inReplyTo = null;

        return new NormalisedMessage(
                type,
                content,
                "human:" + raw.externalSenderId(),  // required format for ActorTypeResolver
                null,       // correlationId
                inReplyTo,  // Long — null in v1
                null,       // artefactRefs
                null        // target
        );
    }
}
```

Note: `inReplyTo` in `NormalisedMessage` is always null in v1. Thread *replies* still post into the correct Slack thread (driven by `SlackThreadCache` in `post()`); Qhorus just doesn't know which message it's replying to. This is the honest v1 state.

### `SlackChannelBackend` (qhorus-slack-channel)

```java
@ApplicationScoped
public class SlackChannelBackend implements HumanParticipatingChannelBackend {

    static final String BACKEND_ID = "slack-bot";

    @ConfigProperty(name = "casehub.qhorus.slack.bot.token", defaultValue = "")
    String botToken;

    // injected: ChannelGateway, SlackBotBindingStore, SlackBotClient, SlackThreadCache,
    //           MeterRegistry (for Micrometer instrumentation)

    private final ConcurrentHashMap<UUID, CacheEntry> cache = new ConcurrentHashMap<>();
    private final SlackInboundNormaliser normaliser;   // pure function, singleton

    @Override public String backendId()    { return BACKEND_ID; }
    @Override public ActorType actorType() { return ActorType.HUMAN; }
    @Override public void open(ChannelRef channel, Map<String, String> metadata) { /* no-op */ }
    @Override public void close(ChannelRef channel) { cache.remove(channel.id()); }
    @Override public InboundNormaliser normaliser()  { return normaliser; }

    public void onChannelInitialised(@Observes ChannelInitialisedEvent event) {
        bindingStore.findByChannelId(event.channelId()).ifPresent(binding -> {
            // channelName from ChannelInitialisedEvent — not stored in SlackBotBinding
            cache.put(event.channelId(),
                      new CacheEntry(binding.slackChannelId, binding.workspaceId,
                                     event.channelName()));
            gateway.deregisterBackend(event.channelId(), BACKEND_ID);
            gateway.registerBackend(event.channelId(), this, "human_participating");
        });
    }

    public CompletionStage<Void> onInboundMessage(@ObservesAsync InboundMessage msg) {
        if (!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())) return done();
        bindingStore.findBySlackChannelId(msg.externalChannelRef()).ifPresentOrElse(
            binding -> {
                // Only record top-level messages as thread roots.
                // Thread replies have slack-thread-ts set; recording their ts would
                // cause subsequent outbound posts to use the reply's ts rather than
                // the original thread root, breaking multi-turn conversations.
                String slackTs  = msg.metadata().get("slack-ts");
                String threadTs = msg.metadata().get("slack-thread-ts");
                if (slackTs != null && threadTs == null) {
                    threadCache.recordInbound(binding.channelId, slackTs);
                }

                CacheEntry entry = cache.get(binding.channelId);
                String channelName = entry != null ? entry.channelName()
                                                   : binding.channelId.toString();
                gateway.receiveHumanMessage(
                        new ChannelRef(binding.channelId, channelName),
                        new InboundHumanMessage(
                                msg.externalSenderId(),
                                msg.content(),
                                msg.receivedAt(),
                                msg.metadata(),
                                null, null));
                meterRegistry.counter("slack_bot_inbound_routed_total").increment();
            },
            () -> meterRegistry.counter("slack_bot_inbound_discarded_total").increment()
        );
        return done();
    }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        CacheEntry entry = cache.get(channel.id());
        if (entry == null) {
            LOG.errorf("No cache entry for channel %s — cannot post", channel.id());
            meterRegistry.counter("slack_bot_post_failed_total", "reason", "no_cache").increment();
            return;
        }
        if (botToken.isBlank()) {
            LOG.error("casehub.qhorus.slack.bot.token not configured");
            meterRegistry.counter("slack_bot_post_failed_total", "reason", "no_token").increment();
            return;
        }

        // approximate thread: reply in most-recent top-level human message's thread
        Optional<String> latestTs = threadCache.getLatestSlackTs(channel.id());
        if (latestTs.isEmpty()) {
            meterRegistry.counter("slack_bot_thread_miss_total").increment();
        }
        String threadTs = latestTs.orElse(null);
        String text = "[" + message.type().name() + "] " + message.content();  // v1 plain text

        SlackBotClient.PostResult result =
                client.postMessage(botToken, entry.slackChannelId(), text, threadTs);

        if (!result.ok()) {
            LOG.errorf("chat.postMessage failed for channel %s: %s", channel.id(), result.error());
            meterRegistry.counter("slack_bot_post_failed_total", "reason", "api_error").increment();
        }
    }

    private record CacheEntry(String slackChannelId, String workspaceId, String channelName) {}
}
```

### `SlackBindingResource` — provisioning (qhorus-slack-channel)

Admin REST endpoint for CRUD operations on bindings.

**Auth:** `@RolesAllowed("admin")` — following the platform pattern where `CurrentPrincipal` groups wire directly to `@RolesAllowed`. This is the **first admin-protected REST endpoint in qhorus**; no precedent exists in qhorus itself. The role name `"admin"` must be confirmed as a platform convention (claudony uses `@Authenticated` for user endpoints and `@RolesAllowed("fleet")` for peer endpoints; the `"admin"` group name for administrative operations is new and needs to be established). Implementer must confirm the role name before shipping. The `@RolesAllowed` mechanism itself is correct.

```
GET    /api/admin/slack/bindings
       Response: 200 [{qhorusChannelId, slackChannelId, workspaceId}, ...]

GET    /api/admin/slack/bindings/{qhorusChannelId}
       Response: 200 {qhorusChannelId, slackChannelId, workspaceId} | 404

POST   /api/admin/slack/bindings
       Body: {"qhorusChannelId":"…", "slackChannelId":"C123ABC", "workspaceId":"T456DEF"}
       Response: 201 Created | 404 (channel not found) | 409 Conflict (binding already exists)

DELETE /api/admin/slack/bindings/{qhorusChannelId}
       Response: 204 No Content  (idempotent — 204 even if nothing was bound)
```

**POST semantics:** Re-binding is an intentional but infrequent operation. The endpoint returns **409 Conflict** if a binding already exists, requiring the admin to DELETE first. This makes rebinding explicit rather than silently overwriting a live channel binding. Upsert semantics would risk accidental overwrite with no audit trail.

**DELETE semantics:** Idempotent. All three cleanup operations (`deleteByChannelId`, `deregisterBackend`, `close`) are no-ops on a non-existent entry, so 204 is always correct. No 404.

**Handlers:**

```java
// injected: ChannelGateway gateway, SlackBotBindingStore bindingStore,
//           ChannelService channelService, SlackChannelBackend slackChannelBackend

@POST
public Response create(SlackBindingRequest request) {
    // 409 if already bound — rebinding requires DELETE then POST
    if (bindingStore.findByChannelId(request.qhorusChannelId()).isPresent()) {
        return Response.status(409)
            .entity("Binding already exists for this channel. DELETE it first to rebind.")
            .build();
    }

    Channel channel = channelService.findById(request.qhorusChannelId())
        .orElseThrow(() -> new NotFoundException("Channel not found"));

    bindingStore.save(new SlackBotBinding(
        channel.id, request.slackChannelId(), request.workspaceId()));

    // Re-fire ChannelInitialisedEvent so SlackChannelBackend registers immediately.
    // Without this, a post-startup binding has no effect until the next restart.
    gateway.initChannel(channel.id, new ChannelRef(channel.id, channel.name));

    return Response.status(201).build();
}

@DELETE
@Path("/{qhorusChannelId}")
public Response delete(@PathParam("qhorusChannelId") UUID qhorusChannelId) {
    bindingStore.deleteByChannelId(qhorusChannelId);

    // Remove from gateway fanOut registry and clear local backend cache.
    // Without this, SlackChannelBackend continues posting to the old Slack channel
    // until the next restart. close() uses only channel.id() — name is unused.
    gateway.deregisterBackend(qhorusChannelId, SlackChannelBackend.BACKEND_ID);
    slackChannelBackend.close(new ChannelRef(qhorusChannelId, ""));

    return Response.noContent().build();
}
```

---

## Data Flows

### Registration (startup)

```
Quarkus start → ChannelGateway.initChannel() per persisted channel
  → ChannelInitialisedEvent(recovered=true)
  → SlackChannelBackend: SlackBotBindingStore.findByChannelId() → found → cache + register
  → ConnectorChannelBackend: ChannelConnectorBinding.findByChannelId() → not found → skip
```

No conflict. Different binding stores, cannot collide.

### Inbound (human types in Slack)

```
Human: "/approve case-123" in Slack channel C123ABC (ts="T1", no thread_ts)
  → POST /connectors/slack-inbound/webhook
  → SlackInboundConnector: HMAC verify → parse → fireAsync(InboundMessage(
        connectorId="slack-inbound",
        externalChannelRef="C123ABC",
        externalSenderId="U789DEF",
        content="/approve case-123",
        metadata={"slack-ts":"T1", "workspace-id":"T456GHI"}))

CDI async observers:
  ConnectorChannelBackend: lookup ("slack-inbound","C123ABC") → not found → DEBUG log
  SlackChannelBackend: lookup by slackChannelId="C123ABC" → found
    → threadCache.recordInbound(channelId, "T1")
    → gateway.receiveHumanMessage(channelRef, InboundHumanMessage)
    → SlackInboundNormaliser.normalise():
        starts with "/" → type=COMMAND
        inReplyTo = null (v1)
        return NormalisedMessage(COMMAND, "/approve case-123", "human:U789DEF", null, null, null, null)
    → messageService.dispatch(COMMAND, ...)
```

### Outbound (agent replies)

```
Agent dispatches RESPONSE
  → fanOut(OutboundMessage(...))
  → SlackChannelBackend.post():
      cache hit: slackChannelId="C123ABC"
      threadTs = threadCache.getLatestSlackTs(channelId) → "T1"
      text = "[RESPONSE] Approved."
      client.postMessage(token, "C123ABC", "[RESPONSE] Approved.", threadTs="T1")
      → POST https://slack.com/api/chat.postMessage
      → {"ok":true,"ts":"T2"}
```

Message appears as a threaded reply in Slack under the human's `/approve` message.

---

## Configuration

```properties
# casehub-connectors (slack-bot submodule)
casehub.connectors.slack-bot.api-base-url=https://slack.com   # override to WireMock in tests

# casehub-qhorus-slack-channel
casehub.qhorus.slack.bot.token=xoxb-...
```

**Credential management:** `xoxb-…` tokens must not live in plaintext config files committed to version control. In deployment, inject via environment variable (`CASEHUB_QHORUS_SLACK_BOT_TOKEN`) or, for production, via the Quarkus Vault extension (`io.quarkus:quarkus-vault`) pointing at HashiCorp Vault. The MP Config `casehub.qhorus.slack.bot.token` key works with both approaches — the injection source is an operational concern.

**V1.5 multi-workspace token plan:** replace the single token with a `@ConfigMapping` map keyed by workspace ID:

```properties
casehub.qhorus.slack.bot.tokens.T456DEF=xoxb-workspace-one
casehub.qhorus.slack.bot.tokens.T789GHI=xoxb-workspace-two
```

`SlackChannelBackend.post()` looks up `tokens.get(entry.workspaceId())`. `SlackBotBinding.workspaceId` is stored now precisely so v1.5 requires no schema migration.

---

## Micrometer Instrumentation

`SlackChannelBackend` records counters consistent with `ConnectorChannelBackend`'s instrumentation approach:

| Counter | Tags | When incremented |
|---|---|---|
| `slack_bot_post_failed_total` | `reason=no_cache\|no_token\|api_error` | `post()` cannot deliver |
| `slack_bot_thread_miss_total` | — | `getLatestSlackTs()` returns empty on outbound |
| `slack_bot_inbound_routed_total` | — | `onInboundMessage()` successfully routes to gateway |
| `slack_bot_inbound_discarded_total` | — | `onInboundMessage()` has no binding for slack channel ID |

---

## Testing

| Layer | Tool | Coverage |
|---|---|---|
| `SlackBotClient` | WireMock (already in connectors-core) | JSON shape of `chat.postMessage`; `Authorization` header present; `thread_ts` included when non-null, absent when null; 429 → single `Retry-After` retry; non-ok response → `PostResult(false,…)` |
| `SlackInboundConnector` (modified) | Existing unit test pattern (signed requests) | `metadata` contains `slack-ts` from `event.ts`; `slack-thread-ts` present when `event.thread_ts` present; absent when not in event |
| `SlackInboundNormaliser` | Pure unit test | `/command` → COMMAND; plain text → QUERY; `senderInstanceId` = `"human:U456DEF"` (correct prefix) |
| `SlackChannelBackend` registration | `@QuarkusTest`, in-memory binding store | `ChannelInitialisedEvent` → registered for bound channels; unbound channels skipped; deregister-before-register idempotent |
| `SlackChannelBackend` inbound routing | `@QuarkusTest` | `InboundMessage` with matching `externalChannelRef` → `receiveHumanMessage()` called with correct `ChannelRef.name()`; non-matching → `inbound_discarded` counter; `threadCache.recordInbound()` called for top-level messages (no `slack-thread-ts`); NOT called for thread replies (has `slack-thread-ts`) |
| `SlackChannelBackend` outbound | `@QuarkusTest` + WireMock | `post()` with cached slackTs → `thread_ts` field in request body; `post()` with empty cache → no `thread_ts` field; blank token → no HTTP call, error logged, counter incremented |
| **End-to-end thread round-trip** | `@QuarkusTest` + WireMock | Inbound message with `slack-ts="T1"` → `recordInbound()` called → outbound `post()` → WireMock captures request, asserts `thread_ts="T1"`. Validates the complete path: metadata field → cache → outbound payload. |
| `SlackBindingResource` | `@QuarkusTest` + REST Assured | GET /bindings returns all; GET /bindings/{id} returns one or 404; POST → 201 and backend registered in gateway; POST duplicate → 409; DELETE → 204 always (idempotent); DELETE calls `deregisterBackend` + `close()` — verify backend no longer receives fanOut after delete |

---

## Cross-Repo Work

### casehub-connectors (connectors#2 — this branch)

1. New `casehub-connectors-slack-bot` Maven submodule containing `SlackBotClient`
2. `InboundConnectorIds` (connectors-core) — add `SLACK_INBOUND = "slack-inbound"`. Update `SlackInboundConnector.ID` to reference it.
3. `SlackInboundConnector.parseMessages()` — add `"slack-ts"` and `"slack-thread-ts"` to `InboundMessage.metadata()`  ← **mandatory for thread support**
4. Tests for all of the above

### casehub-qhorus (new issue — file before session close)

1. New `casehub-qhorus-slack-channel` Maven module
2. All classes above (`SlackBotBinding`, `SlackBotBindingStore`, `SlackThreadCache`, `SlackInboundNormaliser`, `SlackChannelBackend`, `SlackBindingResource`)
3. Flyway `V1__slack_bot_binding.sql`
4. **Flyway location config** — `quarkus.flyway.locations` (in the qhorus deployment config or runtime `application.properties`) must include `classpath:db/slack-channel/migration/`. Quarkus Flyway's default location is `classpath:db/migration` only; without this addition the migration never runs and the table is missing on first deploy. Check whether `connector-backend` set a precedent for adding module-scoped migration paths. **This is a deployment blocker — include in the qhorus issue scope.**
5. **`ConnectorChannelBackend.onInboundMessage()`** (in `qhorus/connector-backend/`) — WARN → DEBUG universally when no binding found. **Tradeoff:** this suppresses warnings for genuine misconfigurations too. Decision: `inbound_messages_discarded_total` counter (tagged with `connector_id`) is the canonical observability signal — alertable, dashboardable. Anyone alerting on the WARN must migrate to the counter. **Note: this file is in qhorus, not connectors — correct repo for this change.**
6. Full test suite including end-to-end thread round-trip test
7. Micrometer counters

Depends on `casehub-connectors-slack-bot` published to GitHub Packages. connectors#2 must publish first.

---

## Known Limitations (V1)

| Limitation | What V1 delivers | V1.5 path |
|---|---|---|
| Thread correlation (precise) | `inReplyTo` always null in `NormalisedMessage` — Qhorus doesn't know which message a Slack reply refers to | Requires `MessageReceivedEvent` to carry ledger ID, or alternative mechanism — design TBD |
| Thread support (outbound) | Agent replies posted in most-recent human message's thread — correct for linear conversations, incorrect for concurrent multi-threaded channels | Precise ledger-ID → slackTs mapping; see above |
| Thread cache lost on restart | In-memory only — threads break for pre-restart messages | Persist `SlackThreadCache` to DB (small table) |
| Native slash commands | Not handled — Slack intercepts before Events API | New `WebhookInboundConnector` for slash command URL |
| Rich outbound formatting | `[TYPE] content` plain text | Block Kit formatting keyed by `MessageType` |
| Single workspace | One `botToken` for all channels | `@ConfigMapping` map keyed by `workspaceId` — schema already supports it |
| Token in plaintext config | Operational risk in dev; use env var or Vault in production | Quarkus Vault extension or platform credential store |