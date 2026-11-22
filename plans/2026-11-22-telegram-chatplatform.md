# Telegram ChatPlatform Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #212 — Epic: Telegram ChatPlatform provider (chat-telegram)
**Issue group:** #212

**Goal:** Add a Telegram ChatPlatform provider via Bot API with 7 native capabilities, webhook-based inbound, and secret token verification.

**Architecture:** Two modules following the established signal-cli/chat-signal pattern. `telegram-bot` provides a thin HTTP client over the Telegram Bot API using `HttpHelper.CLIENT`. `chat-telegram` implements the ChatPlatform SPI with 7 native capabilities (Messaging, Threading, Discovery, Reactions, Members, Presence, Commands) and a WebhookInboundConnector for inbound messages.

**Tech Stack:** Java 21, Quarkus 3.32.2, java.net.http.HttpClient via HttpHelper, Jackson, WireMock for testing.

## Global Constraints

- All HTTP calls use `HttpHelper.CLIENT` — no new HttpClient instances (PP-20260607-9794cb)
- Token passed at call time, not injected into shared clients (PP-20260609-0c3e24)
- Inbound connector IDs as constants in `InboundConnectorIds` (PP-20260607-d4ee52)
- SPI identifier method named `id()` (PP-20260609-e3a2bd)
- Parent POM version: `0.4-SNAPSHOT`
- Jandex plugin required in every new module for CDI bean discovery

---

## Batch 1: Foundation — telegram-bot HTTP client module

After this batch: `TelegramBotClient` exists with WireMock-tested API methods, model records, and Maven module wiring.

### Task 1: telegram-bot module scaffold and model records

**Files:**
- Create: `telegram-bot/pom.xml`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/TelegramChat.java`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/TelegramUser.java`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/TelegramMessage.java`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/TelegramChatMember.java`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/TelegramUpdate.java`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/TelegramCallbackQuery.java`
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/model/SendResponse.java`
- Modify: `pom.xml` (parent POM — add `<module>telegram-bot</module>`)

**Interfaces:**
- Produces: model records used by `TelegramBotClient` (Task 2) and `chat-telegram` module (Tasks 3-5)

- [ ] **Step 1: Create `telegram-bot/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-telegram-bot</artifactId>
  <name>CaseHub Connectors — Telegram Bot</name>
  <description>Telegram Bot API HTTP client. Thin wrapper using HttpHelper.CLIENT
with call-time token passing.</description>

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

    <!-- Testing -->
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
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
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 2: Add module to parent POM**

Add `<module>telegram-bot</module>` after `<module>signal-cli</module>` in `pom.xml`.

- [ ] **Step 3: Create model records**

Create the source directories and all 7 model records. Each is a simple Java record with Jackson annotations for deserialization.

`TelegramUser.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TelegramUser(
        long id,
        @JsonProperty("first_name") String firstName,
        @JsonProperty("last_name") String lastName,
        String username,
        @JsonProperty("is_bot") boolean isBot) {}
```

`TelegramChat.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TelegramChat(
        long id,
        String type,
        String title,
        String description,
        @JsonProperty("first_name") String firstName,
        @JsonProperty("last_name") String lastName,
        String username) {}
```

`TelegramMessage.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TelegramMessage(
        @JsonProperty("message_id") long messageId,
        TelegramChat chat,
        TelegramUser from,
        String text,
        long date,
        @JsonProperty("reply_to_message") TelegramMessage replyToMessage) {}
```

`TelegramChatMember.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TelegramChatMember(TelegramUser user, String status) {}
```

`TelegramCallbackQuery.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TelegramCallbackQuery(
        String id,
        TelegramUser from,
        TelegramMessage message,
        String data) {}
```

`TelegramUpdate.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;

@JsonIgnoreProperties(ignoreUnknown = true)
public record TelegramUpdate(
        @JsonProperty("update_id") long updateId,
        TelegramMessage message,
        @JsonProperty("edited_message") TelegramMessage editedMessage,
        @JsonProperty("callback_query") TelegramCallbackQuery callbackQuery) {}
```

`SendResponse.java`:
```java
package io.casehub.connectors.telegram.bot.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record SendResponse(boolean ok, TelegramMessage result) {}
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl telegram-bot compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add telegram-bot/ pom.xml
git commit -m "feat(#212): scaffold telegram-bot module with model records"
```

---

### Task 2: TelegramBotClient with WireMock tests

**Files:**
- Create: `telegram-bot/src/main/java/io/casehub/connectors/telegram/bot/TelegramBotClient.java`
- Create: `telegram-bot/src/test/java/io/casehub/connectors/telegram/bot/TelegramBotClientTest.java`

**Interfaces:**
- Consumes: model records from Task 1, `HttpHelper.CLIENT` from `connectors-api`
- Produces: `TelegramBotClient` with methods — `sendMessage(String token, String chatId, String text, Long replyToMessageId)` → `TelegramMessage`, `setMessageReaction(String token, String chatId, long messageId, String emoji)` → `boolean`, `getChat(String token, String chatId)` → `TelegramChat`, `getChatMemberCount(String token, String chatId)` → `int`, `getChatMember(String token, String chatId, long userId)` → `TelegramChatMember`, `getChatAdministrators(String token, String chatId)` → `List<TelegramChatMember>`, `setMyCommands(String token, List<BotCommand> commands)` → `boolean`, `deleteMyCommands(String token)` → `boolean`

- [ ] **Step 1: Write failing test for `sendMessage`**

```java
package io.casehub.connectors.telegram.bot;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import io.casehub.connectors.telegram.bot.model.TelegramMessage;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class TelegramBotClientTest {

    private WireMockServer wm;
    private TelegramBotClient client;
    private static final String TOKEN = "123456:ABC-DEF";

    @BeforeEach
    void setUp() {
        wm = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wm.start();
        client = new TelegramBotClient("http://localhost:" + wm.port());
    }

    @AfterEach
    void tearDown() {
        wm.stop();
    }

    @Test
    void sendMessage_returns_message_with_id() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .willReturn(okJson("""
                    {"ok":true,"result":{"message_id":42,"chat":{"id":100,"type":"private"},"from":{"id":1,"first_name":"Bot","is_bot":true},"text":"Hello","date":1700000000}}
                    """)));

        TelegramMessage msg = client.sendMessage(TOKEN, "100", "Hello", null);

        assertThat(msg).isNotNull();
        assertThat(msg.messageId()).isEqualTo(42);
        assertThat(msg.text()).isEqualTo("Hello");
        wm.verify(postRequestedFor(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .withRequestBody(matchingJsonPath("$.chat_id", equalTo("100")))
                .withRequestBody(matchingJsonPath("$.text", equalTo("Hello"))));
    }

    @Test
    void sendMessage_with_reply() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .willReturn(okJson("""
                    {"ok":true,"result":{"message_id":43,"chat":{"id":100,"type":"group"},"from":{"id":1,"first_name":"Bot","is_bot":true},"text":"Reply","date":1700000000}}
                    """)));

        TelegramMessage msg = client.sendMessage(TOKEN, "100", "Reply", 42L);

        assertThat(msg.messageId()).isEqualTo(43);
        wm.verify(postRequestedFor(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .withRequestBody(matchingJsonPath("$.reply_to_message_id", equalTo("42"))));
    }

    @Test
    void sendMessage_returns_null_on_failure() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .willReturn(aResponse().withStatus(400).withBody("{\"ok\":false}")));

        TelegramMessage msg = client.sendMessage(TOKEN, "100", "Hello", null);

        assertThat(msg).isNull();
    }

    @Test
    void getChat_returns_chat() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/getChat"))
                .willReturn(okJson("""
                    {"ok":true,"result":{"id":100,"type":"supergroup","title":"Dev Team","description":"Engineering chat"}}
                    """)));

        var chat = client.getChat(TOKEN, "100");

        assertThat(chat).isNotNull();
        assertThat(chat.title()).isEqualTo("Dev Team");
        assertThat(chat.type()).isEqualTo("supergroup");
    }

    @Test
    void getChatMemberCount_returns_count() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/getChatMemberCount"))
                .willReturn(okJson("{\"ok\":true,\"result\":42}")));

        int count = client.getChatMemberCount(TOKEN, "100");

        assertThat(count).isEqualTo(42);
    }

    @Test
    void getChatAdministrators_returns_list() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/getChatAdministrators"))
                .willReturn(okJson("""
                    {"ok":true,"result":[{"user":{"id":1,"first_name":"Admin","is_bot":false},"status":"administrator"},{"user":{"id":2,"first_name":"Creator","is_bot":false},"status":"creator"}]}
                    """)));

        var admins = client.getChatAdministrators(TOKEN, "100");

        assertThat(admins).hasSize(2);
        assertThat(admins.get(0).status()).isEqualTo("administrator");
        assertThat(admins.get(1).status()).isEqualTo("creator");
    }

    @Test
    void setMessageReaction_returns_true() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/setMessageReaction"))
                .willReturn(okJson("{\"ok\":true,\"result\":true}")));

        boolean ok = client.setMessageReaction(TOKEN, "100", 42, "👍");

        assertThat(ok).isTrue();
    }

    @Test
    void setMyCommands_returns_true() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/setMyCommands"))
                .willReturn(okJson("{\"ok\":true,\"result\":true}")));

        boolean ok = client.setMyCommands(TOKEN,
                java.util.List.of(new TelegramBotClient.BotCommand("start", "Start the bot")));

        assertThat(ok).isTrue();
        wm.verify(postRequestedFor(urlPathEqualTo("/bot" + TOKEN + "/setMyCommands"))
                .withRequestBody(matchingJsonPath("$.commands[0].command", equalTo("start"))));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl telegram-bot test -q`
Expected: FAIL — `TelegramBotClient` does not exist

- [ ] **Step 3: Implement `TelegramBotClient`**

```java
package io.casehub.connectors.telegram.bot;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.connectors.http.HttpHelper;
import io.casehub.connectors.telegram.bot.model.TelegramChat;
import io.casehub.connectors.telegram.bot.model.TelegramChatMember;
import io.casehub.connectors.telegram.bot.model.TelegramMessage;

import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

@jakarta.enterprise.context.ApplicationScoped
public class TelegramBotClient {

    private static final Logger LOG = Logger.getLogger(TelegramBotClient.class.getName());
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Duration TIMEOUT = Duration.ofSeconds(30);

    private final String baseUrl;

    @jakarta.inject.Inject
    public TelegramBotClient(
            @org.eclipse.microprofile.config.inject.ConfigProperty(
                    name = "casehub.telegram.api-url",
                    defaultValue = "https://api.telegram.org")
            final String baseUrl) {
        this.baseUrl = baseUrl;
    }

    public record BotCommand(String command, String description) {}

    public TelegramMessage sendMessage(final String token, final String chatId,
                                       final String text, final Long replyToMessageId) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("chat_id", chatId);
            body.put("text", text);
            if (replyToMessageId != null) {
                body.put("reply_to_message_id", replyToMessageId);
            }
            final JsonNode result = callMethod(token, "sendMessage", body);
            return result != null ? MAPPER.treeToValue(result, TelegramMessage.class) : null;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram sendMessage error", e);
            return null;
        }
    }

    public boolean setMessageReaction(final String token, final String chatId,
                                      final long messageId, final String emoji) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("chat_id", chatId);
            body.put("message_id", messageId);
            final ArrayNode reaction = body.putArray("reaction");
            if (emoji != null && !emoji.isEmpty()) {
                final ObjectNode r = reaction.addObject();
                r.put("type", "emoji");
                r.put("emoji", emoji);
            }
            return callMethod(token, "setMessageReaction", body) != null;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram setMessageReaction error", e);
            return false;
        }
    }

    public TelegramChat getChat(final String token, final String chatId) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("chat_id", chatId);
            final JsonNode result = callMethod(token, "getChat", body);
            return result != null ? MAPPER.treeToValue(result, TelegramChat.class) : null;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram getChat error", e);
            return null;
        }
    }

    public int getChatMemberCount(final String token, final String chatId) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("chat_id", chatId);
            final JsonNode result = callMethod(token, "getChatMemberCount", body);
            return result != null ? result.asInt(0) : 0;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram getChatMemberCount error", e);
            return 0;
        }
    }

    public TelegramChatMember getChatMember(final String token, final String chatId,
                                             final long userId) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("chat_id", chatId);
            body.put("user_id", userId);
            final JsonNode result = callMethod(token, "getChatMember", body);
            return result != null ? MAPPER.treeToValue(result, TelegramChatMember.class) : null;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram getChatMember error", e);
            return null;
        }
    }

    public List<TelegramChatMember> getChatAdministrators(final String token, final String chatId) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("chat_id", chatId);
            final JsonNode result = callMethod(token, "getChatAdministrators", body);
            if (result == null || !result.isArray()) return List.of();
            final List<TelegramChatMember> members = new ArrayList<>();
            for (final JsonNode node : result) {
                members.add(MAPPER.treeToValue(node, TelegramChatMember.class));
            }
            return List.copyOf(members);
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram getChatAdministrators error", e);
            return List.of();
        }
    }

    public boolean setMyCommands(final String token, final List<BotCommand> commands) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            final ArrayNode cmds = body.putArray("commands");
            for (final BotCommand cmd : commands) {
                final ObjectNode c = cmds.addObject();
                c.put("command", cmd.command());
                c.put("description", cmd.description());
            }
            return callMethod(token, "setMyCommands", body) != null;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram setMyCommands error", e);
            return false;
        }
    }

    public boolean deleteMyCommands(final String token) {
        try {
            return callMethod(token, "deleteMyCommands", MAPPER.createObjectNode()) != null;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "telegram deleteMyCommands error", e);
            return false;
        }
    }

    private JsonNode callMethod(final String token, final String method,
                                 final ObjectNode body) throws Exception {
        final HttpResponse<String> resp = HttpHelper.CLIENT.send(
                HttpRequest.newBuilder()
                        .uri(URI.create(baseUrl + "/bot" + token + "/" + method))
                        .header("Content-Type", "application/json")
                        .POST(HttpRequest.BodyPublishers.ofString(MAPPER.writeValueAsString(body)))
                        .timeout(TIMEOUT)
                        .build(),
                HttpResponse.BodyHandlers.ofString());
        if (resp.statusCode() < 200 || resp.statusCode() >= 300) {
            LOG.warning("telegram " + method + " failed: HTTP " + resp.statusCode() + " — " + resp.body());
            return null;
        }
        final JsonNode json = MAPPER.readTree(resp.body());
        if (!json.path("ok").asBoolean(false)) {
            LOG.warning("telegram " + method + " not ok: " + resp.body());
            return null;
        }
        return json.get("result");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl telegram-bot test -q`
Expected: all 8 tests PASS

- [ ] **Step 5: Commit**

```bash
git add telegram-bot/
git commit -m "feat(#212): add TelegramBotClient with WireMock tests"
```

---

## Batch 2: ChatPlatform wiring — chat-telegram module

After this batch: `TelegramChatPlatform` exists with 7 native capabilities, graceful degradation, and full unit tests. Inbound is in Batch 3.

### Task 3: chat-telegram module scaffold and TelegramChatPlatform

**Files:**
- Create: `chat-telegram/pom.xml`
- Create: `chat-telegram/src/main/java/io/casehub/connectors/chat/telegram/TelegramChatPlatform.java`
- Create: `chat-telegram/src/test/java/io/casehub/connectors/chat/telegram/TelegramChatPlatformTest.java`
- Modify: `pom.xml` (parent POM — add `<module>chat-telegram</module>`)
- Modify: `connectors-api/src/main/java/io/casehub/connectors/InboundConnectorIds.java` (add `TELEGRAM_INBOUND`)
- Modify: `connectors-api/src/main/java/io/casehub/connectors/InboundConnectorTypes.java` (add `TELEGRAM`)

**Interfaces:**
- Consumes: `TelegramBotClient` from Task 2, `ChatPlatform` SPI, model records from Task 1
- Produces: `TelegramChatPlatform` implementing `ChatPlatform` with `id()` → `"telegram"`, 7 native capabilities, `supports(Class<?>)` checking `activeCapabilities`, `registerKnownChat(TelegramChat)` for discovery cache

- [ ] **Step 1: Add constants to `connectors-api`**

Add to `InboundConnectorIds.java` before `private InboundConnectorIds() {}`:
```java
public static final String TELEGRAM_INBOUND = "telegram-inbound";
```

Add to `InboundConnectorTypes.java` before `private InboundConnectorTypes() {}`:
```java
public static final String TELEGRAM = "telegram";
```

- [ ] **Step 2: Create `chat-telegram/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-chat-telegram</artifactId>
  <name>CaseHub Connectors — Chat Telegram</name>
  <description>ChatPlatform SPI implementation for Telegram via Bot API.
7 native capabilities: Messaging, Threading, Discovery, Reactions,
Members, Presence, Commands.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-telegram-bot</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
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
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 3: Add module to parent POM**

Add `<module>chat-telegram</module>` after `<module>telegram-bot</module>` in `pom.xml`.

- [ ] **Step 4: Write failing test for TelegramChatPlatform**

```java
package io.casehub.connectors.chat.telegram;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Optional;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.Commands;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;
import io.casehub.connectors.chat.spi.ChannelManagement;
import io.casehub.connectors.chat.spi.MemberManagement;
import io.casehub.connectors.chat.spi.MessageHistory;
import io.casehub.connectors.telegram.bot.TelegramBotClient;
import io.casehub.connectors.telegram.bot.model.TelegramChat;

class TelegramChatPlatformTest {

    private WireMockServer wm;
    private TelegramChatPlatform platform;
    private static final String TOKEN = "123456:ABC-DEF";

    @BeforeEach
    void setUp() {
        wm = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wm.start();
        platform = new TelegramChatPlatform(
                new TelegramBotClient("http://localhost:" + wm.port()),
                Optional.of(TOKEN),
                Optional.empty());
        platform.init();
    }

    @AfterEach
    void tearDown() {
        wm.stop();
    }

    @Test
    void id_returns_telegram() {
        assertThat(platform.id()).isEqualTo("telegram");
    }

    @Test
    void supports_native_capabilities() {
        assertThat(platform.supports(Messaging.class)).isTrue();
        assertThat(platform.supports(Threading.class)).isTrue();
        assertThat(platform.supports(Discovery.class)).isTrue();
        assertThat(platform.supports(Reactions.class)).isTrue();
        assertThat(platform.supports(Presence.class)).isTrue();
        assertThat(platform.supports(Members.class)).isTrue();
        assertThat(platform.supports(Commands.class)).isTrue();
    }

    @Test
    void does_not_support_degraded_capabilities() {
        assertThat(platform.supports(ChannelManagement.class)).isFalse();
        assertThat(platform.supports(MemberManagement.class)).isFalse();
        assertThat(platform.supports(MessageHistory.class)).isFalse();
    }

    @Test
    void send_message_returns_success() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .willReturn(okJson("""
                    {"ok":true,"result":{"message_id":42,"chat":{"id":100,"type":"private"},"from":{"id":1,"first_name":"Bot","is_bot":true},"text":"Hello","date":1700000000}}
                    """)));

        SendResult result = platform.messaging().send(
                new ChatChannelRef("100"), new ChatContent("Hello"));

        assertThat(result.ok()).isTrue();
        assertThat(result.messageRef().messageId()).isEqualTo("42");
    }

    @Test
    void threading_reply_passes_reply_to_message_id() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .willReturn(okJson("""
                    {"ok":true,"result":{"message_id":43,"chat":{"id":100,"type":"group"},"from":{"id":1,"first_name":"Bot","is_bot":true},"text":"Reply","date":1700000000}}
                    """)));

        SendResult result = platform.threading().reply(
                new ChatMessageRef(new ChatChannelRef("100"), "42"),
                new ChatContent("Reply"));

        assertThat(result.ok()).isTrue();
        wm.verify(postRequestedFor(urlPathEqualTo("/bot" + TOKEN + "/sendMessage"))
                .withRequestBody(matchingJsonPath("$.reply_to_message_id", equalTo("42"))));
    }

    @Test
    void discovery_returns_known_chats() {
        platform.registerKnownChat(new TelegramChat(100, "supergroup", "Dev Team", "Engineering", null, null, null));

        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/getChat"))
                .willReturn(okJson("""
                    {"ok":true,"result":{"id":100,"type":"supergroup","title":"Dev Team","description":"Engineering"}}
                    """)));
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/getChatMemberCount"))
                .willReturn(okJson("{\"ok\":true,\"result\":5}")));

        var channels = platform.discovery().listChannels();

        assertThat(channels).hasSize(1);
        assertThat(channels.get(0).name()).isEqualTo("Dev Team");
        assertThat(channels.get(0).memberCount()).isEqualTo(5);
    }

    @Test
    void discovery_returns_empty_without_known_chats() {
        assertThat(platform.discovery().listChannels()).isEmpty();
    }

    @Test
    void reactions_add_calls_set_message_reaction() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/setMessageReaction"))
                .willReturn(okJson("{\"ok\":true,\"result\":true}")));

        platform.reactions().add(
                new ChatMessageRef(new ChatChannelRef("100"), "42"), "👍");

        wm.verify(postRequestedFor(urlPathEqualTo("/bot" + TOKEN + "/setMessageReaction"))
                .withRequestBody(matchingJsonPath("$.message_id", equalTo("42")))
                .withRequestBody(matchingJsonPath("$.reaction[0].emoji", equalTo("👍"))));
    }

    @Test
    void reactions_list_returns_empty() {
        assertThat(platform.reactions().list(
                new ChatMessageRef(new ChatChannelRef("100"), "42")))
                .isEmpty();
    }

    @Test
    void members_returns_administrators() {
        wm.stubFor(post(urlPathEqualTo("/bot" + TOKEN + "/getChatAdministrators"))
                .willReturn(okJson("""
                    {"ok":true,"result":[{"user":{"id":1,"first_name":"Admin","is_bot":false},"status":"administrator"}]}
                    """)));

        var members = platform.members().list(new ChatChannelRef("100"));

        assertThat(members).hasSize(1);
        assertThat(members.get(0).displayName()).isEqualTo("Admin");
    }

    @Test
    void degrades_when_unconfigured() {
        TelegramChatPlatform unconfigured = new TelegramChatPlatform(
                new TelegramBotClient("http://localhost:1"),
                Optional.empty(), Optional.empty());
        unconfigured.init();

        assertThat(unconfigured.supports(Messaging.class)).isFalse();
        assertThat(unconfigured.discovery().listChannels()).isEmpty();
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl chat-telegram test -q`
Expected: FAIL — `TelegramChatPlatform` does not exist

- [ ] **Step 6: Implement `TelegramChatPlatform`**

```java
package io.casehub.connectors.chat.telegram;

import io.casehub.connectors.chat.degraded.ChannelFallbackThreading;
import io.casehub.connectors.chat.degraded.EmptyDiscovery;
import io.casehub.connectors.chat.degraded.EmptyMembers;
import io.casehub.connectors.chat.degraded.EmptyMessageHistory;
import io.casehub.connectors.chat.degraded.NoOpChannelManagement;
import io.casehub.connectors.chat.degraded.NoOpCommands;
import io.casehub.connectors.chat.degraded.NoOpMemberManagement;
import io.casehub.connectors.chat.degraded.NoOpReactions;
import io.casehub.connectors.chat.degraded.UnknownPresence;
import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.Member;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.ChannelManagement;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Commands;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.MemberManagement;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.MessageHistory;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;
import io.casehub.connectors.telegram.bot.TelegramBotClient;
import io.casehub.connectors.telegram.bot.model.TelegramChat;
import io.casehub.connectors.telegram.bot.model.TelegramChatMember;
import io.casehub.connectors.telegram.bot.model.TelegramMessage;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

@ApplicationScoped
public class TelegramChatPlatform implements ChatPlatform {

    private static final Logger LOG = Logger.getLogger(TelegramChatPlatform.class.getName());

    private static final Set<Class<?>> NATIVE_CAPABILITIES = Set.of(
            Messaging.class, Threading.class, Discovery.class,
            Reactions.class, Presence.class, Members.class, Commands.class);

    private final TelegramBotClient client;
    private final String token;

    private final ConcurrentHashMap<String, TelegramChat> knownChats = new ConcurrentHashMap<>();

    private Set<Class<?>> activeCapabilities = Set.of();
    private Messaging messaging;
    private Threading threading;
    private Discovery discovery;
    private Reactions reactions;
    private Presence presence;
    private Members members;
    private Commands commands;
    private ChannelManagement channelManagement;
    private MemberManagement memberManagement;
    private MessageHistory messageHistory;

    @Inject
    public TelegramChatPlatform(
            final TelegramBotClient client,
            @ConfigProperty(name = "casehub.telegram.bot.token") final Optional<String> token,
            @ConfigProperty(name = "casehub.telegram.webhook.secret-token") final Optional<String> webhookSecret) {
        this.client = client;
        this.token = token.orElse("");
    }

    @PostConstruct
    void init() {
        if (token.isBlank()) {
            LOG.warning("telegram: not configured, platform inactive");
            initDegraded();
            return;
        }

        activeCapabilities = NATIVE_CAPABILITIES;
        messaging = this::sendMessage;
        threading = this::replyToMessage;
        discovery = this::listChannels;
        reactions = new TelegramReactions();
        presence = new TelegramPresence();
        members = this::listMembers;
        commands = new NoOpCommands();
        channelManagement = new NoOpChannelManagement();
        memberManagement = new NoOpMemberManagement();
        messageHistory = new EmptyMessageHistory();
    }

    private void initDegraded() {
        activeCapabilities = Set.of();
        messaging = (channel, content) -> SendResult.failure("Telegram not configured");
        threading = new ChannelFallbackThreading(messaging);
        discovery = new EmptyDiscovery();
        reactions = new NoOpReactions();
        presence = new UnknownPresence();
        members = new EmptyMembers();
        commands = new NoOpCommands();
        channelManagement = new NoOpChannelManagement();
        memberManagement = new NoOpMemberManagement();
        messageHistory = new EmptyMessageHistory();
    }

    @Override public String id() { return "telegram"; }
    @Override public Messaging messaging() { return messaging; }
    @Override public Threading threading() { return threading; }
    @Override public Discovery discovery() { return discovery; }
    @Override public Reactions reactions() { return reactions; }
    @Override public Presence presence() { return presence; }
    @Override public Members members() { return members; }
    @Override public ChannelManagement channelManagement() { return channelManagement; }
    @Override public MemberManagement memberManagement() { return memberManagement; }
    @Override public MessageHistory messageHistory() { return messageHistory; }
    @Override public Commands commands() { return commands; }

    @Override
    public boolean supports(final Class<?> capability) {
        return activeCapabilities.contains(capability);
    }

    public void registerKnownChat(final TelegramChat chat) {
        knownChats.put(String.valueOf(chat.id()), chat);
    }

    private SendResult sendMessage(final ChatChannelRef channel, final ChatContent content) {
        final String text = content.markdown() != null ? content.markdown() : content.text();
        final TelegramMessage msg = client.sendMessage(token, channel.id(), text, null);
        if (msg == null) {
            return SendResult.failure("Telegram send failed");
        }
        return SendResult.success(
                new ChatMessageRef(channel, String.valueOf(msg.messageId())),
                Instant.now());
    }

    private SendResult replyToMessage(final ChatMessageRef parent, final ChatContent content) {
        final String text = content.markdown() != null ? content.markdown() : content.text();
        final long replyToId = Long.parseLong(parent.messageId());
        final TelegramMessage msg = client.sendMessage(token, parent.channel().id(), text, replyToId);
        if (msg == null) {
            return SendResult.failure("Telegram reply failed");
        }
        return SendResult.success(
                new ChatMessageRef(parent.channel(), String.valueOf(msg.messageId())),
                Instant.now());
    }

    private List<Channel> listChannels() {
        final List<Channel> result = new ArrayList<>();
        for (final var entry : knownChats.entrySet()) {
            final String chatId = entry.getKey();
            final TelegramChat cached = entry.getValue();
            final TelegramChat chat = client.getChat(token, chatId);
            final int memberCount = client.getChatMemberCount(token, chatId);
            final TelegramChat effective = chat != null ? chat : cached;
            final String name = effective.title() != null ? effective.title()
                    : effective.firstName() != null ? effective.firstName() : chatId;
            result.add(new Channel(
                    new ChatChannelRef(chatId),
                    name,
                    effective.description(),
                    null,
                    "private".equals(effective.type()),
                    memberCount > 0 ? memberCount : null));
        }
        return result;
    }

    private List<Member> listMembers(final ChatChannelRef channel) {
        final var admins = client.getChatAdministrators(token, channel.id());
        return admins.stream()
                .map(cm -> new Member(
                        new MemberRef(String.valueOf(cm.user().id())),
                        cm.user().firstName() != null ? cm.user().firstName() : String.valueOf(cm.user().id())))
                .toList();
    }

    private class TelegramReactions implements Reactions {
        @Override
        public void add(final ChatMessageRef message, final String emoji) {
            client.setMessageReaction(token, message.channel().id(),
                    Long.parseLong(message.messageId()), emoji);
        }

        @Override
        public void remove(final ChatMessageRef message, final String emoji) {
            client.setMessageReaction(token, message.channel().id(),
                    Long.parseLong(message.messageId()), null);
        }

        @Override
        public List<String> list(final ChatMessageRef message) {
            return Collections.emptyList();
        }
    }

    private class TelegramPresence implements Presence {
        @Override
        public PresenceStatus of(final MemberRef member) {
            return PresenceStatus.UNKNOWN;
        }

        @Override
        public void set(final MemberRef member, final PresenceStatus status) {
            // Telegram Bot API does not support setting presence
        }
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl chat-telegram test -q`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add connectors-api/ chat-telegram/ pom.xml
git commit -m "feat(#212): add TelegramChatPlatform with 7 native capabilities"
```

---

## Batch 3: Inbound — webhook connector and translator

After this batch: Telegram webhook inbound works end-to-end with secret token verification, inbound message translation, and known-chat cache population.

### Task 4: TelegramInboundConnector and TelegramInboundTranslator

**Files:**
- Create: `chat-telegram/src/main/java/io/casehub/connectors/chat/telegram/TelegramInboundConnector.java`
- Create: `chat-telegram/src/main/java/io/casehub/connectors/chat/telegram/TelegramInboundTranslator.java`
- Create: `chat-telegram/src/main/java/io/casehub/connectors/chat/telegram/ChatTelegramBeans.java`
- Create: `chat-telegram/src/test/java/io/casehub/connectors/chat/telegram/TelegramInboundConnectorTest.java`
- Create: `chat-telegram/src/test/java/io/casehub/connectors/chat/telegram/TelegramInboundTranslatorTest.java`

**Interfaces:**
- Consumes: `WebhookInboundConnector`, `WebhookRequest`, `WebhookResult` from `connectors-api`; `InboundTranslator` from `chat-spi`; `TelegramUpdate` from Task 1; `TelegramChatPlatform.registerKnownChat()` from Task 3
- Produces: `TelegramInboundConnector` handling webhook POSTs, `TelegramInboundTranslator` translating `InboundMessage` → `ReceivedMessage`, `ChatTelegramBeans` CDI producer

- [ ] **Step 1: Write failing test for TelegramInboundConnector**

```java
package io.casehub.connectors.chat.telegram;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.HttpMethod;
import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;

class TelegramInboundConnectorTest {

    private TelegramInboundConnector connector;
    private static final String SECRET = "my-secret-token";

    @BeforeEach
    void setUp() {
        connector = new TelegramInboundConnector(SECRET, null);
    }

    @Test
    void id_returns_telegram_inbound() {
        assertThat(connector.id()).isEqualTo(InboundConnectorIds.TELEGRAM_INBOUND);
    }

    @Test
    void rejects_missing_secret_token() {
        WebhookRequest request = new WebhookRequest(
                "{\"update_id\":1,\"message\":{\"message_id\":1,\"chat\":{\"id\":100,\"type\":\"private\"},\"from\":{\"id\":42,\"first_name\":\"Alice\",\"is_bot\":false},\"text\":\"Hello\",\"date\":1700000000}}",
                Map.of(), Map.of(), HttpMethod.POST, "/connectors/telegram-inbound/webhook");

        WebhookResult result = connector.handle(request);

        assertThat(result).isInstanceOf(WebhookResult.Unauthorized.class);
    }

    @Test
    void rejects_wrong_secret_token() {
        WebhookRequest request = new WebhookRequest(
                "{\"update_id\":1,\"message\":{\"message_id\":1,\"chat\":{\"id\":100,\"type\":\"private\"},\"from\":{\"id\":42,\"first_name\":\"Alice\",\"is_bot\":false},\"text\":\"Hello\",\"date\":1700000000}}",
                Map.of("x-telegram-bot-api-secret-token", List.of("wrong-secret")),
                Map.of(), HttpMethod.POST, "/connectors/telegram-inbound/webhook");

        WebhookResult result = connector.handle(request);

        assertThat(result).isInstanceOf(WebhookResult.Unauthorized.class);
    }

    @Test
    void delivers_valid_message() {
        WebhookRequest request = new WebhookRequest(
                "{\"update_id\":1,\"message\":{\"message_id\":42,\"chat\":{\"id\":100,\"type\":\"supergroup\",\"title\":\"Dev\"},\"from\":{\"id\":55,\"first_name\":\"Alice\",\"is_bot\":false},\"text\":\"Hello bot\",\"date\":1700000000}}",
                Map.of("x-telegram-bot-api-secret-token", List.of(SECRET)),
                Map.of(), HttpMethod.POST, "/connectors/telegram-inbound/webhook");

        WebhookResult result = connector.handle(request);

        assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
        var delivered = (WebhookResult.Delivered) result;
        assertThat(delivered.messages()).hasSize(1);
        var msg = delivered.messages().get(0);
        assertThat(msg.connectorId()).isEqualTo(InboundConnectorIds.TELEGRAM_INBOUND);
        assertThat(msg.connectorType()).isEqualTo("telegram");
        assertThat(msg.externalSenderId()).isEqualTo("55");
        assertThat(msg.externalChannelRef()).isEqualTo("100");
        assertThat(msg.content()).isEqualTo("Hello bot");
        assertThat(msg.metadata().get("telegram-chat-type")).isEqualTo("supergroup");
        assertThat(msg.metadata().get("telegram-message-id")).isEqualTo("42");
    }

    @Test
    void ignores_update_without_message() {
        WebhookRequest request = new WebhookRequest(
                "{\"update_id\":1}",
                Map.of("x-telegram-bot-api-secret-token", List.of(SECRET)),
                Map.of(), HttpMethod.POST, "/connectors/telegram-inbound/webhook");

        WebhookResult result = connector.handle(request);

        assertThat(result).isInstanceOf(WebhookResult.Ignored.class);
    }

    @Test
    void extracts_reply_metadata() {
        WebhookRequest request = new WebhookRequest(
                "{\"update_id\":1,\"message\":{\"message_id\":43,\"chat\":{\"id\":100,\"type\":\"group\"},\"from\":{\"id\":55,\"first_name\":\"Alice\",\"is_bot\":false},\"text\":\"Reply\",\"date\":1700000000,\"reply_to_message\":{\"message_id\":42,\"chat\":{\"id\":100,\"type\":\"group\"},\"from\":{\"id\":1,\"first_name\":\"Bot\",\"is_bot\":true},\"text\":\"Original\",\"date\":1699999000}}}",
                Map.of("x-telegram-bot-api-secret-token", List.of(SECRET)),
                Map.of(), HttpMethod.POST, "/connectors/telegram-inbound/webhook");

        WebhookResult result = connector.handle(request);

        var msg = ((WebhookResult.Delivered) result).messages().get(0);
        assertThat(msg.metadata().get("telegram-reply-to-message-id")).isEqualTo("42");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl chat-telegram test -Dtest=TelegramInboundConnectorTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement `TelegramInboundConnector`**

```java
package io.casehub.connectors.chat.telegram;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.WebhookInboundConnector;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;
import io.casehub.connectors.telegram.bot.model.TelegramMessage;
import io.casehub.connectors.telegram.bot.model.TelegramUpdate;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.logging.Level;
import java.util.logging.Logger;

@ApplicationScoped
public class TelegramInboundConnector extends WebhookInboundConnector {

    private static final Logger LOG = Logger.getLogger(TelegramInboundConnector.class.getName());
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final String secretToken;
    private final TelegramChatPlatform chatPlatform;

    @Inject
    public TelegramInboundConnector(
            @ConfigProperty(name = "casehub.telegram.webhook.secret-token",
                            defaultValue = "") final String secretToken,
            final TelegramChatPlatform chatPlatform) {
        this.secretToken = secretToken;
        this.chatPlatform = chatPlatform;
    }

    @Override
    public String id() {
        return InboundConnectorIds.TELEGRAM_INBOUND;
    }

    @Override
    public WebhookResult handle(final WebhookRequest request) {
        try {
            final String headerToken = request.header("x-telegram-bot-api-secret-token");
            if (!secretToken.isEmpty() && !secretToken.equals(headerToken)) {
                LOG.warning("telegram-inbound: invalid secret token");
                return new WebhookResult.Unauthorized();
            }

            final TelegramUpdate update = MAPPER.readValue(request.body(), TelegramUpdate.class);
            final TelegramMessage message = extractMessage(update);
            if (message == null || message.text() == null) {
                return new WebhookResult.Ignored();
            }

            if (chatPlatform != null && message.chat() != null) {
                chatPlatform.registerKnownChat(message.chat());
            }

            final Map<String, String> metadata = new HashMap<>();
            metadata.put("telegram-chat-id", String.valueOf(message.chat().id()));
            metadata.put("telegram-message-id", String.valueOf(message.messageId()));
            metadata.put("telegram-sender-id", String.valueOf(message.from().id()));
            metadata.put("telegram-chat-type", message.chat().type());
            if (message.replyToMessage() != null) {
                metadata.put("telegram-reply-to-message-id",
                        String.valueOf(message.replyToMessage().messageId()));
            }

            final InboundMessage inbound = new InboundMessage(
                    InboundConnectorIds.TELEGRAM_INBOUND,
                    InboundConnectorTypes.TELEGRAM,
                    String.valueOf(message.from().id()),
                    String.valueOf(message.chat().id()),
                    message.text(),
                    List.of(),
                    Instant.now(),
                    metadata,
                    null);

            return new WebhookResult.Delivered(List.of(inbound));
        } catch (final Exception e) {
            LOG.log(Level.SEVERE, "telegram-inbound: error handling webhook", e);
            return new WebhookResult.Ignored();
        }
    }

    private static TelegramMessage extractMessage(final TelegramUpdate update) {
        if (update.message() != null) return update.message();
        if (update.editedMessage() != null) return update.editedMessage();
        if (update.callbackQuery() != null && update.callbackQuery().message() != null) {
            return update.callbackQuery().message();
        }
        return null;
    }
}
```

- [ ] **Step 4: Run inbound connector tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl chat-telegram test -Dtest=TelegramInboundConnectorTest -q`
Expected: all 5 tests PASS

- [ ] **Step 5: Write failing test for TelegramInboundTranslator**

```java
package io.casehub.connectors.chat.telegram;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ReceivedMessage;

class TelegramInboundTranslatorTest {

    private final TelegramInboundTranslator translator = new TelegramInboundTranslator();

    @Test
    void connectorType_returns_telegram() {
        assertThat(translator.connectorType()).isEqualTo(InboundConnectorTypes.TELEGRAM);
    }

    @Test
    void translates_simple_message() {
        InboundMessage inbound = new InboundMessage(
                InboundConnectorIds.TELEGRAM_INBOUND,
                InboundConnectorTypes.TELEGRAM,
                "55", "100", "Hello",
                List.of(), Instant.now(),
                Map.of("telegram-chat-id", "100",
                       "telegram-message-id", "42",
                       "telegram-sender-id", "55",
                       "telegram-chat-type", "private"),
                null);

        ReceivedMessage msg = translator.translate(inbound);

        assertThat(msg.platformId()).isEqualTo(InboundConnectorTypes.TELEGRAM);
        assertThat(msg.channel().id()).isEqualTo("100");
        assertThat(msg.messageRef().messageId()).isEqualTo("42");
        assertThat(msg.sender().id()).isEqualTo("55");
        assertThat(msg.content().text()).isEqualTo("Hello");
        assertThat(msg.parentRef()).isNull();
    }

    @Test
    void translates_reply_with_parent_ref() {
        InboundMessage inbound = new InboundMessage(
                InboundConnectorIds.TELEGRAM_INBOUND,
                InboundConnectorTypes.TELEGRAM,
                "55", "100", "Reply",
                List.of(), Instant.now(),
                Map.of("telegram-chat-id", "100",
                       "telegram-message-id", "43",
                       "telegram-sender-id", "55",
                       "telegram-chat-type", "group",
                       "telegram-reply-to-message-id", "42"),
                null);

        ReceivedMessage msg = translator.translate(inbound);

        assertThat(msg.parentRef()).isNotNull();
        assertThat(msg.parentRef().messageId()).isEqualTo("42");
        assertThat(msg.parentRef().channel().id()).isEqualTo("100");
    }
}
```

- [ ] **Step 6: Implement `TelegramInboundTranslator` and `ChatTelegramBeans`**

`TelegramInboundTranslator.java`:
```java
package io.casehub.connectors.chat.telegram;

import java.util.List;

import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

public class TelegramInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return InboundConnectorTypes.TELEGRAM;
    }

    @Override
    public ReceivedMessage translate(final InboundMessage msg) {
        final var channel = new ChatChannelRef(msg.externalChannelRef());
        final var messageRef = new ChatMessageRef(channel,
                msg.metadata().get("telegram-message-id"));

        final String replyToId = msg.metadata().get("telegram-reply-to-message-id");
        final ChatMessageRef parentRef = replyToId != null
                ? new ChatMessageRef(channel, replyToId)
                : null;

        return new ReceivedMessage(
                InboundConnectorTypes.TELEGRAM,
                channel,
                messageRef,
                parentRef,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content(), null, msg.attachments(), List.of()),
                msg.receivedAt());
    }
}
```

`ChatTelegramBeans.java`:
```java
package io.casehub.connectors.chat.telegram;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class ChatTelegramBeans {

    @Produces
    @ApplicationScoped
    public TelegramInboundTranslator telegramInboundTranslator() {
        return new TelegramInboundTranslator();
    }
}
```

- [ ] **Step 7: Run all chat-telegram tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl chat-telegram test -q`
Expected: all tests PASS (platform + inbound connector + translator)

- [ ] **Step 8: Run full build to verify cross-module wiring**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connectors-api,telegram-bot,chat-telegram install -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add chat-telegram/ connectors-api/
git commit -m "feat(#212): add TelegramInboundConnector and TelegramInboundTranslator"
```

---

## Batch 4: Documentation and CLAUDE.md update

After this batch: CLAUDE.md reflects the new modules, consumer/contributor guides updated.

### Task 5: Update documentation

**Files:**
- Modify: `CLAUDE.md` — add `telegram-bot` and `chat-telegram` to modules table, add `chat-telegram` to ChatPlatform implementations list
- Modify: `docs/guides/consumer-guide.md` — add Telegram to ChatPlatform provider list
- Modify: `docs/guides/contributor-guide.md` — add Telegram to provider list if relevant

**Interfaces:**
- Consumes: completed implementation from Tasks 1-4
- Produces: updated documentation

- [ ] **Step 1: Update CLAUDE.md modules table**

Add two rows to the modules table:
```
| `telegram-bot` | Telegram Bot API HTTP client (call-time token, HttpHelper.CLIENT) |
| `chat-telegram` | Telegram ChatPlatform (7 native capabilities: Messaging, Threading, Discovery, Reactions, Members, Presence, Commands) |
```

Add `chat-telegram` to the ChatPlatform implementations list in the project description: `chat-telegram` (Telegram with 7 native capabilities).

Add `TELEGRAM_INBOUND` and `TELEGRAM` constants to the constants/IDs lists.

- [ ] **Step 2: Update consumer guide**

Add Telegram to the ChatPlatform provider table in `docs/guides/consumer-guide.md`.

- [ ] **Step 3: Update contributor guide**

Add Telegram HTTP client and ChatPlatform notes to `docs/guides/contributor-guide.md` if the guide has a providers section.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md docs/guides/
git commit -m "docs(#212): update CLAUDE.md and guides for telegram modules"
```

## References

- [specs/issue-212-telegram-chatplatform/2026-11-22-telegram-chatplatform-design.md] — design spec this plan implements
- [signal-cli/SignalClient.java] — HTTP client pattern reference
- [chat-signal/SignalChatPlatform.java] — ChatPlatform implementation pattern
- [chat-signal/SignalInboundConnector.java] — InboundConnector pattern
- [chat-signal/SignalInboundTranslator.java] — InboundTranslator pattern
- [chat-signal/ChatSignalBeans.java] — CDI producer pattern
- [chat-signal/SignalChatPlatformTest.java] — WireMock testing pattern
- [connectors-api/WebhookInboundConnector.java] — webhook SPI
- [connectors-api/WebhookRequest.java] — header normalisation (lower-case keys)
- [connectors-api/WebhookResult.java] — sealed result types
- [connectors-api/InboundConnectorIds.java] — ID constants
- [connectors-api/InboundConnectorTypes.java] — type constants
- [chat-spi/ChatPlatform.java] — SPI with Builder and @SimulationEligible
- [docs/protocols/connectors/shared-http-client.md] (PP-20260607-9794cb)
- [docs/protocols/connectors/credential-config-ownership.md] (PP-20260609-0c3e24)
- [docs/protocols/connectors/inbound-connector-id-constants.md] (PP-20260607-d4ee52)
- [docs/protocols/connectors/spi-id-method-naming.md] (PP-20260609-e3a2bd)
- [GitHub #212] — focal issue
