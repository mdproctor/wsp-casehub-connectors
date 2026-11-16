# HANDOFF — casehub-connectors

## Last Session

Four Teams issues landed in one session, bringing Teams from 8 to 9 native capabilities and closing the gap between "text in channels" and a proper bi-directional chat platform.

**Key architectural change:** `TeamsChatPlatform` now routes outbound messages through two different APIs based on the channel reference prefix. Team channels (`teamId:channelId`) go through `TeamsClient` → Graph API with application tokens. Personal channels (`personal:conversationId`) go through `BotFrameworkClient` → Bot Framework REST v3 with bot tokens. The routing decision is a single `isPersonalChannel()` check in `sendMessage`/`replyToMessage`.

**Reactions (#232):** Graph API `setReaction`/`unsetReaction` on messages, plus `getMessage` to read the `reactions` array for `list()`. Unlike Slack/Discord which use emoji names, Teams uses a fixed set of reaction types (like, heart, laugh, surprised, sad, angry).

**HTML-to-markdown (#238):** Bot Framework delivers HTML content (`<b>`, `<i>`, `<at>`, `<a href>`). `TeamsInboundTranslator` now converts to both markdown and plain text for `ChatContent(text, markdown)`. Entity decode ordering fix applied — `&amp;` decoded last to prevent double-decode of `&amp;lt;`.

**Attachment download (#233):** `BotFrameworkClient.downloadAttachment` does an authenticated GET on the activity's `contentUrl`. Failed downloads are logged and skipped — partial delivery preferred over failing the message.

**Personal chat outbound (#237):** `withBotClient()` wires `BotFrameworkClient` into `TeamsChatPlatform`. `TeamsBeans` creates the client when `casehub.connectors.teams-bot.default-service-url` is configured. Currently sends plain text only — rich content is a known gap (#248).

**Branch:** `issue-237-teams-personal-chat-outbound` — covers all four issues (landed sequentially on main before branching for #237)

**Issues closed this session:**

| # | Title | What changed |
|---|-------|-------------|
| #232 | Outbound reactions via Graph API | TeamsClient setReaction/unsetReaction/getMessage, TeamsReactions inner class, 9th capability |
| #238 | Inbound HTML to markdown conversion | TeamsInboundTranslator htmlToMarkdown/htmlToPlainText/containsHtml |
| #233 | Attachment download from Bot Framework | BotFrameworkClient.downloadAttachment, TeamsInboundConnector.downloadAttachments |
| #237 | Personal 1:1 chat outbound messaging | TeamsChatPlatform.withBotClient, isPersonalChannel routing, sendPersonalMessage |

**Issues filed this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #246 | SSRF defense on BotFrameworkClient attachment download | XS | Low |
| #247 | Attachment download size limit | XS | Low |
| #248 | Personal channel sends lose rich content (markdown, cards, attachments) | S | Med |
| #249 | Personal send returns empty messageId in SendResult | XS | Low |
| #250 | Nested HTML tags not handled in htmlToMarkdown | S | Med |
| #251 | Personal channel refs crash listMembers, reactions, messageHistory | XS | Low |

**Queue state after this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #234 | Proactive messaging bot-initiated conversations | M | High |
| #235 | Adaptive Card action handling invoke activities | M | High |
| #236 | Messaging extensions support | L | High |

**To resume:** `work continue` for #234 (next in queue).

**Configuration prerequisite for personal chat outbound:** Set `casehub.connectors.teams-bot.default-service-url` (e.g. `https://smba.trafficmanager.net/teams/`). Without it, personal sends throw `UnsupportedOperationException`. The existing `service-url-allowlist` applies to the BotFrameworkClient for SSRF validation on outbound calls.
