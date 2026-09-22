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
