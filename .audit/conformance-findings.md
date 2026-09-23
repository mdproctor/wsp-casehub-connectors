# Conformance Check — issue-106-truelayer-bankfeed

## Spec requirements → implementation mapping

| Spec requirement | Status | Evidence |
|---|---|---|
| Rename BankFeedPlatform → BankPlatform | ✅ | `BankPlatform.java` exists, old file deleted |
| `@SimulationEligible(name="bank-platform", capabilities={...})` | ✅ | Line 5-6 of `BankPlatform.java` |
| `AccountInformation` sub-interface | ✅ | `AccountInformation.java` with all 4 methods |
| `PaymentInitiation` sub-interface | ✅ | `PaymentInitiation.java` with `initiatePayment` + `paymentStatus` |
| `supports(Class<?>)` | ✅ | Line 15 of `BankPlatform.java` |
| User-scoped accessors (`accountInformation(userId)`) | ✅ | Lines 11, 13 of `BankPlatform.java` |
| Payment model records (PaymentRequest, InitiatedPayment, PaymentStatus) | ✅ | All 4 files in `bank/model/` |
| `PaymentDestination` sealed hierarchy (UkAccount, IbanAccount) | ✅ | `PaymentDestination.java` |
| `idempotencyKey` on PaymentRequest | ✅ | First field of record |
| NoOp fallback with capability pattern | ✅ | `NoOpBankPlatform.java`, `NoOpAccountInformation.java`, `NoOpPaymentInitiation.java` |
| `BankPlatformService` routing | ✅ | Renamed, error messages updated |
| bank-truelayer module created | ✅ | `bank-truelayer/pom.xml` with correct deps |
| TrueLayerClient using HttpHelper.CLIENT | ✅ | `TrueLayerClient.java` line 150 |
| TrueLayer DTOs (7 records) | ✅ | All in `dto/` package |
| Error mapping (401→ConsentExpired, 404→NoSuchElement, 429→RateLimited, 5xx→ApiException) | ✅ | `handleResponse()` method |
| TrueLayerConsentService with state-based CSRF | ✅ | `generateAuthLink()` with SecureRandom state, `exchangeCode()` with state lookup |
| Three-step getUserToken (valid→refresh→null) | ✅ | `getUserToken()` + `refreshToken()` methods |
| Per-userId synchronized refresh | ✅ | `refreshLocks` + `synchronized(lock)` |
| In-memory ConcurrentHashMap consent storage | ✅ | `consents` field |
| TrueLayerAuthCallback JAX-RS endpoint | ✅ | `@Path("/auth/truelayer")` with error handling |
| TrueLayerBankPlatform implementing BankPlatform | ✅ | All methods, both capabilities |
| Data mapping (TrueLayer DTOs → SPI records) | ✅ | `TrueLayerAccountInformation`, `TrueLayerPaymentInitiation` |
| TrueLayerBeans CDI wiring | ✅ | Producer methods for client + consent service |
| GraphQL: rename to ConnectorBankApi | ✅ | File renamed |
| GraphQL: @McpDomain("connectors/bank") | ✅ | Line 26 |
| GraphQL: SecurityIdentity injection | ✅ | Line 31 |
| GraphQL: capability accessor routing | ✅ | All methods route through `accountInformation(userId())` |
| GraphQL: payment endpoints | ✅ | `initiatePayment()` + `paymentStatus()` |
| ARC42STORIES: L12 rename | ✅ | Line 243 |
| ARC42STORIES: L14 new layer | ✅ | Line 250 |
| ARC42STORIES: module table | ✅ | Line 191 |
| Consumer guide update | ✅ | BankPlatform section rewritten |
| Contributor guide update | ✅ | References updated |
| Parent pom: bank-truelayer module | ✅ | Added after bank-spi |
| WireMock tests for TrueLayerClient | ✅ | 7 tests |
| Consent service tests | ✅ | 9 tests |
| TrueLayerBankPlatform tests | ✅ | 6 tests |

## Gaps found

### 1. `@NamedOidcClient("truelayer")` NOT used on CDI injection

**Spec says (§CDI wiring, line 474):** `@NamedOidcClient("truelayer") OidcClient oidcClient`
**Implementation:** `TrueLayerBeans.java` line 14 injects bare `OidcClient oidcClient` without `@NamedOidcClient`. The spec's `application.properties` uses named config (`quarkus.oidc-client.truelayer.*`), but the actual `application.properties` in the module uses custom config keys (`casehub.connectors.bank.truelayer.*`), not quarkus-oidc-client named config. This means the OIDC client integration is structurally different from the spec — no quarkus-oidc-client named client is configured or injected.

### 2. `application.properties` does NOT configure quarkus-oidc-client

**Spec says (§CDI wiring, lines 490-494):** `quarkus.oidc-client.truelayer.auth-server-url=...` etc.
**Implementation:** Only custom `casehub.connectors.bank.truelayer.*` properties. No `quarkus.oidc-client.*` configuration. This means `quarkus-oidc-client` is a dependency but is not actually configured or used for client credentials OAuth2.

### 3. Pagination: no multi-page loop or MAX_PAGES cap-hit warning

**Spec says (§Pagination, lines 265-269):** `listTransactions()` follows the paginating-client-fail-soft protocol with `MAX_PAGES` constant and distinct cap-hit warning.
**Implementation:** `TrueLayerClient.listTransactions()` fetches a single page and returns it. No pagination loop, no `MAX_PAGES` enforcement, no partial-result-on-failure WARNING. The `MAX_PAGES = 50` constant is declared but unused.

### 4. No scope check before payment delegation

**Spec says (§Consent scopes, lines 407-409):** `TrueLayerBankPlatform` checks scope before delegating — `paymentInitiation()` verifies PISP consent exists.
**Implementation:** `TrueLayerPaymentInitiation` calls `consentService.getUserToken(userId)` for consent check, but does not verify that PISP scope specifically was granted. `getUserToken()` returns any valid token regardless of scope.

## Code NOT in spec (extra code)

No significant extra code found. All implementation files correspond to spec requirements. The code review fixes (try-with-resources for JsonReader, error handling in auth callback, amount precision validation) are quality improvements not in the original spec but consistent with review findings.

## Summary

**4 conformance gaps found.**

- 2 are CDI wiring deviations (missing `@NamedOidcClient`, missing quarkus-oidc-client config)
- 1 is a missing pagination loop (protocol violation — MAX_PAGES unused)
- 1 is a missing scope check (consent scope not verified before payment operations)
