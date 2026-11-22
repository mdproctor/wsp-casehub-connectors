# Decisions — #212 Telegram ChatPlatform

## D1: HTTP client strategy

**Choice:** Thin hand-written HTTP client (`TelegramBotClient`) using `HttpHelper.CLIENT`
**Alternatives:**
- `telegrambots` SDK — richer type safety but brings its own HTTP client, threading model, and webhook handling that conflict with project conventions (HttpHelper.CLIENT, call-time credentials, existing webhook router)
**Rationale:** The Telegram Bot API is a simple REST API that maps cleanly to the existing thin client pattern used by signal-cli, discord, and slack-bot. Using the SDK would violate PP-20260607-9794cb (HttpHelper.CLIENT) and PP-20260609-0c3e24 (credential config ownership).
**Trade-offs:** No auto-generated model types from the SDK — API response records must be hand-written. But the API surface needed is small (~10 methods).
**Sources:** signal-cli/SignalClient.java, docs/protocols/connectors/shared-http-client.md, docs/protocols/connectors/credential-config-ownership.md
**Exploration:** quick
**Status:** captured

## D2: Inbound strategy

**Choice:** Webhook mode via `WebhookInboundConnector`
**Alternatives:**
- Long polling via `InboundConnector` — simpler for dev (no public URL) but adds background thread management, doesn't scale horizontally
- Both (webhook primary, long-polling fallback) — more code, covers both deployment scenarios but unnecessary complexity
**Rationale:** Webhook is the production standard for Telegram bots. Maps directly to the existing `WebhookInboundConnector` SPI. Telegram POSTs JSON updates to `/connectors/telegram-inbound/webhook`. No background threads, works behind load balancers.
**Trade-offs:** Requires a publicly reachable HTTPS URL for Telegram to POST to. Local dev needs ngrok or similar tunnel. But this is true for all webhook-based connectors (Slack, Teams, WhatsApp).
**Sources:** connectors-api/WebhookInboundConnector.java, core.telegram.org/bots/api (setWebhook)
**Exploration:** quick
**Status:** captured

## D3: Capability scope

**Choice:** 7 native capabilities — Messaging, Threading, Discovery, Reactions, Commands, Members, Presence
**Alternatives:**
- Minimum 3 (Messaging, Reactions, Commands) — matches chat-irc level, defers everything else
- 5 (add Threading, Discovery) — low-cost additions but leaves Members/Presence out
**Rationale:** User preference for full coverage. All 7 map directly to Telegram Bot API methods with no complex additional infrastructure. Approaches chat-discord (9) level.
**Trade-offs:** More API surface to implement and test than the minimum 3. ChannelManagement, MemberManagement, and MessageHistory remain degraded — bots can't create groups via Bot API, member management is limited, and there's no message history retrieval API.
**Sources:** core.telegram.org/bots/api, chat-signal/SignalChatPlatform.java (6 native), chat-discord/DiscordChatPlatform.java (9 native)
**Exploration:** quick
**Status:** captured

## D4: Webhook verification

**Choice:** Verify using `X-Telegram-Bot-Api-Secret-Token` header
**Alternatives:**
- No verification — accept all POSTs, insecure
**Rationale:** Consistent with Slack (HMAC-SHA256) and Discord (Ed25519) verification patterns. Telegram's built-in mechanism: compare header value against configured secret token (set via setWebhook's `secret_token` parameter). Prevents spoofed webhook calls.
**Trade-offs:** None significant — adds one string comparison per request.
**Sources:** core.telegram.org/bots/api (setWebhook secret_token parameter), chat-discord/DiscordInteractionEndpoint.java
**Exploration:** quick
**Status:** captured

## D5: Module structure

**Choice:** Two modules — `telegram-bot` (shared HTTP client) + `chat-telegram` (ChatPlatform + inbound)
**Alternatives:**
- Single module `chat-telegram` — simpler but breaks established separation; future consumers must depend on chat-spi
- Three modules (separate `telegram-inbound`) — overkill; Telegram webhook is simple unlike Gmail push (Pub/Sub lifecycle)
**Rationale:** Follows the established pattern exactly: slack-bot/chat-slack, discord/chat-discord, signal-cli/chat-signal. The HTTP client is reusable by other consumers (MCP tools, Qhorus backends) without pulling in chat-spi.
**Trade-offs:** Two POMs to maintain instead of one. Marginal overhead for the structural benefit.
**Sources:** chat-signal/pom.xml, chat-discord/pom.xml, signal-cli module structure
**Exploration:** quick
**Status:** captured
