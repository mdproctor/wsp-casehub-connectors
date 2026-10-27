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
| 403 (insufficientPermissions) | `ProviderAuthException(INSUFFICIENT_SCOPE)` | User re-consent with broader scopes |
| 403 (other reasons) | `ProviderAuthException(FORBIDDEN)` | Not recoverable via re-auth |
| 404 | `NoSuchElementException` | Existing convention (17+ call sites) |
| 410 | `SyncTokenExpiredException` | Full resync (updated to extend `ProviderApiException`) |
| 429 | `ProviderRateLimitedException(retryAfterSeconds)` | Retry after delay |
| 5xx | `ProviderApiException(statusCode, body, provider)` | Retry or fail |

Note: 403 default is `FORBIDDEN`. Only when Google's error `reason` field is `"insufficientPermissions"` does it map to `INSUFFICIENT_SCOPE`. Google 403 reasons include `domainPolicy`, `forbidden`, `userRateLimitExceeded`, etc. — these are not scope issues and re-consent cannot fix them.

### SyncTokenExpiredException Constructor Change

`SyncTokenExpiredException` extends `ProviderApiException` instead of `RuntimeException`. Constructor updated to accept `(String expiredToken, String provider)`:

```java
public class SyncTokenExpiredException extends ProviderApiException {
    private final String expiredToken;

    public SyncTokenExpiredException(String expiredToken, String provider) {
        super(410, expiredToken, provider);
        this.expiredToken = expiredToken;
    }

    public String expiredToken() { return expiredToken; }
}
```

Existing call sites updated: `throw new SyncTokenExpiredException(request.syncToken(), id())`.

### File Locations

```
connectors-api/src/main/java/io/casehub/connectors/
├── ProviderApiException.java
├── ProviderRateLimitedException.java
├── ProviderAuthException.java
├── AuthFailureReason.java
└── SyncTokenExpiredException.java  (existing — re-parented + constructor updated)
```

## 3. Production Error Handling Changes

### Shared error mapping utility

A shared static utility in `connectors-api` centralises the status-code-to-exception mapping. Each module extracts status code and body from its specific exception type and delegates to the utility. This avoids duplicating the switch statement across 3 modules.

```java
public final class ProviderErrorMapping {

    private static final int DEFAULT_RETRY_AFTER_SECONDS = 30;

    public static RuntimeException mapStatusCode(int statusCode, String body,
            String provider, String errorReason) {
        return switch (statusCode) {
            case 401 -> new ProviderAuthException(AuthFailureReason.TOKEN_EXPIRED, provider);
            case 403 -> {
                var reason = "insufficientPermissions".equals(errorReason)
                    ? AuthFailureReason.INSUFFICIENT_SCOPE
                    : AuthFailureReason.FORBIDDEN;
                yield new ProviderAuthException(reason, provider);
            }
            case 404 -> new NoSuchElementException(body);
            case 410 -> new SyncTokenExpiredException(body, provider);
            case 429 -> new ProviderRateLimitedException(DEFAULT_RETRY_AFTER_SECONDS, provider);
            default -> new ProviderApiException(statusCode, body, provider);
        };
    }
}
```

Note: `GoogleJsonResponseException` does not expose the `Retry-After` HTTP header — the Google API Client Library's `HttpResponse` is not accessible from the exception. A default constant (`DEFAULT_RETRY_AFTER_SECONDS = 30`) is used for all 429 responses from Google APIs.

### Google API Client Library modules (email, document, contacts)

Each module has a thin private method that extracts fields from `GoogleJsonResponseException` and delegates to `ProviderErrorMapping`:

```java
private RuntimeException handleGoogleError(GoogleJsonResponseException e) {
    int code = e.getStatusCode();
    String body = e.getDetails() != null ? e.getDetails().toPrettyString() : e.getMessage();
    String reason = extractErrorReason(e);
    return ProviderErrorMapping.mapStatusCode(code, body, id(), reason);
}

private String extractErrorReason(GoogleJsonResponseException e) {
    if (e.getDetails() != null && e.getDetails().getErrors() != null
            && !e.getDetails().getErrors().isEmpty()) {
        return e.getDetails().getErrors().get(0).getReason();
    }
    return null;
}
```

Current catch blocks replaced with:
```java
try {
    // API call
} catch (GoogleJsonResponseException e) {
    throw handleGoogleError(e);
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

### Google Maps Java Client module (location)

location-google uses a different HTTP stack (`GeoApiContext` / `ApiException`). The error handler is separate from the shared utility because Maps errors are exception-subtype-based, not HTTP-status-based:

```java
private static final int MAPS_DEFAULT_RETRY_SECONDS = 60;

private RuntimeException handleMapsError(ApiException e) {
    return switch (e) {
        case OverQueryLimitException _ ->
            new ProviderRateLimitedException(MAPS_DEFAULT_RETRY_SECONDS, id());
        case RequestDeniedException _ ->
            new ProviderAuthException(AuthFailureReason.FORBIDDEN, id());
        default -> new ProviderApiException(0, e.getMessage(), id());
    };
}
```

Note: `MAPS_DEFAULT_RETRY_SECONDS = 60` is an approximation — Google Maps does not provide a `Retry-After` header in rate limit responses.

**Complete catch structure for location-google methods:**
```java
try {
    // Maps API call via .await()
} catch (ApiException e) {
    throw handleMapsError(e);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new ProviderApiException(0, "Interrupted", id());
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

InterruptedException restores the thread interrupt flag before throwing, preserving the existing behavior.

### Behavioral contract: fail-soft vs throw

**Methods that currently return empty on error stay fail-soft.** The error handler applies only to methods that currently throw. This preserves the existing behavioral contract for callers.

| Method | Current behavior | After change |
|--------|-----------------|--------------|
| `PlaceSearch.searchByText/Nearby/Category()` | Returns empty Page, logs WARNING | **No change** — stays fail-soft |
| `Geocoding.geocode/reverseGeocode()` | Returns empty List, logs WARNING | **No change** — stays fail-soft |
| `PlaceDetails.get()` | Throws RuntimeException | Throws `ProviderApiException` (typed) |
| `Directions.route()` | Throws RuntimeException | Throws `ProviderApiException` (typed) |
| email/document/contacts CRUD methods | Throw RuntimeException | Throw typed exceptions |
| email/document/contacts search/list methods | Throw RuntimeException | Throw typed exceptions (already throw, not fail-soft) |

### Production code prerequisite for location-google

`GoogleLocationPlatform.contextFor()` currently hard-codes `GeoApiContext` construction with no test injection point. Add a package-private constructor:

```java
GoogleLocationPlatform(GeoApiContext context) {
    this.overrideContext = context;
}
```

The `contextFor()` method returns `overrideContext` when non-null, falling back to credential-based construction. This matches the pattern email-google and document-google already use for their service objects.

## 4. Pagination Protocol Compliance

Only methods with internal multi-page loops are governed by the paginating-client-fail-soft protocol. Most Google methods issue a single request per call with cursor-based pagination (caller-driven via page tokens).

### Methods requiring fix

- `GoogleDocumentPlatform.GoogleFileOperations.initialSync()` — currently throws `RuntimeException` on mid-loop `IOException`. Fix: accumulate results, log WARNING, return partial `SyncResult` with `syncToken = null` (forces full resync on next attempt, avoiding duplicate items from partial initial sync).

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
| `GooglePlaceSearchTest` | text/nearby/category search, pagination via nextPageToken, empty results, OverQueryLimitException (stays fail-soft — returns empty + WARNING), RequestDeniedException, malformed responses |
| `GooglePlaceDetailsTest` | get() success, error responses → typed exceptions, null optional fields, malformed detail response |
| `GoogleGeocodingTest` | geocode/reverse success, empty results, error responses (stays fail-soft), InterruptedException interrupt flag preservation |
| `GoogleDirectionsTest` | route() success, no routes, travel mode mapping, error responses → typed exceptions |

#### email-google (2 new test classes)

Existing `GoogleEmailPlatformTest` has substantial WireMock coverage. New classes cover gaps only.

| Class | Covers |
|-------|--------|
| `GoogleEmailErrorHandlingTest` | 401/403/429 responses (403 with `insufficientPermissions` reason vs other reasons), credential resolution failure, skip-on-error in listMessages (individual message fetch failure), buildDateQuery edge cases |
| `GoogleEmailEdgeCaseTest` | Null headers/payload/body in GmailMessageMapper, primaryLabel() mapping. Nested multipart messages tested to document current limitation (mapper only processes one level — known gap, not fixed in this issue) |

#### document-google (4 new test classes)

| Class | Covers |
|-------|--------|
| `GoogleFileOperationsTest` | download() success and 404, 401/403/429 on list/get/upload/delete, null metadata fields (parents, owners, timestamps) |
| `GoogleFolderOperationsTest` | move() success and 404, 401/403/429, create with parent folder |
| `GoogleSearchSharingTest` | search with quote escaping, 401/403/429, getShareLink 401/403/429 |
| `GoogleDocumentSyncTest` | initialSync mid-loop failure → partial results with null token (protocol compliance), MAX_PAGES cap, 429 during sync, incremental sync with deletes |

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
| `ProviderApiExceptionTest` | Constructor, field accessors, inheritance hierarchy, SyncTokenExpiredException extends ProviderApiException, ProviderErrorMapping.mapStatusCode for all status codes including 403 reason discrimination |

### WireMock patterns

**Google API Client Library modules (email, document, contacts):**
```java
wireMockServer.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-1"))
    .willReturn(aResponse().withStatus(429)
        .withHeader("Content-Type", "application/json")
        .withBody("{\"error\":{\"code\":429,\"message\":\"Rate Limit Exceeded\"}}")));
```

**Google Maps Java Client (location):**
```java
wireMockServer.stubFor(get(urlPathEqualTo("/maps/api/place/textsearch/json"))
    .willReturn(aResponse().withStatus(200)
        .withHeader("Content-Type", "application/json")
        .withBody("{\"status\":\"OVER_QUERY_LIMIT\",\"error_message\":\"...\",\"results\":[]}")));
```

Note: Google Maps returns error status in the JSON body with HTTP 200, not via HTTP status codes. The Maps Java Client interprets the `status` field and throws the corresponding `ApiException` subtype.

### Test isolation

All tests use plain JUnit 5 with WireMock — no `@QuarkusTest`, no CDI container. Service objects are constructor-injected (matching existing pattern in email-google and document-google).

## 6. Follow-up Work

- **TrueLayer exception migration:** File a follow-up issue to migrate TrueLayer's existing exception types to extend the new hierarchy. Target state: `TrueLayerApiException extends ProviderApiException`, `ConsentExpiredException extends ProviderAuthException`, `RateLimitedException extends ProviderRateLimitedException`. TrueLayer-specific constructors preserved for backward compatibility; new constructors added that pass required fields (statusCode, provider) to super.
- **calendar-google test coverage:** Similar gaps exist but out of scope for this issue.
- **NoSuchElementException → ProviderNotFoundException:** 17+ call sites — separate issue if desired.

## 7. Deliverables Summary

| Area | Count |
|------|-------|
| New exception classes (connectors-api) | 3 (ProviderApiException, ProviderRateLimitedException, ProviderAuthException) |
| New enum (connectors-api) | 1 (AuthFailureReason) |
| New utility class (connectors-api) | 1 (ProviderErrorMapping) |
| Updated exception (connectors-api) | 1 (SyncTokenExpiredException re-parented) |
| Production error handler methods | 4 (1 shared utility + thin wrappers per module) |
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
- Spec review: `/Users/mdproctor/reviews/casehub-connectors/issue-179-spec-20261027-193850/`
