# Chat Discord Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `casehub-connectors-discord` (shared HTTP + Gateway client) and `casehub-connectors-chat-discord` (ChatPlatform SPI implementation) — a full Discord chat connector with 8 native capabilities, 1 degraded, and Gateway-driven inbound.

**Architecture:** Two modules following the `slack-bot` / `chat-*` split. The `discord` module contains `DiscordClient` (REST API v10), `DiscordGateway` (WebSocket lifecycle), `DiscordGatewayPresenceCache`, `DiscordDiscovery`, and model DTOs. The `chat-discord` module contains `DiscordChatPlatform` (ChatPlatform SPI), `DiscordInboundConnector` (InboundConnector SPI via Gateway), and `DiscordInboundTranslator`. Discord participates in L2 via `InboundConnector` → `InboundMessage` → `ChatInboundAdapter` → `DiscordInboundTranslator` → `ReceivedMessage`.

**Tech Stack:** Java 21, Quarkus 3.32.2, `java.net.http` (HTTP + WebSocket), Jackson, CDI (`@ApplicationScoped`), JUnit 5, AssertJ, WireMock, Awaitility

**Spec:** `specs/issue-29-chat-discord/2026-06-29-chat-discord-design.md`

## Global Constraints

- Java 21 source, run on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All artifacts: `io.casehub` group, `0.2-SNAPSHOT` version
- Config properties namespaced: `casehub.discord.*`
- Both `casehub.discord.token` and `casehub.discord.guild-id` default to `""` — blank = inactive (WARNING + no-op)
- SPI identifier methods: `id()` not `connectorId()` (protocol PP-20260609-e3a2bd)
- Use `HttpHelper.CLIENT` for ALL outbound connections including WebSocket (protocol PP-20260607-9794cb)
- Token passed at call time, never stored on client (protocol PP-20260609-0c3e24)
- Paginating methods return partial results + WARNING on failure (protocol PP-20260610-83747b)
- InboundConnectorIds constant for `"discord-inbound"` (protocol PP-20260607-d4ee52)
- No business logic — pure delivery infrastructure
- Commits must reference issue: `Refs #29`
- Discord REST API base: `https://discord.com/api/v10`
- Discord Gateway: `wss://gateway.discord.gg/?v=10&encoding=json`
- Discord epoch for snowflake conversion: `1420070400000L` (2015-01-01T00:00:00Z)

## File Map

### discord module — Production (discord/src/main/java/io/casehub/connectors/discord/)

| File | Responsibility |
|------|---------------|
| `model/DiscordMessage.java` | Message DTO record |
| `model/DiscordChannel.java` | Channel DTO record |
| `model/PermissionOverwrite.java` | Permission overwrite DTO record |
| `model/DiscordUser.java` | User DTO record |
| `model/DiscordMember.java` | Guild member DTO record |
| `model/DiscordGuild.java` | Guild DTO record |
| `model/PostResult.java` | Send result DTO record |
| `DiscordClient.java` | HTTP client — REST API v10, rate limit retry, pagination |
| `DiscordGateway.java` | WebSocket Gateway v10 — lifecycle, heartbeat, reconnect, resume |
| `GatewayEventListener.java` | Functional interface for Gateway event dispatch |
| `DiscordGatewayPresenceCache.java` | ConcurrentHashMap presence cache, populated by Gateway events |
| `DiscordDiscovery.java` | ConnectorDiscovery SPI — lists guild channels |

### discord module — Test (discord/src/test/java/io/casehub/connectors/discord/)

| File | Responsibility |
|------|---------------|
| `DiscordClientTest.java` | WireMock HTTP tests for all DiscordClient methods |
| `test/EmbeddedDiscordGateway.java` | Mock WebSocket server for Gateway testing |
| `DiscordGatewayTest.java` | Gateway lifecycle, heartbeat, reconnect tests |
| `DiscordDiscoveryTest.java` | Discovery with blank token, normal operation |

### chat-discord module — Production (chat-discord/src/main/java/io/casehub/connectors/chat/discord/)

| File | Responsibility |
|------|---------------|
| `DiscordChatPlatform.java` | ChatPlatform SPI — 8 native + 1 degraded capability |
| `DiscordInboundConnector.java` | InboundConnector SPI — Gateway lifecycle, event translation |
| `DiscordInboundTranslator.java` | InboundMessage → ReceivedMessage translator |

### chat-discord module — Test (chat-discord/src/test/java/io/casehub/connectors/chat/discord/)

| File | Responsibility |
|------|---------------|
| `DiscordChatPlatformTest.java` | All 9 capabilities + inbound integration tests |
| `DiscordInboundTranslatorTest.java` | Unit tests for InboundMessage → ReceivedMessage |

### Core additions

| File | Change |
|------|--------|
| `core/.../InboundConnectorIds.java` | Add `DISCORD_INBOUND = "discord-inbound"` constant |

### Module config

| File | Responsibility |
|------|---------------|
| `discord/pom.xml` | Maven module for discord HTTP + Gateway client |
| `chat-discord/pom.xml` | Maven module for ChatPlatform SPI implementation |
| `pom.xml` (parent) | Add `<module>discord</module>` and `<module>chat-discord</module>` |

---

### Task 1: discord module — model records + DiscordClient + WireMock tests

The HTTP client is the foundation — every capability in Task 4 and every MCP/Qhorus consumer depends on it. Includes module scaffolding, model DTOs, all HTTP methods, rate limit retry, pagination fail-soft, and comprehensive WireMock tests.

**Files:**
- Create: `discord/pom.xml`
- Modify: `pom.xml` (parent — add `<module>discord</module>`)
- Modify: `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java` (add `DISCORD_INBOUND`)
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordMessage.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordChannel.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/PermissionOverwrite.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordUser.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordMember.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordGuild.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/PostResult.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java`
- Create: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`

**Interfaces:**
- Consumes: `HttpHelper.CLIENT` from `casehub-connectors-core`
- Produces:
  - `DiscordClient` — `@ApplicationScoped` bean with methods: `sendMessage(token, channelId, content)`, `sendReply(token, channelId, content, replyToMessageId)`, `getMessages(token, channelId, afterId, limit)`, `listGuildChannels(token)`, `getChannel(token, channelId)`, `createChannel(token, name, topic, type, nsfw, isPrivate)`, `addReaction(token, channelId, messageId, emoji)`, `removeReaction(token, channelId, messageId, emoji)`, `listReactionEmoji(token, channelId, messageId)`, `listGuildMembers(token, limit, afterUserId)`, `getGuildMember(token, userId)`, `getGuild(token, withCounts)`, `getGatewayUrl(token)`. All return DTOs or null on error.
  - Model records in `io.casehub.connectors.discord.model`: `DiscordMessage`, `DiscordChannel`, `PermissionOverwrite`, `DiscordUser`, `DiscordMember`, `DiscordGuild`, `PostResult`

- [ ] **Step 1: Create discord/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-discord</artifactId>
  <name>CaseHub Connectors — Discord</name>
  <description>Discord Bot API client (REST v10 + Gateway WebSocket).
Uses java.net.http — no external SDK dependency.</description>

  <dependencies>
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
    <dependency>
      <groupId>org.awaitility</groupId>
      <artifactId>awaitility</artifactId>
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

- [ ] **Step 2: Add discord module to parent pom.xml**

Add `<module>discord</module>` after `<module>chat-irc</module>` in the `<modules>` section.

- [ ] **Step 3: Add DISCORD_INBOUND to InboundConnectorIds**

Add to `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java`:
```java
public static final String DISCORD_INBOUND = "discord-inbound";
```

- [ ] **Step 4: Create model records**

All in `discord/src/main/java/io/casehub/connectors/discord/model/`:

```java
// DiscordMessage.java
package io.casehub.connectors.discord.model;

import java.time.Instant;

public record DiscordMessage(String id, String channelId, DiscordUser author,
                             String content, Instant timestamp,
                             String referencedMessageId, int type) {}
```

```java
// DiscordChannel.java
package io.casehub.connectors.discord.model;

import java.util.List;

public record DiscordChannel(String id, String name, String topic, int type,
                             String parentId,
                             List<PermissionOverwrite> permissionOverwrites) {
    public DiscordChannel {
        permissionOverwrites = permissionOverwrites == null
                ? List.of() : List.copyOf(permissionOverwrites);
    }
}
```

```java
// PermissionOverwrite.java
package io.casehub.connectors.discord.model;

public record PermissionOverwrite(String id, int type, long allow, long deny) {}
```

```java
// DiscordUser.java
package io.casehub.connectors.discord.model;

public record DiscordUser(String id, String username, String globalName, boolean bot) {}
```

```java
// DiscordMember.java
package io.casehub.connectors.discord.model;

import java.time.Instant;
import java.util.List;

public record DiscordMember(DiscordUser user, String nick, List<String> roles,
                            Instant joinedAt) {
    public DiscordMember {
        roles = roles == null ? List.of() : List.copyOf(roles);
    }
}
```

```java
// DiscordGuild.java
package io.casehub.connectors.discord.model;

public record DiscordGuild(String id, String name, int approximateMemberCount) {}
```

```java
// PostResult.java
package io.casehub.connectors.discord.model;

public record PostResult(boolean ok, String messageId, String channelId, String error) {

    public static PostResult success(final String messageId, final String channelId) {
        return new PostResult(true, messageId, channelId, null);
    }

    public static PostResult failure(final String error) {
        return new PostResult(false, null, null, error);
    }
}
```

- [ ] **Step 5: Write DiscordClientTest — failing tests first**

Create `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`. Use WireMock to stub Discord REST API responses. Write tests for all DiscordClient methods covering: happy path, error responses, rate limit retry (429 + Retry-After → single retry), pagination (getMessages, listGuildMembers with fail-soft on page 2), createChannel with private flag (permission overwrites in request body).

The test class structure:
- `@BeforeEach`: start WireMock, create DiscordClient with field-injected `apiBaseUrl` and `guildId` pointing at WireMock
- `@AfterEach`: stop WireMock
- Token: `"test-bot-token"`
- Guild ID: `"123456789"`

Key tests:
- `sendMessage_success` — 200 response with `{"id":"msg1","channel_id":"ch1"}` → `PostResult.success("msg1","ch1")`
- `sendMessage_errorReturnsFailure` — 403 → `PostResult.failure("403 ...")`
- `sendReply_includesMessageReference` — verify request body contains `"message_reference":{"message_id":"parent1"}`
- `getMessages_paginates` — two pages with `after` param progression, verify accumulated results
- `getMessages_failSoftOnPage2` — page 1 succeeds (3 messages), page 2 returns 500 → returns 3 messages + WARNING
- `listGuildChannels_success` — returns array of channels
- `getChannel_success` — returns single channel
- `getChannel_404ReturnsNull` — 404 → null
- `createChannel_sendsCorrectBody` — verify `{"name":"test","topic":"t","type":0}`
- `createChannel_privateIncludesOverwrites` — verify `permission_overwrites` array in request when `isPrivate=true`
- `addReaction_204Success` — 204 → no error
- `removeReaction_204Success` — 204 → no error
- `listReactionEmoji_extractsFromMessage` — GET message with `reactions` array → extract emoji names
- `listGuildMembers_paginates` — two pages with `after` snowflake param
- `listGuildMembers_failSoftOnPage2` — partial results + WARNING
- `getGuildMember_success` — returns member
- `getGuildMember_404ReturnsNull` — 404 → null
- `getGuild_success` — returns guild with counts
- `rateLimitRetry_429WithRetryAfter` — first call returns 429 + Retry-After:1, second call returns 200
- `rateLimitRetry_doubleRateLimitReturnsFailure` — both calls return 429 → failure result
- `getGatewayUrl_success` — returns URL from `{"url":"wss://gateway.discord.gg/"}`

- [ ] **Step 6: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure (DiscordClient does not exist yet)

- [ ] **Step 7: Implement DiscordClient**

Create `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java`.

Key implementation details:
- `@ApplicationScoped` CDI bean
- `@ConfigProperty(name = "casehub.discord.guild-id", defaultValue = "") String guildId` — injected for guild-scoped methods
- `String apiBaseUrl` field — `@ConfigProperty(name = "casehub.discord.api-base-url", defaultValue = "https://discord.com/api/v10")` — overridable in tests
- All public methods take `String token` as first parameter
- HTTP requests use `HttpHelper.CLIENT.send(request, BodyHandlers.ofString())`
- Authorization header: `"Bot " + token` (Discord uses "Bot" prefix, not "Bearer")
- JSON parsing: use Jackson `ObjectMapper` (injected or created via `new ObjectMapper().registerModule(new JavaTimeModule())`)
- `sendWithRetry(HttpRequest)` — on 429, read Retry-After, sleep, retry once; second 429 → PostResult.failure("rate-limited")
- `getMessages(token, channelId, afterId, limit)` — paginate with `after` param, accumulate up to `MAX_PAGES` (50), fail-soft
- `listGuildMembers(token, limit, afterUserId)` — paginate with `after` user snowflake, fail-soft
- `createChannel(token, name, topic, type, nsfw, isPrivate)` — when `isPrivate`, include `permission_overwrites` array denying `@everyone` (guild ID as role ID) VIEW_CHANNEL (bit `1 << 10` = `1024`)
- `listReactionEmoji(token, channelId, messageId)` — GET the message, extract `reactions[].emoji.name` (for unicode) or `reactions[].emoji.name + ":" + reactions[].emoji.id` (for custom)
- Error responses: non-paginating methods return null or PostResult.failure with status + body logged at WARNING

- [ ] **Step 8: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all DiscordClientTest tests PASS

- [ ] **Step 9: Build the full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```
feat(discord): add DiscordClient HTTP API + model records — Refs #29
```

---

### Task 2: discord module — DiscordGateway WebSocket client + tests

The Gateway handles real-time event delivery via a long-lived WebSocket connection. Implements the full Discord Gateway v10 lifecycle: HELLO → IDENTIFY → HEARTBEAT loop → DISPATCH events → RESUME on disconnect → re-IDENTIFY on INVALID_SESSION.

**Files:**
- Create: `discord/src/main/java/io/casehub/connectors/discord/GatewayEventListener.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/DiscordGateway.java`
- Create: `discord/src/test/java/io/casehub/connectors/discord/test/EmbeddedDiscordGateway.java`
- Create: `discord/src/test/java/io/casehub/connectors/discord/DiscordGatewayTest.java`

**Interfaces:**
- Consumes: `HttpHelper.CLIENT` for WebSocket builder
- Produces:
  - `GatewayEventListener` — `@FunctionalInterface` with `void onEvent(String eventType, JsonNode data)`
  - `DiscordGateway` — `void connect(String token, int intents, GatewayEventListener listener)`, `void disconnect()`, `boolean isConnected()`

- [ ] **Step 1: Create GatewayEventListener**

```java
// discord/src/main/java/io/casehub/connectors/discord/GatewayEventListener.java
package io.casehub.connectors.discord;

import com.fasterxml.jackson.databind.JsonNode;

@FunctionalInterface
public interface GatewayEventListener {
    void onEvent(String eventType, JsonNode data);
}
```

- [ ] **Step 2: Create EmbeddedDiscordGateway test utility**

Create `discord/src/test/java/io/casehub/connectors/discord/test/EmbeddedDiscordGateway.java`.

A minimal WebSocket server (~200 lines) that simulates Discord Gateway v10 for testing:
- Uses `com.sun.net.httpserver.HttpServer` + WebSocket upgrade via raw socket manipulation, OR use `java.net.http.WebSocket` in reverse (server role). Simplest approach: a plain `ServerSocket` that accepts TCP connections, performs the WebSocket HTTP upgrade handshake manually, then sends/receives WebSocket frames.
- On connect: sends HELLO (opcode 10) with `heartbeat_interval: 500` (fast for tests)
- Expects IDENTIFY (opcode 2): validates token, responds with READY (opcode 0, `t: "READY"`) containing `session_id` and `resume_gateway_url`
- Tracks heartbeats (opcode 1): responds with HEARTBEAT_ACK (opcode 11) unless `suppressAcks` is set
- `sendDispatch(eventType, data)` — sends opcode 0 DISPATCH with given event type and data
- `sendReconnect()` — sends opcode 7
- `sendInvalidSession(resumable)` — sends opcode 9
- `disconnect()` — closes the connection
- `getPort()` — returns bound port
- `suppressHeartbeatAcks(boolean)` — for testing ACK timeout
- `getReceivedIdentify()` / `getReceivedResume()` — for verifying client payloads
- Uses virtual threads, binds to port 0

- [ ] **Step 3: Write DiscordGatewayTest — failing tests**

Create `discord/src/test/java/io/casehub/connectors/discord/DiscordGatewayTest.java`.

Tests:
- `connectAndIdentify` — connects, sends IDENTIFY after HELLO, receives READY
- `heartbeatLoop` — after connect, verify heartbeats arrive at the expected interval with correct sequence number
- `heartbeatAckTimeout` — suppress ACKs → gateway reconnects
- `dispatchEvent` — server sends MESSAGE_CREATE DISPATCH → listener receives it
- `resumeOnDisconnect` — server drops connection → client reconnects and sends RESUME with session_id + seq
- `invalidSessionFallback` — server sends INVALID_SESSION → client sends full IDENTIFY
- `reconnectBackoff` — server rejects connections repeatedly → verify exponential delay between attempts
- `reconnectBackoff_logEscalation` — verify WARNING for attempts 1-4, SEVERE at attempt 5+
- `multiFrameMessage` — server sends a DISPATCH split across two WebSocket frames → listener receives complete event
- `guildCreate_presencesDelivered` — server sends GUILD_CREATE with presences[] → verify listener receives the event (presence cache population is tested in Task 5)

- [ ] **Step 4: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml -Dtest=DiscordGatewayTest`
Expected: compilation failure

- [ ] **Step 5: Implement DiscordGateway**

Create `discord/src/main/java/io/casehub/connectors/discord/DiscordGateway.java`.

Key implementation details:
- NOT a CDI bean — instantiated by `DiscordInboundConnector`. No `@ApplicationScoped`. This allows the connector to control the lifecycle and pass the token at construction time.
- `connect(String token, int intents, GatewayEventListener listener)`:
  1. Fetch Gateway URL via `DiscordClient.getGatewayUrl(token)` or use cached URL
  2. Open WebSocket via `HttpHelper.CLIENT.newWebSocketBuilder().buildAsync(URI.create(url + "?v=10&encoding=json"), webSocketListener)`
  3. WebSocket listener implements `WebSocket.Listener`: `onText()` buffers frames in StringBuilder, parses on `last == true`
  4. On HELLO (opcode 10): extract `heartbeat_interval`, start heartbeat virtual thread
  5. On HELLO received: send IDENTIFY (opcode 2) with `{token, intents, properties: {os: "linux", browser: "casehub", device: "casehub"}}`
  6. On READY: cache `session_id`, `resume_gateway_url`
  7. On DISPATCH (opcode 0): update `lastSequence` AtomicLong, call `listener.onEvent(eventType, data)`
  8. On RECONNECT (opcode 7): close and reconnect
  9. On INVALID_SESSION (opcode 9): if `d == true` (resumable) → resume; if `d == false` → full re-identify
- `disconnect()`: close WebSocket, stop heartbeat thread, set state to DISCONNECTED
- Heartbeat thread: virtual thread running a loop — sleep `interval`, send opcode 1 with `lastSequence`, wait for ACK within `interval` ms. Missing ACK → close and reconnect.
- Reconnect: exponential backoff 1s → 2s → 4s → ... → 60s max. Track `consecutiveFailures`. Log WARNING for 1-4, SEVERE at 5+. On reconnect: use `resume_gateway_url`, send RESUME with `session_id` + `seq`. If INVALID_SESSION → full IDENTIFY.
- State machine: enum `GatewayState { DISCONNECTED, CONNECTING, HELLO_RECEIVED, IDENTIFYING, READY, RUNNING, RESUMING }`
- Thread safety: synchronized `connect()`/`disconnect()`. Event dispatch is single-threaded (WebSocket listener thread).

- [ ] **Step 6: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml -Dtest=DiscordGatewayTest`
Expected: all tests PASS

- [ ] **Step 7: Build full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(discord): add DiscordGateway WebSocket client — Refs #29
```

---

### Task 3: discord module — DiscordGatewayPresenceCache + DiscordDiscovery + tests

The presence cache is a simple CDI bean; the discovery implementation follows the `SlackBotDiscovery` pattern exactly.

**Files:**
- Create: `discord/src/main/java/io/casehub/connectors/discord/DiscordGatewayPresenceCache.java`
- Create: `discord/src/main/java/io/casehub/connectors/discord/DiscordDiscovery.java`
- Create: `discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java`

**Interfaces:**
- Consumes: `DiscordClient` from Task 1, `ConnectorDiscovery` from core
- Produces:
  - `DiscordGatewayPresenceCache` — `void update(String userId, PresenceStatus status)`, `PresenceStatus get(String userId)`
  - `DiscordDiscovery implements ConnectorDiscovery` — `id() = "discord"`, `discover()` returns guild text channels as `DiscoveredTarget`

- [ ] **Step 1: Create DiscordGatewayPresenceCache**

```java
// discord/src/main/java/io/casehub/connectors/discord/DiscordGatewayPresenceCache.java
package io.casehub.connectors.discord;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.chat.model.PresenceStatus;

@ApplicationScoped
public class DiscordGatewayPresenceCache {

    private final Map<String, PresenceStatus> cache = new ConcurrentHashMap<>();

    public void update(final String userId, final PresenceStatus status) {
        cache.put(userId, status);
    }

    public PresenceStatus get(final String userId) {
        return cache.getOrDefault(userId, PresenceStatus.UNKNOWN);
    }

    public void clear() {
        cache.clear();
    }
}
```

Note: `PresenceStatus` is from `casehub-connectors-chat-spi`. The `discord` module depends on `core` only, not `chat-spi`. The presence cache needs `PresenceStatus` — this means `discord/pom.xml` needs `casehub-connectors-chat-spi` as a dependency. But wait — this breaks the module split: the `discord` module should be independent of `chat-spi`.

**Design decision:** The presence cache stores Discord status strings (`"online"`, `"idle"`, `"dnd"`, `"offline"`) and the mapping to `PresenceStatus` happens in `chat-discord`. This keeps `discord` independent of `chat-spi`.

Revised:

```java
package io.casehub.connectors.discord;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class DiscordGatewayPresenceCache {

    private final Map<String, String> cache = new ConcurrentHashMap<>();

    public void update(final String userId, final String status) {
        cache.put(userId, status);
    }

    public String get(final String userId) {
        return cache.getOrDefault(userId, "unknown");
    }

    public void clear() {
        cache.clear();
    }
}
```

The `chat-discord` module's `DiscordChatPlatform` maps `"online"` → `ONLINE`, `"idle"` → `AWAY`, etc.

- [ ] **Step 2: Write DiscordDiscoveryTest — failing tests**

Tests:
- `discover_blankTokenReturnsEmpty` — blank token → empty list
- `discover_returnsTextChannels` — stub `listGuildChannels` → verify mapped to `DiscoveredTarget(id, "#name")`
- `discover_filtersNonTextChannels` — voice (type 2), category (type 4) excluded
- `id_returnsDiscord` — `id()` returns `"discord"`

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml -Dtest=DiscordDiscoveryTest`
Expected: compilation failure

- [ ] **Step 4: Implement DiscordDiscovery**

```java
// discord/src/main/java/io/casehub/connectors/discord/DiscordDiscovery.java
package io.casehub.connectors.discord;

import java.util.List;
import java.util.Set;
import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.casehub.connectors.ConnectorDiscovery;
import io.casehub.connectors.DiscoveredTarget;

@ApplicationScoped
public class DiscordDiscovery implements ConnectorDiscovery {

    public static final String ID = "discord";

    private static final Logger LOG = Logger.getLogger(DiscordDiscovery.class.getName());
    private static final Set<Integer> TEXT_CHANNEL_TYPES = Set.of(0, 5, 10, 11, 12);

    private final DiscordClient client;
    private final String token;

    @Inject
    DiscordDiscovery(final DiscordClient client,
                     @ConfigProperty(name = "casehub.discord.token",
                                     defaultValue = "") final String token) {
        this.client = client;
        this.token = token;
    }

    @Override
    public String id() {
        return ID;
    }

    @Override
    public List<DiscoveredTarget> discover() {
        if (token.isBlank()) return List.of();
        final var channels = client.listGuildChannels(token);
        if (channels == null) return List.of();
        return channels.stream()
                .filter(ch -> TEXT_CHANNEL_TYPES.contains(ch.type()))
                .filter(ch -> ch.type() != 15) // exclude forum channels
                .map(ch -> new DiscoveredTarget(ch.id(), "#" + ch.name()))
                .toList();
    }
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml -Dtest=DiscordDiscoveryTest`
Expected: all tests PASS

- [ ] **Step 6: Build full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(discord): add DiscordGatewayPresenceCache + DiscordDiscovery — Refs #29
```

---

### Task 4: chat-discord module — DiscordChatPlatform + all capabilities + contract tests

The ChatPlatform SPI implementation. 8 native capabilities (Messaging, Threading, Discovery, Reactions, Presence, Members, ChannelManagement, MessageHistory) + 1 degraded (MemberManagement). Follows the `IrcChatPlatform` pattern: capabilities as final fields initialized in the constructor.

**Files:**
- Create: `chat-discord/pom.xml`
- Modify: `pom.xml` (parent — add `<module>chat-discord</module>`)
- Create: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java`
- Create: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java`

**Interfaces:**
- Consumes: `DiscordClient` (Task 1), `DiscordGatewayPresenceCache` (Task 3), all SPI interfaces from `chat-spi`
- Produces:
  - `DiscordChatPlatform implements ChatPlatform` — `@ApplicationScoped`, `id() = "discord"`, all 9 capability accessors, `supports(Class<?>)` checking native set

- [ ] **Step 1: Create chat-discord/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-chat-discord</artifactId>
  <name>CaseHub Connectors — Chat Discord</name>
  <description>ChatPlatform SPI implementation for Discord. All capabilities native
except MemberManagement. Gateway-driven inbound via InboundConnector.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-discord</artifactId>
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
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.awaitility</groupId>
      <artifactId>awaitility</artifactId>
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

- [ ] **Step 2: Add chat-discord module to parent pom.xml**

Add `<module>chat-discord</module>` after `<module>discord</module>`.

- [ ] **Step 3: Write DiscordChatPlatformTest — failing tests**

Create `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java`.

Uses WireMock to stub Discord REST API, creates DiscordClient and DiscordGatewayPresenceCache directly (no CDI — plain JUnit like `IrcChatPlatformTest`).

Tests:
- `messaging_send` — sends message, returns SendResult with messageRef
- `messaging_contentExceeds2000CharsReturnsFailure` — 2001 chars → `SendResult.failure("Content exceeds Discord's 2000-character limit")`
- `messaging_prefersMarkdownOverText` — `ChatContent("text", "**markdown**", List.of())` → request body uses `"**markdown**"`
- `threading_reply` — sends with `message_reference` in body
- `discovery_listChannels` — returns text channels, excludes voice/category
- `discovery_excludesForumChannels` — type 15 excluded
- `reactions_addRemoveList` — add reaction (PUT 204), remove (DELETE 204), list (GET message with reactions array)
- `presence_ofMember` — pre-populate cache → returns mapped PresenceStatus
- `presence_setLogsWarning` — `set()` does not throw
- `members_list` — returns guild members mapped to Member records
- `memberManagement_isDegraded` — `supports(MemberManagement.class)` returns false
- `channelManagement_createAndFind` — create channel → POST, find → GET
- `channelManagement_createPrivateChannel` — verify permission_overwrites in request
- `channelManagement_findDerivesIsPrivate` — channel with @everyone VIEW_CHANNEL denied → `isPrivate = true`
- `messageHistory_messagesSince` — GET messages with synthetic snowflake `after` param

- [ ] **Step 4: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml -Dtest=DiscordChatPlatformTest`
Expected: compilation failure

- [ ] **Step 5: Implement DiscordChatPlatform**

Create `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java`.

Key implementation details:
- `@ApplicationScoped`, `@Inject DiscordClient client`, `@Inject DiscordGatewayPresenceCache presenceCache`
- `@ConfigProperty(name = "casehub.discord.token", defaultValue = "") String token`
- `@ConfigProperty(name = "casehub.discord.guild-id", defaultValue = "") String guildId`
- `DISCORD_EPOCH = 1420070400000L`
- `MAX_CONTENT_LENGTH = 2000`
- `VIEW_CHANNEL_BIT = 1L << 10` (= 1024)
- Native capabilities set: all except `MemberManagement.class`
- Capabilities as final fields initialized in `@PostConstruct` (not constructor — needs injected fields)
- `id() = "discord"`
- Messaging: check content length, prefer `content.markdown()`, delegate to `client.sendMessage()`, map `PostResult` → `SendResult`
- Threading: delegate to `client.sendReply()` with `parent.messageId()`
- Discovery: delegate to `client.listGuildChannels()`, filter text channels (types 0, 5, 10, 11, 12), exclude forum (15), map to `Channel`
- Reactions: add/remove delegate to client; list calls `client.listReactionEmoji()`
- Presence: `of()` reads cache string → map to PresenceStatus (`"online"→ONLINE, "idle"→AWAY, "dnd"→DND, "offline"→OFFLINE`, default `UNKNOWN`); `set()` logs WARNING
- Members: delegate to `client.listGuildMembers()`, map `DiscordMember` → `Member`
- ChannelManagement: create delegates to `client.createChannel()` with `type=0, nsfw=false`; find delegates to `client.getChannel()`, maps `Optional.empty()` for null; both derive `isPrivate` from permission overwrites
- MessageHistory: convert `Instant since` to synthetic snowflake `((since.toEpochMilli() - DISCORD_EPOCH) << 22)`, delegate to `client.getMessages()`, map to `ReceivedMessage`
- MemberManagement: `new NoOpMemberManagement()`

- [ ] **Step 6: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml -Dtest=DiscordChatPlatformTest`
Expected: all tests PASS

- [ ] **Step 7: Build full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(chat-discord): add DiscordChatPlatform with 8 native capabilities — Refs #29
```

---

### Task 5: chat-discord module — DiscordInboundConnector + DiscordInboundTranslator + tests

The inbound connector starts the Discord Gateway, translates MESSAGE_CREATE events to `InboundMessage`, seeds the presence cache from GUILD_CREATE, and updates it on PRESENCE_UPDATE. Follows the `IrcInboundConnector` pattern for lifecycle and reconnection.

**Files:**
- Create: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundConnector.java`
- Create: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundTranslator.java`
- Create: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInboundTranslatorTest.java`
- Modify: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java` (add inbound tests)

**Interfaces:**
- Consumes: `DiscordGateway` (Task 2), `DiscordGatewayPresenceCache` (Task 3), `InboundConnector` from core, `InboundTranslator` from chat-spi
- Produces:
  - `DiscordInboundConnector implements InboundConnector` — `id() = InboundConnectorIds.DISCORD_INBOUND`, `start(sink)` connects Gateway, `stop()` disconnects
  - `DiscordInboundTranslator implements InboundTranslator` — `connectorType() = "discord"`, translates `InboundMessage` → `ReceivedMessage`

- [ ] **Step 1: Write DiscordInboundTranslatorTest — failing tests**

Create `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInboundTranslatorTest.java`.

Tests:
- `connectorType_returnsDiscord` — `connectorType()` returns `"discord"`
- `translate_basicMessage` — InboundMessage with metadata → ReceivedMessage with correct channel, messageRef, sender, content, timestamp
- `translate_replyMessage` — InboundMessage with `discord-reference-id` in metadata → ReceivedMessage with non-null parentRef
- `translate_noReference` — InboundMessage without `discord-reference-id` → parentRef is null

- [ ] **Step 2: Write inbound integration tests in DiscordChatPlatformTest**

Add to existing `DiscordChatPlatformTest`:
- `inbound_messageCreateFiresEvent` — use EmbeddedDiscordGateway, connect DiscordInboundConnector, send MESSAGE_CREATE DISPATCH → verify InboundMessage delivered to sink
- `inbound_systemMessagesFiltered` — send MESSAGE_CREATE with type 7 (member join) → not delivered
- `inbound_botMessagesFiltered` — send MESSAGE_CREATE with `author.bot = true` → not delivered
- `inbound_blankTokenConnectorInactive` — blank token → start() returns immediately, no Gateway connection
- `inbound_blankGuildIdConnectorInactive` — blank guild-id → same
- `inbound_presenceCachePopulatedFromGuildCreate` — send GUILD_CREATE with presences[] → verify cache populated
- `inbound_presenceUpdateUpdatesCacheAfterConnect` — send PRESENCE_UPDATE → verify cache updated

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure

- [ ] **Step 4: Implement DiscordInboundTranslator**

```java
// chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundTranslator.java
package io.casehub.connectors.chat.discord;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

@ApplicationScoped
public class DiscordInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return "discord";
    }

    @Override
    public ReceivedMessage translate(final InboundMessage msg) {
        final var channel = new ChatChannelRef(msg.externalChannelRef());
        final var messageRef = new ChatMessageRef(channel,
                msg.metadata().get("discord-message-id"));
        final String refId = msg.metadata().get("discord-reference-id");
        final ChatMessageRef parentRef = refId != null
                ? new ChatMessageRef(channel, refId) : null;
        return new ReceivedMessage(
                "discord",
                channel,
                messageRef,
                parentRef,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 5: Implement DiscordInboundConnector**

Create `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundConnector.java`.

Key implementation details:
- `@ApplicationScoped`, implements `InboundConnector`
- `@Inject DiscordGatewayPresenceCache presenceCache`
- `@ConfigProperty(name = "casehub.discord.token", defaultValue = "") String token`
- `@ConfigProperty(name = "casehub.discord.guild-id", defaultValue = "") String guildId`
- `id()` returns `InboundConnectorIds.DISCORD_INBOUND`
- Intents: `(1 << 0) | (1 << 1) | (1 << 8) | (1 << 9) | (1 << 10) | (1 << 15)` = GUILDS | GUILD_MEMBERS | GUILD_PRESENCES | GUILD_MESSAGES | GUILD_MESSAGE_REACTIONS | MESSAGE_CONTENT
- `start(InboundMessageSink sink)`:
  1. Check `token.isBlank() || guildId.isBlank()` → LOG.warning + return
  2. Create `DiscordGateway` (new instance, not CDI — takes DiscordClient for Gateway URL lookup)
  3. `gateway.connect(token, INTENTS, listener)` where listener handles:
     - `MESSAGE_CREATE`: filter `author.bot == true`, filter to types 0 and 19 only, build InboundMessage, call `sink.receive()`
     - `GUILD_CREATE`: iterate `presences[]`, call `presenceCache.update(userId, status)` for each
     - `PRESENCE_UPDATE`: call `presenceCache.update(userId, status)`
- `stop()`: set `volatile stopping = true`, call `gateway.disconnect()`
- InboundMessage construction: `connectorId = DISCORD_INBOUND`, `connectorType = "discord"`, metadata = `{"discord-message-id", "discord-guild-id", "discord-reference-id" (if type 19)}`, attachments = `List.of()` (deferred)

- [ ] **Step 6: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 7: Build full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(chat-discord): add DiscordInboundConnector + DiscordInboundTranslator — Closes #29
```

---

## Post-Implementation Checklist

After all 5 tasks pass:

- [ ] Full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- [ ] Review all files against spec for completeness
- [ ] Check protocol compliance: shared-http-client, credential-config-ownership, paginating-client-fail-soft, spi-id-method-naming, inbound-connector-id-constants
- [ ] Platform coherence review against `docs/PLATFORM.md`
- [ ] Code review via `superpowers:requesting-code-review`
- [ ] Doc sync via `implementation-doc-sync`
