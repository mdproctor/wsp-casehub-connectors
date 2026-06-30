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
| `listConversations(token)` | `conversations.list` | Yes | Discovery (returns full channel detail, not just `DiscoveredTarget`) |
| `addReaction(token, channel, timestamp, emoji)` | `reactions.add` | No | Reactions |
| `removeReaction(token, channel, timestamp, emoji)` | `reactions.remove` | No | Reactions |
| `getReactions(token, channel, timestamp)` | `reactions.get` | No | Reactions |
| `getPresence(token, userId)` | `users.getPresence` | No | Presence |
| `listConversationMembers(token, channelId)` | `conversations.members` | Yes | Members |
| `getUserInfo(token, userId)` | `users.info` | No | Members |
| `createConversation(token, name, isPrivate)` | `conversations.create` | No | ChannelManagement |
| `getConversationInfo(token, channelId)` | `conversations.info` | No | ChannelManagement |
| `inviteToConversation(token, channelId, userId)` | `conversations.invite` | No | MemberManagement |
| `kickFromConversation(token, channelId, userId)` | `conversations.kick` | No | MemberManagement |
| `getHistory(token, channelId, oldest, limit)` | `conversations.history` | Yes | MessageHistory |

Paginating methods (`listConversationMembers`, `getHistory`) follow the `paginating-client-fail-soft` protocol: return partial results + WARNING on mid-loop failure, bounded by `MAX_PAGES`.

Result records (inner records on `SlackBotClient`):
- `ReactionResult(boolean ok, String error)` — for add/remove reaction
- `ReactionListResult(boolean ok, List<String> emojis, String error)` — for getReactions
- `PresenceResult(boolean ok, String presence, String error)` — for getPresence
- `ConversationInfo(String id, String name, String topic, String purpose, boolean isPrivate)` — channel detail for Discovery and ChannelManagement
- `ConversationListResult(boolean ok, List<ConversationInfo> conversations, String nextCursor, String error)` — paginated result for listConversations
- `MembersResult(boolean ok, List<String> memberIds, String nextCursor, String error)` — page result
- `UserInfoResult(boolean ok, String userId, String displayName, String realName, String error)` — for getUserInfo
- `ConversationResult(boolean ok, ConversationInfo info, String error)` — for create/info single-channel operations
- `HistoryMessage(String ts, String user, String text, String threadTs)` — message in history
- `HistoryResult(boolean ok, List<HistoryMessage> messages, String nextCursor, String error)` — page result

### SlackChatPlatform capabilities

All 9 capabilities native — most capable ChatPlatform implementation (Discord: 8, IRC: 3).

| # | Capability | Implementation |
|---|-----------|---------------|
| 1 | **Messaging** | `postMessage(token, channelId, text, null)` → map `PostResult` to `SendResult` |
| 2 | **Threading** | `postMessage(token, channelId, text, parentRef.messageId())` — Slack's `thread_ts` = parent `ts` |
| 3 | **Discovery** | `listConversations(token)` → returns `List<ConversationInfo>` with id, name, topic, purpose, isPrivate. Separate from existing `listChannels()` which returns `DiscoveredTarget` for the `ConnectorDiscovery` SPI. Both call `conversations.list` but parse different fields — they serve different layers (Chat SPI vs Connector SPI) |
| 4 | **Reactions** | `addReaction/removeReaction/getReactions` — Slack emoji names without colons |
| 5 | **Presence** | `getPresence(token, userId)` → `active` → ONLINE, `away` → AWAY. No DND in basic API |
| 6 | **Members** | `listConversationMembers` → IDs, then `getUserInfo` per ID for display names. N+1 queries are acceptable here — member lists are small and cached by Slack |
| 7 | **ChannelManagement** | `createConversation` for create, `getConversationInfo` for find |
| 8 | **MemberManagement** | `inviteToConversation` for add, `kickFromConversation` for remove |
| 9 | **MessageHistory** | `getHistory` → map `HistoryMessage` to `ReceivedMessage`. `oldest` = epoch seconds from `Instant.since`. `ts` = messageId, `threadTs` = parentRef |

### Credential configuration

`SlackChatPlatform` injects `@ConfigProperty(name = "casehub.slack.token", defaultValue = "")`.

Per `credential-config-ownership` protocol: the token is injected by the caller (SlackChatPlatform) and passed to `SlackBotClient` at call time. This parallels Discord: `casehub.discord.token` for ChatPlatform, `casehub.connectors.discord.token` for MCP tool.

If token is blank, all capabilities degrade (same `@PostConstruct` guard pattern as `DiscordChatPlatform`).

### Slack message identity

Slack `ts` (e.g. `"1234567890.123456"`) serves as the message ID.

- `ChatMessageRef(channel, ts)` — message reference
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
| `paginating-client-fail-soft` | `listConversationMembers`, `getHistory` return partial + WARNING |
| `inbound-connector-id-constants` | `SlackInboundTranslator` uses `InboundConnectorTypes.SLACK` constant |

---

## Out of scope

- **#37 (rich content model)** — separate concern. MCP tools are platform-specific and correctly bypass the ChatPlatform SPI for platform-native features (Discord embeds, Slack blocks).
- **Slack Block Kit in MCP** — `send_slack_bot` could support blocks, but that's a separate enhancement.
- **`list_slack_channels` MCP tool** — Slack channels already appear in `list_channels` via `SlackBotDiscovery`. A Slack-specific tool with richer detail is a separate issue.

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
- WireMock: full end-to-end with embeds

### #40 — SlackChatPlatform
- WireMock for all 11 new `SlackBotClient` methods — success and error paths
- Paginating methods: multi-page, mid-loop failure, page cap
- `SlackChatPlatform`: all 9 capabilities with mocked `SlackBotClient` responses
- Degraded mode: blank token → all capabilities degrade
- `SlackInboundTranslator`: metadata mapping, attachment forwarding
