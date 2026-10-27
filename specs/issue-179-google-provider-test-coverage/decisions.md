# Decisions — Issue #179: Google Provider Test Coverage

## D1: HTTP-level testing for all 4 modules including location-google

**Choice:** All 4 modules get HTTP-level testing via WireMock, including location-google which uses the Google Maps Java Client (GeoApiContext) instead of the standard Google API Client Library.
**Alternatives:**
- Unit-level only for location-google — faster to write but doesn't test HTTP layer, leaves the weakest module untested at the integration level
- Skip location-google — defers the problem but leaves one module far behind
**Rationale:** Consistent test depth across all modules. GeoApiContext supports custom base URL configuration, making WireMock viable.
**Trade-offs:** More initial setup work for location-google's different HTTP stack.
**Sources:** GoogleLocationPlatform.java (GeoApiContext usage), GoogleEmailPlatform.java/GoogleDocumentPlatform.java (WireMock pattern reference)
**Exploration:** quick
**Status:** captured

## D2: Tests + full error handling parity with TrueLayer

**Choice:** This issue includes both edge-case tests AND production error handling improvements to bring all 4 Google modules to TrueLayer-level error handling quality.
**Alternatives:**
- Tests only — keeps scope to M but tests exercise broken/missing error paths with no value
- Tests + targeted fixes — middle ground but arbitrary scope boundary
**Rationale:** Tests need correct production code to be meaningful. Writing tests against RuntimeException-wrapping error paths validates nothing useful.
**Trade-offs:** Scope expands from M to L. Production code changes across 4 modules increase risk.
**Sources:** TrueLayerClient.java (handleResponse() pattern), protocol paginating-client-fail-soft.md
**Exploration:** quick
**Status:** captured

## D3: Provider-agnostic exception types in connectors-api

**Choice:** Create provider-agnostic exception types in connectors-api (ProviderApiException, etc.) usable by any provider module. Not Google-specific.
**Alternatives:**
- New google-common module — clean separation but adds a module for 2-3 classes
- Duplicate in each module — no cross-module dependency but code duplication
**Rationale:** connectors-api is the shared types module. Provider-agnostic naming lets TrueLayer and future providers converge on the same hierarchy. Avoids Google-specific types in a generic library.
**Trade-offs:** TrueLayer's existing exception types become candidates for migration (not in this issue).
**Sources:** UnsupportedCapabilityException (existing connectors-api exception precedent), TrueLayerApiException/ConsentExpiredException/RateLimitedException (TrueLayer pattern)
**Exploration:** quick
**Status:** captured

## D4: Fix pagination protocol compliance

**Choice:** Align all 4 modules with the paginating-client-fail-soft protocol: return accumulated pages on mid-loop failure + WARNING log.
**Alternatives:**
- Test current behavior only — documents violations but doesn't fix them
**Rationale:** Natural extension of error handling work. Protocol violations are defects, not features.
**Trade-offs:** Additional production code changes beyond pure error handling.
**Sources:** protocol paginating-client-fail-soft.md (PP-20260610-83747b)
**Exploration:** quick
**Status:** captured

## D5: Decomposed test classes per concern

**Choice:** 3 new test classes per module (ErrorHandling, Pagination, EdgeCases) + 1 shared exception test in connectors-api. ~13 new test files total.
**Alternatives:**
- Nested inner classes — extend existing test files with @Nested groups. Fewer files but larger classes.
**Rationale:** Matches codebase reference standard (bank-truelayer has 11 focused test files). Clear concern boundaries, easy to run in isolation.
**Trade-offs:** More files, but each is small and focused.
**Sources:** bank-truelayer test structure (11 test files)
**Exploration:** quick
**Status:** captured

## D6: Exception hierarchy — 3 provider-agnostic types

**Choice:** Three exception types in connectors-api: `ProviderApiException` (base, with statusCode/body/provider), `ProviderRateLimitedException` (429 with retryAfterSeconds), `ProviderAuthException` (401/403 — expired token or insufficient scope). 404 continues using `NoSuchElementException` (existing convention). `SyncTokenExpiredException` already exists in connectors-api for 410 Gone — no change needed there.
**Alternatives:**
- 2 types (no auth exception) — simpler but callers can't catch auth failures specifically for token refresh
- 4 types (add sync token) — SyncTokenExpiredException already covers this
**Rationale:** Auth failures (401/403) are a distinct recovery action (re-authenticate / request more scopes) that callers need to distinguish from generic API errors. Three types cover the four recovery actions: retry-after (429), re-auth (401/403), not-found (404), and generic failure (everything else).
**Trade-offs:** Slightly more types than minimal, but each maps to a distinct caller recovery action.
**Depends on:** D3 (exception types live in connectors-api)
**Sources:** TrueLayerClient.handleResponse() (4-way status dispatch), NoSuchElementException convention (email-google, document-google), SyncTokenExpiredException (connectors-api)
**Exploration:** quick
**Status:** captured
