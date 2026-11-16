# Decisions — #234 Proactive Messaging

## D1: Proactive messaging scope

**Choice:** Both cold-start (createConversation via POST /v3/conversations) and stored refs
**Alternatives:**
- Cold-start only — always call createConversation, no storage needed but extra API call every time
- Stored refs only — simpler but limits proactive reach to users who've interacted before
**Rationale:** Full proactive capability requires reaching any user in the tenant. Stored refs optimise the common case (re-contacting known users) while cold-start covers first contact.
**Trade-offs:** More complexity — two code paths (store lookup → fallback to create). Storage SPI adds surface area.
**Sources:** Bot Framework REST API POST /v3/conversations, TeamsInboundConnector.java (conversationUpdate already received but ignored)
**Exploration:** quick
**Status:** captured

## D2: ConversationReference storage mechanism

**Choice:** CDI SPI with InMemory default (ConversationReferenceStore), following HistoryStore pattern
**Alternatives:**
- Simple ConcurrentHashMap cache, no SPI — lost on restart but cold-start covers it, less infrastructure
- No storage at all — always cold-start, simplest but adds latency
**Rationale:** Consistent with established pattern (HistoryStore in email-google-inbound). CDI SPI lets callers bring persistent implementations. Pure infrastructure — no business logic in the store.
**Trade-offs:** More code than a simple cache. In-memory default means refs are lost on restart (acceptable — cold-start fallback).
**Depends on:** D1 (both paths requires storage for the optimised path)
**Sources:** email-google-inbound/HistoryStore SPI, InMemoryHistoryStore
**Exploration:** quick
**Status:** captured

## D3: API surface — how proactive messaging is exposed

**Choice:** Extend sendMessage with `user:<aadObjectId>` channel prefix convention
**Alternatives:**
- New initiateConversation method on ChatPlatform SPI — more explicit but changes the SPI contract
- BotFrameworkClient only — simpler but bypasses the platform abstraction
**Rationale:** Follows the existing `personal:` prefix convention. Transparent to callers — they just send to a user identifier. TeamsChatPlatform resolves internally: check store for existing conversation, else cold-start, then send.
**Trade-offs:** Channel ID conventions accumulate (`teamId:channelId`, `personal:conversationId`, `user:aadObjectId`). Callers need to know which prefix to use.
**Depends on:** D1 (user: path needs both cold-start and store lookup), D2 (store lookup for optimised path)
**Sources:** TeamsChatPlatform.java sendMessage routing (line 85-98), isPersonalChannel (line 203-205)
**Exploration:** quick
**Status:** captured

## D4: conversationUpdate handling

**Choice:** Store ConversationReference AND fire InboundMessage CDI event
**Alternatives:**
- Store only, no event — silently capture ref, observers don't learn about bot installation
- Fire event only, no store — let observers handle storage, separation of concerns but duplicates work
**Rationale:** Both are needed. Storage populates the ref cache for future proactive sends. The event lets observers react to bot installation (e.g. send a welcome message). Event metadata distinguishes conversationUpdate from regular messages.
**Trade-offs:** Two side effects from one activity type. Observers must filter on event type if they only want messages.
**Depends on:** D2 (store is the storage target)
**Sources:** TeamsInboundConnector.java lines 86-89 (currently ignored), Bot Framework conversationUpdate activity schema
**Exploration:** quick
**Status:** captured

## D5: Tenant ID resolution for cold-start createConversation

**Choice:** Config default tenant ID via `casehub.connectors.teams-bot.default-tenant-id`
**Alternatives:**
- Compound channel ref `user:<tenantId>:<aadObjectId>` — multi-tenant from day one but uglier refs, callers must know tenant
- Lookup from store then fall back to config — hybrid but adds complexity
**Rationale:** Most bots are single-tenant. Clean `user:<aadObjectId>` format. Consistent with existing `default-service-url` config pattern. Multi-tenant can be added later if needed.
**Trade-offs:** Single-tenant assumption. Multi-tenant bots would need enhancement.
**Depends on:** D3 (user: prefix format)
**Sources:** TeamsBeans.java config properties, Bot Framework createConversation API (tenantId required in body)
**Exploration:** quick
**Status:** captured
