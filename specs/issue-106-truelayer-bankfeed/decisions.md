## D1: Unified BankPlatform SPI with capability sub-interfaces

**Choice:** Rename `BankFeedPlatform` → `BankPlatform` with capability sub-interfaces: `AccountInformation` (AISP) and `PaymentInitiation` (PISP). Single SPI, single `@SimulationEligible` annotation, capability pattern like ChatPlatform.
**Alternatives:**
- Separate flat SPIs (`BankAccountPlatform` + `BankPaymentPlatform`) — clean separation but forces callers to juggle two services for one domain entity; providers that support both must register as two separate beans
- Single flat interface with all methods — no capability degradation model; forces all providers to implement everything
- Keep `BankFeedPlatform` read-only, add `BankPaymentPlatform` separately — maps to PSD2 AISP/PISP but at the wrong abstraction level; regulatory licensing is a provider concern, not an SPI concern
**Rationale:** PSD2's AISP/PISP distinction is real and domain-significant — these are separately licensed capabilities with different consent scopes and durations. The capability sub-interface pattern honours this: `AccountInformation` maps to AISP, `PaymentInitiation` maps to PISP. A provider with only AISP authorization implements `AccountInformation` only. The unified `BankPlatform` wrapper reflects the domain truth that banking is one domain with multiple regulated capabilities — not separate domains. The simulation framework's recursive wrapper generation (platform#375) supports capability interception.
**Trade-offs:** Requires renaming existing `BankFeedPlatform` and updating `@SimulationEligible(name)`, NoOp, service class, and all test references. Pre-release, this is non-breaking.
**Sources:** `chat-spi/ChatPlatform.java` (capability pattern), `bank-spi/BankFeedPlatform.java` (current flat), platform#375 (recursive wrapper generation), D4 from #94 decisions (flat interface — superseded), PSD2 regulation (AISP/PISP as separately licensed capabilities)
**Exploration:** deep-analysis (evolved through naming discussion → domain modeling → capability architecture)
**Status:** revised (R1-02: rationale reframed to acknowledge PSD2 AISP/PISP as domain-significant; capability names aligned to PSD2 vocabulary)

## D2: Quarkus OIDC client for TrueLayer client credentials

**Choice:** Use `quarkus-oidc-client` (`OidcClient`) for the OAuth2 client credentials grant that authenticates `TrueLayerClient` to TrueLayer's API. Quarkus manages token caching, automatic refresh, and thread-safe access. `TrueLayerClient` injects `OidcClient`, calls `getTokens()` before each request, and uses the returned access token. No custom token management code.
**Alternatives:**
- Custom managed token holder — hand-written token caching, expiry checking, refresh logic, thread-safety (double-checked locking or CompletableFuture dedup); reimplements what Quarkus provides
- Token-per-call (SlackBotClient pattern) — caller passes token on every method; pushes refresh complexity to every caller
- Constructor-injected static credentials (GoogleCalendarPlatform pattern) — works for Google SDK which handles refresh internally; doesn't work with HttpHelper.CLIENT
**Rationale:** TrueLayer's auth endpoint is a standard OAuth2 client_credentials grant. Quarkus provides `quarkus-oidc-client` which handles this natively — token caching, automatic refresh, thread-safe access, and configurable retry. Building a custom managed holder reimplements framework-provided functionality, requires solving the thundering-herd problem for concurrent refresh, and needs explicit error handling for refresh failures. `OidcClient` is already CDI-managed and battle-tested.
**Trade-offs:** Adds `quarkus-oidc-client` dependency to `bank-truelayer`. Configuration is via `application.properties` (`quarkus.oidc-client.*`), which is standard Quarkus convention but less visible than constructor parameters. If TrueLayer's auth endpoint deviates from standard OAuth2, may need custom `OidcClientConfig`.
**Sources:** Quarkus OIDC Client documentation, PP-20260609-0c3e24 (credential-config-ownership protocol), R1-06 (reviewer finding: unconsidered alternative)
**Exploration:** quick (revised after review)
**Status:** revised (R1-06: replaced custom managed token holder with quarkus-oidc-client; addresses thread-safety and refresh failure gaps)

## D3: Consent as provider-internal infrastructure, not on BankPlatform SPI

**Choice:** Consent management is internal to `bank-truelayer` — a `TrueLayerConsentService` CDI bean handles auth link generation, code exchange, token storage, and consent status. `BankPlatform` SPI has no consent methods. Callers that need to check or initiate consent inject `TrueLayerConsentService` directly (or a provider-neutral `ConsentService` interface if a second provider arrives).
**Alternatives:**
- Consent as BankPlatform capability (original D3) — mixes authorization flow mechanics (URL generation, code exchange) with banking domain operations; `generateAuthLink()` is OAuth2 infrastructure, not a banking operation
- Separate cross-cutting `ConsentPlatform` SPI — reusable across regulated domains but speculative; no second regulated SPI exists yet
**Rationale:** Consent is a precondition for banking operations, not a banking operation itself. Putting `generateAuthLink()` on BankPlatform is analogous to putting `login()` on every domain service. The consent flow involves the user's browser, OAuth2 code exchange, and token storage — none of which are banking domain concerns. Keeping consent as provider infrastructure lets `BankPlatform` stay focused on account information and payment initiation. The platform orchestration layer (e.g., casehub-life) injects the consent service alongside the bank platform service, checking consent status before initiating banking operations.
**Trade-offs:** Consent management is not visible at the SPI level — callers must know to inject the consent service separately. No SPI-level `supports(Consent)` check. If a second provider needs consent, we'll need to extract a shared `ConsentService` interface.
**Sources:** R1-03 (reviewer finding: consent is precondition, not domain operation), D7 from #94 decisions (consent deferred), PSD2 regulation
**Exploration:** quick (revised after review)
**Depends on:** D1 (capability sub-interfaces)
**Status:** revised (R1-03: consent removed from BankPlatform SPI; moved to provider-internal infrastructure)

## D4: Dedicated JAX-RS endpoint for consent callback

**Choice:** TrueLayer's consent redirect callback is handled by a dedicated JAX-RS resource (`@Path("/auth/truelayer/callback")`) in `bank-truelayer`. It receives the authorization code via query parameters, calls `TrueLayerConsentService.exchangeCode()`, stores the consent token, and returns a 302 redirect to the application's consent-confirmation page.
**Alternatives:**
- WebhookInboundConnector (original D4) — pattern mismatch: webhooks are server-to-server HTTPS POST with HMAC, returning 200 OK; OAuth2 callbacks are browser GET with query params, requiring 302 redirect. `WebhookRouter` has no redirect result type.
- No callback endpoint (polling only) — consumer owns the HTTP endpoint and calls exchangeCode() after redirect; defers infrastructure to each consumer
**Rationale:** OAuth2 consent redirects and server-to-server webhooks are fundamentally different patterns. The consent callback is a browser redirect (GET, query params, expects 302 back to app). `WebhookRouter.dispatch()` returns 200 OK for all result types — it cannot redirect the user. A dedicated JAX-RS endpoint is the standard OAuth2 callback pattern: simple, correct, well-understood, no modification to existing webhook infrastructure.
**Trade-offs:** Adds a JAX-RS resource to `bank-truelayer` — the module needs `quarkus-rest` (or equivalent) dependency. The callback URL must be registered with TrueLayer as an allowed redirect URI.
**Sources:** R1-04 (reviewer finding: webhook/redirect pattern mismatch), TrueLayer OAuth2 documentation, `webhook/WebhookRouter.java` (verified: no redirect result type)
**Exploration:** quick (revised after review)
**Depends on:** D3 (consent as provider infrastructure)
**Status:** revised (R1-04: replaced WebhookInboundConnector with dedicated JAX-RS endpoint; webhooks can't return 302 redirects)

## D5: Explicit user token on TrueLayerClient, consent token store as provider concern

**Choice:** `TrueLayerClient` takes the user consent token as an explicit parameter on data API methods (same credential-at-call-time pattern as `SlackBotClient`). `TrueLayerConsentService` owns consent token persistence — stores tokens keyed by user ID, retrieves them when `TrueLayerBankPlatform` needs to call the client. No implicit CDI context. `TrueLayerBankPlatform` injects `TrueLayerConsentService`, reads the token for the current user, and passes it to `TrueLayerClient`.
**Alternatives:**
- Request-scoped BankContext (original D5) — no platform precedent for implicit-state CDI context; breaks in non-HTTP contexts (`@Scheduled`, `@ObservesAsync`); ChatPlatform citation was inaccurate (ChatPlatform has no implicit context)
- Token parameter on SPI methods — clutters SPI surface with OAuth2 implementation detail
**Rationale:** The SPI stays clean (no token parameters). The provider implementation bridges between the consent store and the HTTP client. The client follows credential-at-call-time (PP-20260609-0c3e24). No implicit state, no request-scope limitation, no invisible ordering contract. Token persistence is a provider concern — `TrueLayerConsentService` can use in-memory (dev), database (prod), or any storage mechanism without affecting the SPI or the client.
**Trade-offs:** Token storage strategy is deferred to implementation (in-memory for now, database later). `TrueLayerBankPlatform` must resolve the current user's identity to look up their consent token — this requires a user-identity mechanism (platform concern, not connector concern).
**Sources:** R1-05 (reviewer finding: no implicit-state precedent; request-scope limitations), `slack-bot/SlackBotClient.java` (explicit token-per-call), PP-20260609-0c3e24 (credential-config-ownership)
**Exploration:** quick (revised after review)
**Status:** revised (R1-05: replaced implicit BankContext with explicit token passing; consent store is provider-internal)
