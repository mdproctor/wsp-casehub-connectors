# Decisions — #220 Teams Inbound via Bot Framework

## D1: Replace vs coexist with Outgoing Webhooks

**Choice:** Replace — Bot Framework fully replaces the existing Outgoing Webhooks connector
**Alternatives:**
- Coexist — keep both integration modes. More config surface, two code paths to maintain for diminishing benefit.
**Rationale:** Bot Framework subsumes everything Outgoing Webhooks does. One connector, full capability, simpler maintenance and docs.
**Trade-offs:** Orgs using Outgoing Webhooks today must migrate to an Azure Bot registration. Migration requires Azure AD app registration (admin permissions) and Bot Channels Registration. Migration path documented alongside D12 configuration properties.
**Sources:** webhook/src/main/java/.../webhook/TeamsInboundConnector.java (existing implementation)
**Exploration:** quick
**Status:** captured

## D2: JWT validation approach

**Choice:** nimbus-jose-jwt library — standard JOSE/JWT library for OpenID discovery, JWKS fetch, RSA key caching, RS256 validation, and claims checking
**Alternatives:**
- Custom JDK-only JWT validator — ~200–300 lines of security-critical code handling OpenID metadata discovery, JWKS fetching, key caching/rotation, multi-issuer validation, audience checking. Higher risk than single-primitive patterns (SigHelper HMAC, Discord Ed25519).
- SmallRye JWT — Quarkus-specific, creates Spring parity gap.
**Rationale:** Bot Framework JWT validation is a multi-step protocol (OpenID metadata → JWKS fetch → key cache + rotation on kid mismatch → RS256 + issuer/audience/expiry validation). This is fundamentally different from SigHelper (single HMAC primitive) or Discord Ed25519 (one signature check with static key). nimbus-jose-jwt handles the full protocol with battle-tested code, zero framework coupling, and works identically on Quarkus and Spring. It is the library underneath spring-security-oauth2-resource-server — standalone artifact, no framework dependency.
**Trade-offs:** One new dependency (nimbus-jose-jwt). No transitive framework coupling. Well-established library (used by Spring Security, Keycloak, Connect2id).
**Sources:** webhook/src/main/java/.../webhook/SigHelper.java (HMAC pattern — not analogous), chat-discord/src/main/java/.../discord/DiscordInteractionEndpoint.java (Ed25519 — not analogous), nimbus-jose-jwt library
**Exploration:** quick
**Status:** revised
**Revision:** Changed from custom JDK-only to nimbus-jose-jwt per R1-01. The HMAC/Ed25519 analogy was a false equivalence — those are single-primitive verifications, while JWT validation involves OpenID discovery, JWKS, key rotation, multi-issuer claims.

## D3: Module location

**Choice:** chat-teams module
**Alternatives:**
- webhook module — keeps webhook connectors together, but would split validation from platform logic. Bot Framework's single-endpoint architecture means all activity types share JWT validation — splitting across modules forces validation duplication or cross-module sharing.
**Rationale:** Bot Framework routes all activity types (messages, commands, reactions, invoke) through a single webhook endpoint with shared JWT validation. Unlike Slack (where Event Subscriptions and Slash Commands are separate APIs with separate auth and separate endpoints), Bot Framework's single-endpoint architecture makes colocation architecturally correct. The connector needs access to BotFrameworkClient, TeamsMapper, and the InboundTranslator — all in chat-teams.
**Trade-offs:** Existing TeamsInboundConnector in webhook module must be removed (D1).
**Sources:** chat-slack/src/main/java/.../slack/SlackCommandEndpoint.java (Slack split: separate API/auth), chat-slack/src/main/java/.../slack/SlackInboundConnector.java (webhook module — separate API), chat-discord/src/main/java/.../discord/DiscordInboundConnector.java (Discord: WebSocket pull model via InboundConnector, not WebhookInboundConnector)
**Exploration:** quick
**Status:** revised
**Revision:** Corrected rationale per R1-03. Discord is not the right precedent (Discord uses InboundConnector/WebSocket, not WebhookInboundConnector). The correct justification is Bot Framework's single-endpoint architecture where all activity types share one webhook and one auth path, unlike Slack where webhooks and commands are separate APIs.

## D4: Transport SPI

**Choice:** WebhookInboundConnector SPI with new `WebhookResult.Responded` variant
**Alternatives:**
- Standalone JAX-RS endpoint — duplicates webhook infrastructure that WebhookRouter already provides.
- Both (split by activity type) — WebhookInboundConnector for messages, separate endpoint for commands. More complex wiring for no gain since Bot Framework uses a single endpoint.
**Rationale:** Bot Framework is fundamentally a webhook — HTTP POST with JSON body. Using the existing SPI provides WebhookRouter dispatch, ID validation, error handling, HTTP status mapping. Route command/reaction/invoke activities internally. A new `WebhookResult.Responded` variant handles activities that require both message delivery AND a JSON response body (invoke activities for messaging extensions, adaptive card actions).
**Trade-offs:** New sealed type variant in `WebhookResult` — additive change that forces exhaustive switch updates in WebhookRouter (which is the point of the sealed type design).
**Sources:** connectors-api/src/main/java/.../WebhookInboundConnector.java, connectors-api/src/main/java/.../WebhookResult.java, webhook/src/main/java/.../webhook/WebhookRouter.java
**Exploration:** quick
**Status:** revised
**Revision:** Resolved WebhookResult gap per R1-02. Added `WebhookResult.Responded(String responseBody, String contentType, List<InboundMessage> messages)` variant. Challenged is semantically wrong for invoke responses (it means platform verification handshake). Responded is the correct semantic for activities that require both delivery and a JSON response body.

## D5: Issue scope

**Choice:** Full scope — inbound connector + inbound reaction events + commands
**Alternatives:**
- Foundation first — deliver inbound only, reactions and commands as follow-ups.
- Inbound + commands only — defer reactions.
**Rationale:** Ships complete Teams inbound capability in one issue. Bot Framework infrastructure supports all three; separating them would require revisiting the same Activity routing logic multiple times. "Reactions" means handling inbound `messageReaction` Activity events from Bot Framework. Outbound reactions (Reactions SPI: `add`/`remove`/`list` via Graph API) are a separate concern, use a different API surface, and belong in a separate issue.
**Trade-offs:** L/High becomes effectively XL/High. Larger implementation surface.
**Sources:** casehubio/connectors#220 (issue body), chat-spi/src/main/java/.../chat/spi/Reactions.java (outbound SPI — separate concern)
**Exploration:** quick
**Status:** revised
**Revision:** Clarified per R1-07 that "reactions" means inbound messageReaction Activity events only. Outbound Reactions SPI implementation uses Graph API, not Bot Framework, and is out of scope for this issue.

## D6: Command integration model

**Choice:** Message-embedded dispatch — commands arrive as message activities, connector parses command prefix, dispatches to CommandService. Both sync and async response paths supported.
**Alternatives:**
- Separate TeamsCommandEndpoint — artificial separation since Bot Framework doesn't have a separate command API.
**Rationale:** Matches how bots actually work in Teams. All activities arrive through the same webhook. Sync path: return `CommandResponse.Immediate` reply in HTTP response body via `WebhookResult.Responded` (lower latency). Async path: return 200 OK immediately, send `CommandResponse.Deferred` reply via `BotFrameworkClient` (required for long-running commands). Mirrors `DiscordInteractionEndpoint`'s dual-path model (type 4 sync / type 5 async) and `SlackCommandEndpoint`'s immediate/deferred split.
**Trade-offs:** Command parsing is text-based (prefix matching), less structured than Discord/Slack's typed command parameters. Sync path requires the `WebhookResult.Responded` variant (D4).
**Sources:** chat-discord/src/main/java/.../discord/DiscordInteractionEndpoint.java (sync type 4 + async type 5), chat-slack/src/main/java/.../slack/SlackCommandEndpoint.java (immediate + deferred), chat-spi/src/main/java/.../chat/command/CommandResponse.java (Immediate / Deferred sealed variants)
**Exploration:** quick
**Status:** revised
**Revision:** Updated per R1-08 to explicitly support both sync (HTTP response body) and async (Bot Framework REST API) command response paths. Coupled with D4's WebhookResult.Responded variant for the sync path.

## D7: Bot Framework client abstraction

**Choice:** BotFrameworkClient — dedicated class for all Bot Framework REST API calls
**Alternatives:**
- No client abstraction — embed HTTP calls directly in TeamsReactions/TeamsCommands. Fewer classes but tighter coupling.
**Rationale:** Mirrors TeamsClient/MicrosoftGraphClient pattern. Bot Framework outbound calls (reply, react, send proactive) need an HTTP client. Clean separation keeps TeamsInboundConnector focused on webhook handling.
**Trade-offs:** One additional class.
**Sources:** chat-teams/src/main/java/.../chat/teams/TeamsClient.java, microsoft-graph-client/src/main/java/.../microsoft/MicrosoftGraphClient.java
**Exploration:** quick
**Status:** captured

## D8: Outbound auth for BotFrameworkClient

**Choice:** Client credentials flow via ServiceConnectionProvider with a dedicated provider name (`"microsoft-bot"`)
**Alternatives:**
- Direct token management in BotFrameworkClient — embed OAuth client credentials flow inline. Duplicates token caching/refresh logic that ServiceConnectionProvider already provides.
- Reuse existing `"microsoft-app"` provider — conflates Graph API scope (`graph.microsoft.com`) with Bot Framework scope (`api.botframework.com`). Different audiences require separate token requests.
**Rationale:** TeamsChatPlatform already uses `ServiceConnectionProvider.getAccessToken(APPLICATION_ACTOR, "microsoft-app", "default")` for Graph API tokens. BotFrameworkClient targets `https://{serviceUrl}/v3/conversations/...` with scope `https://api.botframework.com/.default` — a different audience requiring a separate token. A new provider name (`"microsoft-bot"`) follows the existing pattern cleanly. Token caching, refresh, and error handling are delegated to the platform's ServiceConnectionProvider infrastructure.
**Trade-offs:** Requires registering the `"microsoft-bot"` provider in ServiceConnectionProvider configuration. The bot's `client_id` and `client_secret` (from Azure Bot registration) must be configured.
**Sources:** chat-teams/src/main/java/.../chat/teams/TeamsChatPlatform.java (resolveApplicationToken pattern), platform-api/.../authn/ServiceConnectionProvider.java (SPI)
**Exploration:** quick (surfaced by R1-05)
**Status:** captured

## D9: Service URL SSRF defense

**Choice:** Configurable hostname allowlist for Bot Framework service URLs
**Alternatives:**
- No validation — trust JWT-validated activities. Insufficient as defense-in-depth, and inconsistent with existing platform SSRF defense patterns.
- Hardcoded allowlist — less flexible for government/sovereign cloud deployments.
**Rationale:** Bot Framework activities include a `serviceUrl` field used by BotFrameworkClient for all outbound calls. An attacker-controlled serviceUrl is a classic SSRF vector. The platform already defends against SSRF: Discord CDN hostname allowlist (`cdn.discordapp.com`, `media.discordapp.net`), document-microsoft CDN allowlist. A configurable allowlist (default: `*.botframework.com`, `smba.trafficmanager.net`; configurable via MP Config for government clouds) is consistent and defense-in-depth.
**Trade-offs:** Additional validation step on every outbound call. Minimal performance impact — hostname check is O(1) against a small set.
**Sources:** ARC42STORIES.MD §3 (Discord CDN SSRF defense), document-microsoft module (CDN allowlist pattern)
**Exploration:** quick (surfaced by R1-06)
**Status:** captured

## D10: Connector ID and type reuse

**Choice:** Reuse `InboundConnectorIds.TEAMS_INBOUND` (`"teams-inbound"`) and `InboundConnectorTypes.TEAMS` (`"teams"`)
**Alternatives:**
- New ID/type (e.g., `"teams-bot-inbound"` / `"teams-bot"`) — preserves old ID for potential coexistence, but fragments the identity for the same logical channel.
**Rationale:** D1 replaces the Outgoing Webhook connector. The connector ID and type represent the channel (Teams), not the transport mechanism. Downstream code that routes on `InboundConnectorTypes.TEAMS` continues to work. The payload format changes (Outgoing Webhook JSON → Bot Framework Activity JSON), which requires a new `InboundTranslator` for Teams that parses Activity JSON. `ChatInboundAdapter` routes on `connectorType()`, so the translator is the natural adaptation point.
**Trade-offs:** Code that directly inspected `InboundMessage.metadata()` or raw body content assuming Outgoing Webhook JSON format will need updating. A Teams `InboundTranslator` (which doesn't exist yet) must be created to parse Bot Framework Activity JSON.
**Sources:** connectors-api/.../InboundConnectorIds.java, connectors-api/.../InboundConnectorTypes.java, chat-spi/.../chat/ChatInboundAdapter.java (routes on connectorType)
**Exploration:** quick (surfaced by R1-10)
**Status:** captured

## D11: Emulator and dev-mode auth

**Choice:** Configurable dev-mode flag to disable JWT validation for local development with Bot Framework Emulator
**Alternatives:**
- Require Azure Bot + tunnel (ngrok/dev tunnel) for all local development — higher friction, blocks local-only iteration.
- Accept emulator issuer in production — security risk.
**Rationale:** Bot Framework Emulator uses different auth (different issuer, or auth disabled entirely). A dev-mode MP Config flag (`casehub.connectors.teams-bot.dev-mode=true`) skips JWT validation when set, following the existing fail-soft credential pattern (blank credential → WARNING log and no-op). Must log SECURITY WARNING on every request processed without auth.
**Trade-offs:** Risk of accidentally enabling dev-mode in production. Mitigated by: (1) default false, (2) WARNING log on every unauthenticated request, (3) deployment-time config validation.
**Sources:** webhook/src/main/java/.../webhook/TeamsInboundConnector.java (sharedSecret.isBlank() → no-op pattern)
**Exploration:** quick (surfaced by R1-11)
**Status:** captured

## D12: Configuration properties

**Choice:** New MP Config namespace `casehub.connectors.teams-bot.*`
**Alternatives:**
- Reuse `casehub.connectors.teams-inbound.*` — confusing overlap with the old Outgoing Webhook config that used `shared-secret`.
**Rationale:** Clean break from Outgoing Webhook config. Properties: `casehub.connectors.teams-bot.app-id` (Azure Bot App ID, used as JWT audience), `casehub.connectors.teams-bot.app-secret` (client secret for outbound auth D8), `casehub.connectors.teams-bot.dev-mode` (D11), `casehub.connectors.teams-bot.service-url-allowlist` (D9 SSRF defense). Old `casehub.connectors.teams-inbound.shared-secret` is no longer used.
**Trade-offs:** Config migration for existing deployments. Migration guide documents: (1) Azure Bot registration steps, (2) new config properties, (3) removal of old shared-secret config.
**Sources:** webhook/src/main/java/.../webhook/TeamsInboundConnector.java (existing config pattern)
**Exploration:** quick (surfaced by R1-12)
**Status:** captured
