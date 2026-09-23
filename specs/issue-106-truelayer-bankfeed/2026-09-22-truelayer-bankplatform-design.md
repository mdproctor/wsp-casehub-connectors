# TrueLayer BankPlatform Implementation — Design Spec

> **Issue:** casehubio/connectors#106
> **Date:** 2026-09-22
> **Status:** Draft
> **Supersedes:** D4, D7, D8 from #94 decisions (flat interface, consent deferral, pagination)

## Overview

First real provider implementation for the banking SPI. Renames
`BankPlatform` → `BankPlatform` with capability sub-interfaces
following the ChatPlatform pattern, adds payment initiation alongside
account information, and implements TrueLayer as the first provider.

Three modules are affected: `bank-spi` (SPI evolution), `bank-truelayer`
(new provider module), and `graphql` (migration to capability accessors
and updated MCP domain). The SPI changes are breaking relative to the
current `BankPlatform` but pre-release — no external consumers.

## Module changes

### bank-spi — SPI evolution

| Change | Detail |
|--------|--------|
| Rename `BankPlatform` → `BankPlatform` | Interface, `@SimulationEligible(name)`, NoOp, service class |
| Add capability sub-interfaces | `AccountInformation`, `PaymentInitiation` |
| Add `supports(Class<?>)` | Runtime capability introspection (ChatPlatform pattern) |
| User-scoped capability accessors | `accountInformation(userId)`, `paymentInitiation(userId)` |
| Add payment model records | `PaymentRequest`, `InitiatedPayment`, `PaymentStatus`, `PaymentDestination` |
| Move existing methods | `listAccounts()`, `balance()`, `listTransactions()`, `getTransaction()` → `AccountInformation` |
| Update `@SimulationEligible` | `name = "bank-platform"`, `capabilities = {"accountInformation", "paymentInitiation"}` |

### bank-truelayer — new module

| Artifact | `casehub-connectors-bank-truelayer` |
|----------|-------------------------------------|
| Package | `io.casehub.connectors.bank.truelayer` |
| Dependencies | `bank-spi`, `connectors-api`, `quarkus-oidc-client`, `quarkus-rest` |

### graphql — migration

| Change | Detail |
|--------|--------|
| Rename `ConnectorBankApi` → `ConnectorBankApi` | Class name, `@McpDomain` value and basePath |
| Update imports | `BankPlatform` → `BankPlatform`, `BankPlatformService` → `BankPlatformService` |
| Route through capability accessors | `p.listAccounts()` → `p.accountInformation(userId).listAccounts()` etc. |
| Add user identity | Inject `SecurityIdentity`, extract userId for capability accessor calls |
| Update `@McpDomain` | `value = "connectors/bank"`, `basePath = "/api/connectors/bank"` |
| Add payment endpoints | Expose `paymentInitiation` operations if PISP is available |

## BankPlatform SPI

```java
@SimulationEligible(name = "bank-platform",
    capabilities = {"accountInformation", "paymentInitiation"})
public interface BankPlatform {

    String id();

    AccountInformation accountInformation(String userId);

    PaymentInitiation paymentInitiation(String userId);

    boolean supports(Class<?> capability);
}
```

Capability sub-interfaces follow the ChatPlatform pattern. Providers
declare which capabilities they support. The simulation framework's
recursive wrapper generation (platform#375) intercepts methods on
returned capability interfaces. The `capabilities` attribute lists
sub-interface accessor methods for recursive wrapper generation.

Banking operations are inherently per-user under PSD2 — each user
grants their own consent. Capability accessors take a `userId` parameter
so the provider can resolve user-specific state (consent tokens, scope
checks) when constructing the capability instance. This is a domain
concern, not an OAuth2 concern: the question "whose accounts?" is
fundamental to banking, unlike chat where operations are per-platform.

`supports(Class<?> capability)` enables runtime capability introspection
without discovery-by-exception. Providers implement it directly (no
Builder needed for 2 capabilities — see §Design rationale).

### AccountInformation (AISP)

```java
public interface AccountInformation {

    List<AccountInfo> listAccounts();

    AccountBalance balance(String accountId);

    Page<Transaction> listTransactions(String accountId,
                                       Instant from, Instant to,
                                       PageRequest pagination);

    Transaction getTransaction(String accountId, String transactionId);
}
```

Methods are unchanged from the current `BankPlatform`. Error contract
unchanged: `NoSuchElementException` for unknown entities, unchecked
provider-specific exceptions for transport errors.

### PaymentInitiation (PISP)

```java
public interface PaymentInitiation {

    InitiatedPayment initiatePayment(PaymentRequest payment);

    PaymentStatus paymentStatus(String paymentId);
}
```

Payment initiation is asynchronous — `initiatePayment()` returns an
`InitiatedPayment` with a hosted payment page URL. The user completes
Strong Customer Authentication (SCA) at their bank via the hosted page.
`paymentStatus()` polls the result.

**Payment webhooks** — TrueLayer sends webhook notifications for payment
lifecycle events (authorized → executed → settled, or → failed). This
initial implementation supports polling only via `paymentStatus()`.
Webhook-based real-time status updates are deferred to
**casehubio/connectors#108** — adding a webhook
receiver (JAX-RS endpoint, TrueLayer signature validation, CDI event
firing) is a natural follow-up that does not affect the SPI surface.

### Payment model records

```java
public record PaymentRequest(String idempotencyKey,
                             BigDecimal amount, String currency,
                             String beneficiaryName,
                             PaymentDestination destination,
                             String reference) {}

public sealed interface PaymentDestination {
    record UkAccount(String sortCode, String accountNumber)
            implements PaymentDestination {}
    record IbanAccount(String iban, String bic)
            implements PaymentDestination {}
}

public record InitiatedPayment(String paymentId,
                                String hostedPaymentPageLink,
                                PaymentStatus status) {}

public enum PaymentStatus {
    AUTHORIZATION_REQUIRED, AUTHORIZING, AUTHORIZED,
    EXECUTED, SETTLED, FAILED
}
```

`PaymentRequest.amount` is always positive, consistent with
`Transaction.amount`. `BigDecimal` for monetary amounts — never
floating point.

`idempotencyKey` is caller-generated (UUID recommended) and must be
unique per logical payment attempt. Providers pass it as-is to the
upstream API (TrueLayer: `Idempotency-Key` header). Callers that
retry a failed `initiatePayment()` reuse the same key — the provider
returns the original result rather than creating a duplicate payment.
This is an SPI concern, not a provider detail: callers must generate
and track the key to retry safely.

`PaymentDestination` is a sealed hierarchy for payment rail types.
`UkAccount` covers UK Faster Payments (sort code + account number);
`IbanAccount` covers SEPA/international (IBAN + BIC). Additional
payment rail types (e.g., `UsAccount(routingNumber, accountNumber)`)
are added as new sealed permits — existing exhaustive switches catch
the compile error. This avoids nullable fields and preserves type
safety as providers for other payment rails are added.

### NoOp fallback

```java
@DefaultBean
@ApplicationScoped
public class NoOpBankPlatform implements BankPlatform {
    @Override public String id() { return "none"; }
    @Override public AccountInformation accountInformation(String userId) {
        return NoOpAccountInformation.INSTANCE;
    }
    @Override public PaymentInitiation paymentInitiation(String userId) {
        return NoOpPaymentInitiation.INSTANCE;
    }
    @Override public boolean supports(Class<?> capability) { return false; }
}
```

`NoOpAccountInformation` returns empty lists for list operations and
throws `UnsupportedOperationException` for single-item lookups (same
as current `NoOpBankPlatform`). `NoOpPaymentInitiation` throws
`UnsupportedOperationException` from its operation methods
(`initiatePayment()`, `paymentStatus()`), not from the accessor.
This follows the ChatPlatform pattern: capability accessors never
throw — callers can safely obtain a reference and check
`supports(PaymentInitiation.class)` before invoking operations.

### Design rationale — no Builder pattern

ChatPlatform uses a Builder to manage 9 optional capabilities with
degradation defaults and native capability tracking. BankPlatform has
2 capabilities — the construction complexity that justifies a Builder
does not exist here:

- `AccountInformation` is fundamental — every bank provider supports it
- `PaymentInitiation` is optional — AISP-only providers omit it
- `supports()` is implemented directly by each provider (trivial for 2 capabilities)
- `NoOpBankPlatform` returns `NoOpPaymentInitiation` (per ChatPlatform pattern)

If the capability count grows beyond 3–4, a Builder should be introduced
(the refactoring is mechanical). Until then, direct construction is
simpler without sacrificing correctness.

## TrueLayer HTTP client

`TrueLayerClient` is a shared HTTP client using `HttpHelper.CLIENT`
(per shared-http-client protocol PP-20260607-9794cb). It handles
TrueLayer Data API and Payments API calls.

### Authentication

Client credentials OAuth2 via `quarkus-oidc-client` (D2). The
`OidcClient` bean manages token caching, automatic refresh, and
thread-safe access. `TrueLayerClient` calls `oidcClient.getTokens()`
before each request.

User consent tokens (per-user, per-bank) are passed as explicit
parameters on data API methods (D5). `TrueLayerClient` never stores
user tokens.

```java
public class TrueLayerClient {

    private static final Logger LOG = Logger.getLogger(TrueLayerClient.class);
    private static final int MAX_PAGES = 50;

    private final OidcClient oidcClient;
    private final String baseUrl;

    // Data API — user consent token passed per-call
    public List<TrueLayerAccount> listAccounts(String userToken) { ... }
    public TrueLayerBalance balance(String userToken, String accountId) { ... }
    public TrueLayerTransactionPage listTransactions(String userToken,
            String accountId, String from, String to, String cursor) { ... }
    public TrueLayerTransaction getTransaction(String userToken,
            String accountId, String transactionId) { ... }

    // Payments API — client credentials (via OidcClient)
    public TrueLayerPaymentResult createPayment(TrueLayerPaymentRequest request) { ... }
    public TrueLayerPaymentStatus paymentStatus(String paymentId) { ... }
}
```

All methods use `HttpHelper.CLIENT.send(...)`. The client returns
TrueLayer-specific DTOs (prefixed `TrueLayer*`). The platform
implementation maps these to SPI model records.

### Pagination

`listTransactions()` follows the paginating-client-fail-soft protocol
(PP-20260610-83747b): returns partial results + WARNING on mid-loop
failure, `MAX_PAGES` constant with distinct cap-hit warning. TrueLayer
uses cursor-based pagination — the cursor string maps directly to
`PageRequest.cursor()`.

### Error handling

TrueLayer API errors map to domain-appropriate exceptions:
- 401 Unauthorized → `ConsentExpiredException` (consent token expired
  or revoked; caller should initiate re-consent)
- 404 Not Found → `NoSuchElementException` (consistent with SPI error
  contract)
- 429 Rate Limited → `RateLimitedException` with `retryAfterSeconds`
- 5xx → `TrueLayerApiException` (transient; caller can retry)

`ConsentExpiredException` and `RateLimitedException` are new exception
types in `bank-truelayer`, not on the SPI. The SPI contract is unchecked
exceptions — these subtypes give callers the ability to distinguish
failure modes without requiring it.

## Consent management

Consent is provider-internal infrastructure, not on the BankPlatform
SPI (D3). `TrueLayerConsentService` handles the full PSD2 consent
lifecycle.

### TrueLayerConsentService

```java
@ApplicationScoped
public class TrueLayerConsentService {

    public AuthLink generateAuthLink(String userId,
                                     List<ConsentScope> scopes,
                                     String redirectUri) { ... }

    public ConsentInfo exchangeCode(String code, String state) { ... }

    public ConsentInfo consentStatus(String userId) { ... }

    public void revokeConsent(String userId) { ... }

    public String getUserToken(String userId) { ... }
}
```

`getUserToken()` retrieves a valid access token for a user. Called by
`TrueLayerAccountInformation` and `TrueLayerPaymentInitiation` before
data API operations. The method implements a three-step resolution:

1. **Valid access token** — if the stored access token has not expired,
   return it immediately.
2. **Refresh** — if the access token has expired but a refresh token is
   available, call TrueLayer's token endpoint with the refresh token to
   obtain a new access token. Store the new access token (and updated
   refresh token if rotated) in `StoredConsent`. Return the new access
   token.
3. **No refresh possible** — if no consent exists, or the refresh token
   is also expired/revoked (TrueLayer returns 401), return null. The
   caller throws `ConsentExpiredException` to trigger re-consent.

This ensures that the PSD2 90-day AISP consent window is honoured.
Without refresh, access tokens expire in ~1 hour, forcing hourly
re-consent despite the 90-day consent grant. The refresh call uses
`HttpHelper.CLIENT` with the client credentials (via `OidcClient`) and
the stored refresh token — no user interaction required.

Thread safety: refresh is synchronized per-userId to prevent concurrent
calls from triggering duplicate refresh requests. A double-check pattern
(re-read after acquiring the lock) handles the race where another thread
already refreshed.

### Consent token persistence

In-memory `ConcurrentHashMap<String, StoredConsent>` for initial
implementation. `StoredConsent` holds the access token, refresh token,
expiry timestamp, and granted scopes. Sufficient for dev and test.

Production persistence (database-backed) is deferred to
**casehubio/connectors#107** — a JVM restart
loses all consent tokens, forcing re-consent (bank redirect, SCA)
for every user. With PSD2's 90-day AISP consent window, this is
significant operational friction in production. The
`TrueLayerConsentService` interface is stable; only the storage
implementation changes.

### Consent callback endpoint

Dedicated JAX-RS resource (D4):

```java
@Path("/auth/truelayer")
public class TrueLayerAuthCallback {

    @GET
    @Path("/callback")
    public Response callback(@QueryParam("code") String code,
                             @QueryParam("state") String state) {
        ConsentInfo consent = consentService.exchangeCode(code, state);
        URI successPage = buildSuccessRedirect(consent);
        return Response.seeOther(successPage).build();
    }
}
```

Receives the authorization code via browser GET, exchanges it for
tokens, stores the consent, and returns 302 redirect to the
application's consent-confirmation page. Standard OAuth2 callback
pattern.

### State parameter — CSRF protection

The OAuth2 `state` parameter prevents CSRF attacks where an attacker
crafts a redirect that associates their TrueLayer consent with a
victim's account (RFC 6749 §10.12).

**Generation:** `generateAuthLink()` creates a cryptographically random
opaque state value (128-bit `SecureRandom`, hex-encoded) and stores it
as `stateValue → {userId, scopes, expiry}` — the state IS the key,
mapping TO the userId. The state is included in the authorization URL's
`state` query parameter. TTL: 10 minutes.

**Validation:** `exchangeCode(code, state)` looks up `state` in the
pending-consent map → retrieves the associated userId and scopes. If the
state doesn't exist or has expired → `InvalidStateException`. On
successful lookup the entry is deleted (single-use). The resolved userId
is used to store the exchanged consent tokens — this is how the callback
(which has no authenticated user context) associates the consent with the
correct user.

This keying model (`state → userId`) is standard OAuth2: the callback
endpoint receives only `code` and `state` from the redirect — no session,
no JWT, no `SecurityIdentity`. The state is the only link back to the
originating user.

### Consent scopes

PSD2 mandates separate consent for AISP and PISP:
- AISP consent: accounts, balance, transactions (90-day duration)
- PISP consent: payment initiation (per-payment or standing consent)

`TrueLayerConsentService` tracks which scopes are granted per user.
`TrueLayerBankPlatform` checks scope before delegating to the client —
e.g., `paymentInitiation()` verifies PISP consent exists.

## TrueLayerBankPlatform

```java
@ApplicationScoped
public class TrueLayerBankPlatform implements BankPlatform {

    private final TrueLayerClient client;
    private final TrueLayerConsentService consentService;

    @Override public String id() { return "truelayer"; }

    @Override public AccountInformation accountInformation(String userId) {
        return new TrueLayerAccountInformation(client, consentService, userId);
    }

    @Override public PaymentInitiation paymentInitiation(String userId) {
        return new TrueLayerPaymentInitiation(client, consentService, userId);
    }

    @Override public boolean supports(Class<?> capability) {
        return capability == AccountInformation.class
            || capability == PaymentInitiation.class;
    }
}
```

`TrueLayerAccountInformation` captures the `userId` at construction.
On each data API call, it calls `consentService.getUserToken(userId)` —
which returns a valid access token (refreshing automatically if the
current token has expired). If `getUserToken()` returns null (no consent
or refresh token revoked), throws `ConsentExpiredException` to signal
that the user must re-consent via the browser redirect flow.

`TrueLayerPaymentInitiation` uses client credentials (via `OidcClient`)
for payment creation. SCA is handled by the hosted payment page — the
user authenticates at their bank, not through the SPI.

### Data mapping

`TrueLayerAccountInformation` maps TrueLayer DTOs → SPI model records:
- `TrueLayerAccount` → `AccountInfo` (id, name, type, currency)
- `TrueLayerBalance` → `AccountBalance` (available, current, currency, asOf)
- `TrueLayerTransaction` → `Transaction` (amount as positive BigDecimal,
  direction from sign, nullable merchantName/category)
- `TrueLayerTransactionPage` → `Page<Transaction>` (cursor mapping)

`TrueLayerPaymentInitiation` maps:
- `PaymentRequest.destination` → TrueLayer sort code/account number
  fields (switches on `PaymentDestination` sealed type — TrueLayer
  supports `UkAccount` only; `IbanAccount` throws
  `UnsupportedOperationException` until SEPA support is added)
- `PaymentRequest.idempotencyKey` → `Idempotency-Key` HTTP header
- `TrueLayerPaymentResult` → `InitiatedPayment`
- `TrueLayerPaymentStatus` → `PaymentStatus` enum

## CDI wiring

```java
@ApplicationScoped
public class TrueLayerBeans {

    @Produces @ApplicationScoped
    public TrueLayerClient trueLayerClient(
            @NamedOidcClient("truelayer") OidcClient oidcClient,
            @ConfigProperty(name = "casehub.connectors.bank.truelayer.base-url",
                            defaultValue = "https://api.truelayer.com") String baseUrl) {
        return new TrueLayerClient(oidcClient, baseUrl);
    }

    @Produces @ApplicationScoped
    public TrueLayerBankPlatform trueLayerBankPlatform(
            TrueLayerClient client, TrueLayerConsentService consentService) {
        return new TrueLayerBankPlatform(client, consentService);
    }
}
```

Client credentials configuration via `application.properties`:

```properties
quarkus.oidc-client.truelayer.auth-server-url=https://auth.truelayer.com
quarkus.oidc-client.truelayer.client-id=${TRUELAYER_CLIENT_ID}
quarkus.oidc-client.truelayer.credentials.secret=${TRUELAYER_CLIENT_SECRET}
quarkus.oidc-client.truelayer.grant.type=client_credentials
```

## Testing

### Unit tests (WireMock)

`TrueLayerClientTest` — WireMock for all HTTP interactions:
- Token refresh lifecycle (expiry, re-fetch)
- Pagination (multi-page, cap-hit, mid-loop failure with partial results)
- Error mapping (401→ConsentExpired, 404→NoSuchElement, 429→RateLimited)
- Data mapping (TrueLayer DTOs → SPI model records)

`TrueLayerConsentServiceTest` — consent lifecycle:
- Auth link generation with userId, correct scopes, and internal state
  storage (`state → {userId, scopes, expiry}`)
- State validation on code exchange (correct state, expired state,
  unknown state, userId resolution from state)
- Code exchange and token storage
- Access token refresh on expiry (valid refresh token → new access token)
- Refresh token rotation (new refresh token stored when returned)
- Refresh failure (revoked refresh token → null → re-consent)
- Thread-safe refresh (concurrent calls share a single refresh)
- Consent revocation

### Simulation integration

`@SimulationEligible(name = "bank-platform")` generates decorators for
all capability methods. Qualified names:

| Qualified name | Method |
|---------------|--------|
| `bank-platform.id` | `BankPlatform.id()` |
| `bank-platform.accountInformation` | `BankPlatform.accountInformation(userId)` |
| `bank-platform.accountInformation.listAccounts` | `AccountInformation.listAccounts()` |
| `bank-platform.accountInformation.balance` | `AccountInformation.balance(accountId)` |
| `bank-platform.accountInformation.listTransactions` | `AccountInformation.listTransactions(...)` |
| `bank-platform.accountInformation.getTransaction` | `AccountInformation.getTransaction(...)` |
| `bank-platform.paymentInitiation` | `BankPlatform.paymentInitiation(userId)` |
| `bank-platform.paymentInitiation.initiatePayment` | `PaymentInitiation.initiatePayment(...)` |
| `bank-platform.paymentInitiation.paymentStatus` | `PaymentInitiation.paymentStatus(...)` |

**Note:** The `capabilities = {"accountInformation", "paymentInitiation"}`
attribute on `@SimulationEligible` tells the `SimulationDecoratorProcessor`
which methods return capability sub-interfaces. The processor generates
wrappers for both the accessor methods and their returned interfaces.
The userId parameter on capability accessors is passed through by the
decorator — simulation stubs at the leaf method level
(`bank-platform.accountInformation.listAccounts`) ignore it.

Test fixtures via `Simulation.forTest()`:

```java
var sim = Simulation.forTest()
    .stub("bank-platform.id", null, "truelayer")
    .stub("bank-platform.accountInformation.listAccounts", null,
          List.of(new AccountInfo("acc-001", "Current Account",
                                  AccountType.CURRENT, "GBP")))
    .build();
```

## Deliverables beyond code

### ARC42STORIES.MD update

- **L12 rename:** "Bank Feed Platform SPI" → "Bank Platform SPI". Update
  description to reflect capability architecture (`AccountInformation`,
  `PaymentInitiation`), `supports()`, user-scoped capability accessors,
  `PaymentDestination` sealed hierarchy, and `BankPlatformService`. Update
  key files list (interface, model records, NoOp, service class). Update
  `@SimulationEligible(name)` reference from `bank-feed-platform` to
  `bank-platform`.
- **L14 (new):** "TrueLayer Bank Provider" — `TrueLayerBankPlatform`
  (AISP + PISP), `TrueLayerClient` (Data API + Payments API),
  `TrueLayerConsentService` (PSD2 consent lifecycle), OAuth2 callback
  endpoint, `quarkus-oidc-client` for client credentials. Key files:
  `TrueLayerBankPlatform.java`, `TrueLayerClient.java`,
  `TrueLayerConsentService.java`, `TrueLayerAuthCallback.java`,
  `TrueLayerBeans.java`.
- **§5 Module structure table:** Add `bank-truelayer` row:
  `casehub-connectors-bank-truelayer` depends on `bank-spi`,
  `connectors-api`, `quarkus-oidc-client`, `quarkus-rest`.
- **§5 Container diagram:** Add L14 container boundary with
  TrueLayer-specific containers.

### Guides update

- `docs/guides/consumer-guide.md` — BankPlatform capability usage,
  consent flow integration pattern
- `docs/guides/contributor-guide.md` — TrueLayer client architecture,
  adding new bank providers

## References

- `bank-spi/BankFeedPlatform.java` — current flat SPI (to be renamed)
- `chat-spi/ChatPlatform.java` — capability sub-interface pattern
- `calendar-google/GoogleCalendarPlatform.java` — provider implementation reference
- `calendar-google/CalendarGoogleBeans.java` — CDI wiring pattern
- `slack-bot/SlackBotClient.java` — credential-at-call-time pattern, pagination
- `connectors-api/HttpHelper.java` — shared HTTP client singleton
- `webhook/WebhookInboundConnector.java` — evaluated and rejected for consent callback (D4)
- PP-20260607-9794cb — shared HTTP client protocol
- PP-20260609-0c3e24 — credential config ownership protocol
- PP-20260610-83747b — paginating client fail-soft protocol
- PP-20260609-e3a2bd — SPI id() naming convention
- D1–D11 from #94 decisions — BankFeedPlatform SPI design (D4 flat interface superseded)
- platform#375 — recursive wrapper generation for capability-based SPIs
- PSD2 regulation — AISP/PISP licensing, consent scopes, 90-day re-consent
- TrueLayer Data API and Payments API documentation
