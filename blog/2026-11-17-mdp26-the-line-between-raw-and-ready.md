---
layout: post
title: "The Line Between Raw and Ready"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [teams, bot-framework, messaging-extensions, builder-pattern, spi-design, api-design, dry, round-trip-testing]
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

## Progressive disclosure

Hero and thumbnail cards follow the same principle at a smaller scale. `ComposeAttachment.hero(title, subtitle, imageUrl)` builds a standalone hero card — no adaptive card body, no preview layer, just the card type and its three fields.

The 3-arg form shipped first because it's the complete specification. But most hero cards in practice don't use all three arguments. A search result heading needs a title. A product listing needs title and price. The image is the rarer case, not the common one.

```java
ComposeAttachment.hero("Search Result").build()
ComposeAttachment.hero("Contoso Widget", "£29.99").build()
ComposeAttachment.hero("Contoso Widget", "£29.99", imageUrl).build()
```

Each overload delegates to the 3-arg form — the same `CardBuilder`, the same JSON envelope, the same protocol mechanics. The API surface grows without widening the builder's boundary. What changes is the noise at the call site: a title-only hero card no longer carries two trailing `null` arguments that mean nothing to the reader.

This is a general pattern worth naming. Ship the full-spec form first — it's correct and complete. Then add the progressive overloads once you can see what callers actually need. The overloads are cheaper to get right because the full-spec form already handles all the edge cases. They just peel off the common shapes.

## Buttons and the boundary within a call

Hero and thumbnail cards in Bot Framework also support action buttons — an array of `CardAction` objects, each with a `type`, `title`, and `value`. The shape is protocol structure. The values are caller content. The boundary runs through a single method call.

```java
ComposeAttachment.hero("Contoso Widget", "£29.99")
    .button("openUrl", "View", "https://contoso.com/widget")
    .button("imBack", "Buy", "buy-widget-123")
    .build()
```

The JSON envelope — an array called `buttons`, each element an object with three keys — is what every caller would construct identically. That belongs in the builder. But the button types (`openUrl`, `imBack`, `messageBack`, `invoke`) are protocol vocabulary the caller selects, not protocol boilerplate the builder can hard-code. So they stay as strings, not an enum.

This is a deliberate trade-off. An enum would catch typos at compile time, but it would also couple the builder to a snapshot of Bot Framework's action type list. When Microsoft adds a new action type, every consumer would need to wait for us to update the enum. A string means forward compatibility at the cost of a runtime error on typos — a cost that's real but far cheaper than a maintenance bottleneck.

![How the boundary runs through button() — shape is protocol, values are caller-chosen](images/compose-attachment-buttons.svg)

The same shape appears again in `tap` — a card-level action that fires when the entire card is tapped, not just a button. Same three fields, same boundary split. The difference is structural: `tap` is a single object (last call wins), while `buttons` is an array (each call appends). The protocol dictates which — a card has one tap target but can have many buttons — and the builder follows.

Internally, the builder stored each action as a `String[]` — `new String[]{type, title, value}`, accessed by index in `build()`. It worked, but it's the kind of code where a reader has to remember that `b[0]` is the type and `b[2]` is the value. The protocol names these fields. The code should too.

```java
private record CardAction(String type, String title, String value) {}
```

`buttons` becomes `List<CardAction>`, `tap` becomes `CardAction`, and the JSON construction reads as it should: `b.type()`, `b.title()`, `b.value()`. The public API — `button(String, String, String)` and `tap(String, String, String)` — doesn't change. The improvement is entirely internal: the builder's implementation now reflects the protocol's vocabulary instead of hiding it behind positional access.

The broader point: a clean protocol boundary doesn't always divide neatly at the method level. Sometimes the builder owns the shape of a structure while the caller owns what goes into it. The question isn't "does the builder handle this feature or not?" It's "what part of this feature is the same for every caller, and what part varies?"

## The shape underneath

A preview on an adaptive card and the content of a standalone hero card look like different things. One decorates a complex card; the other is the card. But at the protocol level, they're identical — a title, an optional subtitle, an optional images array. The same three fields, the same null checks, the same images-array wrapper.

Both `Builder.build()` and `CardBuilder.build()` were constructing this shape independently. That duplication isn't just a cleanup opportunity — it's a signal that the protocol has a shared concept the code hasn't named. Card content is card content, whether it's the main card or the preview.

```java
private static JsonObjectBuilder cardContent(String title, String subtitle, String imageUrl) {
    var content = Json.createObjectBuilder().add("title", title);
    if (subtitle != null) content.add("subtitle", subtitle);
    if (imageUrl != null) content.add("images", Json.createArrayBuilder()
            .add(Json.createObjectBuilder().add("url", imageUrl)));
    return content;
}
```

The return type matters. `CardBuilder.build()` needs to extend the builder with buttons and tap after the shared fields. `Builder.build()` passes it directly into the preview envelope. Returning `JsonObjectBuilder` instead of a built `JsonObject` keeps both paths open — the shape is shared, but each caller decides where it ends.

![Two paths converging on the shared card content structure](images/compose-attachment-shared-shape.svg)

## Proving the chain

The builder outputs a JSON string. `InvokeResponseSerializer` parses it back and nests it inside the Bot Framework response envelope. If the builder produces JSON the serializer can't traverse — an unexpected nesting, a structure it doesn't anticipate — the card is silently lost. The adaptive card builder already had a round-trip test for this. Hero and thumbnail cards didn't.

The fix is mechanical but the principle isn't. A builder that produces JSON for a serializer to consume is a contract across two code paths. Testing one side proves the builder works. Testing the other side proves the serializer works. Only a round-trip test proves they work together — and that neither side can change without the other noticing.

The single-card round-trip is necessary. It's not sufficient. A `ComposeResult` holds a list of attachments — the serializer iterates and re-emits each one. A multi-card test pushes on the things a single card can't: does the second card survive, does ordering hold, does one card's content bleed into the next? Hero, thumbnail, and adaptive cards exercise three different content types through the same serializer path. If any one of them is silently dropped or rewritten, the test catches it.

## Reference handlers

Four reference `InvokeHandler` implementations sit alongside the builder in `chat-teams`. Each one is under 30 lines. `RefSearchHandler` returns three canned search results with thumbnail previews. `RefLinkUnfurlHandler` returns a single unfurl card without a preview. `RefFetchTaskHandler` returns a task module form via `ComposeContinue`. `RefSubmitActionHandler` returns a confirmation card with a preview.

They follow the same pattern as `chat-ref`, `bank-ref`, and the other reference implementations — in-memory, no external dependencies, hard-coded responses. The difference is that `InvokeHandler` is Teams-specific, not a cross-platform SPI, so the reference handlers live directly in `chat-teams` rather than a separate module.

The handlers are the first real consumers of `ComposeAttachment`, and the code reads the way a consumer's code should read — the protocol mechanics are invisible, the card content and preview metadata are front and centre. That's what a convenience layer is for: not hiding complexity, but separating what varies from what doesn't.

## The design observation

"Raw for flexibility" and "builder for convenience" are not in tension when the boundary tracks a real concern split. The test is simple: does the builder make decisions the caller should be making? If yes, it's crossed into domain territory. If it only handles things the caller would always do identically, it's infrastructure helping with infrastructure.

`contentType: "application/vnd.microsoft.card.adaptive"` is something every caller types exactly the same way. That's the builder's job. The card body is something every caller builds differently. That stays a string.
