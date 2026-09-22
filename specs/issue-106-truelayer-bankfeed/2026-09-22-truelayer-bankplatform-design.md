# TrueLayer BankPlatform Implementation — Design Spec

> **Issue:** casehubio/connectors#106
> **Date:** 2026-09-22
> **Status:** Draft
> **Supersedes:** D4, D7, D8 from #94 decisions (flat interface, consent deferral, pagination)

## Overview

First real provider implementation for the banking SPI. Renames
`BankFeedPlatform` → `BankPlatform` with capability sub-interfaces
following the ChatPlatform pattern, adds payment initiation alongside
account information, and implements TrueLayer as the first provider.

Two modules are affected: `bank-spi` (SPI evolution) and `bank-truelayer`
(new provider module). The SPI changes are breaking relative to the
current `BankFeedPlatform` but pre-release — no external consumers.

## Module changes

### bank-spi — SPI evolution

| Change | Detail |
|--------|--------|
| Rename `BankFeedPlatform` → `BankPlatform` | Interface, `@SimulationEligible(name)`, NoOp, service class |
| Add capability sub-interfaces | `AccountInformation`, `PaymentInitiation` |
| Add payment model records | `PaymentRequest`, `InitiatedPayment`, `PaymentStatus` |
| Move existing methods | `listAccounts()`, `balance()`, `listTransactions()`, `getTransaction()` → `AccountInformation` |
| Update `@SimulationEligible` | `name = "bank-platform"` (was `"bank-feed-platform"`) |

### bank-truelayer — new module

| Artifact | `casehub-connectors-bank-truelayer` |
|----------|-------------------------------------|
| Package | `io.casehub.connectors.bank.truelayer` |
| Dependencies | `bank-spi`, `connectors-api`, `quarkus-oidc-client`, `quarkus-rest` |

## BankPlatform SPI

```java
@SimulationEligible(name = "bank-platform")
public interface BankPlatform {

    String id();

    AccountInformation accountInformation();

    PaymentInitiation paymentInitiation();
}
```

Capability sub-interfaces follow the ChatPlatform pattern. Providers
declare which capabilities they support. The simulation framework's
recursive wrapper generation (platform#375) intercepts methods on
returned capability interfaces.

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

Methods are unchanged from the current `BankFeedPlatform`. Error contract
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

### Payment model records

```java
public record PaymentRequest(BigDecimal amount, String currency,
                             String beneficiaryName, String sortCode,
                             String accountNumber, String reference) {}

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
floating point. `sortCode` and `accountNumber` are UK-specific;
future providers may need IBAN. Pre-release, evolving the record
is non-breaking.

### NoOp fallback

```java
@DefaultBean
@ApplicationScoped
public class NoOpBankPlatform implements BankPlatform {
    @Override public String id() { return "none"; }
    @Override public AccountInformation accountInformation() {
        return NoOpAccountInformation.INSTANCE;
    }
    @Override public PaymentInitiation paymentInitiation() {
        throw new UnsupportedOperationException("No bank platform configured");
    }
}
```

`NoOpAccountInformation` returns empty lists for list operations and
throws `UnsupportedOperationException` for single-item lookups (same
as current `NoOpBankFeedPlatform`). `paymentInitiation()` throws
directly — no-op payment initiation is meaningless.

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

    public AuthLink generateAuthLink(List<ConsentScope> scopes,
                                     String redirectUri, String state) { ... }

    public ConsentInfo exchangeCode(String code, String state) { ... }

    public ConsentInfo consentStatus(String userId) { ... }

    public void revokeConsent(String userId) { ... }

    public String getUserToken(String userId) { ... }
}
```

`getUserToken()` retrieves the stored consent token for a user. Returns
null if no consent exists or consent has expired. Called by
`TrueLayerBankPlatform` before data API operations.

### Consent token persistence

In-memory `ConcurrentHashMap<String, StoredConsent>` for initial
implementation. `StoredConsent` holds the access token, refresh token,
expiry timestamp, and granted scopes. Sufficient for dev and test.

Production persistence (database-backed) is a follow-up concern — the
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

    @Override public AccountInformation accountInformation() {
        return new TrueLayerAccountInformation(client, consentService);
    }

    @Override public PaymentInitiation paymentInitiation() {
        return new TrueLayerPaymentInitiation(client, consentService);
    }
}
```

`TrueLayerAccountInformation` reads the user's consent token from
`TrueLayerConsentService` and passes it to `TrueLayerClient` on each
call. If consent is missing or expired, throws
`ConsentExpiredException`.

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
- `TrueLayerPaymentResult` → `InitiatedPayment`
- `TrueLayerPaymentStatus` → `PaymentStatus` enum

## CDI wiring

```java
@ApplicationScoped
public class TrueLayerBeans {

    @Produces @ApplicationScoped
    public TrueLayerClient trueLayerClient(
            OidcClient oidcClient,
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
- Auth link generation with correct scopes
- Code exchange and token storage
- Token retrieval by user ID
- Consent expiry and revocation

### Simulation integration

`@SimulationEligible(name = "bank-platform")` generates decorators for
all capability methods. Qualified names:

| Qualified name | Method |
|---------------|--------|
| `bank-platform.id` | `BankPlatform.id()` |
| `bank-platform.accountInformation.listAccounts` | `AccountInformation.listAccounts()` |
| `bank-platform.accountInformation.balance` | `AccountInformation.balance(accountId)` |
| `bank-platform.accountInformation.listTransactions` | `AccountInformation.listTransactions(...)` |
| `bank-platform.accountInformation.getTransaction` | `AccountInformation.getTransaction(...)` |
| `bank-platform.paymentInitiation.initiatePayment` | `PaymentInitiation.initiatePayment(...)` |
| `bank-platform.paymentInitiation.paymentStatus` | `PaymentInitiation.paymentStatus(...)` |

**Note:** Qualified name format for capability methods depends on how
platform#375's recursive wrapper generator names nested interface
methods. The dot-separated paths above (`bank-platform.accountInformation.listAccounts`)
are the expected pattern — verify against the generator during implementation.

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

- Update existing bank-spi layer entry with capability architecture
- Add new layer for TrueLayer provider
- Update module structure table with `bank-truelayer`

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
