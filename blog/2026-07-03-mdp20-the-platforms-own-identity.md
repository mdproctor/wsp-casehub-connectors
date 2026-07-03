---
layout: post
title: "The Platform's Own Identity"
date: 2026-07-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [jwt, auth, websocket, chat-demo]
series: issue-52-user-identity-login
---

*Part of a series on [#52 — feat(chat-demo): user identity / login](https://github.com/casehubio/connectors/issues/52). Previous: [Making the Demo Talk Back](2026-07-02-mdp19-making-the-demo-talk-back.md).*

Last entry ended with every message showing sender `"ref"`. That's fixed. The demo now has JWT-based authentication — pick a name in a login gate, get a signed token, and every message you send is attributed to that identity.

## The dependency decision

The blocker was casehub-pages#88 — a lightweight dev-auth module providing `POST /dev/auth/login` with name-only authentication and auto-generated RSA keypairs. It shipped, so the question became: does chat-demo duplicate the endpoint or depend on it?

I went with the dependency. `casehub-pages-auth` is a jar with a JAX-RS resource, SmallRye JWT, and a Jandex index. Add it to `pom.xml`, the endpoint appears, tokens validate — zero code to write. The dependency is scoped to the demo profile on an unpublished module, so the "connectors has no casehub deps" constraint in PLATFORM.md doesn't apply. Replicating forty lines of JWT signing to avoid a dependency would have been the worse design.

## Bypassing the SPI

The Messaging SPI's `send(ChatChannelRef, ChatContent)` has no sender parameter — correctly, because real platforms determine sender from credentials. Slack sends as the bot. Discord sends as the bot. Adding a `MemberRef sender` parameter would be a dead parameter on every real implementation.

But the demo IS the platform. It doesn't authenticate against an external system — it needs to store messages with whatever identity the JWT carries. So `ChatResource` injects `ChatBackend` directly and calls `storeMessage("ref", channel, content, new MemberRef(identity), parentRef)` — same backend, same broadcast pipeline, caller-specified sender. The SPI stays clean; the demo gets identity.

## The filter that didn't fire

The design review caught something I would have shipped broken. The spec originally described a `@PreMatching ContainerRequestFilter` that reads the JWT from a WebSocket query parameter and promotes it to an `Authorization` header — the standard pattern for token promotion.

It doesn't work. Quarkus WebSockets Next routes upgrade requests through Vert.x, bypassing the JAX-RS filter chain entirely. The filter compiles, registers, and silently never fires on WebSocket connections. No error, no warning. Every WebSocket connection sails through unauthenticated.

The correct mechanism is `HttpUpgradeCheck` — a Quarkus WebSockets Next SPI that runs at upgrade time. It receives the Vert.x `HttpServerRequest`, reads the query param, validates the JWT via `JWTParser.parse()`, and returns permit or reject. Clean, targeted, and actually works. The gotcha — that `ContainerRequestFilter` is invisible to WebSocket upgrades — went into the knowledge garden as GE-20260703-e4a6b0.

## Auto-membership and presence

Two conveniences that make the demo feel real. Send a message to a channel you haven't joined? `ChatResource` auto-adds you as a member and broadcasts it. First message from an identity with no presence entry? Auto-creates it as ONLINE. Both are idempotent — the membership check runs `members().list()` and skips if you're already there; the presence check only fires on `UNKNOWN`, so an explicit OFFLINE is never overridden.

The frontend side is straightforward: `<pages-dev-auth>` blocks interaction until login, `<pages-identity>` shows the current user in the message input with a click to switch, and every `fetch()` call goes through `authenticatedFetch()` which adds the Bearer header and dispatches `pages-auth-expired` on 401. WebSocket connects with `?token=` and reconnects on identity switch.
