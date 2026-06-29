---
title: "Discord, and What the SPI Survived"
date: 2026-06-29
sequence: 15
tags: [discord, chat-platform, spi-design, gateway, websocket]
---

# Discord, and What the SPI Survived

IRC gave us three native capabilities. The reference implementation gave us nine, but against an in-memory backend — no network, no protocol, no real platform quirks. Discord was the real test: a production API with native support for almost everything the ChatPlatform SPI exposes, plus a WebSocket Gateway for real-time events. If the SPI was going to break, Discord would be the one to break it.

It didn't.

## The Capability Map

Discord supports eight of nine capabilities natively. The mapping was cleaner than I expected:

- **Messaging and Threading** map directly. Discord's inline message replies (`message_reference` field) are exactly what the SPI's `Threading.reply()` models. Discord also has threads-as-channels — but those map to `ChannelManagement`, not `Threading`. The SPI's decomposition was right: reply-to-message and create-sub-channel are genuinely different operations.

- **Presence** was the interesting one. Discord has no REST endpoint for querying another user's presence — you get it from the Gateway's `PRESENCE_UPDATE` stream. This is the same class of problem we hit with IRC (RFC 1459 has no passive AWAY notification), but solvable here because the Gateway is in scope. A `ConcurrentHashMap` seeded from `GUILD_CREATE` and updated by `PRESENCE_UPDATE` gives native presence without polling.

- **MemberManagement** is the one we degraded. Discord's per-channel membership is permission-based — you grant or deny `VIEW_CHANNEL` via role overrides, not by adding or removing members. The SPI's `add(channel, member)` / `remove(channel, member)` contract doesn't map to that model. Honest degradation rather than semantic stretching.

## The Module Split

Two modules, following the `slack-bot` / `chat-*` pattern: `discord` (shared HTTP client + Gateway WebSocket) and `chat-discord` (ChatPlatform SPI + InboundConnector). The split exists because `DiscordClient` will be consumed by MCP tools and a future Qhorus `DiscordChannelBackend` — they need the HTTP client without pulling in `chat-spi`.

## The Gateway

Discord's Gateway is a WebSocket with a stateful lifecycle: `HELLO → IDENTIFY → HEARTBEAT loop → DISPATCH events → RESUME on disconnect`. It's more complex than IRC's TCP socket but the same reconnection pattern applies — exponential backoff, `volatile stopping` flag, log escalation at five consecutive failures.

The embedded test server was the hardest part. We needed a minimal WebSocket server that could send Gateway opcodes, track heartbeats, suppress ACKs on demand, and simulate disconnection. Java's `java.net.http.WebSocket` is client-only, so Claude built a raw `ServerSocket` with manual HTTP upgrade handshake and WebSocket frame encoding — the same approach as the IRC test server, just with JSON opcodes instead of line-delimited text.

## The GUID That Wasn't

The strangest discovery: JDK Corretto 22 has a modified constant in `jdk.internal.net.http.websocket.OpeningHandshake` — `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` instead of RFC 6455's `...B63`. This is the magic GUID used in the WebSocket handshake to compute `Sec-WebSocket-Accept`. A one-character deviation, buried in an internal class with no changelog entry.

The consequence: `HttpClient.newWebSocketBuilder()` from this JDK cannot connect to any RFC-compliant WebSocket server. The accept key validation fails silently. We only found it because the test server was computing the accept key with the RFC GUID and the JDK client was rejecting it. Confirmed via `javap -verbose` on the jmod constant pool.

The test server now uses the JDK's non-standard GUID so tests pass. Production deployment on this JDK would fail against Discord's Gateway — a problem that affects every WebSocket connection from this JVM, not just ours.

## What the SPI Learned

No SPI changes were needed. That's the headline. The composed-capabilities pattern with auto-degradation handled Discord's capability profile without modification — eight native, one degraded, all through the same builder. The `supports()` class-token query works for callers that need to branch on capability availability.

Two gaps are worth watching: `Members.list(ChatChannelRef)` returns guild-scoped results on Discord (correct for public channels, wrong for private ones), and `Presence.set()` is a permanent no-op because Discord doesn't support setting another user's status. Neither warrants an SPI change yet — but if Telegram or Slack exhibit the same patterns, a scope qualifier on Members or a read-only presence variant might earn its place.
