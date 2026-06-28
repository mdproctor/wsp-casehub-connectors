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
- `DiscordDiscovery implements ConnectorDiscovery` — lists guild channels as discoverable targets
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

| Property | Module | Required | Purpose |
|---|---|---|---|
| `casehub.discord.guild-id` | `discord` | Yes | Target guild (server) snowflake ID. Scopes all guild-level operations. |
| `casehub.discord.token` | `chat-discord` | Yes | Bot token. Injected by `DiscordChatPlatform` and `DiscordInboundConnector`, passed to `DiscordClient` at call time. |

Credential ownership follows `credential-config-ownership` protocol: `DiscordClient` takes `String token` as a parameter on every method — it holds no `@ConfigProperty` for credentials. Callers inject their own token.

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
DiscordChannel createChannel(String token, String name, String topic, int type, boolean nsfw);

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
record DiscordChannel(String id, String name, String topic, int type, String parentId) {}
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

HTTP 429 → read `Retry-After` header → sleep → retry. Same pattern as `SlackBotClient`.

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

- **Transport:** `java.net.http.HttpClient.newWebSocketBuilder()` — no external WebSocket library
- **URL:** `wss://gateway.discord.gg/?v=10&encoding=json`
- **HELLO (opcode 10):** Extract `heartbeat_interval`
- **IDENTIFY (opcode 2):** Send `{token, intents, properties: {os, browser, device}}`
- **READY:** Cache `session_id` and `resume_gateway_url`
- **HEARTBEAT (opcode 1):** Dedicated virtual thread. First heartbeat after `interval * jitter`. Subsequent every `interval` ms. Payload: last sequence number.
- **HEARTBEAT_ACK (opcode 11):** Expected after each heartbeat. Missing ACK → close and reconnect.
- **DISPATCH (opcode 0):** Delegate `(eventType, data)` to `GatewayEventListener`
- **Reconnect:** Connect to `resume_gateway_url`, send RESUME (opcode 6) with `session_id` + `seq`. On INVALID_SESSION (opcode 9) → full re-IDENTIFY.
- **Backoff:** Exponential 1s → 2s → 4s → ... → 60s max. Same pattern as `IrcInboundConnector`.
- **Sequence tracking:** AtomicLong updated on every DISPATCH.

### Gateway rate limits

- 120 send events per 60 seconds per connection
- 1000 IDENTIFY calls per 24 hours (global)
- Max send payload: 4096 bytes

---

## DiscordChatPlatform — Capability Mapping

`@ApplicationScoped` CDI bean. Capabilities as constructor-initialized fields. `id() = "discord"`.

### Native capabilities (8 of 9)

| Capability | Discord API | Implementation notes |
|---|---|---|
| **Messaging** | `POST /channels/{id}/messages` | `content.text()` → `content` field. Maps `PostResult` to `SendResult`. |
| **Threading** | `POST /channels/{id}/messages` with `message_reference` | Inline reply with parent preview. Discord threads (child channels) map to ChannelManagement, not Threading. |
| **Discovery** | `GET /guilds/{guild_id}/channels` | Filtered to text channels (type 0, 5) and threads (10, 11, 12). Forum channels (15) included. |
| **Reactions** | `PUT/DELETE reactions/{emoji}/@me` | `add()` and `remove()` are direct. `list()` fetches the message object and extracts the `reactions` array for emoji names. Unicode emoji URL-encoded; custom emoji passed as `name:id`. |
| **Presence** | Gateway PRESENCE_UPDATE cache | `of()` reads `DiscordGatewayPresenceCache`. `set()` is a no-op — Discord doesn't support setting another user's presence. Discord statuses: `online`→ONLINE, `idle`→AWAY, `dnd`→DND, `offline`→OFFLINE. Unknown members return `UNKNOWN`. |
| **Members** | `GET /guilds/{guild_id}/members` | Guild-level, not channel-level. Returns all guild members. Requires GUILD_MEMBERS privileged intent. Paginated with fail-soft. |
| **ChannelManagement** | `POST /guilds/{guild_id}/channels`, `GET /channels/{id}` | `isPrivate` → creates channel with `@everyone` VIEW_CHANNEL denied. `description` maps to `topic` (Discord has no separate description). |
| **MessageHistory** | `GET /channels/{id}/messages?after=` | `Instant since` converted to synthetic snowflake: `(timestamp_ms - DISCORD_EPOCH) << 22`. Paginated (100/page), fail-soft. `parentRef` from `referencedMessageId` on type-19 messages. |

### Degraded capability (1 of 9)

| Capability | Degradation | Reason |
|---|---|---|
| **MemberManagement** | `NoOpMemberManagement` | Discord per-channel membership is permission-based (role/user permission overrides), not add/remove. The SPI's `add(channel, member)` / `remove(channel, member)` don't map to Discord's permission model. |

### DiscordGatewayPresenceCache

`@ApplicationScoped` bean. `ConcurrentHashMap<String, PresenceStatus>` populated by the `DiscordInboundConnector` when it processes PRESENCE_UPDATE Gateway events.

```java
void update(String userId, PresenceStatus status);
PresenceStatus get(String userId);  // returns UNKNOWN if absent
```

---

## DiscordInboundConnector

`@ApplicationScoped` CDI bean implementing `InboundConnector` (pull-based, long-lived connection).

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

- **MESSAGE_CREATE:** Filter out `author.bot == true`. Build `InboundMessage` with `connectorId = "discord-inbound"`, `connectorType = "discord"`. Metadata: `discord-message-id`, `discord-guild-id`, `discord-reference-id` (if reply). Deliver via `sink.receive(inboundMessage)`.
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

Tests: `sendMessage`, `sendReply`, `getMessages` (pagination + fail-soft), `listGuildChannels`, `addReaction`, `listReactionEmoji`, `listGuildMembers` (pagination), `createChannel`, `rateLimitRetry` (429 + Retry-After), `getGatewayUrl`.

No new test dependency — WireMock is on the Quarkus test classpath.

### Layer 2: DiscordGateway — embedded mock WebSocket server

`DiscordGatewayTest` uses a purpose-built embedded WebSocket server (~150 lines, virtual threads, `localhost:0`). Simpler than the IRC server — JSON frames with opcodes, no protocol parsing.

Tests: `connectAndIdentify`, `heartbeatLoop` (interval + seq), `heartbeatAckTimeout` (→ reconnect), `dispatchEvent` (MESSAGE_CREATE delivered), `resumeOnDisconnect` (RESUME with session_id + seq), `invalidSessionFallback` (→ re-IDENTIFY), `reconnectBackoff` (exponential).

### Layer 3: DiscordChatPlatform — contract tests

`DiscordChatPlatformTest` — integration tests using WireMock + mock WebSocket together. Verifies all 9 capabilities through the ChatPlatform SPI.

Tests: `messaging_send`, `threading_reply`, `discovery_listChannels`, `reactions_addRemoveList`, `presence_ofMember`, `presence_setIsNoOp`, `members_list`, `memberManagement_isDegraded`, `channelManagement_createAndFind`, `messageHistory_messagesSince`, `inbound_messageCreateFiresEvent`.

### No real Discord in CI

All tests use mocks. Real Discord testing is manual (create bot, invite to test guild, enable privileged intents in Developer Portal).

---

## SPI Impact

No SPI changes required. Discord maps to the existing ChatPlatform interface cleanly:

- Threading = message reference replies (not Discord threads-as-channels)
- Discord threads are channels → ChannelManagement, not Threading
- Guild scoping via config property, not SPI parameter
- MemberManagement honestly degraded rather than semantic-stretching

The one gap worth noting for future SPI evolution: `Members.list(ChatChannelRef)` returns guild members for Discord, not channel-specific members. The SPI contract implies channel scope; Discord provides guild scope. `supports(Members.class)` returns true because the data is real — just broader. A future `Scope` concept on the Members capability could express this, but it's not worth an SPI change until a second platform exhibits the same pattern.

---

## Platform Coherence

- **Notification consolidation rule:** Discord joins the canonical connector infrastructure. No other repo should implement a Discord connector.
- **Module tier structure:** `discord` is a pure-Java library module (no ChatPlatform SPI dependency). `chat-discord` is the SPI integration layer.
- **Submodule folder naming:** `discord/` and `chat-discord/` — short names, no repo prefix.
- **Maven coordinates:** `io.casehub:casehub-connectors-discord` and `io.casehub:casehub-connectors-chat-discord`, groupId `io.casehub`, version `${revision}`.

## Deferred Items

- Discord message attachment downloading (separate GET per attachment) — follow-up issue
- Multi-guild support (one ChatPlatform instance per guild) — deferred until a real use case
- `Presence.set()` — no-op; Discord doesn't support setting other users' status
- Discord slash commands / interactions — out of scope for ChatPlatform SPI
