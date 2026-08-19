# Signal Chat Platform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create before starting work
**Issue group:** single issue

**Goal:** Add a Signal messenger `ChatPlatform` SPI implementation with 6 native capabilities, backed by signal-cli-rest-api over HTTP/WebSocket.

**Architecture:** Two new Maven modules (`signal-cli` for the HTTP/WebSocket client, `chat-signal` for the ChatPlatform SPI) plus a `SignalConnector` in core for notification-bridge. The connector talks to an external `bbernhard/signal-cli-rest-api` Docker container — no AGPL code in the JVM.

**Tech Stack:** Java 21, Quarkus 3.32.2, java.net.http.HttpClient, java.net.http.WebSocket, Jackson, WireMock (tests)

## Global Constraints

- Java 21 source level, Java 26 JVM
- Quarkus 3.32.2
- All artifacts version `0.2-SNAPSHOT`
- Parent: `io.casehub:casehub-connectors-parent:0.2-SNAPSHOT`
- No AGPL-licensed code as a compile/runtime dependency
- jandex-maven-plugin 3.3.1 required in every module producing CDI beans
- Config properties: `casehub.signal.api-url`, `casehub.signal.number`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

## Batch 1: signal-cli HTTP Client Module

### Task 1: Maven module scaffold + SignalClient with send and health

**Files:**
- Create: `signal-cli/pom.xml`
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/SignalClient.java`
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/model/SendResponse.java`
- Modify: `pom.xml` (parent — add `<module>signal-cli</module>`)
- Test: `signal-cli/src/test/java/io/casehub/connectors/signal/cli/SignalClientTest.java`

**Interfaces:**
- Consumes: nothing (foundation module)
- Produces:
  - `SignalClient(String apiUrl)` — constructor
  - `SendResponse send(String number, String recipient, String message, List<String> base64Attachments)` — sends message; detects `+` prefix for 1:1 vs group
  - `boolean health()` — returns true if container healthy
  - `SendResponse` record: `boolean ok`, `String timestamp`

- [ ] **Step 1: Create `signal-cli/pom.xml`**

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

  <artifactId>casehub-connectors-signal-cli</artifactId>
  <name>CaseHub Connectors — Signal CLI</name>
  <description>HTTP/WebSocket client for signal-cli-rest-api.
Uses java.net.http — no AGPL dependencies.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
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

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>signal-cli</module>` to the `<modules>` section in the parent pom, after `discord`.

- [ ] **Step 3: Create `SendResponse` record**

```java
package io.casehub.connectors.signal.cli.model;

public record SendResponse(boolean ok, String timestamp) {
    public static SendResponse success(String timestamp) {
        return new SendResponse(true, timestamp);
    }
    public static SendResponse failure() {
        return new SendResponse(false, null);
    }
}
```

- [ ] **Step 4: Write failing test for `SignalClient.send()` — 1:1 message**

```java
package io.casehub.connectors.signal.cli;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.signal.cli.model.SendResponse;

class SignalClientTest {

    private WireMockServer wm;
    private SignalClient client;

    @BeforeEach
    void setUp() {
        wm = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wm.start();
        client = new SignalClient("http://localhost:" + wm.port());
    }

    @AfterEach
    void tearDown() {
        wm.stop();
    }

    @Test
    void send_1to1_message() {
        wm.stubFor(post(urlEqualTo("/v2/send"))
                .willReturn(okJson("{\"timestamp\":\"1724025600000\"}")));

        SendResponse result = client.send("+15551000000", "+15552000000",
                "Hello", List.of());

        assertThat(result.ok()).isTrue();
        assertThat(result.timestamp()).isEqualTo("1724025600000");

        wm.verify(postRequestedFor(urlEqualTo("/v2/send"))
                .withRequestBody(matchingJsonPath("$.number", equalTo("+15551000000")))
                .withRequestBody(matchingJsonPath("$.recipients[0]", equalTo("+15552000000")))
                .withRequestBody(matchingJsonPath("$.message", equalTo("Hello"))));
    }

    @Test
    void send_group_message() {
        wm.stubFor(post(urlEqualTo("/v2/send"))
                .willReturn(okJson("{\"timestamp\":\"1724025600001\"}")));

        SendResponse result = client.send("+15551000000", "dGVzdGdyb3VwaWQ=",
                "Group hello", List.of());

        assertThat(result.ok()).isTrue();

        wm.verify(postRequestedFor(urlEqualTo("/v2/send"))
                .withRequestBody(matchingJsonPath("$.number", equalTo("+15551000000")))
                .withRequestBody(matchingJsonPath("$.base64_group_id", equalTo("dGVzdGdyb3VwaWQ=")))
                .withRequestBody(matchingJsonPath("$.message", equalTo("Group hello"))));
    }

    @Test
    void health_returns_true_when_healthy() {
        wm.stubFor(get(urlEqualTo("/v1/health")).willReturn(aResponse().withStatus(204)));

        assertThat(client.health()).isTrue();
    }

    @Test
    void health_returns_false_when_unreachable() {
        SignalClient bad = new SignalClient("http://localhost:1");

        assertThat(bad.health()).isFalse();
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signal-cli -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — `SignalClient` does not exist yet

- [ ] **Step 6: Implement `SignalClient` — send + health**

```java
package io.casehub.connectors.signal.cli;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;

import io.casehub.connectors.signal.cli.model.SendResponse;

public class SignalClient {

    private static final Logger LOG = Logger.getLogger(SignalClient.class.getName());
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Duration TIMEOUT = Duration.ofSeconds(30);

    private final String apiUrl;
    private final HttpClient http;

    public SignalClient(final String apiUrl) {
        this.apiUrl = apiUrl;
        this.http = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();
    }

    public SendResponse send(final String number, final String recipient,
                             final String message, final List<String> base64Attachments) {
        try {
            final ObjectNode body = MAPPER.createObjectNode();
            body.put("number", number);
            body.put("message", message);

            if (recipient.startsWith("+")) {
                final ArrayNode recipients = body.putArray("recipients");
                recipients.add(recipient);
            } else {
                body.put("base64_group_id", recipient);
            }

            if (base64Attachments != null && !base64Attachments.isEmpty()) {
                final ArrayNode atts = body.putArray("base64_attachments");
                base64Attachments.forEach(atts::add);
            }

            final HttpResponse<String> resp = http.send(
                    HttpRequest.newBuilder()
                            .uri(URI.create(apiUrl + "/v2/send"))
                            .header("Content-Type", "application/json")
                            .POST(HttpRequest.BodyPublishers.ofString(MAPPER.writeValueAsString(body)))
                            .timeout(TIMEOUT)
                            .build(),
                    HttpResponse.BodyHandlers.ofString());

            if (resp.statusCode() >= 200 && resp.statusCode() < 300) {
                final JsonNode json = MAPPER.readTree(resp.body());
                final String ts = json.has("timestamp") ? json.get("timestamp").asText() : null;
                return SendResponse.success(ts);
            }
            LOG.warning("signal-cli send failed: HTTP " + resp.statusCode() + " — " + resp.body());
            return SendResponse.failure();
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "signal-cli send error", e);
            return SendResponse.failure();
        }
    }

    public boolean health() {
        try {
            final HttpResponse<Void> resp = http.send(
                    HttpRequest.newBuilder()
                            .uri(URI.create(apiUrl + "/v1/health"))
                            .GET()
                            .timeout(Duration.ofSeconds(5))
                            .build(),
                    HttpResponse.BodyHandlers.discarding());
            return resp.statusCode() >= 200 && resp.statusCode() < 300;
        } catch (final Exception e) {
            return false;
        }
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signal-cli -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 4 tests PASS

- [ ] **Step 8: Commit**

```bash
git add signal-cli/ pom.xml
git commit -m "feat(signal): add signal-cli module with SignalClient send + health"
```

### Task 2: SignalClient — groups, contacts, reactions, members, attachments

**Files:**
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/model/SignalGroup.java`
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/model/SignalContact.java`
- Modify: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/SignalClient.java`
- Test: `signal-cli/src/test/java/io/casehub/connectors/signal/cli/SignalClientTest.java` (extend)

**Interfaces:**
- Consumes: `SignalClient` from Task 1
- Produces:
  - `List<SignalGroup> listGroups(String number)` — `GET /v1/groups/{n}`
  - `SignalGroup getGroup(String number, String groupId)` — `GET /v1/groups/{n}/{gid}`
  - `SignalGroup createGroup(String number, String name, List<String> members)` — `POST /v1/groups/{n}`
  - `void deleteGroup(String number, String groupId)` — `DELETE /v1/groups/{n}/{gid}`
  - `void addMembers(String number, String groupId, List<String> members)` — `POST /v1/groups/{n}/{gid}/members`
  - `void removeMembers(String number, String groupId, List<String> members)` — `DELETE /v1/groups/{n}/{gid}/members`
  - `void addReaction(String number, String recipient, String emoji, String targetAuthor, long targetTimestamp)` — `POST /v1/reactions/{n}`
  - `void removeReaction(String number, String recipient, String emoji, String targetAuthor, long targetTimestamp)` — `DELETE /v1/reactions/{n}`
  - `List<SignalContact> listContacts(String number)` — `GET /v1/contacts/{n}`
  - `byte[] downloadAttachment(String attachmentId)` — `GET /v1/attachments/{id}`
  - `SignalGroup` record: `String id`, `String name`, `String description`, `List<String> members`
  - `SignalContact` record: `String number`, `String profileName`

- [ ] **Step 1: Create model records**

`SignalGroup.java`:
```java
package io.casehub.connectors.signal.cli.model;

import java.util.List;

public record SignalGroup(String id, String name, String description, List<String> members) {}
```

`SignalContact.java`:
```java
package io.casehub.connectors.signal.cli.model;

public record SignalContact(String number, String profileName) {}
```

- [ ] **Step 2: Write failing tests for group operations**

Add to `SignalClientTest`:
```java
@Test
void listGroups() {
    wm.stubFor(get(urlEqualTo("/v1/groups/+15551000000"))
            .willReturn(okJson("[{\"id\":\"Z3JvdXAx\",\"name\":\"Test Group\",\"description\":\"A group\",\"members\":[\"+15552000000\"]}]")));

    List<SignalGroup> groups = client.listGroups("+15551000000");

    assertThat(groups).hasSize(1);
    assertThat(groups.get(0).name()).isEqualTo("Test Group");
    assertThat(groups.get(0).id()).isEqualTo("Z3JvdXAx");
    assertThat(groups.get(0).members()).containsExactly("+15552000000");
}

@Test
void createGroup() {
    wm.stubFor(post(urlEqualTo("/v1/groups/+15551000000"))
            .willReturn(okJson("{\"id\":\"bmV3Z3JvdXA=\",\"name\":\"New Group\"}")));

    SignalGroup group = client.createGroup("+15551000000", "New Group",
            List.of("+15552000000"));

    assertThat(group).isNotNull();
    assertThat(group.id()).isEqualTo("bmV3Z3JvdXA=");
}

@Test
void addReaction() {
    wm.stubFor(post(urlEqualTo("/v1/reactions/+15551000000"))
            .willReturn(aResponse().withStatus(204)));

    client.addReaction("+15551000000", "+15552000000", "👍",
            "+15552000000", 1724025600000L);

    wm.verify(postRequestedFor(urlEqualTo("/v1/reactions/+15551000000"))
            .withRequestBody(matchingJsonPath("$.reaction", equalTo("👍")))
            .withRequestBody(matchingJsonPath("$.target_author", equalTo("+15552000000")))
            .withRequestBody(matchingJsonPath("$.timestamp", equalTo("1724025600000"))));
}

@Test
void listContacts() {
    wm.stubFor(get(urlEqualTo("/v1/contacts/+15551000000"))
            .willReturn(okJson("[{\"number\":\"+15553000000\",\"profileName\":\"Alice\"}]")));

    List<SignalContact> contacts = client.listContacts("+15551000000");

    assertThat(contacts).hasSize(1);
    assertThat(contacts.get(0).profileName()).isEqualTo("Alice");
}

@Test
void downloadAttachment() {
    byte[] content = new byte[]{1, 2, 3, 4};
    wm.stubFor(get(urlEqualTo("/v1/attachments/abc123"))
            .willReturn(aResponse().withBody(content)
                    .withHeader("Content-Type", "application/octet-stream")));

    byte[] result = client.downloadAttachment("abc123");

    assertThat(result).isEqualTo(content);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signal-cli -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — methods don't exist yet

- [ ] **Step 4: Implement all remaining SignalClient methods**

Add to `SignalClient.java` — `listGroups`, `getGroup`, `createGroup`, `deleteGroup`, `addMembers`, `removeMembers`, `addReaction`, `removeReaction`, `listContacts`, `downloadAttachment`. Each method follows the same pattern: build request, send via `http`, parse response via `MAPPER`. Group endpoints use URL path `"/v1/groups/" + number`, reaction endpoints use `"/v1/reactions/" + number`, contacts use `"/v1/contacts/" + number`, attachments use `"/v1/attachments/" + attachmentId`.

Key implementation details:
- `addReaction`/`removeReaction` body: `{"recipient": recipient, "reaction": emoji, "target_author": targetAuthor, "timestamp": targetTimestamp}`
- `addMembers`/`removeMembers` body: `{"members": ["+15552000000"]}`
- `createGroup` body: `{"name": name, "members": ["+15552000000"]}`
- `downloadAttachment` returns `byte[]` from `HttpResponse.BodyHandlers.ofByteArray()`
- `deleteGroup` uses HTTP DELETE, returns void
- All methods log warnings on failure; group/contact list methods return `List.of()` on error

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signal-cli -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add signal-cli/
git commit -m "feat(signal): add SignalClient group, contact, reaction, and attachment methods"
```

---

## Batch 2: chat-signal ChatPlatform Module

### Task 3: Maven module scaffold + SignalChatPlatform with 6 native capabilities

**Files:**
- Create: `chat-signal/pom.xml`
- Create: `chat-signal/src/main/java/io/casehub/connectors/chat/signal/SignalChatPlatform.java`
- Modify: `pom.xml` (parent — add `<module>chat-signal</module>`)
- Test: `chat-signal/src/test/java/io/casehub/connectors/chat/signal/SignalChatPlatformTest.java`

**Interfaces:**
- Consumes: `SignalClient` (Task 1-2), `ChatPlatform` SPI, all capability interfaces, degraded implementations
- Produces:
  - `SignalChatPlatform` — `@ApplicationScoped` bean, `id()` returns `"signal"`
  - `supports(Class<?>)` — returns true for Messaging, Discovery, Members, Reactions, ChannelManagement, MemberManagement
  - Graceful degradation when unconfigured

- [ ] **Step 1: Create `chat-signal/pom.xml`**

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

  <artifactId>casehub-connectors-chat-signal</artifactId>
  <name>CaseHub Connectors — Chat Signal</name>
  <description>ChatPlatform SPI implementation for Signal via signal-cli-rest-api.
6 native capabilities: Messaging, Discovery, Members, Reactions,
ChannelManagement, MemberManagement.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-signal-cli</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-spi</artifactId>
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

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>chat-signal</module>` after `signal-cli` in `<modules>`.

- [ ] **Step 3: Write failing tests for SignalChatPlatform**

```java
package io.casehub.connectors.chat.signal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import java.util.List;
import java.util.Set;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.model.*;
import io.casehub.connectors.chat.spi.*;
import io.casehub.connectors.signal.cli.SignalClient;
import io.casehub.connectors.signal.cli.model.*;

class SignalChatPlatformTest {

    private SignalClient client;

    @BeforeEach
    void setUp() {
        client = mock(SignalClient.class);
        when(client.health()).thenReturn(true);
    }

    @Test
    void id_returns_signal() {
        SignalChatPlatform platform = createPlatform();
        assertThat(platform.id()).isEqualTo("signal");
    }

    @Test
    void supports_native_capabilities() {
        SignalChatPlatform platform = createPlatform();
        assertThat(platform.supports(Messaging.class)).isTrue();
        assertThat(platform.supports(Discovery.class)).isTrue();
        assertThat(platform.supports(Members.class)).isTrue();
        assertThat(platform.supports(Reactions.class)).isTrue();
        assertThat(platform.supports(ChannelManagement.class)).isTrue();
        assertThat(platform.supports(MemberManagement.class)).isTrue();
    }

    @Test
    void does_not_support_degraded_capabilities() {
        SignalChatPlatform platform = createPlatform();
        assertThat(platform.supports(Threading.class)).isFalse();
        assertThat(platform.supports(Presence.class)).isFalse();
        assertThat(platform.supports(MessageHistory.class)).isFalse();
    }

    @Test
    void discovery_returns_groups_and_contacts() {
        when(client.listGroups("+15551000000")).thenReturn(List.of(
                new SignalGroup("Z3JvdXAx", "Dev Team", "Engineering chat",
                        List.of("+15552000000", "+15553000000"))));
        when(client.listContacts("+15551000000")).thenReturn(List.of(
                new SignalContact("+15554000000", "Alice")));

        SignalChatPlatform platform = createPlatform();
        List<Channel> channels = platform.discovery().listChannels();

        assertThat(channels).hasSize(2);
        Channel group = channels.stream()
                .filter(c -> c.ref().id().equals("Z3JvdXAx")).findFirst().orElseThrow();
        assertThat(group.name()).isEqualTo("Dev Team");
        assertThat(group.topic()).isEqualTo("Engineering chat");
        assertThat(group.isPrivate()).isFalse();
        assertThat(group.memberCount()).isEqualTo(2);

        Channel contact = channels.stream()
                .filter(c -> c.ref().id().equals("+15554000000")).findFirst().orElseThrow();
        assertThat(contact.name()).isEqualTo("Alice");
        assertThat(contact.isPrivate()).isTrue();
        assertThat(contact.memberCount()).isEqualTo(2);
    }

    @Test
    void send_message_builds_correct_message_ref() {
        when(client.send(eq("+15551000000"), eq("+15552000000"), eq("Hello"), any()))
                .thenReturn(SendResponse.success("1724025600000"));

        SignalChatPlatform platform = createPlatform();
        SendResult result = platform.messaging().send(
                new ChatChannelRef("+15552000000"), new ChatContent("Hello"));

        assertThat(result.ok()).isTrue();
        assertThat(result.messageRef().messageId()).isEqualTo("+15551000000:1724025600000");
    }

    @Test
    void degrades_when_unconfigured() {
        SignalChatPlatform platform = new SignalChatPlatform(client, "", "");
        platform.init();

        assertThat(platform.supports(Messaging.class)).isFalse();
        assertThat(platform.discovery().listChannels()).isEmpty();
    }

    private SignalChatPlatform createPlatform() {
        SignalChatPlatform p = new SignalChatPlatform(client,
                "http://localhost:8080", "+15551000000");
        p.init();
        return p;
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-signal -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure

- [ ] **Step 5: Implement `SignalChatPlatform`**

Full `@ApplicationScoped` bean implementing `ChatPlatform` with:
- Constructor injection: `SignalClient`, `@ConfigProperty casehub.signal.api-url`, `@ConfigProperty casehub.signal.number`
- `@PostConstruct init()`: check config + health, set up capabilities or degrade
- 6 native capabilities wired to `SignalClient` methods
- 3 degraded: `ChannelFallbackThreading`, `UnknownPresence`, `EmptyMessageHistory`
- `NATIVE_CAPABILITIES = Set.of(Messaging.class, Discovery.class, Members.class, Reactions.class, ChannelManagement.class, MemberManagement.class)`
- Message identity: `messageRef = senderNumber + ":" + timestamp`
- Discovery: concatenate `listGroups()` mapped to `Channel` + `listContacts()` mapped to `Channel`
- Reactions.list() returns `Collections.emptyList()`
- ChannelManagement: `create()`/`delete()` throw `IllegalArgumentException` for phone-number IDs
- MemberManagement: delegates to `SignalClient.addMembers()`/`removeMembers()`; throws for phone-number channels

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-signal -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add chat-signal/ pom.xml
git commit -m "feat(signal): add chat-signal module with SignalChatPlatform — 6 native capabilities"
```

---

## Batch 3: Core Constants + SignalConnector + Inbound Pipeline

### Task 4: Core constants + SignalConnector for notification-bridge

**Files:**
- Modify: `core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java` — add `SIGNAL`
- Modify: `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java` — add `SIGNAL_INBOUND`
- Create: `core/src/main/java/io/casehub/connectors/signal/SignalConnector.java`
- Test: `core/src/test/java/io/casehub/connectors/signal/SignalConnectorTest.java`

**Interfaces:**
- Consumes: `Connector` SPI, `HttpHelper`, `InboundConnectorTypes`, `InboundConnectorIds`
- Produces:
  - `InboundConnectorTypes.SIGNAL = "signal"`
  - `InboundConnectorIds.SIGNAL_INBOUND = "signal-inbound"`
  - `SignalConnector` — `Connector` SPI bean, `id()="signal"`, `channelType()="signal"`

- [ ] **Step 1: Add constants to `InboundConnectorTypes` and `InboundConnectorIds`**

Add `public static final String SIGNAL = "signal";` to `InboundConnectorTypes`.
Add `public static final String SIGNAL_INBOUND = "signal-inbound";` to `InboundConnectorIds`.

- [ ] **Step 2: Write failing test for `SignalConnector`**

```java
package io.casehub.connectors.signal;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.ConnectorMessage;

class SignalConnectorTest {

    private WireMockServer wm;
    private SignalConnector connector;

    @BeforeEach
    void setUp() {
        wm = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wm.start();
        connector = new SignalConnector();
        connector.apiUrl = "http://localhost:" + wm.port();
        connector.number = "+15551000000";
    }

    @AfterEach
    void tearDown() {
        wm.stop();
    }

    @Test
    void id_and_channelType() {
        assertThat(connector.id()).isEqualTo("signal");
        assertThat(connector.channelType()).isEqualTo("signal");
    }

    @Test
    void send_1to1_formats_recipients_field() {
        wm.stubFor(post(urlEqualTo("/v2/send"))
                .willReturn(okJson("{\"timestamp\":\"123\"}")));

        boolean ok = connector.send(new ConnectorMessage(
                "+15552000000", "Alert", "System down"));

        assertThat(ok).isTrue();
        wm.verify(postRequestedFor(urlEqualTo("/v2/send"))
                .withRequestBody(containing("\"recipients\"")));
    }

    @Test
    void send_group_formats_base64_group_id_field() {
        wm.stubFor(post(urlEqualTo("/v2/send"))
                .willReturn(okJson("{\"timestamp\":\"123\"}")));

        boolean ok = connector.send(new ConnectorMessage(
                "Z3JvdXAx", null, "Group notification"));

        assertThat(ok).isTrue();
        wm.verify(postRequestedFor(urlEqualTo("/v2/send"))
                .withRequestBody(containing("\"base64_group_id\"")));
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure

- [ ] **Step 4: Implement `SignalConnector`**

```java
package io.casehub.connectors.signal;

import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.http.HttpHelper;

@ApplicationScoped
public class SignalConnector implements Connector {

    private static final Logger LOG = Logger.getLogger(SignalConnector.class.getName());

    @ConfigProperty(name = "casehub.signal.api-url", defaultValue = "")
    String apiUrl;

    @ConfigProperty(name = "casehub.signal.number", defaultValue = "")
    String number;

    @Override
    public String id() {
        return "signal";
    }

    @Override
    public boolean send(final ConnectorMessage message) {
        if (apiUrl.isBlank() || number.isBlank()) {
            LOG.warning("Signal connector not configured");
            return false;
        }

        final String dest = message.destination();
        final String text = message.title() != null && !message.title().isBlank()
                ? "*" + message.title() + "*\n" + (message.body() != null ? message.body() : "")
                : message.body();

        final StringBuilder json = new StringBuilder("{");
        json.append("\"number\":").append(HttpHelper.jsonQuote(number));
        json.append(",\"message\":").append(HttpHelper.jsonQuote(text));

        if (dest.startsWith("+")) {
            json.append(",\"recipients\":[").append(HttpHelper.jsonQuote(dest)).append("]");
        } else {
            json.append(",\"base64_group_id\":").append(HttpHelper.jsonQuote(dest));
        }
        json.append("}");

        return HttpHelper.postJson(apiUrl + "/v2/send", json.toString());
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add core/
git commit -m "feat(signal): add SignalConnector + SIGNAL constants to core"
```

### Task 5: SignalInboundConnector + SignalInboundTranslator

**Files:**
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/SignalWebSocket.java`
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/SignalEventListener.java`
- Create: `signal-cli/src/main/java/io/casehub/connectors/signal/cli/model/SignalMessage.java`
- Create: `chat-signal/src/main/java/io/casehub/connectors/chat/signal/SignalInboundConnector.java`
- Create: `chat-signal/src/main/java/io/casehub/connectors/chat/signal/SignalInboundTranslator.java`
- Test: `signal-cli/src/test/java/io/casehub/connectors/signal/cli/SignalWebSocketTest.java`
- Test: `chat-signal/src/test/java/io/casehub/connectors/chat/signal/SignalInboundConnectorTest.java`
- Test: `chat-signal/src/test/java/io/casehub/connectors/chat/signal/SignalInboundTranslatorTest.java`

**Interfaces:**
- Consumes: `SignalClient` (Task 1-2), `InboundConnector` SPI, `InboundTranslator` SPI, `InboundConnectorTypes.SIGNAL`, `InboundConnectorIds.SIGNAL_INBOUND`
- Produces:
  - `SignalEventListener` — functional interface: `void onEvent(SignalMessage message)`
  - `SignalMessage` record: `String sender`, `long timestamp`, `String groupId`, `String message`, `List<String> attachmentIds`, `String quoteSender`, `Long quoteTimestamp`
  - `SignalWebSocket` — connects, receives JSON events, parses to `SignalMessage`, dispatches to listener, reconnects on failure
  - `SignalInboundConnector` — `InboundConnector` SPI bean, `id()="signal-inbound"`, manages WebSocket lifecycle
  - `SignalInboundTranslator` — `InboundTranslator` SPI bean, `connectorType()="signal"`, translates `InboundMessage` → `ReceivedMessage`

- [ ] **Step 1: Create `SignalMessage` record and `SignalEventListener` interface**

`SignalMessage.java`:
```java
package io.casehub.connectors.signal.cli.model;

import java.util.List;

public record SignalMessage(
        String sender,
        long timestamp,
        String groupId,
        String message,
        List<String> attachmentIds,
        String quoteSender,
        Long quoteTimestamp) {

    public String channelRef() {
        return groupId != null ? groupId : sender;
    }
}
```

`SignalEventListener.java`:
```java
package io.casehub.connectors.signal.cli;

import io.casehub.connectors.signal.cli.model.SignalMessage;

public interface SignalEventListener {
    void onMessage(SignalMessage message);
}
```

- [ ] **Step 2: Write failing test for `SignalWebSocket` event parsing**

```java
package io.casehub.connectors.signal.cli;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.connectors.signal.cli.model.SignalMessage;

class SignalWebSocketTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void parses_direct_message_event() throws Exception {
        String json = """
                {"envelope":{"source":"+15552000000","timestamp":1724025600000,
                "dataMessage":{"message":"Hello","attachments":[]}}}""";

        List<SignalMessage> received = new ArrayList<>();
        SignalMessage msg = SignalWebSocket.parseEvent(json);

        assertThat(msg).isNotNull();
        assertThat(msg.sender()).isEqualTo("+15552000000");
        assertThat(msg.timestamp()).isEqualTo(1724025600000L);
        assertThat(msg.message()).isEqualTo("Hello");
        assertThat(msg.groupId()).isNull();
    }

    @Test
    void parses_group_message_event() throws Exception {
        String json = """
                {"envelope":{"source":"+15552000000","timestamp":1724025600001,
                "dataMessage":{"message":"Group msg","groupInfo":{"groupId":"Z3JvdXAx"},
                "attachments":[]}}}""";

        SignalMessage msg = SignalWebSocket.parseEvent(json);

        assertThat(msg).isNotNull();
        assertThat(msg.groupId()).isEqualTo("Z3JvdXAx");
    }

    @Test
    void parses_quote_reply() throws Exception {
        String json = """
                {"envelope":{"source":"+15552000000","timestamp":1724025600002,
                "dataMessage":{"message":"Reply","attachments":[],
                "quote":{"id":1724025600000,"author":"+15553000000"}}}}""";

        SignalMessage msg = SignalWebSocket.parseEvent(json);

        assertThat(msg.quoteSender()).isEqualTo("+15553000000");
        assertThat(msg.quoteTimestamp()).isEqualTo(1724025600000L);
    }

    @Test
    void returns_null_for_non_message_events() throws Exception {
        String json = """
                {"envelope":{"source":"+15552000000","timestamp":123,
                "typingMessage":{"action":"STARTED"}}}""";

        assertThat(SignalWebSocket.parseEvent(json)).isNull();
    }
}
```

- [ ] **Step 3: Implement `SignalWebSocket` — event parsing + WebSocket lifecycle**

Static `parseEvent(String json)` method parses signal-cli-rest-api JSON format. The WebSocket lifecycle connects via `java.net.http.WebSocket`, calls `parseEvent()` on each text frame, dispatches non-null results to the `SignalEventListener`. Reconnection with exponential backoff (1s, 2s, 4s, 8s, max 30s) + jitter.

- [ ] **Step 4: Run WebSocket tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signal-cli -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 5: Write failing test for `SignalInboundTranslator`**

```java
package io.casehub.connectors.chat.signal;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ReceivedMessage;

class SignalInboundTranslatorTest {

    private final SignalInboundTranslator translator = new SignalInboundTranslator();

    @Test
    void connectorType_returns_signal() {
        assertThat(translator.connectorType()).isEqualTo("signal");
    }

    @Test
    void translates_direct_message() {
        InboundMessage msg = new InboundMessage(
                "signal-inbound", "signal", "+15552000000", "+15552000000",
                "Hello", List.of(), Instant.parse("2024-08-19T12:00:00Z"),
                Map.of("signal-sender", "+15552000000",
                       "signal-timestamp", "1724025600000"),
                null);

        ReceivedMessage result = translator.translate(msg);

        assertThat(result.platformId()).isEqualTo("signal");
        assertThat(result.channel().id()).isEqualTo("+15552000000");
        assertThat(result.messageRef().messageId()).isEqualTo("+15552000000:1724025600000");
        assertThat(result.parentRef()).isNull();
        assertThat(result.sender().id()).isEqualTo("+15552000000");
    }

    @Test
    void translates_quote_reply_with_parent_ref() {
        InboundMessage msg = new InboundMessage(
                "signal-inbound", "signal", "+15552000000", "Z3JvdXAx",
                "Reply", List.of(), Instant.parse("2024-08-19T12:01:00Z"),
                Map.of("signal-sender", "+15552000000",
                       "signal-timestamp", "1724025600060",
                       "signal-quote-sender", "+15553000000",
                       "signal-quote-timestamp", "1724025600000"),
                null);

        ReceivedMessage result = translator.translate(msg);

        assertThat(result.parentRef()).isNotNull();
        assertThat(result.parentRef().messageId()).isEqualTo("+15553000000:1724025600000");
    }
}
```

- [ ] **Step 6: Implement `SignalInboundTranslator`**

```java
package io.casehub.connectors.chat.signal;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.*;
import io.casehub.connectors.chat.spi.InboundTranslator;

@ApplicationScoped
public class SignalInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return InboundConnectorTypes.SIGNAL;
    }

    @Override
    public ReceivedMessage translate(final InboundMessage msg) {
        final var channel = new ChatChannelRef(msg.externalChannelRef());
        final var messageRef = new ChatMessageRef(channel,
                msg.metadata().get("signal-sender") + ":"
                + msg.metadata().get("signal-timestamp"));

        final String quoteSender = msg.metadata().get("signal-quote-sender");
        final String quoteTs = msg.metadata().get("signal-quote-timestamp");
        final ChatMessageRef parentRef = quoteSender != null && quoteTs != null
                ? new ChatMessageRef(channel, quoteSender + ":" + quoteTs)
                : null;

        return new ReceivedMessage(
                InboundConnectorTypes.SIGNAL,
                channel,
                messageRef,
                parentRef,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content(), null, msg.attachments(), List.of()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 7: Write failing test for `SignalInboundConnector`**

Test that `start(sink)` wires up correctly and `id()` returns `"signal-inbound"`. Test `stop()` disconnects. Use a mock `SignalClient` and verify `InboundMessage` construction from a simulated event.

- [ ] **Step 8: Implement `SignalInboundConnector`**

`@ApplicationScoped` bean implementing `InboundConnector`. `start(sink)` creates `SignalWebSocket`, connects, registers a `SignalEventListener` that constructs `InboundMessage` from `SignalMessage` events and calls `sink.receive()`. `stop()` disconnects WebSocket. Attachment download via `SignalClient.downloadAttachment()` on virtual threads, with failure tracking in metadata.

- [ ] **Step 9: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signal-cli,chat-signal -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add signal-cli/ chat-signal/
git commit -m "feat(signal): add SignalInboundConnector + SignalInboundTranslator — full inbound pipeline"
```

---

## Batch 4: Full Build Verification

### Task 6: Full build + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — update "What This Project Is" section with Signal connector details

**Interfaces:**
- Consumes: all prior tasks
- Produces: green full build, updated docs

- [ ] **Step 1: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 2: Fix any compilation or test failures**

If failures: diagnose, fix, re-run.

- [ ] **Step 3: Update CLAUDE.md**

Add `chat-signal` (Signal with 6 native capabilities) to the ChatPlatform implementations list. Add `signal-cli` to the shared HTTP clients list. Add `send_signal` MCP tool mention if applicable.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with Signal connector"
```

## References

- `specs/signal-chat-platform/2026-08-19-signal-chat-platform-design.md` — design spec this plan implements
- `specs/signal-chat-platform/decisions.md` — D1–D5 design decisions
- `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java:42` — SPI interface
- `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java:27` — closest analogue
- `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundConnector.java:30` — inbound pattern
- `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundTranslator.java:18` — translator pattern
- `core/src/main/java/io/casehub/connectors/slack/SlackConnector.java:28` — Connector SPI pattern
- `core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java:13` — type constants
- `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java:11` — ID constants
- `discord/pom.xml` — Maven module structure reference
