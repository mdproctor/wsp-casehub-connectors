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
Calls `GET /users/@me/guilds` with cursor pagination (`after` parameter, `MAX_PAGES` cap, fail-soft partial return on mid-page error — matching `listGuildMembers` pattern). Returns `null` on first-page failure (matching `getGuild` error pattern), empty list for a valid response with zero guilds.

Channel-scoped methods (`sendMessage`, `sendReply`, `getMessages`, `getChannel`, `deleteChannel`, reactions) are unchanged — they already work on globally unique channel IDs without guild context.

`listGuildChannels(token, guildId)` sets the guild ID on each returned `DiscordChannel` since `GET /guilds/{id}/channels` does not include `guild_id` per channel in the response.

**#39 Finding 1 — configurable CDN hosts:**

Replace the hardcoded `Set<String> allowedCdnHosts` field with:
```java
@ConfigProperty(name = "casehub.discord.attachment.allowed-cdn-hosts",
                defaultValue = "cdn.discordapp.com,media.discordapp.net")
String allowedCdnHostsConfig;
```
Add a `@PostConstruct` to DiscordClient to parse the comma-separated string into a `Set<String>`. The public `downloadAttachment(DiscordAttachment)` method uses the parsed set. The package-private `downloadAttachment(DiscordAttachment, Set<String>)` overload remains for testing.

**Test impact:** `DiscordClientTest` sets fields directly (`client.apiBaseUrl`, `client.guildId`). Adding `@PostConstruct` means tests must call `init()` after setting `allowedCdnHostsConfig`, or continue setting the parsed `allowedCdnHosts` set directly (package-private). The existing `downloadAttachment(attachment, hosts)` test overload is unaffected.

### Layer 2: DiscordChannel model — add guildId

Add `guildId` to the record:
```java
public record DiscordChannel(String id, String guildId, String name, String topic, int type,
                             String parentId,
                             List<PermissionOverwrite> permissionOverwrites) { ... }
```

Update `parseChannel()` in DiscordClient: `guild_id` is nullable (DM channels omit it), parsed as `node.has("guild_id") ? node.get("guild_id").asText() : null`. Add an overload `parseChannel(node, guildId)` for use by `listGuildChannels` where the guild ID is known from the request parameter but absent from the response JSON.

### Layer 3: DiscordChatPlatform — multi-guild aware

**Constructor:** Remove `guildId` injection. Inject only `client`, `presenceCache`, and `token`.

**Init (`@PostConstruct`):**
- If `token.isBlank()` → degraded mode, `activeCapabilities = Set.of()`
- Call `client.listBotGuilds(token)`:
  - If `null` (API error) → log SEVERE with diagnostic guidance ("check token and network connectivity"), degraded mode
  - If empty (valid response, zero guilds) → log WARNING ("bot not added to any guild"), degraded mode
  - Otherwise → cache as `List<DiscordGuild> guilds`; fetch guild details with member counts via `getGuild(token, guild.id(), true)` per guild, cache as `Map<String, DiscordGuild> guildDetails`; set `activeCapabilities = NATIVE_CAPABILITIES`; initialize capabilities normally

**`supports()` uses `activeCapabilities`** (computed at init, following Slack's pattern). ChannelManagement remains in `NATIVE_CAPABILITIES` for multi-guild — `find()` and `delete()` are guild-agnostic, and `create()` throwing is within the SPI contract ("Operations may throw unchecked exceptions on failure"). The degradation default `NoOpChannelManagement` also throws from `create()` and `delete()`, so excluding the capability would break working operations.

**Channel-to-guild map:** Maintain `Map<String, String> channelToGuild` mapping channel IDs to guild IDs, populated during `listChannels()` from the per-guild iteration. Used by `listMembers()` to avoid redundant `getChannel()` calls.

**Capability behavior:**

| Capability | Behavior |
|---|---|
| `discovery().listChannels()` | Iterate cached guilds, call `listGuildChannels(token, guild.id())` for each, use cached `guildDetails` for member counts (no per-call `getGuild` needed), populate `channelToGuild` map, aggregate results |
| `members().listMembers(channel)` | If single guild → use `guilds.get(0).id()`. If multiple → look up `channelToGuild` map, fall back to `client.getChannel(token, channel.id())` only for channels not previously seen. Then `listGuildMembers(token, guildId, ...)` |
| `channelManagement().create(...)` | If `guilds.size() == 1` → use that guild. If multiple → throw `IllegalStateException("Multiple guilds — channel creation requires disambiguation")` |
| `channelManagement().delete` | Already guild-agnostic (operates on channel IDs) — no change |
| `channelManagement().find(channelId)` | Guild-agnostic (operates on channel ID). When the returned `DiscordChannel` has a non-null `guildId`, insert into `channelToGuild` map — avoids redundant `getChannel()` if `listMembers()` is called next |
| `isPrivateChannel(channel)` | Use `channel.guildId()` as the @everyone role ID (Discord convention: @everyone role ID == guild ID). For channels from `listGuildChannels`, guildId is set from the request parameter. For channels from `getChannel`, guildId comes from the API response. |
| Messaging, threading, reactions, presence, history | Already channel-scoped — no change |

**Guild cache staleness:** The guild list and details are cached at `@PostConstruct` time and not refreshed at runtime. If the bot is added to or removed from a guild while running, the change is not reflected until restart. The Gateway receives `GUILD_CREATE`/`GUILD_DELETE` events — runtime cache invalidation is a future enhancement (wsp-casehub-connectors#1).

### Layer 4: DiscordInboundConnector — event-driven guild context

Remove `guildId` field and config injection. Only `token` needed.

- `start()` activation guard: `token.isBlank()` only — the Gateway connects to all guilds
- **Gateway scaling:** `GUILD_CREATE` fires for every guild at connect, each containing initial presences, channels, and member lists. `MESSAGE_CREATE` events flow from all guilds through the single event handler. Volume scales linearly with guild count — negligible for typical CaseHub deployments (single-digit guilds), worth monitoring for bots in dozens of guilds.
- `deliverMessage()`: extract guild from event data instead of hardcoding from config:
  ```java
  String eventGuildId = data.has("guild_id") ? data.get("guild_id").asText() : "unknown";
  metadata.put("discord-guild-id", eventGuildId);
  ```
  This also fixes an existing bug: the current code hardcodes the config value even if the event came from a different guild.

### Layer 5: DiscordDiscovery — multi-guild aggregation

Remove `guildId` field and config injection.

`discover()`: call `client.listBotGuilds(token)`. If `null` (API error), return `List.of()` — fail-soft, matching the existing `listGuildChannels` null-check pattern. Otherwise iterate guilds, call `listGuildChannels(token, guild.id())` for each, filter to text channel types (`0, 5, 10, 11, 12`), aggregate.

DiscordDiscovery and DiscordChatPlatform discover guilds independently — they're in different modules with different lifecycles. This doubles the `listBotGuilds` call at startup (two HTTP round-trips). Acceptable because: the calls happen seconds apart, both modules handle errors in their own context, and `DiscordClient` remains a stateless HTTP transport.

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
