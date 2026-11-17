---
layout: post
title: "The Dispatch That Didn't Change"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [teams, bot-framework, messaging-extensions, documentation]
series: issue-257-teams-manifest-messaging-ext
---

# The Dispatch That Didn't Change

The messaging extensions SPI landed in #236 — three new `InvokeResponse` variants (`ComposeResult`, `ComposeAuth`, `ComposeContinue`), their serialization, and comprehensive round-trip tests. What remained was the gap between "the code handles everything" and "a consumer can actually deploy this to a real Teams tenant."

That gap turned out to be entirely documentation.

The interesting thing about Bot Framework messaging extensions is how little the code side needs to know about the manifest. A Teams app manifest declares `composeExtensions` with `commands` — search commands, action commands, link unfurling handlers — and each one maps to an invoke activity name that the existing `InvokeService` already dispatches by exact match. The manifest is configuration; the SPI is the runtime. We didn't need to touch either `TeamsInboundConnector` or `InvokeService` because they were already general enough.

What *did* need documenting was the two-level dispatch that most guides gloss over. The invoke activity name (`composeExtension/query`, `composeExtension/fetchTask`, etc.) determines which `InvokeHandler` bean receives the request. But when a manifest declares two search commands — say, "Search products" and "Search orders" — both arrive as `composeExtension/query`. The `commandId` field inside `InvokeRequest.value()` is what distinguishes them, and that field maps directly to the `id` in the manifest's `commands[]` array. A consumer deploying their first multi-command extension will hit this wall immediately if the docs don't explain it.

We produced two artifacts. A manifest template at `chat-teams/src/main/resources/manifest-template.json` covering all three extension types — search, action, and link unfurling — in a single file. The template uses `_comment` fields for inline documentation since JSON has no comment syntax. And a standalone setup guide at `docs/guides/teams-messaging-extensions-setup.md` covering the Azure bot registration prerequisites (as a checklist with links, not a step-by-step walkthrough that rots when Microsoft changes their portal), the manifest structure, `InvokeHandler` wiring, the `commandId` routing pattern with a worked example, and the Quarkus configuration properties.

The prerequisite checklist is deliberately light — four bullet points with links to Microsoft's docs. Azure portal UI changes frequently enough that maintaining our own screenshots would be a constant source of drift. The manifest and handler wiring are what Microsoft doesn't document from the consumer's perspective, and that's where the guide earns its keep.
