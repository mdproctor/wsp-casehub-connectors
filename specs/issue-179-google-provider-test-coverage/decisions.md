# Decisions — Issue #179: Google Provider Test Coverage

## D1: HTTP-level testing for all 4 modules including location-google

**Choice:** All 4 modules get HTTP-level testing via WireMock, including location-google which uses the Google Maps Java Client (GeoApiContext) instead of the standard Google API Client Library. location-google requires a production code change to add a test constructor accepting a pre-built `GeoApiContext` (matching the pattern email-google and document-google already have for their service objects).
**Alternatives:**
- Unit-level only for location-google — faster to write but doesn't test HTTP layer, leaves the weakest module untested at the integration level
- Skip location-google — defers the problem but leaves one module far behind
- Google's MockHttpTransport (for the 3 API Client Library modules) — lighter than WireMock but WireMock is the established pattern in this codebase
**Rationale:** Consistent test depth across all modules. GeoApiContext supports `baseUrlOverride()` for WireMock viability. Two HTTP stacks exist (Google API Client Library for email/document/contacts, Google Maps Java Client for location) requiring different WireMock stub patterns.
**Trade-offs:** More initial setup work for location-google's different HTTP stack. Production code change needed for test constructor.
**Sources:** GoogleLocationPlatform.java:contextFor() (GeoApiContext construction), GoogleEmailPlatform.java/GoogleDocumentPlatform.java (WireMock pattern reference), GeoApiContext.Builder.baseUrlOverride()
**Exploration:** quick
**Status:** revised (R1-03, R1-06, R1-10)

## D2: Tests + full error handling parity with TrueLayer

**Choice:** This issue includes both edge-case tests AND production error handling improvements to bring all 4 Google modules to TrueLayer-level error handling quality. Implementation follows TDD order: write tests against current behavior first, then update production code and tests together.
**Alternatives:**
- Tests only — keeps scope to M but tests exercise broken/missing error paths with no value
- Tests + targeted fixes — middle ground but arbitrary scope boundary
- Split into test-only + error-handling issues (R1-04) — reduces risk but the WireMock infrastructure and test scaffolding work is identical in both issues
**Rationale:** Tests need correct production code to be meaningful. The TDD implementation order (test current → fix production → update assertions) provides regression safety within the combined scope.
**Trade-offs:** Scope expands from M to L. Production code changes across 4 modules increase risk, mitigated by TDD implementation order.
**Sources:** TrueLayerClient.java (handleResponse() pattern), protocol paginating-client-fail-soft.md
**Exploration:** quick
**Status:** revised (R1-04)

## D3: Provider-agnostic exception types in connectors-api

**Choice:** Create provider-agnostic exception types in connectors-api (ProviderApiException, etc.) usable by any provider module. Not Google-specific. File a follow-up issue for migrating TrueLayer's existing exception types to the new hierarchy.
**Alternatives:**
- New google-common module — clean separation but adds a module for 2-3 classes
- Duplicate in each module — no cross-module dependency but code duplication
**Rationale:** connectors-api is the shared types module. Provider-agnostic naming lets TrueLayer and future providers converge on the same hierarchy. Avoids Google-specific types in a generic library.
**Trade-offs:** TrueLayer's existing exception types become candidates for migration (follow-up issue to be filed). Parallel hierarchies exist temporarily.
**Sources:** UnsupportedCapabilityException (existing connectors-api exception precedent), TrueLayerApiException/ConsentExpiredException/RateLimitedException (TrueLayer pattern)
**Exploration:** quick
**Status:** revised (R1-05)

## D4: Fix pagination protocol compliance (scoped to internal multi-page loops)

**Choice:** Fix pagination protocol compliance only for methods with internal multi-page loops: `GoogleDocumentPlatform.initialSync()`, and any other methods that issue multiple HTTP requests in a loop. Single-request methods with cursor-based pagination (caller-driven via page tokens) are not governed by the protocol.
**Alternatives:**
- Test current behavior only — documents violations but doesn't fix them
- Fix all paginating methods — over-scoped; protocol only applies to internal multi-page loops
**Rationale:** Protocol scope clause: "Any method that issues multiple HTTP requests to enumerate a paginated resource." Most Google methods issue one request per call with cursor-based pagination.
**Trade-offs:** Narrower than originally scoped — fewer production changes needed.
**Sources:** protocol paginating-client-fail-soft.md (PP-20260610-83747b), GoogleDocumentPlatform.initialSync(), GoogleCalendarPlatform.listEvents() (already compliant)
**Exploration:** quick
**Status:** revised (R1-08)

## D5: Capability-based test decomposition

**Choice:** Decompose tests by capability boundary for modules with inner capabilities; by concern for flat modules. New test classes cover gaps not already in existing test files. Existing tests stay.
- **location-google:** GooglePlaceSearchTest, GooglePlaceDetailsTest, GoogleGeocodingTest, GoogleDirectionsTest (4 new)
- **email-google:** GoogleEmailErrorHandlingTest, GoogleEmailEdgeCaseTest (2 new — existing file already has substantial WireMock coverage)
- **document-google:** GoogleFileOperationsTest, GoogleFolderOperationsTest, GoogleSearchSharingTest, GoogleDocumentSyncTest (4 new)
- **contacts-google:** GoogleContactReadTest, GoogleGroupReadTest, GoogleContactWriteTest (3 new)
- **connectors-api:** ProviderApiExceptionTest (1 new)
~14 new test files total.
**Alternatives:**
- Uniform 3-class decomposition (ErrorHandling, Pagination, EdgeCases) — doesn't match module capability structure
- Nested inner classes — fewer files but larger classes
**Rationale:** Capability-based decomposition aligns with the actual module structure. Each test class is self-contained and covers all concerns (errors, pagination, edge cases) for its capability.
**Trade-offs:** Non-uniform file counts across modules, but each reflects the module's actual complexity.
**Sources:** bank-truelayer test structure, GoogleLocationPlatform inner classes, GoogleDocumentPlatform inner classes, GoogleContactsPlatform inner classes
**Exploration:** quick
**Status:** revised (R1-07, R1-14)

## D6: Exception hierarchy — 3 provider-agnostic types with auth reason discriminator

**Choice:** Three exception types in connectors-api:
- `ProviderApiException(int statusCode, String responseBody, String provider)` — base for any non-2xx provider response
- `ProviderRateLimitedException(int retryAfterSeconds, String provider)` — 429 with retry guidance
- `ProviderAuthException(AuthFailureReason reason, String provider)` — with enum `AuthFailureReason { TOKEN_EXPIRED, INSUFFICIENT_SCOPE, FORBIDDEN }` to distinguish 401 (automatic token refresh) from 403 (user re-consent required)

Additionally: `SyncTokenExpiredException` updated to extend `ProviderApiException` instead of `RuntimeException`, for hierarchy consistency.

404 continues using `NoSuchElementException` (existing convention with 17+ call sites — migration out of scope).
**Alternatives:**
- 2 types (no auth exception) — callers can't catch auth failures for token refresh
- Flat ProviderAuthException without reason discriminator (original D6) — conflates 401 and 403 recovery paths
- ProviderNotFoundException for 404 — clean but 17+ site migration out of scope
**Rationale:** 401 and 403 have genuinely different recovery paths: 401 triggers automatic token refresh via refresh token; 403 requires user interaction for scope escalation. The `@RequiresScopes` annotation on all Google platforms confirms scope-related 403s are an expected runtime condition. A reason discriminator keeps the type hierarchy at 3 while enabling callers to branch on recovery action.
**Trade-offs:** Slightly more complex than flat ProviderAuthException. SyncTokenExpiredException parent class change could affect callers catching RuntimeException specifically (unlikely — all existing catch sites use the specific type).
**Depends on:** D3 (exception types live in connectors-api)
**Sources:** TrueLayerClient.handleResponse() (4-way status dispatch), Google OAuth 401 vs 403 semantics, @RequiresScopes annotations on Google platforms, SyncTokenExpiredException (connectors-api)
**Exploration:** quick
**Status:** revised (R1-01, R1-02)

## D7: Scope — exactly 4 modules, plain JUnit, existing tests stay

**Choice:** Scope is exactly these 4 modules: location-google, email-google, document-google, contacts-google. calendar-google has similar gaps but is not in this issue. Tests use plain JUnit with constructor-injected service objects (matching existing pattern in email-google and document-google). Existing test files are kept — new test classes cover only the gaps.
**Alternatives:**
- Include calendar-google — expands scope further
- Use @QuarkusTest — tests CDI wiring but slower, heavier, different code path
**Rationale:** Issue #179 explicitly names these 4 modules. Plain JUnit is faster, simpler, and matches the established test pattern. @QuarkusTest is appropriate for integration tests but not for unit-level error handling and edge case tests.
**Trade-offs:** CDI wiring not tested at unit level. calendar-google deferred.
**Sources:** Issue #179 body, GoogleEmailPlatformTest.java (plain JUnit + WireMock pattern)
**Exploration:** quick (implicit decision surfaced by R1-12, R1-13)
**Status:** captured
