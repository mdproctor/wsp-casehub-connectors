# Signal Chat Platform — Design Spec

## Overview

Add a Signal messenger integration to the connectors repo as a full `ChatPlatform` SPI implementation with 6 native capabilities (Messaging, Discovery, Members, Reactions, ChannelManagement, MemberManagement) and 3 degraded (Threading, Presence, MessageHistory). The connector communicates with Signal via an external `signal-cli-rest-api` Docker container over HTTP REST and WebSocket, avoiding AGPL license infection from signal-cli's dependency chain.

A secondary `SignalConnector` (Connector SPI) provides notification-bridge integration for one-way outbound delivery.

## Architecture

### Module Structure

Two new Maven modules, following the `slack-bot`/`chat-slack` and `discord`/`chat-discord` pattern:

```
connectors/
├── signal-cli/                    # HTTP/WebSocket client for signal-cli-rest-api
│   └── io.casehub.connectors.signal.cli
│       ├── SignalClient.java      # REST client
│       ├── SignalWebSocket.java   # WebSocket event listener
│       ├── SignalEventListener.java  # Callback interface for inbound events
│       └── model/                 # API DTOs
│           ├── SignalGroup.java
│           ├── SignalContact.java
│           ├── SignalMessage.java
│           ├── SignalReaction.java
│           └── SendResponse.java
│
├── chat-signal/                   # ChatPlatform SPI implementation + inbound pipeline
│   └── io.casehub.connectors.chat.signal
│       ├── SignalChatPlatform.java
│       ├── SignalInboundConnector.java   # InboundConnector SPI — WebSocket lifecycle
│       └── SignalInboundTranslator.java  # InboundTranslator — L7 message translation
│
├── core/                          # (existing) — add SignalConnector (raw HTTP, no signal-cli dep)
│   └── io.casehub.connectors.signal
│       └── SignalConnector.java
```

### Dependencies

- `signal-cli` → `jackson-databind`, `java.net.http.HttpClient` (no Quarkus REST client — keeps module framework-agnostic)
- `chat-signal` → `signal-cli`, `chat-spi` (core is transitive via chat-spi — needed for `InboundConnector`, `InboundConnectorTypes`, `InboundConnectorIds`)
- `core` — SignalConnector uses raw HTTP via `HttpHelper.postJson()` (no dependency on `signal-cli`; follows SlackConnector pattern)

### Runtime Dependency

`bbernhard/signal-cli-rest-api` Docker container. Configuration:

- `casehub.signal.api-url` — base URL of the signal-cli-rest-api instance (e.g. `http://localhost:8080`)
- `casehub.signal.number` — registered phone number (e.g. `+15551234567`)

## Integration Approach

The connector talks to `signal-cli-rest-api` (a Docker container wrapping `signal-cli`) over HTTP REST for outbound operations and WebSocket for inbound message reception. No AGPL-licensed code enters the JVM — the process boundary provides legal separation.

### Operational Considerations

- **Phone number registration:** Requires interactive SMS/voice verification, unlike Slack/Discord static tokens. Re-verification may be needed periodically.
- **Signal's anti-bot posture:** No official bot API. Rate limiting or account suspension is a deployment risk for high-volume sending.
- **Container lifecycle:** signal-cli must be kept within ~3 months of Signal server changes. Single-maintainer project (bbernhard).
- **E2E encryption:** The container holds decryption keys and accesses plaintext. Container access control is a deployment concern.

## Capability Mapping

### Native Capabilities (6)

| Capability | Implementation | Signal API Endpoints |
|---|---|---|
| Messaging | Send to group or 1:1 via phone number | `POST /v2/send` |
| Discovery | List groups + contacts as channels | `GET /v1/groups/{n}`, `GET /v1/contacts/{n}` |
| Members | Group member list from group details | `GET /v1/groups/{n}/{gid}` |
| Reactions | Add/remove emoji reactions; `list()` returns empty (see below) | `POST/DELETE /v1/reactions/{n}` |
| ChannelManagement | Create/delete/find groups | `POST/GET/DELETE /v1/groups/{n}/{gid}` |
| MemberManagement | Add/remove group members | `POST/DELETE /v1/groups/{n}/{gid}/members` |

### Degraded Capabilities (3)

| Capability | Degraded Implementation | Reason |
|---|---|---|
| Threading | `ChannelFallbackThreading` | Signal has quote-replies but no true threads |
| Presence | `UnknownPresence` | No online status API |
| MessageHistory | `EmptyMessageHistory` | E2E encrypted, no server-side history |

### Reactions — Partial Native Support

`Reactions.add()` and `Reactions.remove()` are fully native via signal-cli-rest-api endpoints. `Reactions.list()` returns `Collections.emptyList()` because Signal uses E2E encryption — reactions are delivered peer-to-peer and not stored server-side. There is no endpoint to query existing reactions on a message.

This is a known contract gap: `supports(Reactions.class)` returns `true`, but `list()` always returns empty. The alternative — demoting to `NoOpReactions` — would sacrifice working `add()`/`remove()` functionality. Callers that depend on `list()` should check the platform identity (`"signal"`) and handle accordingly.

### ChannelManagement — Group-Only Mutations

`ChannelManagement` operations behave differently for group channels vs contact channels:

| Method | Group channel (base64 ID) | Contact channel (phone number) |
|---|---|---|
| `find(channelId)` | Returns group-as-channel from `GET /v1/groups/{n}/{gid}` | Returns contact-as-channel from contacts list |
| `create(name, ...)` | Creates Signal group via `POST /v1/groups/{n}` | Throws `IllegalArgumentException` — contacts are not creatable |
| `delete(channelId)` | Leaves group via `DELETE /v1/groups/{n}/{gid}` | Throws `IllegalArgumentException` — contacts are not deletable |

The same `+`-prefix format detection used in `Messaging.send()` routing applies here. Contact channel IDs (phone numbers) are discovery-only — `create()` and `delete()` are group-only operations.

**`delete()` semantics:** Signal's `DELETE /v1/groups/{n}/{gid}` means "leave the group" from the bot's perspective. The group continues to exist for other members. This is a third semantic variant of the §12 risk #7 gap: Slack archives (reversible), Discord deletes (irreversible), Signal leaves (group persists). Documented in caller-facing Javadoc.

### Graceful Degradation

Following the Discord pattern: if `casehub.signal.api-url` or `casehub.signal.number` is blank, or the container is unreachable at startup, `@PostConstruct` sets all capabilities to degraded/no-op implementations with a warning log.

## Channel Model

Discovery returns both Signal groups and contacts as channels.

### Groups as Channels

| Channel field | Source |
|---|---|
| `ref.id` | signal-cli group ID (base64-encoded) |
| `name` | `group.name` |
| `topic` | `group.description` |
| `description` | `null` |
| `isPrivate` | `false` |
| `memberCount` | `group.members.size()` |

### Contacts as Channels

| Channel field | Source |
|---|---|
| `ref.id` | phone number (e.g. `+15551234567`) |
| `name` | `contact.profileName` (fallback: phone number) |
| `topic` | `null` |
| `description` | `null` |
| `isPrivate` | `true` |
| `memberCount` | `2` |

### Rationale

Signal's primary interaction paradigm is 1:1 encrypted messaging. Groups are secondary. Excluding contacts from Discovery would marginalise Signal's primary use case and prevent agent discovery of messaging targets.

### Routing by ChatChannelRef.id Format

`Messaging.send()` must distinguish group vs 1:1 targets because signal-cli-rest-api uses different JSON fields. Detection is by format: phone numbers start with `+` (e.g. `+15551234567`), group IDs are base64-encoded strings. The `SignalClient.send()` method inspects the recipient and serializes to the appropriate JSON field.

## Message Identity

Signal identifies messages by a `(sender, timestamp)` pair, not a single opaque ID. `ChatMessageRef.messageId` encodes both as `sender:timestamp` (e.g. `+15551234567:1724025600000`). This compound encoding is necessary because reactions and quote-replies both require the target author and timestamp.

## `signal-cli` Module

### SignalClient

REST client using `java.net.http.HttpClient` with Jackson serialization.

| Method | Endpoint | Purpose |
|---|---|---|
| `send(number, recipient, message, attachments)` | `POST /v2/send` | Send message |
| `listGroups(number)` | `GET /v1/groups/{n}` | List all groups |
| `getGroup(number, groupId)` | `GET /v1/groups/{n}/{gid}` | Group details with members |
| `createGroup(number, name, members)` | `POST /v1/groups/{n}` | Create group |
| `deleteGroup(number, groupId)` | `DELETE /v1/groups/{n}/{gid}` | Delete group |
| `addMembers(number, groupId, members)` | `POST /v1/groups/{n}/{gid}/members` | Add members |
| `removeMembers(number, groupId, members)` | `DELETE /v1/groups/{n}/{gid}/members` | Remove members |
| `addReaction(number, recipient, emoji, targetAuthor, timestamp)` | `POST /v1/reactions/{n}` | React to message |
| `removeReaction(number, recipient, emoji, targetAuthor, timestamp)` | `DELETE /v1/reactions/{n}` | Remove reaction |
| `listContacts(number)` | `GET /v1/contacts/{n}` | List contacts |
| `downloadAttachment(attachmentId)` | `GET /v1/attachments/{id}` | Download attachment content |
| `health()` | `GET /v1/health` | Container health check |

### SignalWebSocket

WebSocket client connecting to signal-cli-rest-api for real-time inbound events.

- **Client implementation:** JDK `java.net.http.WebSocket` — no additional dependency. The Discord Gateway required Vert.x `WebSocketClient` due to an RFC 6455 `Sec-WebSocket-Accept` validation issue in the JDK client (#35). This does not apply to signal-cli-rest-api: it runs locally as a Docker container using a standard Go WebSocket server (gorilla/websocket or equivalent), which provides compliant RFC 6455 handshakes. If JDK compatibility issues arise during implementation, Vert.x is the proven fallback.
- Parses JSON events, dispatches to `SignalEventListener` callback interface
- Reconnection: exponential backoff with jitter
- Container health: detect WebSocket closure, fall back to health endpoint polling
- Event filtering: consume messages and reactions, drop typing indicators, receipts, sync messages
- Simpler than Discord Gateway — plain JSON stream, no opcodes/heartbeat/intents

### Model DTOs

Plain Java records mapping the API's JSON structures: `SignalGroup`, `SignalContact`, `SignalMessage`, `SignalReaction`, `SendResponse`.

## `chat-signal` Module

### SignalChatPlatform

`@ApplicationScoped` CDI bean implementing `ChatPlatform`.

**Initialization:**
```
@PostConstruct init():
  if apiUrl or number is blank → initDegraded(), return
  if health check fails → initDegraded(), log warning, return
  initialize 6 native capabilities + 3 degraded
```

**Member mapping:** `MemberRef.id` = phone number, `displayName` = profile name (fallback: phone number).

### Inbound Pipeline (L2 → L7)

Follows the established two-stage inbound architecture (Discord, IRC, Slack pattern).

**Required additions to `core`:**
- `InboundConnectorTypes.SIGNAL = "signal"` — semantic type constant for CloudEvent routing and `ChatInboundAdapter` translator lookup
- `InboundConnectorIds.SIGNAL_INBOUND = "signal-inbound"` — connector instance ID

**Stage 1 — `SignalInboundConnector` (L2, in `chat-signal`):**

Implements `InboundConnector`. Manages `SignalWebSocket` lifecycle.

```
start(sink):
  create SignalWebSocket with SignalEventListener callback
  on message received →
    attachments = parseAttachments(event)  // download via SignalClient.downloadAttachment()
    metadata = {"signal-sender": sender, "signal-timestamp": timestamp}
    if event has quote →
      metadata += {"signal-quote-sender": quote.sender, "signal-quote-timestamp": quote.timestamp}
    construct InboundMessage(
      connectorId = SIGNAL_INBOUND,
      connectorType = SIGNAL,
      externalSenderId = sender phone number,
      externalChannelRef = group ID or sender phone number,
      content = message text,
      attachments = attachments,
      receivedAt = Instant.now(),
      metadata = metadata
    )
    sink.receive(msg)

stop():
  disconnect SignalWebSocket

id() → "signal-inbound"
```

**Attachment handling:** signal-cli-rest-api WebSocket events include attachment IDs for received messages. `SignalInboundConnector` fetches attachment content via `SignalClient.downloadAttachment(attachmentId)` and constructs `Attachment` records (filename, contentType, byte content). Following the `DiscordInboundConnector` pattern, download failures are tracked in metadata (`signal-attachment-count`, `signal-attachment-download-failures`) and the message is delivered with successfully downloaded attachments only.

`InboundConnectorService` discovers this bean at startup, calls `start(sink)`, and fires `Event<InboundMessage>.fireAsync()` on each received message. This delivers to:
- `ConnectorsCloudEventAdapter` — creates CloudEvent with type `io.casehub.connectors.inbound.signal`
- `ChatInboundAdapter` — routes to `SignalInboundTranslator` by `connectorType()`
- Any other `@ObservesAsync InboundMessage` observers

**Stage 2 — `SignalInboundTranslator` (L7, in `chat-signal`):**

Implements `InboundTranslator`. Translates `InboundMessage` → `ReceivedMessage`.

```
connectorType() → "signal"

translate(msg):
  channel = ChatChannelRef(msg.externalChannelRef())
  messageRef = ChatMessageRef(channel, msg.metadata("signal-sender") + ":" + msg.metadata("signal-timestamp"))
  quoteSender = msg.metadata("signal-quote-sender")
  quoteTs = msg.metadata("signal-quote-timestamp")
  parentRef = (quoteSender != null && quoteTs != null)
      ? ChatMessageRef(channel, quoteSender + ":" + quoteTs)
      : null
  sender = MemberRef(msg.externalSenderId())
  content = ChatContent(msg.content(), null, msg.attachments(), List.of())
  return ReceivedMessage("signal", channel, messageRef, parentRef, sender, content, msg.receivedAt())
```

`ChatInboundAdapter` fires `Event<ReceivedMessage>.fireAsync()` after translation.

## SignalConnector (in `core` module)

Simple `Connector` SPI bean for notification-bridge integration, using raw HTTP via `HttpHelper.postJson()` — no dependency on the `signal-cli` module. Follows the `SlackConnector` pattern: inline JSON construction with `HttpHelper.jsonQuote()`.

- `id()` → `"signal"`
- `channelType()` → `"signal"` (auto-registers with `DeliveryChannelRegistry`)
- `send(ConnectorMessage)` → `POST /v2/send` to signal-cli-rest-api via `HttpHelper.postJson()`
  - `message.destination()` is the recipient: phone number (1:1) or base64 group ID
  - Format detection: `+` prefix → `"recipients"` JSON field; otherwise → `"base64_group_id"` JSON field
  - `message.body()` → `"message"` JSON field
- Config: `@ConfigProperty(name = "casehub.signal.api-url")`, `@ConfigProperty(name = "casehub.signal.number")`
- Follows the Slack dual-implementation pattern (`SlackConnector` + `SlackChatPlatform`)

## Testing Strategy

- **SignalClient:** Unit tests with mock HTTP server (WireMock or similar) verifying request serialization and response deserialization for each endpoint
- **SignalChatPlatform:** Unit tests verifying capability mapping — group→Channel, contact→Channel, message identity encoding/decoding, graceful degradation when unconfigured
- **SignalWebSocket:** Unit tests for event parsing and filtering, reconnection backoff logic
- **SignalInboundConnector:** Unit tests verifying `start(sink)` delivers `InboundMessage` with correct `connectorId`, `connectorType`, and metadata; `stop()` disconnects WebSocket
- **SignalInboundTranslator:** Unit tests verifying `InboundMessage` → `ReceivedMessage` translation, `sender:timestamp` message identity encoding/decoding, `connectorType()` returns `InboundConnectorTypes.SIGNAL`
- **SignalConnector:** Unit tests for raw HTTP JSON construction — group vs 1:1 format detection, payload serialization
- **Integration:** `@QuarkusTest` verifying CDI wiring — `SignalChatPlatform` registers with `ChatPlatformService`, `SignalConnector` registers with notification-bridge, `SignalInboundConnector` discovered by `InboundConnectorService`
- **No live Signal dependency in tests:** All tests use mocked HTTP/WebSocket endpoints

## References

- `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java` — SPI contract
- `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcChatPlatform.java` — simplest connector (3 native)
- `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java` — closest analogue (8 native, WebSocket inbound)
- `discord/src/main/java/io/casehub/connectors/discord/DiscordGateway.java` — WebSocket lifecycle pattern
- `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java` — HTTP client pattern
- [signal-cli](https://github.com/AsamK/signal-cli) — underlying Signal protocol implementation (AGPL-3.0)
- [signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) — Docker REST/WebSocket wrapper
- [signal-cli-rest-api API docs](https://bbernhard.github.io/signal-cli-rest-api/) — endpoint reference
- D1–D5 in `specs/signal-chat-platform/decisions.md` — design decisions with rationale
