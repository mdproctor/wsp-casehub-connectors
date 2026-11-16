# Teams Proactive Messaging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #234 — Proactive messaging (bot-initiated conversations)
**Issue group:** #234

**Goal:** Enable the Teams ChatPlatform bot to initiate 1:1 conversations with users who have not previously messaged it, and capture ConversationReference data from inbound conversationUpdate activities.

**Architecture:** Add `createConversation()` to `BotFrameworkClient` for the `POST /v3/conversations` API. Create a `ConversationReferenceStore` CDI SPI (with `InMemoryConversationReferenceStore` default) to cache conversation refs. Extend `TeamsChatPlatform.sendMessage()` with a `user:<aadObjectId>` channel prefix that resolves via the store or cold-starts a new conversation. Handle `conversationUpdate` activities in `TeamsInboundConnector` to populate the store and fire CDI events.

**Tech Stack:** Java 21, Quarkus 3.32.2, Bot Framework REST API, jakarta.json, CDI

## Global Constraints

- Java source level 21 on Java 26 JVM
- All code in `chat-teams` module (package `io.casehub.connectors.chat.teams`)
- No business logic — pure delivery infrastructure
- Tests use JUnit 5 + AssertJ (no mocking frameworks — use StubHttpResponse pattern)
- `BotFrameworkClient` uses `HttpHelper.CLIENT` for HTTP calls
- Config properties under `casehub.connectors.teams-bot.*` namespace

---

## Batch 1: ConversationReferenceStore SPI and createConversation

### Task 1: ConversationReferenceStore SPI + InMemoryConversationReferenceStore

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ConversationReference.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ConversationReferenceStore.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InMemoryConversationReferenceStore.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/InMemoryConversationReferenceStoreTest.java`

**Interfaces:**
- Produces: `ConversationReference(String aadObjectId, String conversationId, String serviceUrl, String tenantId)` — record
- Produces: `ConversationReferenceStore.find(String aadObjectId)` → `Optional<ConversationReference>`
- Produces: `ConversationReferenceStore.store(ConversationReference ref)` → `void`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.connectors.chat.teams;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class InMemoryConversationReferenceStoreTest {

    private final InMemoryConversationReferenceStore store = new InMemoryConversationReferenceStore();

    @Test
    void find_unknownUser_returnsEmpty() {
        assertThat(store.find("unknown-aad-id")).isEmpty();
    }

    @Test
    void storeThenFind_returnsRef() {
        var ref = new ConversationReference("aad-1", "conv-1",
                "https://smba.trafficmanager.net/teams/", "tenant-1");
        store.store(ref);
        var found = store.find("aad-1");
        assertThat(found).isPresent();
        assertThat(found.get().conversationId()).isEqualTo("conv-1");
        assertThat(found.get().serviceUrl()).isEqualTo("https://smba.trafficmanager.net/teams/");
        assertThat(found.get().tenantId()).isEqualTo("tenant-1");
    }

    @Test
    void store_overwritesExistingRef() {
        var ref1 = new ConversationReference("aad-1", "conv-1",
                "https://service1.botframework.com/", "tenant-1");
        var ref2 = new ConversationReference("aad-1", "conv-2",
                "https://service2.botframework.com/", "tenant-1");
        store.store(ref1);
        store.store(ref2);
        var found = store.find("aad-1");
        assertThat(found).isPresent();
        assertThat(found.get().conversationId()).isEqualTo("conv-2");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InMemoryConversationReferenceStoreTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation errors (classes don't exist)

- [ ] **Step 3: Create ConversationReference record**

```java
package io.casehub.connectors.chat.teams;

public record ConversationReference(
    String aadObjectId,
    String conversationId,
    String serviceUrl,
    String tenantId
) {}
```

- [ ] **Step 4: Create ConversationReferenceStore interface**

```java
package io.casehub.connectors.chat.teams;

import java.util.Optional;

public interface ConversationReferenceStore {
    Optional<ConversationReference> find(String aadObjectId);
    void store(ConversationReference ref);
}
```

- [ ] **Step 5: Create InMemoryConversationReferenceStore**

```java
package io.casehub.connectors.chat.teams;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class InMemoryConversationReferenceStore implements ConversationReferenceStore {

    private static final Logger LOG = Logger.getLogger(InMemoryConversationReferenceStore.class.getName());

    private final ConcurrentHashMap<String, ConversationReference> store = new ConcurrentHashMap<>();

    public InMemoryConversationReferenceStore() {
        LOG.warning("teams-bot: using in-memory ConversationReferenceStore — "
                + "conversation references will be lost on restart. "
                + "Provide a persistent ConversationReferenceStore via CDI for reliable proactive messaging.");
    }

    @Override
    public Optional<ConversationReference> find(String aadObjectId) {
        return Optional.ofNullable(store.get(aadObjectId));
    }

    @Override
    public void store(ConversationReference ref) {
        store.put(ref.aadObjectId(), ref);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InMemoryConversationReferenceStoreTest`
Expected: 3 tests PASS

- [ ] **Step 7: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/ConversationReference.java chat-teams/src/main/java/io/casehub/connectors/chat/teams/ConversationReferenceStore.java chat-teams/src/main/java/io/casehub/connectors/chat/teams/InMemoryConversationReferenceStore.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/InMemoryConversationReferenceStoreTest.java
git commit -m "feat(#234): ConversationReferenceStore SPI with InMemory default"
```

### Task 2: BotFrameworkClient.createConversation

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkClient.java` (add `createConversation` method)
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/BotFrameworkClientTest.java` (add tests)

**Interfaces:**
- Consumes: `validateServiceUrl(String)`, `resolveToken()`, `stripTrailingSlash(String)` — existing methods in BotFrameworkClient
- Produces: `createConversation(String serviceUrl, String botAppId, String aadObjectId, String tenantId)` → `String` (returns conversationId)

- [ ] **Step 1: Write the failing tests**

Add to `BotFrameworkClientTest.java`:

```java
@Test
void createConversation_returnsConversationId() {
    var responses = new java.util.ArrayDeque<StubHttpResponse>();
    responses.add(new StubHttpResponse(201, """
            {"id":"new-conv-123"}"""));
    var client = new BotFrameworkClient(null, DEFAULT_ALLOWLIST) {
        @Override
        String postJsonWithResponse(String url, String body) {
            var resp = responses.poll();
            assertThat(url).contains("/v3/conversations");
            assertThat(body).contains("\"28:bot-app-id\"");
            assertThat(body).contains("\"aad-user-1\"");
            assertThat(body).contains("\"tenant-1\"");
            return resp.body();
        }
    };
    String convId = client.createConversation(
            "https://smba.trafficmanager.net/teams/",
            "bot-app-id", "aad-user-1", "tenant-1");
    assertThat(convId).isEqualTo("new-conv-123");
}

@Test
void createConversation_blockedHost_throws() {
    var client = new BotFrameworkClient(null, DEFAULT_ALLOWLIST);
    assertThatThrownBy(() ->
            client.createConversation("https://evil.example.com/",
                    "bot-app-id", "aad-user-1", "tenant-1"))
            .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=BotFrameworkClientTest`
Expected: compilation error (createConversation and postJsonWithResponse don't exist)

- [ ] **Step 3: Add postJsonWithResponse and createConversation to BotFrameworkClient**

Add `postJsonWithResponse` method (extracted from `postJson` to allow reuse and testability):

```java
String postJsonWithResponse(String url, String body) {
    try {
        String token = resolveToken();
        var request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer " + token)
                .POST(HttpRequest.BodyPublishers.ofString(body, StandardCharsets.UTF_8))
                .build();
        var response = HttpHelper.CLIENT.send(request,
                HttpResponse.BodyHandlers.ofString());
        if (response.statusCode() < 200 || response.statusCode() >= 300) {
            throw new RuntimeException("teams-bot: outbound call failed: "
                    + response.statusCode() + " " + url + " body: " + response.body());
        }
        return response.body();
    } catch (RuntimeException e) {
        throw e;
    } catch (Exception e) {
        throw new RuntimeException("teams-bot: outbound call failed: " + url, e);
    }
}
```

Add `createConversation` method:

```java
String createConversation(String serviceUrl, String botAppId,
                          String aadObjectId, String tenantId) {
    validateServiceUrl(serviceUrl);
    String url = stripTrailingSlash(serviceUrl) + "/v3/conversations";
    String body = Json.createObjectBuilder()
            .add("bot", Json.createObjectBuilder().add("id", "28:" + botAppId))
            .add("members", Json.createArrayBuilder()
                    .add(Json.createObjectBuilder().add("id", aadObjectId)))
            .add("tenantId", tenantId)
            .add("isGroup", false)
            .build().toString();
    String responseBody = postJsonWithResponse(url, body);
    try (var reader = Json.createReader(new java.io.StringReader(responseBody))) {
        return reader.readObject().getString("id");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=BotFrameworkClientTest`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkClient.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/BotFrameworkClientTest.java
git commit -m "feat(#234): BotFrameworkClient.createConversation for proactive messaging"
```

## Batch 2: Proactive sendMessage routing + conversationUpdate handling

### Task 3: TeamsChatPlatform user: prefix routing

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java` (add `user:` routing, store field, builder methods)
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java` (add config properties, wire store)
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java` (add proactive messaging tests)

**Interfaces:**
- Consumes: `ConversationReferenceStore.find(String)`, `ConversationReferenceStore.store(ConversationReference)` — from Task 1
- Consumes: `BotFrameworkClient.createConversation(String, String, String, String)` → `String` — from Task 2
- Consumes: `BotFrameworkClient.sendToConversation(String, String, String)` — existing
- Produces: `TeamsChatPlatform.withConversationReferenceStore(ConversationReferenceStore)` → `TeamsChatPlatform`
- Produces: `TeamsChatPlatform.withProactiveConfig(String defaultTenantId, String botAppId)` → `TeamsChatPlatform`
- Produces: `isUserChannel(String channelId)` → `boolean` (static, checks `user:` prefix)
- Produces: `extractAadObjectId(String channelId)` → `String` (static, strips `user:` prefix)

- [ ] **Step 1: Write the failing tests**

Add a `@Nested class ProactiveMessagingTest` in `TeamsChatPlatformTest.java`:

```java
@Nested
class ProactiveMessagingTest {

    private final java.util.List<String> botCalls = new java.util.ArrayList<>();
    private final InMemoryConversationReferenceStore store = new InMemoryConversationReferenceStore();

    private ChatPlatform buildProactivePlatform() {
        var graph = new MicrosoftGraphClient(request ->
                new StubHttpResponse(200, """
                        {"value":[]}"""));
        var botClient = new BotFrameworkClient(null,
                "*.botframework.com,smba.trafficmanager.net") {
            @Override
            void sendToConversation(String serviceUrl, String conversationId, String text) {
                botCalls.add("send:" + conversationId + ":" + text);
            }
            @Override
            String createConversation(String serviceUrl, String botAppId,
                                      String aadObjectId, String tenantId) {
                botCalls.add("create:" + aadObjectId);
                return "created-conv-for-" + aadObjectId;
            }
        };
        return new TeamsChatPlatform(new TeamsClient(graph))
                .withBotClient(botClient, "https://smba.trafficmanager.net/teams/")
                .withConversationReferenceStore(store)
                .withProactiveConfig("tenant-1", "bot-app-id")
                .build();
    }

    @Test
    void userChannel_storeMiss_createsConversationThenSends() {
        var p = buildProactivePlatform();
        var ref = new ChatChannelRef("user:aad-user-1");
        var result = p.messaging().send(ref, new ChatContent("Hello proactive"));
        assertThat(result.ok()).isTrue();
        assertThat(botCalls).containsExactly("create:aad-user-1",
                "send:created-conv-for-aad-user-1:Hello proactive");
        assertThat(result.messageRef().channel().id()).startsWith("personal:");
    }

    @Test
    void userChannel_storeHit_sendsDirect() {
        store.store(new ConversationReference("aad-user-1", "existing-conv",
                "https://smba.trafficmanager.net/teams/", "tenant-1"));
        var p = buildProactivePlatform();
        var ref = new ChatChannelRef("user:aad-user-1");
        var result = p.messaging().send(ref, new ChatContent("Hello again"));
        assertThat(result.ok()).isTrue();
        assertThat(botCalls).containsExactly("send:existing-conv:Hello again");
    }

    @Test
    void userChannel_storeMiss_storesRefAfterCreate() {
        var p = buildProactivePlatform();
        p.messaging().send(new ChatChannelRef("user:aad-user-1"),
                new ChatContent("Hello"));
        assertThat(store.find("aad-user-1")).isPresent();
        assertThat(store.find("aad-user-1").get().conversationId())
                .isEqualTo("created-conv-for-aad-user-1");
    }

    @Test
    void userChannel_noBotClient_throws() {
        var graph = new MicrosoftGraphClient(request ->
                new StubHttpResponse(200, """
                        {"value":[]}"""));
        var p = new TeamsChatPlatform(new TeamsClient(graph)).build();
        assertThatThrownBy(() ->
                p.messaging().send(new ChatChannelRef("user:aad-user-1"),
                        new ChatContent("Hello")))
                .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsChatPlatformTest`
Expected: compilation errors (new methods don't exist)

- [ ] **Step 3: Add fields and builder methods to TeamsChatPlatform**

Add fields after line 35 (`defaultServiceUrl`):

```java
private ConversationReferenceStore conversationReferenceStore;
private String defaultTenantId;
private String botAppId;
```

Add builder methods after `withBotClient` (line 65):

```java
public TeamsChatPlatform withConversationReferenceStore(ConversationReferenceStore store) {
    this.conversationReferenceStore = store;
    return this;
}

public TeamsChatPlatform withProactiveConfig(String defaultTenantId, String botAppId) {
    this.defaultTenantId = defaultTenantId;
    this.botAppId = botAppId;
    return this;
}
```

Add static helpers after `extractConversationId` (line 209):

```java
static boolean isUserChannel(String channelId) {
    return channelId != null && channelId.startsWith("user:");
}

static String extractAadObjectId(String channelId) {
    return channelId.substring("user:".length());
}
```

- [ ] **Step 4: Add user: routing to sendMessage**

Modify `sendMessage` (line 85-98) to add the `user:` branch:

```java
private SendResult sendMessage(ChatChannelRef channel, ChatContent content) {
    if (isPersonalChannel(channel.id())) {
        return sendPersonalMessage(extractConversationId(channel.id()), content);
    }
    if (isUserChannel(channel.id())) {
        return sendProactiveMessage(extractAadObjectId(channel.id()), content);
    }
    var tc = TeamsMapper.parseTeamAndChannel(channel.id());
    return TokenRetry.withTokenRetry(this::resolveApplicationToken,
                                     token -> {
                                         var uploadedRefs = uploadAttachments(tc[0], tc[1],
                                                                              content.attachments(), token);
                                         var result = client.sendMessage(tc[0], tc[1],
                                                                         TeamsMapper.toMessageBody(content, uploadedRefs), token);
                                         return TeamsMapper.toSendResult(tc[0], tc[1], result);
                                     });
}
```

- [ ] **Step 5: Add sendProactiveMessage method**

Add after `sendPersonalMessage` (line 225):

```java
private SendResult sendProactiveMessage(String aadObjectId, ChatContent content) {
    if (botClient == null) {
        throw new UnsupportedOperationException(
                "Proactive messaging requires BotFrameworkClient — configure teams-bot properties");
    }
    String serviceUrl = defaultServiceUrl;
    if (serviceUrl == null || serviceUrl.isBlank()) {
        throw new UnsupportedOperationException(
                "Proactive messaging requires a default service URL — set casehub.connectors.teams-bot.default-service-url");
    }

    String conversationId;
    String resolvedServiceUrl;

    if (conversationReferenceStore != null) {
        var existing = conversationReferenceStore.find(aadObjectId);
        if (existing.isPresent()) {
            conversationId = existing.get().conversationId();
            resolvedServiceUrl = existing.get().serviceUrl();
            botClient.sendToConversation(resolvedServiceUrl, conversationId, content.text());
            return new SendResult(true,
                    new ChatMessageRef(new ChatChannelRef("personal:" + conversationId), ""),
                    java.time.Instant.now(), null);
        }
    }

    if (defaultTenantId == null || defaultTenantId.isBlank()
            || botAppId == null || botAppId.isBlank()) {
        throw new UnsupportedOperationException(
                "Proactive messaging cold-start requires default-tenant-id and app-id — "
                + "configure casehub.connectors.teams-bot properties");
    }

    conversationId = botClient.createConversation(serviceUrl, botAppId, aadObjectId, defaultTenantId);

    if (conversationReferenceStore != null) {
        conversationReferenceStore.store(
                new ConversationReference(aadObjectId, conversationId, serviceUrl, defaultTenantId));
    }

    botClient.sendToConversation(serviceUrl, conversationId, content.text());
    return new SendResult(true,
            new ChatMessageRef(new ChatChannelRef("personal:" + conversationId), ""),
            java.time.Instant.now(), null);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsChatPlatformTest`
Expected: all tests PASS (including new ProactiveMessagingTest)

- [ ] **Step 7: Wire config and store in TeamsBeans**

Modify `teamsChatPlatform` in `TeamsBeans.java` to add config properties and inject the store:

```java
@Produces
@ApplicationScoped
ChatPlatform teamsChatPlatform(MicrosoftGraphClient graph,
                               ServiceConnectionProvider connectionProvider,
                               TeamsCommands teamsCommands,
                               ConversationReferenceStore conversationReferenceStore,
                               @ConfigProperty(name = "casehub.connectors.teams.default-team-id",
                                               defaultValue = "") String defaultTeamId,
                               @ConfigProperty(name = "casehub.connectors.teams-bot.service-url-allowlist",
                                               defaultValue = "*.botframework.com,smba.trafficmanager.net") String allowlist,
                               @ConfigProperty(name = "casehub.connectors.teams-bot.default-service-url",
                                               defaultValue = "") String defaultServiceUrl,
                               @ConfigProperty(name = "casehub.connectors.teams-bot.default-tenant-id",
                                               defaultValue = "") String defaultTenantId,
                               @ConfigProperty(name = "casehub.connectors.teams-bot.app-id",
                                               defaultValue = "") String appId) {
    var builder = new TeamsChatPlatform(new TeamsClient(graph), connectionProvider,
                                        defaultTeamId)
                          .withCommands(teamsCommands)
                          .withConversationReferenceStore(conversationReferenceStore);
    if (!defaultServiceUrl.isBlank()) {
        var botClient = new BotFrameworkClient(connectionProvider, allowlist);
        builder.withBotClient(botClient, defaultServiceUrl);
    }
    if (!defaultTenantId.isBlank() && !appId.isBlank()) {
        builder.withProactiveConfig(defaultTenantId, appId);
    }
    return builder.build();
}
```

- [ ] **Step 8: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java
git commit -m "feat(#234): user: channel prefix for proactive messaging in TeamsChatPlatform"
```

### Task 4: conversationUpdate handling in TeamsInboundConnector

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java` (handle conversationUpdate)
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java` (inject store into connector)
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java` (add conversationUpdate tests)

**Interfaces:**
- Consumes: `ConversationReferenceStore.store(ConversationReference)` — from Task 1
- Consumes: `ConversationReference(String, String, String, String)` — from Task 1

- [ ] **Step 1: Write the failing tests**

Add to `TeamsInboundConnectorTest.java`:

```java
@Test
void conversationUpdate_membersAdded_storesRefAndDelivers() {
    var store = new InMemoryConversationReferenceStore();
    var auth = new BotFrameworkAuth("test-app-id", true,
            "https://login.botframework.com/v1/.well-known/openidconfiguration",
            "https://api.botframework.com");
    var client = new BotFrameworkClient(null,
            "*.botframework.com,smba.trafficmanager.net");
    var commandService = new CommandService(List.of(), List.of());
    var connectorWithStore = new TeamsInboundConnector(auth, client, commandService,
            "test-app-id", store);

    String body = """
            {"type":"conversationUpdate",\
            "membersAdded":[{"id":"aad-user-1","aadObjectId":"aad-user-1"}],\
            "from":{"id":"29:bot-id","aadObjectId":"aad-bot"},\
            "conversation":{"id":"conv-new-1","conversationType":"personal"},\
            "channelData":{"tenant":{"id":"t-1"}},\
            "serviceUrl":"https://smba.trafficmanager.net/teams/",\
            "id":"act-cu-1","timestamp":"2026-11-16T10:00:00Z"}""";
    var req = postRequest(body);
    var result = connectorWithStore.handle(req);

    assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
    var delivered = (WebhookResult.Delivered) result;
    assertThat(delivered.messages()).hasSize(1);
    var msg = delivered.messages().get(0);
    assertThat(msg.metadata().get("teams-activity-type")).isEqualTo("conversationUpdate");
    assertThat(msg.metadata().get("teams-members-added")).isEqualTo("aad-user-1");

    var storedRef = store.find("aad-user-1");
    assertThat(storedRef).isPresent();
    assertThat(storedRef.get().conversationId()).isEqualTo("conv-new-1");
    assertThat(storedRef.get().serviceUrl()).isEqualTo("https://smba.trafficmanager.net/teams/");
}

@Test
void conversationUpdate_botOnlyAdded_storesNothingButDelivers() {
    var store = new InMemoryConversationReferenceStore();
    var auth = new BotFrameworkAuth("test-app-id", true,
            "https://login.botframework.com/v1/.well-known/openidconfiguration",
            "https://api.botframework.com");
    var client = new BotFrameworkClient(null,
            "*.botframework.com,smba.trafficmanager.net");
    var commandService = new CommandService(List.of(), List.of());
    var connectorWithStore = new TeamsInboundConnector(auth, client, commandService,
            "test-app-id", store);

    String body = """
            {"type":"conversationUpdate",\
            "membersAdded":[{"id":"28:test-app-id","aadObjectId":"test-app-id"}],\
            "from":{"id":"29:user-id","aadObjectId":"aad-user-1"},\
            "conversation":{"id":"conv-1","conversationType":"personal"},\
            "channelData":{"tenant":{"id":"t-1"}},\
            "serviceUrl":"https://smba.trafficmanager.net/teams/",\
            "id":"act-cu-2","timestamp":"2026-11-16T10:00:00Z"}""";
    var req = postRequest(body);
    var result = connectorWithStore.handle(req);

    assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
    assertThat(store.find("test-app-id")).isEmpty();
}

@Test
void conversationUpdate_noStore_stillDelivers() {
    String body = """
            {"type":"conversationUpdate",\
            "membersAdded":[{"id":"aad-user-1","aadObjectId":"aad-user-1"}],\
            "from":{"id":"29:bot-id","aadObjectId":"aad-bot"},\
            "conversation":{"id":"conv-1","conversationType":"personal"},\
            "channelData":{"tenant":{"id":"t-1"}},\
            "serviceUrl":"https://smba.trafficmanager.net/teams/",\
            "id":"act-cu-3","timestamp":"2026-11-16T10:00:00Z"}""";
    var req = postRequest(body);
    var result = connector.handle(req);

    assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest`
Expected: compilation error (new constructor doesn't exist), or assertion failures (conversationUpdate returns Ignored)

- [ ] **Step 3: Add ConversationReferenceStore field and constructor overload to TeamsInboundConnector**

Add field after line 36 (`appId`):

```java
private final ConversationReferenceStore conversationReferenceStore;
```

Add new constructor that accepts the store:

```java
TeamsInboundConnector(BotFrameworkAuth auth, BotFrameworkClient client,
                      CommandService commandService, String appId,
                      ConversationReferenceStore conversationReferenceStore) {
    this.auth = auth;
    this.client = client;
    this.commandService = commandService;
    this.appId = appId;
    this.conversationReferenceStore = conversationReferenceStore;
}
```

Update the existing constructor to delegate with `null` store:

```java
TeamsInboundConnector(BotFrameworkAuth auth, BotFrameworkClient client,
                      CommandService commandService, String appId) {
    this(auth, client, commandService, appId, null);
}
```

- [ ] **Step 4: Replace conversationUpdate case in doHandle with handleConversationUpdate**

Replace lines 86-89 in `doHandle`:

```java
case "conversationUpdate" -> handleConversationUpdate(activity);
```

Add `handleConversationUpdate` method:

```java
private WebhookResult handleConversationUpdate(JsonObject activity) {
    String serviceUrl = activity.getString("serviceUrl", "");
    String conversationId = extractConversationId(activity);
    String tenantId = extractTenancyId(activity);

    var membersAdded = activity.containsKey("membersAdded")
            ? activity.getJsonArray("membersAdded") : null;
    var membersRemoved = activity.containsKey("membersRemoved")
            ? activity.getJsonArray("membersRemoved") : null;

    var addedIds = new java.util.ArrayList<String>();
    if (membersAdded != null && conversationReferenceStore != null) {
        for (var member : membersAdded) {
            var memberObj = member.asJsonObject();
            String aadObjectId = memberObj.getString("aadObjectId", "");
            if (!aadObjectId.isBlank() && !isBotMember(memberObj)) {
                conversationReferenceStore.store(
                        new ConversationReference(aadObjectId, conversationId, serviceUrl, tenantId));
                addedIds.add(aadObjectId);
            }
        }
    } else if (membersAdded != null) {
        for (var member : membersAdded) {
            var memberObj = member.asJsonObject();
            String aadObjectId = memberObj.getString("aadObjectId", "");
            if (!aadObjectId.isBlank() && !isBotMember(memberObj)) {
                addedIds.add(aadObjectId);
            }
        }
    }

    var removedIds = new java.util.ArrayList<String>();
    if (membersRemoved != null) {
        for (var member : membersRemoved) {
            var memberObj = member.asJsonObject();
            String aadObjectId = memberObj.getString("aadObjectId", "");
            if (!aadObjectId.isBlank() && !isBotMember(memberObj)) {
                removedIds.add(aadObjectId);
            }
        }
    }

    var metadata = buildBaseMetadata(activity);
    metadata.put("teams-activity-type", "conversationUpdate");
    if (!addedIds.isEmpty()) {
        metadata.put("teams-members-added", String.join(",", addedIds));
    }
    if (!removedIds.isEmpty()) {
        metadata.put("teams-members-removed", String.join(",", removedIds));
    }

    var msg = new io.casehub.connectors.InboundMessage(
            "conversationUpdate",
            extractSenderId(activity),
            extractFromName(activity),
            extractChannelRef(activity),
            tenantId,
            metadata
    );
    return new WebhookResult.Delivered(List.of(msg));
}

private boolean isBotMember(JsonObject member) {
    String id = member.getString("id", "");
    return !appId.isBlank() && (id.equals("28:" + appId) || id.equals(appId));
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest`
Expected: all tests PASS

- [ ] **Step 6: Wire store injection in TeamsBeans.teamsInboundConnector**

Modify `teamsInboundConnector` in `TeamsBeans.java` to inject the store:

```java
@Produces
@ApplicationScoped
io.casehub.connectors.WebhookInboundConnector teamsInboundConnector(
        @ConfigProperty(name = "casehub.connectors.teams-bot.app-id", defaultValue = "") String appId,
        @ConfigProperty(name = "casehub.connectors.teams-bot.dev-mode", defaultValue = "false") boolean devMode,
        @ConfigProperty(name = "casehub.connectors.teams-bot.service-url-allowlist",
                        defaultValue = "*.botframework.com,smba.trafficmanager.net") String allowlist,
        @ConfigProperty(name = "casehub.connectors.teams-bot.openid-metadata-url",
                        defaultValue = "https://login.botframework.com/v1/.well-known/openidconfiguration") String metadataUrl,
        @ConfigProperty(name = "casehub.connectors.teams-bot.issuer",
                        defaultValue = "https://api.botframework.com") String issuer,
        io.casehub.connectors.chat.command.CommandService commandService,
        ServiceConnectionProvider connectionProvider,
        ConversationReferenceStore conversationReferenceStore) {
    var auth   = new BotFrameworkAuth(appId, devMode, metadataUrl, issuer);
    var client = new BotFrameworkClient(connectionProvider, allowlist);
    return new TeamsInboundConnector(auth, client, commandService, appId, conversationReferenceStore);
}
```

- [ ] **Step 7: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java
git commit -m "feat(#234): handle conversationUpdate — store refs and fire events"
```

## Batch 3: Full build verification

### Task 5: Full build and cleanup

**Files:**
- No new files

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 2: Fix any compilation or test failures across modules**

If any module fails due to the new `ConversationReferenceStore` CDI SPI (e.g. integration-tests module missing a producer), add the necessary test beans or config.

- [ ] **Step 3: Commit any fixes**

```bash
git add -A
git commit -m "fix(#234): resolve cross-module build issues"
```

## References

- [2026-11-16-teams-proactive-messaging-design.md] — design spec this plan implements
- [chat-teams/src/main/java/.../BotFrameworkClient.java] — Bot Framework REST client (lines 15-130)
- [chat-teams/src/main/java/.../TeamsChatPlatform.java] — sendMessage routing (line 85-98), sendPersonalMessage (line 211-225)
- [chat-teams/src/main/java/.../TeamsInboundConnector.java] — conversationUpdate handling (line 86-89)
- [chat-teams/src/main/java/.../TeamsBeans.java] — CDI wiring (lines 12-66)
- [email-google-inbound/.../HistoryStore.java] — CDI SPI pattern precedent
- [email-google-inbound/.../InMemoryHistoryStore.java] — InMemory default pattern precedent
- [GitHub #234] — Proactive messaging (bot-initiated conversations)
- [D1-D5 in decisions.md] — design decisions
