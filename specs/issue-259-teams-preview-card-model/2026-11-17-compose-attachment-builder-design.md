# ComposeAttachment — Convenience builder for Teams messaging extension attachments

**Issue:** casehubio/connectors#259
**Branch:** issue-259-teams-preview-card-model
**Date:** 2026-11-17

## Problem

Consumers building Teams messaging extensions construct Bot Framework attachment
JSON as raw strings — manually assembling `contentType`, `content`, and `preview`
objects. This is error-prone (manual JSON escaping, easy to malform) and
repetitive (every attachment repeats the same content type boilerplate).

Example of current consumer code:

```java
String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\","
    + "\"content\":{\"type\":\"AdaptiveCard\"},"
    + "\"preview\":{\"contentType\":\"application/vnd.microsoft.card.thumbnail\","
    + "\"content\":{\"title\":\"Result 1\"}}}";
return new InvokeResponse.ComposeResult("list", List.of(att));
```

## Solution

A single utility class `ComposeAttachment` in the `chat-teams` module that
produces JSON strings for Bot Framework compose extension attachments. It
eliminates manual JSON construction while preserving the raw-JSON contract
established in decision D4 of #236.

## API

```java
public final class ComposeAttachment {

    public static Builder adaptive(String contentJson) { ... }

    public static final class Builder {
        public Builder preview(String title, String subtitle) { ... }
        public Builder preview(String title, String subtitle, String imageUrl) { ... }
        public String build() { ... }
    }
}
```

### Usage

```java
// Search result with thumbnail preview
String att = ComposeAttachment.adaptive(cardJson)
        .preview("Product Name", "£29.99")
        .build();

// Preview with image
String att = ComposeAttachment.adaptive(cardJson)
        .preview("Product Name", "£29.99", "https://img.example.com/thumb.png")
        .build();

// No preview (selectItem, link unfurl)
String att = ComposeAttachment.adaptive(cardJson).build();

// In handler context
return new InvokeResponse.ComposeResult("list", List.of(
    ComposeAttachment.adaptive(card1).preview("Item 1", "Details").build(),
    ComposeAttachment.adaptive(card2).preview("Item 2", "Details").build()
));
```

## Output format

Without preview:

```json
{
  "contentType": "application/vnd.microsoft.card.adaptive",
  "content": { ... adaptive card JSON ... }
}
```

With preview (title + subtitle):

```json
{
  "contentType": "application/vnd.microsoft.card.adaptive",
  "content": { ... adaptive card JSON ... },
  "preview": {
    "contentType": "application/vnd.microsoft.card.thumbnail",
    "content": {
      "title": "...",
      "subtitle": "..."
    }
  }
}
```

With preview (title + subtitle + image):

```json
{
  "contentType": "application/vnd.microsoft.card.adaptive",
  "content": { ... adaptive card JSON ... },
  "preview": {
    "contentType": "application/vnd.microsoft.card.thumbnail",
    "content": {
      "title": "...",
      "subtitle": "...",
      "images": [{ "url": "..." }]
    }
  }
}
```

## Design decisions

- **D1 — Attachment-level builder:** Produces individual attachment JSON strings.
  Consumer collects into `List<String>` and passes to `ComposeResult`. Composable
  — works with mixed attachment types in a single response.
- **D2 — Optional preview:** Calling `.preview()` adds a thumbnail preview card;
  omitting it produces an attachment without preview. Matches reality — selectItem
  and queryLink don't need previews; query search results do.
- **D3 — Static factory with builder:** `ComposeAttachment.adaptive(cardJson)`
  returns a builder. The factory method names the card type and hides the content
  type constant. Follows the `RichCard.builder()` pattern in this codebase.

## Module placement

`chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java`

Teams-specific — the attachment format is defined by the Bot Framework compose
extension protocol, not a cross-platform concern.

## What doesn't change

- `ComposeResult` signature stays the same: `(String attachmentLayout, List<String> attachmentsJson)`
- Raw JSON path always available — consumers can still pass hand-crafted strings
- No new dependencies — uses existing `jakarta.json` already in the module
- No new modules — this is one class in `chat-teams`

## Testing

Unit tests in `ComposeAttachmentTest`:

1. **Adaptive without preview** — produces correct JSON with contentType and content, no preview key
2. **Adaptive with title+subtitle preview** — includes thumbnail preview with title and subtitle
3. **Adaptive with title+subtitle+image preview** — includes images array in preview content
4. **Round-trip** — builder output parses correctly when fed through `InvokeResponseSerializer` as part of a `ComposeResult`
5. **Content JSON preserved** — arbitrary card JSON passed to `adaptive()` appears verbatim in the content field

## References

- InvokeResponse.java:17 — `ComposeResult` record definition
- InvokeResponseSerializer.java:58-68 — `ComposeResult` serialization
- TeamsInboundConnectorTest.java:435-561 — existing messaging extension tests with raw attachment JSON
- decisions.md (D4 from #236) — raw JSON as foundation, revisit with builder
- RichCard.java:25-56 — builder pattern precedent in this codebase
- casehubio/connectors#259 — issue body with hypothetical API
