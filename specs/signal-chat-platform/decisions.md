# Signal Chat Platform — Design Decisions

## D1: Integration approach

**Choice:** signal-cli-rest-api as an external process, connector communicates via HTTP REST + WebSocket
**Alternatives:**
- signal-cli embedded (in-process Java library) — tightest control, but AGPL copyleft infects the entire platform
- signal-cli JSON-RPC daemon — similar separation to REST API but less documented, smaller community
**Rationale:** Process separation provides the strongest legal argument against AGPL copyleft propagation. The REST API is the most mature and well-documented interface (42 endpoints, 22k+ GitHub stars). WebSocket support for inbound message reception. No AGPL code enters our JVM.
**Trade-offs:** External runtime dependency (Docker container). signal-cli must be kept within ~3 months of Signal server changes. Network latency on every API call vs in-process.
**Sources:** signal-cli GitHub (AsamK/signal-cli), signal-cli-rest-api GitHub (bbernhard/signal-cli-rest-api), Signal Foundation license blog, AGPL-3.0 text
**Exploration:** deep-analysis
**Status:** captured

## D2: Module structure

**Choice:** Two modules — `signal` (HTTP client) + `chat-signal` (ChatPlatform SPI implementation)
**Alternatives:**
- Single `chat-signal` module containing both client and SPI impl — simpler, but breaks the repo's established pattern
- Single module with internal package separation — unusual pattern for this codebase
**Rationale:** Follows the established pattern in this repo (slack-bot + chat-slack, discord + chat-discord). Keeps the HTTP client reusable if a standalone outbound Signal connector is needed later.
**Trade-offs:** Two Maven modules to maintain instead of one. No second consumer of the Signal client exists today.
**Sources:** slack-bot module, chat-slack module, discord module, chat-discord module
**Exploration:** quick
**Status:** captured

## D3: Channel model

**Choice:** Discovery.listChannels() returns Signal groups only. 1:1 conversations are reachable via Messaging.send() with a phone-number ChatChannelRef but are not surfaced through Discovery.
**Alternatives:**
- Groups + 1:1 contacts — Discovery returns both, contacts as synthetic channels with phone number or profile name. Full visibility but blurs the channel abstraction.
**Rationale:** Matches how Slack and Discord handle DMs — they're reachable for messaging but don't appear in channel listings. Groups have names, topics, descriptions, and member counts that map cleanly to the Channel model. 1:1 conversations have none of these.
**Trade-offs:** Callers must know a phone number to message a 1:1 contact — they can't discover contacts through the ChatPlatform SPI.
**Depends on:** D1 (signal-cli-rest-api provides separate group and contact endpoints)
**Sources:** ChatPlatform SPI Channel model, Discord/Slack Discovery implementations, signal-cli-rest-api group endpoints
**Exploration:** quick
**Status:** captured

## D4: Inbound message reception

**Choice:** WebSocket-based real-time event stream from signal-cli-rest-api
**Alternatives:**
- Polling via GET /v1/receive/{number} — simpler but adds latency and wastes resources
- Outbound only — skip inbound, add later
**Rationale:** Real-time push matches the Discord Gateway pattern already established in this repo. WebSocket receives messages, reactions, typing indicators as they happen. signal-cli-rest-api documents WebSocket as the recommended approach for receiving.
**Trade-offs:** WebSocket connection must be maintained (reconnection logic needed). More complex than polling.
**Depends on:** D1 (signal-cli-rest-api provides the WebSocket endpoint)
**Sources:** signal-cli-rest-api WebSocket documentation, Discord Gateway pattern (DiscordGateway.java)
**Exploration:** quick
**Status:** captured
