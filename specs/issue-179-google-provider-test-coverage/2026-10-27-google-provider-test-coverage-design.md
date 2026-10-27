# Design: Google Provider Test Coverage & Error Handling Parity

**Issue:** casehubio/connectors#179
**Scope:** location-google, email-google, document-google, contacts-google
**Scale:** L (4 modules, production error handling + tests)

## 1. Problem Statement

Four Google provider modules have minimal test coverage — each has a single test file covering basic mapping logic. Two modules (location-google, contacts-google) have zero API-level testing. None of the four handle 401, 403, or 429 errors. The production error handling is inconsistent: some methods throw `RuntimeException`, others silently return empty results.

### Current State

| Module | Test files | Mocking | API-level tests | Error coverage |
|--------|-----------|---------|-----------------|----------------|
| location-google | 1 (170 lines) | None | No | None |
| email-google | 1 (463 lines) | WireMock | Yes | Partial (404, 500) |
| document-google | 1 (424 lines) | WireMock | Yes | Partial (404) |
| contacts-google | 1 (115 lines) | None | No | None |

**Reference standard:** bank-truelayer — 11 test files, centralised `handleResponse()` mapping 401/404/429/500 to typed exceptions.

## 2. Provider-Agnostic Exception Hierarchy

New exception types in `connectors-api`, usable by any provider module.

### Types

```
ProviderApiException (extends RuntimeException)
├── statusCode: int
├── responseBody: String
└── provider: String

ProviderRateLimitedException (extends ProviderApiException)
└── retryAfterSeconds: int

ProviderAuthException (extends ProviderApiException)
└── reason: AuthFailureReason { TOKEN_EXPIRED, INSUFFICIENT_SCOPE, FORBIDDEN }
```

### HTTP Status Mapping

| Status | Exception | Recovery |
|--------|-----------|----------|
| 401 | `ProviderAuthException(TOKEN_EXPIRED)` | Automatic token refresh |
| 403 | `ProviderAuthException(INSUFFICIENT_SCOPE)` | User re-consent with broader scopes |
| 404 | `NoSuchElementException` | Existing convention (17+ call sites) |
| 410 | `SyncTokenExpiredException` | Full resync (updated to extend `ProviderApiException`) |
| 429 | `ProviderRateLimitedException(retryAfterSeconds)` | Retry after delay |
| 5xx | `ProviderApiException(statusCode, body, provider)` | Retry or fail |

### File Locations

```
connectors-api/src/main/java/io/casehub/connectors/
├── ProviderApiException.java
├── ProviderRateLimitedException.java
├── ProviderAuthException.java
└── AuthFailureReason.java
```

`SyncTokenExpiredException` already exists in connectors-api — updated to extend `ProviderApiException` instead of `RuntimeException`.

## 3. Production Error Handling Changes

Each Google module gets a centralised error handler method (following TrueLayer's `handleResponse()` pattern), adapted for the module's HTTP stack.

### Google API Client Library modules (email, document, contacts)

These modules catch `GoogleJsonResponseException` which carries `getStatusCode()` and `getDetails()`. Each module adds a private `handleGoogleError(GoogleJsonResponseException e)` method:

```java
private <T> T handleGoogleError(GoogleJsonResponseException e) {
    int code = e.getStatusCode();
    String body = e.getDetails() != null ? e.getDetails().toPrettyString() : e.getMessage();
    return switch (code) {
        case 401 -> throw new ProviderAuthException(AuthFailureReason.TOKEN_EXPIRED, id());
        case 403 -> throw new ProviderAuthException(AuthFailureReason.INSUFFICIENT_SCOPE, id());
        case 404 -> throw new NoSuchElementException(body);
        case 410 -> throw new SyncTokenExpiredException(body);
        case 429 -> {
            int retryAfter = parseRetryAfter(e);
            throw new ProviderRateLimitedException(retryAfter, id());
        }
        default -> throw new ProviderApiException(code, body, id());
    };
}
```

Current `catch (IOException e) { throw new RuntimeException(...) }` blocks are replaced with `catch (GoogleJsonResponseException e) { handleGoogleError(e); } catch (IOException e) { throw new ProviderApiException(0, e.getMessage(), id()); }`.

### Google Maps Java Client module (location)

location-google catches `ApiException | InterruptedException | IOException`. `ApiException` carries status information differently — via `ApiException.getStatus()` which returns `OverQueryLimitException`, `RequestDeniedException`, etc.

```java
private <T> T handleMapsError(ApiException e) {
    return switch (e) {
        case OverQueryLimitException _ -> throw new ProviderRateLimitedException(60, id());
        case RequestDeniedException _ -> throw new ProviderAuthException(AuthFailureReason.FORBIDDEN, id());
        default -> throw new ProviderApiException(0, e.getMessage(), id());
    };
}
```

### Production code prerequisite for location-google

`GoogleLocationPlatform.contextFor()` currently hard-codes `GeoApiContext` construction with no test injection point. Add a package-private constructor:

```java
GoogleLocationPlatform(GeoApiContext context) {
    this.testContext = context;
}
```

The `contextFor()` method returns `testContext` when non-null, falling back to credential-based construction. This matches the pattern email-google and document-google already use for their service objects.

## 4. Pagination Protocol Compliance

Only methods with internal multi-page loops are governed by the paginating-client-fail-soft protocol. Most Google methods issue a single request per call with cursor-based pagination (caller-driven via page tokens).

### Methods requiring fix

- `GoogleDocumentPlatform.GoogleFileOperations.initialSync()` — currently throws `RuntimeException` on mid-loop `IOException`. Fix: accumulate results, log WARNING, return partial `SyncResult`.

### Already compliant

- `GoogleCalendarPlatform.listEvents()` — returns partial on IOException, warns on MAX_PAGES
- `GoogleCalendarPlatform.listEventsSync()` — returns partial with warning
- `GoogleDocumentPlatform.GoogleFileOperations.incrementalSync()` — returns partial on mid-loop failure

### Not applicable (single-request, cursor-based)

- All `list()`, `search()`, `listContacts()` methods — caller drives pagination via page tokens; each call is one HTTP request.

## 5. Test Architecture

### Test decomposition

Tests decompose by capability boundary for modules with inner capabilities, by concern for flat modules. New test classes cover gaps not in existing test files (which are kept).

#### location-google (4 new test classes)

All require the new test constructor + WireMock with `GeoApiContext.Builder().baseUrlOverride()`.

| Class | Covers |
|-------|--------|
| `GooglePlaceSearchTest` | text/nearby/category search, pagination via nextPageToken, empty results, OverQueryLimitException, RequestDeniedException, malformed responses |
| `GooglePlaceDetailsTest` | get() success, 404/RuntimeException path, null optional fields, malformed detail response |
| `GoogleGeocodingTest` | geocode/reverse success, empty results, error responses, InterruptedException interrupt flag |
| `GoogleDirectionsTest` | route() success, no routes, travel mode mapping, error responses |

#### email-google (2 new test classes)

Existing `GoogleEmailPlatformTest` has substantial WireMock coverage. New classes cover gaps only.

| Class | Covers |
|-------|--------|
| `GoogleEmailErrorHandlingTest` | 401/403/429 responses, credential resolution failure, skip-on-error in listMessages (individual message fetch failure), buildDateQuery edge cases |
| `GoogleEmailEdgeCaseTest` | Nested multipart messages, null headers/payload/body in GmailMessageMapper, primaryLabel() mapping |

#### document-google (4 new test classes)

| Class | Covers |
|-------|--------|
| `GoogleFileOperationsTest` | download() success and 404, 401/403/429 on list/get/upload/delete, null metadata fields (parents, owners, timestamps) |
| `GoogleFolderOperationsTest` | move() success and 404, 401/403/429, create with parent folder |
| `GoogleSearchSharingTest` | search with quote escaping, 401/403/429, getShareLink 401/403/429 |
| `GoogleDocumentSyncTest` | initialSync mid-loop failure → partial results (protocol compliance), MAX_PAGES cap, 429 during sync, incremental sync with deletes |

#### contacts-google (3 new test classes)

All require WireMock setup (not present today — follows email/document pattern via `PeopleService.Builder().setRootUrl()`).

| Class | Covers |
|-------|--------|
| `GoogleContactReadTest` | list() pagination/empty, get() success/404, search() query/empty, listSync() initial/incremental/expired token, mapContacts filtering deleted, 401/403/429 |
| `GoogleGroupReadTest` | list() success/empty, listContacts() batch get/empty group, 401/403/429 |
| `GoogleContactWriteTest` | create() success, update() with etag, delete() success/404, mapToPerson reverse mapping, 401/403/429 |

#### connectors-api (1 new test class)

| Class | Covers |
|-------|--------|
| `ProviderApiExceptionTest` | Constructor, field accessors, inheritance hierarchy, SyncTokenExpiredException extends ProviderApiException |

### WireMock patterns

**Google API Client Library modules (email, document, contacts):**
```java
wireMockServer.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-1"))
    .willReturn(aResponse().withStatus(429)
        .withHeader("Retry-After", "30")
        .withHeader("Content-Type", "application/json")
        .withBody("{\"error\":{\"code\":429,\"message\":\"Rate Limit Exceeded\"}}")));
```

**Google Maps Java Client (location):**
```java
// Different path structure, API key as query param
wireMockServer.stubFor(get(urlPathEqualTo("/maps/api/place/textsearch/json"))
    .willReturn(aResponse().withStatus(200)
        .withHeader("Content-Type", "application/json")
        .withBody("{\"status\":\"OVER_QUERY_LIMIT\",\"error_message\":\"...\",\"results\":[]}")));
```

Note: Google Maps returns error status in the JSON body with HTTP 200, not via HTTP status codes. The Maps Java Client interprets the `status` field and throws the corresponding `ApiException` subtype.

### Test isolation

All tests use plain JUnit 5 with WireMock — no `@QuarkusTest`, no CDI container. Service objects are constructor-injected (matching existing pattern in email-google and document-google).

## 6. Follow-up Work

- **TrueLayer exception migration:** File a follow-up issue to migrate TrueLayer's existing exception types (`TrueLayerApiException`, `ConsentExpiredException`, `RateLimitedException`) to extend the new `ProviderApiException` hierarchy.
- **calendar-google test coverage:** Similar gaps exist but out of scope for this issue.
- **NoSuchElementException → ProviderNotFoundException:** 17+ call sites — separate issue if desired.

## 7. Deliverables Summary

| Area | Count |
|------|-------|
| New exception types (connectors-api) | 4 (3 new + 1 updated parent) |
| Production error handler methods | 4 (one per module) |
| Pagination protocol fixes | 1 (initialSync) |
| Production test constructors | 1 (location-google) |
| New test files | 14 |
| Modified existing test files | 0 |

## References

- `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerClient.java` — handleResponse() pattern
- `docs/protocols/connectors/paginating-client-fail-soft.md` (PP-20260610-83747b) — pagination fail-soft protocol
- `docs/protocols/connectors/shared-http-client.md` (PP-20260607-9794cb) — HttpHelper.CLIENT (not used by Google modules)
- `docs/protocols/connectors/credential-config-ownership.md` (PP-20260609-0c3e24) — credentials at call time
- `connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java` — existing sync exception
- `connectors-api/src/main/java/io/casehub/connectors/UnsupportedCapabilityException.java` — existing connectors-api exception precedent
- Issue #179 — original scope definition
- Decision review: `/Users/mdproctor/reviews/casehub-connectors/issue-179-decision-20261027-192031/`
