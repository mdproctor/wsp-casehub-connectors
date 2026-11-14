# Decisions — #220 Teams Inbound via Bot Framework

## D1: Replace vs coexist with Outgoing Webhooks

**Choice:** Replace — Bot Framework fully replaces the existing Outgoing Webhooks connector
**Alternatives:**
- Coexist — keep both integration modes. More config surface, two code paths to maintain for diminishing benefit.
**Rationale:** Bot Framework subsumes everything Outgoing Webhooks does. One connector, full capability, simpler maintenance and docs.
**Trade-offs:** Orgs using Outgoing Webhooks today must migrate to an Azure Bot registration.
**Sources:** webhook/src/main/java/.../webhook/TeamsInboundConnector.java (existing implementation)
**Exploration:** quick
**Status:** captured

## D2: JWT validation approach

**Choice:** Custom JDK-only JWT validator — fetch JWKS, cache RSA keys, validate with java.security
**Alternatives:**
- SmallRye JWT — Quarkus-specific, creates Spring parity gap. Would need spring-security-oauth2-resource-server for the Spring deployment path.
- Manual HMAC kept — hybrid approach, but HMAC is Outgoing Webhooks, not Bot Framework.
**Rationale:** Matches project pattern of raw HTTP + JDK crypto (SigHelper, Ed25519 in Discord). Zero new dependencies, works identically on Quarkus and Spring.
**Trade-offs:** ~100 lines of custom JWT code to maintain. No framework-managed key rotation — must implement cache-and-refresh ourselves.
**Sources:** webhook/src/main/java/.../webhook/SigHelper.java, chat-discord/src/main/java/.../discord/DiscordInteractionEndpoint.java (Ed25519), connectors-spring/pom.xml (Spring module)
**Exploration:** quick
**Status:** captured

## D3: Module location

**Choice:** chat-teams module
**Alternatives:**
- webhook module — keeps webhook connectors together, but Bot Framework is much richer than other webhooks (handles commands, reactions, not just messages).
**Rationale:** Follows Discord pattern where DiscordInboundConnector + DiscordInteractionEndpoint both live in chat-discord. Keeps all Teams logic together. Bot Framework client needs access to TeamsClient/TeamsMapper.
**Trade-offs:** Existing TeamsInboundConnector in webhook module must be removed.
**Sources:** chat-discord/ (module pattern), chat-teams/ (target module)
**Exploration:** quick
**Status:** captured

## D4: Transport SPI

**Choice:** WebhookInboundConnector SPI
**Alternatives:**
- Standalone JAX-RS endpoint — duplicates webhook infrastructure that WebhookRouter already provides.
- Both (split by activity type) — WebhookInboundConnector for messages, separate endpoint for commands. More complex wiring for no gain since Bot Framework uses a single endpoint.
**Rationale:** Bot Framework is fundamentally a webhook — HTTP POST with JSON body. Using the existing SPI provides WebhookRouter dispatch, ID validation, error handling, HTTP status mapping. Route command/reaction activities internally.
**Trade-offs:** WebhookResult doesn't have a "respond with custom JSON" variant — invoke activities may need Challenged() or a new variant.
**Sources:** connectors-api/src/main/java/.../WebhookInboundConnector.java, connectors-api/src/main/java/.../WebhookResult.java, webhook/src/main/java/.../webhook/WebhookRouter.java
**Exploration:** quick
**Status:** captured

## D5: Issue scope

**Choice:** Full scope — inbound connector + reactions + commands
**Alternatives:**
- Foundation first — deliver inbound only, reactions and commands as follow-ups.
- Inbound + commands only — defer reactions.
**Rationale:** Ships complete Teams capability parity in one issue. Bot Framework infrastructure supports all three; separating them would require revisiting the same Activity routing logic multiple times.
**Trade-offs:** L/High becomes effectively XL/High. Larger implementation surface.
**Sources:** casehubio/connectors#220 (issue body)
**Exploration:** quick
**Status:** captured

## D6: Command integration model

**Choice:** Message-embedded dispatch — commands arrive as message activities, connector parses command prefix, dispatches to CommandService, sends reply via API
**Alternatives:**
- Separate TeamsCommandEndpoint — artificial separation since Bot Framework doesn't have a separate command API.
**Rationale:** Matches how bots actually work in Teams. All activities arrive through the same webhook. Command responses go back via Bot Framework REST API (reply to conversation), not in the HTTP response body.
**Trade-offs:** Command parsing is text-based (prefix matching), less structured than Discord/Slack's typed command parameters.
**Sources:** chat-discord/src/main/java/.../discord/DiscordInteractionEndpoint.java, chat-slack/src/main/java/.../slack/SlackCommandEndpoint.java, chat-spi/src/main/java/.../chat/command/CommandService.java
**Exploration:** quick
**Status:** captured

## D7: Bot Framework client abstraction

**Choice:** BotFrameworkClient — dedicated class for all Bot Framework REST API calls
**Alternatives:**
- No client abstraction — embed HTTP calls directly in TeamsReactions/TeamsCommands. Fewer classes but tighter coupling.
**Rationale:** Mirrors TeamsClient/MicrosoftGraphClient pattern. Bot Framework outbound calls (reply, react, send proactive) need an HTTP client. Clean separation keeps TeamsInboundConnector focused on webhook handling.
**Trade-offs:** One additional class.
**Sources:** chat-teams/src/main/java/.../chat/teams/TeamsClient.java, microsoft-graph-client/src/main/java/.../microsoft/MicrosoftGraphClient.java
**Exploration:** quick
**Status:** captured
