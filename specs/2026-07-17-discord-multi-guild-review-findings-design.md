# Discord Multi-Guild Support + Code Review Findings

**Issues:** #39 (code review findings), #31 (multi-guild support)
**Date:** 2026-07-17

## Problem

`DiscordClient` carries `guildId` as a `@ConfigProperty` field. Five methods read it implicitly, making the HTTP client a guild-scoped singleton when it should be a stateless transport. This prevents operating across multiple guilds and conflates transport with policy.

Additionally, three code review findings from #30/#33/#34 need resolution:
1. `allowedCdnHosts` is hardcoded — should be configurable
2. `downloadAttachment` with null URL path is untested
3. HTTP/1.1 connection reuse on oversized abort (known limitation)

## Design

### Layer 1: DiscordClient — guild-agnostic HTTP transport

Remove the `guildId` field and `@ConfigProperty` injection. Add `guildId` as a parameter to all guild-scoped methods:

| Current | New |
|---|---|
| `listGuildChannels(token)` | `listGuildChannels(token, guildId)` |
| `createChannel(token, name, topic, type, nsfw, isPrivate)` | `createChannel(token, guildId, name, topic, type, nsfw, isPrivate)` |
| `listGuildMembers(token, limit, afterUserId)` | `listGuildMembers(token, guildId, limit, afterUserId)` |
| `getGuildMember(token, userId)` | `getGuildMember(token, guildId, userId)` |
| `getGuild(token, withCounts)` | `getGuild(token, guildId, withCounts)` |

Remove `guildId()` accessor.

New method:
```java
public List<DiscordGuild> listBotGuilds(String token)
```
Calls `GET /users/@me/guilds`. Returns guilds the bot is a member of.

Channel-scoped methods (`sendMessage`, `sendReply`, `getMessages`, `getChannel`, `deleteChannel`, reactions) are unchanged — they already work on globally unique channel IDs without guild context.

**#39 Finding 1 — configurable CDN hosts:**

Replace the hardcoded `Set<String> allowedCdnHosts` field with:
```java
@ConfigProperty(name = "casehub.discord.attachment.allowed-cdn-hosts",
                defaultValue = "cdn.discordapp.com,media.discordapp.net")
String allowedCdnHostsConfig;
```
Add a `@PostConstruct` to DiscordClient to parse the comma-separated string into a `Set<String>`. The public `downloadAttachment(DiscordAttachment)` method uses the parsed set. The package-private `downloadAttachment(DiscordAttachment, Set<String>)` overload remains for testing.

### Layer 2: DiscordChannel model — add guildId

Add `guildId` to the record:
```java
public record DiscordChannel(String id, String guildId, String name, String topic, int type,
                             String parentId,
                             List<PermissionOverwrite> permissionOverwrites) { ... }
```

Update `parseChannel()` in DiscordClient: `guild_id` is nullable (DM channels omit it), parsed as `node.has("guild_id") ? node.get("guild_id").asText() : null`.

### Layer 3: DiscordChatPlatform — multi-guild aware

**Constructor:** Remove `guildId` injection. Inject only `client`, `presenceCache`, and `token`.

**Init (`@PostConstruct`):**
- If `token.isBlank()` → degraded mode (no guild-id check needed)
- Call `client.listBotGuilds(token)` → cache as `List<DiscordGuild> guilds`
- If empty → warn, degraded mode
- Otherwise → initialize capabilities normally

**Capability behavior:**

| Capability | Behavior |
|---|---|
| `discovery().listChannels()` | Iterate cached guilds, call `listGuildChannels(token, guild.id())` for each, call `getGuild(token, guild.id(), true)` per guild for member counts, aggregate results |
| `members().listMembers(channel)` | Call `client.getChannel(token, channel.id())` to resolve channel's `guildId`, then `listGuildMembers(token, guildId, ...)` |
| `channelManagement().create(...)` | If `guilds.size() == 1` → use that guild. If multiple → throw `IllegalStateException("Multiple guilds — channel creation requires disambiguation")` |
| `channelManagement().delete/find` | Already guild-agnostic (operates on channel IDs) — no change |
| `isPrivateChannel(channel)` | Use `channel.guildId()` as the @everyone role ID (Discord convention: @everyone role ID == guild ID) |
| Messaging, threading, reactions, presence, history | Already channel-scoped — no change |

### Layer 4: DiscordInboundConnector — event-driven guild context

Remove `guildId` field and config injection. Only `token` needed.

- `start()` activation guard: `token.isBlank()` only — the Gateway connects to all guilds
- `deliverMessage()`: extract guild from event data instead of hardcoding from config:
  ```java
  String eventGuildId = data.has("guild_id") ? data.get("guild_id").asText() : "unknown";
  metadata.put("discord-guild-id", eventGuildId);
  ```
  This also fixes an existing bug: the current code hardcodes the config value even if the event came from a different guild.

### Layer 5: DiscordDiscovery — multi-guild aggregation

Remove `guildId` field and config injection.

`discover()`: call `client.listBotGuilds(token)`, iterate guilds, call `listGuildChannels(token, guild.id())` for each, filter to text channel types (`0, 5, 10, 11, 12`), aggregate.

DiscordDiscovery and DiscordChatPlatform discover guilds independently — they're in different modules with different lifecycles.

### #39 Finding 2 — null URL downloadAttachment test

Add a test to `DiscordClientTest`:
- Create `DiscordAttachment` with `null` URL
- Verify `downloadAttachment` returns `null` without throwing
- Documents the graceful handling path (NPE from `URI.create(null)` caught by the Exception handler)

### #39 Finding 3 — HTTP/1.1 oversized abort

No code change. The streaming abort returns `null` correctly. HTTP/2 handles this via RST_STREAM; HTTP/1.1 may leave the connection unusable under unlikely conditions (many oversized downloads in sequence). Known limitation, not a bug.

## Config changes

| Property | Status |
|---|---|
| `casehub.discord.guild-id` | **Removed** — bot discovers guilds via API |
| `casehub.discord.token` | Unchanged — sole activation guard |
| `casehub.discord.attachment.allowed-cdn-hosts` | **New** — comma-separated, defaults to `cdn.discordapp.com,media.discordapp.net` |
| `casehub.discord.api-base-url` | Unchanged |
| `casehub.discord.attachment.max-bytes` | Unchanged |

## Modules affected

- `discord` — DiscordClient, DiscordDiscovery, DiscordChannel, DiscordClientTest, DiscordDiscoveryTest
- `chat-discord` — DiscordChatPlatform, DiscordInboundConnector, DiscordChatPlatformTest, DiscordInboundConnectorTest

## Not in scope

- Guild filtering (restrict operations to a subset of guilds) — no use case yet
- Per-guild ChatPlatform instances — unnecessary; single instance handles all guilds
- Channel creation disambiguation for multi-guild — throw clearly, revisit when needed
