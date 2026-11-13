# Microsoft Graph Providers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #211 — Epic: Microsoft Graph providers — email, calendar, documents, SharePoint
**Issue group:** #211

**Goal:** Five Platform SPI providers backed by Microsoft Graph REST API, sharing one HTTP client module and one credential resolution path via ServiceConnectionProvider.

**Architecture:** Raw HTTP via `HttpHelper.CLIENT` with domain-specific clients per module. `MicrosoftGraphClient` in shared module handles base URL, auth header injection, `@odata.nextLink` pagination, and error mapping. Each provider module (email, calendar, document, chat) has its own domain client, mapper, and platform impl. Document module uses composed `MicrosoftDriveClient` with basePath parameter for OneDrive/SharePoint. Teams uses dual permission model (delegated + application).

**Tech Stack:** Java 21, Quarkus 3.32.2, `java.net.http.HttpClient` (`HttpHelper.CLIENT`), `jakarta.json`, CDI/Arc

## Global Constraints

- Java source level 21 (running on Java 26 JVM)
- All HTTP calls use `HttpHelper.CLIENT` — never `HttpClient.newBuilder().build()` (protocol PP-20260607-9794cb)
- Credentials passed at call time — no `@ConfigProperty` token fields on shared clients (protocol PP-20260609-0c3e24)
- SPI identifier method is `id()` — not `providerId()` or `typeId()` (protocol PP-20260609-e3a2bd)
- Paginating methods return partial results + WARNING on mid-loop failure (protocol PP-20260610-83747b)
- Parent version: `0.4-SNAPSHOT`, groupId: `io.casehub`
- ArtifactId pattern: `casehub-connectors-<module-name>`
- Build plugin: `jandex-maven-plugin:3.3.1` required in every module
- Graph API base URL: `https://graph.microsoft.com/v1.0`
- Provider name for delegated auth: `"microsoft"`, for application auth: `"microsoft-app"`

---

## Batch 1: Shared Foundation

### Task 1: MicrosoftGraphClient, GraphErrorHandler, TokenRetry

**Files:**
- Create: `microsoft-graph-client/pom.xml`
- Create: `microsoft-graph-client/src/main/java/io/casehub/connectors/microsoft/MicrosoftGraphClient.java`
- Create: `microsoft-graph-client/src/main/java/io/casehub/connectors/microsoft/GraphErrorHandler.java`
- Create: `microsoft-graph-client/src/main/java/io/casehub/connectors/microsoft/TokenRetry.java`
- Modify: `pom.xml` (parent — add module)
- Test: `microsoft-graph-client/src/test/java/io/casehub/connectors/microsoft/MicrosoftGraphClientTest.java`
- Test: `microsoft-graph-client/src/test/java/io/casehub/connectors/microsoft/GraphErrorHandlerTest.java`
- Test: `microsoft-graph-client/src/test/java/io/casehub/connectors/microsoft/TokenRetryTest.java`

**Interfaces:**
- Consumes: `HttpHelper.CLIENT` from `connectors-api`, `ProviderErrorMapping.mapStatusCode()` from `connectors-api`, `Page<T>` and `PageRequest` from `connectors-api`, `ProviderAuthException`, `ProviderRateLimitedException`, `SyncTokenExpiredException` from `connectors-api`
- Produces: `MicrosoftGraphClient` — `get(String path, String accessToken)` → `JsonObject`, `post(String path, JsonObject body, String accessToken)` → `JsonObject`, `patch(String path, JsonObject body, String accessToken)` → `JsonObject`, `delete(String path, String accessToken)` → `void`, `getBytes(String path, String accessToken)` → `byte[]`, `putBytes(String path, byte[] body, String contentType, String accessToken)` → `void`, `getRaw(String path, String accessToken)` → `HttpResponse<byte[]>`, `getBytesFromUrl(String absoluteUrl)` → `byte[]`, `<T> getPage(String path, String accessToken, Function<JsonObject, T> mapper, String cursor)` → `Page<T>`. `GraphErrorHandler` — `handle(HttpResponse<?> response)` → throws. `TokenRetry` — `<T> withTokenRetry(Supplier<String> tokenSupplier, Function<String, T> operation)` → `T`, `withTokenRetryVoid(Supplier<String> tokenSupplier, Consumer<String> operation)` → `void`.

- [ ] **Step 1: Create module pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-connectors-microsoft-graph-client</artifactId>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>microsoft-graph-client</module>` after `github-client` in the parent `pom.xml` modules list.

- [ ] **Step 3: Write GraphErrorHandler tests**

```java
package io.casehub.connectors.microsoft;

import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderRateLimitedException;
import io.casehub.connectors.ProviderApiException;
import io.casehub.connectors.SyncTokenExpiredException;
import org.junit.jupiter.api.Test;
import java.net.http.HttpResponse;
import java.net.http.HttpHeaders;
import java.util.Map;
import java.util.List;
import java.util.NoSuchElementException;
import static org.assertj.core.api.Assertions.*;

class GraphErrorHandlerTest {

    @Test
    void maps401ToTokenExpired() {
        var response = stubResponse(401,
                """
                {"error":{"code":"InvalidAuthenticationToken","message":"expired"}}""",
                Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(ProviderAuthException.class)
                .extracting("reason")
                .isEqualTo(ProviderAuthException.AuthFailureReason.TOKEN_EXPIRED);
    }

    @Test
    void maps403WithAccessDeniedToInsufficientScope() {
        var response = stubResponse(403,
                """
                {"error":{"code":"Authorization_RequestDenied","message":"denied"}}""",
                Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(ProviderAuthException.class)
                .extracting("reason")
                .isEqualTo(ProviderAuthException.AuthFailureReason.INSUFFICIENT_SCOPE);
    }

    @Test
    void maps403WithUnknownCodeToForbidden() {
        var response = stubResponse(403,
                """
                {"error":{"code":"SomeOtherError","message":"denied"}}""",
                Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(ProviderAuthException.class)
                .extracting("reason")
                .isEqualTo(ProviderAuthException.AuthFailureReason.FORBIDDEN);
    }

    @Test
    void maps404ToNoSuchElement() {
        var response = stubResponse(404,
                """
                {"error":{"code":"ErrorItemNotFound","message":"not found"}}""",
                Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(NoSuchElementException.class);
    }

    @Test
    void maps410ToSyncTokenExpired() {
        var response = stubResponse(410,
                """
                {"error":{"code":"resyncRequired","message":"token expired"}}""",
                Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(SyncTokenExpiredException.class);
    }

    @Test
    void maps429WithRetryAfterHeader() {
        var response = stubResponse(429,
                """
                {"error":{"code":"activityLimitReached","message":"throttled"}}""",
                Map.of("Retry-After", List.of("45")));
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(ProviderRateLimitedException.class)
                .extracting("retryAfterSeconds")
                .isEqualTo(45);
    }

    @Test
    void maps429WithoutRetryAfterDefaultsTo30() {
        var response = stubResponse(429,
                """
                {"error":{"code":"activityLimitReached","message":"throttled"}}""",
                Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(ProviderRateLimitedException.class)
                .extracting("retryAfterSeconds")
                .isEqualTo(30);
    }

    @Test
    void maps500ToProviderApiException() {
        var response = stubResponse(500, "internal error", Map.of());
        assertThatThrownBy(() -> GraphErrorHandler.handle(response, "microsoft"))
                .isInstanceOf(ProviderApiException.class);
    }

    @Test
    void doesNotThrowFor2xx() {
        var response = stubResponse(200, "{}", Map.of());
        assertThatCode(() -> GraphErrorHandler.handle(response, "microsoft"))
                .doesNotThrowAnyException();
    }

    // Stub helper — creates a minimal HttpResponse for testing
    @SuppressWarnings("unchecked")
    private static HttpResponse<String> stubResponse(int statusCode, String body,
            Map<String, List<String>> headers) {
        // Use a test stub or mock — implementation in test utility
        return new StubHttpResponse(statusCode, body, headers);
    }
}
```

Create a `StubHttpResponse` test utility that implements `HttpResponse<String>` with the given status code, body, and headers.

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl microsoft-graph-client -Dtest=GraphErrorHandlerTest -f pom.xml`
Expected: FAIL — `GraphErrorHandler` does not exist yet.

- [ ] **Step 5: Implement GraphErrorHandler**

```java
package io.casehub.connectors.microsoft;

import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderAuthException.AuthFailureReason;
import io.casehub.connectors.ProviderErrorMapping;
import io.casehub.connectors.ProviderRateLimitedException;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import java.io.StringReader;
import java.net.http.HttpResponse;
import java.util.Set;

public final class GraphErrorHandler {

    private static final int DEFAULT_RETRY_AFTER = 30;
    private static final Set<String> SCOPE_ERROR_CODES = Set.of(
            "Authorization_RequestDenied", "ErrorAccessDenied", "AccessDenied");

    private GraphErrorHandler() {}

    public static void handle(HttpResponse<?> response, String provider) {
        int status = response.statusCode();
        if (status >= 200 && status < 300) return;

        String body = bodyAsString(response);
        String errorCode = extractErrorCode(body);

        if (status == 403) {
            var reason = SCOPE_ERROR_CODES.contains(errorCode)
                    ? AuthFailureReason.INSUFFICIENT_SCOPE
                    : AuthFailureReason.FORBIDDEN;
            throw new ProviderAuthException(reason, provider);
        }

        if (status == 429) {
            int retryAfter = parseRetryAfter(response, DEFAULT_RETRY_AFTER);
            throw new ProviderRateLimitedException(retryAfter, provider);
        }

        throw ProviderErrorMapping.mapStatusCode(status, body, provider, errorCode);
    }

    private static String extractErrorCode(String body) {
        try (var reader = Json.createReader(new StringReader(body))) {
            JsonObject json = reader.readObject();
            return json.getJsonObject("error").getString("code", "");
        } catch (Exception e) {
            return "";
        }
    }

    private static int parseRetryAfter(HttpResponse<?> response, int defaultValue) {
        return response.headers().firstValue("Retry-After")
                .map(v -> { try { return Integer.parseInt(v); } catch (NumberFormatException e) { return defaultValue; } })
                .orElse(defaultValue);
    }

    private static String bodyAsString(HttpResponse<?> response) {
        Object body = response.body();
        if (body instanceof String s) return s;
        if (body instanceof byte[] b) return new String(b);
        return "";
    }
}
```

- [ ] **Step 6: Run GraphErrorHandler tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl microsoft-graph-client -Dtest=GraphErrorHandlerTest -f pom.xml`
Expected: PASS

- [ ] **Step 7: Write TokenRetry tests**

```java
package io.casehub.connectors.microsoft;

import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderAuthException.AuthFailureReason;
import org.junit.jupiter.api.Test;
import java.util.concurrent.atomic.AtomicInteger;
import static org.assertj.core.api.Assertions.*;

class TokenRetryTest {

    @Test
    void returnsResultOnFirstAttempt() {
        var result = TokenRetry.withTokenRetry(() -> "token", t -> "ok:" + t);
        assertThat(result).isEqualTo("ok:token");
    }

    @Test
    void retriesOnTokenExpired() {
        var calls = new AtomicInteger(0);
        var result = TokenRetry.withTokenRetry(
                () -> "token-" + calls.incrementAndGet(),
                token -> {
                    if (token.equals("token-1")) {
                        throw new ProviderAuthException(AuthFailureReason.TOKEN_EXPIRED, "microsoft");
                    }
                    return "ok:" + token;
                });
        assertThat(result).isEqualTo("ok:token-2");
    }

    @Test
    void propagatesNonTokenExpiredAuthException() {
        assertThatThrownBy(() -> TokenRetry.withTokenRetry(
                () -> "token",
                t -> { throw new ProviderAuthException(AuthFailureReason.FORBIDDEN, "microsoft"); }))
                .isInstanceOf(ProviderAuthException.class)
                .extracting("reason").isEqualTo(AuthFailureReason.FORBIDDEN);
    }

    @Test
    void voidVariantRetries() {
        var calls = new AtomicInteger(0);
        TokenRetry.withTokenRetryVoid(
                () -> "token-" + calls.incrementAndGet(),
                token -> {
                    if (token.equals("token-1")) {
                        throw new ProviderAuthException(AuthFailureReason.TOKEN_EXPIRED, "microsoft");
                    }
                });
        assertThat(calls.get()).isEqualTo(2);
    }
}
```

- [ ] **Step 8: Implement TokenRetry**

```java
package io.casehub.connectors.microsoft;

import io.casehub.connectors.ProviderAuthException;
import io.casehub.connectors.ProviderAuthException.AuthFailureReason;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

public final class TokenRetry {

    private TokenRetry() {}

    public static <T> T withTokenRetry(Supplier<String> tokenSupplier,
                                        Function<String, T> operation) {
        try {
            return operation.apply(tokenSupplier.get());
        } catch (ProviderAuthException e) {
            if (e.reason() == AuthFailureReason.TOKEN_EXPIRED) {
                return operation.apply(tokenSupplier.get());
            }
            throw e;
        }
    }

    public static void withTokenRetryVoid(Supplier<String> tokenSupplier,
                                           Consumer<String> operation) {
        try {
            operation.accept(tokenSupplier.get());
        } catch (ProviderAuthException e) {
            if (e.reason() == AuthFailureReason.TOKEN_EXPIRED) {
                operation.accept(tokenSupplier.get());
                return;
            }
            throw e;
        }
    }
}
```

- [ ] **Step 9: Run TokenRetry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl microsoft-graph-client -Dtest=TokenRetryTest -f pom.xml`
Expected: PASS

- [ ] **Step 10: Write MicrosoftGraphClient tests**

```java
package io.casehub.connectors.microsoft;

import io.casehub.connectors.Page;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;
import java.io.StringReader;
import static org.assertj.core.api.Assertions.*;

class MicrosoftGraphClientTest {

    @Test
    void getReturnsJsonBody() {
        // Mock HttpHelper.CLIENT via test constructor
        var client = new MicrosoftGraphClient(request -> stubJsonResponse(200,
                """
                {"id":"123","displayName":"Test"}"""));
        var result = client.get("/me/profile", "bearer-token");
        assertThat(result.getString("id")).isEqualTo("123");
    }

    @Test
    void getThrowsOnErrorStatus() {
        var client = new MicrosoftGraphClient(request -> stubJsonResponse(404,
                """
                {"error":{"code":"ErrorItemNotFound","message":"not found"}}"""));
        assertThatThrownBy(() -> client.get("/me/messages/invalid", "token"))
                .isInstanceOf(java.util.NoSuchElementException.class);
    }

    @Test
    void getPageReturnsSinglePage() {
        var client = new MicrosoftGraphClient(request -> stubJsonResponse(200,
                """
                {"value":[{"id":"1"},{"id":"2"}]}"""));
        var page = client.getPage("/me/messages", "token",
                json -> json.getString("id"), null);
        assertThat(page.items()).containsExactly("1", "2");
        assertThat(page.hasMore()).isFalse();
        assertThat(page.nextCursor()).isNull();
    }

    @Test
    void getPageFollowsOdataNextLink() {
        var client = new MicrosoftGraphClient(request -> {
            String uri = request.uri().toString();
            if (uri.contains("page2")) {
                return stubJsonResponse(200,
                        """
                        {"value":[{"id":"3"}]}""");
            }
            return stubJsonResponse(200,
                    """
                    {"value":[{"id":"1"},{"id":"2"}],"@odata.nextLink":"https://graph.microsoft.com/v1.0/me/messages?page2"}""");
        });
        // First page
        var page1 = client.getPage("/me/messages", "token",
                json -> json.getString("id"), null);
        assertThat(page1.items()).containsExactly("1", "2");
        assertThat(page1.hasMore()).isTrue();
        assertThat(page1.nextCursor()).contains("page2");

        // Second page using cursor
        var page2 = client.getPage("/me/messages", "token",
                json -> json.getString("id"), page1.nextCursor());
        assertThat(page2.items()).containsExactly("3");
        assertThat(page2.hasMore()).isFalse();
    }

    @Test
    void getPageUsesFullCursorUrlIgnoringPath() {
        var client = new MicrosoftGraphClient(request -> {
            // Verify the cursor URL is used directly, not BASE_URL + path
            assertThat(request.uri().toString())
                    .isEqualTo("https://graph.microsoft.com/v1.0/me/messages?$skip=10");
            return stubJsonResponse(200,
                    """
                    {"value":[{"id":"3"}]}""");
        });
        client.getPage("/ignored", "token", json -> json.getString("id"),
                "https://graph.microsoft.com/v1.0/me/messages?$skip=10");
    }

    @Test
    void postSendsJsonBody() {
        var client = new MicrosoftGraphClient(request -> {
            assertThat(request.method()).isEqualTo("POST");
            return stubJsonResponse(201,
                    """
                    {"id":"new-1"}""");
        });
        var body = Json.createObjectBuilder().add("subject", "test").build();
        var result = client.post("/me/events", body, "token");
        assertThat(result.getString("id")).isEqualTo("new-1");
    }

    @Test
    void deleteReturnsVoidOn204() {
        var client = new MicrosoftGraphClient(request ->
                stubJsonResponse(204, ""));
        assertThatCode(() -> client.delete("/me/events/123", "token"))
                .doesNotThrowAnyException();
    }

    @Test
    void setsAuthorizationBearerHeader() {
        var client = new MicrosoftGraphClient(request -> {
            assertThat(request.headers().firstValue("Authorization"))
                    .hasValue("Bearer my-token");
            return stubJsonResponse(200, "{}");
        });
        client.get("/me/profile", "my-token");
    }

    // Helper: create a stub response from status + body string
    private StubHttpResponse stubJsonResponse(int status, String body) {
        return new StubHttpResponse(status, body);
    }
}
```

`MicrosoftGraphClient` takes a `Function<HttpRequest, HttpResponse<String>>` test constructor for mocking HTTP calls. Production constructor uses `HttpHelper.CLIENT::send`.

- [ ] **Step 11: Implement MicrosoftGraphClient**

```java
package io.casehub.connectors.microsoft;

import io.casehub.connectors.Page;
import io.casehub.connectors.http.HttpHelper;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.json.Json;
import jakarta.json.JsonArray;
import jakarta.json.JsonObject;
import java.io.StringReader;
import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Function;
import java.util.logging.Level;
import java.util.logging.Logger;

@ApplicationScoped
public class MicrosoftGraphClient {

    private static final String BASE_URL = "https://graph.microsoft.com/v1.0";
    private static final int MAX_PAGES = 50;
    private static final Logger LOG = Logger.getLogger(MicrosoftGraphClient.class.getName());

    private final HttpSender sender;

    @FunctionalInterface
    interface HttpSender {
        HttpResponse<String> send(HttpRequest request) throws Exception;
    }

    public MicrosoftGraphClient() {
        this.sender = request -> HttpHelper.CLIENT.send(request,
                HttpResponse.BodyHandlers.ofString());
    }

    MicrosoftGraphClient(HttpSender sender) {
        this.sender = sender;
    }

    public JsonObject get(String path, String accessToken) {
        var request = jsonRequest(path, accessToken).GET().build();
        return sendAndParse(request);
    }

    public JsonObject post(String path, JsonObject body, String accessToken) {
        var request = jsonRequest(path, accessToken)
                .POST(HttpRequest.BodyPublishers.ofString(body.toString()))
                .header("Content-Type", "application/json")
                .build();
        return sendAndParse(request);
    }

    public JsonObject patch(String path, JsonObject body, String accessToken) {
        var request = jsonRequest(path, accessToken)
                .method("PATCH", HttpRequest.BodyPublishers.ofString(body.toString()))
                .header("Content-Type", "application/json")
                .build();
        return sendAndParse(request);
    }

    public void delete(String path, String accessToken) {
        var request = jsonRequest(path, accessToken).DELETE().build();
        var response = doSend(request);
        if (response.statusCode() >= 300) {
            GraphErrorHandler.handle(response, "microsoft");
        }
    }

    public byte[] getBytes(String path, String accessToken) {
        var request = HttpRequest.newBuilder(URI.create(BASE_URL + path))
                .header("Authorization", "Bearer " + accessToken)
                .GET().build();
        try {
            var response = HttpHelper.CLIENT.send(request,
                    HttpResponse.BodyHandlers.ofByteArray());
            if (response.statusCode() >= 300) {
                GraphErrorHandler.handle(stubStringResponse(response), "microsoft");
            }
            return response.body();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void putBytes(String path, byte[] body, String contentType,
            String accessToken) {
        var request = HttpRequest.newBuilder(URI.create(BASE_URL + path))
                .header("Authorization", "Bearer " + accessToken)
                .header("Content-Type", contentType)
                .PUT(HttpRequest.BodyPublishers.ofByteArray(body))
                .build();
        try {
            var response = HttpHelper.CLIENT.send(request,
                    HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() >= 300) {
                GraphErrorHandler.handle(response, "microsoft");
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public HttpResponse<byte[]> getRaw(String path, String accessToken) {
        var request = HttpRequest.newBuilder(URI.create(BASE_URL + path))
                .header("Authorization", "Bearer " + accessToken)
                .GET().build();
        try {
            return HttpHelper.CLIENT.send(request,
                    HttpResponse.BodyHandlers.ofByteArray());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public byte[] getBytesFromUrl(String absoluteUrl) {
        var request = HttpRequest.newBuilder(URI.create(absoluteUrl))
                .GET().build();
        try {
            var response = HttpHelper.CLIENT.send(request,
                    HttpResponse.BodyHandlers.ofByteArray());
            if (response.statusCode() >= 300) {
                throw new RuntimeException("CDN fetch failed: " + response.statusCode());
            }
            return response.body();
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public <T> Page<T> getPage(String path, String accessToken,
            Function<JsonObject, T> mapper, String cursor) {
        String url = (cursor != null) ? cursor : BASE_URL + path;
        var request = HttpRequest.newBuilder(URI.create(url))
                .header("Authorization", "Bearer " + accessToken)
                .header("Accept", "application/json")
                .GET().build();

        var response = doSend(request);
        GraphErrorHandler.handle(response, "microsoft");

        var json = parseJson(response.body());
        JsonArray valueArray = json.getJsonArray("value");
        List<T> items = new ArrayList<>();
        if (valueArray != null) {
            for (var item : valueArray) {
                items.add(mapper.apply(item.asJsonObject()));
            }
        }
        String nextLink = json.containsKey("@odata.nextLink")
                ? json.getString("@odata.nextLink") : null;
        return new Page<>(List.copyOf(items), nextLink, nextLink != null);
    }

    private HttpRequest.Builder jsonRequest(String path, String accessToken) {
        return HttpRequest.newBuilder(URI.create(BASE_URL + path))
                .header("Authorization", "Bearer " + accessToken)
                .header("Accept", "application/json");
    }

    private HttpResponse<String> doSend(HttpRequest request) {
        try {
            return sender.send(request);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    private JsonObject sendAndParse(HttpRequest request) {
        var response = doSend(request);
        GraphErrorHandler.handle(response, "microsoft");
        String body = response.body();
        if (body == null || body.isBlank()) return JsonObject.EMPTY_JSON_OBJECT;
        return parseJson(body);
    }

    private static JsonObject parseJson(String body) {
        try (var reader = Json.createReader(new StringReader(body))) {
            return reader.readObject();
        }
    }
}
```

- [ ] **Step 12: Run all tests in microsoft-graph-client**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl microsoft-graph-client -f pom.xml`
Expected: ALL PASS

- [ ] **Step 13: Commit**

```bash
git add microsoft-graph-client/ pom.xml
git commit -m "feat(#211): add microsoft-graph-client shared module

MicrosoftGraphClient (raw HTTP via HttpHelper.CLIENT), GraphErrorHandler
(Graph-specific 403/429 mapping), TokenRetry (401 re-resolve pattern).

Refs #211"
```

---

## Batch 2: Email Provider

### Task 2: MicrosoftEmailPlatform (Outlook Mail)

**Files:**
- Create: `email-microsoft/pom.xml`
- Create: `email-microsoft/src/main/java/io/casehub/connectors/email/microsoft/MicrosoftMailClient.java`
- Create: `email-microsoft/src/main/java/io/casehub/connectors/email/microsoft/MailMapper.java`
- Create: `email-microsoft/src/main/java/io/casehub/connectors/email/microsoft/MicrosoftEmailPlatform.java`
- Create: `email-microsoft/src/main/java/io/casehub/connectors/email/microsoft/EmailMicrosoftBeans.java`
- Modify: `pom.xml` (parent — add module)
- Test: `email-microsoft/src/test/java/io/casehub/connectors/email/microsoft/MailMapperTest.java`
- Test: `email-microsoft/src/test/java/io/casehub/connectors/email/microsoft/MicrosoftMailClientTest.java`
- Test: `email-microsoft/src/test/java/io/casehub/connectors/email/microsoft/MicrosoftEmailPlatformTest.java`

**Interfaces:**
- Consumes: `MicrosoftGraphClient` from Task 1, `TokenRetry` from Task 1, `EmailPlatform` SPI from `email-spi`, `ServiceConnectionProvider` from `platform-api`, `Page<T>`, `PageRequest` from `connectors-api`
- Produces: `MicrosoftEmailPlatform implements EmailPlatform` — `id()` → `"microsoft"`, all `EmailPlatform` methods implemented against Graph Mail API

- [ ] **Step 1: Create module pom.xml and add to parent**

Same pattern as Task 1. ArtifactId: `casehub-connectors-email-microsoft`. Dependencies: `casehub-connectors-email-spi`, `casehub-connectors-microsoft-graph-client`, `casehub-platform-api`, `quarkus-arc`. Test deps: `quarkus-junit5`, `assertj-core`.

- [ ] **Step 2: Write MailMapper tests**

Test mapping from Graph JSON to SPI types: `Mailbox`, `EmailSummary`, `EmailMessage`, `EmailAttachment`. Test `wellKnownName` mapping, body content type, address extraction, attachment metadata.

- [ ] **Step 3: Implement MailMapper**

Static methods: `toMailbox(JsonObject)` → `Mailbox`, `toEmailSummary(JsonObject)` → `EmailSummary`, `toEmailMessage(JsonObject, String mailboxId)` → `EmailMessage`. Handle Outlook-specific fields.

- [ ] **Step 4: Run MailMapper tests — verify PASS**

- [ ] **Step 5: Write MicrosoftMailClient tests**

Mock `MicrosoftGraphClient` and verify correct path construction, parameter passing, and response mapping. Test: `listMailFolders`, `listMessages` with date filter and cursor, `getMessage` with attachment expansion, `getAttachmentContent`, `searchMessages`.

- [ ] **Step 6: Implement MicrosoftMailClient**

```java
package io.casehub.connectors.email.microsoft;

import io.casehub.connectors.Page;
import io.casehub.connectors.microsoft.MicrosoftGraphClient;
import jakarta.json.JsonObject;
import java.time.Instant;
import java.time.format.DateTimeFormatter;
import java.util.List;

public class MicrosoftMailClient {
    private final MicrosoftGraphClient graph;

    public MicrosoftMailClient(MicrosoftGraphClient graph) {
        this.graph = graph;
    }

    public List<JsonObject> listMailFolders(String accessToken) {
        return graph.getPage("/me/mailFolders", accessToken,
                json -> json, null).items();
    }

    public Page<JsonObject> listMessages(String folderId, String accessToken,
            Instant from, Instant to, int top, String cursor) {
        String filter = buildDateFilter(from, to);
        String path = "/me/mailFolders/" + folderId + "/messages?$top=" + top
                + (filter.isEmpty() ? "" : "&$filter=" + filter);
        return graph.getPage(path, accessToken, json -> json, cursor);
    }

    public JsonObject getMessage(String messageId, String accessToken) {
        return graph.get("/me/messages/" + messageId
                + "?$expand=attachments", accessToken);
    }

    public byte[] getAttachmentContent(String messageId, String attachmentId,
            String accessToken) {
        return graph.getBytes("/me/messages/" + messageId
                + "/attachments/" + attachmentId + "/$value", accessToken);
    }

    public Page<JsonObject> searchMessages(String query, String accessToken,
            int top, String cursor) {
        String path = "/me/messages?$search=\"" + query + "\"&$top=" + top;
        return graph.getPage(path, accessToken, json -> json, cursor);
    }

    private static String buildDateFilter(Instant from, Instant to) {
        var parts = new java.util.ArrayList<String>();
        if (from != null) {
            parts.add("receivedDateTime ge " + DateTimeFormatter.ISO_INSTANT.format(from));
        }
        if (to != null) {
            parts.add("receivedDateTime lt " + DateTimeFormatter.ISO_INSTANT.format(to));
        }
        return String.join(" and ", parts);
    }
}
```

- [ ] **Step 7: Run MicrosoftMailClient tests — verify PASS**

- [ ] **Step 8: Write MicrosoftEmailPlatform tests**

Test the full SPI contract: `id()` returns `"microsoft"`, `listMailboxes()` delegates correctly, `listMessages()` passes date range and pagination through, `getMessage()` maps correctly, `search()` delegates.

- [ ] **Step 9: Implement MicrosoftEmailPlatform and EmailMicrosoftBeans**

```java
package io.casehub.connectors.email.microsoft;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.email.spi.*;
import io.casehub.connectors.microsoft.TokenRetry;
import io.casehub.platform.api.authn.ServiceConnectionProvider;
import java.time.Instant;
import java.util.List;

public class MicrosoftEmailPlatform implements EmailPlatform {

    private final MicrosoftMailClient mailClient;
    private final ServiceConnectionProvider connectionProvider;

    public MicrosoftEmailPlatform(MicrosoftMailClient mailClient,
            ServiceConnectionProvider connectionProvider) {
        this.mailClient = mailClient;
        this.connectionProvider = connectionProvider;
    }

    MicrosoftEmailPlatform(MicrosoftMailClient mailClient) {
        this(mailClient, null);
    }

    @Override public String id() { return "microsoft"; }

    @Override
    public List<Mailbox> listMailboxes() {
        return TokenRetry.withTokenRetry(this::resolveToken,
                token -> mailClient.listMailFolders(token).stream()
                        .map(MailMapper::toMailbox).toList());
    }

    @Override
    public Page<EmailSummary> listMessages(String mailboxId, Instant from,
            Instant to, PageRequest pagination) {
        return TokenRetry.withTokenRetry(this::resolveToken,
                token -> {
                    var page = mailClient.listMessages(mailboxId, token,
                            from, to, pagination.pageSize(), pagination.cursor());
                    return new Page<>(page.items().stream()
                            .map(MailMapper::toEmailSummary).toList(),
                            page.nextCursor(), page.hasMore());
                });
    }

    @Override
    public EmailMessage getMessage(String mailboxId, String messageId) {
        return TokenRetry.withTokenRetry(this::resolveToken,
                token -> MailMapper.toEmailMessage(
                        mailClient.getMessage(messageId, token), mailboxId));
    }

    @Override
    public byte[] getAttachmentContent(String mailboxId, String messageId,
            String attachmentId) {
        return TokenRetry.withTokenRetry(this::resolveToken,
                token -> mailClient.getAttachmentContent(messageId, attachmentId, token));
    }

    @Override
    public Page<EmailSummary> search(String query, PageRequest pagination) {
        return TokenRetry.withTokenRetry(this::resolveToken,
                token -> {
                    var page = mailClient.searchMessages(query, token,
                            pagination.pageSize(), pagination.cursor());
                    return new Page<>(page.items().stream()
                            .map(MailMapper::toEmailSummary).toList(),
                            page.nextCursor(), page.hasMore());
                });
    }

    private String resolveToken(/* actorId TBD at platform level */) {
        if (connectionProvider == null) throw new IllegalStateException("No connection provider");
        return connectionProvider.getAccessToken(null, "microsoft", "default")
                .accessToken();
    }
}
```

CDI producer:
```java
package io.casehub.connectors.email.microsoft;

import io.casehub.connectors.microsoft.MicrosoftGraphClient;
import io.casehub.platform.api.authn.ServiceConnectionProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class EmailMicrosoftBeans {
    @Produces @ApplicationScoped
    MicrosoftEmailPlatform microsoftEmailPlatform(MicrosoftGraphClient graph,
            ServiceConnectionProvider connectionProvider) {
        return new MicrosoftEmailPlatform(new MicrosoftMailClient(graph),
                connectionProvider);
    }
}
```

- [ ] **Step 10: Run all email-microsoft tests — verify PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-microsoft -f pom.xml`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add email-microsoft/ pom.xml
git commit -m "feat(#211): add email-microsoft provider (Outlook Mail)

MicrosoftMailClient, MailMapper, MicrosoftEmailPlatform with TokenRetry.
Implements EmailPlatform SPI against Graph Mail API.

Refs #211"
```

---

## Batch 3: Calendar Provider

### Task 3: MicrosoftCalendarPlatform (Outlook Calendar)

**Files:**
- Create: `calendar-microsoft/pom.xml`
- Create: `calendar-microsoft/src/main/java/io/casehub/connectors/calendar/microsoft/MicrosoftCalendarClient.java`
- Create: `calendar-microsoft/src/main/java/io/casehub/connectors/calendar/microsoft/CalendarMapper.java`
- Create: `calendar-microsoft/src/main/java/io/casehub/connectors/calendar/microsoft/MicrosoftCalendarPlatform.java`
- Create: `calendar-microsoft/src/main/java/io/casehub/connectors/calendar/microsoft/CalendarMicrosoftBeans.java`
- Modify: `pom.xml` (parent — add module)
- Test: `calendar-microsoft/src/test/java/io/casehub/connectors/calendar/microsoft/CalendarMapperTest.java`
- Test: `calendar-microsoft/src/test/java/io/casehub/connectors/calendar/microsoft/MicrosoftCalendarClientTest.java`
- Test: `calendar-microsoft/src/test/java/io/casehub/connectors/calendar/microsoft/MicrosoftCalendarPlatformTest.java`

**Interfaces:**
- Consumes: `MicrosoftGraphClient`, `TokenRetry` from Task 1, `CalendarPlatform` SPI from `calendar-spi`, `ServiceConnectionProvider`, `SyncResult`, `SyncRequest` from `connectors-api`
- Produces: `MicrosoftCalendarPlatform implements CalendarPlatform` — `id()` → `"microsoft"`, all methods including `listEventsSync` with delta query support

- [ ] **Step 1: Create module pom.xml and add to parent**

ArtifactId: `casehub-connectors-calendar-microsoft`. Dependencies: `casehub-connectors-calendar-spi`, `casehub-connectors-microsoft-graph-client`, `casehub-platform-api`, `quarkus-arc`.

- [ ] **Step 2: Write CalendarMapper tests**

Test: `toCalendarInfo(JsonObject)` → `CalendarInfo`, `toCalendarEvent(JsonObject)` → `CalendarEvent`. Test EventTiming sealed hierarchy: `Timed` (dateTime + timeZone) vs `AllDay` (date only). Test attendees mapping, recurrence, isAllDay flag.

- [ ] **Step 3: Implement CalendarMapper**

Static methods for JSON → SPI model mapping. Handle `start`/`end` objects with `dateTime`+`timeZone` fields vs `date` field for all-day events.

- [ ] **Step 4: Write MicrosoftCalendarClient tests**

Test: `listCalendars`, `listEvents` with date range, `getEvent`, `createEvent`, `updateEvent`, `deleteEvent`, `deltaSync` (initial with date range, incremental with deltaLink token, 410 → SyncTokenExpiredException).

- [ ] **Step 5: Implement MicrosoftCalendarClient**

Key methods:
- `listCalendars(accessToken)` → `GET /me/calendars`
- `listEvents(calendarId, from, to, accessToken)` → `GET /me/calendars/{id}/calendarView?startDateTime=...&endDateTime=...`
- `getEvent(calendarId, eventId, accessToken)` → `GET /me/events/{id}`
- `createEvent(calendarId, body, accessToken)` → `POST /me/calendars/{id}/events`
- `updateEvent(eventId, body, accessToken)` → `PATCH /me/events/{id}`
- `deleteEvent(eventId, accessToken)` → `DELETE /me/events/{id}`
- `deltaSync(calendarId, syncToken, from, to, accessToken)` → `GET /me/calendarView/delta`

Delta sync: initial call includes `startDateTime` (6 months ago) and `endDateTime` (12 months ahead) when `syncToken` is null. Incremental call uses the `deltaLink` as the sync token. Parse `@removed` property for deleted events.

- [ ] **Step 6: Implement MicrosoftCalendarPlatform and CalendarMicrosoftBeans**

Same pattern as email — delegates to `MicrosoftCalendarClient` with `TokenRetry`. `listEventsSync` maps delta response to `SyncResult<CalendarEvent>`.

- [ ] **Step 7: Run all calendar-microsoft tests — verify PASS**

- [ ] **Step 8: Commit**

```bash
git add calendar-microsoft/ pom.xml
git commit -m "feat(#211): add calendar-microsoft provider (Outlook Calendar)

MicrosoftCalendarClient, CalendarMapper, MicrosoftCalendarPlatform.
Implements CalendarPlatform SPI with delta sync support.

Refs #211"
```

---

## Batch 4: Document Provider

### Task 4: MicrosoftDrivePlatform + SharePointDrivePlatform (OneDrive + SharePoint)

**Files:**
- Create: `document-microsoft/pom.xml`
- Create: `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/MicrosoftDriveClient.java`
- Create: `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/DriveMapper.java`
- Create: `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/MicrosoftDrivePlatform.java`
- Create: `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/SharePointDrivePlatform.java`
- Create: `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/DocumentMicrosoftBeans.java`
- Modify: `pom.xml` (parent — add module)
- Test: `document-microsoft/src/test/java/io/casehub/connectors/document/microsoft/DriveMapperTest.java`
- Test: `document-microsoft/src/test/java/io/casehub/connectors/document/microsoft/MicrosoftDriveClientTest.java`
- Test: `document-microsoft/src/test/java/io/casehub/connectors/document/microsoft/MicrosoftDrivePlatformTest.java`
- Test: `document-microsoft/src/test/java/io/casehub/connectors/document/microsoft/SharePointDrivePlatformTest.java`

**Interfaces:**
- Consumes: `MicrosoftGraphClient`, `TokenRetry` from Task 1, `DocumentPlatform` SPI from `document-spi`, `ServiceConnectionProvider`, `Page<T>`, `PageRequest`, `SyncResult`, `SyncRequest` from `connectors-api`
- Produces: `MicrosoftDrivePlatform implements DocumentPlatform` (id `"microsoft-onedrive"`, basePath `/me/drive`), `SharePointDrivePlatform implements DocumentPlatform` (id `"microsoft-sharepoint"`, basePath `/sites/{siteId}/drive`). Both use composed `MicrosoftDriveClient(graph, basePath)`.

- [ ] **Step 1: Create module pom.xml and add to parent**

ArtifactId: `casehub-connectors-document-microsoft`. Dependencies: `casehub-connectors-document-spi`, `casehub-connectors-microsoft-graph-client`, `casehub-platform-api`, `quarkus-arc`.

- [ ] **Step 2: Write DriveMapper tests**

Test: `toDocumentSummary(JsonObject)` → `DocumentSummary`, `toDocumentMetadata(JsonObject)` → `DocumentMetadata`, `toFolder(JsonObject)` → `Folder`. Test file vs folder discrimination (presence of `folder` facet), size, MIME type, last modified.

- [ ] **Step 3: Implement DriveMapper**

- [ ] **Step 4: Write MicrosoftDriveClient tests**

Test: `listChildren` with pagination and cursor, `getItem`, `downloadContent` (302 redirect handling with SSRF validation), `uploadContent` (simple ≤4MB and resumable >4MB paths), `deleteItem`, `delta` sync (initial, incremental, 410). Test `validateRedirectHost` against allowed CDN hosts.

Key test for download:
```java
@Test
void downloadContentFollows302WithSsrfValidation() {
    // First call returns 302 with Location header
    // Second call fetches from the CDN URL
    var client = new MicrosoftDriveClient(mockGraph, "/me/drive");
    // mockGraph.getRaw returns 302 with Location: https://myorg-my.sharepoint.com/...
    // mockGraph.getBytesFromUrl returns file bytes
    byte[] result = client.downloadContent("file-1", "token");
    assertThat(result).isEqualTo(expectedBytes);
}

@Test
void downloadContentRejectsNonAllowedHost() {
    // Location header points to evil.com
    var client = new MicrosoftDriveClient(mockGraph, "/me/drive");
    assertThatThrownBy(() -> client.downloadContent("file-1", "token"))
            .isInstanceOf(IllegalStateException.class);
}
```

- [ ] **Step 5: Implement MicrosoftDriveClient**

Composed drive client with `basePath` constructor parameter. Key implementation details:
- `downloadContent()`: call `graph.getRaw()`, check for 302, validate redirect host against `ALLOWED_CDN_HOSTS`, then `graph.getBytesFromUrl()`.
- `uploadContent()`: detect size, use simple PUT for ≤4MB, createUploadSession + chunked PUT for >4MB.
- `delta()`: handle `@odata.deltaLink` for sync token, `@removed` facet for deleted items.

- [ ] **Step 6: Write MicrosoftDrivePlatform and SharePointDrivePlatform tests**

Test both impls verify they delegate to `MicrosoftDriveClient` with correct basePath. Test `id()`, `supports()` for all 4 capability interfaces, all file/folder/search/sharing operations.

SharePointDrivePlatform test: verify degraded mode when siteId is blank.

- [ ] **Step 7: Implement MicrosoftDrivePlatform, SharePointDrivePlatform, DocumentMicrosoftBeans**

`MicrosoftDrivePlatform` uses basePath `/me/drive`, id `"microsoft-onedrive"`.
`SharePointDrivePlatform` takes `siteId` in constructor, uses basePath `/sites/{siteId}/drive`, id `"microsoft-sharepoint"`. When siteId is blank, logs WARNING and all operations throw `IllegalStateException`.

CDI producer creates both beans, injecting `@ConfigProperty(name = "casehub.connectors.sharepoint.site-id", defaultValue = "")` for SharePoint.

Both impls use inner classes for capability sub-interfaces (following Google pattern): `DriveFileOperations implements FileOperations`, `DriveFolderOperations implements FolderOperations`, `DriveSearchOperations implements SearchOperations`, `DriveSharingOperations implements SharingOperations`.

- [ ] **Step 8: Run all document-microsoft tests — verify PASS**

- [ ] **Step 9: Commit**

```bash
git add document-microsoft/ pom.xml
git commit -m "feat(#211): add document-microsoft provider (OneDrive + SharePoint)

MicrosoftDriveClient (composed, basePath param), DriveMapper,
MicrosoftDrivePlatform (OneDrive), SharePointDrivePlatform (SharePoint).
302 redirect with SSRF defense, resumable upload for >4MB files.

Refs #211"
```

---

## Batch 5: Chat Provider

### Task 5: TeamsChatPlatform (Teams)

**Files:**
- Create: `chat-teams/pom.xml`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsMapper.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java`
- Modify: `pom.xml` (parent — add module)
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsMapperTest.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsClientTest.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java`

**Interfaces:**
- Consumes: `MicrosoftGraphClient`, `TokenRetry` from Task 1, `ChatPlatform` SPI from `chat-spi`, `ServiceConnectionProvider`
- Produces: `TeamsChatPlatform implements ChatPlatform` — `id()` → `"teams"`, 8 native capabilities (Messaging, Threading, Discovery, Presence, Members, ChannelManagement, MemberManagement, MessageHistory), 2 degraded (Reactions, Commands). Uses `ChatPlatform.Builder` pattern.

**Prerequisite:** `@RequiresScopes` must be `@Repeatable` in platform-api for dual-provider scope declarations. If not yet available, the `@RequiresScopes` annotations are commented with a TODO and the module compiles without them — the runtime behavior is unaffected (scopes are declarative metadata for consent flow).

- [ ] **Step 1: Create module pom.xml and add to parent**

ArtifactId: `casehub-connectors-chat-teams`. Dependencies: `casehub-connectors-chat-spi`, `casehub-connectors-microsoft-graph-client`, `casehub-platform-api`, `quarkus-arc`.

- [ ] **Step 2: Write TeamsMapper tests**

Test: `toChannel(JsonObject, String teamId)` → `Channel` (compound `teamId:channelId` identifier, displayName, memberCount). Test `toMember(JsonObject)` → `Member`, `toMessage(JsonObject)` → `ReceivedMessage`, `toPresenceStatus(JsonObject)` → `PresenceStatus`.

- [ ] **Step 3: Implement TeamsMapper**

Channel identifier uses `:` separator: `teamId + ":" + channelId`. Parse `membershipType` (standard/private/shared).

- [ ] **Step 4: Write TeamsClient tests**

Test both delegated and application permission operations:
- `listJoinedTeams(userAccessToken)` → `GET /me/joinedTeams`
- `listChannels(teamId, userAccessToken)` → `GET /teams/{id}/channels`
- `sendMessage(teamId, channelId, content, appAccessToken)` → `POST /teams/{id}/channels/{id}/messages`
- `replyToMessage(teamId, channelId, messageId, content, appAccessToken)` → reply POST
- `listMessages(teamId, channelId, appAccessToken, cursor)` → paginated GET
- `getPresence(userId, userAccessToken)` → `GET /users/{id}/presence`
- `listMembers(teamId, userAccessToken)` → `GET /teams/{id}/members`
- `createChannel(teamId, name, userAccessToken)` → POST
- `deleteChannel(teamId, channelId, userAccessToken)` → DELETE

- [ ] **Step 5: Implement TeamsClient**

Separate methods clearly indicate which token type they expect via parameter naming (`userAccessToken` vs `appAccessToken`).

- [ ] **Step 6: Write TeamsChatPlatform tests**

Test: `id()` returns `"teams"`, `supports()` returns true for 8 capabilities and false for Reactions and Commands. Test messaging sends with application token, discovery uses delegated token (compound channel IDs). Test that degraded capabilities (Reactions, Commands) return no-op results.

- [ ] **Step 7: Implement TeamsChatPlatform and TeamsBeans**

Uses `ChatPlatform.Builder` pattern. Two token resolution methods:
- `resolveUserToken()` — `connectionProvider.getAccessToken(actorId, "microsoft", "default")`
- `resolveAppToken()` — `connectionProvider.getAccessToken("__application__", "microsoft-app", "default")`

Messaging and Threading capabilities use `resolveAppToken()`. All other capabilities use `resolveUserToken()`.

Discovery: `listChannels()` calls `listJoinedTeams()` then `listChannels()` per team, flattening into a single list with compound IDs.

Channel parsing helper: `parseTeamAndChannel(String compoundId)` → splits `teamId:channelId`.

- [ ] **Step 8: Run all chat-teams tests — verify PASS**

- [ ] **Step 9: Commit**

```bash
git add chat-teams/ pom.xml
git commit -m "feat(#211): add chat-teams provider (Teams ChatPlatform)

TeamsClient, TeamsMapper, TeamsChatPlatform with dual permission model.
8 native capabilities (delegated + application tokens), 2 degraded.

Refs #211"
```

---

## Batch 6: Integration and Documentation

### Task 6: Integration tests, CLAUDE.md, consumer guide

**Files:**
- Create: `integration-tests/src/test/java/io/casehub/connectors/microsoft/MicrosoftIntegrationTest.java`
- Modify: `CLAUDE.md` (add new modules to module table)
- Modify: `docs/guides/consumer-guide.md` (add Microsoft provider sections)

**Interfaces:**
- Consumes: All 5 modules from Tasks 1-5

- [ ] **Step 1: Write integration smoke test**

Gated by `@EnabledIfSystemProperty(named = "microsoft.test.enabled", matches = "true")`. Tests basic connectivity for each provider against a real Microsoft 365 developer sandbox.

```java
@QuarkusTest
@EnabledIfSystemProperty(named = "microsoft.test.enabled", matches = "true")
class MicrosoftIntegrationTest {

    @Inject MicrosoftEmailPlatform emailPlatform;
    @Inject MicrosoftCalendarPlatform calendarPlatform;
    @Inject MicrosoftDrivePlatform drivePlatform;

    @Test
    void emailListMailboxes() {
        var mailboxes = emailPlatform.listMailboxes();
        assertThat(mailboxes).isNotEmpty();
        assertThat(mailboxes).anyMatch(m -> m.name().equalsIgnoreCase("Inbox"));
    }

    @Test
    void calendarListCalendars() {
        var calendars = calendarPlatform.listCalendars();
        assertThat(calendars).isNotEmpty();
    }

    @Test
    void driveListFiles() {
        var files = drivePlatform.files().list("root",
                PageRequest.first(10));
        assertThat(files).isNotNull();
    }
}
```

- [ ] **Step 2: Update CLAUDE.md module table**

Add to the `## Modules` table:
```
| `microsoft-graph-client` | Shared Microsoft Graph HTTP client (auth, pagination, errors) |
| `email-microsoft` | Microsoft Outlook EmailPlatform provider |
| `calendar-microsoft` | Microsoft Outlook CalendarPlatform provider |
| `document-microsoft` | Microsoft OneDrive + SharePoint DocumentPlatform provider |
| `chat-teams` | Microsoft Teams ChatPlatform provider |
```

Also update the `## What This Project Is` section: add Microsoft providers to the listed implementations for EmailPlatform, CalendarPlatform, DocumentPlatform, and ChatPlatform.

- [ ] **Step 3: Update consumer guide**

Add a new `### Microsoft Graph` section under each platform heading (Email, Calendar, Documents, Chat) in `docs/guides/consumer-guide.md`. Document: Maven coordinates, config properties, OAuth2 scopes required.

- [ ] **Step 4: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add integration-tests/ CLAUDE.md docs/guides/consumer-guide.md
git commit -m "feat(#211): add Microsoft integration tests and documentation

Integration smoke tests (gated by microsoft.test.enabled property).
Updated CLAUDE.md module table and consumer guide with Microsoft providers.

Refs #211"
```

---

## References

- `specs/issue-211-microsoft-graph-providers/2026-11-13-microsoft-graph-providers-design.md` — design spec
- `specs/issue-211-microsoft-graph-providers/decisions.md` — captured decisions D1-D7
- `connectors-api/src/main/java/io/casehub/connectors/http/HttpHelper.java` — shared HTTP client
- `connectors-api/src/main/java/io/casehub/connectors/ProviderErrorMapping.java` — error mapping
- `connectors-api/src/main/java/io/casehub/connectors/Page.java` — pagination record
- `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatform.java` — EmailPlatform SPI
- `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/CalendarPlatform.java` — CalendarPlatform SPI  (lines 14-36)
- `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java` — DocumentPlatform SPI
- `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java` — ChatPlatform SPI (line 48)
- `email-google/src/main/java/io/casehub/connectors/email/google/GoogleEmailPlatform.java:39` — reference EmailPlatform impl
- `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java:36` — reference CalendarPlatform impl
- `document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java:40` — reference DocumentPlatform impl (inner class pattern)
- Protocol PP-20260607-9794cb — shared HTTP client
- Protocol PP-20260609-0c3e24 — credential config ownership
- Protocol PP-20260609-e3a2bd — SPI id() naming
- Protocol PP-20260610-83747b — fail-soft pagination
- Protocol PP-20260609-0625c9 — `@Blocking` on MCP tools
- GitHub #211 — epic issue
