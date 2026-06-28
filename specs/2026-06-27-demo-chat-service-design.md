# Demo Chat Service — Design Spec

A lightweight Quarkus chat application for demos and integration testing.
Extends the ChatPlatform SPI with new capabilities, adds SQLite persistence
as an internal backend, REST/WebSocket endpoints, and a casehub-pages UI.
No Docker, no external accounts — launches with `mvn quarkus:dev`.

## Deliverables

Two linked workstreams:

1. **Pages enhancements** — filed as issues against `casehubio/pages`,
   implemented by the pages session
2. **Connectors work** — ChatPlatform SPI extensions, internal storage
   backends, demo application module

## 1. Pages Enhancements (3 issues)

### 1.1 WebSocket Dataset Provider

Datasets declare `ws://` or `wss://` URLs instead of `http://`:

```typescript
dataset("messages", "ws://localhost:8080/ws/chat?dataset=messages");
```

Pages opens the WebSocket on first subscription. The server sends a
`snapshot` on connect, then incremental events.

**Server → browser message format:**

```json
{"type": "snapshot",  "rows": [["general", "agent-1", "Hello", "2026-06-27T10:00:00Z"]]}
{"type": "append",   "rows": [["general", "agent-2", "Hi back", "2026-06-27T10:00:01Z"]]}
{"type": "replace",  "key": "agent-1", "row": ["agent-1", "AWAY"]}
{"type": "remove",   "key": "msg-123"}
```

- `snapshot` — full dataset replacement (connect, reconnect, server reset)
- `append` — add rows to the end (messages, log entries)
- `replace` — update row by key column value (presence, metadata)
- `remove` — delete row by key (member left, reaction removed)

**Key column:** The dataset schema declares which column is the key for
`replace`/`remove`. Append-only datasets need no key.

**Incremental connect:** The WebSocket URL accepts an optional `since`
query parameter: `ws://host/ws/chat?dataset=messages&since=<ISO-8601>`.
When present, the server sends only events after that timestamp on
connect — no full snapshot. When absent, the server sends a full
`snapshot`. This prevents prohibitive payloads for large datasets
(monitoring dashboards with 100K rows) while keeping the simple default
for small datasets like chat.

**Reconnection:** Exponential backoff. On reconnect without `since`, the
server sends a fresh `snapshot`. Clients that track their last-seen
timestamp can reconnect with `since` for incremental recovery.

**Lifecycle:** WebSocket closes when the dataset has no active subscribers.

### 1.2 WebSocket Multiplexing

Multiple datasets share one WebSocket connection. Each message includes a
`dataset` field; the client demuxes by `dataset` query parameter:

```json
{"dataset": "messages",  "type": "append",  "rows": [...]}
{"dataset": "presence",  "type": "replace", "key": "agent-1", "row": [...]}
```

Pages detects that multiple datasets share the same base URL (same
host/path, differing only in `dataset` query param) and opens one
WebSocket. Incoming events are routed to the correct dataset by matching
the `dataset` field in the message to the query param each dataset
registered with.

**Subscription lifecycle:** Individual datasets can subscribe and
unsubscribe independently. The WebSocket connection closes only when ALL
datasets sharing that connection have zero active subscribers. A single
dataset unsubscribing (e.g., navigating away from messages while members
panel stays visible) does not close the connection.

### 1.3 Submit Action on Form Inputs

Form inputs gain a fire-and-forget mode for submitting to a REST endpoint:

```typescript
textInput({
  field: "message",
  submit: {
    url: "/api/channels/{channelId}/messages",
    method: "POST",
    clearOnSubmit: true,
  },
})
```

On Enter or button click, POST the field value to the URL and clear the
input. The response arrives back through the WebSocket dataset — the input
does not update the dataset directly.

Generalises to any form that submits to a REST endpoint rather than editing
an existing record.

## 2. ChatPlatform SPI Extensions

No new SPI is introduced. The existing ChatPlatform capability model is
extended with operations that real chat platforms support and the demo needs.
This follows the established design philosophy: composed capabilities with
auto-degradation, `supports(Class<?>)` using class tokens, and named
degradation types.

### 2.1 Extended Existing Capabilities

**Reactions** — gains `list()`:

```java
public interface Reactions {
    void add(ChatMessageRef message, String emoji);       // existing
    void remove(ChatMessageRef message, String emoji);    // existing
    List<String> list(ChatMessageRef message);             // new
}
```

Degraded: `NoOpReactions.list()` returns `List.of()`.

**Presence** — gains `set()`:

```java
public interface Presence {
    PresenceStatus of(MemberRef member);                   // existing
    void set(MemberRef member, PresenceStatus status);     // new
}
```

Degraded: `UnknownPresence.set()` is a no-op.

### 2.2 New Capability Interfaces

**ChannelManagement** — create and find channels. Separate from Discovery
because `supports(ChannelManagement.class)` must be independently queryable
— a platform can list channels without supporting creation.

```java
public interface ChannelManagement {
    Channel create(String name, String topic, String description, boolean isPrivate);
    Optional<Channel> find(String channelId);
}
```

Degraded: `NoOpChannelManagement` — `create()` throws
`UnsupportedOperationException`, `find()` returns `Optional.empty()`.

**MemberManagement** — add and remove channel members. Separate from
Members for the same reason — listing members ≠ managing membership.

```java
public interface MemberManagement {
    void add(ChatChannelRef channel, Member member);      // Member carries displayName
    void remove(ChatChannelRef channel, MemberRef member); // MemberRef sufficient for removal
}
```

Degraded: `NoOpMemberManagement` — both methods are no-ops.

**MessageHistory** — query past messages. Returns the existing
`ReceivedMessage` type (already in `chat-spi` model — has `MemberRef
sender`, `ChatContent content`, `Instant receivedAt`, etc.).

```java
public interface MessageHistory {
    List<ReceivedMessage> messages(ChatChannelRef channel, Instant since);
}
```

`since` is required (non-null). Pass `Instant.EPOCH` for all messages.

Degraded: `EmptyMessageHistory` — returns `List.of()`.

### 2.3 Channel Record Extended

The existing `Channel` record has `topic` but no `description`. These are
different concepts — topic is dynamic ("Sprint 47 standup"), description is
static ("Engineering general channel"). Slack and Discord distinguish them;
IRC has topic only.

```java
public record Channel(
    ChatChannelRef ref,
    String name,
    String topic,
    String description,     // new
    boolean isPrivate
) {}
```

Breaking change to the record constructor. IrcChatPlatform passes `null`
for description. All callers update — the migration is mechanical.

### 2.4 Builder and DefaultChatPlatform Updated

```java
// ChatPlatform interface gains:
ChannelManagement channelManagement();
MemberManagement memberManagement();
MessageHistory messageHistory();

// Builder gains:
public Builder channelManagement(ChannelManagement cm) { ... }
public Builder memberManagement(MemberManagement mm) { ... }
public Builder messageHistory(MessageHistory mh) { ... }
```

Builder defaults: `NoOpChannelManagement`, `NoOpMemberManagement`,
`EmptyMessageHistory`. Messaging remains required; all others degrade.

### 2.5 Sender — No Change to Messaging.send()

`Messaging.send(ChatChannelRef, ChatContent)` does NOT gain a sender
parameter. The sender is determined by the message direction:

- **Outbound** (CaseHub → platform via `messaging().send()`): sender is
  the platform's own identity. RefChatPlatform stores it as its `id()`.
- **Inbound** (external → platform): sender comes through the existing L2
  infrastructure: `InboundMessage` → `ChatInboundAdapter` →
  `InboundTranslator` → `ReceivedMessage(sender=MemberRef)`.

The demo's REST endpoint for posting messages fires `InboundMessage`
events with `connectorType = "ref"`. The existing `RefInboundTranslator`
(in `chat-ref`) translates them to `ReceivedMessage` with the agent's
identity as sender. No demo-specific translator is needed — the demo IS
the ref platform. This is architecturally identical to how IRC inbound
works — `IrcInboundConnector` → `IrcInboundTranslator` → `ReceivedMessage`.

Both inbound and outbound messages are stored internally and returned
by `MessageHistory.messages()`.

## 3. Internal Storage Backend

The persistence mechanism is internal to the reference implementation —
NOT a public SPI in chat-spi. IrcChatPlatform delegates to IrcClient,
a future SlackChatPlatform would delegate to SlackBotClient, and
RefChatPlatform delegates to its internal backend. Each implementation
owns its storage.

### 3.1 ChatBackend Interface

Lives in `chat-ref` (package-level visibility to chat-demo), not in
`chat-spi`:

```java
public interface ChatBackend {
    // Channels
    Channel createChannel(String name, String topic, String description, boolean isPrivate);
    Optional<Channel> findChannel(String channelId);
    List<Channel> listChannels();

    // Messages
    ReceivedMessage storeMessage(String platformId, ChatChannelRef channel,
                                 ChatContent content, MemberRef sender,
                                 ChatMessageRef parentRef);
    List<ReceivedMessage> messages(ChatChannelRef channel, Instant since);

    // Reactions
    void addReaction(ChatMessageRef message, String emoji);
    void removeReaction(ChatMessageRef message, String emoji);
    List<String> reactions(ChatMessageRef message);

    // Presence
    void setPresence(MemberRef member, PresenceStatus status);
    PresenceStatus presence(MemberRef member);

    // Members
    void addMember(ChatChannelRef channel, Member member);
    void removeMember(ChatChannelRef channel, MemberRef member);
    List<Member> members(ChatChannelRef channel);
}
```

This is deliberately a flat interface — it is implementation detail, not
a public contract. The capability decomposition lives in the SPI;
the backend is a single storage surface that the SPI delegates to.

### 3.2 Implementations

| Location | Class | CDI | Backing |
|----------|-------|-----|---------|
| `chat-ref` | `InMemoryChatBackend` | `@DefaultBean @ApplicationScoped` | ConcurrentHashMap + CopyOnWriteArrayList |
| `chat-demo` | `SqliteChatBackend` | `@ApplicationScoped` | SQLite file via JDBC |

Follows the connectors CDI pattern: `@DefaultBean` for the fallback
(InMemory), plain `@ApplicationScoped` for the real implementation
(SQLite) that overrides when present on the classpath.

### 3.3 RefChatPlatform Refactored

`RefChatPlatform` injects `ChatBackend` and wires all capabilities —
including the three new ones — to delegate to it:

```java
@ApplicationScoped
public class RefChatPlatform implements ChatPlatform {
    @Inject ChatBackend backend;

    @Override public Messaging messaging() {
        return (channel, content) -> {
            var msg = backend.storeMessage(id(), channel, content, new MemberRef(id()), null);
            return SendResult.success(msg.messageRef(), msg.receivedAt());
        };
    }

    @Override public MessageHistory messageHistory() {
        return (channel, since) -> backend.messages(channel, since);
    }

    @Override public ChannelManagement channelManagement() {
        return new ChannelManagement() {
            public Channel create(...) { return backend.createChannel(...); }
            public Optional<Channel> find(String id) { return backend.findChannel(id); }
        };
    }

    // ... all capabilities delegate to backend
}
```

All capabilities are natively supported. `supports()` returns true for
all capability classes.

## 4. SQLite Schema

Two shapes matching the domain:

```sql
-- Append-only
CREATE TABLE messages (
    id          TEXT PRIMARY KEY,
    channel_id  TEXT NOT NULL,
    parent_id   TEXT,
    sender_id   TEXT NOT NULL,
    content     TEXT NOT NULL,     -- JSON (ChatContent serialised)
    created_at  TEXT NOT NULL,     -- ISO-8601
    FOREIGN KEY (channel_id) REFERENCES channels(id)
);

-- Mutable state
CREATE TABLE channels (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    topic       TEXT,
    description TEXT,
    is_private  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE members (
    channel_id  TEXT NOT NULL,
    member_id   TEXT NOT NULL,
    display_name TEXT,
    PRIMARY KEY (channel_id, member_id)
);

CREATE TABLE presence (
    member_id   TEXT PRIMARY KEY,
    status      TEXT NOT NULL      -- ONLINE, OFFLINE, AWAY, DND, UNKNOWN
);

CREATE TABLE reactions (
    message_id  TEXT NOT NULL,
    emoji       TEXT NOT NULL,
    PRIMARY KEY (message_id, emoji)
);
```

### 4.1 Demo Fixtures

A pre-populated `demo-seed.db` shipped in
`chat-demo/src/main/resources/`. On first startup, if no database exists at
the configured path, the demo module copies the seed database.

Seed contains: two channels (`#general`, `#incidents`), three agents, a
short conversation with threading and reactions.

```properties
casehub.chat.backend.type=sqlite
casehub.chat.backend.path=chat-demo.db
casehub.chat.backend.seed=demo-seed.db
```

When `seed` is set and `path` doesn't exist, copy seed → path.

## 5. chat-demo Module

### 5.1 Inbound Message Path

Agent → demo messages use the existing L2 inbound infrastructure:

```
REST POST /api/channels/{id}/messages
  → fires InboundMessage (connectorType = "ref")
  → ChatInboundAdapter observes
  → RefInboundTranslator translates to ReceivedMessage(sender=MemberRef)
  → DemoMessageObserver stores in ChatBackend + pushes to WebSocket
```

No demo-specific translator or connector type constant is needed. The
demo fires `InboundMessage` with `connectorType = "ref"` and the existing
`RefInboundTranslator` (in `chat-ref`) handles translation.

The demo module provides:
- `DemoMessageObserver` — `@ObservesAsync ReceivedMessage` handler that
  stores the message in `ChatBackend` and pushes to WebSocket. Filters
  by `platformId` to only handle ref-platform messages.

### 5.2 REST Endpoints

Each endpoint maps to a ChatPlatform capability:

```
POST   /api/channels                    → channelManagement().create()
GET    /api/channels                    → discovery().listChannels()
POST   /api/channels/{id}/messages      → fires InboundMessage (L2 path, connectorType="ref")
GET    /api/channels/{id}/messages      → messageHistory().messages(?since=)
POST   /api/channels/{id}/messages/{msgId}/replies → fires InboundMessage with parentRef
POST   /api/channels/{id}/messages/{msgId}/reactions     → reactions().add()
DELETE /api/channels/{id}/messages/{msgId}/reactions/{e} → reactions().remove()
GET    /api/channels/{id}/messages/{msgId}/reactions     → reactions().list()
GET    /api/channels/{id}/members       → members().list()
POST   /api/channels/{id}/members       → memberManagement().add(Member)
DELETE /api/channels/{id}/members/{m}   → memberManagement().remove(MemberRef)
PUT    /api/presence/{memberId}         → presence().set()
GET    /api/presence/{memberId}         → presence().of()
```

Every endpoint exercises a real SPI capability. The demo validates the
ChatPlatform contract, not a parallel surface.

### 5.3 WebSocket Endpoint

Single endpoint at `/ws/chat`. Multiplexes events for all datasets
(messages, members, presence, channels).

When state changes (via REST or inbound events), the WebSocket pushes the
corresponding event to all connected browsers using the pages dataset
protocol from Section 1.

**Known limitation — outbound-to-WebSocket gap:** Mutations through
`messaging().send()` or other ChatPlatform SPI methods called directly
(not via REST) store to `ChatBackend` but do not push to WebSocket. Only
the inbound path (`ReceivedMessage` events observed by
`DemoMessageObserver`) and REST-triggered capability calls push. For the
standalone demo this is a non-issue — all external interaction comes
through REST/inbound. If the demo is later embedded as a test substrate
in a multi-module Quarkus app where other beans call ChatPlatform
directly, a CDI event on ChatBackend mutations would close this gap.

### 5.4 Module Structure

```
chat-demo/
  pom.xml
  src/main/java/                   ← REST resources, WebSocket, DemoMessageObserver
  src/main/resources/
    application.properties
    demo-seed.db
  src/main/webui/                  ← Quinoa (profile-gated, -Pdemo -Pui)
    package.json
    esbuild.config.mjs
    src/app.ts
  src/test/java/
```

**Profile gating:** The entire module is activated by `-Pdemo`. Excluded
from the default Maven reactor — `mvn clean install` never builds it.
Not published to GitHub Packages. Quinoa is doubly gated (`-Pdemo -Pui`).

### 5.5 Dependencies

```xml
<dependency>chat-spi</dependency>
<dependency>chat-ref</dependency>         <!-- RefChatPlatform + ChatBackend -->
<dependency>quarkus-websockets-next</dependency>
<dependency>quarkus-rest-jackson</dependency>
<dependency>quarkus-jdbc-sqlite</dependency>
<dependency>quarkus-quinoa (profile-gated)</dependency>
```

## 6. Pages Chat UI

Built with casehub-pages TypeScript DSL via Quinoa.

### 6.1 Layout

```typescript
page("Chat",
  columns([3, 7, 2],
    [sidebar(channels)],
    [rows(messageList, input)],
    [panel("Members", members)],
  ),
  { settings: { mode: "dark" } },
);
```

### 6.2 Datasets

All WebSocket-backed, multiplexed over `/ws/chat`:

```typescript
dataset("messages",  "ws://localhost:8080/ws/chat?dataset=messages");
dataset("members",   "ws://localhost:8080/ws/chat?dataset=members");
dataset("presence",  "ws://localhost:8080/ws/chat?dataset=presence");
dataset("channels",  "ws://localhost:8080/ws/chat?dataset=channels");
```

### 6.3 Components

- **Channel list** — `sidebar()` driven by `channels` dataset. Click
  filters the messages dataset via standard pages cross-filtering.
- **Message list** — `table()` with `messages` dataset. Columns: sender,
  content, timestamp. `append` events make new messages appear instantly.
  Custom CSS for chat-bubble styling.
- **Compose input** — `textInput()` with submit action (Section 1.3).
  POST to `/api/channels/{id}/messages`, clear on submit.
- **Member list** — `table()` with `members` dataset, cross-filtered with
  `presence` for status indicators.

## 7. Testing

### 7.1 ChatBackend Contract Tests

Abstract `ChatBackendContract` test class exercising the full internal
backend interface. Both `InMemoryChatBackend` and `SqliteChatBackend`
extend it and run the same tests.

### 7.2 ChatPlatform Capability Tests

Tests exercising the new capabilities through the ChatPlatform SPI:
- `channelManagement().create()` / `find()` — verify channel lifecycle
- `messageHistory().messages(channel, since)` — verify history query
- `reactions().list()` — verify reaction listing
- `presence().set()` / `of()` — verify presence roundtrip
- `memberManagement().add()` / `remove()` / `members().list()` — verify
  membership lifecycle

### 7.3 REST + WebSocket Integration Tests

`@QuarkusTest` with SQLite backend:
- POST message via REST → assert WebSocket pushes `append`
- POST message → assert `messageHistory().messages()` returns it with
  correct sender
- Set presence via REST → assert WebSocket pushes `replace`
- POST reply → assert parentRef is set correctly in stored message
- Connect WebSocket → assert `snapshot` with current state
- Disconnect + reconnect → assert fresh `snapshot`

### 7.4 Inbound Path Tests

Verify the L2 integration:
- POST message via REST → assert `InboundMessage` fires with
  `connectorType = "ref"`, correct `externalChannelRef`, `externalSenderId`,
  and `parent-id` in metadata (for replies)
- Assert `RefInboundTranslator` produces correct `ReceivedMessage`
- Assert sender propagates end-to-end from REST request to stored message

### 7.5 Demo Fixture Verification

Test that loads `demo-seed.db` and asserts expected channels, messages,
and members exist.

### 7.6 UI Verification

Manual — `mvn quarkus:dev -Pdemo -Pui` and browser. No automated UI tests
in initial scope.

## 8. Sequencing

1. ChatPlatform SPI extensions — new capabilities + extended interfaces
   (independent of pages and demo)
2. Channel record extended with `description` (breaking, mechanical
   migration)
3. **IrcChatPlatform updated** — breaking change: add 3 new methods
   (`channelManagement()`, `memberManagement()`, `messageHistory()`)
   returning `NoOpChannelManagement`, `NoOpMemberManagement`,
   `EmptyMessageHistory`. Update Channel constructor calls to include
   `description` parameter (null). Update `supports()` set. Degraded
   types (NoOpReactions, UnknownPresence) gain new methods automatically
   — no IrcChatPlatform change needed for those.
4. ChatBackend interface + InMemoryChatBackend in chat-ref
5. RefChatPlatform refactored to inject ChatBackend and wire all
   capabilities
6. File pages issues (3 issues against `casehubio/pages`)
7. chat-demo: SqliteChatBackend + REST + WebSocket + DemoMessageObserver
8. chat-demo: Quinoa UI (after pages issues are implemented)
