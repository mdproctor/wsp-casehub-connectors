# Teams Inbound via Bot Framework — Design Spec

**Issue:** casehubio/connectors#220
**Branch:** `issue-220-teams-inbound-bot-framework`
**Scale:** L → XL | **Complexity:** High

## Summary

Replace the existing `TeamsInboundConnector` (Outgoing Webhooks model with HMAC-SHA256 shared secret) with a Bot Framework integration that receives all Teams activity types through a single webhook endpoint. This delivers:

1. **Inbound messages** — Bot Framework `message` activities parsed into `InboundMessage` and translated to `ReceivedMessage` via a new `TeamsInboundTranslator`
2. **Inbound reaction events** — `messageReaction` activities delivered as `InboundMessage` with reaction metadata
3. **Slash commands** — commands embedded in message activities, dispatched to `CommandService`, with sync and async response paths
4. **Bot Framework client** — outbound REST API for reply, proactive messaging, and deferred command responses

The existing `TeamsInboundConnector` in the `webhook` module is removed. The new implementation lives entirely in `chat-teams`.

## Architecture

### Module Changes

**`chat-teams` (modified)** — all new classes live here:

| Class | Responsibility |
|-------|---------------|
| `TeamsInboundConnector` | `WebhookInboundConnector` — JWT validation via `BotFrameworkAuth`, Activity JSON parsing, routing by `activity.type` |
| `BotFrameworkAuth` | JWT validation — OpenID metadata discovery, JWKS key cache with 24h refresh + kid-mismatch refresh, RS256 signature + claims checking via nimbus-jose-jwt |
| `BotFrameworkClient` | Outbound Bot Framework REST API v3 — reply to activity, send to conversation. Auth via `ServiceConnectionProvider("microsoft-bot")`. SSRF defense on `serviceUrl`. Uses `HttpHelper.CLIENT` |
| `TeamsInboundTranslator` | `InboundTranslator` — `connectorType() = "teams"`, translates `InboundMessage` (Bot Framework Activity JSON in metadata) → `ReceivedMessage` with proper `ChatChannelRef` (compound `teamId:channelId`), `ChatMessageRef`, `MemberRef`, `ChatContent` |
| `TeamsCommands` | `Commands` SPI — registers bot commands via `BotFrameworkClient` (not applicable — Teams bot commands are registered in the app manifest, not via API). Handles command dispatch from message activities |

**`connectors-api` (modified):**

| Change | Detail |
|--------|--------|
| `WebhookResult.Responded` | New sealed variant: `Responded(String responseBody, String contentType, List<InboundMessage> messages)` — for activities requiring both message delivery AND a JSON response body (invoke activities, card actions) |

**`webhook` (modified):**

| Change | Detail |
|--------|--------|
| Remove `TeamsInboundConnector` | Old Outgoing Webhooks implementation |
| Remove `TeamsInboundConnectorTest` | Associated tests |
| Update `WebhookBeans` | Remove Teams producer method |
| Update `WebhookRouter` | Handle new `Responded` variant — deliver messages then return HTTP 200 with response body |

**`chat-teams/pom.xml` (modified):**

| Dependency | Scope |
|-----------|-------|
| `nimbus-jose-jwt` | compile |
| `casehub-connectors-api` (existing) | — already present via chat-spi |
| `casehub-connectors-webhook` | compile — for `WebhookInboundConnector` base class |

### Activity Flow

```
POST /connectors/teams-inbound/webhook
  │
  ├── BotFrameworkAuth.validate(Authorization header)
  │     ├── Fetch OpenID metadata (cached): https://login.botframework.com/v1/.well-known/openidconfiguration
  │     ├── Fetch JWKS (cached 24h, refresh on kid mismatch): https://login.botframework.com/v1/.well-known/keys
  │     ├── Validate RS256 signature against matching kid
  │     ├── Check claims: iss=https://api.botframework.com, aud=APP_ID, exp
  │     └── Dev-mode (D11): skip validation, log SECURITY WARNING per request
  │
  ├── Parse Activity JSON (jakarta.json)
  │
  └── Route by activity.type:
        │
        ├── "message"
        │     ├── Check for command prefix (registered command names)
        │     │     ├── YES → CommandService.dispatch()
        │     │     │         ├── Immediate → reply via WebhookResult.Responded (sync)
        │     │     │         └── Deferred  → reply via BotFrameworkClient (async), return Ignored
        │     │     └── NO → Return Delivered(InboundMessage)
        │     │                → ChatInboundAdapter → TeamsInboundTranslator → ReceivedMessage
        │
        ├── "messageReaction"
        │     └── Return Delivered(InboundMessage)
        │           metadata keys:
        │             teams-reaction-type: like|heart|laugh|surprised|sad|angry
        │             teams-reaction-action: added|removed
        │             teams-reaction-message-id: <replyToId of reacted message>
        │
        ├── "conversationUpdate"
        │     └── Return Ignored (log member join/leave at INFO)
        │
        └── "invoke"
              └── Return Responded(JSON body, "application/json", messages)
                    For card actions and messaging extensions
```

### Activity JSON → InboundMessage Mapping

Bot Framework Activity fields mapped to `InboundMessage`:

| InboundMessage field | Activity source |
|---------------------|----------------|
| `connectorId` | `InboundConnectorIds.TEAMS_INBOUND` (`"teams-inbound"`) |
| `connectorType` | `InboundConnectorTypes.TEAMS` (`"teams"`) |
| `externalSenderId` | `activity.from.aadObjectId` (preferred) or `activity.from.id` |
| `externalChannelRef` | `channelData.team.id` + `:` + `channelData.channel.id` (compound format matching existing `TeamsMapper.parseTeamAndChannel`) |
| `content` | `activity.text` (with bot mention stripped — Teams prepends `<at>BotName</at>`) |
| `attachments` | `List.of()` (attachment download deferred to future work) |
| `receivedAt` | `Instant.parse(activity.timestamp)` |
| `tenancyId` | `channelData.tenant.id` |
| `metadata` | See below |

**Metadata keys:**

| Key | Value | When present |
|-----|-------|-------------|
| `teams-activity-id` | `activity.id` | Always |
| `teams-activity-type` | `activity.type` | Always |
| `teams-service-url` | `activity.serviceUrl` | Always |
| `teams-conversation-id` | `activity.conversation.id` | Always |
| `teams-conversation-type` | `activity.conversation.conversationType` (`"channel"` or `"personal"`) | Always |
| `teams-from-name` | `activity.from.name` | When present |
| `teams-reply-to-id` | `activity.replyToId` | When present (threading) |
| `teams-reaction-type` | Reaction type string | messageReaction only |
| `teams-reaction-action` | `"added"` or `"removed"` | messageReaction only |
| `teams-reaction-message-id` | `activity.replyToId` | messageReaction only |

### TeamsInboundTranslator

Translates `InboundMessage` → `ReceivedMessage` for the chat pipeline:

| ReceivedMessage field | Source |
|----------------------|--------|
| `platformId` | `"microsoft-teams"` (matches `TeamsChatPlatform.PLATFORM_ID`) |
| `channel` | `ChatChannelRef(msg.externalChannelRef())` — compound `teamId:channelId` |
| `message` | `ChatMessageRef(channel, metadata["teams-activity-id"])` |
| `parent` | `ChatMessageRef(channel, metadata["teams-reply-to-id"])` if present, else `null` |
| `sender` | `MemberRef(msg.externalSenderId())` |
| `content` | `ChatContent(msg.content())` |
| `receivedAt` | `msg.receivedAt()` |

### BotFrameworkClient

Outbound REST API calls to `{serviceUrl}/v3/conversations/...`:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `replyToActivity(serviceUrl, conversationId, activityId, text)` | `POST /v3/conversations/{conversationId}/activities/{activityId}` | Reply to a specific message |
| `sendToConversation(serviceUrl, conversationId, text)` | `POST /v3/conversations/{conversationId}/activities` | Send a new message to a conversation |

**Auth:** Bearer token via `ServiceConnectionProvider.getAccessToken(APPLICATION_ACTOR, "microsoft-bot", "default")`. Token endpoint: `https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token`, scope: `https://api.botframework.com/.default`.

**SSRF defense:** Before every outbound call, validate `serviceUrl` hostname against a configurable allowlist (default: `*.botframework.com`, `smba.trafficmanager.net`). Reject with `IllegalArgumentException` if hostname not in allowlist.

**HTTP client:** `HttpHelper.CLIENT` (shared `java.net.http.HttpClient` used across the project).

### BotFrameworkAuth

JWT validation using nimbus-jose-jwt:

```
BotFrameworkAuth(appId, devMode)
  │
  ├── validate(authorizationHeader) → boolean
  │     ├── If devMode → log SECURITY WARNING, return true
  │     ├── Extract Bearer token
  │     ├── Parse JWT header → get kid
  │     ├── Look up RSA key from cached JWKS (refresh on kid miss)
  │     ├── Verify RS256 signature
  │     └── Check claims:
  │           iss = "https://api.botframework.com"
  │           aud = appId
  │           exp > now (with 5-minute clock skew)
  │
  └── Key management:
        ├── OpenID metadata URL: https://login.botframework.com/v1/.well-known/openidconfiguration
        ├── JWKS URL: from metadata.jwks_uri
        ├── Cache duration: 24 hours
        └── On kid mismatch: force refresh JWKS (rate-limited to prevent abuse)
```

### Commands Integration

`TeamsCommands` implements `Commands`:

```java
public class TeamsCommands implements Commands {
    void registerAll(List<CommandDefinition> commands);
    // Stores command definitions for prefix matching.
    // Teams bot commands are registered in the Azure Bot app manifest,
    // not via API — registerAll() caches locally for dispatch routing.
}
```

Command dispatch flow (inside `TeamsInboundConnector`):

1. Receive `message` activity
2. Strip bot mention (`<at>BotName</at>`) from text
3. Check if remaining text starts with a registered command name
4. If match: create `CommandInvocation(commandName, arguments, userId, channelId, "microsoft-teams", metadata)`
5. Dispatch to `CommandService.dispatch()`
6. Handle response:
   - `Immediate` → build Activity reply JSON, return `WebhookResult.Responded`
   - `Deferred` → return `WebhookResult.Ignored`, spawn virtual thread to send reply via `BotFrameworkClient.replyToActivity()`

### CDI Wiring (TeamsBeans)

```java
@Produces @ApplicationScoped
WebhookInboundConnector teamsInboundConnector(
    @ConfigProperty(name = "casehub.connectors.teams-bot.app-id", defaultValue = "") String appId,
    @ConfigProperty(name = "casehub.connectors.teams-bot.app-secret", defaultValue = "") String appSecret,
    @ConfigProperty(name = "casehub.connectors.teams-bot.dev-mode", defaultValue = "false") boolean devMode,
    @ConfigProperty(name = "casehub.connectors.teams-bot.service-url-allowlist",
                    defaultValue = "*.botframework.com,smba.trafficmanager.net") String allowlist,
    CommandService commandService,
    ServiceConnectionProvider connectionProvider) {
    // Construct BotFrameworkAuth, BotFrameworkClient, TeamsCommands
    // Return new TeamsInboundConnector(...)
}

@Produces @ApplicationScoped
InboundTranslator teamsInboundTranslator() {
    return new TeamsInboundTranslator();
}
```

The existing `ChatPlatform` producer in `TeamsBeans` is updated to wire `TeamsCommands` into the builder:
```java
.commands(teamsCommands)  // was: degraded NoOpCommands
```

## Security

### Inbound Authentication

| Concern | Mitigation |
|---------|-----------|
| JWT forgery | RS256 signature validation against Azure AD JWKS keys |
| Key rotation | 24h cache with on-demand refresh on kid mismatch (rate-limited) |
| Token replay | `exp` claim validation with 5-minute clock skew tolerance |
| Issuer spoofing | Hardcoded issuer check: `https://api.botframework.com` |
| Audience confusion | `aud` must match configured `app-id` |

### Outbound SSRF Defense

| Concern | Mitigation |
|---------|-----------|
| Attacker-controlled serviceUrl | Hostname allowlist validated before every outbound call |
| Government/sovereign clouds | Allowlist is configurable via MP Config |
| Allowlist bypass | URL parsed and hostname extracted before comparison — no path/port tricks |

### Dev-Mode

| Concern | Mitigation |
|---------|-----------|
| Accidental production use | Default `false`, SECURITY WARNING logged per request, deployment config validation |

## Configuration

```properties
# Required — Azure Bot registration
casehub.connectors.teams-bot.app-id=<Azure Bot App ID — also used as JWT audience>
casehub.connectors.teams-bot.app-secret=<client secret for outbound auth>

# SSRF defense (defaults shown)
casehub.connectors.teams-bot.service-url-allowlist=*.botframework.com,smba.trafficmanager.net

# Local development only — NEVER enable in production
casehub.connectors.teams-bot.dev-mode=false
```

**Migration from Outgoing Webhooks:**
1. Create an Azure Bot resource in Azure Portal
2. Register the bot in Teams App Studio / Developer Portal
3. Set `casehub.connectors.teams-bot.app-id` and `casehub.connectors.teams-bot.app-secret`
4. Remove old `casehub.connectors.teams-inbound.shared-secret` config
5. Update webhook URL in Teams to point to `/connectors/teams-inbound/webhook`

## Scope Boundary

**In scope:**
- Bot Framework webhook connector (JWT validation, Activity parsing, routing)
- `TeamsInboundTranslator` (Activity → ReceivedMessage)
- `BotFrameworkClient` (outbound reply/send)
- `TeamsCommands` (command dispatch from message activities)
- Inbound `messageReaction` event handling
- `WebhookResult.Responded` variant
- Remove old `TeamsInboundConnector` from webhook module
- Configuration migration

**Out of scope (separate issues):**
- Outbound reactions (Reactions SPI `add`/`remove`/`list`) — requires Graph API or undocumented Teams-specific endpoint, not Bot Framework REST API
- Attachment download from Bot Framework activities
- Proactive messaging (bot-initiated conversations)
- Adaptive Card action handling beyond basic invoke routing
- Messaging extensions

## Testing Strategy

**Unit tests:**
- `BotFrameworkAuthTest` — valid JWT, expired JWT, wrong issuer, wrong audience, kid mismatch → key refresh, dev-mode bypass, malformed token
- `TeamsInboundConnectorTest` — all 4 activity type routes, signature pass/fail, command prefix detection and dispatch, bot message filtering, bot mention stripping
- `BotFrameworkClientTest` — outbound call construction, SSRF defense (allowed/blocked hosts), auth header
- `TeamsInboundTranslatorTest` — Activity → ReceivedMessage field mapping, compound channel ID, threading (replyToId), reaction metadata
- `TeamsCommandsTest` — registerAll caching, prefix matching
- `WebhookRouterTest` — `Responded` variant handling (deliver messages + return body)

**Integration test (in `integration-tests` module):**
- End-to-end: signed Bot Framework Activity POST → `InboundMessage` delivery → `ReceivedMessage` translation → verify all fields

## References

- `webhook/src/main/java/.../webhook/TeamsInboundConnector.java` — existing implementation being replaced
- `chat-teams/src/main/java/.../chat/teams/TeamsChatPlatform.java` — platform builder, dual token model
- `chat-teams/src/main/java/.../chat/teams/TeamsClient.java` — Graph API client pattern
- `chat-teams/src/main/java/.../chat/teams/TeamsMapper.java` — compound ID format, message mapping
- `chat-discord/src/main/java/.../chat/discord/DiscordInboundConnector.java` — InboundConnector pattern
- `chat-discord/src/main/java/.../chat/discord/DiscordInteractionEndpoint.java` — command endpoint pattern (sync/async)
- `chat-slack/src/main/java/.../slack/SlackCommandEndpoint.java` — command endpoint pattern
- `chat-discord/src/main/java/.../chat/discord/DiscordInboundTranslator.java` — translator pattern
- `connectors-api/src/main/java/.../WebhookResult.java` — sealed result type
- `connectors-api/src/main/java/.../WebhookInboundConnector.java` — base class
- `webhook/src/main/java/.../webhook/WebhookRouter.java` — dispatch infrastructure
- `webhook/src/main/java/.../webhook/SigHelper.java` — HMAC utilities
- `microsoft-graph-client/src/main/java/.../microsoft/MicrosoftGraphClient.java` — Graph client pattern
- `docs/guides/contributor-guide.md` — inbound pipeline architecture
- Bot Framework OpenID metadata: `https://login.botframework.com/v1/.well-known/openidconfiguration`
- Bot Framework REST API v3: reply/send activity endpoints
- nimbus-jose-jwt library (framework-agnostic JWT, used by Spring Security internally)
