# Teams Inbound via Bot Framework — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #220 — chat-teams: Teams inbound via Bot Framework
**Issue group:** #220

**Goal:** Replace the existing Outgoing Webhooks `TeamsInboundConnector`
with a full Bot Framework integration that handles inbound messages,
reaction events, and slash commands through a single webhook endpoint.

**Architecture:** All Bot Framework classes live in `chat-teams`. A new
`BotFrameworkAuth` handles JWT validation via nimbus-jose-jwt, a new
`BotFrameworkClient` handles outbound REST API calls, and the
`TeamsInboundConnector` (as a `WebhookInboundConnector`) routes
Activity types to the appropriate handler. The old connector in `webhook`
is removed atomically.

**Tech Stack:** Java 21, Quarkus 3.32.2, nimbus-jose-jwt, jakarta.json,
java.net.http.HttpClient

## Global Constraints

- Java 21 source level, Java 26 JVM
- All casehubio artifacts are `0.4-SNAPSHOT`
- `WebhookInboundConnector` IDs must match `[a-z0-9][a-z0-9\-]*`
- `WebhookInboundConnector.handle()` must not throw
- Connector ID reuse: `InboundConnectorIds.TEAMS_INBOUND` / `InboundConnectorTypes.TEAMS`
- No new Quarkus-specific dependencies (Spring parity via `connectors-spring`)
- `HttpHelper.CLIENT` for all outbound HTTP
- SSRF defense on all outbound `serviceUrl` calls
- Use `ServiceConnectionProvider` for outbound auth tokens
- TDD: write failing test → verify fail → implement → verify pass → commit

---

## Batch 1: BotFrameworkAuth — JWT validation

### Task 1: BotFrameworkAuth with dev-mode and JWT validation

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkAuth.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/BotFrameworkAuthTest.java`
- Modify: `chat-teams/pom.xml` (add nimbus-jose-jwt dependency)

**Interfaces:**
- Consumes: nothing
- Produces: `BotFrameworkAuth(String appId, boolean devMode, String metadataUrl, String issuer)` with `boolean validate(String authorizationHeader, String activityServiceUrl)`

- [ ] **Step 1: Add nimbus-jose-jwt dependency to chat-teams/pom.xml**

Add to `<dependencies>` in `chat-teams/pom.xml`:

```xml
<dependency>
  <groupId>com.nimbusds</groupId>
  <artifactId>nimbus-jose-jwt</artifactId>
  <version>10.3</version>
</dependency>
```

- [ ] **Step 2: Write failing test — dev-mode bypass**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class BotFrameworkAuthTest {

    @Test
    void devMode_skipsValidation() {
        var auth = new BotFrameworkAuth("test-app-id", true,
                "https://login.botframework.com/v1/.well-known/openidconfiguration",
                "https://api.botframework.com");
        assertThat(auth.validate(null, "https://smba.trafficmanager.net/teams/"))
                .isTrue();
    }

    @Test
    void devMode_false_nullHeader_returnsFalse() {
        var auth = new BotFrameworkAuth("test-app-id", false,
                "https://login.botframework.com/v1/.well-known/openidconfiguration",
                "https://api.botframework.com");
        assertThat(auth.validate(null, "https://smba.trafficmanager.net/teams/"))
                .isFalse();
    }

    @Test
    void devMode_false_emptyHeader_returnsFalse() {
        var auth = new BotFrameworkAuth("test-app-id", false,
                "https://login.botframework.com/v1/.well-known/openidconfiguration",
                "https://api.botframework.com");
        assertThat(auth.validate("", "https://smba.trafficmanager.net/teams/"))
                .isFalse();
    }

    @Test
    void devMode_false_noBearerPrefix_returnsFalse() {
        var auth = new BotFrameworkAuth("test-app-id", false,
                "https://login.botframework.com/v1/.well-known/openidconfiguration",
                "https://api.botframework.com");
        assertThat(auth.validate("Basic abc123", "https://smba.trafficmanager.net/teams/"))
                .isFalse();
    }

    @Test
    void devMode_false_malformedJwt_returnsFalse() {
        var auth = new BotFrameworkAuth("test-app-id", false,
                "https://login.botframework.com/v1/.well-known/openidconfiguration",
                "https://api.botframework.com");
        assertThat(auth.validate("Bearer not.a.jwt", "https://smba.trafficmanager.net/teams/"))
                .isFalse();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=BotFrameworkAuthTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — class `BotFrameworkAuth` does not exist

- [ ] **Step 4: Implement BotFrameworkAuth**

```java
package io.casehub.connectors.chat.teams;

import com.nimbusds.jose.JWSVerifier;
import com.nimbusds.jose.crypto.RSASSAVerifier;
import com.nimbusds.jose.jwk.JWK;
import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jose.jwk.RSAKey;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;
import io.casehub.connectors.http.HttpHelper;

import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Instant;
import java.util.Date;
import java.util.concurrent.locks.ReentrantLock;
import java.util.logging.Level;
import java.util.logging.Logger;

class BotFrameworkAuth {

    private static final Logger LOG = Logger.getLogger(BotFrameworkAuth.class.getName());
    private static final long CACHE_DURATION_MS = 24 * 60 * 60 * 1000L;
    private static final long REFRESH_COOLDOWN_MS = 5 * 60 * 1000L;
    private static final long CLOCK_SKEW_SECONDS = 300;

    private final String appId;
    private final boolean devMode;
    private final String metadataUrl;
    private final String expectedIssuer;

    private volatile JWKSet cachedKeys;
    private volatile long cacheTimestamp;
    private volatile long lastRefreshAttempt;
    private final ReentrantLock refreshLock = new ReentrantLock();

    BotFrameworkAuth(String appId, boolean devMode, String metadataUrl, String expectedIssuer) {
        this.appId = appId;
        this.devMode = devMode;
        this.metadataUrl = metadataUrl;
        this.expectedIssuer = expectedIssuer;
    }

    boolean validate(String authorizationHeader, String activityServiceUrl) {
        if (devMode) {
            LOG.warning("SECURITY WARNING: teams-bot dev-mode enabled — skipping JWT validation");
            return true;
        }

        if (authorizationHeader == null || authorizationHeader.isBlank()) return false;
        if (!authorizationHeader.startsWith("Bearer ")) return false;

        String token = authorizationHeader.substring("Bearer ".length()).trim();

        try {
            SignedJWT jwt = SignedJWT.parse(token);
            String kid = jwt.getHeader().getKeyID();
            if (kid == null) return false;

            RSAKey rsaKey = findKey(kid);
            if (rsaKey == null) return false;

            JWSVerifier verifier = new RSASSAVerifier(rsaKey);
            if (!jwt.verify(verifier)) return false;

            JWTClaimsSet claims = jwt.getJWTClaimsSet();

            String issuer = claims.getIssuer();
            if (!expectedIssuer.equals(issuer)) return false;

            java.util.List<String> audience = claims.getAudience();
            if (audience == null || !audience.contains(appId)) return false;

            Date expiration = claims.getExpirationTime();
            if (expiration == null) return false;
            Instant expInstant = expiration.toInstant();
            if (Instant.now().isAfter(expInstant.plusSeconds(CLOCK_SKEW_SECONDS))) return false;

            Object serviceUrlClaim = claims.getClaim("serviceurl");
            if (serviceUrlClaim != null && activityServiceUrl != null
                    && !activityServiceUrl.equals(serviceUrlClaim.toString())) {
                return false;
            }

            return true;
        } catch (Exception e) {
            LOG.log(Level.WARNING, "teams-bot: JWT validation failed", e);
            return false;
        }
    }

    private RSAKey findKey(String kid) {
        JWKSet keys = getKeys();
        if (keys == null) return null;

        JWK jwk = keys.getKeyByKeyId(kid);
        if (jwk == null) {
            keys = refreshKeys(kid);
            if (keys == null) return null;
            jwk = keys.getKeyByKeyId(kid);
        }
        return (jwk instanceof RSAKey rsa) ? rsa : null;
    }

    private JWKSet getKeys() {
        if (cachedKeys != null && (System.currentTimeMillis() - cacheTimestamp) < CACHE_DURATION_MS) {
            return cachedKeys;
        }
        return refreshKeys(null);
    }

    private JWKSet refreshKeys(String triggeringKid) {
        long now = System.currentTimeMillis();
        if (triggeringKid != null && (now - lastRefreshAttempt) < REFRESH_COOLDOWN_MS) {
            LOG.warning("teams-bot: JWKS refresh rate-limited — rejecting unknown kid: " + triggeringKid);
            return cachedKeys;
        }

        if (!refreshLock.tryLock()) return cachedKeys;
        try {
            if (cachedKeys != null && triggeringKid != null
                    && cachedKeys.getKeyByKeyId(triggeringKid) != null) {
                return cachedKeys;
            }

            String jwksUrl = fetchJwksUrl();
            if (jwksUrl == null) return cachedKeys;

            var request = HttpRequest.newBuilder()
                    .uri(URI.create(jwksUrl))
                    .GET()
                    .build();
            var response = HttpHelper.CLIENT.send(request,
                    HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() != 200) return cachedKeys;

            cachedKeys = JWKSet.parse(response.body());
            cacheTimestamp = System.currentTimeMillis();
            lastRefreshAttempt = cacheTimestamp;
            return cachedKeys;
        } catch (Exception e) {
            LOG.log(Level.WARNING, "teams-bot: failed to refresh JWKS", e);
            return cachedKeys;
        } finally {
            refreshLock.unlock();
        }
    }

    private String fetchJwksUrl() {
        try {
            var request = HttpRequest.newBuilder()
                    .uri(URI.create(metadataUrl))
                    .GET()
                    .build();
            var response = HttpHelper.CLIENT.send(request,
                    HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() != 200) return null;

            var json = jakarta.json.Json.createReader(
                    new java.io.StringReader(response.body())).readObject();
            return json.getString("jwks_uri", null);
        } catch (Exception e) {
            LOG.log(Level.WARNING, "teams-bot: failed to fetch OpenID metadata", e);
            return null;
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=BotFrameworkAuthTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — all 5 tests

- [ ] **Step 6: Commit**

```bash
git add chat-teams/pom.xml chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkAuth.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/BotFrameworkAuthTest.java
git commit -m "feat(#220): add BotFrameworkAuth with JWT validation and dev-mode"
```

---

## Batch 2: BotFrameworkClient — outbound REST API with SSRF defense

### Task 2: BotFrameworkClient with SSRF defense

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkClient.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/BotFrameworkClientTest.java`

**Interfaces:**
- Consumes: `ServiceConnectionProvider` (from `casehub-platform-api`), `HttpHelper.CLIENT`
- Produces: `BotFrameworkClient(ServiceConnectionProvider connectionProvider, String serviceUrlAllowlist)` with `replyToActivity(String serviceUrl, String conversationId, String activityId, String text)` and `sendToConversation(String serviceUrl, String conversationId, String text)`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import org.junit.jupiter.api.Test;

class BotFrameworkClientTest {

    @Test
    void validateServiceUrl_allowedHost_passes() {
        var client = new BotFrameworkClient(null,
                "*.botframework.com,smba.trafficmanager.net");
        // Should not throw
        client.validateServiceUrl("https://smba.trafficmanager.net/teams/");
    }

    @Test
    void validateServiceUrl_allowedWildcard_passes() {
        var client = new BotFrameworkClient(null,
                "*.botframework.com,smba.trafficmanager.net");
        client.validateServiceUrl("https://us.botframework.com/");
    }

    @Test
    void validateServiceUrl_blockedHost_throws() {
        var client = new BotFrameworkClient(null,
                "*.botframework.com,smba.trafficmanager.net");
        assertThatThrownBy(() ->
                client.validateServiceUrl("https://evil.example.com/"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("evil.example.com");
    }

    @Test
    void validateServiceUrl_noScheme_throws() {
        var client = new BotFrameworkClient(null,
                "*.botframework.com,smba.trafficmanager.net");
        assertThatThrownBy(() ->
                client.validateServiceUrl("not-a-url"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=BotFrameworkClientTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — class `BotFrameworkClient` does not exist

- [ ] **Step 3: Implement BotFrameworkClient**

```java
package io.casehub.connectors.chat.teams;

import io.casehub.connectors.http.HttpHelper;
import io.casehub.platform.api.authn.ServiceConnectionProvider;

import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

import jakarta.json.Json;

class BotFrameworkClient {

    private static final Logger LOG = Logger.getLogger(BotFrameworkClient.class.getName());
    static final String APPLICATION_ACTOR = "__application__";

    private final ServiceConnectionProvider connectionProvider;
    private final List<String> allowedHostPatterns;

    BotFrameworkClient(ServiceConnectionProvider connectionProvider, String serviceUrlAllowlist) {
        this.connectionProvider = connectionProvider;
        this.allowedHostPatterns = List.of(serviceUrlAllowlist.split(","));
    }

    void replyToActivity(String serviceUrl, String conversationId, String activityId, String text) {
        validateServiceUrl(serviceUrl);
        String url = serviceUrl.replaceAll("/+$", "")
                + "/v3/conversations/" + conversationId + "/activities/" + activityId;
        String body = Json.createObjectBuilder()
                .add("type", "message")
                .add("text", text)
                .build().toString();
        postJson(url, body);
    }

    void sendToConversation(String serviceUrl, String conversationId, String text) {
        validateServiceUrl(serviceUrl);
        String url = serviceUrl.replaceAll("/+$", "")
                + "/v3/conversations/" + conversationId + "/activities";
        String body = Json.createObjectBuilder()
                .add("type", "message")
                .add("text", text)
                .build().toString();
        postJson(url, body);
    }

    void validateServiceUrl(String serviceUrl) {
        try {
            URI uri = URI.create(serviceUrl);
            String host = uri.getHost();
            if (host == null) {
                throw new IllegalArgumentException("teams-bot: no host in serviceUrl: " + serviceUrl);
            }
            for (String pattern : allowedHostPatterns) {
                String p = pattern.trim();
                if (p.startsWith("*.")) {
                    String suffix = p.substring(1);
                    if (host.endsWith(suffix) || host.equals(p.substring(2))) return;
                } else {
                    if (host.equals(p)) return;
                }
            }
            throw new IllegalArgumentException(
                    "teams-bot: serviceUrl host not in allowlist: " + host);
        } catch (IllegalArgumentException e) {
            throw e;
        } catch (Exception e) {
            throw new IllegalArgumentException(
                    "teams-bot: invalid serviceUrl: " + serviceUrl, e);
        }
    }

    private void postJson(String url, String body) {
        try {
            String token = resolveToken();
            var request = HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .header("Content-Type", "application/json")
                    .header("Authorization", "Bearer " + token)
                    .POST(HttpRequest.BodyPublishers.ofString(body, StandardCharsets.UTF_8))
                    .build();
            var response = HttpHelper.CLIENT.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() < 200 || response.statusCode() >= 300) {
                LOG.warning("teams-bot: outbound call failed: " + response.statusCode()
                        + " " + url);
            }
        } catch (Exception e) {
            LOG.log(Level.WARNING, "teams-bot: outbound call failed: " + url, e);
        }
    }

    private String resolveToken() {
        if (connectionProvider == null) return "test-token";
        return connectionProvider.getAccessToken(APPLICATION_ACTOR, "microsoft-bot",
                "default").accessToken();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=BotFrameworkClientTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — all 4 tests

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkClient.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/BotFrameworkClientTest.java
git commit -m "feat(#220): add BotFrameworkClient with SSRF defense"
```

---

## Batch 3: TeamsInboundTranslator + TeamsCommands

### Task 3: TeamsInboundTranslator

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundTranslator.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundTranslatorTest.java`

**Interfaces:**
- Consumes: `InboundTranslator` SPI, `InboundConnectorTypes.TEAMS`, `InboundMessage`, `ReceivedMessage`, `ChatContent`, `ChatChannelRef`, `ChatMessageRef`, `MemberRef`
- Produces: `TeamsInboundTranslator` implementing `InboundTranslator` — `connectorType()` returns `"teams"`, `translate(InboundMessage)` returns `ReceivedMessage`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;

class TeamsInboundTranslatorTest {

    private final TeamsInboundTranslator translator = new TeamsInboundTranslator();

    @Test
    void connectorType_isTeams() {
        assertThat(translator.connectorType()).isEqualTo(InboundConnectorTypes.TEAMS);
    }

    @Test
    void translate_channelMessage_mapsAllFields() {
        var msg = new InboundMessage(
                InboundConnectorIds.TEAMS_INBOUND,
                InboundConnectorTypes.TEAMS,
                "user-aad-id",
                "team-id-1:channel-id-1",
                "Hello from Teams",
                List.of(),
                Instant.parse("2026-11-15T10:00:00Z"),
                Map.of(
                        "teams-activity-id", "act-123",
                        "teams-activity-type", "message",
                        "teams-service-url", "https://smba.trafficmanager.net/teams/",
                        "teams-conversation-id", "conv-123",
                        "teams-conversation-type", "channel"),
                "tenant-1");

        var result = translator.translate(msg);

        assertThat(result.platformId()).isEqualTo("microsoft-teams");
        assertThat(result.channel().id()).isEqualTo("team-id-1:channel-id-1");
        assertThat(result.message().messageId()).isEqualTo("act-123");
        assertThat(result.sender().id()).isEqualTo("user-aad-id");
        assertThat(result.content().text()).isEqualTo("Hello from Teams");
        assertThat(result.parent()).isNull();
    }

    @Test
    void translate_withReplyToId_setsParent() {
        var msg = new InboundMessage(
                InboundConnectorIds.TEAMS_INBOUND,
                InboundConnectorTypes.TEAMS,
                "user-1",
                "team-1:ch-1",
                "reply text",
                List.of(),
                Instant.now(),
                Map.of(
                        "teams-activity-id", "act-456",
                        "teams-activity-type", "message",
                        "teams-service-url", "https://smba.trafficmanager.net/teams/",
                        "teams-conversation-id", "conv-1",
                        "teams-conversation-type", "channel",
                        "teams-reply-to-id", "act-123"),
                null);

        var result = translator.translate(msg);

        assertThat(result.parent()).isNotNull();
        assertThat(result.parent().messageId()).isEqualTo("act-123");
    }

    @Test
    void translate_personalChat_usesConversationId() {
        var msg = new InboundMessage(
                InboundConnectorIds.TEAMS_INBOUND,
                InboundConnectorTypes.TEAMS,
                "user-1",
                "personal:conv-personal-123",
                "DM text",
                List.of(),
                Instant.now(),
                Map.of(
                        "teams-activity-id", "act-789",
                        "teams-activity-type", "message",
                        "teams-service-url", "https://smba.trafficmanager.net/teams/",
                        "teams-conversation-id", "conv-personal-123",
                        "teams-conversation-type", "personal"),
                null);

        var result = translator.translate(msg);
        assertThat(result.channel().id()).isEqualTo("personal:conv-personal-123");
    }

    @Test
    void translate_nonMessageActivity_throwsIllegalArgument() {
        var msg = new InboundMessage(
                InboundConnectorIds.TEAMS_INBOUND,
                InboundConnectorTypes.TEAMS,
                "user-1",
                "team-1:ch-1",
                "",
                List.of(),
                Instant.now(),
                Map.of(
                        "teams-activity-id", "act-react",
                        "teams-activity-type", "messageReaction",
                        "teams-service-url", "https://smba.trafficmanager.net/teams/",
                        "teams-conversation-id", "conv-1",
                        "teams-conversation-type", "channel"),
                null);

        assertThatThrownBy(() -> translator.translate(msg))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("messageReaction");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundTranslatorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — class `TeamsInboundTranslator` does not exist

- [ ] **Step 3: Implement TeamsInboundTranslator**

```java
package io.casehub.connectors.chat.teams;

import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

public class TeamsInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return InboundConnectorTypes.TEAMS;
    }

    @Override
    public ReceivedMessage translate(InboundMessage msg) {
        String activityType = msg.metadata().get("teams-activity-type");
        if (!"message".equals(activityType)) {
            throw new IllegalArgumentException(
                    "Non-message activity type: " + activityType
                    + " — not translatable to ReceivedMessage");
        }

        var channel = new ChatChannelRef(msg.externalChannelRef());
        String activityId = msg.metadata().get("teams-activity-id");
        var messageRef = new ChatMessageRef(channel, activityId);

        String replyToId = msg.metadata().get("teams-reply-to-id");
        ChatMessageRef parentRef = replyToId != null
                ? new ChatMessageRef(channel, replyToId) : null;

        return new ReceivedMessage(
                TeamsChatPlatform.PLATFORM_ID,
                channel,
                messageRef,
                parentRef,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundTranslatorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — all 4 tests

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundTranslator.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundTranslatorTest.java
git commit -m "feat(#220): add TeamsInboundTranslator"
```

### Task 4: TeamsCommands

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsCommands.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsCommandsTest.java`

**Interfaces:**
- Consumes: `Commands` SPI, `CommandDefinition`
- Produces: `TeamsCommands` implementing `Commands`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.command.CommandDefinition;

class TeamsCommandsTest {

    @Test
    void registerAll_doesNotThrow() {
        var cmds = new TeamsCommands();
        cmds.registerAll(List.of(
                new CommandDefinition("help", "Shows help", List.of()),
                new CommandDefinition("status", "Shows status", List.of())));
        // Verify it doesn't throw — registration is logging only
    }

    @Test
    void registerAll_emptyList_doesNotThrow() {
        var cmds = new TeamsCommands();
        cmds.registerAll(List.of());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsCommandsTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — class `TeamsCommands` does not exist

- [ ] **Step 3: Implement TeamsCommands**

```java
package io.casehub.connectors.chat.teams;

import io.casehub.connectors.chat.command.CommandDefinition;
import io.casehub.connectors.chat.spi.Commands;

import java.util.List;
import java.util.logging.Logger;

public class TeamsCommands implements Commands {

    private static final Logger LOG = Logger.getLogger(TeamsCommands.class.getName());

    @Override
    public void registerAll(List<CommandDefinition> commands) {
        LOG.info("teams: " + commands.size()
                + " commands available (register via Azure Bot app manifest)");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsCommandsTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — both tests

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsCommands.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsCommandsTest.java
git commit -m "feat(#220): add TeamsCommands (Commands SPI)"
```

---

## Batch 4: TeamsInboundConnector — activity routing and command dispatch

### Task 5: TeamsInboundConnector with full activity routing

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java`

**Interfaces:**
- Consumes: `BotFrameworkAuth`, `BotFrameworkClient`, `CommandService`, `WebhookInboundConnector`, `WebhookRequest`, `WebhookResult`, `InboundMessage`, `InboundConnectorIds`, `InboundConnectorTypes`
- Produces: `TeamsInboundConnector(BotFrameworkAuth auth, BotFrameworkClient client, CommandService commandService, String appId)` extending `WebhookInboundConnector`

- [ ] **Step 1: Add webhook dependency to chat-teams/pom.xml**

Add to `<dependencies>` in `chat-teams/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-connectors-webhook</artifactId>
  <version>${project.version}</version>
</dependency>
```

Note: This is needed for `WebhookInboundConnector` base class. If this creates a circular dependency issue (webhook already depends on something that chat-teams needs), we may need to extract `WebhookInboundConnector` to `connectors-api`. Check: `WebhookInboundConnector` is already in `connectors-api` — no new dependency needed. Skip this step if `connectors-api` already has the base class.

Actually, `WebhookInboundConnector` is in `connectors-api` (confirmed from file path `connectors-api/src/main/java/io/casehub/connectors/WebhookInboundConnector.java`). No additional dependency needed.

- [ ] **Step 2: Write failing tests**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.HttpMethod;
import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;
import io.casehub.connectors.chat.command.CommandService;

class TeamsInboundConnectorTest {

    private TeamsInboundConnector connector;

    @BeforeEach
    void setUp() {
        var auth = new BotFrameworkAuth("test-app-id", true,
                "https://login.botframework.com/v1/.well-known/openidconfiguration",
                "https://api.botframework.com");
        var client = new BotFrameworkClient(null,
                "*.botframework.com,smba.trafficmanager.net");
        var commandService = new CommandService(List.of(), List.of());
        connector = new TeamsInboundConnector(auth, client, commandService, "test-app-id");
    }

    @Test
    void id_isTeamsInbound() {
        assertThat(connector.id()).isEqualTo(InboundConnectorIds.TEAMS_INBOUND);
    }

    @Test
    void getRequest_returnsIgnored() {
        var req = new WebhookRequest("", Map.of(), Map.of(), HttpMethod.GET,
                "https://test.example.com/connectors/teams-inbound/webhook");
        assertThat(connector.handle(req)).isInstanceOf(WebhookResult.Ignored.class);
    }

    @Test
    void messageActivity_returnsDelivered() {
        String body = """
                {"type":"message","text":"Hello bot",\
                "from":{"id":"29:user-id","aadObjectId":"aad-user-1","name":"User"},\
                "conversation":{"id":"conv-1","conversationType":"channel"},\
                "channelData":{"team":{"id":"team-1"},"channel":{"id":"ch-1"},"tenant":{"id":"t-1"}},\
                "serviceUrl":"https://smba.trafficmanager.net/teams/",\
                "id":"act-1","timestamp":"2026-11-15T10:00:00Z"}""";
        var req = postRequest(body);
        var result = connector.handle(req);

        assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
        var delivered = (WebhookResult.Delivered) result;
        assertThat(delivered.messages()).hasSize(1);
        assertThat(delivered.messages().get(0).content()).isEqualTo("Hello bot");
        assertThat(delivered.messages().get(0).externalSenderId()).isEqualTo("aad-user-1");
        assertThat(delivered.messages().get(0).externalChannelRef()).isEqualTo("team-1:ch-1");
        assertThat(delivered.messages().get(0).tenancyId()).isEqualTo("t-1");
    }

    @Test
    void messageReactionActivity_flattensReactions() {
        String body = """
                {"type":"messageReaction",\
                "reactionsAdded":[{"type":"like"},{"type":"heart"}],\
                "replyToId":"msg-123",\
                "from":{"id":"29:user-id","aadObjectId":"aad-user-1"},\
                "conversation":{"id":"conv-1","conversationType":"channel"},\
                "channelData":{"team":{"id":"team-1"},"channel":{"id":"ch-1"},"tenant":{"id":"t-1"}},\
                "serviceUrl":"https://smba.trafficmanager.net/teams/",\
                "id":"act-2","timestamp":"2026-11-15T10:00:00Z"}""";
        var req = postRequest(body);
        var result = connector.handle(req);

        assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
        var delivered = (WebhookResult.Delivered) result;
        assertThat(delivered.messages()).hasSize(2);
        assertThat(delivered.messages().get(0).metadata().get("teams-reaction-type")).isEqualTo("like");
        assertThat(delivered.messages().get(0).metadata().get("teams-reaction-action")).isEqualTo("added");
        assertThat(delivered.messages().get(1).metadata().get("teams-reaction-type")).isEqualTo("heart");
    }

    @Test
    void conversationUpdateActivity_returnsIgnored() {
        String body = """
                {"type":"conversationUpdate",\
                "membersAdded":[{"id":"29:new-user"}],\
                "from":{"id":"29:user-id","aadObjectId":"aad-user-1"},\
                "conversation":{"id":"conv-1","conversationType":"channel"},\
                "channelData":{"team":{"id":"team-1"},"channel":{"id":"ch-1"},"tenant":{"id":"t-1"}},\
                "serviceUrl":"https://smba.trafficmanager.net/teams/",\
                "id":"act-3","timestamp":"2026-11-15T10:00:00Z"}""";
        var req = postRequest(body);
        assertThat(connector.handle(req)).isInstanceOf(WebhookResult.Ignored.class);
    }

    @Test
    void selfMessage_returnsIgnored() {
        String body = """
                {"type":"message","text":"Bot echo",\
                "from":{"id":"test-app-id","aadObjectId":"test-app-id"},\
                "conversation":{"id":"conv-1","conversationType":"channel"},\
                "channelData":{"team":{"id":"team-1"},"channel":{"id":"ch-1"},"tenant":{"id":"t-1"}},\
                "serviceUrl":"https://smba.trafficmanager.net/teams/",\
                "id":"act-4","timestamp":"2026-11-15T10:00:00Z"}""";
        var req = postRequest(body);
        assertThat(connector.handle(req)).isInstanceOf(WebhookResult.Ignored.class);
    }

    @Test
    void personalChat_usesPersonalPrefix() {
        String body = """
                {"type":"message","text":"DM",\
                "from":{"id":"29:user-id","aadObjectId":"aad-user-1"},\
                "conversation":{"id":"conv-personal-1","conversationType":"personal"},\
                "channelData":{"tenant":{"id":"t-1"}},\
                "serviceUrl":"https://smba.trafficmanager.net/teams/",\
                "id":"act-5","timestamp":"2026-11-15T10:00:00Z"}""";
        var req = postRequest(body);
        var result = connector.handle(req);

        assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
        var delivered = (WebhookResult.Delivered) result;
        assertThat(delivered.messages().get(0).externalChannelRef())
                .isEqualTo("personal:conv-personal-1");
    }

    @Test
    void nullText_usesEmptyString() {
        String body = """
                {"type":"message",\
                "from":{"id":"29:user-id","aadObjectId":"aad-user-1"},\
                "conversation":{"id":"conv-1","conversationType":"channel"},\
                "channelData":{"team":{"id":"team-1"},"channel":{"id":"ch-1"},"tenant":{"id":"t-1"}},\
                "serviceUrl":"https://smba.trafficmanager.net/teams/",\
                "id":"act-6","timestamp":"2026-11-15T10:00:00Z"}""";
        var req = postRequest(body);
        var result = connector.handle(req);

        assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
        var delivered = (WebhookResult.Delivered) result;
        assertThat(delivered.messages().get(0).content()).isEqualTo("");
    }

    private static WebhookRequest postRequest(String body) {
        return new WebhookRequest(body, Map.of(), Map.of(), HttpMethod.POST,
                "https://test.example.com/connectors/teams-inbound/webhook");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — class `TeamsInboundConnector` (in `chat.teams` package) does not exist

- [ ] **Step 4: Implement TeamsInboundConnector**

Implement the full connector with activity routing, mention stripping, command dispatch, reaction flattening, self-filtering, and personal/channel context handling. This is the largest class. Key behaviors:

1. Parse Activity JSON via `jakarta.json`
2. Validate JWT via `BotFrameworkAuth.validate()`
3. Self-filter: skip if `from.id` matches `appId`
4. Route by `activity.type`:
   - `message`: strip mentions, check for commands, deliver as `InboundMessage`
   - `messageReaction`: flatten `reactionsAdded`/`reactionsRemoved` into individual `InboundMessage`s
   - `conversationUpdate`: log and return `Ignored`
   - all others: return `Ignored`

The implementation is substantial — the test suite above defines the expected behavior for each path. Follow the spec's §Activity Flow diagram exactly.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — all 7 tests

- [ ] **Step 6: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java
git commit -m "feat(#220): add TeamsInboundConnector with activity routing"
```

---

## Batch 5: CDI wiring, old connector removal, and docs

### Task 6: Wire CDI beans and remove old connector atomically

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java` — add producers for `TeamsInboundConnector`, `TeamsInboundTranslator`, `TeamsCommands`; update `ChatPlatform` producer to wire commands
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java` — add `withCommands(Commands)` fluent setter
- Modify: `webhook/src/main/java/io/casehub/connectors/webhook/WebhookBeans.java` — remove Teams producer
- Delete: `webhook/src/main/java/io/casehub/connectors/webhook/TeamsInboundConnector.java` (use `ide_refactor_safe_delete`)
- Delete: `webhook/src/test/java/io/casehub/connectors/webhook/TeamsInboundConnectorTest.java` (use `ide_refactor_safe_delete`)
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java` (existing — verify commands wiring)

**Interfaces:**
- Consumes: All classes from Tasks 1-5
- Produces: Fully wired CDI bean graph

- [ ] **Step 1: Add `withCommands()` to TeamsChatPlatform**

Add a fluent setter before `build()`:

```java
private Commands commands;

public TeamsChatPlatform withCommands(Commands commands) {
    this.commands = commands;
    return this;
}
```

Update `build()` to wire commands:

```java
public ChatPlatform build() {
    var builder = ChatPlatform.builder(PLATFORM_ID)
            .messaging(this::sendMessage)
            .threading(this::replyToMessage)
            .discovery(this::listChannels)
            .presence(new TeamsPresence())
            .members(this::listMembers)
            .channelManagement(new TeamsChannelManagement())
            .memberManagement(new TeamsMemberManagement())
            .messageHistory(this::listMessageHistory);
    if (commands != null) {
        builder.commands(commands);
    }
    return builder.build();
}
```

- [ ] **Step 2: Update TeamsBeans — add new producers and update ChatPlatform producer**

Add producers for `TeamsCommands`, `TeamsInboundConnector`, and `TeamsInboundTranslator` as specified in the spec §CDI Wiring. Update the `ChatPlatform` producer to accept `TeamsCommands` and call `.withCommands(teamsCommands)`.

- [ ] **Step 3: Remove old TeamsInboundConnector from webhook module**

Use `ide_refactor_safe_delete` for:
- `webhook/src/main/java/io/casehub/connectors/webhook/TeamsInboundConnector.java`
- `webhook/src/test/java/io/casehub/connectors/webhook/TeamsInboundConnectorTest.java`

Remove the `teamsInboundConnector` producer method from `WebhookBeans.java`.

- [ ] **Step 4: Run full build to verify no duplicate connector IDs**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams,webhook -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — webhook tests pass (old connector removed), chat-teams tests pass (new connector wired)

- [ ] **Step 5: Commit atomically**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java webhook/src/main/java/io/casehub/connectors/webhook/WebhookBeans.java
git rm webhook/src/main/java/io/casehub/connectors/webhook/TeamsInboundConnector.java webhook/src/test/java/io/casehub/connectors/webhook/TeamsInboundConnectorTest.java
git commit -m "feat(#220): wire CDI beans and remove old TeamsInboundConnector"
```

### Task 7: Update contributor guide and verify full build

**Files:**
- Modify: `docs/guides/contributor-guide.md` — update webhook signature table, commands architecture, webhook connectors list

- [ ] **Step 1: Update contributor-guide.md**

In the webhook signature verification table, replace the Teams row:

| Connector | Algorithm | Notes |
|-----------|-----------|-------|
| `TeamsInboundConnector` | JWT/RS256 via `BotFrameworkAuth` | Azure AD JWKS; lives in `chat-teams` module (not `webhook`); embedded command dispatch |

In the Commands architecture section, add note about Teams' unified webhook transport.

Move Teams from the webhook module connector list to the chat-teams module section.

- [ ] **Step 2: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 3: Commit**

```bash
git add docs/guides/contributor-guide.md
git commit -m "docs(#220): update contributor guide for Bot Framework integration"
```

---

## References

- [2026-11-15-teams-inbound-bot-framework-design.md] — design spec this plan implements
- [webhook/src/main/java/.../webhook/TeamsInboundConnector.java] — existing implementation being replaced
- [webhook/src/main/java/.../webhook/WebhookBeans.java:19-22] — Teams producer to remove
- [chat-teams/src/main/java/.../chat/teams/TeamsChatPlatform.java] — platform builder to extend
- [chat-teams/src/main/java/.../chat/teams/TeamsBeans.java] — CDI producers to extend
- [chat-slack/src/main/java/.../chat/slack/SlackCommands.java] — Commands pattern reference
- [chat-discord/src/main/java/.../chat/discord/DiscordInboundTranslator.java] — translator pattern reference
- [chat-spi/src/main/java/.../chat/command/CommandService.java] — dispatch + definitions API
- [connectors-api/src/main/java/.../WebhookInboundConnector.java] — base class contract
- [connectors-api/src/main/java/.../http/HttpHelper.java:26-28] — shared HTTP client
- [GitHub #220] — focal issue
