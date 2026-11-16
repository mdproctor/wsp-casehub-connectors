# HANDOFF — casehub-connectors

## Last Session

Proactive messaging (#234) — the bot can now initiate 1:1 conversations with users who haven't messaged it, using the Bot Framework REST API `POST /v3/conversations`. Also captures `ConversationReference` data from inbound `conversationUpdate` activities.

**Key architectural change:** `TeamsChatPlatform.sendMessage()` now has three-way routing based on channel prefix:
- `personal:<conversationId>` → Bot Framework `sendToConversation` (existing conversations)
- `user:<aadObjectId>` → **new**: resolve via `ConversationReferenceStore` or cold-start `createConversation`, then send
- `teamId:channelId` → Graph API (team channels)

The `user:` path returns `personal:<conversationId>` in `SendResult`, so callers can use the direct conversation ID for subsequent messages.

**ConversationReferenceStore SPI:** CDI interface with `InMemoryConversationReferenceStore` (`@DefaultBean`, `ConcurrentHashMap`). Follows the `HistoryStore` pattern from `email-google-inbound`. Persistence is optional — the cold-start path covers recovery from lost refs.

**conversationUpdate handling:** `TeamsInboundConnector` now handles `conversationUpdate` activities instead of ignoring them. Stores non-bot member refs and fires `InboundMessage` with `teams-activity-type: conversationUpdate` metadata. Store failures are try-caught (best-effort optimisation). `membersRemoved` events do not delete stored refs.

**New config properties:**
- `casehub.connectors.teams-bot.default-tenant-id` — required for cold-start proactive messaging
- `casehub.connectors.teams-bot.app-id` — already existed for inbound, now also used for `createConversation` body

**Branch:** `issue-234-teams-proactive-messaging`

**Commits:**
- `e6f872d3` — ConversationReferenceStore SPI with InMemory default
- `5eb5eb96` — BotFrameworkClient.createConversation for proactive messaging
- `8d5391b2` — user: channel prefix for proactive messaging in TeamsChatPlatform
- `d0f8bc64` — handle conversationUpdate — store refs and fire events
- `80d6279a` — review fixes — try-catch store, scope variables, membersRemoved test

**Issues closed this session:**

| # | Title | What changed |
|---|-------|-------------|
| #234 | Proactive messaging bot-initiated conversations | ConversationReferenceStore SPI, BotFrameworkClient.createConversation, user: prefix routing, conversationUpdate handling |

**Issues filed this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #252 | Integration test for proactive messaging flow | S | Low |
| #253 | replyToMessage should handle user: channel prefix | XS | Low |

**Queue state after this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #235 | Adaptive Card action handling invoke activities | M | High |
| #236 | Messaging extensions support | L | High |

**To resume:** `work continue` for #235 (next in queue).

**Configuration for proactive messaging:**
- `casehub.connectors.teams-bot.default-service-url` — Bot Framework service URL (also needed for personal: sends)
- `casehub.connectors.teams-bot.default-tenant-id` — Azure AD tenant ID for cold-start
- `casehub.connectors.teams-bot.app-id` — bot application ID (shared with inbound JWT validation)
- `casehub.connectors.teams-bot.service-url-allowlist` — SSRF defence (default: `*.botframework.com,smba.trafficmanager.net`)
