# Teams Inbound via Bot Framework — Design Spec

**Issue:** casehubio/connectors#220
**Branch:** `issue-220-teams-inbound-bot-framework`
**Scale:** L → XL | **Complexity:** High

## Summary

Replace the existing `TeamsInboundConnector` (Outgoing Webhooks model with HMAC-SHA256 shared secret) with a Bot Framework integration that receives all Teams activity types through a single webhook endpoint. This delivers:

1. **Inbound messages** — Bot Framework `message` activities parsed into `InboundMessage` and translated to `ReceivedMessage` via a new `TeamsInboundTranslator`
2. **Inbound reaction events** — `messageReaction` activities delivered as `InboundMessage` with reaction metadata
3. **Slash commands** — commands embedded in message activities, dispatched to `CommandService`, with sync and async response paths
4. **Bot Framework client** — outbound REST API for reply and command responses

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
| `TeamsCommands` | `Commands` SPI — logs available commands on registration. Teams bot commands are registered in the Azure Bot app manifest, not via API — `registerAll()` has no outbound effect (same pattern as `SlackCommands`). Command prefix matching uses `CommandService.definitions()` |

**`webhook` (modified):**

| Change | Detail |
|--------|--------|
| Remove `TeamsInboundConnector` | Old Outgoing Webhooks implementation |
| Remove `TeamsInboundConnectorTest` | Associated tests |
| Update `WebhookBeans` | Remove Teams producer method |

Removal of the old `TeamsInboundConnector` from `webhook` and addition of the new one in `chat-teams` must be atomic (same commit) — `WebhookRouter` throws `IllegalStateException` on duplicate connector IDs at startup.

**`docs/guides/contributor-guide.md` (modified):**

| Change | Detail |
|--------|--------|
| Webhook signature verification table | Replace HMAC-SHA256 entry for Teams with JWT/RS256 via `BotFrameworkAuth` |
| Commands architecture section | Add note: platforms with unified webhook transport (Teams) embed command dispatch in the connector rather than using a separate JAX-RS endpoint |
| Webhook connectors list | Move Teams from `webhook` module to `chat-teams` |

**`chat-teams/pom.xml` (modified):**

| Dependency | Scope |
|-----------|-------|
| `nimbus-jose-jwt` | compile |
| `casehub-connectors-api` (existing) | — already present via chat-spi |

### Activity Flow

```
POST /connectors/teams-inbound/webhook
  │
  ├── Parse Activity JSON (jakarta.json)
  │     (Parsing untrusted JSON before authentication is standard practice —
  │      the Bot Framework SDK does the same. Jakarta JSON does not execute
  │      arbitrary code, and Quarkus enforces HTTP body size limits.)
  │
  ├── BotFrameworkAuth.validate(Authorization header, activity.serviceUrl)
  │     ├── Fetch OpenID metadata (cached): https://login.botframework.com/v1/.well-known/openidconfiguration
  │     ├── Fetch JWKS (cached 24h, refresh on kid mismatch — rate-limited): https://login.botframework.com/v1/.well-known/keys
  │     ├── Validate RS256 signature against matching kid
  │     ├── Check claims: iss=configured issuer, aud=APP_ID, serviceurl=activity.serviceUrl, exp
  │     └── Dev-mode: skip validation, log SECURITY WARNING per request
  │
  ├── Self-filter: if activity.from.id matches configured app-id → return Ignored
  │     (defense-in-depth — Bot Framework does not normally echo outbound messages,
  │      but filtering prevents infinite loops if behavior changes)
  │
  └── Route by activity.type:
        │
        ├── "message"
        │     ├── Strip bot mention from text (entity-based):
        │     │     find mention entity in activity.entities[] where
        │     │     type="mention" and mentioned.id matches appId,
        │     │     remove that entity's text value from activity.text,
        │     │     then trim. Preserves mentions of other users.
        │     ├── Check for command prefix (first word matched case-insensitively
        │     │   against CommandService.definitions() names)
        │     │     ├── YES → CommandService.dispatch(commandName, invocation)
        │     │     │         ├── Immediate → reply via BotFrameworkClient.replyToActivity()
        │     │     │         │               synchronously, return Ignored
        │     │     │         └── Deferred  → return Ignored, spawn virtual thread:
        │     │     │                          create DeferredReply wrapping
        │     │     │                          BotFrameworkClient.replyToActivity(serviceUrl,
        │     │     │                            conversationId, activityId, text)
        │     │     │                          call def.callback().accept(deferredReply)
        │     │     │                          (ephemeral parameter ignored — Bot Framework
        │     │     │                           has no concept of ephemeral messages)
        │     │     │         (Bot Framework discards HTTP response body for message
        │     │     │          activities — all command replies go through outbound API)
        │     │     └── NO → Return Delivered(InboundMessage)
        │     │                → ChatInboundAdapter → TeamsInboundTranslator → ReceivedMessage
        │
        ├── "messageReaction"
        │     └── Flatten reactionsAdded[] and reactionsRemoved[] arrays:
        │           emit one InboundMessage per individual reaction entry.
        │           Return Delivered(List<InboundMessage>)
        │           Per-reaction InboundMessage:
        │             content: "" (empty string — reactions have no text)
        │             metadata keys:
        │               teams-reaction-type: like|heart|laugh|surprised|sad|angry
        │               teams-reaction-action: "added" or "removed"
        │               teams-reaction-message-id: activity.replyToId
        │           (CloudEvent consumers use teams-activity-type to identify reactions;
        │            the chat pipeline — TeamsInboundTranslator — rejects non-message
        │            activities, so reactions flow only through CloudEvents)
        │
        ├── "conversationUpdate"
        │     └── Return Ignored (log member join/leave at INFO)
        │
        └── all other types (invoke, installationUpdate, etc.)
              └── Return Ignored (log at DEBUG)
                    Invoke activities require response bodies for proper UI behavior
                    in Teams — full invoke handling is out of scope (casehubio/connectors#235)
```

**Architectural note — command dispatch divergence:** The contributor guide prescribes a separate JAX-RS endpoint for command invocations (e.g. `DiscordInteractionEndpoint`, `SlackCommandEndpoint`). Teams diverges because Bot Framework delivers commands as `message` activities to the same webhook — there is no separate command interaction endpoint. Command dispatch is therefore embedded in `TeamsInboundConnector.handle()`. This is documented in the contributor guide update.

### Activity JSON → InboundMessage Mapping

Bot Framework Activity fields mapped to `InboundMessage`:

| InboundMessage field | Activity source |
|---------------------|----------------|
| `connectorId` | `InboundConnectorIds.TEAMS_INBOUND` (`"teams-inbound"`) |
| `connectorType` | `InboundConnectorTypes.TEAMS` (`"teams"`) |
| `externalSenderId` | `activity.from.aadObjectId` (preferred) or `activity.from.id` |
| `externalChannelRef` | Channel context: `channelData.team.id` + `:` + `channelData.channel.id` (compound format matching existing `TeamsMapper.parseTeamAndChannel`). Personal/1:1 context: `"personal:" + activity.conversation.id` (no team/channel in channelData). The `personal:` prefix distinguishes personal refs from compound `teamId:channelId` — `TeamsMapper.parseTeamAndChannel` will reject personal refs with a clear error since outbound personal messaging requires `BotFrameworkClient`, not Graph API (tracked as casehubio/connectors#237). |
| `content` | For `message` activities: `activity.text` (with bot mention stripped via entity-based removal — see below). If `activity.text` is null (attachment-only messages), use `""` — no mention stripping needed. For `messageReaction` activities: `""` (empty string — reactions have no text field). |
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

**Activity type guard:** `translate()` checks `metadata["teams-activity-type"]`. Only `"message"` activities are translatable. For any other type (e.g. `"messageReaction"`), the translator throws `IllegalArgumentException("Non-message activity type: <type> — not translatable to ReceivedMessage")`. `ChatInboundAdapter` catches and logs this (line 48-50), so non-message `InboundMessage` events flow only through the CloudEvent pipeline (`ConnectorsCloudEventAdapter`), which correctly serializes all metadata including reaction details.

Translates `InboundMessage` → `ReceivedMessage` for the chat pipeline (message activities only):

| ReceivedMessage field | Source |
|----------------------|--------|
| `platformId` | `"microsoft-teams"` (matches `TeamsChatPlatform.PLATFORM_ID`) |
| `channel` | `ChatChannelRef(msg.externalChannelRef())` — compound `teamId:channelId` for channel context, `personal:conversationId` for 1:1 context |
| `message` | `ChatMessageRef(channel, metadata["teams-activity-id"])` |
| `parent` | `ChatMessageRef(channel, metadata["teams-reply-to-id"])` if present, else `null` |
| `sender` | `MemberRef(msg.externalSenderId())` |
| `content` | `ChatContent(msg.content())` |
| `receivedAt` | `msg.receivedAt()` |

**Limitation — HTML formatting:** Teams messages arrive with HTML formatting (`<at>BotName</at>` for mentions, `<b>bold</b>`, `<i>italic</i>`). The single-arg `ChatContent` constructor treats content as plain text — formatting is discarded. This is consistent with `DiscordInboundTranslator`. Future enhancement: convert HTML to markdown via `ChatContent.markdown` field.

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
BotFrameworkAuth(appId, devMode, metadataUrl, issuer)
  │
  ├── validate(authorizationHeader, activityServiceUrl) → boolean
  │     ├── If devMode → log SECURITY WARNING, return true
  │     ├── Extract Bearer token
  │     ├── Parse JWT header → get kid
  │     ├── Look up RSA key from cached JWKS (refresh on kid miss)
  │     ├── Verify RS256 signature
  │     └── Check claims:
  │           iss = configured issuer (default: "https://api.botframework.com")
  │           aud = appId
  │           serviceurl = activityServiceUrl (cryptographic proof the token
  │                        was issued for this specific service URL)
  │           exp > now (with 5-minute clock skew)
  │
  └── Key management:
        ├── OpenID metadata URL: configurable (default: https://login.botframework.com/v1/.well-known/openidconfiguration)
        │     GCC High: https://login.botframework.us/v1/.well-known/openidconfiguration
        ├── JWKS URL: from metadata.jwks_uri
        ├── Cache duration: 24 hours
        └── On kid mismatch: force refresh JWKS
              ├── Rate limit: at most once per 5 minutes
              ├── During cooldown: reject tokens with unknown kid (return Unauthorized)
              └── Log WARNING when refresh is rate-limited (potential attack or key rotation in progress)
```

### Commands Integration

`TeamsCommands` implements `Commands`:

```java
public class TeamsCommands implements Commands {
    @Override
    public void registerAll(List<CommandDefinition> commands) {
        LOG.info("teams: " + commands.size()
                + " commands available (register via Azure Bot app manifest)");
    }
}
```

Command dispatch flow (inside `TeamsInboundConnector`):

1. Receive `message` activity
2. If `activity.text` is null (attachment-only messages), set text to `""` and skip to step 3 (no mention stripping needed). Otherwise, strip bot mention from text using entity-based removal: find the mention entity in `activity.entities[]` where `type = "mention"` and `mentioned.id` matches the configured `appId`, remove that entity's `text` value from `activity.text`, then trim. This preserves mentions of other users (unlike regex `<at>.*?</at>` which strips all mentions).
3. Extract the first whitespace-delimited word from the remaining text
4. Match case-insensitively against registered command names from `CommandService.definitions()`
5. If match:
   - Command name: the matched definition name (preserving original casing)
   - Arguments: `Map.of("text", remainingTextAfterCommandName.trim())` — follows the Slack pattern (single `"text"` argument containing everything after the command name). Empty map if no remaining text.
   - Create `CommandInvocation(commandName, arguments, userId, channelId, "microsoft-teams", metadata)`
   - Metadata includes `teams-activity-id`, `teams-service-url`, `teams-conversation-id`
6. Dispatch to `CommandService.dispatch()`
7. Handle response (Bot Framework discards HTTP response body for message activities — all replies go through outbound API):
   - `Immediate` → call `BotFrameworkClient.replyToActivity(serviceUrl, conversationId, activityId, imm.text())` synchronously, return `WebhookResult.Ignored`. `imm.ephemeral()` is ignored — Bot Framework has no ephemeral messages.
   - `Deferred` → return `WebhookResult.Ignored`, spawn virtual thread:
     ```java
     DeferredReply reply = (text, ephemeral) ->
         botFrameworkClient.replyToActivity(serviceUrl, conversationId, activityId, text);
     // ephemeral parameter ignored — Bot Framework has no ephemeral messages
     def.callback().accept(reply);
     ```
     `serviceUrl`, `conversationId`, and `activityId` are captured from the activity and threaded into the `DeferredReply` closure.

Example: user types `help deploy production` → command = `"help"`, arguments = `{text: "deploy production"}`

### CDI Wiring (TeamsBeans)

```java
@Produces @ApplicationScoped
TeamsCommands teamsCommands() {
    return new TeamsCommands();
}

@Produces @ApplicationScoped
WebhookInboundConnector teamsInboundConnector(
    @ConfigProperty(name = "casehub.connectors.teams-bot.app-id", defaultValue = "") String appId,
    @ConfigProperty(name = "casehub.connectors.teams-bot.dev-mode", defaultValue = "false") boolean devMode,
    @ConfigProperty(name = "casehub.connectors.teams-bot.service-url-allowlist",
                    defaultValue = "*.botframework.com,smba.trafficmanager.net") String allowlist,
    @ConfigProperty(name = "casehub.connectors.teams-bot.openid-metadata-url",
                    defaultValue = "https://login.botframework.com/v1/.well-known/openidconfiguration") String metadataUrl,
    @ConfigProperty(name = "casehub.connectors.teams-bot.issuer",
                    defaultValue = "https://api.botframework.com") String issuer,
    CommandService commandService,
    ServiceConnectionProvider connectionProvider) {
    var auth = new BotFrameworkAuth(appId, devMode, metadataUrl, issuer);
    var client = new BotFrameworkClient(connectionProvider, allowlist);
    return new TeamsInboundConnector(auth, client, commandService, appId);
}

@Produces @ApplicationScoped
InboundTranslator teamsInboundTranslator() {
    return new TeamsInboundTranslator();
}
```

The existing `ChatPlatform` producer in `TeamsBeans` is updated to accept `TeamsCommands` and wire it into the builder:
```java
@Produces @ApplicationScoped
ChatPlatform teamsChatPlatform(MicrosoftGraphClient graph,
                                ServiceConnectionProvider connectionProvider,
                                TeamsCommands teamsCommands,
                                @ConfigProperty(name = "casehub.connectors.teams.default-team-id",
                                                defaultValue = "") String defaultTeamId) {
    return new TeamsChatPlatform(new TeamsClient(graph), connectionProvider, defaultTeamId)
            .withCommands(teamsCommands)
            .build();
}
```

`TeamsChatPlatform` gains a `withCommands(Commands)` fluent setter that threads the `Commands` instance into the `ChatPlatform.builder().commands(...)` call. CDI dependency chain: `TeamsCommands` → `ChatPlatform` → `CommandService` → `TeamsInboundConnector` — no cycles.

## Security

### Inbound Authentication

| Concern | Mitigation |
|---------|-----------|
| JWT forgery | RS256 signature validation against Azure AD JWKS keys |
| Key rotation | 24h cache with on-demand refresh on kid mismatch (rate-limited) |
| Expired token acceptance | `exp` claim validation with 5-minute clock skew tolerance. Replay of valid (non-expired) tokens within their validity window is an accepted risk — HTTPS provides transport-layer replay protection. Bot Framework JWTs do not include a `jti` claim for nonce tracking. |
| Issuer spoofing | Configurable issuer check (default: `https://api.botframework.com`; GCC High: `https://api.botframework.us`) |
| Audience confusion | `aud` must match configured `app-id` |
| Service URL binding | JWT `serviceurl` claim must match `activity.serviceUrl` — cryptographic proof the token was issued for this specific channel service URL. Complements SSRF hostname allowlist with per-request origin verification. |

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

# SSRF defense (defaults shown)
casehub.connectors.teams-bot.service-url-allowlist=*.botframework.com,smba.trafficmanager.net

# Bot Framework auth (defaults for public cloud — override for sovereign/GCC High)
casehub.connectors.teams-bot.openid-metadata-url=https://login.botframework.com/v1/.well-known/openidconfiguration
casehub.connectors.teams-bot.issuer=https://api.botframework.com

# Local development only — NEVER enable in production
casehub.connectors.teams-bot.dev-mode=false
```

**Outbound auth:** `BotFrameworkClient` authenticates via `ServiceConnectionProvider.getAccessToken(APPLICATION_ACTOR, "microsoft-bot", "default")`. The `"microsoft-bot"` service connection must be configured in the platform's authn registry with:
- Token endpoint: `https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token`
- Scope: `https://api.botframework.com/.default`
- Client credentials: the Azure Bot app ID and client secret

This follows the established pattern — `TeamsChatPlatform` already uses `"microsoft-app"` and `"microsoft"` service connections. The connector never touches raw client secrets.

**Migration from Outgoing Webhooks:**
1. Create an Azure Bot resource in Azure Portal
2. Register the bot in Teams App Studio / Developer Portal
3. Configure `"microsoft-bot"` service connection in the platform's authn registry
4. Set `casehub.connectors.teams-bot.app-id`
5. Remove old `casehub.connectors.teams-inbound.shared-secret` config
6. Update webhook URL in Teams to point to `/connectors/teams-inbound/webhook`

## Scope Boundary

**In scope:**
- Bot Framework webhook connector (JWT validation, Activity parsing, routing)
- `TeamsInboundTranslator` (Activity → ReceivedMessage)
- `BotFrameworkClient` (outbound reply/send)
- `TeamsCommands` (command dispatch from message activities)
- Inbound `messageReaction` event handling
- Remove old `TeamsInboundConnector` from webhook module
- Configuration migration
- Contributor guide updates

**Out of scope (tracked as separate issues):**
- Outbound reactions (Reactions SPI `add`/`remove`/`list`) — requires Graph API or undocumented Teams-specific endpoint, not Bot Framework REST API → casehubio/connectors#232
- Attachment download from Bot Framework activities → casehubio/connectors#233
- Proactive messaging (bot-initiated conversations) → casehubio/connectors#234
- Adaptive Card action handling (invoke activities) → casehubio/connectors#235
- Messaging extensions → casehubio/connectors#236
- Personal/1:1 chat outbound messaging (requires BotFrameworkClient, not Graph API) → casehubio/connectors#237

## Testing Strategy

**Unit tests:**
- `BotFrameworkAuthTest` — valid JWT, expired JWT, wrong issuer, wrong audience, wrong serviceurl claim, kid mismatch → key refresh, kid mismatch during rate-limit cooldown → reject, dev-mode bypass, malformed token, configurable issuer/metadata URL
- `TeamsInboundConnectorTest` — message/messageReaction/conversationUpdate/unknown activity routing, messageReaction flattening (multiple reactions per activity → multiple InboundMessages), signature pass/fail, command prefix detection and dispatch (Immediate + Deferred with DeferredReply), self-message filtering (from.id = app-id → Ignored), entity-based bot mention stripping (preserves other user mentions), personal vs channel context channel ref format
- `BotFrameworkClientTest` — outbound call construction, SSRF defense (allowed/blocked hosts), auth header via ServiceConnectionProvider
- `TeamsInboundTranslatorTest` — Activity → ReceivedMessage field mapping, compound channel ID, personal channel ID (`personal:` prefix), threading (replyToId), activity type guard (messageReaction → throws IllegalArgumentException)
- `TeamsCommandsTest` — registerAll logging, case-insensitive first-word matching, argument extraction (Slack "text" pattern)

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
