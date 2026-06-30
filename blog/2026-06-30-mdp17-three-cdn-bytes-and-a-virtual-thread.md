---
layout: post
title: "Three CDN bytes and a virtual thread"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [discord, mcp, attachments, embeds]
series: issue-30-discord-enhancements
---

Discord attachments, embeds, and MCP tools — three issues, one branch. The attachment work is where the interesting decisions live.

## The heap you didn't ask for

The first implementation used `BodyHandlers.ofByteArray()` to download CDN attachments. Standard approach — GET the URL, get a `byte[]` back, check the length. The problem: `ofByteArray()` materializes the *entire* response body into heap before `send()` returns. A Discord Nitro user can attach 500MB files. That's a 500MB `byte[]` allocation with no opportunity to abort mid-stream.

The fix is `BodyHandlers.ofInputStream()` — read in 8KB chunks, count bytes, abort when the configurable limit (default 8MB) is exceeded. `Content-Length` pre-check is a fast-fail optimization for the common case, but chunked transfer encoding omits the header entirely, so streaming enforcement is the primary gate.

This is the kind of thing where the common approach (`ofByteArray` → check length) looks obviously correct and silently isn't.

## Event loop, meet blocking I/O

The Gateway listener runs on the Vert.x WebSocket event loop thread. `HttpHelper.CLIENT.send()` is a blocking call. Downloading attachments on the event loop would stall heartbeat processing and eventually cause a gateway disconnection — Discord interprets missed heartbeats as a dead client.

The fix: when a MESSAGE_CREATE event has attachments, offload to a virtual thread. Messages without attachments stay on the event loop — no overhead for the common case. The virtual thread downloads each attachment, collects successes and failures, then delivers the `InboundMessage` with metadata distinguishing "no attachments" from "all downloads failed" (`discord-attachment-count` and `discord-attachment-download-failures`).

## SSRF on trusted payloads

The attachment URLs come from Discord's authenticated WebSocket — they're Discord CDN addresses. But URL validation is cheap and defense-in-depth is the right call. The download method validates that the host is `cdn.discordapp.com` or `media.discordapp.net` before proceeding. CDN URLs include signed expiry parameters; a 403 is treated as non-retryable.

## Embeds and MCP tools

Rich embeds are outbound-only — a `DiscordEmbed` record with nested `Field`, `Footer`, `Author` types, serialized manually to match the existing `ObjectNode` pattern throughout `DiscordClient`. The existing `sendMessage` and `sendReply` methods delegate to new overloads with `List.of()`.

The MCP tools (`send_discord`, `list_discord_channels`) follow the `SlackBotMcpTool` pattern exactly — own credential config property, `@Blocking`, mesh bridge notification on success.

The embed model supports the full Discord field set. The MCP tool exposes three parameters (title, description, color) — enough for the common case, with the full model accessible to callers using `DiscordClient` directly.
