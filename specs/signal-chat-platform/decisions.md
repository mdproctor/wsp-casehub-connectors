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
