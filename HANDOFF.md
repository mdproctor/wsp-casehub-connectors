# HANDOFF — casehub-connectors

## Last Session

Replaced the Outgoing Webhooks `TeamsInboundConnector` with a full Bot Framework integration. JWT validation via nimbus-jose-jwt, activity routing (message, messageReaction, conversationUpdate), command dispatch embedded in the connector, SSRF defense on outbound `serviceUrl`, entity-based bot mention stripping, and personal/1:1 chat support.

**Branch:** `issue-220-teams-inbound-bot-framework` — ready for work-end

**Issues closed this session:**

| # | Title | What changed |
|---|-------|-------------|
| #220 | Teams inbound via Bot Framework | `BotFrameworkAuth` (JWT/JWKS + dev-mode), `BotFrameworkClient` (outbound REST + SSRF), `TeamsInboundConnector` (activity routing + command dispatch), `TeamsInboundTranslator`, `TeamsCommands`; old connector removed from `webhook` module; `TeamsChatPlatform.withCommands()` wired; contributor guide updated |

**New classes (all in `chat-teams`):**

| Class | Responsibility |
|-------|---------------|
| `BotFrameworkAuth` | JWT validation — nimbus-jose-jwt, OpenID metadata discovery, JWKS cache (24h + kid-miss refresh), RS256 + claims, dev-mode bypass |
| `BotFrameworkClient` | Bot Framework REST API v3 — replyToActivity, sendToConversation. SSRF hostname allowlist. Auth via ServiceConnectionProvider("microsoft-bot") |
| `TeamsInboundConnector` | `WebhookInboundConnector` — activity routing, self-filter, mention stripping, command dispatch (Immediate + Deferred) |
| `TeamsInboundTranslator` | `InboundTranslator` — Activity → ReceivedMessage with activity type guard |
| `TeamsCommands` | `Commands` SPI — logging-only registration (matches SlackCommands pattern) |

**Removed:**
- `webhook/TeamsInboundConnector` (old Outgoing Webhooks model)
- `webhook/TeamsInboundConnectorTest`
- `WebhookBeans` Teams producer
- `WebhookRouterTest.get_teamsConnector_returns200Ignored` (coverage moved to chat-teams)

**Config namespace:** `casehub.connectors.teams-bot.*` (app-id, dev-mode, service-url-allowlist, openid-metadata-url, issuer)

**Issues filed for deferred scope:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #232 | Outbound reactions (Reactions SPI) via Bot Framework | M | Med |
| #233 | Attachment download from Bot Framework activities | S | Low |
| #234 | Proactive messaging (bot-initiated conversations) | M | Med |
| #235 | Adaptive Card action handling (invoke activities) | M | High |
| #236 | Messaging extensions support | L | High |
| #237 | Personal/1:1 chat outbound messaging via BotFrameworkClient | S | Med |

**Queue state after this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #223 | actorId resolution for per-user OAuth | M | Med |

**To resume:** `work continue` for #223.
