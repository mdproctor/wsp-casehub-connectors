---
layout: post
title: "Telegram Joins the Chat"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [telegram, chat-platform, bot-api, connectors]
series: issue-337-seed-loader-test
---

# Telegram Joins the Chat

Telegram is the second most deployed integration on n8n at 951K workflow deployments — behind only Google Sheets. We had no coverage for it. That gap closed today with `chat-telegram`, the seventh ChatPlatform provider.

## The Two-Module Pattern Holds

Every chat provider in connectors follows the same structure: a shared HTTP client module (`slack-bot`, `discord`, `signal-cli`) paired with a ChatPlatform implementation module (`chat-slack`, `chat-discord`, `chat-signal`). The HTTP client wraps the platform's API using `HttpHelper.CLIENT` with call-time token passing. The ChatPlatform wires those API calls into the SPI's capability interfaces.

Telegram follows the pattern exactly: `telegram-bot` provides `TelegramBotClient` (10 Bot API methods), and `chat-telegram` provides `TelegramChatPlatform` with 5 native capabilities.

The issue originally suggested using the `telegrambots` Java SDK. We dropped that early in design — the SDK brings its own HTTP client, threading model, and webhook server, all of which conflict with established protocols. The Bot API is a simple REST interface: `POST https://api.telegram.org/bot{token}/{method}` with a JSON body. A thin client over `HttpHelper.CLIENT` is a few hundred lines and stays consistent with every other provider.

## Five Capabilities, Honestly Claimed

The Bot API maps more naturally to ChatPlatform than I expected. Five native capabilities:

| Capability | Bot API method | Notes |
|------------|---------------|-------|
| Messaging | `sendMessage` | Text with optional `reply_markup` |
| Threading | `sendMessage` + `reply_to_message_id` | Native reply threading, not a channel-fallback |
| Discovery | `getChat` + `getChatMemberCount` | Cache-driven — see below |
| Reactions | `setMessageReaction` | Emoji reactions; `list()` returns empty (API doesn't expose them) |
| Members | `getChatAdministrators` | Returns admins; individual lookups via `getChatMember` |

The initial design claimed 7 capabilities including Presence and Commands. Code review caught that both were stubs — `Presence` returned `UNKNOWN` for everything (identical to the degraded fallback), and `Commands` was `NoOpCommands`. Claiming them as "native" when the implementation was indistinguishable from degraded mode was dishonest signalling. Both are achievable via the Bot API (`getChatMember` status for Presence, `setMyCommands` for Commands) and filed as follow-up issues.

Five capabilities are degraded: Presence, Commands, ChannelManagement (bots can't create groups), MemberManagement (limited ban/unban doesn't map to add/remove), and MessageHistory (no retrieval API).

## The Discovery Problem

The one genuinely interesting constraint: Telegram Bot API has no "list all chats" endpoint. Bots are passive — they learn about chats only when they receive messages. Signal has a similar limitation, but its contacts/groups API at least returns previously seen entities.

The solution is a `ConcurrentHashMap<String, TelegramChat>` populated by the inbound connector on each webhook update. When the inbound connector receives a message, it calls `registerKnownChat()` on the platform, and Discovery returns those cached chats refreshed via `getChat`. It's ephemeral — lost on restart — which is acceptable for now. A persistent cache can follow if the limitation bites.

## Webhook Inbound with Secret Token

Telegram's webhook verification is refreshingly simple compared to Slack's HMAC-SHA256 or Discord's Ed25519. You set a `secret_token` when calling `setWebhook`, and Telegram includes it as the `X-Telegram-Bot-Api-Secret-Token` header on every update POST. The connector compares the header value against the configured secret — one string comparison.

The `TelegramInboundConnector` extends `WebhookInboundConnector`, parses the `TelegramUpdate` JSON, extracts the message (supporting regular messages, edited messages, and callback queries), and fires an `InboundMessage`. Reply-to metadata flows through for threading support in the `TelegramInboundTranslator`.

## Live Validation

We tested against the real Telegram API using a bot created via BotFather. Four live tests: send message, get chat info, set reaction, and threaded reply. All four delivered to a real Telegram client. The live test is env-var-gated (`@EnabledIfEnvironmentVariable`) so it skips in CI — it's there for developers who want to verify against the real API with their own bot token.

The gap analysis identified Telegram as priority one for ChatPlatform coverage. With this implementation, casehub covers the top two n8n integrations (Google Sheets is an OpenAPI dynamic step, not a ChatPlatform) and 7 of the top 10 overall.
