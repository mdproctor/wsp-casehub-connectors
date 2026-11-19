---
layout: post
title: "Teams Hardening: Fifteen Issues in One Branch"
date: 2026-11-19
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [teams, chat-platform, bot-framework, testing, documentation]
series: feat/306-lipsync-warmup-delegation
---

# Teams Hardening: Fifteen Issues in One Branch

The branch started with a one-liner — `LipSyncEnricher` wasn't delegating `warmUp()` to its wrapped `TextToSpeech` delegate, so wrapping a provider with the phoneme enricher silently broke its initialisation path. `MarkTimingEnricher` had been fixed in the same PR that introduced the problem, but `LipSyncEnricher` was left for a follow-up. Five minutes of work, but it's the kind of thing that surfaces as a mysterious "model didn't load" in production.

The remaining fourteen issues were all Teams. What made them worth batching was that they shared a dependency chain — the `BotFrameworkClient` signature change for rich content pulled a thread that ran through almost everything else.

## The signature change that touched everything

`BotFrameworkClient.sendToConversation` and `replyToActivity` accepted a plain `String text`. Team channel sends went through `TeamsMapper.toMessageBody()` which built HTML, adaptive cards, and file attachment references. Personal and user channel sends bypassed all of that — they passed `content.text()` directly. Markdown, cards, and attachments were silently dropped.

The fix was changing both methods to accept a `JsonObject` activity body. We added `TeamsMapper.toActivity()` to build the Bot Framework v3 format — `text`, `textFormat`, and an `attachments` array with parsed adaptive card JSON objects (the Graph API format uses string content; Bot Framework wants parsed objects). Every caller needed updating: `sendPersonalMessage`, `sendProactiveMessage`, `replyToUserChannel`, the inline personal reply path in `replyToMessage`, and the command dispatch in `TeamsInboundConnector`.

The surprise was that the inbound connector also called these methods — for command responses. Those were plain text replies, so they needed wrapping in minimal activity JSON. The compiler caught every call site, which is the whole point of making the type change rather than adding an overload.

## The null guard pattern

`TeamsChatPlatform.uploadAttachments` assumed the Graph `/filesFolder` response always includes `parentReference.driveId`. If Graph returns a `DriveItem` without `parentReference` — possible for root-level items — this throws `NullPointerException` before any `ProviderApiException` can surface. The fix is a null check that throws `ProviderApiException` with a clear message, keeping the error in the domain the caller expects.

## Test coverage as documentation

Five of the fifteen issues were test-only. `safeAttachmentName` and `extensionFromMimeType` had no coverage — making the methods package-private for direct testing was cleaner than threading assertions through the full send path. The reply-with-attachments path mirrored the send path but had no test proving the upload and reply endpoints were actually hit.

The two integration tests — proactive messaging flow and invoke handler end-to-end — verify complete chains that the unit tests can't. The proactive test exercises create-conversation → store-reference → reuse-on-second-send → reply, all in one flow. The invoke test verifies that a `POST` with an invoke activity body reaches an `InvokeHandler`, the response serialises correctly, and the `InboundMessage` carries the right metadata.

## Documentation gap

Four issues were documentation sync — adding `WebhookResult.Responded` and `InvokeHandler` SPI to the consumer guide, contributor guide, ARC42, and CLAUDE.md. The messaging extensions setup guide got settings URL configuration in the manifest template and a `jq` one-liner for stripping the `_comment` fields before packaging. The user-scoped accessor pattern (`calendarOps(userId)`, `emailOps(userId)`, `files(userId)`) was retrofitted months ago but the guides still showed the old direct-method interfaces.

One issue — attachment download size limit — turned out to be already implemented. `BotFrameworkClient.downloadAttachment` had both a `Content-Length` pre-check and a streaming size guard. Closed it.

The remaining queue item is `#218` — SharePoint multi-site support. That's a design question, not a hardening task, so it gets its own session.
