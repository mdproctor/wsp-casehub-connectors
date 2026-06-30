---
layout: post
title: "Nine out of nine and one isNull that wasn't"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [slack, discord, chat-spi, mcp, attachments]
---

Three features on one branch again. Discord needed two fixes — message history wasn't downloading attachments, and the inbound translator was silently dropping them. The MCP tool needed the rest of the embed surface exposed. And Slack needed a ChatPlatform.

## The attachment gap nobody noticed

`DiscordChatPlatform.toReceivedMessage()` was constructing `ChatContent` with `List.of()` for attachments even though `DiscordMessage` already carried the parsed metadata. The `downloadAttachment` infrastructure — SSRF defense, streaming size cap, CDN allowlisting — was right there in `DiscordClient`. It just wasn't wired in.

More interesting: `DiscordInboundTranslator.translate()` was doing the same thing. The inbound connector downloads attachments on virtual threads, carefully tracking successes and failures in metadata. Then the translator throws them away with `new ChatContent(msg.content())` — the single-arg constructor defaults attachments to `List.of()`. Every attachment the Gateway path downloaded was invisible to anything using the ChatPlatform SPI.

The fix is one line in the translator, and a loop in `toReceivedMessage()` that calls `downloadAttachment` for each entry. Failed downloads return null and get skipped — same pattern as the inbound path, minus the virtual threads. Message history callers are already blocking; adding parallelism for ≤100 messages adds complexity without meaningful gain.

## Discord embeds: validate locally or get a 50035

The embed MCP work was straightforward — six new `@ToolArg` parameters mapping to fields `DiscordEmbed` already supports. The interesting part was validation. Discord's API rejects oversized embeds with a 50035 error and nested field paths that no caller can interpret. Validating locally — 256-char title, 4096-char description, 25 fields max, 6000-char total — gives a clear `"Failed: embedTitle exceeds 256 characters"` instead.

## Slack gets all nine

The ChatPlatform SPI has nine capabilities. IRC implements three. Discord implements eight — everything except `MemberManagement`. Slack can do all nine.

The architecture mirrors Discord: `SlackBotClient` is the shared HTTP client (now with 14 methods, up from 2), `SlackChatPlatform` is the CDI bean mapping SPI calls to client calls. `SlackInboundTranslator` converts webhook events to `ReceivedMessage` — and forwards attachments, learning from the Discord bug.

The `listChannels` refactoring is worth noting. The existing method returned `List<DiscoveredTarget>` — just id and name, designed for the `ConnectorDiscovery` SPI. The `Discovery` capability needs `Channel` with topic, purpose, and isPrivate. Rather than duplicating the pagination loop, `listConversations` became the primary method and `listChannels` delegates to it.

For member listing, the spec originally said "N+1 queries are acceptable — member lists are small and cached by Slack." The design review caught this — Slack channels in active workspaces regularly have hundreds of members. The fix: `listUsers` fetches the workspace user directory in pages, and members are joined locally by ID. Members whose info isn't in the batch get their user ID as the display name.

Slack's `ts` — `"1234567890.123456"` — serves as both message ID and timestamp. The microsecond precision matters: two messages posted within the same second would get identical `Instant` values if you parse only the integer part. The implementation splits on `"."`, takes the integer part as epoch seconds and the fractional part as microseconds times 1000 for nanos.

The Jakarta JSON `isNull` NPE surfaced during integration — `JsonObject.isNull("thread_ts")` throws when the key is entirely absent. Slack omits `thread_ts` for non-threaded messages rather than sending null. The method name strongly implies "is this null or missing?" but the implementation is `get(key).equals(JsonValue.NULL)`, and `get` returns null for absent keys. The fix is `containsKey(key) && !isNull(key)`, which is the correct idiom for Jakarta JSON but nothing in the API guides you there.
