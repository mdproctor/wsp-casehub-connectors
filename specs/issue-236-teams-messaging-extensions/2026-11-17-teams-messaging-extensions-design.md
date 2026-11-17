# Teams Messaging Extensions Support

**Issue:** casehubio/connectors#236
**Deferred from:** casehubio/connectors#220 (Teams inbound via Bot Framework)
**Scale:** L | **Complexity:** High

## Overview

Bot Framework messaging extensions (`composeExtension` activities) enable search-based and action-based message extensions in the Teams compose box, plus link unfurling. All messaging extension interactions arrive as `type: "invoke"` activities with names prefixed `composeExtension/`.

The existing `InvokeHandler` → `InvokeService` → `InvokeResponse` → `InvokeResponseSerializer` pipeline from #235 handles dispatch. This design adds 3 new `InvokeResponse` variants for the distinct messaging extension response shapes, their serialization, and comprehensive tests.

No new modules, no new SPIs, no new webhook handler. The work is entirely within `chat-teams`.

## Scope

All three messaging extension types:
- **Search-based** — `composeExtension/query`, `composeExtension/selectItem`
- **Action-based** — `composeExtension/fetchTask`, `composeExtension/submitAction`
- **Link unfurling** — `composeExtension/queryLink`
- **Settings** — `composeExtension/querySettingUrl`, `composeExtension/setting`
- **Auth flows** — SSO/OAuth auth initiation responses

## InvokeResponse Variants

Three new variants added to the existing sealed interface in `InvokeResponse.java`:

### ComposeResult

```java
record ComposeResult(String attachmentLayout, List<String> attachmentsJson)
    implements InvokeResponse {}
```

Covers search results, link unfurling cards, selectItem, and submitAction results. All produce the same JSON envelope. `attachmentLayout` is typically `"list"` or `"grid"`. Each element in `attachmentsJson` is a complete attachment JSON object string containing at minimum `contentType` and `content` fields, with an optional `preview` field for search result thumbnails. Example: `{"contentType":"application/vnd.microsoft.card.adaptive","content":{...},"preview":{"contentType":"application/vnd.microsoft.card.thumbnail","content":{...}}}`.

### ComposeAuth

```java
record ComposeAuth(String suggestedActionsJson)
    implements InvokeResponse {}
```

Covers SSO/OAuth auth flow initiation. The `suggestedActionsJson` contains the OAuth URL and action configuration.

### ComposeContinue

```java
record ComposeContinue(String title, String cardJson)
    implements InvokeResponse {}
```

Covers task module responses for `fetchTask` and `submitAction`. Same shape as the existing `TaskModuleContinue` but wrapped in the `composeExtension` envelope instead of the `task` envelope.

## Invoke Name → Response Mapping

| Invoke name | Typical response |
|---|---|
| `composeExtension/query` | `ComposeResult` (list/grid of cards) |
| `composeExtension/queryLink` | `ComposeResult` (single unfurl card) |
| `composeExtension/selectItem` | `ComposeResult` (single card) |
| `composeExtension/submitAction` | `ComposeResult`, `ComposeContinue`, or `Message` |
| `composeExtension/fetchTask` | `ComposeContinue` (task module form) |
| `composeExtension/querySettingUrl` | `ComposeAuth` (config URL) |
| `composeExtension/setting` | `Accepted` (acknowledgment) |

## Serialization

`InvokeResponseSerializer.toJson()` gets 3 new cases in the switch expression.

**ComposeResult:**
```json
{
  "composeExtension": {
    "type": "result",
    "attachmentLayout": "list",
    "attachments": [{ "contentType": "...", "content": {...}, "preview": {...} }]
  }
}
```

**ComposeAuth:**
```json
{
  "composeExtension": {
    "type": "auth",
    "suggestedActions": { "actions": [{ "type": "openUrl", "value": "..." }] }
  }
}
```

**ComposeContinue:**
```json
{
  "composeExtension": {
    "type": "continue",
    "value": {
      "title": "Form Title",
      "card": {
        "contentType": "application/vnd.microsoft.card.adaptive",
        "content": { }
      }
    }
  }
}
```

## Attachment Model

Raw JSON strings, consistent with the existing `CardUpdate(String cardJson)` pattern. This library is delivery infrastructure — callers build card JSON however they want. No typed Adaptive Card model.

## What Does NOT Change

- `TeamsInboundConnector.handleInvoke()` — already routes all invoke activities through `InvokeService` and returns `WebhookResult.Responded`. The new variants flow through unchanged.
- `InvokeHandler` SPI — consumers implement handlers for `composeExtension/*` names exactly as they do for `adaptiveCard/action`.
- `InvokeService` — dispatches by name, returns `InvokeResponse`. No changes.
- `InvokeRequest` — already carries `name`, `value`, `senderId`, `channelRef`, `metadata`. All fields are populated for messaging extension activities.
- `WebhookResult.Responded` — already returns both a response body and inbound messages.

## Testing

### InvokeResponseSerializerTest

- `composeResult_serializesWithAttachments` — list layout, 2 attachments
- `composeResult_emptyAttachments` — edge case
- `composeResult_gridLayout` — verifies layout parameter
- `composeAuth_serializesWithSuggestedActions`
- `composeContinue_serializesWithTitleAndCard`

### TeamsInboundConnectorTest

Round-trip tests: invoke activity JSON in → InvokeHandler dispatched → InvokeResponse serialized → WebhookResult.Responded out.

- `composeExtensionQuery_withHandler_returnsRespondedWithResults`
- `composeExtensionQueryLink_returnsUnfurlCard`
- `composeExtensionSelectItem_returnsSingleCard`
- `composeExtensionFetchTask_returnsTaskModule`
- `composeExtensionSubmitAction_returnsResult`
- `composeExtensionSetting_returnsAccepted`
- `composeExtension_noHandler_returnsAccepted` — existing fallback behaviour

## Files Changed

| File | Change |
|---|---|
| `chat-teams/.../InvokeResponse.java` | Add `ComposeResult`, `ComposeAuth`, `ComposeContinue` variants |
| `chat-teams/.../InvokeResponseSerializer.java` | Add 3 serialization cases |
| `chat-teams/.../InvokeResponseSerializerTest.java` | Add 5 serialization tests |
| `chat-teams/.../TeamsInboundConnectorTest.java` | Add 7 round-trip tests |

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java` — existing sealed interface
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java` — existing serializer
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java` — existing SPI
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeService.java` — existing dispatch
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java:225-271` — existing handleInvoke
- casehubio/connectors#235 — Adaptive Card action handling (invoke activities)
- casehubio/connectors#220 — Teams inbound via Bot Framework (parent issue)
