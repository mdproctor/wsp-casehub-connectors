# Teams Proactive Messaging — Design Spec

**Issue:** casehubio/connectors#234
**Branch:** issue-234-teams-proactive-messaging
**Date:** 2026-11-16

## Overview

Add proactive messaging to the Teams ChatPlatform — the bot can initiate 1:1 conversations with users who have not previously messaged it, using the Bot Framework REST API `POST /v3/conversations`. Also capture `ConversationReference` data from inbound `conversationUpdate` activities to optimise future sends.

## Scope

**In scope:**
- `BotFrameworkClient.createConversation()` method
- `ConversationReferenceStore` CDI SPI with `InMemoryConversationReferenceStore` default
- `user:<aadObjectId>` channel prefix in `TeamsChatPlatform.sendMessage()`
- `conversationUpdate` activity handling in `TeamsInboundConnector` (store ref + fire event)
- Config property `casehub.connectors.teams-bot.default-tenant-id`

**Out of scope:**
- Multi-tenant bot support (single configured tenant)
- Rich content on the Bot Framework path (text only, matching existing `personal:` path)
- Adaptive Card proactive messages (separate concern, #235)
- Persistent storage implementations (callers bring their own `ConversationReferenceStore` impl)

## Data Model

```java
public record ConversationReference(
    String aadObjectId,
    String conversationId,
    String serviceUrl,
    String tenantId
) {}
```

Keyed by `aadObjectId`. Immutable record — a new interaction with a different `serviceUrl` or `conversationId` overwrites the stored entry.

## ConversationReferenceStore SPI

```java
public interface ConversationReferenceStore {
    Optional<ConversationReference> find(String aadObjectId);
    void store(ConversationReference ref);
}
```

Lives in `chat-teams` module (Teams-specific concept, not cross-platform).

`InMemoryConversationReferenceStore` — `@DefaultBean`, `@ApplicationScoped`, `ConcurrentHashMap<String, ConversationReference>`. Lost on restart; cold-start path (`createConversation`) covers recovery. Callers requiring persistence provide their own CDI alternative.

## BotFrameworkClient Changes

### New method: `createConversation`

```java
public String createConversation(String serviceUrl, String aadObjectId, String tenantId)
```

- Validates `serviceUrl` via existing `validateServiceUrl()`
- `POST {serviceUrl}/v3/conversations` with body:
  ```json
  {
    "bot": { "id": "<bot-app-id>" },
    "members": [{ "id": "<aadObjectId>" }],
    "tenantId": "<tenantId>",
    "isGroup": false
  }
  ```
- Bot app ID: resolved from config `casehub.connectors.teams-bot.app-id` (new config property)
- Auth: existing `resolveToken()` mechanism (`microsoft-bot` service connection)
- Returns the `id` field from the response JSON (the new conversation ID)
- Throws on non-2xx (unlike existing methods that log warnings — conversation creation failure is not recoverable)

### Existing method behaviour

`sendToConversation()` and `replyToActivity()` remain unchanged. Once `createConversation` returns a conversation ID, the existing `sendToConversation` handles message delivery.

## TeamsChatPlatform.sendMessage() — Three-Way Routing

Current two-way routing (`personal:` vs `teamId:channelId`) becomes three-way:

```
sendMessage(channel, content):
  if channel starts with "personal:"
    → existing path: sendToConversation(defaultServiceUrl, conversationId, text)
  else if channel starts with "user:"
    → NEW proactive path (see below)
  else
    → existing path: Graph API sendMessage(teamId, channelId, ...)
```

### Proactive path (`user:<aadObjectId>`)

1. Extract `aadObjectId` from channel ref
2. `conversationReferenceStore.find(aadObjectId)`
3. **Hit:** use stored `serviceUrl` and `conversationId` → `botClient.sendToConversation()`
4. **Miss:** `botClient.createConversation(defaultServiceUrl, aadObjectId, defaultTenantId)` → store the returned ref → `botClient.sendToConversation()`
5. Return `SendResult` with `personal:<conversationId>` as the resolved channel ref

The returned `personal:<conversationId>` channel ref allows callers to use the direct conversation ID for subsequent messages, bypassing store lookup.

## TeamsBeans — CDI Wiring Changes

New config properties:
- `casehub.connectors.teams-bot.default-tenant-id` — tenant ID for cold-start `createConversation`
- `casehub.connectors.teams-bot.app-id` — bot application ID for the `createConversation` request body

`TeamsChatPlatform` builder gains:
- `withConversationReferenceStore(ConversationReferenceStore store)`
- `withDefaultTenantId(String tenantId)`
- `withBotAppId(String appId)`

The `user:` path requires `botClient`, `defaultServiceUrl`, `defaultTenantId`, and `botAppId` all configured. Missing any of these when a `user:` channel is received throws `UnsupportedOperationException` with a descriptive message (matching existing pattern for missing `botClient`).

## TeamsInboundConnector — conversationUpdate Handling

Currently ignored (log + return `Ignored`). Changed to:

1. Check `membersAdded` array in the activity
2. For each added member that is NOT the bot itself:
   - Extract `aadObjectId`, `conversationId`, `serviceUrl`, `tenantId` from the activity
   - Construct `ConversationReference` and call `store.store(ref)`
3. Build an `InboundMessage` with metadata:
   - `teams-activity-type`: `conversationUpdate`
   - `teams-members-added`: comma-separated AAD Object IDs of added members (if present)
   - `teams-members-removed`: comma-separated AAD Object IDs of removed members (if present)
   - Standard fields: `teams-service-url`, `teams-conversation-id`, `teams-tenant-id`
4. Return `WebhookResult.Deliver` with the constructed message

`membersRemoved` events do NOT remove stored refs — a removed member might be re-added, and the conversation ID remains valid. Ref cleanup is the caller's responsibility.

## Configuration Summary

| Property | Required for | Default |
|----------|-------------|---------|
| `casehub.connectors.teams-bot.default-service-url` | `personal:` and `user:` paths | — |
| `casehub.connectors.teams-bot.default-tenant-id` | `user:` cold-start path | — |
| `casehub.connectors.teams-bot.app-id` | `user:` cold-start path | — |

All three are required for proactive messaging. The `personal:` path only needs `default-service-url` (unchanged).

## Error Handling

- `createConversation` non-2xx: throw, propagate to caller. Common failures: 403 (bot not installed in tenant), 404 (user not found), 429 (rate limited). Map via existing `ProviderErrorMapping` where applicable.
- Store miss + createConversation success: store the ref, then send. If send fails after create, the ref is still stored (conversation exists even if first message failed).
- Store miss + createConversation failure: no ref stored, exception propagates.
- `conversationUpdate` store failure: log warning, still fire the inbound event. Storage is best-effort optimisation.

## Testing Strategy

### Unit Tests

1. **`BotFrameworkClient.createConversation()`**
   - Verify request body structure (bot.id, members, tenantId, isGroup)
   - Verify conversation ID extracted from response
   - Verify SSRF validation called on serviceUrl
   - Verify exception on non-2xx response

2. **`InMemoryConversationReferenceStore`**
   - `find()` returns empty for unknown user
   - `store()` then `find()` returns the ref
   - `store()` overwrites existing ref for same user

3. **`TeamsChatPlatform.sendMessage()` — user: routing**
   - `user:` with store hit → sendToConversation with stored serviceUrl/conversationId
   - `user:` with store miss → createConversation then sendToConversation
   - `user:` with store miss → ref stored after createConversation
   - `user:` without botClient configured → UnsupportedOperationException
   - Verify returned SendResult has `personal:` prefix (not `user:`)

4. **`TeamsInboundConnector` — conversationUpdate**
   - `membersAdded` with non-bot member → ref stored, event fired
   - `membersAdded` with bot only → no ref stored (bot self-add)
   - `membersRemoved` → no ref removal, event fired
   - Verify InboundMessage metadata fields

### Integration Test

Full proactive flow: create conversation → store ref → send message → verify delivery via `SentMessageCapture`.

## References

- Bot Framework REST API: `POST /v3/conversations` — creates 1:1 conversation
- `chat-teams/src/main/java/.../BotFrameworkClient.java` — existing Bot Framework client
- `chat-teams/src/main/java/.../TeamsChatPlatform.java` — sendMessage routing (line 85-98)
- `chat-teams/src/main/java/.../TeamsInboundConnector.java` — conversationUpdate handling (line 86-89)
- `chat-teams/src/main/java/.../TeamsBeans.java` — CDI wiring and config properties
- `email-google-inbound/.../HistoryStore.java` — CDI SPI pattern precedent
- D1-D5 in `specs/issue-234-teams-proactive-messaging/decisions.md`
