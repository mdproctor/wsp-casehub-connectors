# Google Provider Test Coverage & Error Handling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #179 — test: expand Google provider test coverage — edge cases for location, email, document, contacts
**Issue group:** #179

**Goal:** Bring 4 Google provider modules to TrueLayer-level error handling quality with typed exceptions and comprehensive edge-case test coverage.

**Architecture:** Create a provider-agnostic exception hierarchy in `connectors-api` (`ProviderApiException` base with rate-limit and auth subtypes), add a shared `ProviderErrorMapping` utility for status-code-to-exception mapping, then update each Google module's error handling and add 14 new test files organized by capability boundary. Two HTTP stacks: Google API Client Library (email, document, contacts) and Google Maps Java Client (location) — each requires different WireMock patterns.

**Tech Stack:** Java 21, JUnit 5, WireMock, AssertJ, Google API Client Library, Google Maps Services

## Global Constraints

- Java source level: 21 (running on JVM 26)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Module test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> -am`
- WireMock: `org.wiremock:wiremock-standalone` (version managed in parent POM)
- Exception naming: record-style accessors (no `get` prefix), matches `UnsupportedCapabilityException` pattern
- 404 convention: `NoSuchElementException` (existing, not changed)
- Fail-soft methods stay fail-soft; only currently-throwing methods get typed exceptions

---

## Batch 1: Exception Foundation

### Task 1: Provider-agnostic exception hierarchy + shared error mapping

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/AuthFailureReason.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/ProviderApiException.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/ProviderRateLimitedException.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/ProviderAuthException.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/ProviderErrorMapping.java`
- Modify: `connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java`
- Create: `connectors-api/src/test/java/io/casehub/connectors/ProviderExceptionTest.java`

**Interfaces:**
- Produces: `ProviderApiException(int statusCode, String responseBody, String provider)` — base exception
- Produces: `ProviderRateLimitedException(int retryAfterSeconds, String provider)` extends ProviderApiException
- Produces: `ProviderAuthException(AuthFailureReason reason, String provider)` extends ProviderApiException
- Produces: `AuthFailureReason` enum — `TOKEN_EXPIRED`, `INSUFFICIENT_SCOPE`, `FORBIDDEN`
- Produces: `ProviderErrorMapping.mapStatusCode(int statusCode, String body, String provider, String errorReason)` → RuntimeException
- Produces: `SyncTokenExpiredException(String expiredToken, String provider)` extends ProviderApiException

- [ ] **Step 1: Write the test class**

Create `connectors-api/src/test/java/io/casehub/connectors/ProviderExceptionTest.java`:

```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ProviderExceptionTest {

    @Test
    void providerApiException_carriesFields() {
        var ex = new ProviderApiException(500, "Internal Server Error", "google");
        assertThat(ex.statusCode()).isEqualTo(500);
        assertThat(ex.responseBody()).isEqualTo("Internal Server Error");
        assertThat(ex.provider()).isEqualTo("google");
        assertThat(ex).isInstanceOf(RuntimeException.class);
        assertThat(ex.getMessage()).contains("google").contains("500");
    }

    @Test
    void providerRateLimitedException_carriesRetryAfter() {
        var ex = new ProviderRateLimitedException(30, "google");
        assertThat(ex.retryAfterSeconds()).isEqualTo(30);
        assertThat(ex.statusCode()).isEqualTo(429);
        assertThat(ex.provider()).isEqualTo("google");
        assertThat(ex).isInstanceOf(ProviderApiException.class);
    }

    @Test
    void providerAuthException_tokenExpired() {
        var ex = new ProviderAuthException(AuthFailureReason.TOKEN_EXPIRED, "google");
        assertThat(ex.reason()).isEqualTo(AuthFailureReason.TOKEN_EXPIRED);
        assertThat(ex.statusCode()).isEqualTo(401);
        assertThat(ex.provider()).isEqualTo("google");
        assertThat(ex).isInstanceOf(ProviderApiException.class);
    }

    @Test
    void providerAuthException_insufficientScope() {
        var ex = new ProviderAuthException(AuthFailureReason.INSUFFICIENT_SCOPE, "google");
        assertThat(ex.reason()).isEqualTo(AuthFailureReason.INSUFFICIENT_SCOPE);
        assertThat(ex.statusCode()).isEqualTo(403);
    }

    @Test
    void providerAuthException_forbidden() {
        var ex = new ProviderAuthException(AuthFailureReason.FORBIDDEN, "google");
        assertThat(ex.reason()).isEqualTo(AuthFailureReason.FORBIDDEN);
        assertThat(ex.statusCode()).isEqualTo(403);
    }

    @Test
    void syncTokenExpiredException_extendsProviderApiException() {
        var ex = new SyncTokenExpiredException("token-123", "google");
        assertThat(ex).isInstanceOf(ProviderApiException.class);
        assertThat(ex.expiredToken()).isEqualTo("token-123");
        assertThat(ex.statusCode()).isEqualTo(410);
        assertThat(ex.provider()).isEqualTo("google");
    }

    @Test
    void errorMapping_401_tokenExpired() {
        var ex = ProviderErrorMapping.mapStatusCode(401, "Unauthorized", "google", null);
        assertThat(ex).isInstanceOf(ProviderAuthException.class);
        assertThat(((ProviderAuthException) ex).reason()).isEqualTo(AuthFailureReason.TOKEN_EXPIRED);
    }

    @Test
    void errorMapping_403_insufficientPermissions() {
        var ex = ProviderErrorMapping.mapStatusCode(403, "Forbidden", "google", "insufficientPermissions");
        assertThat(ex).isInstanceOf(ProviderAuthException.class);
        assertThat(((ProviderAuthException) ex).reason()).isEqualTo(AuthFailureReason.INSUFFICIENT_SCOPE);
    }

    @Test
    void errorMapping_403_otherReason_forbidden() {
        var ex = ProviderErrorMapping.mapStatusCode(403, "Forbidden", "google", "domainPolicy");
        assertThat(ex).isInstanceOf(ProviderAuthException.class);
        assertThat(((ProviderAuthException) ex).reason()).isEqualTo(AuthFailureReason.FORBIDDEN);
    }

    @Test
    void errorMapping_403_nullReason_forbidden() {
        var ex = ProviderErrorMapping.mapStatusCode(403, "Forbidden", "google", null);
        assertThat(ex).isInstanceOf(ProviderAuthException.class);
        assertThat(((ProviderAuthException) ex).reason()).isEqualTo(AuthFailureReason.FORBIDDEN);
    }

    @Test
    void errorMapping_404_noSuchElement() {
        var ex = ProviderErrorMapping.mapStatusCode(404, "Not Found", "google", null);
        assertThat(ex).isInstanceOf(java.util.NoSuchElementException.class);
    }

    @Test
    void errorMapping_410_syncTokenExpired() {
        var ex = ProviderErrorMapping.mapStatusCode(410, "expired-token", "google", null);
        assertThat(ex).isInstanceOf(SyncTokenExpiredException.class);
        assertThat(((SyncTokenExpiredException) ex).expiredToken()).isEqualTo("expired-token");
    }

    @Test
    void errorMapping_429_rateLimited() {
        var ex = ProviderErrorMapping.mapStatusCode(429, "Too Many Requests", "google", null);
        assertThat(ex).isInstanceOf(ProviderRateLimitedException.class);
        assertThat(((ProviderRateLimitedException) ex).retryAfterSeconds()).isEqualTo(30);
    }

    @Test
    void errorMapping_500_genericProviderError() {
        var ex = ProviderErrorMapping.mapStatusCode(500, "Server Error", "google", null);
        assertThat(ex).isInstanceOf(ProviderApiException.class);
        assertThat(((ProviderApiException) ex).statusCode()).isEqualTo(500);
    }
}
```

- [ ] **Step 2: Run test to verify it fails (compilation)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -am -Dtest=ProviderExceptionTest`
Expected: compilation failure — classes don't exist yet.

- [ ] **Step 3: Create AuthFailureReason enum**

Create `connectors-api/src/main/java/io/casehub/connectors/AuthFailureReason.java`:

```java
package io.casehub.connectors;

public enum AuthFailureReason {
    TOKEN_EXPIRED,
    INSUFFICIENT_SCOPE,
    FORBIDDEN
}
```

- [ ] **Step 4: Create ProviderApiException**

Create `connectors-api/src/main/java/io/casehub/connectors/ProviderApiException.java`:

```java
package io.casehub.connectors;

public class ProviderApiException extends RuntimeException {

    private final int statusCode;
    private final String responseBody;
    private final String provider;

    public ProviderApiException(int statusCode, String responseBody, String provider) {
        super(provider + " API error (HTTP " + statusCode + "): " + responseBody);
        this.statusCode = statusCode;
        this.responseBody = responseBody;
        this.provider = provider;
    }

    public int statusCode() { return statusCode; }
    public String responseBody() { return responseBody; }
    public String provider() { return provider; }
}
```

- [ ] **Step 5: Create ProviderRateLimitedException**

Create `connectors-api/src/main/java/io/casehub/connectors/ProviderRateLimitedException.java`:

```java
package io.casehub.connectors;

public class ProviderRateLimitedException extends ProviderApiException {

    private final int retryAfterSeconds;

    public ProviderRateLimitedException(int retryAfterSeconds, String provider) {
        super(429, "Rate limited — retry after " + retryAfterSeconds + "s", provider);
        this.retryAfterSeconds = retryAfterSeconds;
    }

    public int retryAfterSeconds() { return retryAfterSeconds; }
}
```

- [ ] **Step 6: Create ProviderAuthException**

Create `connectors-api/src/main/java/io/casehub/connectors/ProviderAuthException.java`:

```java
package io.casehub.connectors;

public class ProviderAuthException extends ProviderApiException {

    private final AuthFailureReason reason;

    public ProviderAuthException(AuthFailureReason reason, String provider) {
        super(reason == AuthFailureReason.TOKEN_EXPIRED ? 401 : 403,
              reason.name(), provider);
        this.reason = reason;
    }

    public AuthFailureReason reason() { return reason; }
}
```

- [ ] **Step 7: Update SyncTokenExpiredException to extend ProviderApiException**

Modify `connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java`:

```java
package io.casehub.connectors;

public class SyncTokenExpiredException extends ProviderApiException {

    private final String expiredToken;

    public SyncTokenExpiredException(String expiredToken, String provider) {
        super(410, expiredToken, provider);
        this.expiredToken = expiredToken;
    }

    public String expiredToken() { return expiredToken; }
}
```

- [ ] **Step 8: Create ProviderErrorMapping**

Create `connectors-api/src/main/java/io/casehub/connectors/ProviderErrorMapping.java`:

```java
package io.casehub.connectors;

import java.util.NoSuchElementException;

public final class ProviderErrorMapping {

    private static final int DEFAULT_RETRY_AFTER_SECONDS = 30;

    private ProviderErrorMapping() {}

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

- [ ] **Step 9: Update SyncTokenExpiredException call sites**

Three call sites need the new `provider` parameter added:

In `document-google/src/main/java/.../GoogleDocumentPlatform.java` (incrementalSync, ~line 293):
Change: `throw new SyncTokenExpiredException(request.syncToken())`
To: `throw new SyncTokenExpiredException(request.syncToken(), id())`

In `contacts-google/src/main/java/.../GoogleContactsPlatform.java` (listSync, ~line 153):
Change: `throw new SyncTokenExpiredException(request.syncToken())`
To: `throw new SyncTokenExpiredException(request.syncToken(), id())`

In `calendar-google/src/main/java/.../GoogleCalendarPlatform.java` (listEventsSync):
Change: `throw new SyncTokenExpiredException(request.syncToken())`
To: `throw new SyncTokenExpiredException(request.syncToken(), id())`

- [ ] **Step 10: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -am -Dtest=ProviderExceptionTest`
Expected: all 14 tests PASS.

- [ ] **Step 11: Run full build to verify nothing broken**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all existing tests still pass, SyncTokenExpiredException call sites compile with new constructor.

- [ ] **Step 12: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/AuthFailureReason.java \
       connectors-api/src/main/java/io/casehub/connectors/ProviderApiException.java \
       connectors-api/src/main/java/io/casehub/connectors/ProviderRateLimitedException.java \
       connectors-api/src/main/java/io/casehub/connectors/ProviderAuthException.java \
       connectors-api/src/main/java/io/casehub/connectors/ProviderErrorMapping.java \
       connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java \
       connectors-api/src/test/java/io/casehub/connectors/ProviderExceptionTest.java \
       document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java \
       contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleContactsPlatform.java \
       calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java
git commit -m "feat(#179): add provider-agnostic exception hierarchy and error mapping

Adds ProviderApiException, ProviderRateLimitedException, ProviderAuthException
with AuthFailureReason discriminator, and ProviderErrorMapping utility to
connectors-api. Re-parents SyncTokenExpiredException to extend
ProviderApiException. Refs #179"
```

---

## Batch 2: email-google Error Handling + Tests

### Task 2: email-google production error handling + GoogleEmailErrorHandlingTest

**Files:**
- Modify: `email-google/src/main/java/io/casehub/connectors/email/google/GoogleEmailPlatform.java`
- Create: `email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailErrorHandlingTest.java`

**Interfaces:**
- Consumes: `ProviderErrorMapping.mapStatusCode(int, String, String, String)` from Task 1
- Consumes: `ProviderApiException`, `ProviderAuthException`, `ProviderRateLimitedException`, `AuthFailureReason` from Task 1

- [ ] **Step 1: Write GoogleEmailErrorHandlingTest**

Create `email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailErrorHandlingTest.java`:

```java
package io.casehub.connectors.email.google;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import com.google.api.client.http.javanet.NetHttpTransport;
import com.google.api.client.json.gson.GsonFactory;
import com.google.api.services.gmail.Gmail;
import io.casehub.connectors.AuthFailureReason;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.ProviderApiException;
import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderRateLimitedException;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GoogleEmailErrorHandlingTest {

    private WireMockServer wireMock;
    private GoogleEmailPlatform platform;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        WireMock.configureFor("localhost", wireMock.port());

        Gmail gmailService = new Gmail.Builder(
                new NetHttpTransport(),
                GsonFactory.getDefaultInstance(),
                request -> {})
                .setApplicationName("test")
                .setRootUrl("http://localhost:" + wireMock.port() + "/")
                .build();

        platform = new GoogleEmailPlatform(gmailService);
    }

    @AfterEach
    void tearDown() {
        wireMock.stop();
    }

    @Test
    void listMailboxes_401_throwsProviderAuthTokenExpired() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/labels"))
                .willReturn(aResponse().withStatus(401)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":401,"message":"Invalid Credentials",
                                 "errors":[{"reason":"authError","message":"Invalid Credentials"}]}}""")));

        assertThatThrownBy(() -> platform.listMailboxes())
                .isInstanceOf(ProviderAuthException.class)
                .satisfies(ex -> {
                    var auth = (ProviderAuthException) ex;
                    assertThat(auth.reason()).isEqualTo(AuthFailureReason.TOKEN_EXPIRED);
                    assertThat(auth.provider()).isEqualTo("google");
                });
    }

    @Test
    void listMailboxes_403_insufficientPermissions_throwsInsufficientScope() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/labels"))
                .willReturn(aResponse().withStatus(403)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":403,"message":"Insufficient Permission",
                                 "errors":[{"reason":"insufficientPermissions"}]}}""")));

        assertThatThrownBy(() -> platform.listMailboxes())
                .isInstanceOf(ProviderAuthException.class)
                .satisfies(ex -> {
                    var auth = (ProviderAuthException) ex;
                    assertThat(auth.reason()).isEqualTo(AuthFailureReason.INSUFFICIENT_SCOPE);
                });
    }

    @Test
    void listMailboxes_403_domainPolicy_throwsForbidden() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/labels"))
                .willReturn(aResponse().withStatus(403)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":403,"message":"Domain Policy",
                                 "errors":[{"reason":"domainPolicy"}]}}""")));

        assertThatThrownBy(() -> platform.listMailboxes())
                .isInstanceOf(ProviderAuthException.class)
                .satisfies(ex -> {
                    var auth = (ProviderAuthException) ex;
                    assertThat(auth.reason()).isEqualTo(AuthFailureReason.FORBIDDEN);
                });
    }

    @Test
    void listMailboxes_429_throwsRateLimited() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/labels"))
                .willReturn(aResponse().withStatus(429)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":429,"message":"Rate Limit Exceeded"}}""")));

        assertThatThrownBy(() -> platform.listMailboxes())
                .isInstanceOf(ProviderRateLimitedException.class)
                .satisfies(ex -> {
                    var rl = (ProviderRateLimitedException) ex;
                    assertThat(rl.retryAfterSeconds()).isEqualTo(30);
                    assertThat(rl.provider()).isEqualTo("google");
                });
    }

    @Test
    void listMailboxes_500_throwsProviderApiException() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/labels"))
                .willReturn(aResponse().withStatus(500)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":500,"message":"Internal Server Error"}}""")));

        assertThatThrownBy(() -> platform.listMailboxes())
                .isInstanceOf(ProviderApiException.class)
                .satisfies(ex -> {
                    var api = (ProviderApiException) ex;
                    assertThat(api.statusCode()).isEqualTo(500);
                    assertThat(api.provider()).isEqualTo("google");
                });
    }

    @Test
    void getMessage_401_throwsProviderAuthTokenExpired() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-1"))
                .willReturn(aResponse().withStatus(401)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":401,"message":"Invalid Credentials",
                                 "errors":[{"reason":"authError"}]}}""")));

        assertThatThrownBy(() -> platform.getMessage("INBOX", "msg-1"))
                .isInstanceOf(ProviderAuthException.class);
    }

    @Test
    void getMessage_429_throwsRateLimited() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-1"))
                .willReturn(aResponse().withStatus(429)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":429,"message":"Rate Limit Exceeded"}}""")));

        assertThatThrownBy(() -> platform.getMessage("INBOX", "msg-1"))
                .isInstanceOf(ProviderRateLimitedException.class);
    }

    @Test
    void search_429_throwsRateLimited() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages"))
                .willReturn(aResponse().withStatus(429)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":429,"message":"Rate Limit Exceeded"}}""")));

        assertThatThrownBy(() -> platform.search("test query", PageRequest.first(10)))
                .isInstanceOf(ProviderRateLimitedException.class);
    }
}
```

Note: `assertThat` inside `satisfies` needs a static import from AssertJ — add `import static org.assertj.core.api.Assertions.assertThat;`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-google -am -Dtest=GoogleEmailErrorHandlingTest`
Expected: FAIL — current code throws `RuntimeException`, not `ProviderAuthException`/`ProviderRateLimitedException`.

- [ ] **Step 3: Add handleGoogleError to GoogleEmailPlatform**

Add two private methods to `GoogleEmailPlatform`:

```java
private RuntimeException handleGoogleError(com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    int code = e.getStatusCode();
    String body = e.getDetails() != null ? e.getDetails().toPrettyString() : e.getMessage();
    String reason = extractErrorReason(e);
    return ProviderErrorMapping.mapStatusCode(code, body, id(), reason);
}

private static String extractErrorReason(com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    if (e.getDetails() != null && e.getDetails().getErrors() != null
            && !e.getDetails().getErrors().isEmpty()) {
        return e.getDetails().getErrors().get(0).getReason();
    }
    return null;
}
```

Update all catch blocks that currently catch `IOException` and throw `RuntimeException`:

**listMailboxes (~line 82):** Split catch:
```java
} catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    throw handleGoogleError(e);
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

**listMessages outer (~line 122):** Same split pattern.

**getMessage (~line 135):** Replace existing `GoogleJsonResponseException` catch:
```java
} catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    throw handleGoogleError(e);
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

**getAttachmentContent (~line 155):** Same as getMessage.

**search outer (~line 200):** Same split pattern.

The skip-on-error in listMessages inner loop (~line 114) and search inner loop (~line 193) stay unchanged — they log and skip individual message failures.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-google -am -Dtest=GoogleEmailErrorHandlingTest`
Expected: all 8 tests PASS.

- [ ] **Step 5: Run existing tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-google -am`
Expected: all existing `GoogleEmailPlatformTest` tests still PASS.

- [ ] **Step 6: Commit**

```bash
git add email-google/src/main/java/io/casehub/connectors/email/google/GoogleEmailPlatform.java \
       email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailErrorHandlingTest.java
git commit -m "feat(#179): add typed error handling to email-google with tests

Replace RuntimeException catch blocks with ProviderErrorMapping dispatch.
Add GoogleEmailErrorHandlingTest covering 401/403/429/500 responses with
403 reason discrimination. Refs #179"
```

### Task 3: GoogleEmailEdgeCaseTest

**Files:**
- Create: `email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailEdgeCaseTest.java`

**Interfaces:**
- Consumes: `GoogleEmailPlatform(Gmail)` package-private constructor
- Consumes: `GmailMessageMapper` static methods

- [ ] **Step 1: Write GoogleEmailEdgeCaseTest**

Create `email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailEdgeCaseTest.java`:

```java
package io.casehub.connectors.email.google;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import com.google.api.client.http.javanet.NetHttpTransport;
import com.google.api.client.json.gson.GsonFactory;
import com.google.api.services.gmail.Gmail;
import io.casehub.connectors.PageRequest;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
import static org.assertj.core.api.Assertions.assertThat;

class GoogleEmailEdgeCaseTest {

    private WireMockServer wireMock;
    private GoogleEmailPlatform platform;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        WireMock.configureFor("localhost", wireMock.port());

        Gmail gmailService = new Gmail.Builder(
                new NetHttpTransport(),
                GsonFactory.getDefaultInstance(),
                request -> {})
                .setApplicationName("test")
                .setRootUrl("http://localhost:" + wireMock.port() + "/")
                .build();

        platform = new GoogleEmailPlatform(gmailService);
    }

    @AfterEach
    void tearDown() {
        wireMock.stop();
    }

    @Test
    void getMessage_nullHeaders_handledGracefully() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-1"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"id":"msg-1","threadId":"t1","labelIds":["INBOX"],
                                 "payload":{"mimeType":"text/plain",
                                            "body":{"data":"SGVsbG8="}}}""")));

        var message = platform.getMessage("INBOX", "msg-1");
        assertThat(message).isNotNull();
        assertThat(message.id()).isEqualTo("msg-1");
    }

    @Test
    void getMessage_emptyPayloadBody_handledGracefully() {
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-2"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"id":"msg-2","threadId":"t1","labelIds":["INBOX"],
                                 "payload":{"mimeType":"text/plain","headers":[
                                   {"name":"Subject","value":"Test"},
                                   {"name":"From","value":"sender@test.com"},
                                   {"name":"Date","value":"Mon, 1 Jan 2026 00:00:00 +0000"}
                                 ],"body":{}}}""")));

        var message = platform.getMessage("INBOX", "msg-2");
        assertThat(message).isNotNull();
    }

    @Test
    void listMessages_individualFetchFailure_skipsAndContinues() {
        // List returns 2 message IDs
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"messages":[{"id":"msg-ok"},{"id":"msg-fail"}]}""")));

        // First message succeeds
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-ok"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"id":"msg-ok","threadId":"t1","labelIds":["INBOX"],
                                 "snippet":"Hello","internalDate":"1704067200000",
                                 "payload":{"headers":[
                                   {"name":"Subject","value":"OK"},
                                   {"name":"From","value":"a@test.com"}]}}""")));

        // Second message fails with 500
        wireMock.stubFor(get(urlPathEqualTo("/gmail/v1/users/me/messages/msg-fail"))
                .willReturn(aResponse().withStatus(500)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":500,"message":"Server error"}}""")));

        var page = platform.listMessages("INBOX", null, null, PageRequest.first(10));
        // Should return the successful message and skip the failed one
        assertThat(page.items()).hasSize(1);
        assertThat(page.items().get(0).id()).isEqualTo("msg-ok");
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-google -am -Dtest=GoogleEmailEdgeCaseTest`
Expected: PASS — these test current behavior (null handling, skip-on-error).

- [ ] **Step 3: Commit**

```bash
git add email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailEdgeCaseTest.java
git commit -m "test(#179): add email-google edge case tests

Cover null headers, empty payload body, and individual message fetch
failure skip-on-error behavior. Documents nested multipart as known
gap (mapper only processes one level). Refs #179"
```

---

## Batch 3: document-google Error Handling + Tests

### Task 4: document-google production error handling + GoogleFileOperationsTest

**Files:**
- Modify: `document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java`
- Create: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleFileOperationsTest.java`

**Interfaces:**
- Consumes: `ProviderErrorMapping.mapStatusCode(int, String, String, String)` from Task 1
- Consumes: `GoogleDocumentPlatform(Drive)` package-private constructor (existing)

- [ ] **Step 1: Write GoogleFileOperationsTest**

Create `document-google/src/test/java/io/casehub/connectors/document/google/GoogleFileOperationsTest.java`:

```java
package io.casehub.connectors.document.google;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import com.google.api.client.http.javanet.NetHttpTransport;
import com.google.api.client.json.gson.GsonFactory;
import com.google.api.services.drive.Drive;
import io.casehub.connectors.AuthFailureReason;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.ProviderApiException;
import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderRateLimitedException;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.NoSuchElementException;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.delete;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GoogleFileOperationsTest {

    private WireMockServer wireMock;
    private GoogleDocumentPlatform platform;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        WireMock.configureFor("localhost", wireMock.port());

        Drive driveService = new Drive.Builder(
                new NetHttpTransport(),
                GsonFactory.getDefaultInstance(),
                request -> {})
                .setApplicationName("test")
                .setRootUrl("http://localhost:" + wireMock.port() + "/")
                .build();

        platform = new GoogleDocumentPlatform(driveService);
    }

    @AfterEach
    void tearDown() {
        wireMock.stop();
    }

    @Test
    void download_success_returnsBytes() {
        wireMock.stubFor(get(urlPathEqualTo("/drive/v3/files/file-1"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/octet-stream")
                        .withBody("file content bytes")));

        byte[] content = platform.files().download("file-1");
        assertThat(content).isNotNull();
    }

    @Test
    void download_404_throwsNoSuchElement() {
        wireMock.stubFor(get(urlPathEqualTo("/drive/v3/files/missing"))
                .willReturn(aResponse().withStatus(404)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":404,"message":"File not found",
                                 "errors":[{"reason":"notFound"}]}}""")));

        assertThatThrownBy(() -> platform.files().download("missing"))
                .isInstanceOf(NoSuchElementException.class);
    }

    @Test
    void list_401_throwsProviderAuthTokenExpired() {
        wireMock.stubFor(get(urlPathEqualTo("/drive/v3/files"))
                .willReturn(aResponse().withStatus(401)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":401,"message":"Invalid Credentials",
                                 "errors":[{"reason":"authError"}]}}""")));

        assertThatThrownBy(() -> platform.files().list("root", PageRequest.first(10)))
                .isInstanceOf(ProviderAuthException.class)
                .satisfies(ex -> assertThat(((ProviderAuthException) ex).reason())
                        .isEqualTo(AuthFailureReason.TOKEN_EXPIRED));
    }

    @Test
    void get_429_throwsRateLimited() {
        wireMock.stubFor(get(urlPathEqualTo("/drive/v3/files/file-1"))
                .willReturn(aResponse().withStatus(429)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":429,"message":"Rate Limit Exceeded"}}""")));

        assertThatThrownBy(() -> platform.files().get("file-1"))
                .isInstanceOf(ProviderRateLimitedException.class);
    }

    @Test
    void delete_403_insufficientPermissions_throwsInsufficientScope() {
        wireMock.stubFor(delete(urlPathEqualTo("/drive/v3/files/file-1"))
                .willReturn(aResponse().withStatus(403)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":403,"message":"Insufficient Permission",
                                 "errors":[{"reason":"insufficientPermissions"}]}}""")));

        assertThatThrownBy(() -> platform.files().delete("file-1"))
                .isInstanceOf(ProviderAuthException.class)
                .satisfies(ex -> assertThat(((ProviderAuthException) ex).reason())
                        .isEqualTo(AuthFailureReason.INSUFFICIENT_SCOPE));
    }

    @Test
    void get_nullOwner_handledGracefully() {
        wireMock.stubFor(get(urlPathEqualTo("/drive/v3/files/file-1"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"id":"file-1","name":"test.txt","mimeType":"text/plain",
                                 "size":"100","modifiedTime":"2026-01-01T00:00:00.000Z"}""")));

        var metadata = platform.files().get("file-1");
        assertThat(metadata).isNotNull();
        assertThat(metadata.id()).isEqualTo("file-1");
    }
}
```

- [ ] **Step 2: Run test — expect failures for error tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -am -Dtest=GoogleFileOperationsTest`
Expected: FAIL — error tests expect typed exceptions, production throws RuntimeException.

- [ ] **Step 3: Add handleGoogleError to GoogleDocumentPlatform**

Add the same two private methods (`handleGoogleError` and `extractErrorReason`) as email-google. Then update all catch blocks in GoogleDocumentPlatform to split `IOException` into `GoogleJsonResponseException` + `IOException`:

Update catch blocks in: `list()`, `get()`, `download()`, `upload()`, `delete()`, `list()` (folders), `create()` (folders), `move()`, `search()`, `getShareLink()`.

Pattern for each — replace:
```java
} catch (GoogleJsonResponseException e) {
    if (e.getStatusCode() == 404) throw new NoSuchElementException(...);
    throw new RuntimeException(...);
} catch (IOException e) {
    throw new RuntimeException(...);
}
```
With:
```java
} catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    throw handleGoogleError(e);
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

For methods that only catch `IOException` (no existing `GoogleJsonResponseException` handling):
```java
} catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    throw handleGoogleError(e);
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

- [ ] **Step 4: Run tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -am -Dtest=GoogleFileOperationsTest`
Expected: PASS.

- [ ] **Step 5: Run existing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -am`
Expected: all tests PASS (existing `GoogleDocumentPlatformTest` should still pass — 404 → `NoSuchElementException` convention unchanged).

- [ ] **Step 6: Commit**

```bash
git add document-google/
git commit -m "feat(#179): add typed error handling to document-google with file ops tests

Replace RuntimeException catch blocks with ProviderErrorMapping dispatch.
Add GoogleFileOperationsTest covering download success/404, 401/403/429
on list/get/delete, and null metadata handling. Refs #179"
```

### Task 5: GoogleFolderOperationsTest + GoogleSearchSharingTest + GoogleDocumentSyncTest + initialSync fix

**Files:**
- Modify: `document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java` (initialSync fix)
- Create: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleFolderOperationsTest.java`
- Create: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleSearchSharingTest.java`
- Create: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleDocumentSyncTest.java`

**Interfaces:**
- Consumes: `GoogleDocumentPlatform(Drive)` constructor, `ProviderAuthException`, `ProviderRateLimitedException`

- [ ] **Step 1: Write GoogleFolderOperationsTest**

Create `document-google/src/test/java/io/casehub/connectors/document/google/GoogleFolderOperationsTest.java` — test move() success and 404, 401/403/429 on list/create/move. Follow the same WireMock setUp pattern as GoogleFileOperationsTest. Key tests:

```java
@Test
void move_success_returnsMetadata() {
    // Stub PATCH /drive/v3/files/file-1 with addParents/removeParents
    // Assert returned metadata has correct id
}

@Test
void move_404_throwsNoSuchElement() {
    // Stub PATCH returning 404
    assertThatThrownBy(() -> platform.folders().move("file-1", "folder-2"))
            .isInstanceOf(NoSuchElementException.class);
}

@Test
void create_429_throwsRateLimited() {
    // Stub POST /drive/v3/files returning 429
    assertThatThrownBy(() -> platform.folders().create("root", "New Folder"))
            .isInstanceOf(ProviderRateLimitedException.class);
}
```

- [ ] **Step 2: Write GoogleSearchSharingTest**

Create `document-google/src/test/java/io/casehub/connectors/document/google/GoogleSearchSharingTest.java` — test search with quote escaping, 401/403/429 on search and getShareLink. Key tests:

```java
@Test
void search_queryWithQuotes_escapedCorrectly() {
    // Stub GET /drive/v3/files with query containing escaped quotes
    // Assert results returned
}

@Test
void getShareLink_401_throwsProviderAuth() {
    // Stub POST /drive/v3/files/file-1/permissions returning 401
    assertThatThrownBy(() -> platform.sharing().getShareLink("file-1"))
            .isInstanceOf(ProviderAuthException.class);
}
```

- [ ] **Step 3: Write GoogleDocumentSyncTest with initialSync partial result test**

Create `document-google/src/test/java/io/casehub/connectors/document/google/GoogleDocumentSyncTest.java`:

```java
@Test
void initialSync_midLoopFailure_returnsPartialResultsWithNullToken() {
    // Stub getStartPageToken
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes/startPageToken"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"startPageToken":"token-1"}""")));

    // First page succeeds
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes"))
            .withQueryParam("pageToken", equalTo("token-1"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"changes":[{"fileId":"f1","file":{"id":"f1","name":"a.txt",
                             "mimeType":"text/plain","modifiedTime":"2026-01-01T00:00:00.000Z"}}],
                             "nextPageToken":"token-2","newStartPageToken":null}""")));

    // Second page fails with 500
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes"))
            .withQueryParam("pageToken", equalTo("token-2"))
            .willReturn(aResponse().withStatus(500)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"error":{"code":500,"message":"Internal error"}}""")));

    var result = platform.files().listSync(SyncRequest.initial());
    // Partial result: page 1 items returned, null syncToken forces full resync
    assertThat(result.items()).hasSize(1);
    assertThat(result.syncToken()).isNull();
}

@Test
void incrementalSync_expiredToken_throwsSyncTokenExpired() {
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes"))
            .willReturn(aResponse().withStatus(404)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"error":{"code":404,"message":"Page token expired"}}""")));

    assertThatThrownBy(() -> platform.files().listSync(SyncRequest.withToken("old-token")))
            .isInstanceOf(SyncTokenExpiredException.class);
}
```

- [ ] **Step 4: Run sync test — expect failure for initialSync partial result**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -am -Dtest=GoogleDocumentSyncTest#initialSync_midLoopFailure_returnsPartialResultsWithNullToken`
Expected: FAIL — current initialSync throws RuntimeException instead of returning partial.

- [ ] **Step 5: Fix initialSync to return partial results**

In `GoogleDocumentPlatform.java`, modify `initialSync()` (~line 221–253):

Replace the catch block that throws RuntimeException with partial result handling:

```java
private SyncResult<DocumentSummary> initialSync(SyncRequest request) throws IOException {
    var startToken = driveService.changes().getStartPageToken().execute();
    String pageToken = startToken.getStartPageToken();
    var accumulated = new java.util.ArrayList<DocumentSummary>();
    String finalToken = null;

    while (pageToken != null) {
        try {
            var response = driveService.changes().list(pageToken)
                    .setFields("changes(fileId,file(id,name,mimeType,size,modifiedTime,parents,owners,webViewLink)),nextPageToken,newStartPageToken")
                    .setPageSize(100)
                    .execute();

            if (response.getChanges() != null) {
                for (var change : response.getChanges()) {
                    if (change.getFile() != null) {
                        accumulated.add(toSummary(change.getFile()));
                    }
                }
            }
            finalToken = response.getNewStartPageToken();
            pageToken = response.getNextPageToken();
        } catch (IOException e) {
            LOG.warnf("Google Drive initial sync failed mid-loop after %d items: %s",
                    accumulated.size(), e.getMessage());
            return new SyncResult<>(List.copyOf(accumulated), List.of(), null, false);
        }
    }
    return new SyncResult<>(List.copyOf(accumulated), List.of(), finalToken, false);
}
```

Note: `syncToken = null` on partial failure forces full resync on next call.

- [ ] **Step 6: Run all document-google tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -am`
Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add document-google/
git commit -m "feat(#179): add folder/search/sync tests + fix initialSync protocol compliance

Add GoogleFolderOperationsTest, GoogleSearchSharingTest, GoogleDocumentSyncTest.
Fix initialSync to return partial results with null syncToken on mid-loop failure
per paginating-client-fail-soft protocol. Refs #179"
```

---

## Batch 4: contacts-google WireMock + Error Handling + Tests

### Task 6: contacts-google test constructor + WireMock + error handling + GoogleContactReadTest

**Files:**
- Modify: `contacts-google/pom.xml` (add WireMock dependency)
- Modify: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleContactsPlatform.java`
- Create: `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactReadTest.java`

**Interfaces:**
- Consumes: `ProviderErrorMapping.mapStatusCode(int, String, String, String)` from Task 1

- [ ] **Step 1: Add WireMock dependency to contacts-google/pom.xml**

Add to `<dependencies>`:
```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Add test constructor to GoogleContactsPlatform**

Add a package-private constructor that accepts a `PeopleService`:

```java
private PeopleService overrideService;

GoogleContactsPlatform(com.google.api.services.people.v1.PeopleService peopleService) {
    this((ServiceConnectionProvider) null);
    this.overrideService = peopleService;
}
```

Modify `buildService(String actorId)` to return override when present:
```java
private PeopleService buildService(String actorId) {
    if (overrideService != null) return overrideService;
    // existing implementation...
}
```

- [ ] **Step 3: Add handleGoogleError to GoogleContactsPlatform**

Add the same `handleGoogleError` and `extractErrorReason` private methods. Update all catch blocks:

For methods that catch `IOException` and throw `RuntimeException` (get, create, update, delete):
```java
} catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    throw handleGoogleError(e);
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

For methods that catch `IOException` and return empty (list, search, groupRead.list, groupRead.listContacts) — **keep fail-soft**:
```java
} catch (IOException e) {
    LOG.warnf("...", e.getMessage());
    return new Page<>(List.of(), null, false);  // unchanged
}
```

For listSync that already catches `GoogleJsonResponseException`:
```java
} catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
    throw handleGoogleError(e);  // 410 → SyncTokenExpiredException via ProviderErrorMapping
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

- [ ] **Step 4: Write GoogleContactReadTest**

Create `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactReadTest.java`:

```java
package io.casehub.connectors.contacts.google;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import com.google.api.client.http.javanet.NetHttpTransport;
import com.google.api.client.json.gson.GsonFactory;
import com.google.api.services.people.v1.PeopleService;
import io.casehub.connectors.AuthFailureReason;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderRateLimitedException;
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncTokenExpiredException;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.NoSuchElementException;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GoogleContactReadTest {

    private WireMockServer wireMock;
    private GoogleContactsPlatform platform;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        WireMock.configureFor("localhost", wireMock.port());

        PeopleService peopleService = new PeopleService.Builder(
                new NetHttpTransport(),
                GsonFactory.getDefaultInstance(),
                request -> {})
                .setApplicationName("test")
                .setRootUrl("http://localhost:" + wireMock.port() + "/")
                .build();

        platform = new GoogleContactsPlatform(peopleService);
    }

    @AfterEach
    void tearDown() {
        wireMock.stop();
    }

    @Test
    void list_success_returnsPaginatedContacts() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/me/connections"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"connections":[
                                  {"resourceName":"people/1","names":[{"displayName":"Alice"}],
                                   "emailAddresses":[{"value":"alice@test.com"}]}
                                ],"totalPeople":1,"nextPageToken":"page2"}""")));

        var page = platform.contactRead("user-1").list(PageRequest.first(10));
        assertThat(page.items()).hasSize(1);
        assertThat(page.items().get(0).displayName()).isEqualTo("Alice");
        assertThat(page.nextCursor()).isEqualTo("page2");
    }

    @Test
    void list_emptyResult_returnsEmptyPage() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/me/connections"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"totalPeople":0}""")));

        var page = platform.contactRead("user-1").list(PageRequest.first(10));
        assertThat(page.items()).isEmpty();
    }

    @Test
    void get_success_returnsContact() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/1"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"resourceName":"people/1","names":[{"displayName":"Alice"}],
                                 "emailAddresses":[{"value":"alice@test.com"}]}""")));

        var contact = platform.contactRead("user-1").get("people/1");
        assertThat(contact.displayName()).isEqualTo("Alice");
    }

    @Test
    void get_404_throwsNoSuchElement() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/missing"))
                .willReturn(aResponse().withStatus(404)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":404,"message":"Person not found",
                                 "errors":[{"reason":"notFound"}]}}""")));

        assertThatThrownBy(() -> platform.contactRead("user-1").get("people/missing"))
                .isInstanceOf(NoSuchElementException.class);
    }

    @Test
    void get_401_throwsProviderAuth() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/1"))
                .willReturn(aResponse().withStatus(401)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":401,"message":"Invalid credentials",
                                 "errors":[{"reason":"authError"}]}}""")));

        assertThatThrownBy(() -> platform.contactRead("user-1").get("people/1"))
                .isInstanceOf(ProviderAuthException.class)
                .satisfies(ex -> assertThat(((ProviderAuthException) ex).reason())
                        .isEqualTo(AuthFailureReason.TOKEN_EXPIRED));
    }

    @Test
    void get_429_throwsRateLimited() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/1"))
                .willReturn(aResponse().withStatus(429)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":429,"message":"Quota exceeded"}}""")));

        assertThatThrownBy(() -> platform.contactRead("user-1").get("people/1"))
                .isInstanceOf(ProviderRateLimitedException.class);
    }

    @Test
    void listSync_expiredToken_throwsSyncTokenExpired() {
        wireMock.stubFor(get(urlPathEqualTo("/v1/people/me/connections"))
                .willReturn(aResponse().withStatus(410)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"error":{"code":410,"message":"Sync token expired"}}""")));

        assertThatThrownBy(() -> platform.contactRead("user-1").listSync(SyncRequest.withToken("old")))
                .isInstanceOf(SyncTokenExpiredException.class);
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-google -am -Dtest=GoogleContactReadTest`
Expected: PASS.

- [ ] **Step 6: Run all contacts-google tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-google -am`
Expected: all tests PASS (existing `GoogleContactsPlatformTest` still passes).

- [ ] **Step 7: Commit**

```bash
git add contacts-google/
git commit -m "feat(#179): add WireMock setup, typed errors, and contact read tests

Add test constructor accepting PeopleService, WireMock dependency.
Replace RuntimeException with ProviderErrorMapping dispatch in CRUD methods.
Add GoogleContactReadTest covering list/get/search/sync with 401/403/429/410.
Refs #179"
```

### Task 7: GoogleGroupReadTest + GoogleContactWriteTest

**Files:**
- Create: `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleGroupReadTest.java`
- Create: `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactWriteTest.java`

**Interfaces:**
- Consumes: `GoogleContactsPlatform(PeopleService)` constructor from Task 6

- [ ] **Step 1: Write GoogleGroupReadTest**

Create `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleGroupReadTest.java`:

Follow the same WireMock setUp pattern. Key tests:
```java
@Test
void list_success_returnsGroups() {
    // Stub GET /v1/contactGroups
    // Assert groups returned with names and types
}

@Test
void list_emptyResult_returnsEmptyList() {
    // Stub empty contactGroups response
}

@Test
void listContacts_success_returnsBatchedContacts() {
    // Stub GET /v1/people:batchGet
    // Assert contacts in group returned
}

@Test
void listContacts_emptyGroup_returnsEmptyPage() {
    // Stub empty batchGet response
}

@Test
void list_429_remainsFailSoft() {
    // Stub 429 — group list is fail-soft, should return empty not throw
}
```

- [ ] **Step 2: Write GoogleContactWriteTest**

Create `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactWriteTest.java`:

Key tests:
```java
@Test
void create_success_returnsContactId() {
    // Stub POST /v1/people:createContact
    // Assert returned contact has resourceName
}

@Test
void update_success_withEtag() {
    // Stub GET /v1/people/1 (to get etag), then PATCH /v1/people/1:updateContact
}

@Test
void delete_success() {
    // Stub DELETE /v1/people/1:deleteContact returning 200
}

@Test
void delete_404_throwsNoSuchElement() {
    // Stub DELETE returning 404
    assertThatThrownBy(() -> platform.contactWrite("user-1").delete("people/missing"))
            .isInstanceOf(NoSuchElementException.class);
}

@Test
void create_401_throwsProviderAuth() {
    // Stub POST returning 401
    assertThatThrownBy(() -> platform.contactWrite("user-1").create(testContact()))
            .isInstanceOf(ProviderAuthException.class);
}
```

- [ ] **Step 3: Run all contacts-google tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-google -am`
Expected: all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleGroupReadTest.java \
       contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactWriteTest.java
git commit -m "test(#179): add group read and contact write tests for contacts-google

Cover list groups, list contacts in group, create/update/delete
with etag handling, error responses. Refs #179"
```

---

## Batch 5: location-google WireMock + Error Handling + Tests

### Task 8: location-google test constructor + WireMock + error handling + GooglePlaceSearchTest

**Files:**
- Modify: `location-google/pom.xml` (add WireMock dependency)
- Modify: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleLocationPlatform.java`
- Create: `location-google/src/test/java/io/casehub/connectors/location/google/GooglePlaceSearchTest.java`

**Interfaces:**
- Consumes: `ProviderAuthException`, `ProviderRateLimitedException` from Task 1
- Produces: `GoogleLocationPlatform(GeoApiContext)` package-private constructor

- [ ] **Step 1: Add WireMock dependency to location-google/pom.xml**

Add to `<dependencies>`:
```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Add test constructor to GoogleLocationPlatform**

Add a package-private constructor accepting `GeoApiContext`:

```java
private GeoApiContext overrideContext;

GoogleLocationPlatform(GeoApiContext context) {
    this((StaticCredentialStore) null);
    this.overrideContext = context;
}
```

Modify `contextFor(String userId)` to return override when present:
```java
private GeoApiContext contextFor(String userId) {
    if (overrideContext != null) return overrideContext;
    // existing implementation...
}
```

- [ ] **Step 3: Add handleMapsError to GoogleLocationPlatform**

Add error handling methods:
```java
private static final int MAPS_DEFAULT_RETRY_SECONDS = 60;

private RuntimeException handleMapsError(ApiException e) {
    return switch (e) {
        case com.google.maps.errors.OverQueryLimitException _ ->
            new ProviderRateLimitedException(MAPS_DEFAULT_RETRY_SECONDS, id());
        case com.google.maps.errors.RequestDeniedException _ ->
            new ProviderAuthException(AuthFailureReason.FORBIDDEN, id());
        default -> new ProviderApiException(0, e.getMessage(), id());
    };
}
```

Update catch blocks in `PlaceDetails.get()` and `Directions.route()` (the methods that currently throw):
```java
} catch (ApiException e) {
    throw handleMapsError(e);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new ProviderApiException(0, "Interrupted", id());
} catch (IOException e) {
    throw new ProviderApiException(0, e.getMessage(), id());
}
```

Search and geocoding methods **stay fail-soft** — keep their existing catch blocks that log WARNING and return empty. Do NOT apply handleMapsError to these.

- [ ] **Step 4: Write GooglePlaceSearchTest**

Create `location-google/src/test/java/io/casehub/connectors/location/google/GooglePlaceSearchTest.java`:

Note: Google Maps Java Client uses a different WireMock pattern — errors come as JSON body `status` field with HTTP 200, not as HTTP status codes.

```java
package io.casehub.connectors.location.google;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import com.google.maps.GeoApiContext;
import io.casehub.connectors.PageRequest;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.urlPathEqualTo;
import static org.assertj.core.api.Assertions.assertThat;

class GooglePlaceSearchTest {

    private WireMockServer wireMock;
    private GoogleLocationPlatform platform;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        WireMock.configureFor("localhost", wireMock.port());

        GeoApiContext geoContext = new GeoApiContext.Builder()
                .apiKey("test-key")
                .baseUrlOverride("http://localhost:" + wireMock.port())
                .build();

        platform = new GoogleLocationPlatform(geoContext);
    }

    @AfterEach
    void tearDown() {
        wireMock.stop();
    }

    @Test
    void searchByText_success_returnsPlaces() {
        wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/textsearch/json"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"status":"OK","results":[
                                  {"place_id":"p1","name":"Coffee Shop",
                                   "formatted_address":"123 Main St",
                                   "geometry":{"location":{"lat":51.5,"lng":-0.1}},
                                   "types":["cafe"],"rating":4.5}
                                ]}""")));

        var page = platform.placeSearch("user-1")
                .searchByText("coffee", PageRequest.first(10));
        assertThat(page.items()).hasSize(1);
        assertThat(page.items().get(0).name()).isEqualTo("Coffee Shop");
    }

    @Test
    void searchByText_emptyResults() {
        wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/textsearch/json"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"status":"ZERO_RESULTS","results":[]}""")));

        var page = platform.placeSearch("user-1")
                .searchByText("nonexistent", PageRequest.first(10));
        assertThat(page.items()).isEmpty();
    }

    @Test
    void searchByText_overQueryLimit_returnsEmptyFailSoft() {
        wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/textsearch/json"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"status":"OVER_QUERY_LIMIT",
                                 "error_message":"You have exceeded your rate-limit",
                                 "results":[]}""")));

        // Search is fail-soft — returns empty page, does not throw
        var page = platform.placeSearch("user-1")
                .searchByText("coffee", PageRequest.first(10));
        assertThat(page.items()).isEmpty();
    }

    @Test
    void searchByText_requestDenied_returnsEmptyFailSoft() {
        wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/textsearch/json"))
                .willReturn(aResponse().withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {"status":"REQUEST_DENIED",
                                 "error_message":"API key invalid",
                                 "results":[]}""")));

        var page = platform.placeSearch("user-1")
                .searchByText("coffee", PageRequest.first(10));
        assertThat(page.items()).isEmpty();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl location-google -am -Dtest=GooglePlaceSearchTest`
Expected: PASS — search tests verify current fail-soft behavior.

- [ ] **Step 6: Run all location-google tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl location-google -am`
Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add location-google/
git commit -m "feat(#179): add WireMock setup, typed errors, and place search tests

Add test constructor accepting GeoApiContext, WireMock dependency.
Add handleMapsError for PlaceDetails.get and Directions.route (throwing methods).
Search methods stay fail-soft. GooglePlaceSearchTest covers text search,
empty results, and rate-limit/denied fail-soft behavior. Refs #179"
```

### Task 9: GooglePlaceDetailsTest + GoogleGeocodingTest + GoogleDirectionsTest

**Files:**
- Create: `location-google/src/test/java/io/casehub/connectors/location/google/GooglePlaceDetailsTest.java`
- Create: `location-google/src/test/java/io/casehub/connectors/location/google/GoogleGeocodingTest.java`
- Create: `location-google/src/test/java/io/casehub/connectors/location/google/GoogleDirectionsTest.java`

**Interfaces:**
- Consumes: `GoogleLocationPlatform(GeoApiContext)` constructor from Task 8

- [ ] **Step 1: Write GooglePlaceDetailsTest**

Follow the same WireMock setUp pattern. Key tests:

```java
@Test
void get_success_returnsPlaceDetail() {
    wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/details/json"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"status":"OK","result":{
                              "place_id":"p1","name":"Coffee Shop",
                              "formatted_address":"123 Main St",
                              "geometry":{"location":{"lat":51.5,"lng":-0.1}},
                              "formatted_phone_number":"+44 20 1234 5678",
                              "opening_hours":{"open_now":true},
                              "reviews":[{"author_name":"Alice","rating":5,"text":"Great!"}],
                              "price_level":2}}""")));

    var detail = platform.placeDetails("user-1").get("p1");
    assertThat(detail.name()).isEqualTo("Coffee Shop");
    assertThat(detail.phoneNumber()).isEqualTo("+44 20 1234 5678");
}

@Test
void get_notFound_throwsProviderApiException() {
    wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/details/json"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"status":"NOT_FOUND","result":{}}""")));

    assertThatThrownBy(() -> platform.placeDetails("user-1").get("missing"))
            .isInstanceOf(ProviderApiException.class);
}

@Test
void get_requestDenied_throwsProviderAuth() {
    wireMock.stubFor(get(urlPathEqualTo("/maps/api/place/details/json"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"status":"REQUEST_DENIED","error_message":"API key invalid"}""")));

    assertThatThrownBy(() -> platform.placeDetails("user-1").get("p1"))
            .isInstanceOf(ProviderAuthException.class);
}
```

- [ ] **Step 2: Write GoogleGeocodingTest**

Key tests:
```java
@Test
void geocode_success_returnsResults() {
    // Stub /maps/api/geocode/json with OK status + results
}

@Test
void geocode_zeroResults_returnsEmptyList() {
    // Stub ZERO_RESULTS — fail-soft returns empty
}

@Test
void geocode_overQueryLimit_returnsEmptyFailSoft() {
    // OVER_QUERY_LIMIT — fail-soft, returns empty, does not throw
}

@Test
void reverseGeocode_success_returnsResults() {
    // Stub with latlng query param
}
```

- [ ] **Step 3: Write GoogleDirectionsTest**

Key tests:
```java
@Test
void route_success_returnsRoute() {
    // Stub /maps/api/directions/json with routes
}

@Test
void route_noRoutes_throwsProviderApiException() {
    // Stub with empty routes array — current code throws RuntimeException
}

@Test
void route_overQueryLimit_throwsRateLimited() {
    // OVER_QUERY_LIMIT on route() throws (not fail-soft)
    assertThatThrownBy(() -> platform.directions("user-1").route(...))
            .isInstanceOf(ProviderRateLimitedException.class);
}
```

- [ ] **Step 4: Run all location-google tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl location-google -am`
Expected: all tests PASS.

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and test.

- [ ] **Step 6: Commit**

```bash
git add location-google/src/test/java/io/casehub/connectors/location/google/GooglePlaceDetailsTest.java \
       location-google/src/test/java/io/casehub/connectors/location/google/GoogleGeocodingTest.java \
       location-google/src/test/java/io/casehub/connectors/location/google/GoogleDirectionsTest.java
git commit -m "test(#179): add place details, geocoding, and directions tests

Cover PlaceDetails.get with success/notFound/requestDenied → typed exceptions.
Geocoding tests verify fail-soft behavior on OVER_QUERY_LIMIT.
Directions tests verify route success and typed errors on failure. Refs #179"
```

---

## Batch 6: Follow-up Issue + Docs

### Task 10: File TrueLayer migration follow-up issue + update CLAUDE.md

**Files:**
- No code changes — GitHub issue creation + doc update

- [ ] **Step 1: File TrueLayer migration follow-up issue**

```bash
gh issue create --repo casehubio/connectors \
    --title "refactor: migrate TrueLayer exceptions to extend ProviderApiException hierarchy" \
    --body "$(cat <<'EOF'
## Parent
Follows from #179

## Summary

#179 introduced provider-agnostic exception types in connectors-api. TrueLayer still has its own parallel hierarchy:
- `TrueLayerApiException extends RuntimeException` → should extend `ProviderApiException`
- `ConsentExpiredException extends RuntimeException` → should extend `ProviderAuthException`
- `RateLimitedException extends RuntimeException` → should extend `ProviderRateLimitedException`

## Target State

Each TrueLayer exception extends the corresponding provider-agnostic type, passing required fields (statusCode, provider) to super. TrueLayer-specific constructors preserved for backward compatibility.

## Scale
S — 3 classes, 30 usage sites
EOF
)"
```

- [ ] **Step 2: Update CLAUDE.md exception types section**

Add the new exception types to the CLAUDE.md description where `UnsupportedCapabilityException` is mentioned, noting `ProviderApiException`, `ProviderRateLimitedException`, `ProviderAuthException`, and `ProviderErrorMapping` in connectors-api.

- [ ] **Step 3: Commit doc changes**

```bash
git add CLAUDE.md
git commit -m "docs(#179): add provider exception hierarchy to CLAUDE.md Refs #179"
```

---

## References

- [2026-10-27-google-provider-test-coverage-design.md] — design spec this plan implements
- [connectors-api/.../SyncTokenExpiredException.java:3] — existing sync exception re-parented
- [connectors-api/.../UnsupportedCapabilityException.java:5] — exception pattern reference
- [email-google/.../GoogleEmailPlatform.java:57] — test constructor pattern
- [document-google/.../GoogleDocumentPlatform.java:58] — test constructor pattern
- [document-google/.../GoogleDocumentPlatform.java:221] — initialSync protocol violation
- [bank-truelayer/.../TrueLayerClient.java:181] — handleResponse() reference pattern
- [docs/protocols/connectors/paginating-client-fail-soft.md] — PP-20260610-83747b
- [GitHub #179] — focal issue
