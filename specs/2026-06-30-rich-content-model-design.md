# Platform-Agnostic Rich Content Model — Design Spec

**Issues:** #37 (rich content model), #41 (Block Kit support), #42 (list_slack_channels)
**Branch:** issue-37-rich-content-model
**Date:** 2026-06-30

---

## Problem

Discord embeds, Slack Block Kit, and Teams Adaptive Cards are platform-specific rich content formats. The current `ChatContent(text, markdown, attachments)` model carries no structured rich content. Discord embeds are exposed via `DiscordClient` directly, bypassing the ChatPlatform SPI. Slack blocks don't exist in the codebase at all.

MCP tools (`send_slack_bot`, `send_discord`) bypass the ChatPlatform SPI entirely, talking directly to vendor HTTP clients. This creates parallel code paths for the same operations and prevents code sharing.

The `Channel` model lacks member count, and platform-specific discovery tools (`list_discord_channels`) duplicate what the ChatPlatform SPI's `Discovery` capability already provides.

## Design Principles

- **Content, not layout.** The abstraction expresses structured information (title, description, fields, images). Layout decisions (dividers, column sets, block ordering) are rendering concerns for each platform translator.
- **Graceful degradation.** Platforms that can't render rich content fall back to `text`. Fields that don't map (e.g. `color` on Slack) are silently dropped.
- **No interactivity.** Buttons, inputs, and actions are out of scope — this is a delivery library, not an application framework.
- **Text is always required.** Every message has a `text` fallback. Every platform can render text. This is a deliberate capability regression from `send_discord`, which allowed embed-only messages (no text). On Discord, `text` appears above the embed as a separate element — embed-only messages look visually cleaner. The tradeoff is accepted: cross-platform consistency and guaranteed fallback outweigh Discord-specific visual polish.

## Research

No existing library solves cross-platform rich content abstraction. Microsoft Adaptive Cards is the closest standard but isn't adopted by Slack or Discord. Microsoft Bot Framework uses an Activity model with card Attachments and graceful cross-channel degradation — the architectural model we draw from.

The industry has converged on compositional block models (Slack Block Kit, Teams Adaptive Cards, Google Chat Cards V2, Matrix's MSC1767 extensible events). Discord embeds are the outlier with a flat card model. A card model subsumes the compositional model at the content level — a single RichCard expresses the content; each platform renders it using its native block/element primitives.

---

## 1. RichCard Model

Lives in `chat-spi` module, package `io.casehub.connectors.chat.model`. Pure-Java record in a Tier 2 module (`chat-spi` depends on `quarkus-arc` for CDI used by the SPI interfaces; `RichCard` itself has no CDI annotations).

```java
public record RichCard(
    String title,
    String description,
    String url,
    Integer color,
    List<Field> fields,
    String thumbnailUrl,
    String imageUrl,
    String footer,
    String author
) {
    public record Field(String name, String value, boolean inline) {}

    public RichCard {
        if (title == null && description == null) {
            throw new IllegalArgumentException("RichCard requires at least title or description");
        }
        fields = fields == null ? List.of() : List.copyOf(fields);
    }
}
```

**Builder** for readable construction (9 positional parameters with 7 nullable is caller-hostile):

```java
RichCard card = RichCard.builder()
    .title("Deploy Summary")
    .description("3 services updated")
    .color(0x00FF00)
    .fields(List.of(new Field("env", "prod", true)))
    .build();
```

A `Builder` inner class follows the `ChatPlatform.Builder` pattern in the codebase. The canonical record constructor remains available for programmatic translation from platform-specific models.

No platform-specific limit enforcement. Each platform translator validates its own limits and returns `SendResult.failure(...)`.

## 2. ChatContent Extension

```java
public record ChatContent(
    String text,
    String markdown,
    List<Attachment> attachments,
    List<RichCard> cards
) {
    public ChatContent {
        Objects.requireNonNull(text, "text");
        attachments = attachments == null ? List.of() : List.copyOf(attachments);
        cards = cards == null ? List.of() : List.copyOf(cards);
    }

    public ChatContent(final String text) {
        this(text, null, List.of(), List.of());
    }
}
```

Existing callers using `new ChatContent("text")` are unaffected — `cards` defaults to empty.

**Canonical constructor breakage:** The 3-arg canonical constructor `(text, markdown, attachments)` becomes a 4-arg constructor `(text, markdown, attachments, cards)`. This breaks ~18 call sites (5 production, 13 test). Per platform design philosophy, all call sites are updated — no compatibility shim. Affected production files:
- `RefInboundTranslator.java`, `IrcInboundTranslator.java`, `DiscordInboundTranslator.java`, `SlackInboundTranslator.java` — add `List.of()` for cards
- `DiscordChatPlatform.java` (line 341) — add `List.of()` for cards

Affected test files: `ChatContentTest`, `IrcChatPlatformTest`, `SlackChatPlatformTest`, `DiscordChatPlatformTest` — mechanical migration, add `List.of()` as 4th argument.

**Rendering contract:**
- `cards` empty → send text/markdown as today
- `cards` non-empty → send both text and cards; text serves as notification fallback (Slack uses `text` for push notifications when blocks are present)

## 3. Channel Model Enrichment

```java
public record Channel(
    ChatChannelRef ref,
    String name,
    String topic,
    String description,
    boolean isPrivate,
    Integer memberCount
) {
    public Channel(ChatChannelRef ref, String name, String topic,
                   String description, boolean isPrivate) {
        this(ref, name, topic, description, isPrivate, null);
    }
}
```

`Integer` (nullable) — `null` means "not available" (IRC, platforms without count support).

**Data sources:**
- Slack: `conversations.list` supports `num_members` — pass `include_num_members=true` query parameter in `SlackBotClient.listConversations`. Requires adding `numMembers` (nullable `Integer`) to `SlackBotClient.ConversationInfo` record (currently `(id, name, topic, purpose, isPrivate)` — becomes `(id, name, topic, purpose, isPrivate, numMembers)`)
- Discord: `approximate_member_count` from guild endpoint (`?with_counts=true`) — cheap single call, no pagination
- IRC/Ref: `null`

## 4. Platform Translators

### DiscordChatPlatform — RichCard → DiscordEmbed

Near 1:1 mapping:

| RichCard | DiscordEmbed |
|---|---|
| `title` | `title` |
| `description` | `description` |
| `url` | `url` |
| `color` | `color` |
| `fields` | `fields` (Field → DiscordEmbed.Field) |
| `thumbnailUrl` | `thumbnailUrl` |
| `imageUrl` | `imageUrl` |
| `footer` | `footer` (→ DiscordEmbed.Footer) |
| `author` | `author` (→ DiscordEmbed.Author) |

Discord-specific limits enforced before sending: title 256, description 4096, footer 2048, author 256, fields 25, total 6000 chars. Multiple RichCards → multiple embeds (up to 10).

**Cross-platform validity note:** Discord requires `title` when `url` is set — a RichCard with `url` + `description` but no `title` returns `SendResult.failure(...)` on Discord while succeeding on Slack (where `url` is embedded as a mrkdwn link in the description section). This is an inherent platform constraint, not a RichCard validation gap — the RichCard only requires at least one of `title`/`description`.

### SlackChatPlatform — RichCard → Block Kit JSON

| RichCard | Block Kit |
|---|---|
| `title` | Header block (`plain_text`) |
| `description` | Section block (`mrkdwn` text) |
| `url` | Bare mrkdwn link in description section (e.g. `<url>`) — dropped if no `description`. Title is not repeated in the link text since it already appears as a Header block above. |
| `thumbnailUrl` | Section accessory image (on description section) |
| `fields` | Section block with fields array (`mrkdwn` text objects) |
| `imageUrl` | Image block |
| `footer` + `author` | Context block with text elements |
| `color` | Dropped silently (Block Kit has no color support) |

Multiple RichCards separated by divider blocks.

Requires `SlackBotClient.postMessage` to accept Block Kit JSON — see §5.

### IrcChatPlatform — graceful degradation

Cards ignored. `text` sent as-is. No change.

### RefChatPlatform — stores cards

`ChatContent` stored including cards in the in-memory backend. Tests can assert on card content.

## 5. SlackBotClient Block Kit Support (#41)

New overload:

```java
public PostResult postMessage(String token, String channelId,
                              String text, String threadTs,
                              String blocksJson)
```

`blocksJson` is a raw JSON string — the serialized Block Kit array. `SlackBotClient` is a thin HTTP client; it doesn't know about `RichCard`. It adds `"blocks": <blocksJson>` to the payload when non-null.

Existing 4-parameter `postMessage` delegates with `blocksJson = null` — no breaking change for existing callers (`SlackChannelBackend` in casehub-qhorus).

Block Kit serialization lives in `SlackChatPlatform` — a private `serializeToBlocks(List<RichCard>)` method builds the JSON string using `jakarta.json`.

Payload example:
```json
{
  "channel": "C123",
  "text": "Fallback text",
  "blocks": [
    {"type": "header", "text": {"type": "plain_text", "text": "Title"}},
    {"type": "section", "text": {"type": "mrkdwn", "text": "Description"}},
    {"type": "section", "fields": [
      {"type": "mrkdwn", "text": "*Name*\nValue"}
    ]}
  ]
}
```

## 6. MCP Tool Consolidation

### New: `ChatPlatformMcpTool`

Single `@ApplicationScoped` class with two `@Tool @Blocking` methods, injecting `ChatPlatformService` and `ConnectorMeshBridge`. `@Blocking` is required per protocol PP-20260609-e3a2bd — platform translators call `SlackBotClient`/`DiscordClient` which use synchronous `HttpClient.send()`.

**`send_chat`** — replaces `send_slack_bot` and `send_discord`:

Parameters:
- `platform` — required: "slack", "discord", "irc", "ref"
- `channel` — required: channel ID
- `text` — required: message text / notification fallback
- `parentMessageId` — optional: thread reply (Slack ts, Discord message ID)
- `cardTitle` — optional: RichCard title
- `cardDescription` — optional: RichCard description
- `cardColor` — optional: RGB decimal integer
- `cardUrl` — optional: title hyperlink
- `cardThumbnailUrl` — optional: small image
- `cardImageUrl` — optional: full-width image
- `cardFooter` — optional: footer text
- `cardAuthor` — optional: author text
- `cardFields` — optional: JSON array `[{"name":"...","value":"...","inline":true}]`

Implementation:
1. `ChatPlatformService.platform(platform)` — look up platform
2. Build `ChatContent` — if any card parameter is non-null, construct `RichCard`, add to `cards`
3. If `parentMessageId` set → `platform.threading().reply(parentRef, content)`; else → `platform.messaging().send(channelRef, content)`
4. Check `SendResult` — return `"Sent to <channel> (messageId=<id>)"` on success or `"Failed: <reason>"` on failure
5. `meshBridge.notifyDelivered(platform.id(), channel, McpContentSanitizer.sanitize(text))` on success — uses `ChatPlatform.id()` which returns `InboundConnectorTypes` values (`"slack"`, `"discord"`). Note: for Slack, this changes the bridge ID from `"slack-bot"` (used by `SlackBotMcpTool`) to `"slack"` — the platform type, not the delivery mechanism. This is consistent with the Connector SPI tools which use the connector type string.

**`list_chat_channels`** — replaces `list_discord_channels`, satisfies #42:

Parameters:
- `platform` — required: "slack", "discord", etc.

Implementation:
1. `ChatPlatformService.platform(platform)` — look up platform
2. `platform.discovery().listChannels()`
3. Format `Channel` list: name, ID, topic, description, private flag, member count

### Removed

- `SlackBotMcpTool` (class deleted)
- `DiscordMcpTool` (class deleted — both `send_discord` and `list_discord_channels` replaced)

### Unchanged

`send_slack`, `send_teams`, `send_sms`, `send_whatsapp`, `send_email`, `list_channels` — these use the Connector SPI, a different abstraction layer.

**Tool replacement — `send_chat` replaces `send_slack_bot` and `send_discord`:**
`send_chat` replaces both platform-specific tools immediately. `SlackBotMcpTool` and `DiscordMcpTool` are deleted — no deprecation period. This platform has no external users; a clean break is better than maintaining parallel tools. LLM system prompts are updated to use `send_chat` exclusively.

`send_slack` (webhook, Connector SPI) is unaffected — it serves a different use case: webhook-based delivery without a bot token. No overlap with `send_chat`.

### Module dependency change

`mcp/pom.xml`:
- **Add:** `casehub-connectors-chat-slack`, `casehub-connectors-chat-discord`
- **Remove:** `casehub-connectors-slack-bot`, `casehub-connectors-discord` (now transitive via `chat-slack` and `chat-discord`)

`chat-spi` is not listed explicitly — it is a transitive dependency of both implementation modules. `ChatPlatformService(@All List<ChatPlatform>)` discovers `SlackChatPlatform` and `DiscordChatPlatform` via CDI because their modules are on the classpath.

## 7. Deferred Concerns

| Concern | Issue | Why deferred |
|---|---|---|
| Inbound rich content parsing | #44 | Outbound is the primary use case; no current consumer reads rich content from inbound messages |
| Teams ChatPlatform implementation | #45 | Requires Teams Bot registration and API client; webhook connector stays for now. **Forward-compatibility assessment:** RichCard's fields map naturally to Adaptive Card elements — `title` → TextBlock (large), `description` → TextBlock, `fields` → FactSet, `imageUrl`/`thumbnailUrl` → Image, `footer`/`author` → TextBlock (subtle). `color` has no direct equivalent (Adaptive Cards use container styles). No RichCard schema changes expected for #45. |
| Multiple cards per `send_chat` | #46 | Single card covers the primary LLM use case. Multi-card translation logic (Discord 10-embed limit, Slack divider insertion) is implemented in platform translators and unit-tested at the SPI level; not exercisable via MCP until #46 adds a `cards` JSON array parameter. |
| Chat demo UI | #28 (reopened) | Blocked on casehub-pages WebSocket provider |

## 8. Impact Summary

### Files created
- `chat-spi/.../model/RichCard.java`
- `mcp/.../ChatPlatformMcpTool.java`

### Files modified
- `chat-spi/.../model/ChatContent.java` — add `cards` field
- `chat-spi/.../model/Channel.java` — add `memberCount` field
- `chat-discord/.../DiscordChatPlatform.java` — RichCard → DiscordEmbed translation
- `chat-slack/.../SlackChatPlatform.java` — RichCard → Block Kit translation
- `chat-ref/.../RefChatPlatform.java` — pass cards through to backend
- `slack-bot/.../SlackBotClient.java` — add `blocksJson` parameter overload; add `numMembers` to `ConversationInfo`; add `include_num_members=true` to `listConversations` API call
- `mcp/pom.xml` — dependency swap

### Files deleted
- `mcp/.../SlackBotMcpTool.java`
- `mcp/.../DiscordMcpTool.java`

### Tests
- `RichCard` validation tests (title/description requirement, field copying, builder)
- `ChatContent` backward compatibility (existing constructor still works)
- `Channel` backward compatibility (5-param constructor still works)
- `DiscordChatPlatform` — send with cards, limit enforcement, multiple cards
- `SlackChatPlatform` — send with cards, Block Kit JSON structure verification
- `SlackBotClient` — postMessage with blocks parameter
- `ChatPlatformMcpTool` — send_chat routing, list_chat_channels formatting
- `RefChatPlatform` — cards round-trip through in-memory backend
