---
layout: post
title: "The Line Between Raw and Ready"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [teams, bot-framework, messaging-extensions, builder-pattern, spi-design]
series: issue-258-reference-invoke-handlers
---

# The Line Between Raw and Ready

When we built the messaging extension dispatch, the attachment model was deliberately raw. `ComposeResult` takes `List<String>` — each string is a complete attachment JSON blob. The caller constructs it however they want. No types, no builders, no opinions. This was the right call for a connector library that sits at the infrastructure layer: the moment you model card structure, you're making decisions about content, and content isn't your job.

But every consumer building a search extension hits the same wall. The Bot Framework attachment envelope has a fixed shape — a `contentType` string that's always the same, a `content` object holding the adaptive card, and an optional `preview` block with its own `contentType`, `title`, `subtitle`, and `images` array. None of that is caller content. It's protocol boilerplate that every extension repeats identically.

The friction shows up immediately in test code, which is a reliable signal:

```java
String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\","
    + "\"content\":{\"type\":\"AdaptiveCard\"},"
    + "\"preview\":{\"contentType\":\"application/vnd.microsoft.card.thumbnail\","
    + "\"content\":{\"title\":\"Result 1\"}}}";
```

That's not someone expressing a domain concept. That's someone transcribing a protocol by hand.

## Where the line sits

The interesting design question isn't whether to add a builder — it's where the builder stops. `ComposeAttachment.adaptive()` takes the card JSON as an opaque string and wraps it in the protocol envelope. The preview is optional — `.preview("Title", "Subtitle")` or `.preview("Title", "Subtitle", imageUrl)`. The output is a JSON string that plugs directly into `ComposeResult`'s `List<String>`.

```java
return new InvokeResponse.ComposeResult("list", List.of(
    ComposeAttachment.adaptive(card1).preview("Contoso Widget", "£29.99").build(),
    ComposeAttachment.adaptive(card2).preview("Northwind Gadget", "£49.99").build()
));
```

The builder handles everything the protocol dictates — `contentType` values, the envelope nesting, the thumbnail preview structure. The caller provides everything that varies — the card body and the preview metadata. The raw `List<String>` contract on `ComposeResult` doesn't change, so any consumer that prefers hand-crafted JSON can still use it.

![What ComposeAttachment handles vs. what the caller provides](images/compose-attachment-boundary.svg)

The boundary is clean because it tracks a real distinction: protocol structure vs. caller content. If the builder had tried to model the adaptive card body — text blocks, images, actions — it would have crossed into content territory. The card body is the caller's business. The attachment envelope is the protocol's.

## Reference handlers

Four reference `InvokeHandler` implementations sit alongside the builder in `chat-teams`. Each one is under 30 lines. `RefSearchHandler` returns three canned search results with thumbnail previews. `RefLinkUnfurlHandler` returns a single unfurl card without a preview. `RefFetchTaskHandler` returns a task module form via `ComposeContinue`. `RefSubmitActionHandler` returns a confirmation card with a preview.

They follow the same pattern as `chat-ref`, `bank-ref`, and the other reference implementations — in-memory, no external dependencies, hard-coded responses. The difference is that `InvokeHandler` is Teams-specific, not a cross-platform SPI, so the reference handlers live directly in `chat-teams` rather than a separate module.

The handlers are the first real consumers of `ComposeAttachment`, and the code reads the way a consumer's code should read — the protocol mechanics are invisible, the card content and preview metadata are front and centre. That's what a convenience layer is for: not hiding complexity, but separating what varies from what doesn't.

## The design observation

"Raw for flexibility" and "builder for convenience" are not in tension when the boundary tracks a real concern split. The test is simple: does the builder make decisions the caller should be making? If yes, it's crossed into domain territory. If it only handles things the caller would always do identically, it's infrastructure helping with infrastructure.

`contentType: "application/vnd.microsoft.card.adaptive"` is something every caller types exactly the same way. That's the builder's job. The card body is something every caller builds differently. That stays a string.
