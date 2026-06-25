---
layout: post
title: "Duck Typing for Chat Platforms"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [spi-design, chat-platform, capability-composition]
---

## Duck Typing for Chat Platforms

I wanted a common programming model for chat systems — Slack, Discord, IRC — that doesn't pretend they're all the same. They're not. Slack has threads and reactions. IRC has channels and nothing else. The model had to express that honestly, without forcing callers to branch on capabilities or pretend every platform is Slack.

The design question took four rounds of review to settle. Three standard approaches failed for the same reason: they define capabilities at the interface level and handle variation at the method level. A single interface with `Optional` returns (Approach A) gives you methods that don't work. Capability interfaces with `instanceof` checks (Approach B) push branching to the caller. Silent degradation (Approach C) is a dishonest API.

What worked was composing the platform from focused capability interfaces — `Messaging`, `Threading`, `Discovery`, `Reactions`, `Presence`, `Members` — where each platform assembles the capabilities it supports and the builder auto-fills degradation defaults for the rest.

```java
ChatPlatform irc = ChatPlatform.builder("irc")
    .messaging(new IrcMessaging(client))
    .members(new IrcMembers(client))
    .build();
// threading → ChannelFallbackThreading (posts to channel)
// reactions → NoOpReactions (silently ignored)
// supports(Threading.class) → false
```

The degradation types are first-class named classes — `ChannelFallbackThreading`, `NoOpReactions`, `UnknownPresence` — not anonymous behaviour buried in conditionals. They're testable, reusable across platforms, and visible in the builder call. `supports(Class<?>)` uses the capability interface class as a token, not an enum, so adding a new capability never touches existing types.

The hardest design decision was inbound. I initially designed a `Receiving` capability on `ChatPlatform`, but the reviews surfaced a fatal problem: Slack inbound already flows through `WebhookInboundConnector` → `InboundMessage` on the CDI event bus. A parallel `Receiving` SPI would split the event bus — new platforms would be invisible to `ConnectorsCloudEventAdapter` and every `@ObservesAsync InboundMessage` observer.

The fix was to drop `Receiving` entirely. New platforms (Discord, IRC) implement the existing `InboundConnector` SPI — proven lifecycle, managed by `InboundConnectorService`. A `ChatInboundAdapter` observes `InboundMessage` and, for chat platforms, fires a typed `ReceivedMessage` via per-platform `InboundTranslator` beans. Single event bus. No gaps. No new lifecycle management.

The module split was another review catch. I'd initially proposed one `chat` module for everything — the spec reviewers pointed out it contradicts the existing principle that every transport module is independently deployable. The right structure is five modules from day one: `chat-spi`, `chat-ref`, `chat-slack`, `chat-discord`, `chat-irc`. An app needing Slack chat never pulls in IRC protocol handling.

The SPI foundation is implemented — `chat-spi` (model types, capability interfaces, builder, degradation types, `ChatPlatformService`, `ChatInboundAdapter`) and `chat-ref` (full-fidelity in-memory reference implementation). The reference impl serves as the SPI contract test target: every capability exercised end-to-end in `@QuarkusTest` with zero external infrastructure.

The platform adapters — Slack, Discord, IRC — come next.
