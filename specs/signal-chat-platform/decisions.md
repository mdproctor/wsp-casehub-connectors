# Signal Chat Platform — Design Decisions

## D1: Integration approach

**Choice:** signal-cli-rest-api as an external process, connector communicates via HTTP REST + WebSocket
**Alternatives:**
- signal-cli embedded (in-process Java library) — tightest control, but AGPL copyleft infects the entire platform
- signal-cli JSON-RPC daemon — similar separation to REST API but less documented, smaller community
**Rationale:** Process separation provides the strongest legal argument against AGPL copyleft propagation. The REST API is the most mature and well-documented interface (42 endpoints, 22k+ GitHub stars). WebSocket support for inbound message reception. No AGPL code enters our JVM.
**Trade-offs:**
- External runtime dependency (Docker container) — every deploying application (qhorus, devtown, aml, clinical) needs a sidecar container provisioned, monitored, and maintained. Container crash detection and unavailability handling required in the connector.
- Phone number registration lifecycle — Signal requires interactive SMS/voice verification, periodic re-verification, and ongoing trust management. Not a static token like Slack/Discord.
- Single-maintainer supply-chain risk — signal-cli-rest-api is maintained by a single developer (bbernhard). Signal has historically broken third-party clients. The ~3 months buffer claimed for Signal server changes may be optimistic.
- Signal's anti-bot posture — unlike Slack/Discord which provide official bot APIs, Signal actively discourages automation. Rate limiting or account suspension is a deployment risk.
- E2E encryption security posture — the container holds decryption keys and accesses plaintext. Container access control, message logging policy, and key management are downstream implementation concerns that must be addressed in the deployment spec.
- Network latency on every API call vs in-process.
- signal-cli must be kept within ~3 months of Signal server changes.
**Sources:** signal-cli GitHub (AsamK/signal-cli), signal-cli-rest-api GitHub (bbernhard/signal-cli-rest-api), Signal Foundation license blog, AGPL-3.0 text
**Exploration:** deep-analysis
**Status:** captured

## D2: Module structure

**Choice:** Two modules — `signal-cli` (HTTP/WebSocket client) + `chat-signal` (ChatPlatform SPI implementation)
**Alternatives:**
- Single `chat-signal` module containing both client and SPI impl — simpler, but breaks the repo's established pattern and couples the client to `chat-spi`
- Single module with internal package separation — unusual pattern for this codebase
- `signal` as client module name — ambiguous with Java signal handling and reactive signal patterns
**Rationale:** Follows the established pattern in this repo (slack-bot + chat-slack, discord + chat-discord). The client module (`signal-cli`) has no dependency on `chat-spi`, keeping it independently usable — for example, by a standalone `SignalConnector` bean in the core module for notification-bridge integration (see D5). The `signal-cli` name explicitly references the signal-cli ecosystem and avoids ambiguity with Java `java.lang.Signal` or reactive signal patterns. WebSocket client code lives in `signal-cli` alongside the REST client because both connect to the same signal-cli-rest-api service — following the Discord pattern where `DiscordGateway` (WebSocket) lives in the `discord` module. The WebSocket is a transport concern of the signal-cli-rest-api connection, not an SPI concern.
**Trade-offs:** Two Maven modules to maintain instead of one. No second consumer of the Signal client exists today (though D5 identifies one: a Connector for notification-bridge).
**Depends on:** D1 (assumes signal-cli-rest-api integration approach)
**Sources:** slack-bot module, chat-slack module, discord module, chat-discord module
**Exploration:** quick
**Status:** revised (R1-05: renamed `signal` → `signal-cli` for disambiguation; clarified WebSocket placement rationale; added D1 dependency)

## D3: Channel model

**Choice:** Discovery.listChannels() returns both Signal groups and contacts. Groups map to Channel records with name, member count, and group ID ref. Contacts map to Channel records with profile name (or phone number), memberCount=2, isPrivate=true, and phone-number ChatChannelRef.
**Alternatives:**
- Groups only — Discovery returns Signal groups, 1:1 contacts reachable only via Messaging.send() with a known phone number. Matches Slack/Discord DM handling but marginalises Signal's primary use case.
- Contacts only — ignores Signal's group capabilities entirely
**Rationale:** Signal's primary interaction paradigm is 1:1 encrypted messaging — groups were added later and remain secondary. The Slack/Discord analogy (DMs reachable but not discoverable) is structurally wrong for Signal because it inverts the primary/secondary relationship. An LLM agent using `send_chat` must be able to discover both groups AND contacts to make the connector practically useful. Contacts as Channel records are sparse (topic=null, description=null) but all required fields populate: ref (phone number), name (profile name or phone number), isPrivate (true), memberCount (2). signal-cli-rest-api provides separate endpoints: `/v1/groups/{number}` for groups and `/v1/contacts/{number}` for registered contacts.
**Trade-offs:** Contact-as-channel blurs the Channel abstraction — a 1:1 conversation is not a "channel" in the Slack/Discord sense. Sparse Channel records (no topic, no description) are less informative but still functional.
**Depends on:** D1 (signal-cli-rest-api provides separate group and contact endpoints)
**Sources:** ChatPlatform SPI Channel model, Discord/Slack Discovery implementations, signal-cli-rest-api group and contact endpoints
**Exploration:** quick
**Status:** revised (R1-02: expanded Discovery to include contacts alongside groups; revised Slack/Discord analogy)

## D4: Inbound message reception

**Choice:** WebSocket-based real-time event stream from signal-cli-rest-api
**Alternatives:**
- Polling via GET /v1/receive/{number} — simpler but has queue-drain semantics (messages are consumed on read, not idempotent). A failed poll or missed interval permanently loses messages. More fragile than WebSocket with reconnection logic.
- Outbound only — skip inbound, add later
**Rationale:** Real-time push via WebSocket provides the lowest-latency, most reliable inbound path. signal-cli-rest-api documents WebSocket as the recommended approach for receiving. The Signal WebSocket protocol is substantially simpler than the Discord Gateway — no opcodes, no heartbeat protocol, no session resume, no intents-based filtering. It is a plain JSON event stream. Pattern reuse with DiscordGateway is limited to the WebSocket lifecycle concept (connect, receive events, reconnect on failure), not the protocol implementation. The implementation will be much simpler than `DiscordGateway`.
**Failure modes:**
- Double-hop reliability: WebSocket connects to signal-cli-rest-api container, which maintains its own connection to Signal servers. Container restarts, network partitions between JVM and container, and signal-cli's reconnection to Signal servers all add failure modes not present in Discord (direct WebSocket to Discord servers).
- Reconnection: exponential backoff with jitter, following the DiscordGateway reconnection pattern.
- Container health: connector should poll a health endpoint or detect WebSocket closure to distinguish container unavailability from Signal server issues.
**Event filtering:** The WebSocket delivers messages, reactions, delivery receipts, read receipts, typing indicators, sync messages (multi-device), and group update messages. The connector should consume messages and reactions (mapping to ChatPlatform events), filter typing indicators and receipts (no SPI mapping), and handle group updates for Discovery cache invalidation.
**Depends on:** D1 (signal-cli-rest-api provides the WebSocket endpoint)
**Sources:** signal-cli-rest-api WebSocket documentation, Discord Gateway pattern (DiscordGateway.java)
**Exploration:** quick
**Status:** revised (R1-06: refined Discord analogy, added failure modes and event filtering)

## D5: SPI level — ChatPlatform vs Connector

**Choice:** ChatPlatform SPI as the primary integration, with an optional `SignalConnector` (Connector SPI) for notification-bridge integration
**Alternatives:**
- Connector SPI only — fire-and-forget `send(ConnectorMessage)`. Automatic notification-bridge via `channelType()`. Dedicated `send_signal` MCP tool. No discovery, no reactions, no group operations, no message history. Matches WhatsApp/Twilio pattern.
- ChatPlatform SPI only — rich interaction model but no notification-bridge integration. Matches Discord pattern (no `DiscordConnector` exists today).
- Both (chosen) — ChatPlatform for rich interaction, Connector for notification-bridge. Matches Slack pattern (`SlackConnector` + `SlackChatPlatform`).
**Rationale:** Signal via signal-cli-rest-api supports 6 native ChatPlatform capabilities: Messaging, Discovery, Reactions, Members, ChannelManagement, MemberManagement. Only Threading (quote-replies, not threads), Presence (no online status), and MessageHistory (no server-side history) are degraded. This is richer than IRC (3 native: Messaging, Discovery, Members) and comparable to Discord (8 native). The Connector SPI's `send(ConnectorMessage)` is too thin for interactive messaging — no discovery, no reactions, no group management. Signal shares phone-number identity with WhatsApp/SMS but differs in capability depth: WhatsApp's Meta Cloud API is designed for business notification delivery (templates, 24-hour windows), while signal-cli-rest-api exposes a full messaging client with group management, reactions, and member operations. The `SignalConnector` bean in the core module provides notification-bridge integration via `channelType() → "signal"`, following the Slack dual-implementation pattern. `SignalConnector` uses raw HTTP via `HttpHelper.postJson()` independently of the `signal-cli` client module — only `chat-signal` (ChatPlatform) depends on `signal-cli`. This preserves core's zero inter-module dependency invariant.
**Trade-offs:** Two beans (ChatPlatform + Connector) instead of one. The Connector's `send()` duplicates a subset of ChatPlatform's `messaging().send()` at a lower abstraction level — specifically, `POST /v2/send` is implemented twice: raw HTTP in `SignalConnector` (core) and via `SignalClient.send()` (signal-cli). This duplication is the cost of preserving core's zero-dependency invariant. This is the same trade-off Slack already pays.
**Depends on:** D1 (integration approach), D2 (module structure)
**Sources:** ChatPlatform SPI (9 capability interfaces), Connector SPI, SlackConnector + SlackChatPlatform dual pattern, WhatsAppConnector (Connector-only), IrcChatPlatform (3/9 native), DiscordChatPlatform (8/9 native)
**Exploration:** quick (surfaced by reviewer R1-03 as implicit decision)
**Status:** revised (R1-03: SignalConnector uses raw HTTP in core, not signal-cli; D2 dependency note removed)
