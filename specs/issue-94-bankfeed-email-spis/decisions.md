## D1: Demo + Ref pattern per SPI — following the Demo SPI Convention

**Choice:** Each new SPI gets two non-live modules: a `<connector>-ref/` module (plain class, no CDI, for SPI contract testing) and a `<connector>-demo/` module (CDI-managed, `@Alternative @Priority(300) @IfBuildProfile("demo")`, for scripted scenarios and development). Demo impls use `Demo<X>Platform` naming and return `"demo"` from `id()`.
**Alternatives:**
- Single "sim" impl — one class serving both testing and demo; simpler but creates a test gap (demo-profile-gated class unavailable in `@QuarkusTest` under test profile) and conflicts with the Demo SPI Convention
- Demo only, no ref — skip the ref module; forces tests to run under demo profile, conflating test and demo concerns
- Simulation as primary impl — not profile-gated, default when no provider configured; risks confusion about what's "real"
**Rationale:** The Demo SPI Convention (`~/claude/casehub/parent/docs/platform/demo-spi-convention.md`) is the authoritative platform document governing exactly this. Its §6 Checklist explicitly names BankFeedPlatform and EmailPlatform as the target SPIs. The convention specifies module naming (`<connector>-demo/`), class naming (`Demo<X>Platform`), CDI annotations (`@Alternative @Priority(300) @IfBuildProfile("demo")`), bootstrap endpoints (`POST /scenario/bootstrap`), and injection endpoints (`/scenario/inject`). The ref pattern (`RefCalendarPlatform`, `RefChatPlatform`) is proven for SPI contract testing — plain classes with no CDI, manually instantiated in tests, available in all profiles.
**Trade-offs:** Two modules per SPI instead of one. The ref module is small (thin delegation to a backend interface) and the demo module follows a templated convention pattern, so maintenance cost is bounded.
**Sources:** `~/claude/casehub/parent/docs/platform/demo-spi-convention.md` §2, §6; `calendar-ref/RefCalendarPlatform.java` (ref pattern); `chat-ref/RefChatPlatform.java` (ref pattern)
**Exploration:** quick
**Status:** revised (R1-02: aligned with Demo SPI Convention; separated ref and demo roles)

## D2: Categorisation as an optional SPI capability

**Choice:** Categorisation is an optional capability on BankFeedPlatform, available natively when the provider supports it (e.g. Plaid) and absent otherwise. Follows the ChatPlatform capability pattern with graceful degradation.
**Alternatives:**
- Consumer concern only — categorisation lives entirely in consumer domains (life/AML); SPI returns raw transactions. Prevents exposing provider-native categorisation. Contradicts issue #94 acceptance criteria.
- Required SPI method — `categorise()` on the flat interface; forces all providers to implement it, even those without native support. Creates empty/stub implementations.
**Rationale:** Issue #94 acceptance criteria explicitly specify "categorisation interfaces" as part of the BankFeedPlatform SPI. Some providers offer native categorisation (Plaid via `categories` endpoint, TrueLayer via data enrichment). The ChatPlatform capability pattern handles exactly this: `supports(Categorisation.class)` lets callers check availability, and a graceful degradation default (`NoCategorisation`) applies when the provider doesn't support it. Consumer domains can still layer domain-specific categorisation on top of provider-native results.
**Trade-offs:** SPI surface grows. Consumers must check capability presence before calling categorisation methods. But this is the established pattern — ChatPlatform callers do the same with `supports(Reactions.class)`.
**Sources:** Issue #94 acceptance criteria; `chat-spi/ChatPlatform.java` (capability pattern with `supports()`); Plaid API docs (categories endpoint); TrueLayer API docs (data enrichment)
**Exploration:** quick
**Status:** revised (R1-03: reinstated per issue acceptance criteria; modelled as optional capability per ChatPlatform pattern)

## D3: EmailPlatform complements existing email modules

**Choice:** EmailPlatform is a query/read SPI (list inbox, get message, search) that sits alongside EmailConnector (outbound) and EmailInboundConnector (push inbound).
**Alternatives:**
- Supersede existing — unified platform SPI covering send/receive/query; cleaner single object but collapses three architectural layers into one
**Rationale:** Transport connectors and platform SPIs are architecturally distinct layers. `EmailConnector` is an outbound transport abstraction in L1. `EmailInboundConnector` is a pull-based inbound transport in L3. `EmailPlatform` is a domain abstraction at the platform SPI level (alongside `CalendarPlatform`, `ChatPlatform`). Unifying them would mean a single SPI spanning three architectural layers — a layer violation. This is exactly how `ChatPlatform` (L7) coexists with chat connectors (L1/L2) and the Discord inbound connector (L8). The pattern is established.
**Trade-offs:** Three separate entry points for email (send, receive, query) rather than one unified platform. Consumers using both EmailPlatform queries and EmailInboundConnector push events may observe the same message from both paths — the consistency model for this overlap needs explicit design (see D10).
**Sources:** `email/EmailConnector.java` (L1 outbound), `email-inbound/EmailInboundConnector.java` (L3 pull inbound), `chat-spi/ChatPlatform.java` (L7 platform SPI coexisting with L1/L2/L8 connectors)
**Exploration:** quick
**Status:** revised (R1-04: rationale replaced — cites architectural layer distinction instead of backward compatibility; cross-references D10 for consistency model)

## D4: Capability-based SPI for BankFeedPlatform, flat interface for EmailPlatform

**Choice:** BankFeedPlatform uses a capability-based SPI following the ChatPlatform pattern (Builder, `supports()`, graceful degradation defaults). Core capabilities: Accounts, Transactions, Balance. Optional capabilities: Categorisation, ConsentManagement, InstitutionSearch. EmailPlatform uses a flat interface like CalendarPlatform — its surface is small and provider variance is less extreme.
**Alternatives:**
- Flat interface for both — all operations as direct methods on BankFeedPlatform. Works only if all providers support all operations, which they don't: PSD2 providers have consent management and standing orders, Plaid has institution search and categorisation, screen scrapers have neither. Either forces the SPI to the lowest common denominator (too thin) or forces empty implementations (the exact problem ChatPlatform's capability pattern solved).
- Capability-based for both — over-engineers EmailPlatform where provider variance is minimal (IMAP, Microsoft Graph, Google API all support inbox listing and message retrieval).
**Rationale:** Bank feed providers vary significantly: PSD2/Open Banking (TrueLayer/Yapily) offers consent management, real-time balance, and standing orders; aggregators (Plaid) offer categorisation and institution search; screen scrapers offer bare minimum. A flat interface either stays at the lowest common denominator or includes capabilities not all providers support. The ChatPlatform pattern is proven — `ChatPlatform.supports(Reactions.class)` lets callers check availability. Starting capability-based is the safer choice: refactoring from capability-based to flat (all capabilities become required) is trivial, while refactoring from flat to capability-based breaks every caller.
**Trade-offs:** More initial complexity for BankFeedPlatform — Builder, degradation defaults, capability interfaces. But the pattern is templated from ChatPlatform and the complexity serves the architecture.
**Sources:** `chat-spi/ChatPlatform.java` (9 capabilities, Builder, `supports()`, degradation defaults); `calendar-spi/CalendarPlatform.java` (flat — appropriate for homogeneous providers); Plaid/TrueLayer/Yapily API documentation (provider variance evidence)
**Exploration:** quick
**Status:** revised (R1-05: BankFeedPlatform changed to capability-based; EmailPlatform stays flat; rationale cites provider variance evidence)

## D5: Push events via WebhookInboundConnector, demo injection via convention endpoints

**Choice:** Production push events use the existing WebhookInboundConnector SPI. Demo injection uses the Demo SPI Convention's `/scenario/inject` REST endpoint pattern, firing the same CDI events as the real connector adapter. Platform SPI stays query-only.
**Alternatives:**
- Method on concrete sim/demo class — requires consumers to know and cast to the concrete type; couples injection to the impl class rather than CDI event bus
- Platform SPI event source — puts event subscription on the platform SPI; creates two overlapping mechanisms with WebhookInboundConnector
**Rationale:** The Demo SPI Convention §4 specifies injection endpoints that fire CDI events identical to those from real external systems. The key constraint: "Application code observing `@ObservesAsync InboundMessage` cannot distinguish injected events from real ones." This keeps injection decoupled from the impl class and transparent to application code. The architectural layering (query on platform SPI, push via InboundConnector) is confirmed as sound.
**Trade-offs:** Consumers need to know about two mechanisms (platform SPI for query, InboundConnector events for push). But this is the established pattern — ChatPlatform (L7) coexists with Discord/Slack inbound connectors (L2/L8).
**Sources:** `~/claude/casehub/parent/docs/platform/demo-spi-convention.md` §4; `email-inbound/EmailInboundConnector.java` (existing push pattern); `chat-discord/DiscordInboundConnector.java` (Gateway → InboundMessage)
**Exploration:** quick
**Status:** revised (R1-06: demo injection mechanism changed from method-on-class to convention REST endpoint pattern)

## D6: Bootstrap endpoint for scenario data loading

**Choice:** Demo impls receive scenario data via `POST /scenario/bootstrap` at runtime, pushed by the scenario executor. The demo impl exposes a `loadXxx(JsonNode)` method called by the bootstrap resource.
**Alternatives:**
- JSON files on classpath — bakes scenario data into the build; requires different builds for different scenarios; less flexible
- Programmatic seed — hardcoded in Java; even less flexible
- Config-driven file path — runtime file location; adds config complexity without the flexibility of the REST endpoint
**Rationale:** The Demo SPI Convention §3 specifies runtime bootstrap via `POST /scenario/bootstrap` pushed by the scenario executor. This allows different datasets per scenario run without rebuilding. For `@QuarkusTest` fixtures, the test setup calls the bootstrap endpoint with fixture data before assertions — same mechanism, same code path.
**Trade-offs:** Requires the scenario executor to know the bootstrap endpoint and push data at startup. But this is the established convention pattern.
**Sources:** `~/claude/casehub/parent/docs/platform/demo-spi-convention.md` §3; `calendar-ref/InMemoryCalendarBackend.java` (starts empty, events created programmatically — not classpath JSON)
**Exploration:** quick
**Status:** revised (R1-07: changed from classpath JSON to convention bootstrap endpoint; withdrawn claim about "moving away from" InMemoryCalendarBackend)

## D7: Consent/authorization model as optional capability

**Choice:** Consent management is an optional capability on BankFeedPlatform. Providers that require explicit consent (PSD2 — TrueLayer, Yapily) support the `ConsentManagement` capability. The SPI's error model distinguishes "consent required/expired" from "API error" via a specific exception type (`ConsentRequiredException`).
**Alternatives:**
- Ignore consent at SPI level — treat it as an implementation detail; callers can't distinguish "no consent" from "API error" and can't initiate consent flows
- Required on all providers — forces providers without consent models (Plaid Link, screen scrapers) to implement no-op consent methods
- Cross-cutting concern outside SPI — separate consent service; decouples consent from data access but loses the ability to express consent state through the SPI error model
**Rationale:** PSD2 mandates explicit user consent with redirect-based Strong Customer Authentication (SCA) before any account data can be accessed. Consent has a lifecycle: initiated → active → expired → renewed/revoked. A `listTransactions()` call that fails because consent expired must communicate this differently from a transport error — the caller needs to redirect the user to re-consent, not retry. The capability-based SPI (D4) naturally accommodates this as an optional capability.
**Trade-offs:** Full consent lifecycle design is complex (redirect URL generation, callback handling, token storage). This decision establishes the SPI-level modelling; the detailed consent flow design is deferred to the spec's main body.
**Sources:** PSD2 regulation; TrueLayer/Yapily API documentation (SCA redirect flows); `chat-spi/ChatPlatform.java` (optional capability pattern)
**Exploration:** surfaced by review (R1-09)
**Status:** captured

## D8: Cursor-based pagination at SPI level for BankFeedPlatform

**Choice:** BankFeedPlatform's `listTransactions()` accepts pagination parameters and returns paginated results using a cursor-based model. EmailPlatform's inbox queries also expose pagination. The pattern follows the platform's existing cursor pagination approach.
**Alternatives:**
- Internal pagination only — impl paginates internally, returns flat list (CalendarPlatform/GoogleCalendarPlatform pattern); risks OOM on bank transaction histories spanning years and millions of records
- Offset-based pagination — simpler but breaks when data changes between pages; inconsistent results on large datasets
- Stream/reactive — returns a reactive stream; more complex, not aligned with the platform's synchronous SPI model
**Rationale:** Bank transaction histories can span years with millions of records. Returning all transactions as a flat `List<Transaction>` will OOM on real accounts. CalendarPlatform's flat-list approach works because calendar events within a date range are bounded (hundreds at most). The platform already has cursor pagination patterns: `SlackBotClient.listChannels()` with `PageResult` fail-soft (PP-20260610-83747b). The SPI should expose pagination to callers rather than hiding it internally.
**Trade-offs:** Callers must manage pagination (cursor passing, page iteration). But this is inherent to the domain — hiding it would create worse problems (OOM, timeouts on large datasets).
**Sources:** `slack-bot/SlackBotClient.java` (cursor pagination with `PageResult`); `calendar-google/GoogleCalendarPlatform.java` (internal pagination — appropriate for bounded result sets, not for bank transactions); PP-20260610-83747b (paginating client fail-soft protocol)
**Exploration:** surfaced by review (R1-10)
**Status:** captured

## D9: Multi-provider routing via BankFeedPlatformService

**Choice:** Follow the established platform service pattern: one `BankFeedPlatform` implementation per provider, discovered via CDI and routed by `id()` through `BankFeedPlatformService`. Cross-provider aggregation (viewing accounts across multiple banks) is a consumer concern.
**Alternatives:**
- Single BankFeedPlatform aggregating across providers — one impl that internally routes to multiple backends; creates a god-class and makes the SPI responsible for cross-provider concerns (deduplication, merged pagination, error aggregation)
- Per-bank instances — one BankFeedPlatform per connected bank rather than per provider; conflates the bank-provider relationship (one provider like TrueLayer serves many banks)
**Rationale:** `ChatPlatformService.platform(id)` and `CalendarPlatformService.platform(id)` are the established patterns. Each provider registers as a separate CDI bean with a unique `id()`. The routing service discovers all beans via `@All List<BankFeedPlatform>` and routes by ID. A user with accounts at multiple banks via different providers calls `service.platform("truelayer")` for PSD2 banks and `service.platform("plaid")` for aggregated banks. This keeps each provider implementation focused and testable.
**Trade-offs:** Consumers that want a unified view across providers must aggregate themselves. But this is the right layer separation — the SPI provides per-provider access, consumers compose the cross-provider view.
**Sources:** `chat-spi/ChatPlatformService.java` (routes by `id()`, `@All List<ChatPlatform>`); `calendar-spi/CalendarPlatformService.java` (same pattern)
**Exploration:** surfaced by review (R1-11)
**Status:** captured

## D10: EmailPlatform/EmailInboundConnector consistency model

**Choice:** EmailPlatform queries and EmailInboundConnector push events may surface the same message. This is by design — the two paths serve different consumer patterns (poll vs push) and overlap is expected. No deduplication at the SPI level; consumers that observe both must be idempotent.
**Alternatives:**
- SPI-level deduplication — EmailPlatform marks messages as "already delivered via push"; adds coupling between the two SPIs and state tracking
- Exclusive paths — EmailPlatform excludes messages already pushed via EmailInboundConnector; breaks the query model (inbox listing would have gaps)
**Rationale:** This follows the existing platform pattern. EmailInboundConnector's documented delivery guarantee is at-least-once (SEEN flag in `finally`, JVM crash can redeliver). Observers must already be idempotent. Adding deduplication at the SPI level would couple EmailPlatform to EmailInboundConnector's state, violating the layer separation that D3 establishes.
**Trade-offs:** Consumers using both paths must handle duplicates. But this is already a documented requirement — `@ObservesAsync InboundMessage` observers must be idempotent per the §8 crosscutting concerns.
**Sources:** `email-inbound/EmailInboundConnector.java` (at-least-once, idempotent observers required); ARC42STORIES.MD §8 (idempotent observer requirement)
**Exploration:** surfaced by review (R1-04, implicit decision in D3)
**Status:** captured
