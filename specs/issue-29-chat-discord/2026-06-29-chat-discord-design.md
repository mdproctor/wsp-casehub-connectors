# Discord Chat Connector — Design Spec

**Issue:** casehubio/connectors#29
**Branch:** issue-29-chat-discord
**Date:** 2026-06-29

---

## Overview

Discord ChatPlatform SPI implementation with a shared HTTP client module for the Discord Bot REST API v10 and a WebSocket Gateway client for real-time inbound events. Discord is the first external platform with native support for all capabilities except MemberManagement — it stress-tests the SPI in ways IRC (3 native) and ref (in-memory) could not.

## Module Structure

Two modules, following the `slack-bot` / `chat-*` split established by the Slack connector.

### `discord` module (artifactId: `casehub-connectors-discord`)

Shared HTTP client and Gateway WebSocket client. Independently consumable by MCP tools and a future Qhorus `DiscordChannelBackend`.

- `DiscordClient` — CDI `@ApplicationScoped` bean wrapping `HttpHelper.CLIENT`
- `DiscordGateway` — WebSocket client for Gateway v10 lifecycle
- `DiscordGatewayPresenceCache` — `@ApplicationScoped` ConcurrentHashMap populated by PRESENCE_UPDATE events
- `DiscordDiscovery implements ConnectorDiscovery` — `id() = "discord"`. Lists guild channels as discoverable targets. Blank token → returns empty list (fail-soft).
- Discord model records (DTOs)
- Dependencies: `casehub-connectors-core`

### `chat-discord` module (artifactId: `casehub-connectors-chat-discord`)

ChatPlatform SPI implementation and inbound connector.

- `DiscordChatPlatform implements ChatPlatform` — `@ApplicationScoped`, 8 native capabilities + 1 degraded
- `DiscordInboundConnector implements InboundConnector` — starts Gateway, translates DISPATCH to `InboundMessage`
- `DiscordInboundTranslator implements InboundTranslator` — translates `InboundMessage` to `ReceivedMessage`
- 8 capability implementations as constructor-initialized fields
- Dependencies: `casehub-connectors-chat-spi`, `casehub-connectors-discord`

### Why this split

`DiscordClient` is independently consumable. MCP tools (`send_discord`, `list_discord_channels`) and a future `DiscordChannelBackend` in Qhorus depend on the HTTP client without needing `chat-spi`. Mirrors `casehub-connectors-slack-bot` consumed by both `mcp` and `casehub-qhorus-slack-channel`.

---

## Configuration

| Property | Module | Default | Purpose |
|---|---|---|---|
| `casehub.discord.guild-id` | `discord` | `""` | Target guild (server) snowflake ID. Scopes all guild-level operations. Blank → connector inactive (WARNING + no-op). |
| `casehub.discord.token` | `discord` | `""` | Bot token. Injected by callers (`DiscordDiscovery`, `DiscordChatPlatform`, `DiscordInboundConnector`) and passed to `DiscordClient` at call time. Blank → connector inactive (WARNING + no-op). |

Both properties use `defaultValue = ""` so that deployments which include the `discord` or `chat-discord` module on the classpath without configuring Discord properties do not cause a Quarkus startup failure. Callers check both `token.isBlank()` and `guildId.isBlank()` — if either is blank, the caller logs WARNING and becomes inactive.

Credential ownership follows `credential-config-ownership` protocol: `DiscordClient` takes `String token` as a parameter on every method — it holds no `@ConfigProperty` for credentials. Callers inject their own token. The token property lives in the `discord` module so that all callers — including `DiscordDiscovery` in `discord` — can inject it directly. This follows the `slack-bot` precedent where `SlackBotDiscovery` injects `casehub.connectors.slack-bot.token` from the same module.

---

## DiscordClient — HTTP API

CDI `@ApplicationScoped` bean. All methods take `String token` as first parameter.

### API surface

```java
// Messages
PostResult sendMessage(String token, String channelId, String content);
PostResult sendReply(String token, String channelId, String content, String replyToMessageId);
List<DiscordMessage> getMessages(String token, String channelId, String afterId, int limit);

// Channels
List<DiscordChannel> listGuildChannels(String token);
DiscordChannel getChannel(String token, String channelId);
DiscordChannel createChannel(String token, String name, String topic, int type, boolean nsfw, boolean isPrivate);

// Reactions
void addReaction(String token, String channelId, String messageId, String emoji);
void removeReaction(String token, String channelId, String messageId, String emoji);
List<String> listReactionEmoji(String token, String channelId, String messageId);

// Members
List<DiscordMember> listGuildMembers(String token, int limit, String afterUserId);
DiscordMember getGuildMember(String token, String userId);

// Guild
DiscordGuild getGuild(String token, boolean withCounts);

// Gateway
String getGatewayUrl(String token);
```

### Model records

Package: `io.casehub.connectors.discord.model`

```java
record DiscordMessage(String id, String channelId, DiscordUser author, String content,
                      Instant timestamp, String referencedMessageId, int type) {}
record DiscordChannel(String id, String name, String topic, int type, String parentId, List<PermissionOverwrite> permissionOverwrites) {}
record PermissionOverwrite(String id, int type, long allow, long deny) {}
record DiscordUser(String id, String username, String globalName, boolean bot) {}
record DiscordMember(DiscordUser user, String nick, List<String> roles, Instant joinedAt) {}
record DiscordGuild(String id, String name, int approximateMemberCount) {}
record PostResult(boolean ok, String messageId, String channelId, String error) {}
```

### Protocol compliance

- **shared-http-client:** Uses `HttpHelper.CLIENT` for all requests
- **credential-config-ownership:** Token passed at call time, not stored
- **paginating-client-fail-soft:** `listGuildMembers` and `getMessages` paginate with `MAX_PAGES` constant, return partial results + WARNING on mid-loop failure
- **spi-id-method-naming:** `DiscordDiscovery.id()` not `connectorId()`
- **inbound-connector-id-constants:** `InboundConnectorIds.DISCORD_INBOUND = "discord-inbound"` added to `core`

### Rate limit handling

HTTP 429 → read `Retry-After` header → sleep → retry **once**. If the retry also returns 429, return the failure result (`PostResult(ok=false, error="rate-limited")`). Same single-retry pattern as `SlackBotClient.sendWithRetry()`.

### Error response mapping

Non-paginating methods follow these conventions:

| Method | Error status | Result |
|---|---|---|
| `sendMessage`, `sendReply` | 4xx/5xx | `PostResult(ok=false, error="<status> <reason>")` |
| `getChannel` | 404 | Returns `null` → `ChannelManagement.find()` maps to `Optional.empty()` |
| `getChannel` | 403/other | Returns `null` + WARNING log |
| `getGuildMember` | 404 | Returns `null` → `Presence.of()` treats absent member as `UNKNOWN` |
| `getGuild` | 4xx/5xx | Returns `null` + WARNING log |
| `addReaction`, `removeReaction` | 4xx/5xx | WARNING log, no return value (void methods) |
| `listReactionEmoji` | 4xx/5xx | Returns empty list + WARNING log |

All error paths log at WARNING level with the HTTP status code and response body. No exceptions propagate from `DiscordClient` — consistent with the fail-soft philosophy.

### Base URL

`https://discord.com/api/v10` — hardcoded. The API version is part of the contract.

---

## DiscordGateway — WebSocket Client

Long-lived WebSocket connection to Discord Gateway v10. Lives in the `discord` module — transport-level, independent of ChatPlatform SPI.

### Connection lifecycle

```
DISCONNECTED → CONNECTING → HELLO_RECEIVED → IDENTIFYING → READY → RUNNING
                                                                     ↓
                                                              (connection lost)
                                                                     ↓
                                                              RESUMING → RUNNING
                                                                     ↓
                                                              (INVALID_SESSION)
                                                                     ↓
                                                              IDENTIFYING → READY → RUNNING
```

### API

```java
public class DiscordGateway {
    void connect(String token, int intents, GatewayEventListener listener);
    void disconnect();
    boolean isConnected();
}

@FunctionalInterface
public interface GatewayEventListener {
    void onEvent(String eventType, JsonNode data);
}
```

### Protocol details

- **Transport:** `HttpHelper.CLIENT.newWebSocketBuilder()` — uses the shared `HttpClient` per `shared-http-client` protocol (PP-20260607-9794cb). No external WebSocket library.
- **URL:** `wss://gateway.discord.gg/?v=10&encoding=json`
- **HELLO (opcode 10):** Extract `heartbeat_interval`
- **IDENTIFY (opcode 2):** Send `{token, intents, properties: {os, browser, device}}`
- **READY:** Cache `session_id` and `resume_gateway_url`
- **HEARTBEAT (opcode 1):** Dedicated virtual thread. First heartbeat after `interval * jitter`. Subsequent every `interval` ms. Payload: last sequence number.
- **HEARTBEAT_ACK (opcode 11):** Expected after each heartbeat. Missing ACK → close and reconnect.
- **DISPATCH (opcode 0):** Delegate `(eventType, data)` to `GatewayEventListener`
- **Message accumulation:** `WebSocket.Listener.onText(WebSocket, CharSequence, boolean last)` delivers frames, not messages. Buffer partial frames (where `last == false`) in a `StringBuilder`; only parse and dispatch when `last == true`. Large DISPATCH events (e.g., GUILD_CREATE with many channels/members) can span multiple frames.
- **Reconnect:** Connect to `resume_gateway_url`, send RESUME (opcode 6) with `session_id` + `seq`. On INVALID_SESSION (opcode 9) → full re-IDENTIFY.
- **Backoff:** Exponential 1s → 2s → 4s → ... → 60s max. Same pattern as `IrcInboundConnector.connectLoop()`. Track `consecutiveFailures`; log at WARNING for attempts 1–4, escalate to SEVERE at attempt 5+ (matching IRC's log escalation to surface sustained connectivity failures).
- **Sequence tracking:** AtomicLong updated on every DISPATCH.

### Gateway rate limits

- 120 send events per 60 seconds per connection
- 1000 IDENTIFY calls per 24 hours (global)
- Max send payload: 4096 bytes

**v1 budget analysis:** v1 sends only HEARTBEAT, IDENTIFY, and RESUME over the WebSocket. At a typical `heartbeat_interval` of ~41s, heartbeats consume ~1.5 events/minute — well within the 120/60s budget. No proactive rate tracking needed. If future versions add frequent outbound events (e.g., PRESENCE_UPDATE, VOICE_STATE_UPDATE), add a send-side counter with defer-on-limit.

---

## DiscordChatPlatform — Capability Mapping

`@ApplicationScoped` CDI bean. Capabilities as constructor-initialized fields. `id() = "discord"`.

### Native capabilities (8 of 9)

| Capability | Discord API | Implementation notes |
|---|---|---|
| **Messaging** | `POST /channels/{id}/messages` | Prefers `content.markdown()` when non-null (Discord natively renders Markdown), falls back to `content.text()`. Content exceeding 2000 characters → `SendResult.failure("Content exceeds Discord's 2000-character limit")`. Maps `PostResult` to `SendResult`. |
| **Threading** | `POST /channels/{id}/messages` with `message_reference` | Inline reply with parent preview. Discord threads (child channels) map to ChannelManagement, not Threading. |
| **Discovery** | `GET /guilds/{guild_id}/channels` | Filtered to text channels (type 0, 5) and threads (10, 11, 12). Forum channels (type 15) excluded — they require thread creation before messaging and would cause 400 errors if a caller attempted `messaging.send()` on a discovered forum channel. |
| **Reactions** | `PUT/DELETE reactions/{emoji}/@me` | `add()` and `remove()` are direct. `list()` fetches the message object and extracts the `reactions` array for emoji names. Unicode emoji URL-encoded; custom emoji passed as `name:id`. |
| **Presence** | Gateway PRESENCE_UPDATE cache | `of()` reads `DiscordGatewayPresenceCache`. `set()` logs WARNING and returns — Discord's API does not support setting another user's presence. This is a permanent platform limitation, not a deferred feature. Discord statuses: `online`→ONLINE, `idle`→AWAY, `dnd`→DND, `offline`→OFFLINE. Unknown members return `UNKNOWN`. **Dependency:** `DiscordGatewayPresenceCache` is populated only when `chat-discord` is on the classpath (the `DiscordInboundConnector` feeds PRESENCE_UPDATE events into it). Deployments using `discord` without `chat-discord` (e.g., MCP tools, future `DiscordChannelBackend`) will see all `of()` calls return `UNKNOWN`. |
| **Members** | `GET /guilds/{guild_id}/members` | **Known semantic deviation:** returns guild-level members, not channel-scoped. All guild members are returned regardless of which `ChatChannelRef` is passed. For public channels this is defensible — all guild members can see them by default. **For private channels, results are incorrect:** a private channel denies `@everyone` VIEW_CHANNEL, so most returned members cannot actually access the channel. Callers needing accurate private-channel membership must filter results themselves using permission data. `supports(Members.class)` returns `true` because the data is real for public channels (the common case). Requires GUILD_MEMBERS privileged intent. Paginated with fail-soft. |
| **ChannelManagement** | `POST /guilds/{guild_id}/channels`, `GET /channels/{id}` | **create():** passes `type = 0` (GUILD_TEXT) and `nsfw = false` to `DiscordClient.createChannel()`. `isPrivate` → `createChannel(..., isPrivate=true)` includes a `permission_overwrites` array denying `@everyone` the `VIEW_CHANNEL` permission (bit `1 << 10`). SPI `topic` → Discord `topic` field. SPI `description` → ignored (Discord has no separate description; returned as `null`). **find():** derives `Channel.isPrivate` from `DiscordChannel.permissionOverwrites()`: scan for a role-type (type 0) overwrite where `id` equals the guild ID (from `casehub.discord.guild-id` config) and `deny` has bit 10 (`VIEW_CHANNEL`) set. Same derivation applies to channels returned by `Discovery.listChannels()`. |
| **MessageHistory** | `GET /channels/{id}/messages?after=` | `Instant since` converted to synthetic snowflake: `(timestamp_ms - DISCORD_EPOCH) << 22`. Paginated (100/page), fail-soft. `parentRef` from `referencedMessageId` on type-19 messages. |

### Degraded capability (1 of 9)

| Capability | Degradation | Reason |
|---|---|---|
| **MemberManagement** | `NoOpMemberManagement` | Discord per-channel membership is permission-based (role/user permission overrides), not add/remove. The SPI's `add(channel, member)` / `remove(channel, member)` don't map to Discord's permission model. |

### DiscordGatewayPresenceCache

`@ApplicationScoped` bean in the `discord` module. `ConcurrentHashMap<String, PresenceStatus>` populated by the `DiscordInboundConnector` (in `chat-discord`) when it processes PRESENCE_UPDATE Gateway events.

**Classpath dependency:** The cache is populated only when `chat-discord` is on the classpath. Without the Gateway connection provided by `DiscordInboundConnector`, all `get()` calls return `UNKNOWN`. This is by design — deployments using `discord` alone (MCP tools, future `DiscordChannelBackend`) get correct but empty presence data, not errors.

```java
void update(String userId, PresenceStatus status);
PresenceStatus get(String userId);  // returns UNKNOWN if absent
```

---

## DiscordInboundConnector

`@ApplicationScoped` CDI bean implementing `InboundConnector` (pull-based, long-lived connection).

### Blank-config fail-soft

`start(sink)` checks `token.isBlank() || guildId.isBlank()` before starting the connect loop. Blank → `LOG.warning("discord-inbound: token or guild-id not configured, connector inactive")` and returns immediately. This follows the credential-config-ownership protocol (PP-20260609-0c3e24): blank credentials → WARNING + no-op. Without this, a deployment including `chat-discord` without configuring Discord properties would either enter a permanent reconnect loop (blank token → Gateway close code 4004) or produce incorrect API calls (blank guild-id → malformed URLs). The same blank-config check applies to `DiscordDiscovery` and `DiscordChatPlatform`.

### Shutdown path

`stop()` follows the `IrcInboundConnector.stop()` pattern:
1. Set `volatile boolean stopping = true`
2. Call `gateway.disconnect()` — closes the WebSocket cleanly
3. Shut down the reconnect executor (`executor.shutdownNow()`)

The `stopping` flag is checked before each reconnect attempt in the connect loop — prevents reconnection during JVM shutdown.

### Gateway intents

| Intent | Bit | Purpose |
|---|---|---|
| GUILDS | 1 << 0 | Channel lifecycle events |
| GUILD_MEMBERS | 1 << 1 | Member events (privileged) |
| GUILD_PRESENCES | 1 << 8 | Presence updates (privileged) |
| GUILD_MESSAGES | 1 << 9 | MESSAGE_CREATE inbound |
| GUILD_MESSAGE_REACTIONS | 1 << 10 | Reaction events |
| MESSAGE_CONTENT | 1 << 15 | Message content access (privileged) |

Three privileged intents. Bots in < 100 guilds: toggle in Developer Portal. 100+ guilds: requires Discord approval (deployment concern, not design concern).

### Event handling

- **MESSAGE_CREATE:** Filter out `author.bot == true`. Filter to message types 0 (DEFAULT) and 19 (REPLY) only — skip all other types (system messages: member join type 7, boost type 8, follow type 12, interaction types 20–21, etc.). Without the type filter, system messages with `author.bot == false` would be delivered as regular chat messages. This matches the IRC precedent where `IrcClient` only fires the callback on `PRIVMSG`. Build `InboundMessage` with `connectorId = "discord-inbound"`, `connectorType = "discord"`. Metadata: `discord-message-id`, `discord-guild-id`, `discord-reference-id` (if reply, type 19). Deliver via `sink.receive(inboundMessage)`.
- **GUILD_CREATE:** Iterate `data.presences[]` and call `presenceCache.update(userId, status)` for each entry. This seeds the presence cache on connection — without it, all `Presence.of()` queries return `UNKNOWN` until individual members change status (which could take hours). No `InboundMessage` generated.
- **PRESENCE_UPDATE:** Extract `user.id` + `status`. Update `DiscordGatewayPresenceCache`. No `InboundMessage` generated.
- All other events: ignored.

### InboundMessage metadata keys

| Key | Value | When present |
|---|---|---|
| `discord-message-id` | Message snowflake ID | Always |
| `discord-guild-id` | Guild snowflake ID | Always |
| `discord-reference-id` | Referenced message ID | When message is a reply (type 19) |

### DiscordInboundTranslator

`@ApplicationScoped` CDI bean implementing `InboundTranslator`. `connectorType() = "discord"`. Translates `InboundMessage` → `ReceivedMessage` using metadata keys above.

### Attachments

Deferred to a follow-up issue. Discord message attachments require a separate GET to download content. v1 produces empty attachment lists.

---

## Testing Strategy

### Layer 1: DiscordClient — WireMock HTTP mocks

`DiscordClientTest` stubs Discord REST API responses. Verifies request construction, response parsing, rate limit retry, pagination fail-soft, error mapping.

Tests: `sendMessage`, `sendReply`, `getMessages` (pagination + fail-soft), `listGuildChannels`, `addReaction`, `listReactionEmoji`, `listGuildMembers` (pagination), `createChannel`, `createChannel_privateIncludesPermissionOverwrites`, `rateLimitRetry` (429 + Retry-After, single retry), `rateLimitRetry_secondRetryReturnsFailure`, `getChannel_404ReturnsNull`, `getChannel_403ReturnsNullWithWarning`, `sendMessage_4xxReturnsErrorPostResult`, `getGatewayUrl`.

No new test dependency — WireMock is on the Quarkus test classpath.

### Layer 2: DiscordGateway — embedded mock WebSocket server

`DiscordGatewayTest` uses a purpose-built embedded WebSocket server (~150 lines, virtual threads, `localhost:0`). Simpler than the IRC server — JSON frames with opcodes, no protocol parsing.

Tests: `connectAndIdentify`, `heartbeatLoop` (interval + seq), `heartbeatAckTimeout` (→ reconnect), `dispatchEvent` (MESSAGE_CREATE delivered), `resumeOnDisconnect` (RESUME with session_id + seq), `invalidSessionFallback` (→ re-IDENTIFY), `reconnectBackoff` (exponential), `reconnectBackoff_logEscalation` (WARNING→SEVERE at attempt 5), `multiFrameMessage_accumulatedBeforeParse`, `guildCreate_presencesCacheSeeded`.

### Layer 3: DiscordChatPlatform — contract tests

`DiscordChatPlatformTest` — integration tests using WireMock + mock WebSocket together. Verifies all 9 capabilities through the ChatPlatform SPI.

Tests: `messaging_send`, `messaging_contentExceeds2000CharsReturnsFailure`, `messaging_prefersMarkdownOverText`, `threading_reply`, `discovery_listChannels`, `discovery_excludesForumChannels`, `reactions_addRemoveList`, `presence_ofMember`, `presence_setLogsWarning`, `members_list`, `memberManagement_isDegraded`, `channelManagement_createAndFind`, `channelManagement_createPrivateChannel`, `channelManagement_findDerivesIsPrivate`, `messageHistory_messagesSince`, `inbound_messageCreateFiresEvent`, `inbound_systemMessagesFiltered`, `inbound_blankTokenConnectorInactive`, `inbound_blankGuildIdConnectorInactive`, `discovery_blankTokenReturnsEmpty`.

### No real Discord in CI

All tests use mocks. Real Discord testing is manual (create bot, invite to test guild, enable privileged intents in Developer Portal).

---

## SPI Impact

No SPI changes required. Discord maps to the existing ChatPlatform interface cleanly:

- Threading = message reference replies (not Discord threads-as-channels)
- Discord threads are channels → ChannelManagement, not Threading
- Guild scoping via config property, not SPI parameter
- MemberManagement honestly degraded rather than semantic-stretching

Two gaps worth noting for future SPI evolution:

1. **Members scope:** `Members.list(ChatChannelRef)` returns guild members for Discord, not channel-specific members. The SPI contract implies channel scope; Discord provides guild scope. For public channels this is the broadest correct answer (all guild members can see them). For private channels, the results are incorrect — members who cannot access the channel are included. `supports(Members.class)` returns true because the data is real for the common case (public channels). A future `Scope` concept on the Members capability could express this, but it's not worth an SPI change until a second platform exhibits the same pattern.

2. **Topic/description divergence:** The ARC42 §10 entry states "Topic is dynamic, description is static — Slack/Discord distinguish them." This is correct for Slack (topic vs. purpose) but incorrect for Discord — Discord channels have only a `topic` field with no separate description. The ARC42 entry should be corrected to: "Topic is dynamic, description is static — Slack distinguishes them (topic/purpose); Discord has only topic; IRC has only topic."

---

## Platform Coherence

- **Notification consolidation rule:** Discord joins the canonical connector infrastructure. No other repo should implement a Discord connector.
- **Module tier structure:** `discord` is a pure-Java library module (no ChatPlatform SPI dependency). `chat-discord` is the SPI integration layer.
- **Submodule folder naming:** `discord/` and `chat-discord/` — short names, no repo prefix.
- **Maven coordinates:** `io.casehub:casehub-connectors-discord` and `io.casehub:casehub-connectors-chat-discord`, groupId `io.casehub`, version `${revision}`.

## Deferred Items

Each deferred item has a tracked GitHub issue in `casehubio/connectors`:

| Item | Issue | Rationale |
|---|---|---|
| Discord message attachment downloading | casehubio/connectors#30 | Requires separate GET per attachment; v1 produces empty attachment lists |
| Multi-guild support | casehubio/connectors#31 | One ChatPlatform instance per guild; deferred until a real use case |
| Discord slash commands / interactions | casehubio/connectors#32 | Out of scope for ChatPlatform SPI; separate interaction model |
| Discord embeds (rich structured messages) | casehubio/connectors#33 | Common bot feature; requires DiscordClient API extension for embed objects |
| MCP tools (`send_discord`, `list_discord_channels`) | casehubio/connectors#34 | Future consumers of `DiscordClient`; MCP module integration |

Note: `Presence.set()` is a permanent no-op, not a deferred feature — Discord's API does not support setting another user's presence. No issue needed.
