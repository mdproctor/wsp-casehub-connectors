# MCP Slack Bot Tools — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `send_slack_bot` and `list_channels` MCP tools backed by `SlackBotClient`, introduce the `ConnectorDiscovery` SPI in `core`, and fix the `@Blocking` omission on all existing MCP tools.

**Architecture:** `DiscoveredTarget` and `ConnectorDiscovery` are new types in `core`. `SlackBotClient` gains a `listChannels(token)` method. `SlackBotDiscovery` (new `@ApplicationScoped` CDI bean in `slack-bot`) owns `ConnectorDiscovery` + bot token config. `SlackBotMcpTool` and `ChannelDiscoveryMcpTool` (new in `mcp`) expose `send_slack_bot` and `list_channels`. `send_slack_bot` bypasses `ConnectorService` to return the Slack `ts`.

**Tech Stack:** Java 21, Quarkus 3.32.2, `quarkus-mcp-server-core:1.11.1`, `jakarta.json` (manual parsing), WireMock 3.9.1, JUnit 5, AssertJ. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn ...`

---

## File Map

**Create:**
- `core/src/main/java/io/casehub/connectors/DiscoveredTarget.java`
- `core/src/main/java/io/casehub/connectors/ConnectorDiscovery.java`
- `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotDiscovery.java`
- `slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotDiscoveryTest.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/SlackBotMcpTool.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/ChannelDiscoveryMcpTool.java`
- `mcp/src/test/java/io/casehub/connectors/slack/bot/SlackBotMcpToolTest.java` ← package is `slack.bot`, module is `mcp`
- `mcp/src/test/java/io/casehub/connectors/mcp/ChannelDiscoveryMcpToolTest.java`

**Modify:**
- `core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java` — Javadoc only
- `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java` — add `ID` constant, `listChannels()`, `parseChannels()`
- `slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotClientTest.java` — 3 new tests + new imports
- `mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java` — make all types and members public
- `mcp/pom.xml` — add `casehub-connectors-slack-bot` compile dep + `wiremock-standalone` test dep
- `mcp/src/main/java/io/casehub/connectors/mcp/SlackMcpTool.java` — add `@Blocking`
- `mcp/src/main/java/io/casehub/connectors/mcp/TeamsMcpTool.java` — add `@Blocking`
- `mcp/src/main/java/io/casehub/connectors/mcp/TwilioSmsMcpTool.java` — add `@Blocking`
- `mcp/src/main/java/io/casehub/connectors/mcp/WhatsAppMcpTool.java` — add `@Blocking`
- `mcp/src/main/java/io/casehub/connectors/mcp/EmailMcpTool.java` — add `@Blocking`

---

## Task 1: Core SPI — `DiscoveredTarget`, `ConnectorDiscovery`, `ConnectorMeshBridge` Javadoc

**Files:**
- Create: `core/src/main/java/io/casehub/connectors/DiscoveredTarget.java`
- Create: `core/src/main/java/io/casehub/connectors/ConnectorDiscovery.java`
- Modify: `core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java`

No new tests needed — `DiscoveredTarget` is a record (compiler-generated accessors) and `ConnectorDiscovery` is a pure interface. The build itself is the verification.

- [ ] **Step 1: Create `DiscoveredTarget.java`**

```java
package io.casehub.connectors;

/**
 * A delivery target discovered at runtime via {@link ConnectorDiscovery}.
 *
 * @param id          the identifier to pass to MCP tools (e.g. Slack channel ID {@code C123ABC})
 * @param displayName human-readable label shown in {@code list_channels} output (e.g. {@code #general})
 */
public record DiscoveredTarget(String id, String displayName) {}
```

- [ ] **Step 2: Create `ConnectorDiscovery.java`**

```java
package io.casehub.connectors;

import java.util.List;

/**
 * Optional SPI for connectors whose delivery targets are discoverable at runtime.
 *
 * <p>Implementations are {@code @ApplicationScoped} CDI beans discovered automatically.
 * The {@code list_channels} MCP tool aggregates all registered implementations via
 * {@code @All List<ConnectorDiscovery>}.
 *
 * <h2>Contract for implementations</h2>
 * <ul>
 * <li>Must not throw — exceptions propagate to the MCP tool caller and silence all
 *     other discoveries. Catch internally and return an empty list on failure.</li>
 * <li>Must return quickly — no long-running blocking calls without virtual-thread
 *     offloading.</li>
 * </ul>
 */
public interface ConnectorDiscovery {
    String connectorId();
    List<DiscoveredTarget> discover();
}
```

- [ ] **Step 3: Update `ConnectorMeshBridge` Javadoc**

In `ConnectorMeshBridge.java`, find:
```java
 * @param destination  delivery target: webhook URL, E.164 number, or email address
```
Replace with:
```java
 * @param destination  delivery target: webhook URL, E.164 number, email address, or channel ID
```

- [ ] **Step 4: Build `core` to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All existing core tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  core/src/main/java/io/casehub/connectors/DiscoveredTarget.java \
  core/src/main/java/io/casehub/connectors/ConnectorDiscovery.java \
  core/src/main/java/io/casehub/connectors/ConnectorMeshBridge.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(core): ConnectorDiscovery SPI and DiscoveredTarget record

Closes #16 (partial — SPI foundations)

Refs #16"
```

---

## Task 2: `SlackBotClient` — `ID` constant and `listChannels()`

**Files:**
- Modify: `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java`
- Modify: `slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotClientTest.java`

- [ ] **Step 1: Write three failing tests in `SlackBotClientTest`**

Add these imports at the top of `SlackBotClientTest.java` (merge with existing imports):
```java
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.getRequestedFor;

import java.util.List;

import io.casehub.connectors.DiscoveredTarget;
```

Add these tests inside `SlackBotClientTest` (after the existing rate-limit section):

```java
// ── Channel discovery ─────────────────────────────────────────────────────────

@Test
void listChannels_returnsDiscoveredTargets() {
    wireMock.stubFor(get(urlEqualTo(
            "/api/conversations.list?types=public_channel,private_channel&limit=200"))
            .willReturn(okJson("{\"ok\":true,\"channels\":["
                    + "{\"id\":\"C123ABC\",\"name\":\"general\"},"
                    + "{\"id\":\"C456DEF\",\"name\":\"engineering\"}"
                    + "]}")));

    final List<DiscoveredTarget> result = client.listChannels("xoxb-test-token");

    assertThat(result).hasSize(2);
    assertThat(result.get(0).id()).isEqualTo("C123ABC");
    assertThat(result.get(0).displayName()).isEqualTo("#general");
    assertThat(result.get(1).id()).isEqualTo("C456DEF");
    assertThat(result.get(1).displayName()).isEqualTo("#engineering");
}

@Test
void listChannels_sendsAuthorizationHeader() {
    wireMock.stubFor(get(urlEqualTo(
            "/api/conversations.list?types=public_channel,private_channel&limit=200"))
            .willReturn(okJson("{\"ok\":true,\"channels\":[]}")));

    client.listChannels("xoxb-my-token");

    wireMock.verify(getRequestedFor(urlEqualTo(
            "/api/conversations.list?types=public_channel,private_channel&limit=200"))
            .withHeader("Authorization", equalTo("Bearer xoxb-my-token")));
}

@Test
void listChannels_slackReturnsNotOk_returnsEmptyList() {
    wireMock.stubFor(get(urlEqualTo(
            "/api/conversations.list?types=public_channel,private_channel&limit=200"))
            .willReturn(okJson("{\"ok\":false,\"error\":\"invalid_auth\"}")));

    final List<DiscoveredTarget> result = client.listChannels("xoxb-bad");

    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot \
  -Dtest=SlackBotClientTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: FAIL — `listChannels` method does not exist yet.

- [ ] **Step 3: Add `ID` constant and `listChannels()` + `parseChannels()` to `SlackBotClient`**

In `SlackBotClient.java`, add these imports:
```java
import java.util.List;
import io.casehub.connectors.DiscoveredTarget;
```

Add `ID` constant immediately after the class declaration:
```java
public static final String ID = "slack-bot";
```

Add `listChannels()` and `parseChannels()` methods before `sendWithRetry()`:
```java
/**
 * Lists channels accessible to the bot.
 *
 * @param token bot token ({@code xoxb-…})
 * @return list of discovered targets; empty on error or empty workspace
 */
public List<DiscoveredTarget> listChannels(final String token) {
    final HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(apiBaseUrl + "/api/conversations.list"
                    + "?types=public_channel,private_channel&limit=200"))
            .header("Authorization", "Bearer " + token)
            .timeout(REQUEST_TIMEOUT)
            .GET()
            .build();
    try {
        final HttpResponse<String> response =
                HttpHelper.CLIENT.send(request, HttpResponse.BodyHandlers.ofString());
        return parseChannels(response.body());
    } catch (final InterruptedException e) {
        Thread.currentThread().interrupt();
        return List.of();
    } catch (final Exception e) {
        LOG.warning("SlackBotClient: listChannels HTTP error — " + e.getMessage());
        return List.of();
    }
}

private List<DiscoveredTarget> parseChannels(final String body) {
    if (body == null || body.isBlank()) return List.of();
    try (var reader = Json.createReader(new StringReader(body))) {
        final JsonObject obj = reader.readObject();
        if (!obj.getBoolean("ok", false)) return List.of();
        return obj.getJsonArray("channels").stream()
                .map(v -> v.asJsonObject())
                .map(ch -> new DiscoveredTarget(
                        ch.getString("id"),
                        "#" + ch.getString("name")))
                .toList();
    } catch (final Exception e) {
        LOG.warning("SlackBotClient: listChannels parse error — " + e.getMessage());
        return List.of();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot \
  -Dtest=SlackBotClientTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All `SlackBotClientTest` tests pass (including the 3 new ones).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java \
  slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotClientTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(slack-bot): SlackBotClient.ID constant and listChannels()

Refs #16"
```

---

## Task 3: `SlackBotDiscovery`

**Files:**
- Create: `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotDiscovery.java`
- Create: `slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotDiscoveryTest.java`

- [ ] **Step 1: Write failing tests in `SlackBotDiscoveryTest`**

```java
package io.casehub.connectors.slack.bot;

import static com.github.tomakehurst.wiremock.client.WireMock.anyRequestedFor;
import static com.github.tomakehurst.wiremock.client.WireMock.anyUrl;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.okJson;
import static com.github.tomakehurst.wiremock.client.WireMock.urlEqualTo;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.DiscoveredTarget;

class SlackBotDiscoveryTest {

    private WireMockServer wireMock;
    private SlackBotClient client;

    @BeforeEach
    void start() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        client = new SlackBotClient();
        client.apiBaseUrl = "http://localhost:" + wireMock.port();
    }

    @AfterEach
    void stop() {
        wireMock.stop();
    }

    @Test
    void connectorId_returnsSlackBotId() {
        final SlackBotDiscovery discovery = new SlackBotDiscovery(client, "any-token");
        assertThat(discovery.connectorId()).isEqualTo(SlackBotClient.ID);
    }

    @Test
    void discover_delegatesToClient_withConfiguredToken() {
        wireMock.stubFor(get(urlEqualTo(
                "/api/conversations.list?types=public_channel,private_channel&limit=200"))
                .willReturn(okJson("{\"ok\":true,\"channels\":["
                        + "{\"id\":\"C111\",\"name\":\"general\"}"
                        + "]}")));

        final SlackBotDiscovery discovery = new SlackBotDiscovery(client, "xoxb-test");
        final List<DiscoveredTarget> result = discovery.discover();

        assertThat(result).hasSize(1);
        assertThat(result.get(0).id()).isEqualTo("C111");
        assertThat(result.get(0).displayName()).isEqualTo("#general");
    }

    @Test
    void discover_blankToken_returnsEmptyListWithoutHttpCall() {
        final SlackBotDiscovery discovery = new SlackBotDiscovery(client, "");

        final List<DiscoveredTarget> result = discovery.discover();

        assertThat(result).isEmpty();
        wireMock.verify(0, WireMock.anyRequestedFor(WireMock.anyUrl()));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot \
  -Dtest=SlackBotDiscoveryTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: FAIL — `SlackBotDiscovery` class does not exist.

- [ ] **Step 3: Create `SlackBotDiscovery.java`**

```java
package io.casehub.connectors.slack.bot;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.casehub.connectors.ConnectorDiscovery;
import io.casehub.connectors.DiscoveredTarget;

/**
 * Implements {@link ConnectorDiscovery} for Slack via the Slack Web API.
 *
 * <p>Holds the MCP-deployment-specific bot token — kept separate from
 * {@link SlackBotClient} so the shared HTTP client is not contaminated with
 * config that is irrelevant to Qhorus consumers.
 */
@ApplicationScoped
public class SlackBotDiscovery implements ConnectorDiscovery {

    private final SlackBotClient slackBotClient;
    private final String botToken;

    @Inject
    SlackBotDiscovery(final SlackBotClient slackBotClient,
                      @ConfigProperty(name = "casehub.connectors.slack-bot.token",
                                      defaultValue = "") final String botToken) {
        this.slackBotClient = slackBotClient;
        this.botToken = botToken;
    }

    @Override
    public String connectorId() {
        return SlackBotClient.ID;
    }

    @Override
    public List<DiscoveredTarget> discover() {
        if (botToken.isBlank()) return List.of();
        return slackBotClient.listChannels(botToken);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot \
  -Dtest=SlackBotDiscoveryTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All 3 `SlackBotDiscoveryTest` tests pass.

- [ ] **Step 5: Run full `slack-bot` test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All `slack-bot` tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotDiscovery.java \
  slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotDiscoveryTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(slack-bot): SlackBotDiscovery implements ConnectorDiscovery

Refs #16"
```

---

## Task 4: `mcp` module — public `McpToolTestSupport` and `pom.xml` deps

**Files:**
- Modify: `mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java`
- Modify: `mcp/pom.xml`

No TDD here — these are mechanical visibility changes and build wiring. The test suite passing is the verification.

- [ ] **Step 1: Make `McpToolTestSupport` fully public**

Replace the entire file content with:

```java
package io.casehub.connectors.mcp;

import java.util.List;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;

/** Shared test doubles for MCP tool unit tests. */
public final class McpToolTestSupport {

    private McpToolTestSupport() {}

    /** Records the last call to {@link Connector#send(ConnectorMessage)}. */
    public static final class RecordingConnector implements Connector {

        private final String id;
        public ConnectorMessage lastMessage;

        public RecordingConnector(final String id) {
            this.id = id;
        }

        @Override
        public String id() {
            return id;
        }

        @Override
        public void send(final ConnectorMessage message) {
            this.lastMessage = message;
        }

        public void reset() {
            lastMessage = null;
        }
    }

    /** Records all calls to {@link ConnectorMeshBridge#notifyDelivered}. */
    public static final class RecordingBridge implements ConnectorMeshBridge {

        public String lastConnectorId;
        public String lastDestination;
        public String lastContent;

        @Override
        public void notifyDelivered(final String connectorId,
                                    final String destination,
                                    final String content) {
            this.lastConnectorId = connectorId;
            this.lastDestination = destination;
            this.lastContent = content;
        }

        public void reset() {
            lastConnectorId = lastDestination = lastContent = null;
        }
    }

    /** Builds a ConnectorService backed only by the given connectors. */
    static ConnectorService serviceWith(final Connector... connectors) {
        return new ConnectorService(List.of(connectors));
    }
}
```

- [ ] **Step 2: Add deps to `mcp/pom.xml`**

Add after the existing `assertj-core` test dependency:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-slack-bot</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
      <scope>test</scope>
    </dependency>
```

Note: `casehub-connectors-slack-bot` is a compile dependency (not test — `mcp` production code will inject `SlackBotClient`). `wiremock-standalone` is test-only; version comes from parent `dependencyManagement` (set in connectors#15).

- [ ] **Step 3: Build `mcp` to verify existing tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All existing mcp tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  mcp/src/test/java/io/casehub/connectors/mcp/McpToolTestSupport.java \
  mcp/pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "chore(mcp): make McpToolTestSupport public; add slack-bot and wiremock deps

Refs #16"
```

---

## Task 5: `SlackBotMcpTool`

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/SlackBotMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/slack/bot/SlackBotMcpToolTest.java`

The test file is in the `mcp` module but in package `io.casehub.connectors.slack.bot`. This is intentional — it gives the test access to `SlackBotClient.apiBaseUrl` (package-private). The path in the filesystem is `mcp/src/test/java/io/casehub/connectors/slack/bot/`.

- [ ] **Step 1: Write failing tests in `SlackBotMcpToolTest`**

```java
package io.casehub.connectors.slack.bot;

import static com.github.tomakehurst.wiremock.client.WireMock.anyRequestedFor;
import static com.github.tomakehurst.wiremock.client.WireMock.anyUrl;
import static com.github.tomakehurst.wiremock.client.WireMock.equalTo;
import static com.github.tomakehurst.wiremock.client.WireMock.matchingJsonPath;
import static com.github.tomakehurst.wiremock.client.WireMock.okJson;
import static com.github.tomakehurst.wiremock.client.WireMock.post;
import static com.github.tomakehurst.wiremock.client.WireMock.postRequestedFor;
import static com.github.tomakehurst.wiremock.client.WireMock.urlEqualTo;
import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.mcp.McpToolTestSupport;
import io.casehub.connectors.mcp.SlackBotMcpTool;

class SlackBotMcpToolTest {

    private WireMockServer wireMock;
    private SlackBotClient client;
    private McpToolTestSupport.RecordingBridge bridge;
    private SlackBotMcpTool tool;

    @BeforeEach
    void start() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        client = new SlackBotClient();
        client.apiBaseUrl = "http://localhost:" + wireMock.port();
        bridge = new McpToolTestSupport.RecordingBridge();
        tool = new SlackBotMcpTool(client, bridge, "xoxb-test-token");
    }

    @AfterEach
    void stop() {
        wireMock.stop();
    }

    // ── Success path ──────────────────────────────────────────────────────────

    @Test
    void sendSlackBot_success_returnsPostedWithTs() {
        wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
                .willReturn(okJson("{\"ok\":true,\"ts\":\"1638535627.000200\"}")));

        final String result = tool.sendSlackBot("C123ABC", "Hello", null);

        assertThat(result).isEqualTo("Posted to C123ABC (ts=1638535627.000200)");
    }

    @Test
    void sendSlackBot_success_bridgeCalledWithSlackBotIdChannelAndSanitizedText() {
        wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
                .willReturn(okJson("{\"ok\":true,\"ts\":\"1638535627.000200\"}")));

        tool.sendSlackBot("C123ABC", "line1\nline2", null);

        assertThat(bridge.lastConnectorId).isEqualTo(SlackBotClient.ID);
        assertThat(bridge.lastDestination).isEqualTo("C123ABC");
        assertThat(bridge.lastContent).isEqualTo("line1 line2");
    }

    @Test
    void sendSlackBot_withThreadTs_passesThreadTsToClient() {
        wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
                .willReturn(okJson("{\"ok\":true,\"ts\":\"1638535628.000300\"}")));

        tool.sendSlackBot("C123ABC", "Reply", "1638535627.000200");

        wireMock.verify(postRequestedFor(urlEqualTo("/api/chat.postMessage"))
                .withRequestBody(matchingJsonPath("$.thread_ts",
                        equalTo("1638535627.000200"))));
    }

    @Test
    void sendSlackBot_longText_contentTruncatedTo500InBridge() {
        wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
                .willReturn(okJson("{\"ok\":true,\"ts\":\"1638535627.000200\"}")));
        final String longBody = "x".repeat(600);

        tool.sendSlackBot("C123ABC", longBody, null);

        assertThat(bridge.lastContent).hasSize(500);
    }

    // ── Failure paths ─────────────────────────────────────────────────────────

    @Test
    void sendSlackBot_blankToken_returnsFailedWithoutHttpCall() {
        final SlackBotMcpTool blankTool = new SlackBotMcpTool(client, bridge, "");

        final String result = blankTool.sendSlackBot("C123ABC", "Hello", null);

        assertThat(result).isEqualTo(
                "Failed: casehub.connectors.slack-bot.token is not configured");
        assertThat(bridge.lastConnectorId).isNull();
        wireMock.verify(0, WireMock.anyRequestedFor(WireMock.anyUrl()));
    }

    @Test
    void sendSlackBot_slackReturnsNotOk_returnsFailedNoBridgeCall() {
        wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
                .willReturn(okJson("{\"ok\":false,\"error\":\"channel_not_found\"}")));

        final String result = tool.sendSlackBot("CBAD", "Hello", null);

        assertThat(result).isEqualTo("Failed: channel_not_found");
        assertThat(bridge.lastConnectorId).isNull();
    }

    @Test
    void sendSlackBot_networkError_returnsFailedString() {
        wireMock.stop();

        final String result = tool.sendSlackBot("C123ABC", "Hello", null);

        assertThat(result).startsWith("Failed:");
        assertThat(bridge.lastConnectorId).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -Dtest="io.casehub.connectors.slack.bot.SlackBotMcpToolTest" \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: FAIL — `SlackBotMcpTool` class does not exist.

- [ ] **Step 3: Create `SlackBotMcpTool.java`**

```java
package io.casehub.connectors.mcp;

import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.smallrye.common.annotation.Blocking;

import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.slack.bot.SlackBotClient;
import io.casehub.connectors.slack.bot.SlackBotClient.PostResult;

@ApplicationScoped
public class SlackBotMcpTool {

    private static final Logger LOG = Logger.getLogger(SlackBotMcpTool.class.getName());

    private final SlackBotClient slackBotClient;
    private final ConnectorMeshBridge meshBridge;
    private final String botToken;

    @Inject
    SlackBotMcpTool(final SlackBotClient slackBotClient,
                    final ConnectorMeshBridge meshBridge,
                    @ConfigProperty(name = "casehub.connectors.slack-bot.token",
                                    defaultValue = "") final String botToken) {
        this.slackBotClient = slackBotClient;
        this.meshBridge = meshBridge;
        this.botToken = botToken;
    }

    @Tool(name = "send_slack_bot",
          description = "Posts a message to a Slack channel using a configured bot token. "
                      + "Returns the message timestamp (ts) on success — save it to reply "
                      + "in-thread. Requires casehub.connectors.slack-bot.token on the server. "
                      + "Returns 'Posted to <channel> (ts=<ts>)' on success or "
                      + "'Failed: <reason>' on error.")
    @Blocking
    public String sendSlackBot(
            @ToolArg(description = "Slack channel ID (e.g. C123ABC). "
                                 + "Use list_channels to discover available IDs.")
            final String channel,
            @ToolArg(description = "Message text.")
            final String text,
            @ToolArg(description = "Thread timestamp for in-thread replies. Use the ts from "
                                 + "a previous send_slack_bot call. Omit for a new message.",
                     required = false)
            final String threadTs) {
        try {
            if (botToken.isBlank()) {
                return "Failed: casehub.connectors.slack-bot.token is not configured";
            }
            final PostResult result =
                    slackBotClient.postMessage(botToken, channel, text, threadTs);
            if (!result.ok()) {
                return "Failed: " + result.error();
            }
            meshBridge.notifyDelivered(SlackBotClient.ID, channel,
                    McpContentSanitizer.sanitize(text));
            return "Posted to " + channel + " (ts=" + result.ts() + ")";
        } catch (final Exception e) {
            LOG.warning("send_slack_bot failed [" + e.getClass().getSimpleName()
                    + "]: " + e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -Dtest="io.casehub.connectors.slack.bot.SlackBotMcpToolTest" \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All 7 `SlackBotMcpToolTest` tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  mcp/src/main/java/io/casehub/connectors/mcp/SlackBotMcpTool.java \
  mcp/src/test/java/io/casehub/connectors/slack/bot/SlackBotMcpToolTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): SlackBotMcpTool — send_slack_bot tool

Refs #16"
```

---

## Task 6: `ChannelDiscoveryMcpTool`

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/ChannelDiscoveryMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/ChannelDiscoveryMcpToolTest.java`

- [ ] **Step 1: Write failing tests in `ChannelDiscoveryMcpToolTest`**

```java
package io.casehub.connectors.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.ConnectorDiscovery;
import io.casehub.connectors.DiscoveredTarget;

class ChannelDiscoveryMcpToolTest {

    private static StubDiscovery stub(final String id,
                                      final List<DiscoveredTarget> targets) {
        return new StubDiscovery(id, targets);
    }

    private static final class StubDiscovery implements ConnectorDiscovery {
        private final String connectorId;
        private final List<DiscoveredTarget> targets;

        StubDiscovery(final String connectorId, final List<DiscoveredTarget> targets) {
            this.connectorId = connectorId;
            this.targets = targets;
        }

        @Override public String connectorId() { return connectorId; }
        @Override public List<DiscoveredTarget> discover() { return targets; }
    }

    private static final class ThrowingDiscovery implements ConnectorDiscovery {
        @Override public String connectorId() { return "bad"; }
        @Override public List<DiscoveredTarget> discover() {
            throw new RuntimeException("simulated failure");
        }
    }

    @Test
    void listChannels_singleConnector_formatsOutput() {
        final ChannelDiscoveryMcpTool tool = new ChannelDiscoveryMcpTool(List.of(
                stub("slack-bot", List.of(
                        new DiscoveredTarget("C123ABC", "#general"),
                        new DiscoveredTarget("C456DEF", "#engineering")))));

        final String result = tool.listChannels();

        assertThat(result).isEqualTo(
                "slack-bot:\n"
                + "  #general (C123ABC)\n"
                + "  #engineering (C456DEF)");
    }

    @Test
    void listChannels_multipleConnectors_formatsAll() {
        final ChannelDiscoveryMcpTool tool = new ChannelDiscoveryMcpTool(List.of(
                stub("slack-bot", List.of(new DiscoveredTarget("C1", "#general"))),
                stub("demo", List.of(new DiscoveredTarget("main", "#main")))));

        final String result = tool.listChannels();

        assertThat(result).contains("slack-bot:");
        assertThat(result).contains("  #general (C1)");
        assertThat(result).contains("demo:");
        assertThat(result).contains("  #main (main)");
    }

    @Test
    void listChannels_emptyDiscover_skipsConnector() {
        final ChannelDiscoveryMcpTool tool = new ChannelDiscoveryMcpTool(List.of(
                stub("slack-bot", List.of())));

        final String result = tool.listChannels();

        assertThat(result).isEqualTo("No channels discovered.");
    }

    @Test
    void listChannels_noConnectors_returnsNoneDiscovered() {
        final ChannelDiscoveryMcpTool tool = new ChannelDiscoveryMcpTool(List.of());

        final String result = tool.listChannels();

        assertThat(result).isEqualTo("No channels discovered.");
    }

    @Test
    void listChannels_discoveryThrows_logsWarnAndContinues() {
        final ChannelDiscoveryMcpTool tool = new ChannelDiscoveryMcpTool(List.of(
                new ThrowingDiscovery(),
                stub("slack-bot", List.of(new DiscoveredTarget("C1", "#general")))));

        final String result = tool.listChannels();

        assertThat(result).isEqualTo("slack-bot:\n  #general (C1)");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -Dtest=ChannelDiscoveryMcpToolTest \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: FAIL — `ChannelDiscoveryMcpTool` does not exist.

- [ ] **Step 3: Create `ChannelDiscoveryMcpTool.java`**

```java
package io.casehub.connectors.mcp;

import java.util.List;
import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkus.arc.All;
import io.quarkiverse.mcp.server.Tool;
import io.smallrye.common.annotation.Blocking;

import io.casehub.connectors.ConnectorDiscovery;
import io.casehub.connectors.DiscoveredTarget;

@ApplicationScoped
public class ChannelDiscoveryMcpTool {

    private static final Logger LOG =
            Logger.getLogger(ChannelDiscoveryMcpTool.class.getName());

    private final List<ConnectorDiscovery> discoveries;

    @Inject
    ChannelDiscoveryMcpTool(@All final List<ConnectorDiscovery> discoveries) {
        this.discoveries = discoveries;
    }

    @Tool(name = "list_channels",
          description = "Lists discoverable channels across all configured connectors "
                      + "(e.g. Slack Bot). Returns channel IDs to use with send_slack_bot. "
                      + "Only connectors with a token configured appear in the output.")
    @Blocking
    public String listChannels() {
        final StringBuilder sb = new StringBuilder();
        for (final ConnectorDiscovery d : discoveries) {
            final List<DiscoveredTarget> targets;
            try {
                targets = d.discover();
            } catch (final Exception e) {
                LOG.warning("ConnectorDiscovery[" + d.connectorId()
                        + "] threw: " + e.getMessage());
                continue;
            }
            if (targets.isEmpty()) continue;
            sb.append(d.connectorId()).append(":\n");
            for (final DiscoveredTarget t : targets) {
                sb.append("  ").append(t.displayName())
                  .append(" (").append(t.id()).append(")\n");
            }
        }
        return sb.isEmpty() ? "No channels discovered." : sb.toString().stripTrailing();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -Dtest=ChannelDiscoveryMcpToolTest \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All 5 `ChannelDiscoveryMcpToolTest` tests pass.

- [ ] **Step 5: Run full `mcp` suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All mcp tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  mcp/src/main/java/io/casehub/connectors/mcp/ChannelDiscoveryMcpTool.java \
  mcp/src/test/java/io/casehub/connectors/mcp/ChannelDiscoveryMcpToolTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(mcp): ChannelDiscoveryMcpTool — list_channels tool

Refs #16"
```

---

## Task 7: `@Blocking` on all five existing MCP tools

**Files:**
- Modify: `mcp/src/main/java/io/casehub/connectors/mcp/SlackMcpTool.java`
- Modify: `mcp/src/main/java/io/casehub/connectors/mcp/TeamsMcpTool.java`
- Modify: `mcp/src/main/java/io/casehub/connectors/mcp/TwilioSmsMcpTool.java`
- Modify: `mcp/src/main/java/io/casehub/connectors/mcp/WhatsAppMcpTool.java`
- Modify: `mcp/src/main/java/io/casehub/connectors/mcp/EmailMcpTool.java`

Each tool calls `HttpHelper.CLIENT.send()` via `ConnectorService`. Without `@Blocking`, the Vert.x I/O thread blocks. This is a correctness fix, not a behavioural change — existing tests continue to pass unchanged.

- [ ] **Step 1: Add `@Blocking` import and annotation to each tool**

In each of the five files, add this import alongside the existing imports:
```java
import io.smallrye.common.annotation.Blocking;
```

Then add `@Blocking` immediately before the `@Tool` annotation on each tool method:

**`SlackMcpTool.java`** — before `@Tool(name = "send_slack", ...)`
```java
@Blocking
@Tool(name = "send_slack", ...)
```

**`TeamsMcpTool.java`** — before `@Tool(name = "send_teams", ...)`
```java
@Blocking
@Tool(name = "send_teams", ...)
```

**`TwilioSmsMcpTool.java`** — before `@Tool(name = "send_sms", ...)`
```java
@Blocking
@Tool(name = "send_sms", ...)
```

**`WhatsAppMcpTool.java`** — before `@Tool(name = "send_whatsapp", ...)`
```java
@Blocking
@Tool(name = "send_whatsapp", ...)
```

**`EmailMcpTool.java`** — before `@Tool(name = "send_email", ...)`
```java
@Blocking
@Tool(name = "send_email", ...)
```

- [ ] **Step 2: Run full `mcp` test suite to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All mcp tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  mcp/src/main/java/io/casehub/connectors/mcp/SlackMcpTool.java \
  mcp/src/main/java/io/casehub/connectors/mcp/TeamsMcpTool.java \
  mcp/src/main/java/io/casehub/connectors/mcp/TwilioSmsMcpTool.java \
  mcp/src/main/java/io/casehub/connectors/mcp/WhatsAppMcpTool.java \
  mcp/src/main/java/io/casehub/connectors/mcp/EmailMcpTool.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "fix(mcp): add @Blocking to all five existing MCP tool methods

All tools call HttpHelper.CLIENT.send() via ConnectorService.
Without @Blocking, the Vert.x I/O thread blocks. GE-20260604-96d82a.

Refs #16"
```

---

## Task 8: Full build and final verification

- [ ] **Step 1: Run complete reactor build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install \
  -f /Users/mdproctor/claude/casehub/connectors/pom.xml
```

Expected: BUILD SUCCESS. All modules build. Test counts:
- `core`: existing tests pass
- `slack-bot`: existing tests + 3 `listChannels` + 3 `SlackBotDiscovery` tests pass
- `mcp`: existing tests + 7 `SlackBotMcpTool` + 5 `ChannelDiscoveryMcpTool` tests pass

If any test fails, investigate and fix before proceeding to code review.
