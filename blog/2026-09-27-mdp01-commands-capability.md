---
title: "Commands capability — slash commands as a ChatPlatform primitive"
date: 2026-09-27
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [chatplatform, spi, discord, slack, commands, interactions]
series: issue-32-discord-slash-commands
---

The ChatPlatform SPI has nine capabilities — messaging, threading, discovery,
reactions, presence, members, channel management, member management, message
history. Today it got its tenth: Commands.

## The design question

Slash commands exist on every major chat platform, but they work differently.
Discord has a rich interaction model — typed parameters, autocomplete, modal
dialogs, message components. Slack's slash commands are simpler — a single
text string the user typed after the command name. Signal doesn't have them
at all.

I wanted a cross-platform abstraction that captured the common ground without
flattening the platforms into a lowest-common-denominator API. The answer was
batching: batch 1 covers typed parameters (STRING, INTEGER, BOOLEAN, NUMBER)
and immediate + deferred responses. Batch 2 — buttons, select menus, modals,
autocomplete — stays future work. The parameter types map directly to Discord's
native types. Slack degrades gracefully — parameters become documentation-only
hints, and the raw text arrives as a single `"text"` argument.

## Seven decisions, no debates

The design converged quickly. Every decision had a clear precedent in the
existing codebase:

- Generic ChatPlatform capability, not Discord-specific — follows the
  established capability pattern
- `CommandHandler` SPI with CDI discovery — mirrors `InboundConnector`
- Immediate + deferred responses — both platforms need it, and without
  deferred support the 3-second timeout rules out anything that touches
  a database
- Auto-registration at startup — `CommandService` discovers handlers, calls
  `Commands.registerAll()` on supporting platforms
- Dedicated JAX-RS endpoints per platform — interactions are request-response
  operations, not inbound messages, so they don't belong in the
  `WebhookInboundConnector` pipeline
- Typed parameters with a small initial set — platform-specific types
  (USER, CHANNEL) deferred to batch 2
- All in existing modules — no new Maven modules needed

The one decision that took thought was where platform-specific type mappings
live. My first draft had `CommandParameterType.toDiscordType()` — a method
on the cross-platform enum that returned Discord's integer type codes. The
self-review caught it: that leaks Discord knowledge into `chat-spi`. The
mapping moved to `DiscordCommands`, where it belongs.

## The recursive constructor trap

The implementation surfaced a genuine gotcha. Both the Discord and Slack
endpoints use a dual-constructor pattern — a CDI constructor with
`@ConfigProperty` parameters for production, and a package-private
constructor with direct types for testing. The Slack endpoint's CDI
constructor accepts `ManagedExecutor` and the test constructor accepts
`ExecutorService`. Since `ManagedExecutor extends ExecutorService`, calling
`this(commandService, signingSecret, executor)` from the CDI constructor
matches *itself* — Java's overload resolution picks the most specific type,
and `ManagedExecutor` is more specific than `ExecutorService`.

The result: `StackOverflowError` at runtime. Compiles fine. No warning.
The fix is to duplicate the field assignments instead of delegating. I
revised the existing garden entry on the dual-constructor pattern to add
this caveat — it's the kind of thing that bites you once and you never
forget.

## What this unlocks

Commands is the first interactive capability on ChatPlatform — everything
before it was fire-and-forget (send a message, add a reaction, list
channels). With Commands, the platform can receive structured input from
users and respond synchronously. Combined with the `@McpDomain` machinery
and the scenario engine, slash commands become first-class participants in
platform automation. An LLM can register a `/status` command that queries
the ledger, checks the calendar, and responds with a summary — all through
the same SPI that handles messaging.

The next piece is email. The `EmailPlatform` SPI exists but has no
implementation — just a `NoOpEmailPlatform` fallback. Two issues are
queued: #113 for an in-memory reference implementation, #114 for a Gmail
provider using Google's Java client library (same auth pattern as the
existing `calendar-google` module). Once those land, email joins chat,
calendar, and banking as a full participant in the automation surface.
