---
layout: post
title: "First real chat platform — IRC earns its stripes"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [irc, chat-spi, inbound-connector, testing]
series: issue-25-chat-irc-module
---

The ChatPlatform SPI shipped last session as an abstraction. This session gave it its first real implementation — and the implementation pushed back on the design in several places.

## The spec that kept getting corrected

I started with what seemed like a clean design: IrcClient as pure TCP transport, IrcChatPlatform as the CDI adapter, and an embedded test server since no pure Java IRC server library exists on Maven Central. The brainstorming produced a spec in about an hour. Then the code review started finding holes.

The biggest one: I'd designed the inbound path to bypass L2 entirely. The reasoning was that IRC uses a persistent TCP connection — it's not a webhook, so why route through InboundConnector? Claude caught this immediately. `ConnectorsCloudEventAdapter` observes `InboundMessage` events. Bypass L2 and IRC traffic becomes invisible to CloudEvents, auditing, and every other observer on that bus. `InboundConnectorTypes.IRC = "irc"` was already forward-declared in core — the platform had anticipated this, and I'd ignored the signal.

The fix restructured the module around `InboundConnector` instead of around `@PostConstruct`. That solved a second problem I hadn't noticed: blocking CDI startup on a network connection. `EmailInboundConnector` had already solved this — persistent TCP, virtual thread, exponential backoff, application starts regardless of server reachability. Same pattern, different protocol.

## RFC 1459 doesn't do what you think it does

I claimed native presence support. The AWAY command exists, so tracking away/online status seemed straightforward — just listen for AWAY broadcasts on the read loop.

There are no AWAY broadcasts. RFC 1459 only surfaces another user's away status in two places: RPL_AWAY (301) as a response to a PRIVMSG you send them, and WHOIS. No passive notification. The command exists, the mental model from Slack and Discord strongly suggests it broadcasts, but the RFC says otherwise. We degraded to `UnknownPresence` — the honest answer.

## The collector pattern

The interesting design problem was request/response coordination. IRC is line-based and asynchronous — `JOIN #test` triggers `353` (NAMES) and `366` (end of names) replies from the server, but they arrive on the read loop thread while the caller blocks on the main thread.

We used a `CompletableFuture` keyed by command+channel. Before sending JOIN, register a future under `"366:#test"`. The read loop completes it when 366 arrives. The caller blocks with a 10-second timeout. For LIST, the same pattern accumulates 322 replies until 323 (list end) completes the future with the collected list.

The key insight is that `CompletableFuture.complete()` establishes a happens-before relationship, so the accumulated data is safely published to the waiting thread without explicit synchronisation. Simpler than a state machine, naturally supports concurrent operations on different channels.

One thing the review caught: when the read loop exits on IOException, pending futures were never failed. Any thread blocked in `join()` or `names()` would hang for the full 10-second timeout even though the connection was already dead. A `failAllCollectors()` in the read loop's finally block fixed that — completes everything exceptionally so operations return immediately.

## Building your own test server

No pure Java IRC server exists as a Maven dependency. We checked — ircd4j is abandoned with 37 commits, zachbroad/irc-server has no build system. We could have forked zachbroad's repo and added one, but the server is designed as a standalone `main()` — making it embeddable with `start()/stop()` means understanding someone else's concurrency model and refactoring it. And we need test-specific APIs (`getReceivedMessages()`, `sendToChannel()`) either way. For 300 lines of purpose-built test server, writing our own was the simpler path.

The resulting module maps cleanly: Messaging, Discovery, and Members are native (IRC does these well). Threading degrades to channel fallback (IRC has no threads). Reactions are no-op (no emoji reactions in basic IRC). Presence is unknown (no passive notification in RFC 1459). The honest capability mapping.
