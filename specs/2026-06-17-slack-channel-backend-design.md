# Slack ChannelBackend — Design Spec (Revised)

**Date:** 2026-06-17 (revised from 2026-06-06 post-implementation review)
**Issue:** casehubio/connectors#2 (closed, connectors-side done), casehubio/qhorus#261 (open — qhorus-side)
**Cross-repo:** casehubio/qhorus (new module)
**Branch:** issue-2-slack-channel-backend (connectors — merged), new branch in qhorus for #261

---

## What Changed Since the Original Spec

The original spec (2026-06-06) was reviewed against the implemented `ConnectorChannelBackend`,
`ChannelGateway`, and the qhorus API types. Six bugs and three design weaknesses were found:

| # | Issue | Original spec | This revision |
|---|---|---|---|
| 1 | `post()` posts `"[EVENT] null"` for EVENT messages | No null guard | `if (message.content() == null) return;` |
| 2 | `onInboundMessage()` hits DB on every Slack event | `bindingStore.findBySlackChannelId()` per message | In-memory `slackToChannel` reverse index; O(1) lookup |
| 3 | Channel deletion leaves stale `SlackBotBinding` in DB | `close()` only removes from local cache | `close()` also calls `bindingStore.deleteByChannelId()` and clears reverse index |
| 4 | `@RolesAllowed("admin")` not used anywhere in qhorus | Spec introduced it with uncertainty flag | Removed; auth is a platform concern deferred to a future issue |
| 5 | `done()` helper undefined | Referenced as `return done();` with no definition | Defined as `return CompletableFuture.completedFuture(null);` |
| 6 | `SlackThreadCache` specified "bounded map, 24h TTL" | No implementation guidance | Simplified to `ConcurrentHashMap<UUID, String>` — no TTL, no new dependency |
| 7 | `SlackChannelBackend` used field injection | Spec showed `@Inject`-annotated fields | Revised to constructor injection, matching `ConnectorChannelBackend` |
| 8 | REST resource missing `@Produces`/`@Consumes` | No JSON content annotations | Added |
| 9 | `fanOut()` threading model not documented | Silent | `post()` runs on a virtual thread spawned by `ChannelGateway.fanOut()` — blocking call is safe |

---

## Context

The Qhorus `ChannelBackend` SPI defines how external systems participate in Qhorus channels as human
participants or agent endpoints. The generic `ConnectorChannelBackend` (in `qhorus/connector-backend`)
already bridges Slack ↔ Qhorus via `SlackConnector` (outbound webhook) and `SlackInboundConnector`
(inbound Events API). That bridge is deliberately generic — it does not understand Slack-native semantics.

This spec covers a dedicated `SlackChannelBackend` that brings three capabilities the generic bridge
cannot provide:

1. **Thread support** — agent replies post into the Slack thread started by the human's message
2. **Typed slash-command-pattern messages** — messages starting with `/` are classified as `COMMAND`
3. **Workspace-level bot token** — one `xoxb-…` covers all Slack channels; no per-channel webhook URL

The existing `SlackConnector` (Incoming Webhooks) is **not deprecated**. It serves fire-and-forget
notification use cases. The new backend serves bidirectional conversation channels.

---

## Clarifications on What V1 Delivers

### Thread support — approximate, not ledger-correlated

V1 implements *approximate threading*: the most-recent inbound `slack-ts` per channel is cached, and
every outbound post to that channel uses it as `thread_ts`. Correct for linear Q/A (dominant case).
Breaks for concurrent multi-threaded channels — documented v1 limitation.

Precise correlation (`OutboundMessage.inReplyTo` Qhorus ledger ID → Slack `ts`) is infeasible in v1.
Root cause: `gateway.receiveHumanMessage()` is void — there is no way to get the ledger ID of the
just-dispatched message to build a `slackTs → ledgerId` mapping. Fixing this requires either a return
value from `receiveHumanMessage()` or a CDI event that carries the ledger ID. Deferred to v1.5.

### Slash commands — Events API text pattern only

V1 detects text starting with `/` in regular Events API message events and classifies it as `COMMAND`.
Native Slack registered slash commands (intercepted before the Events API) are **not** supported —
they require a separate `WebhookInboundConnector` and are deferred.

---

## Out of Scope (V1)

- Native Slack slash commands (see above)
- Interactive Block Kit elements (button clicks, modal submissions)
- Multi-workspace OAuth bot installation flow — v1 assumes one manually-provisioned bot token
- Rich Block Kit outbound formatting — v1 uses type-prefixed plain text; Block Kit is v2
- Precise ledger-ID ↔ Slack-ts thread correlation — deferred; requires `receiveHumanMessage()` API change
- REST endpoint authentication — qhorus has no REST auth mechanism yet; `SlackBindingResource` must be
  network-isolated in deployment. Auth is a platform concern tracked separately.

---

## Architecture

### Two-repo split

**casehub-connectors (connectors#2 — already done on main):**
1. `casehub-connectors-slack-bot` submodule — `SlackBotClient` (pure HTTP client for Slack Web API)
2. `InboundConnectorIds.SLACK_INBOUND = "slack-inbound"` constant added to core
3. `SlackInboundConnector.parseMessages()` — adds `"slack-ts"` and `"slack-thread-ts"` to metadata

**casehub-qhorus (qhorus#261 — not yet started):**
New module `casehub-qhorus-slack-channel`.

### Dependency direction

```
connectors-core / connectors-slack-bot  (SlackBotClient, SlackInboundConnector)
      ↑
qhorus-slack-channel  (SlackChannelBackend — depends on connectors-slack-bot + qhorus-api + qhorus-runtime)
```

Same pattern as the existing `connector-backend` module. No new dependency cycles.

### No new inbound connector

The existing `SlackInboundConnector` handles the Slack Events API for both generic connector channels
and Slack Bot channels. `SlackChannelBackend` discriminates at the CDI routing layer via
`SlackBotBindingStore`.

---

## Module Structure

```
casehub-connectors/
  slack-bot/                             ← DONE (merged to main)
    SlackBotClient.java
  webhook/
    SlackInboundConnector.java           ← DONE: slack-ts, slack-thread-ts in metadata

casehub-qhorus/
  slack-channel/                         ← NEW module (qhorus#261)
    pom.xml
    src/main/java/io/casehub/qhorus/slack/
      SlackBotBinding.java
      SlackBotBindingStore.java
      SlackThreadCache.java
      SlackInboundNormaliser.java
      SlackChannelBackend.java
      SlackBindingResource.java
    src/main/resources/db/slack-channel/migration/
      V1__slack_bot_binding.sql
```

---

## Class Design

### `SlackBotClient` (connectors-slack-bot) — already implemented

Refer to the implementation in `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java`.
The class is stable. Key facts for `SlackChannelBackend` authors:

- `postMessage(token, channelId, text, threadTs)` — `threadTs` null = new top-level message
- Returns `PostResult(ok, ts, error)` — `ts` is the Slack message timestamp, usable for future thread replies
- On HTTP 429: reads `Retry-After`, sleeps once, retries once — safe on virtual threads
- `apiBaseUrl` can be overridden in tests by injecting a config property pointing at WireMock

### `SlackInboundConnector` change (connectors/webhook) — already implemented

`parseMessages()` now populates `InboundMessage.metadata()` with:
- `"slack-ts"` — the Slack `ts` of the message (present on all messages)
- `"slack-thread-ts"` — the Slack `thread_ts` (present only if the message is a thread reply)

Without these fields, `SlackChannelBackend` cannot implement the thread cache correctly.

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

`workspaceId` is stored now to support per-workspace tokens in v1.5 without a schema migration.

### `SlackBotBindingStore` (qhorus-slack-channel)

```java
@ApplicationScoped
public class SlackBotBindingStore {
    List<SlackBotBinding>     findAll();
    Optional<SlackBotBinding> findByChannelId(UUID channelId);
    Optional<SlackBotBinding> findBySlackChannelId(String slackChannelId);
    @Transactional void       save(SlackBotBinding binding);
    @Transactional void       deleteByChannelId(UUID channelId);
}
```

`save()` and `deleteByChannelId()` require `@Transactional`. `find*()` methods can run without a
transaction but should be inside one when called from a transactional context.

### `SlackThreadCache` (CDI, in-memory, qhorus-slack-channel)

Approximate threading: records the most recent top-level inbound `slack-ts` per channel.

```java
@ApplicationScoped
class SlackThreadCache {

    // channelId → most-recent top-level slack-ts
    // Unbounded for v1: one entry per Slack-backed channel, cleared on close().
    private final ConcurrentHashMap<UUID, String> latestTs = new ConcurrentHashMap<>();

    void recordInbound(UUID channelId, String slackTs) {
        latestTs.put(channelId, slackTs);
    }

    Optional<String> getLatestSlackTs(UUID channelId) {
        return Optional.ofNullable(latestTs.get(channelId));
    }

    void remove(UUID channelId) {
        latestTs.remove(channelId);
    }
}
```

**No TTL, no Caffeine dependency for v1.** Entries are bounded by the number of Slack-backed channels
(not by message volume). Entries are explicitly removed in `SlackChannelBackend.close()` — the map
stays small. If the JVM restarts, the cache is empty; the first post after restart goes to the channel
root rather than a thread. This is the documented v1 restart behaviour.

### `SlackInboundNormaliser` (qhorus-slack-channel)

A pure, stateless function — no injected dependencies. Instantiated directly in `SlackChannelBackend`.

```java
class SlackInboundNormaliser implements InboundNormaliser {

    @Override
    public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
        String content = raw.content();

        // slash-command-pattern text → COMMAND; everything else → QUERY
        MessageType type = content.startsWith("/") ? MessageType.COMMAND : MessageType.QUERY;

        // inReplyTo: always null in v1.
        // Root cause: gateway.receiveHumanMessage() is void — there is no way to retrieve
        // the ledger ID of the just-dispatched message to build a slackTs → ledgerId mapping.
        // Fixing this requires a receiveHumanMessage() return value or a dispatch CDI event.
        Long inReplyTo = null;

        return new NormalisedMessage(
                type,
                content,
                "human:" + raw.externalSenderId(),  // required format for ActorTypeResolver
                null,       // correlationId
                inReplyTo,
                null,       // artefactRefs
                null        // target
        );
    }
}
```

Note: `ChannelGateway.receiveHumanMessage()` reads `raw.metadata().get("message-type")` for normaliser
telemetry. Slack metadata contains `slack-ts`, `workspace-id`, and `slack-thread-ts` — none of these
clash with `"message-type"`. The telemetry key will always be `metadata_key_used=false` for Slack bot
channels, correctly reflecting that the normaliser uses content inspection, not metadata override.

### `SlackChannelBackend` (qhorus-slack-channel)

**Threading model:** `ChannelGateway.fanOut()` spawns a virtual thread per backend before calling
`post()`. `SlackBotClient.postMessage()` blocks on an HTTP call — this is safe on a virtual thread
and requires no `@Blocking` annotation (that is a Vert.x event loop concern, not a CDI thread concern).

**Inbound lookup:** `onInboundMessage()` uses an in-memory reverse index (`slackToChannel`) populated
during `onChannelInitialised()`. No DB round trip per inbound message.

**`close()` cleanup:** clears both in-memory structures AND the DB binding. This handles channel
deletion via `delete_channel` (which calls `ChannelGateway.closeChannel()` → `backend.close()`).
Admins who use the REST `DELETE /api/admin/slack/bindings/{id}` endpoint get the same cleanup via
`SlackBindingResource.delete()` calling `bindingStore.deleteByChannelId()` and `backend.close()`.
Either path leaves a consistent state.

```java
@ApplicationScoped
public class SlackChannelBackend implements HumanParticipatingChannelBackend {

    static final String BACKEND_ID = "slack-bot";

    private static final Logger LOG = Logger.getLogger(SlackChannelBackend.class);

    private final ChannelGateway gateway;
    private final SlackBotBindingStore bindingStore;
    private final SlackBotClient client;
    private final SlackThreadCache threadCache;
    private final MeterRegistry meterRegistry;

    // Keyed by Qhorus channelId
    private final ConcurrentHashMap<UUID, CacheEntry> cache = new ConcurrentHashMap<>();
    // Reverse index: slackChannelId → Qhorus channelId; avoids DB lookup per inbound message
    private final ConcurrentHashMap<String, UUID> slackToChannel = new ConcurrentHashMap<>();

    private final SlackInboundNormaliser normaliser = new SlackInboundNormaliser();

    @ConfigProperty(name = "casehub.qhorus.slack.bot.token", defaultValue = "")
    String botToken;

    @Inject
    public SlackChannelBackend(ChannelGateway gateway,
                               SlackBotBindingStore bindingStore,
                               SlackBotClient client,
                               SlackThreadCache threadCache,
                               MeterRegistry meterRegistry) {
        this.gateway = gateway;
        this.bindingStore = bindingStore;
        this.client = client;
        this.threadCache = threadCache;
        this.meterRegistry = meterRegistry;
    }

    @Override public String backendId()    { return BACKEND_ID; }
    @Override public ActorType actorType() { return ActorType.HUMAN; }
    @Override public InboundNormaliser normaliser() { return normaliser; }
    @Override public void open(ChannelRef channel, Map<String, String> metadata) { /* no-op */ }

    @Override
    public void close(ChannelRef channel) {
        CacheEntry removed = cache.remove(channel.id());
        if (removed != null) {
            slackToChannel.remove(removed.slackChannelId());
            threadCache.remove(channel.id());
            bindingStore.deleteByChannelId(channel.id());
        }
    }

    public void onChannelInitialised(@Observes ChannelInitialisedEvent event) {
        bindingStore.findByChannelId(event.channelId()).ifPresent(binding -> {
            CacheEntry entry = new CacheEntry(binding.slackChannelId, binding.workspaceId,
                                              event.channelName());
            cache.put(event.channelId(), entry);
            slackToChannel.put(binding.slackChannelId, event.channelId());
            gateway.deregisterBackend(event.channelId(), BACKEND_ID);
            gateway.registerBackend(event.channelId(), this, "human_participating");
        });
    }

    /**
     * Returns CompletionStage<Void> so that tests using
     * Event.fireAsync().toCompletableFuture().join() reliably wait for this
     * observer to finish before asserting.
     */
    public CompletionStage<Void> onInboundMessage(@ObservesAsync InboundMessage msg) {
        if (!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())) {
            return CompletableFuture.completedFuture(null);
        }

        // O(1) reverse lookup — no DB call per message
        UUID channelId = slackToChannel.get(msg.externalChannelRef());
        if (channelId == null) {
            meterRegistry.counter("slack_bot_inbound_discarded_total").increment();
            return CompletableFuture.completedFuture(null);
        }

        // Only top-level messages are thread roots.
        // Thread replies have slack-thread-ts set; recording their ts would cause subsequent
        // outbound posts to use the reply's ts as thread root, breaking multi-turn conversations.
        String slackTs  = msg.metadata().get("slack-ts");
        String threadTs = msg.metadata().get("slack-thread-ts");
        if (slackTs != null && threadTs == null) {
            threadCache.recordInbound(channelId, slackTs);
        }

        CacheEntry entry = cache.get(channelId);
        String channelName = entry != null ? entry.channelName() : channelId.toString();
        gateway.receiveHumanMessage(
                new ChannelRef(channelId, channelName),
                new InboundHumanMessage(
                        msg.externalSenderId(),
                        msg.content(),
                        msg.receivedAt(),
                        msg.metadata(),
                        null, null));  // correlationId, inReplyTo: both null in v1
        meterRegistry.counter("slack_bot_inbound_routed_total").increment();

        return CompletableFuture.completedFuture(null);
    }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        // EVENT messages have null content — do not post to Slack.
        if (message.content() == null) return;

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

        // Approximate thread: reply in most-recent top-level human message's thread.
        // See Known Limitations — inReplyTo in OutboundMessage carries the Qhorus ledger ID,
        // not a Slack ts. Precise mapping is a v1.5 design item.
        Optional<String> latestTs = threadCache.getLatestSlackTs(channel.id());
        if (latestTs.isEmpty()) {
            meterRegistry.counter("slack_bot_thread_miss_total").increment();
        }
        String text = "[" + message.type().name() + "] " + message.content();

        SlackBotClient.PostResult result =
                client.postMessage(botToken, entry.slackChannelId(), text, latestTs.orElse(null));

        if (!result.ok()) {
            LOG.errorf("chat.postMessage failed for channel %s: %s", channel.id(), result.error());
            meterRegistry.counter("slack_bot_post_failed_total", "reason", "api_error").increment();
        }
        // result.ts() is the new message's Slack ts — not used in v1 but available for v1.5
        // precise thread correlation if we want to record outbound message ts → ledger ID mappings.
    }

    private record CacheEntry(String slackChannelId, String workspaceId, String channelName) {}
}
```

### `SlackBindingResource` — provisioning (qhorus-slack-channel)

**Auth:** qhorus currently has no REST authentication mechanism. `SlackBindingResource` is an admin
endpoint that must be network-isolated in deployment (not exposed to the public internet). Auth is a
platform concern tracked separately. No `@RolesAllowed` or security annotation is applied in v1.

```
GET    /api/admin/slack/bindings
       Produces: application/json
       Response: 200 [{qhorusChannelId, slackChannelId, workspaceId}, ...]

GET    /api/admin/slack/bindings/{qhorusChannelId}
       Response: 200 {qhorusChannelId, slackChannelId, workspaceId} | 404

POST   /api/admin/slack/bindings
       Consumes: application/json
       Body: {"qhorusChannelId":"…", "slackChannelId":"C123ABC", "workspaceId":"T456DEF"}
       Response: 201 Created | 404 (channel not found) | 409 Conflict (binding already exists)

DELETE /api/admin/slack/bindings/{qhorusChannelId}
       Response: 204 No Content  (idempotent — 204 even if no binding exists)
```

**POST semantics:** Returns 409 if a binding already exists — rebinding requires DELETE then POST.
This makes rebinding explicit rather than silently overwriting a live channel binding.

**DELETE semantics:** Idempotent. Cleans up the DB binding, deregisters the backend from the gateway,
and calls `slackChannelBackend.close()` to clear the in-memory cache and thread cache entries.

```java
@Path("/api/admin/slack/bindings")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class SlackBindingResource {

    // injected: ChannelGateway gateway, SlackBotBindingStore bindingStore,
    //           ChannelService channelService, SlackChannelBackend slackChannelBackend

    @GET
    public List<SlackBindingDto> list() {
        return bindingStore.findAll().stream().map(SlackBindingDto::from).toList();
    }

    @GET
    @Path("/{channelId}")
    public Response get(@PathParam("channelId") UUID channelId) {
        return bindingStore.findByChannelId(channelId)
                .map(b -> Response.ok(SlackBindingDto.from(b)).build())
                .orElse(Response.status(404).build());
    }

    @POST
    @Transactional
    public Response create(SlackBindingRequest request) {
        if (bindingStore.findByChannelId(request.qhorusChannelId()).isPresent()) {
            return Response.status(409)
                .entity("Binding already exists. DELETE it first to rebind.")
                .build();
        }
        Channel channel = channelService.findById(request.qhorusChannelId())
            .orElseThrow(() -> new NotFoundException("Channel not found"));

        bindingStore.save(new SlackBotBinding(channel.id, request.slackChannelId(), request.workspaceId()));

        // Re-fire ChannelInitialisedEvent (recovered=false) so SlackChannelBackend registers
        // immediately. Without this a post-startup binding has no effect until restart.
        gateway.initChannel(channel.id, new ChannelRef(channel.id, channel.name));

        return Response.status(201).build();
    }

    @DELETE
    @Path("/{channelId}")
    public Response delete(@PathParam("channelId") UUID channelId) {
        // Remove DB binding, deregister backend, clear in-memory cache.
        // All three are no-ops if no binding exists — 204 is always correct.
        bindingStore.deleteByChannelId(channelId);
        gateway.deregisterBackend(channelId, SlackChannelBackend.BACKEND_ID);
        slackChannelBackend.close(new ChannelRef(channelId, ""));
        return Response.noContent().build();
    }
}
```

Note: `close()` removes `channelId` from `cache`, clears the `slackToChannel` entry, removes from
`SlackThreadCache`, and calls `bindingStore.deleteByChannelId()`. The explicit `bindingStore.deleteByChannelId()`
call in `SlackBindingResource.delete()` is therefore a no-op after `close()` runs, but the `close()` here is
called without a binding present in the admin `DELETE` flow — the `if (removed != null)` guard in `close()`
prevents double deletion. Safe.

---

## Data Flows

### Registration (startup)

```
Quarkus start → ChannelGateway.onStart() → initChannel(channelId, ref, recovered=true) per channel
  → ChannelInitialisedEvent(recovered=true)
  → SlackChannelBackend.onChannelInitialised():
      bindingStore.findByChannelId() → found
      cache.put(channelId, entry)
      slackToChannel.put(slackChannelId, channelId)
      deregisterBackend → registerBackend("human_participating")
  → ConnectorChannelBackend.onChannelInitialised():
      bindingStore.findByChannelId() → not found (different store) → skip
```

No conflict. The two backends use different binding stores and cannot collide.
`registerBackend()` throws `DuplicateParticipatingBackendException` if two `human_participating`
backends try to register on the same channel — this cannot happen because a channel can only have
a `SlackBotBinding` or a `ChannelConnectorBinding`, never both.

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

CDI async observers (both observe the same event):
  ConnectorChannelBackend: findByConnectorKey("slack-inbound","C123ABC") → not found
    → LOG.debug (after WARN→DEBUG change) → discard counter incremented
  SlackChannelBackend: slackToChannel.get("C123ABC") → UUID (O(1) — no DB call)
    → threadCache.recordInbound(channelId, "T1")
    → gateway.receiveHumanMessage(channelRef, InboundHumanMessage)
    → SlackInboundNormaliser.normalise():
        starts with "/" → type=COMMAND
        return NormalisedMessage(COMMAND, "/approve case-123", "human:U789DEF", null, null, null, null)
    → messageService.dispatch(COMMAND, ...)
    → normaliser telemetry EVENT fired (metadata_key_used=false — no "message-type" in Slack metadata)
```

### Outbound (agent replies)

```
Agent dispatches RESPONSE
  → fanOut(OutboundMessage(type=RESPONSE, content="Approved.", inReplyTo=42L, ...))
  → ChannelGateway spawns virtual thread → SlackChannelBackend.post():
      message.content() != null → proceed
      cache hit: slackChannelId="C123ABC"
      threadTs = threadCache.getLatestSlackTs(channelId) → "T1"
      text = "[RESPONSE] Approved."
      client.postMessage(token, "C123ABC", "[RESPONSE] Approved.", "T1")
      → POST https://slack.com/api/chat.postMessage
      → {"ok":true,"ts":"T2"}
      (T2 is the reply's ts — not recorded in v1; would be needed for precise thread correlation)
```

### Agent sends EVENT message

```
Agent dispatches EVENT (content=null)
  → fanOut(OutboundMessage(type=EVENT, content=null, ...))
  → SlackChannelBackend.post():
      message.content() == null → return immediately (no Slack API call)
```

### Channel deletion

```
delete_channel tool called
  → channelGateway.closeChannel(channelId, ref)
  → SlackChannelBackend.close(ref):
      removed = cache.remove(channelId)              // evict cache entry
      slackToChannel.remove(removed.slackChannelId)  // evict reverse index
      threadCache.remove(channelId)                  // evict thread cache
      bindingStore.deleteByChannelId(channelId)      // remove DB record
  → Slack binding fully cleaned up — no manual admin action required
```

---

## Configuration

```properties
# casehub-connectors (slack-bot submodule) — already in application.properties
casehub.connectors.slack-bot.api-base-url=https://slack.com   # override to WireMock in tests

# casehub-qhorus-slack-channel
casehub.qhorus.slack.bot.token=xoxb-...
```

**Credential management:** `xoxb-…` tokens must not be in plaintext config committed to version control.
In deployment, inject via environment variable (`CASEHUB_QHORUS_SLACK_BOT_TOKEN`) or HashiCorp Vault.

**V1.5 multi-workspace token plan:** replace the single token with a `@ConfigMapping` map keyed by
workspace ID:

```properties
casehub.qhorus.slack.bot.tokens.T456DEF=xoxb-workspace-one
casehub.qhorus.slack.bot.tokens.T789GHI=xoxb-workspace-two
```

`SlackChannelBackend.post()` looks up `tokens.get(entry.workspaceId())`. `SlackBotBinding.workspaceId`
is stored now precisely so v1.5 requires no schema migration.

---

## Micrometer Instrumentation

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
| `SlackBotClient` | WireMock | Already tested — see connectors `slack-bot` tests |
| `SlackInboundConnector` (modified) | Existing unit test pattern | Already tested — slack-ts fields in metadata |
| `SlackInboundNormaliser` | Pure unit test | `/command` → COMMAND; plain text → QUERY; `senderInstanceId` = `"human:U456DEF"` |
| `SlackChannelBackend` registration | `@QuarkusTest`, in-memory binding store | `ChannelInitialisedEvent` → registered + cache + reverse-index populated for bound channels; unbound channels skipped; deregister-before-register idempotent |
| `SlackChannelBackend` inbound routing | `@QuarkusTest` | Inbound with matching `externalChannelRef` → O(1) lookup via `slackToChannel` → `receiveHumanMessage()` called; non-matching → `discarded` counter; `threadCache.recordInbound()` called for top-level messages (no `slack-thread-ts`); NOT called for thread replies |
| `SlackChannelBackend` outbound — EVENT guard | Unit test | `post()` with `content=null` → no HTTP call, no error |
| `SlackChannelBackend` outbound — threading | `@QuarkusTest` + WireMock | `post()` with cached slackTs → `thread_ts` in request body; `post()` with empty cache → no `thread_ts`; blank token → no HTTP call, error logged, counter incremented |
| `SlackChannelBackend` close | Unit test | `close()` removes from `cache`, `slackToChannel`, `threadCache`, and calls `deleteByChannelId()` — verify all four; subsequent `post()` finds no cache entry and logs error |
| **End-to-end thread round-trip** | `@QuarkusTest` + WireMock | Inbound message with `slack-ts="T1"` → `recordInbound()` called → outbound `post()` → WireMock captures request, asserts `thread_ts="T1"` |
| **EVENT message round-trip** | `@QuarkusTest` + WireMock | Outbound EVENT with null content → WireMock receives no request |
| `SlackBindingResource` | `@QuarkusTest` + REST Assured | GET /bindings returns all; GET /{id} → one or 404; POST → 201 + backend registered + cache + reverse-index populated; POST duplicate → 409; DELETE → 204 always; DELETE calls `bindingStore.deleteByChannelId` + `deregisterBackend` + `close()` — verify cache and slackToChannel are empty after delete |
| `ConnectorChannelBackend` WARN→DEBUG | `@QuarkusTest` | Slack event with no connector binding → DEBUG (not WARN) in log output; `inbound_messages_discarded_total` counter incremented |

---

## Cross-Repo Work

### casehub-connectors (connectors#2 — DONE, merged to main)

1. ✅ `casehub-connectors-slack-bot` Maven submodule with `SlackBotClient`
2. ✅ `InboundConnectorIds.SLACK_INBOUND = "slack-inbound"` constant
3. ✅ `SlackInboundConnector.parseMessages()` — `"slack-ts"`, `"slack-thread-ts"` in metadata

### casehub-qhorus (qhorus#261 — not started)

1. New `casehub-qhorus-slack-channel` Maven module with `pom.xml`
2. All classes: `SlackBotBinding`, `SlackBotBindingStore`, `SlackThreadCache`,
   `SlackInboundNormaliser`, `SlackChannelBackend`, `SlackBindingResource`
3. Flyway `V1__slack_bot_binding.sql`
4. **Flyway location config** — `quarkus.flyway.locations` must include
   `classpath:db/slack-channel/migration/`. Default is `classpath:db/migration` only.
   Without this the migration never runs and the table is missing on first deploy.
   Check whether `connector-backend` set a precedent for module-scoped migration paths.
   **Deployment blocker — include in qhorus#261 scope.**
5. **`ConnectorChannelBackend.onInboundMessage()`** (in `qhorus/connector-backend`) —
   change WARN to DEBUG when no binding found. The log noise comes from Slack bot events
   that have no `ChannelConnectorBinding` (correct: they're handled by `SlackChannelBackend`).
   The `inbound_messages_discarded_total` counter (tagged with `connector_id`) is the
   alerting surface — anyone relying on WARN must migrate to the counter.
   **Note: this file is in qhorus/connector-backend, not in the new slack-channel module.**
6. Full test suite including end-to-end thread round-trip and EVENT null-content guard

Depends on `casehub-connectors-slack-bot` published to GitHub Packages (0.2-SNAPSHOT — already published ✅).

---

## Known Limitations (V1)

| Limitation | What V1 delivers | V1.5 path |
|---|---|---|
| Thread correlation (precise) | `inReplyTo` always null in `NormalisedMessage`; `OutboundMessage.inReplyTo` (Qhorus ledger Long) has no Slack ts mapping | Requires `receiveHumanMessage()` return value or dispatch CDI event; design TBD |
| Thread support (outbound) | Replies in most-recent human message's thread — correct for linear Q/A, wrong for concurrent multi-threaded | Precise ledger-ID → slackTs mapping once thread correlation is solved |
| Thread cache lost on restart | In-memory `ConcurrentHashMap` — first post after restart goes to channel root | Persist to DB if post-restart threading becomes a user-visible problem |
| Native slash commands | Not handled | New `WebhookInboundConnector` for slash command URL |
| Rich outbound formatting | `[TYPE] content` plain text; EVENT messages silently dropped | Block Kit formatting keyed by `MessageType` |
| Single workspace | One `botToken` for all channels | `@ConfigMapping` map keyed by `workspaceId` — schema already supports it |
| REST endpoint auth | No auth — network isolation required in deployment | Platform auth mechanism to be established; tracked separately |
