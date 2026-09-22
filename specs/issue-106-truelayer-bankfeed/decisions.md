## D1: Unified BankPlatform SPI with capability sub-interfaces

**Choice:** Rename `BankFeedPlatform` → `BankPlatform` with capability sub-interfaces: `AccountInformation` (reads) and `PaymentInitiation` (writes). Single SPI, single `@SimulationEligible` annotation, capability pattern like ChatPlatform.
**Alternatives:**
- Separate flat SPIs (`BankAccountPlatform` + `BankPaymentPlatform`) — artificial read/write split at the SPI level; "account" naturally includes payments
- Single flat interface with all methods — no capability degradation model; forces all providers to implement everything
- Keep `BankFeedPlatform` read-only, add `BankPaymentPlatform` separately — mirrors PSD2 regulatory distinction (AISP/PISP) but leaks regulatory concern into domain model
**Rationale:** From the domain perspective, bank accounts both hold data and execute payments. Splitting by read/write is an API design artifact, not a domain truth. Capability sub-interfaces allow providers to declare what they support (TrueLayer supports both; a read-only provider only wires accounts). The simulation framework's recursive wrapper generation (platform#375) now supports capability interception, removing the flat-interface constraint from D4 (#94).
**Trade-offs:** Requires renaming existing `BankFeedPlatform` and updating `@SimulationEligible(name)`, NoOp, service class, and all test references. Pre-release, this is non-breaking.
**Sources:** `chat-spi/ChatPlatform.java` (capability pattern), `bank-spi/BankFeedPlatform.java` (current flat), platform#375 (recursive wrapper generation), D4 from #94 decisions (flat interface — superseded)
**Exploration:** deep-analysis (evolved through naming discussion → domain modeling → capability architecture)
**Status:** captured

## D2: Managed token holder for TrueLayer OAuth2

**Choice:** `TrueLayerClient` manages OAuth2 access token lifecycle internally — checks expiry before each request, refreshes automatically. Callers pass client credentials (client ID, client secret) at construction time only; they never handle tokens directly.
**Alternatives:**
- Token-per-call (SlackBotClient pattern) — caller passes token on every method; pushes refresh complexity to every caller
- Constructor-injected static credentials (GoogleCalendarPlatform pattern) — works for Google SDK which handles refresh internally; doesn't work with HttpHelper.CLIENT
**Rationale:** TrueLayer access tokens expire (~60 min). Token-per-call would force every caller to implement refresh logic — unnecessary duplication since all callers share the same client credentials. The managed holder encapsulates the token lifecycle. Consistent with credential-config-ownership protocol: CDI factory (Beans class) holds `@ConfigProperty` for credentials and passes them at construction.
**Trade-offs:** Client is stateful (holds mutable token + expiry). Thread-safety needed for concurrent token refresh. Slightly more complex than stateless token-per-call.
**Sources:** `slack-bot/SlackBotClient.java` (token-per-call pattern), `calendar-google/GoogleCalendarPlatform.java` (constructor credentials), PP-20260609-0c3e24 (credential-config-ownership protocol)
**Exploration:** quick
**Status:** captured

## D3: Consent management as a BankPlatform capability

**Choice:** PSD2 consent management is a `ConsentManagement` capability sub-interface on `BankPlatform` — `generateAuthLink()`, `exchangeCode()`, `getConsentStatus()`, `revokeConsent()`. Each provider implements its own consent flow mechanics.
**Alternatives:**
- Separate cross-cutting `ConsentPlatform` SPI — could serve other regulated domains (insurance, health), but speculative; no second regulated SPI exists yet
- Internal to TrueLayerClient (no SPI surface) — callers get errors when consent is missing/expired and handle it ad-hoc; no platform-level consent visibility
**Rationale:** Consent is intrinsic to banking operations — you can't list accounts or initiate payments without it. Making it a capability keeps the consent lifecycle visible at the SPI level (callers can check consent status, initiate consent flows) while keeping it provider-specific (TrueLayer's redirect URLs differ from Yapily's). D7 from #94 explicitly deferred this to "when a real PSD2 provider is implemented" — this is that moment.
**Trade-offs:** Ties consent to BankPlatform rather than a reusable cross-cutting concern. If a second regulated SPI needs consent, we may extract a shared interface. Pre-release, this refactoring is cheap.
**Sources:** D7 from #94 decisions (consent deferred), PSD2 regulation (AISP/PISP consent requirements), TrueLayer auth documentation
**Exploration:** quick
**Depends on:** D1 (capability sub-interfaces)
**Status:** captured

## D4: Consent callback via WebhookInboundConnector

**Choice:** TrueLayer's consent redirect callback (after user authorizes at their bank) is handled via the existing `WebhookInboundConnector` SPI. The callback is just another inbound webhook — consistent with how other external callbacks work in the platform.
**Alternatives:**
- Dedicated REST endpoint in bank-truelayer — tighter coupling but simpler; no routing through generic webhook system
- No callback endpoint (polling only) — `generateAuthLink()` returns a link, consumer calls `exchangeCode()` after redirect; consumer owns the HTTP endpoint
**Rationale:** WebhookInboundConnector already handles inbound callbacks from external systems. TrueLayer's consent redirect is the same pattern — an external system redirecting back with an authorization code. Reusing the existing SPI avoids duplicating endpoint infrastructure and keeps the inbound flow consistent across all connectors.
**Trade-offs:** Adds a dependency on the webhook module. The webhook system's generic routing must map the TrueLayer callback path to the bank-truelayer handler.
**Sources:** `webhook/WebhookInboundConnector.java` (existing SPI), `core/InboundConnector.java` (inbound pattern), TrueLayer auth redirect documentation
**Exploration:** quick
**Depends on:** D3 (consent as capability)
**Status:** captured

## D5: Request-scoped BankContext for per-user consent tokens

**Choice:** A request-scoped `BankContext` CDI bean holds the current user's consent token. `TrueLayerBankPlatform` reads it implicitly — no token in SPI method signatures. The client (`TrueLayerClient`) still takes the token as a parameter per credential-config-ownership; the platform impl bridges BankContext → client call.
**Alternatives:**
- Token parameter on each SPI method — explicit but clutters every `Accounts` method signature; SPI surface becomes noisy and tied to OAuth2 implementation detail
- Consent store internal to client — TrueLayerClient queries a platform-level consent store by userId; tighter coupling to storage, harder to test
**Rationale:** TrueLayer has two token layers: client credentials (machine-to-machine, shared, managed by D2) and user consent tokens (per-user, per-bank, obtained via consent flow). User tokens can't live in the singleton client. A request-scoped context holder keeps the SPI clean while making the token available where needed. The platform (caller) sets the context; the provider reads it.
**Trade-offs:** Implicit state — the caller must set BankContext before calling accounts(). If forgotten, the platform gets a null token and fails at runtime. Mitigated by clear error messages and test fixtures that set up context.
**Sources:** ChatPlatform (implicit context patterns), PP-20260609-0c3e24 (credential-config-ownership — token at call time on the client, implicit on the SPI)
**Exploration:** quick
**Depends on:** D2 (managed token holder), D3 (consent capability)
**Status:** captured
