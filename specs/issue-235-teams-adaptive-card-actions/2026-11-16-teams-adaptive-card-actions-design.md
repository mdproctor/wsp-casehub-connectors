# Design — #235 Adaptive Card Action Handling (Invoke Activities)

Bot Framework invoke activities deliver Adaptive Card action submissions
and task module interactions. These require a synchronous HTTP response
body — the only mechanism to update a card in-place or return a task
module dialog. This design adds transport support (`WebhookResult.Responded`),
a handler SPI for response construction, and invoke activity routing in the
Teams inbound connector.

## Scope

**In scope:**
- `WebhookResult.Responded` variant (transport layer — connectors-api)
- `WebhookRouter` handling of Responded (webhook module)
- `InvokeHandler` / `InvokeService` SPI (handler layer — chat-teams)
- `InvokeRequest` / `InvokeResponse` types (chat-teams)
- `TeamsInboundConnector` invoke activity routing (chat-teams)
- Tests for all layers

**Invoke subtypes handled:**
- `adaptiveCard/action` — Adaptive Card Universal Action (Action.Submit)
- `task/fetch` — request to open a task module dialog
- `task/submit` — task module form submission

**Out of scope:**
- `composeExtension/*` — messaging extensions → casehubio/connectors#236
- Other invoke subtypes (handled by graceful acknowledge fallback)

## Architecture

### Layer 1: Transport — WebhookResult.Responded

Add the sixth variant to the sealed interface in `connectors-api`:

```java
record Responded(String responseBody, String contentType,
                 List<InboundMessage> messages) implements WebhookResult {}
```

This is the only WebhookResult variant that both delivers InboundMessages
AND returns a custom HTTP response body. Existing variants either deliver
messages (Delivered, DeferredDelivery) or return a body (Challenged) — never both.

**WebhookRouter handling:** In `dispatch()`, add:
```java
case WebhookResult.Responded(var responseBody, var contentType, var msgs) -> {
    msgs.forEach(service::receive);
    yield Response.ok(responseBody).type(contentType).build();
}
```

The sealed type forces a compile error in WebhookRouter when the variant is
added — this is the intended design (D6, pre-validated in #220 D4).

### Layer 2: Handler SPI — InvokeHandler + InvokeService

All types in `chat-teams` module, package `io.casehub.connectors.chat.teams`.

**InvokeRequest** — input record:
```java
public record InvokeRequest(
    String name,             // invoke activity name: "adaptiveCard/action", "task/fetch", "task/submit"
    JsonObject value,        // action payload (card action data, task module data)
    String senderId,         // from.aadObjectId or from.id
    String channelRef,       // compound teamId:channelId or personal:conversationId
    Map<String, String> metadata  // activity context (service URL, conversation ID, etc.)
) {}
```

**InvokeResponse** — sealed interface with typed variants:
```java
public sealed interface InvokeResponse {
    record Accepted() implements InvokeResponse {}
    record CardUpdate(String cardJson) implements InvokeResponse {}
    record Message(String text) implements InvokeResponse {}
    record TaskModule(String taskInfoJson) implements InvokeResponse {}
    record Error(int statusCode, String message) implements InvokeResponse {}
}
```

Each variant maps to exactly one Bot Framework response schema:
- `Accepted` → `{ "statusCode": 200 }`
- `CardUpdate` → `{ "statusCode": 200, "type": "application/vnd.microsoft.card.adaptive", "value": <parsed cardJson> }`
- `Message` → `{ "statusCode": 200, "type": "application/vnd.microsoft.activity.message", "value": "<text>" }`
- `TaskModule` → `{ "statusCode": 200, "type": "application/vnd.microsoft.card.adaptive", "value": { "task": { "type": "continue", "value": <parsed taskInfoJson> } } }`
- `Error` → `{ "status": <statusCode>, "body": "<message>" }`

The connector handles JSON serialization — consumers work with typed Java.

**InvokeHandler** — CDI bean interface:
```java
public interface InvokeHandler {
    String name();  // invoke activity name this handler responds to
    InvokeResponse handle(InvokeRequest request);
}
```

One bean per invoke name. Multiple handlers for the same name is a
startup error (same pattern as duplicate webhook connector IDs).

**InvokeService** — CDI bean, discovers handlers at startup:
```java
@ApplicationScoped
public class InvokeService {
    private final Map<String, InvokeHandler> handlers;

    @Inject
    InvokeService(@All List<InvokeHandler> handlers) { ... }

    public InvokeResponse dispatch(InvokeRequest request) {
        InvokeHandler handler = handlers.get(request.name());
        if (handler == null) {
            LOG.fine("teams-invoke: no handler for: " + request.name());
            return new InvokeResponse.Accepted();
        }
        return handler.handle(request);
    }
}
```

### Layer 3: Connector Wiring

Add `case "invoke"` to `TeamsInboundConnector.doHandle()`:

```java
case "invoke" -> handleInvoke(activity);
```

The `handleInvoke` method:
1. Extracts `name` and `value` from the activity JSON
2. Constructs `InvokeRequest` with sender, channel, metadata
3. Dispatches via `InvokeService.dispatch()` to get `InvokeResponse`
4. Constructs an `InboundMessage` with invoke-specific metadata:
   - `teams-activity-type` = `invoke`
   - `teams-invoke-name` = the invoke name
   - `teams-invoke-data` = serialized value payload
5. Serializes the `InvokeResponse` to Bot Framework JSON
6. Returns `WebhookResult.Responded(responseJson, "application/json", List.of(inboundMessage))`

The InboundMessage is always delivered regardless of whether a handler exists.
This satisfies the delivery infrastructure obligation — downstream observers
can react to invoke events even without an InvokeHandler.

### Dependency Injection

`TeamsInboundConnector` already receives `BotFrameworkAuth`, `BotFrameworkClient`,
`CommandService`, `appId`, and `ConversationReferenceStore` via constructor
injection. Add `InvokeService` as a new constructor parameter:

```java
TeamsInboundConnector(BotFrameworkAuth auth, BotFrameworkClient client,
                      CommandService commandService, String appId,
                      ConversationReferenceStore conversationReferenceStore,
                      InvokeService invokeService)
```

The existing two-arg and five-arg constructors for backward compatibility
remain, defaulting `invokeService` to null. When null, `handleInvoke`
falls back to `InvokeResponse.Accepted()` — same behavior as no registered
handlers.

### InvokeResponse Serialization

A package-private `InvokeResponseSerializer` handles the mapping from
sealed variants to Bot Framework JSON. Keeps serialization logic out of
the connector and makes it independently testable:

```java
class InvokeResponseSerializer {
    static String toJson(InvokeResponse response) {
        return switch (response) {
            case InvokeResponse.Accepted() -> "{\"statusCode\":200}";
            case InvokeResponse.CardUpdate(var cardJson) -> ...;
            case InvokeResponse.Message(var text) -> ...;
            case InvokeResponse.TaskModule(var taskInfoJson) -> ...;
            case InvokeResponse.Error(var code, var msg) -> ...;
        };
    }
}
```

Uses `jakarta.json` (already a dependency) for safe JSON construction —
no string concatenation with user-controlled values.

## Testing Strategy

**Unit tests (chat-teams):**
- `InvokeServiceTest` — handler discovery, dispatch by name, no-handler
  fallback (Accepted), duplicate handler detection (startup error)
- `InvokeResponseSerializerTest` — each variant produces correct Bot
  Framework JSON schema (Accepted, CardUpdate, Message, TaskModule, Error)
- `TeamsInboundConnectorTest` — new cases:
  - Invoke activity with registered handler → Responded with handler's
    response body + InboundMessage delivered
  - Invoke activity without handler → Responded with acknowledge +
    InboundMessage delivered
  - Invoke activity with null InvokeService (backward compat) → Responded
    with acknowledge
  - Invoke metadata correctness (teams-invoke-name, teams-invoke-data)
  - Self-message filtering still applies to invoke activities

**Unit tests (webhook):**
- `WebhookRouterTest` — new case: Responded variant delivers messages
  AND returns response body with correct content type

**Bot Framework invoke activity JSON fixtures** for tests:
- `adaptiveCard/action` with Action.Submit data
- `task/fetch` with task module request
- `task/submit` with task module form data

## References

- `connectors-api/src/main/java/.../WebhookResult.java` — sealed interface (6th variant)
- `webhook/src/main/java/.../WebhookRouter.java:92-127` — exhaustive switch to update
- `chat-teams/src/main/java/.../TeamsInboundConnector.java:90-99` — activity type switch
- `chat-spi/src/main/java/.../chat/command/CommandHandler.java` — prior art for handler SPI
- `chat-spi/src/main/java/.../chat/command/CommandService.java` — prior art for dispatch
- `chat-spi/src/main/java/.../chat/command/CommandResponse.java` — prior art for sealed response
- `chat-discord/src/main/java/.../discord/DiscordInteractionEndpoint.java` — Discord pattern (dedicated endpoint, rejected for Teams per D3/D4 from #220)
- `#220 decisions D4` — pre-validated Responded variant specification
- `#220 decisions D6` — Responded is only for invoke activities
- `casehubio/connectors#235` — issue
- `casehubio/connectors#236` — messaging extensions (composeExtension, deferred)
