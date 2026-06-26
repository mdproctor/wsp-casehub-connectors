# Chat IRC Module — Design Spec

**Issue:** casehubio/connectors#25
**Date:** 2026-06-25
**Status:** Approved (revised after two review passes)

## Overview

New `chat-irc` submodule implementing the `ChatPlatform` SPI (L7) for the IRC
protocol. First concrete platform adapter after the reference `InMemoryChatPlatform`
(`chat-ref`). Participates in both L2 (InboundMessage event bus, CloudEvents) and
L7 (Chat SPI) via the standard `InboundConnector` → `ChatInboundAdapter` →
`InboundTranslator` path.

Includes an `IrcClient` (raw TCP transport), an `IrcInboundConnector` (L2 lifecycle
and reconnect), an `IrcChatPlatform` (L7 adapter), an `IrcInboundTranslator`
(L2→L7 bridge), and an embedded pure-Java test IRC server for integration testing
with zero external dependencies.

## Module Structure

**Module:** `chat-irc` (directory), artifact `casehub-connectors-chat-irc`

**Dependencies:**
- `casehub-connectors-core` (compile) — `InboundConnector`, `InboundMessage`,
  `InboundMessageSink`, `InboundConnectorIds`, `InboundConnectorTypes`
- `casehub-connectors-chat-spi` (compile) — ChatPlatform SPI and model types
- `quarkus-arc` (compile) — CDI `@ApplicationScoped`
- `jandex-maven-plugin` (build) — CDI bean discovery
- `quarkus-junit` + `assertj-core` (test)

**Packages:**

| Package | Contents |
|---------|----------|
| `io.casehub.connectors.chat.irc` | `IrcClient`, `IrcChatPlatform`, `IrcInboundConnector`, `IrcInboundTranslator` |
| `io.casehub.connectors.chat.irc.protocol` | `IrcMessage`, `IrcParser`, `IrcCommand`, `ChannelInfo` |
| `io.casehub.connectors.chat.irc.test` (test) | `EmbeddedIrcServer` |

**Core additions (outside this module):**
- `InboundConnectorIds.IRC = "irc-inbound"` — new constant added to `core`
- `InboundConnectorTypes.IRC = "irc"` — already forward-declared in `core`
  (alongside `DISCORD`), confirming the platform anticipated IRC support

## Inbound Architecture

IRC uses a single persistent TCP connection for both sending and receiving — unlike
webhook platforms where outbound (HTTP API) and inbound (webhook POST) are separate
transports. Despite this, IRC participates in the standard L2 inbound path so that
all platform observers (CloudEvents, auditing, future adapters) see IRC traffic.

### Message flow

```
IRC PRIVMSG → IrcClient read loop → IrcInboundConnector
  → sink.receive(InboundMessage)
  → InboundConnectorService fires Event<InboundMessage>.fireAsync()
    → ConnectorsCloudEventAdapter (L6) → fires CloudEvent
    → ChatInboundAdapter → IrcInboundTranslator → fires Event<ReceivedMessage> (L7)
```

### InboundMessage field mapping

| InboundMessage field | IRC source |
|---------------------|------------|
| `connectorId` | `"irc-inbound"` (`InboundConnectorIds.IRC`) |
| `connectorType` | `"irc"` (`InboundConnectorTypes.IRC`) |
| `externalSenderId` | nick of the sender |
| `externalChannelRef` | channel name (e.g. `#general`) |
| `content` | message text |
| `attachments` | empty list — IRC has no attachments |
| `receivedAt` | `Instant.now()` at time of receipt |
| `metadata` | `{"nick-prefix": "nick!user@host"}` |
| `tenancyId` | null |

## IRC Protocol Layer

### IrcMessage

```java
record IrcMessage(String prefix, String command, List<String> params)
```

Represents a parsed IRC line. Covers named commands (`PRIVMSG`, `JOIN`) and numeric
replies (`001`, `353`). Prefix is the `nick!user@host` source (nullable for
client-originated messages). Last param handles the `:trailing` format per RFC 1459.

### IrcParser

Stateless utility:
- `IrcMessage parse(String line)` — raw IRC line → `IrcMessage`
- `String format(String command, String... params)` — builds outbound IRC line,
  auto-adds `:` to trailing param if it contains spaces

### IrcCommand

Enum of commands we send/handle:

**Commands:** `NICK`, `USER`, `JOIN`, `PART`, `PRIVMSG`, `PING`, `PONG`, `LIST`,
`NAMES`, `QUIT`

**Numeric replies:** `RPL_WELCOME(001)`, `RPL_NAMREPLY(353)`, `RPL_ENDOFNAMES(366)`,
`RPL_LIST(322)`, `RPL_LISTEND(323)`

10 commands total. The protocol layer has zero dependencies on the ChatPlatform SPI
or InboundConnector — pure IRC wire format, independently testable with
string-in/string-out unit tests.

### ChannelInfo

```java
record ChannelInfo(String name, int memberCount, String topic)
```

Protocol-level record returned by `IrcClient.listChannels()`. Lives in the protocol
package, not the SPI package — `IrcChatPlatform` maps it to `Channel` (Chat SPI
model).

## IrcClient

`@ApplicationScoped` CDI bean. Pure TCP transport — owns the socket connection and
IRC session state, but does NOT own reconnection or backoff. Shared by both
`IrcInboundConnector` (lifecycle + inbound) and `IrcChatPlatform` (outbound).

### Configuration

Via `@ConfigProperty` (IrcClient is the single owner of connection config):
- `casehub.connectors.chat-irc.host` (required) — hostname
- `casehub.connectors.chat-irc.port` (default: 6667)
- `casehub.connectors.chat-irc.nick` (required) — bot nick

### Lifecycle

- `boolean connect()` — opens socket, sends NICK/USER, blocks until RPL_WELCOME
  (001), starts background read thread. Returns `true` on success, `false` on
  failure. Called by `IrcInboundConnector`, not directly by application code.
- `void disconnect()` — sends QUIT, closes socket, stops read thread. Always
  succeeds (idempotent).
- `boolean isConnected()` — reflects actual TCP socket state

### Channel Operations

- `boolean join(String channel)` — sends JOIN, blocks until NAMES reply confirms.
  Returns `true` on success, `false` on timeout or if not connected.
- `void part(String channel)` — sends PART. Fire-and-forget.
- `boolean send(String channel, String message)` — sends PRIVMSG. Returns `true`
  if socket write succeeded, `false` if not connected.

### Query Operations

- `List<ChannelInfo> listChannels()` — sends LIST, collects RPL_LIST (322) until
  RPL_LISTEND (323). Returns empty list if not connected.
- `List<String> names(String channel)` — sends NAMES, collects RPL_NAMREPLY (353)
  until RPL_ENDOFNAMES (366). Returns nick list; empty list if not connected.

### Inbound Callback

- `void setMessageCallback(Consumer<IrcMessage> callback)` — set by
  `IrcInboundConnector` before `connect()`. The read loop calls this for every
  received PRIVMSG.

### Read Loop (background thread)

Single thread reading lines from socket, parsing via `IrcParser`:
- PING → auto-responds with PONG (keepalive)
- PRIVMSG → fires the registered `Consumer<IrcMessage>` callback
- Numeric replies → routed to waiting operations via `CompletableFuture`-based
  request/response collectors

### Request/Response Coordination

Operations expecting replies (JOIN→NAMES, LIST→LISTEND) register a collector before
sending the command. The read loop feeds matching replies into it.
`CompletableFuture` with 10-second timeout — no complex state machine.

### Thread Safety

One write lock for outbound commands (socket output stream). Read loop is a single
thread. Collector map is a `ConcurrentHashMap`.

### Failure Contract

IrcClient is pure transport — it does NOT reconnect or backoff.

- Read loop: on `IOException`, sets `isConnected() = false` and exits. Does not
  retry.
- `send()`, `join()`: return `false` when not connected
- `listChannels()`, `names()`: return empty list when not connected
- No unchecked exceptions thrown from any public method

Reconnection and backoff are `IrcInboundConnector`'s responsibility (see below).

## IrcInboundConnector

`@ApplicationScoped` CDI bean implementing `InboundConnector`. Owns the reconnect
loop and feeds inbound messages into the L2 event bus.

Follows `EmailInboundConnector` pattern: connector owns the loop, transport is
stateless.

### Configuration

Via `@ConfigProperty`:
- `casehub.connectors.chat-irc.channels` (optional) — comma-separated channels to
  auto-join on connect and re-join after reconnect

### Lifecycle

- `id()` → `"irc-inbound"` (`InboundConnectorIds.IRC`)
- `start(InboundMessageSink sink)` — stores sink, submits a background virtual
  thread running the connect loop
- `stop()` — sets `volatile boolean stopping = true`, calls `ircClient.disconnect()`

### Connect Loop

```
while (!stopping) {
    try {
        ircClient.setMessageCallback(msg -> deliverToSink(msg, sink));
        if (!ircClient.connect()) throw new IOException("connect failed");
        joinConfiguredChannels();
        awaitDisconnect();  // blocks until read loop exits
    } catch (Exception e) {
        if (!stopping) {
            consecutiveFailures++;
            Level level = consecutiveFailures >= 5 ? SEVERE : WARNING;
            LOG.log(level, "irc-inbound: connection failed (attempt "
                + consecutiveFailures + "): " + e.getMessage());
            sleepQuietly(backoffSeconds * 1000L);
            backoffSeconds = Math.min(backoffSeconds * 2, maxReconnectDelay);
        }
    }
}
```

Backoff resets on successful connect (same as `EmailInboundConnector`).

Channel re-join: `joinConfiguredChannels()` runs on every iteration — after initial
connect and after every reconnect. `IrcInboundConnector` owns the `channels` config,
so it knows which channels to re-join.

### InboundMessage Construction

When a PRIVMSG arrives on the read loop:
```java
sink.receive(new InboundMessage(
    InboundConnectorIds.IRC,          // "irc-inbound"
    InboundConnectorTypes.IRC,        // "irc"
    extractNick(ircMessage.prefix()), // sender nick
    ircMessage.params().get(0),       // channel
    ircMessage.params().get(1),       // message text
    List.of(),                        // no attachments
    Instant.now(),
    Map.of("nick-prefix", ircMessage.prefix()),
    null));                           // no tenancyId
```

## IrcInboundTranslator

`@ApplicationScoped` CDI bean implementing `InboundTranslator`. Converts
`InboundMessage` → `ReceivedMessage` for the Chat SPI path. Follows
`RefInboundTranslator` pattern.

- `connectorType()` → `"irc"` (`InboundConnectorTypes.IRC`)
- `translate(InboundMessage msg)` → `ReceivedMessage` with:
  - `platformId`: `"irc"`
  - `channel`: `ChatChannelRef(msg.externalChannelRef())`
  - `messageRef`: `ChatMessageRef(channel, UUID.randomUUID().toString())` — IRC
    PRIVMSGs have no native message ID; synthetic UUID matches `InMemoryStore`
    pattern
  - `parentRef`: `null` — IRC has no threaded replies
  - `sender`: `MemberRef(msg.externalSenderId())`
  - `content`: `ChatContent(msg.content(), null, msg.attachments())`
  - `receivedAt`: `msg.receivedAt()`

## IrcChatPlatform

`@ApplicationScoped` CDI bean implementing `ChatPlatform` directly (following
`RefChatPlatform` pattern — no builder). Injects `IrcClient` for outbound
operations. Owns no configuration — connection config belongs to `IrcClient`,
channel config belongs to `IrcInboundConnector`.

### Capability Mapping

| Capability | Implementation | Native? |
|-----------|---------------|---------|
| Messaging | `ircClient.send(channel, text)` → `SendResult.success(new ChatMessageRef(channelRef, UUID.randomUUID().toString()), Instant.now())` on success; `SendResult.failure("not connected")` when disconnected | Yes |
| Threading | `ChannelFallbackThreading` (degraded, from chat-spi) | No |
| Discovery | `ircClient.listChannels()` → maps each `ChannelInfo` to `Channel(ref, name, topic, false)` — `isPrivate = false` because IRC LIST only returns visible channels (secret/private channels are hidden by design) | Yes |
| Reactions | `NoOpReactions` (degraded, from chat-spi) | No |
| Presence | `UnknownPresence` (degraded, from chat-spi) | No |
| Members | `ircClient.names(channel)` → maps each nick to `Member(new MemberRef(nick), nick)` | Yes |

`supports()` returns `true` for: `Messaging`, `Discovery`, `Members`.
Returns `false` for: `Threading`, `Reactions`, `Presence`.

### Outbound SendResult Construction

IRC PRIVMSG is fire-and-forget — no server-assigned message ID in either direction.
`messaging().send()` constructs:
```java
SendResult.success(
    new ChatMessageRef(channelRef, UUID.randomUUID().toString()),
    Instant.now())
```
Synthetic UUID matches the inbound `IrcInboundTranslator` pattern and the
`InMemoryStore` precedent.

### Startup Ordering

`IrcChatPlatform` does no lifecycle management — `IrcInboundConnector.start()`
(called by `InboundConnectorService` on `@Observes StartupEvent`) connects the
shared `IrcClient`. By the time any caller uses
`ChatPlatformService.platform("irc").messaging().send(...)`, the client is connected
(or in backoff). If called before connection, outbound methods degrade gracefully:
`SendResult.failure("not connected")` / empty list.

## Embedded Test IRC Server

`EmbeddedIrcServer` in `src/test/java`. Minimal RFC 1459 server for testing only.

### API

- `EmbeddedIrcServer(int port)` — pass `0` for random port
- `start()` / `stop()` — lifecycle
- `getPort()` — actual bound port
- `getReceivedMessages()` — messages sent to channels (for asserting outbound)
- `sendToChannel(String channel, String nick, String message)` — inject a PRIVMSG as
  if another user sent it (for testing inbound path)

### Supported Commands

- NICK/USER → stores nick, sends 001 RPL_WELCOME
- JOIN → adds client to channel, sends 353/366 NAMES reply
- PART → removes client from channel
- PRIVMSG → stores for assertion, echoes to other clients in channel
- NAMES → sends 353/366 for requested channel
- LIST → sends 322/323 for all channels (member count and topic included)
- PING/PONG → responds to client PINGs, sends PINGs to test keepalive
- QUIT → closes connection cleanly

### Multi-Client

Accepts multiple connections (one thread per client via virtual threads). Required for
testing "another user sends a message" for the inbound path — one connection is the
bot, a second simulates an external user.

### Not Implemented

MODE, KICK, OPER, WHOIS, TLS, passwords, server-to-server, AWAY.

## Test Strategy

### Unit Tests (no server, no CDI)

- `IrcParserTest` — string-in/string-out. Parse raw lines, format commands. Edge
  cases: no prefix, empty trailing param, multiple spaces.
- `IrcMessageTest` — record construction, equality.
- `IrcInboundTranslatorTest` — `InboundMessage` → `ReceivedMessage` field mapping,
  synthetic message ID generation, null parentRef.

### Integration Tests (EmbeddedIrcServer, no CDI)

- `IrcClientTest` — against `EmbeddedIrcServer`. Connect/disconnect, join/part,
  send/receive PRIVMSG, listChannels (verify `ChannelInfo` fields), names,
  PING/PONG keepalive, failure behaviour when not connected (send returns false,
  listChannels returns empty list), read loop exits on server disconnect.
- Random port (`new EmbeddedIrcServer(0)`) to avoid port conflicts in CI.

### Contract Tests (EmbeddedIrcServer + CDI)

- `IrcChatPlatformTest` — mirrors `RefChatPlatformContractTest` pattern. Verifies
  ChatPlatform SPI contract through the IRC adapter: send message (verify
  `SendResult.ok()`, synthetic `messageRef`), list channels (verify `isPrivate =
  false`), list members, threading degrades to channel send, reactions are no-op,
  presence returns UNKNOWN.
- `@QuarkusTest` with config overrides pointing at embedded server.

### L2 Integration Tests (EmbeddedIrcServer + CDI)

- `IrcInboundConnectorTest` — verifies the full L2 path: embedded server simulates
  external user PRIVMSG → `InboundMessage` CDI event fires with correct fields
  (connectorId, connectorType, externalSenderId, externalChannelRef, content).
- Reconnect test: embedded server drops connection → `IrcInboundConnector` reconnects
  with backoff → channels re-joined → messages flow again.
- Inbound end-to-end: external PRIVMSG → `InboundMessage` event → `ChatInboundAdapter`
  → `IrcInboundTranslator` → `ReceivedMessage` event fires with correct platformId,
  channel, sender, content.

### Out of Scope

- Real IRC server testing (Ergo, Freenode)
- TLS
- IRCv3 capabilities (away-notify, etc.)

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Single module vs split | Single `chat-irc` | Matches `chat-ref`. No other consumer for IrcClient standalone. |
| L2 participation | Yes — `InboundConnector` + `InboundTranslator` | IRC must participate in the L2 event bus so CloudEvents, auditing, and all `InboundMessage` observers see IRC traffic. `EmailInboundConnector` is the precedent for persistent TCP connections. `InboundConnectorTypes.IRC` was already forward-declared in core, confirming the platform anticipated this. |
| L1 participation | No — no `Connector` implementation, no MCP tool | IRC messaging is available through `ChatPlatformService.platform("irc")` only, not through `ConnectorService.send()`. IRC's bidirectional persistent connection doesn't fit the fire-and-forget `Connector.send(ConnectorMessage)` contract. |
| Inbound path | Standard L2: `InboundConnector` → `InboundMessage` → `ChatInboundAdapter` → `IrcInboundTranslator` → `ReceivedMessage` | Bypassing L2 would make IRC invisible to CloudEvents and any observer of `InboundMessage`. |
| Reconnect ownership | `IrcInboundConnector` owns the reconnect loop; `IrcClient` is pure transport | Matches `EmailInboundConnector` pattern — `idleLoop()` owns reconnect, IMAP `Store`/`Folder` are stateless transport. IrcClient read thread exits on IOException; connector reconnects and re-joins channels. |
| Config ownership | `IrcClient` owns `host/port/nick`; `IrcInboundConnector` owns `channels`; `IrcChatPlatform` owns nothing | Connection params belong to the transport. Auto-join channels belong to the lifecycle manager. The outbound adapter injects IrcClient and delegates. |
| Test server | Embedded pure Java | No pure Java IRC server on Maven Central. 10 commands covers the full contract. |
| Presence | Degraded (`UnknownPresence`) | RFC 1459 has no passive AWAY notification. `Presence.of()` via WHOIS would be a blocking network call — violates the implicit cheap-call contract. IRCv3 `away-notify` could enable native presence in a future revision. |
| Threading | Degraded (`ChannelFallbackThreading`) | IRC has no native threaded replies. |
| Implementation pattern | Direct `implements ChatPlatform` | Follows `RefChatPlatform` pattern. Builder produces `DefaultChatPlatform` record — wrapping it in a CDI bean is boilerplate for no gain. |
| Config naming | `casehub.connectors.chat-irc.*` | Platform convention: `casehub.connectors.<module>.<property>`. Bare `irc.*` violates namespace. |
| Message IDs | Synthetic UUID per message (both inbound and outbound) | IRC PRIVMSGs have no native message ID in either direction. UUID generation matches `InMemoryStore` pattern. |
| Channel.isPrivate | Always `false` from LIST results | IRC LIST only returns visible channels — secret/private channels are hidden by design. No MODE query needed. |
| Failure on disconnect | Graceful degradation — `false` / empty lists from IrcClient; `SendResult.failure()` from IrcChatPlatform | Outbound operations never throw. IrcInboundConnector handles reconnection with exponential backoff. |
