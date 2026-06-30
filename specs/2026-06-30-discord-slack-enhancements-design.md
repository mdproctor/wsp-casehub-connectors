# Discord & Slack Enhancements — Design Spec

**Branch:** `issue-36-discord-slack-enhancements`
**Covers:** #36, #38, #40
**Date:** 2026-06-30

---

## Scope

Three features plus one bug fix, all on a single branch:

1. **#36** — Discord message history attachment downloading
2. **#38** — Advanced embed MCP parameters for `send_discord`
3. **#40** — `SlackChatPlatform` implementation (new `chat-slack` module)
4. **Bug fix** — `DiscordInboundTranslator` drops attachments during translation

---

## Feature 1: Discord message history attachment downloading (#36)

### Problem

`DiscordChatPlatform.getMessageHistory()` returns messages with `ChatContent.attachments = List.of()`, even though `DiscordMessage.attachments()` already carries parsed attachment metadata from the Discord API. The downloading infrastructure exists in `DiscordClient.downloadAttachment()` with SSRF defense, streaming size enforcement, and CDN host allowlisting — it just isn't wired into the message history path.

### Design

In `DiscordChatPlatform.toReceivedMessage()`, iterate over `dm.attachments()` and call `client.downloadAttachment(da)` for each. Failed downloads are silently dropped (null return → skip), consistent with the inbound connector path.

Downloads are synchronous within the `getMessageHistory()` call. The caller already chose a blocking API; parallelizing with virtual threads adds complexity for a path that returns ≤100 messages. This differs from the inbound connector, which uses virtual threads because it runs on the Gateway event thread.

**Memory risk**: 100 messages × N attachments × `maxAttachmentBytes` (default 8 MB) could produce significant heap pressure. In practice, most messages carry 0–1 attachments well under 1 MB. The per-attachment cap (`casehub.discord.attachment.max-bytes`) bounds individual downloads. A total byte cap across all attachments is not added in V1 — the combination of 100-message limit and per-attachment size enforcement provides sufficient bounding for realistic workloads. If heap pressure becomes a concern, reduce `maxAttachmentBytes` via config.

### Changes

| File | Change |
|------|--------|
| `DiscordChatPlatform.toReceivedMessage()` | Download each `DiscordAttachment` via `client.downloadAttachment()`, collect into `ChatContent.attachments` |

---

## Bug Fix: DiscordInboundTranslator attachment forwarding

### Problem

`DiscordInboundTranslator.translate()` creates `new ChatContent(msg.content())` — the single-arg constructor sets `attachments = List.of()`. The `InboundMessage.attachments()` from the inbound connector (already downloaded with SSRF defense) are discarded.

### Fix

Change to `new ChatContent(msg.content(), null, msg.attachments())`.

### Changes

| File | Change |
|------|--------|
| `DiscordInboundTranslator.translate()` | Pass `msg.attachments()` to `ChatContent` constructor |

---

## Feature 2: Advanced embed MCP parameters (#38)

### Problem

`DiscordMcpTool.sendDiscord()` exposes only `embedTitle`, `embedDescription`, `embedColor`. The `DiscordEmbed` model already supports fields, thumbnailUrl, imageUrl, url, footer, and author — but these aren't accessible from the MCP surface.

### Design

Add six new optional `@ToolArg` parameters to `sendDiscord()`:

| Parameter | Type | Description | Maps to |
|-----------|------|-------------|---------|
| `embedUrl` | String | URL — makes embed title a hyperlink | `DiscordEmbed.url` |
| `embedThumbnailUrl` | String | Small image top-right corner | `DiscordEmbed.thumbnailUrl` |
| `embedImageUrl` | String | Full-width image below description | `DiscordEmbed.imageUrl` |
| `embedFooter` | String | Footer text | `DiscordEmbed.footer.text` |
| `embedAuthor` | String | Author name | `DiscordEmbed.author.name` |
| `embedFields` | String | JSON array of field objects | `DiscordEmbed.fields` |

`embedFields` format: `[{"name":"Field 1","value":"Value 1","inline":true}]`
- Malformed JSON → `"Failed: embedFields must be a JSON array of {name, value, inline} objects"`
- Each object requires `name` and `value`; `inline` defaults to `false` if absent
- `hasEmbed` detection extends to include any of the new parameters being non-blank

**Embed limit validation** (Discord API limits, validated before sending):
- `embedTitle`: max 256 characters → `"Failed: embedTitle exceeds 256 characters"`
- `embedDescription`: max 4096 characters → `"Failed: embedDescription exceeds 4096 characters"`
- `embedFields`: max 25 fields → `"Failed: embedFields exceeds 25 fields"`
- Field `name`: max 256 characters → `"Failed: embedFields[N].name exceeds 256 characters"`
- Field `value`: max 1024 characters → `"Failed: embedFields[N].value exceeds 1024 characters"`
- `embedFooter`: max 2048 characters → `"Failed: embedFooter exceeds 2048 characters"`
- `embedAuthor`: max 256 characters → `"Failed: embedAuthor exceeds 256 characters"`
- Total embed content (title + description + footer + author + all field names + all field values): max 6000 characters → `"Failed: total embed content exceeds 6000 characters"`
- `embedUrl` requires `embedTitle` → `"Failed: embedUrl requires embedTitle"` (Discord's embed URL creates a hyperlink on the title; without a title, the URL is silently ignored)

All existing parameters (`embedTitle`, `embedDescription`, `embedColor`) are unchanged. The `hasEmbed` check expands to include any embed parameter being non-blank.

### Protocols

- `mcp-tool-blocking-annotation`: `@Blocking` already present
- `mcp-tool-exception-catch-all`: top-level try/catch already present

### Changes

| File | Change |
|------|--------|
| `DiscordMcpTool.sendDiscord()` | Add 6 `@ToolArg` parameters, parse `embedFields` JSON, extend `hasEmbed` check, construct full `DiscordEmbed` |

---

## Feature 3: SlackChatPlatform (#40)

### Module structure

New `chat-slack/` module:

```
chat-slack/
  pom.xml
  src/main/java/io/casehub/connectors/chat/slack/
    SlackChatPlatform.java
    SlackInboundTranslator.java
  src/test/java/io/casehub/connectors/chat/slack/
    SlackChatPlatformTest.java
    SlackInboundTranslatorTest.java
```

**Maven coordinates:** `io.casehub:casehub-connectors-chat-slack:0.2-SNAPSHOT`

**Dependencies:**
- `casehub-connectors-core`
- `casehub-connectors-chat-spi`
- `casehub-connectors-slack-bot`
- `quarkus-arc`
- Test: `quarkus-junit`, `assertj-core`, `wiremock-standalone`
- Jandex plugin for CDI discovery

### SlackBotClient expansion

`SlackBotClient` grows from 2 to 14 API methods. All follow the existing pattern: `token` as first parameter, `HttpHelper.CLIENT` for HTTP, typed result records.

| Method | Slack API | Pagination | Used by |
|--------|-----------|------------|---------|
| `listConversations(token)` | `conversations.list` | Yes | Primary paginating method for `conversations.list`. Returns `List<ConversationInfo>`. `listChannels()` delegates to this: `listConversations(token).stream().map(c -> new DiscoveredTarget(c.id(), "#" + c.name())).toList()`. One pagination loop, one `parsePage()`, one maintenance surface. |
| `addReaction(token, channel, timestamp, emoji)` | `reactions.add` | No | Reactions |
| `removeReaction(token, channel, timestamp, emoji)` | `reactions.remove` | No | Reactions |
| `getReactions(token, channel, timestamp)` | `reactions.get` | No | Reactions |
| `getPresence(token, userId)` | `users.getPresence` | No | Presence |
| `listConversationMembers(token, channelId)` | `conversations.members` | Yes | Members |
| `listUsers(token)` | `users.list` | Yes | Members (workspace user batch fetch for display name resolution — avoids N+1 `users.info` calls) |
| `createConversation(token, name, isPrivate)` | `conversations.create` | No | ChannelManagement |
| `getConversationInfo(token, channelId)` | `conversations.info` | No | ChannelManagement |
| `inviteToConversation(token, channelId, userId)` | `conversations.invite` | No | MemberManagement |
| `kickFromConversation(token, channelId, userId)` | `conversations.kick` | No | MemberManagement |
| `getHistory(token, channelId, oldest, limit)` | `conversations.history` | No (limit=100 fits in single page) | MessageHistory |

Paginating methods (`listConversations`, `listConversationMembers`, `listUsers`) follow the `paginating-client-fail-soft` protocol: return partial results + WARNING on mid-loop failure, bounded by `MAX_PAGES`. `getHistory` is not paginated — `limit=100` fits in a single `conversations.history` response.

Result records (inner records on `SlackBotClient`):
- `ReactionResult(boolean ok, String error)` — for add/remove reaction
- `ReactionListResult(boolean ok, List<String> emojis, String error)` — for getReactions
- `PresenceResult(boolean ok, String presence, String error)` — for getPresence
- `ConversationInfo(String id, String name, String topic, String purpose, boolean isPrivate)` — channel detail for Discovery and ChannelManagement
- `ConversationListResult(boolean ok, List<ConversationInfo> conversations, String nextCursor, String error)` — paginated result for listConversations
- `MembersResult(boolean ok, List<String> memberIds, String nextCursor, String error)` — page result
- `UserInfo(String id, String displayName, String realName)` — workspace user entry
- `UserListResult(boolean ok, List<UserInfo> users, String nextCursor, String error)` — paginated result for listUsers
- `ConversationResult(boolean ok, ConversationInfo info, String error)` — for create/info single-channel operations
- `HistoryMessage(String ts, String user, String text, String threadTs)` — message in history. No attachment field — Slack file downloads require separate `files.info` API calls with different authorization (file URLs are not direct CDN links like Discord). Slack attachment downloading is a separate enhancement. Discord attachment downloading (Feature 1) uses `DiscordClient.downloadAttachment()` with direct CDN URLs and existing SSRF defense.
- `HistoryResult(boolean ok, List<HistoryMessage> messages, String error)` — single-page result (no pagination)

### SlackChatPlatform capabilities

All 9 capabilities native — most capable ChatPlatform implementation (Discord: 8, IRC: 3).

| # | Capability | Implementation |
|---|-----------|---------------|
| 1 | **Messaging** | `postMessage(token, channelId, text, null)` → map `PostResult` to `SendResult`: `PostResult.ts()` → `ChatMessageRef(ChatChannelRef(channelId), ts)`, `Instant` from full `ts` string parsed as decimal — split on `"."`, integer part as epoch seconds, fractional part as microseconds (× 1000 → nanos), via `Instant.ofEpochSecond(seconds, nanoAdjustment)`. Preserves Slack's microsecond precision (e.g. `"1234567890.123456"` → `Instant` at 1234567890s + 123456µs). `SendResult.success(messageRef, timestamp)` on `PostResult.ok()`, `SendResult.failure(PostResult.error())` otherwise. |
| 2 | **Threading** | `postMessage(token, channelId, text, parentRef.messageId())` — Slack's `thread_ts` = parent `ts` |
| 3 | **Discovery** | Client layer: `listConversations(token)` → `List<ConversationInfo>`. SPI layer: `SlackChatPlatform` maps `ConversationInfo` → `Channel(ChatChannelRef(c.id()), c.name(), c.topic(), c.purpose(), c.isPrivate())`. The `Discovery` interface returns `List<Channel>` — same pattern as `DiscordChatPlatform` mapping `DiscordChannel` → `Channel`. |
| 4 | **Reactions** | `addReaction/removeReaction/getReactions` — Slack emoji names without colons |
| 5 | **Presence** | `getPresence(token, userId)` → `active` → ONLINE, `away` → AWAY. No DND in basic API. `set()` logs a warning and returns (no-op) — Slack's `users.setPresence` can only set the bot's own presence, not another user's. Same pattern as `DiscordPresence.set()`. |
| 6 | **Members** | `listConversationMembers` → IDs, then `listUsers` → workspace user map (paginated, follows `paginating-client-fail-soft`). Join locally by user ID. Members whose user info is unavailable (page cap reached) use their user ID as displayName. O(pages) HTTP calls instead of O(N). |
| 7 | **ChannelManagement** | `createConversation` for create, `getConversationInfo` for find |
| 8 | **MemberManagement** | `inviteToConversation` for add, `kickFromConversation` for remove |
| 9 | **MessageHistory** | `getHistory(token, channelId, oldest, 100)` → map `HistoryMessage` to `ReceivedMessage`. Limit fixed at 100 (matching Discord). Single API call — `conversations.history` returns up to 1000 per page, so 100 fits in one request and no pagination is needed. `oldest` = full-precision ts-format string from `Instant.since`: `since.getEpochSecond() + "." + String.format("%06d", since.getNano() / 1000)`. Preserves microsecond precision and prevents duplicate messages on sequential history queries. `ts` = messageId, `threadTs` = parentRef. |

### Credential configuration

`SlackChatPlatform` injects `@ConfigProperty(name = "casehub.slack.token", defaultValue = "")`.

Per `credential-config-ownership` protocol: the token is injected by the caller (SlackChatPlatform) and passed to `SlackBotClient` at call time. This parallels Discord: `casehub.discord.token` for ChatPlatform, `casehub.connectors.discord.token` for MCP tool.

If token is blank, all capabilities degrade (same `@PostConstruct` guard pattern as `DiscordChatPlatform`).

**Required bot OAuth scopes:**

| Scope | Required by | Notes |
|-------|-------------|-------|
| `chat:write` | `chat.postMessage` | Already required for existing `SlackBotClient` |
| `channels:read` | `conversations.list`, `conversations.info`, `conversations.members` | Public channels |
| `channels:history` | `conversations.history` | Public channels |
| `channels:manage` | `conversations.create`, `conversations.invite`, `conversations.kick` | Public channels |
| `groups:read` | `conversations.list`, `conversations.info`, `conversations.members` | Private channels |
| `groups:history` | `conversations.history` | Private channels |
| `groups:write` | `conversations.create`, `conversations.invite`, `conversations.kick` | Private channels |
| `reactions:read` | `reactions.get` | |
| `reactions:write` | `reactions.add`, `reactions.remove` | |
| `users:read` | `users.list`, `users.getPresence` | |

Minimum for read-only capabilities (Discovery, Members, Presence, Reactions read, MessageHistory): `channels:read`, `channels:history`, `groups:read`, `groups:history`, `reactions:read`, `users:read`. Full 9-capability support requires all 10 scopes above.

### Slack message identity

Slack `ts` (e.g. `"1234567890.123456"`) serves as both message ID and timestamp. The string is used as-is for `ChatMessageRef.messageId` (identity) and parsed as a decimal for `Instant` (ordering). Full microsecond precision is preserved: split on `"."`, integer part = epoch seconds, fractional part = microseconds × 1000 → nanos.

- `ChatMessageRef(channel, ts)` — message reference (ts as string)
- `thread_ts` → `ChatMessageRef(channel, thread_ts)` as `parentRef` for threaded messages
- `null` parentRef for top-level messages

### SlackInboundTranslator

Implements `InboundTranslator`. Maps Slack webhook `InboundMessage` metadata:

| Metadata key | ChatPlatform mapping |
|------|------|
| `slack-ts` | `ChatMessageRef.messageId` |
| `slack-thread-ts` | parent `ChatMessageRef.messageId` (if present) |
| `msg.externalChannelRef()` | `ChatChannelRef.id` |
| `msg.externalSenderId()` | `MemberRef.id` |
| `msg.attachments()` | `ChatContent.attachments` (forwarded, not dropped) |

`connectorType()` returns `InboundConnectorTypes.SLACK`.

### Platform ID

`SlackChatPlatform.id()` returns `InboundConnectorTypes.SLACK` (`"slack"`), consistent with `DiscordChatPlatform.id()` returning `InboundConnectorTypes.DISCORD`.

### Protocols applied

| Protocol | How |
|----------|-----|
| `shared-http-client` | All new SlackBotClient methods use `HttpHelper.CLIENT` |
| `credential-config-ownership` | Token injected by SlackChatPlatform, passed at call time |
| `spi-id-method-naming` | `id()` not `connectorId()` |
| `paginating-client-fail-soft` | `listConversations`, `listConversationMembers`, `listUsers` return partial + WARNING |
| `inbound-connector-id-constants` | `SlackInboundTranslator` uses `InboundConnectorTypes.SLACK` constant |

---

## Out of scope

- **#37 (rich content model)** — separate concern. MCP tools are platform-specific and correctly bypass the ChatPlatform SPI for platform-native features (Discord embeds, Slack blocks).
- **Slack Block Kit in MCP** — `send_slack_bot` could support blocks, but that's a separate enhancement. Filed as #41.
- **`list_slack_channels` MCP tool** — Slack channels already appear in `list_channels` via `SlackBotDiscovery`. A Slack-specific tool with richer detail is a separate enhancement. Filed as #42.

---

## Test strategy

### #36 — Message history attachments
- Unit test: `DiscordChatPlatform.toReceivedMessage()` with messages carrying attachments → verify `ChatContent.attachments` populated
- Unit test: attachment download failure (null return) → verify graceful skip
- WireMock: mock Discord CDN for download, verify SSRF defense

### Bug fix — Translator attachment forwarding
- Unit test: `DiscordInboundTranslator.translate()` with `InboundMessage` carrying attachments → verify they appear in `ReceivedMessage.content().attachments()`

### #38 — Embed MCP parameters
- Unit test: all six new parameters → verify `DiscordEmbed` construction
- Unit test: `embedFields` JSON parsing — valid, malformed, missing fields
- Unit test: `hasEmbed` detection with only new parameters (no title/description)
- Unit test: embed limit validation — title > 256, description > 4096, > 25 fields, field name > 256, field value > 1024, footer > 2048, author > 256, total > 6000
- Unit test: `embedUrl` without `embedTitle` → `"Failed: embedUrl requires embedTitle"`
- WireMock: full end-to-end with embeds

### #40 — SlackChatPlatform
- WireMock for all new `SlackBotClient` methods — success and error paths
- Paginating methods (`listConversations`, `listConversationMembers`, `listUsers`): multi-page, mid-loop failure, page cap
- Members: workspace user batch fetch with local join, graceful fallback to user ID as displayName when user info unavailable
- `SlackChatPlatform`: all 9 capabilities with mocked `SlackBotClient` responses
- Degraded mode: blank token → all capabilities degrade
- `SlackInboundTranslator`: metadata mapping, attachment forwarding
