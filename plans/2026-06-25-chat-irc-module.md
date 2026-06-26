# Chat IRC Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `casehub-connectors-chat-irc` — an IRC ChatPlatform adapter with L2 inbound integration and an embedded test IRC server.

**Architecture:** Single `chat-irc` module with four production classes (`IrcClient`, `IrcChatPlatform`, `IrcInboundConnector`, `IrcInboundTranslator`), a protocol layer (`IrcMessage`, `IrcParser`, `IrcCommand`, `ChannelInfo`), and an embedded `EmbeddedIrcServer` for testing. IRC participates in L2 via `InboundConnector` → `InboundMessage` → `ChatInboundAdapter` → `IrcInboundTranslator` → `ReceivedMessage`. `IrcInboundConnector` owns the reconnect loop; `IrcClient` is pure transport.

**Tech Stack:** Java 21, Quarkus 3.32.2, `java.net.Socket` (TCP), CDI (`@ApplicationScoped`), JUnit 5, AssertJ

**Spec:** `specs/2026-06-25-chat-irc-module-design.md`

## Global Constraints

- Java 21 source, run on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All artifacts: `io.casehub` group, `0.2-SNAPSHOT` version
- Config properties namespaced: `casehub.connectors.chat-irc.*`
- SPI identifier methods: `id()` not `connectorId()` (protocol PP-20260609-e3a2bd)
- No business logic — pure delivery infrastructure
- Commits must reference issue: `Refs #25`

## File Map

### Production (chat-irc/src/main/java/io/casehub/connectors/chat/irc/)

| File | Responsibility |
|------|---------------|
| `protocol/IrcMessage.java` | Parsed IRC line record |
| `protocol/IrcParser.java` | Line↔IrcMessage conversion |
| `protocol/IrcCommand.java` | Command/numeric reply enum |
| `protocol/ChannelInfo.java` | LIST result record |
| `IrcClient.java` | TCP transport — socket, read loop, send/query operations |
| `IrcInboundConnector.java` | L2 InboundConnector — lifecycle, reconnect loop, InboundMessage delivery |
| `IrcInboundTranslator.java` | InboundMessage → ReceivedMessage translator |
| `IrcChatPlatform.java` | ChatPlatform SPI adapter — outbound operations |

### Test (chat-irc/src/test/java/io/casehub/connectors/chat/irc/)

| File | Responsibility |
|------|---------------|
| `test/EmbeddedIrcServer.java` | Minimal RFC 1459 server for testing |
| `protocol/IrcParserTest.java` | Parse/format unit tests |
| `IrcClientTest.java` | Integration tests against EmbeddedIrcServer |
| `IrcInboundTranslatorTest.java` | Unit test for InboundMessage → ReceivedMessage |
| `IrcChatPlatformTest.java` | CDI contract tests |
| `IrcInboundConnectorTest.java` | L2 integration + reconnect tests |

### Core additions

| File | Change |
|------|--------|
| `core/.../InboundConnectorIds.java` | Add `IRC = "irc-inbound"` constant |

### Module config

| File | Responsibility |
|------|---------------|
| `chat-irc/pom.xml` | Maven module |
| `pom.xml` (parent) | Add `<module>chat-irc</module>` |

---

### Task 1: Module scaffold + protocol layer (IrcMessage, IrcParser, IrcCommand, ChannelInfo)

The protocol layer is the foundation — pure string parsing with no dependencies on SPI or CDI. Every subsequent task builds on it.

**Files:**
- Create: `chat-irc/pom.xml`
- Modify: `pom.xml` (parent — add `<module>chat-irc</module>`)
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/IrcMessage.java`
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/IrcParser.java`
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/IrcCommand.java`
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/ChannelInfo.java`
- Create: `chat-irc/src/test/java/io/casehub/connectors/chat/irc/protocol/IrcParserTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `record IrcMessage(String prefix, String command, List<String> params)` — used by IrcClient, EmbeddedIrcServer, IrcInboundConnector
  - `IrcParser.parse(String line)` → `IrcMessage` — used by IrcClient, EmbeddedIrcServer
  - `IrcParser.format(String command, String... params)` → `String` — used by IrcClient, EmbeddedIrcServer
  - `enum IrcCommand` with `NICK, USER, JOIN, PART, PRIVMSG, PING, PONG, LIST, NAMES, QUIT` and numeric constants `RPL_WELCOME("001"), RPL_NAMREPLY("353"), RPL_ENDOFNAMES("366"), RPL_LIST("322"), RPL_LISTEND("323")`
  - `record ChannelInfo(String name, int memberCount, String topic)` — returned by `IrcClient.listChannels()`

- [ ] **Step 1: Create chat-irc/pom.xml**

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

  <artifactId>casehub-connectors-chat-irc</artifactId>
  <name>CaseHub Connectors — Chat IRC</name>
  <description>ChatPlatform implementation for IRC. Participates in L2 (InboundMessage)
and L7 (Chat SPI). Embedded test IRC server included.</description>

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

In `pom.xml` (project root), add `<module>chat-irc</module>` after `<module>chat-ref</module>` in the `<modules>` block.

- [ ] **Step 3: Write IrcParserTest — parse tests**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/protocol/IrcParserTest.java`:

```java
package io.casehub.connectors.chat.irc.protocol;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class IrcParserTest {

    @Test
    void parseServerMessageWithPrefixAndTrailing() {
        IrcMessage msg = IrcParser.parse(":nick!user@host PRIVMSG #channel :hello world");
        assertThat(msg.prefix()).isEqualTo("nick!user@host");
        assertThat(msg.command()).isEqualTo("PRIVMSG");
        assertThat(msg.params()).containsExactly("#channel", "hello world");
    }

    @Test
    void parseNumericReply() {
        IrcMessage msg = IrcParser.parse(":server 001 botname :Welcome to IRC");
        assertThat(msg.prefix()).isEqualTo("server");
        assertThat(msg.command()).isEqualTo("001");
        assertThat(msg.params()).containsExactly("botname", "Welcome to IRC");
    }

    @Test
    void parseWithNoPrefix() {
        IrcMessage msg = IrcParser.parse("PING :server.example.com");
        assertThat(msg.prefix()).isNull();
        assertThat(msg.command()).isEqualTo("PING");
        assertThat(msg.params()).containsExactly("server.example.com");
    }

    @Test
    void parseWithMultipleMiddleParams() {
        IrcMessage msg = IrcParser.parse(":server 353 bot = #channel :nick1 nick2 nick3");
        assertThat(msg.prefix()).isEqualTo("server");
        assertThat(msg.command()).isEqualTo("353");
        assertThat(msg.params()).containsExactly("bot", "=", "#channel", "nick1 nick2 nick3");
    }

    @Test
    void parseWithNoTrailing() {
        IrcMessage msg = IrcParser.parse(":nick!user@host JOIN #channel");
        assertThat(msg.prefix()).isEqualTo("nick!user@host");
        assertThat(msg.command()).isEqualTo("JOIN");
        assertThat(msg.params()).containsExactly("#channel");
    }

    @Test
    void parseWithEmptyTrailing() {
        IrcMessage msg = IrcParser.parse(":nick!user@host PRIVMSG #channel :");
        assertThat(msg.params()).containsExactly("#channel", "");
    }

    @Test
    void formatSimpleCommand() {
        String line = IrcParser.format("NICK", "botname");
        assertThat(line).isEqualTo("NICK botname");
    }

    @Test
    void formatWithTrailingContainingSpaces() {
        String line = IrcParser.format("PRIVMSG", "#channel", "hello world");
        assertThat(line).isEqualTo("PRIVMSG #channel :hello world");
    }

    @Test
    void formatWithSingleParamContainingSpaces() {
        String line = IrcParser.format("QUIT", "goodbye all");
        assertThat(line).isEqualTo("QUIT :goodbye all");
    }

    @Test
    void formatNoParams() {
        String line = IrcParser.format("QUIT");
        assertThat(line).isEqualTo("QUIT");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=IrcParserTest -f pom.xml`
Expected: compilation failure — `IrcParser` and `IrcMessage` do not exist yet.

- [ ] **Step 5: Implement IrcMessage**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/IrcMessage.java`:

```java
package io.casehub.connectors.chat.irc.protocol;

import java.util.List;

public record IrcMessage(String prefix, String command, List<String> params) {}
```

- [ ] **Step 6: Implement IrcParser**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/IrcParser.java`:

```java
package io.casehub.connectors.chat.irc.protocol;

import java.util.ArrayList;
import java.util.List;

public final class IrcParser {

    private IrcParser() {}

    public static IrcMessage parse(final String line) {
        String remaining = line;
        String prefix = null;

        if (remaining.startsWith(":")) {
            int space = remaining.indexOf(' ');
            prefix = remaining.substring(1, space);
            remaining = remaining.substring(space + 1);
        }

        String trailing = null;
        int trailingStart = remaining.indexOf(" :");
        if (trailingStart >= 0) {
            trailing = remaining.substring(trailingStart + 2);
            remaining = remaining.substring(0, trailingStart);
        }

        String[] parts = remaining.split(" ");
        String command = parts[0];
        List<String> params = new ArrayList<>();
        for (int i = 1; i < parts.length; i++) {
            params.add(parts[i]);
        }
        if (trailing != null) {
            params.add(trailing);
        }

        return new IrcMessage(prefix, command, List.copyOf(params));
    }

    public static String format(final String command, final String... params) {
        if (params.length == 0) {
            return command;
        }
        StringBuilder sb = new StringBuilder(command);
        for (int i = 0; i < params.length; i++) {
            sb.append(' ');
            if (i == params.length - 1 && params[i].contains(" ")) {
                sb.append(':');
            }
            sb.append(params[i]);
        }
        return sb.toString();
    }
}
```

- [ ] **Step 7: Implement IrcCommand**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/IrcCommand.java`:

```java
package io.casehub.connectors.chat.irc.protocol;

public enum IrcCommand {
    NICK, USER, JOIN, PART, PRIVMSG, PING, PONG, LIST, NAMES, QUIT;

    public static final String RPL_WELCOME = "001";
    public static final String RPL_NAMREPLY = "353";
    public static final String RPL_ENDOFNAMES = "366";
    public static final String RPL_LIST = "322";
    public static final String RPL_LISTEND = "323";
}
```

- [ ] **Step 8: Implement ChannelInfo**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/ChannelInfo.java`:

```java
package io.casehub.connectors.chat.irc.protocol;

public record ChannelInfo(String name, int memberCount, String topic) {}
```

- [ ] **Step 9: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=IrcParserTest -f pom.xml`
Expected: all 10 tests PASS.

- [ ] **Step 10: Add InboundConnectorIds.IRC to core**

In `core/src/main/java/io/casehub/connectors/InboundConnectorIds.java`, add after the `TEAMS_INBOUND` line:

```java
public static final String IRC = "irc-inbound";
```

- [ ] **Step 11: Build entire project to verify no breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add chat-irc/pom.xml pom.xml \
  chat-irc/src/main/java/io/casehub/connectors/chat/irc/protocol/ \
  chat-irc/src/test/java/io/casehub/connectors/chat/irc/protocol/ \
  core/src/main/java/io/casehub/connectors/InboundConnectorIds.java
git commit -m "feat(chat-irc): add module scaffold, protocol layer, and InboundConnectorIds.IRC — Refs #25"
```

---

### Task 2: EmbeddedIrcServer

The test server is needed by every subsequent integration test. It must handle NICK/USER registration, JOIN/PART/NAMES, LIST, PRIVMSG echo, PING/PONG, and QUIT. Multi-client via virtual threads.

**Files:**
- Create: `chat-irc/src/test/java/io/casehub/connectors/chat/irc/test/EmbeddedIrcServer.java`

**Interfaces:**
- Consumes: `IrcParser.parse()`, `IrcParser.format()`, `IrcCommand.*` (from Task 1)
- Produces:
  - `EmbeddedIrcServer(int port)` — `0` for random port
  - `void start()`, `void stop()`
  - `int getPort()`
  - `List<ReceivedPrivmsg> getReceivedMessages()` where `record ReceivedPrivmsg(String from, String channel, String text)`
  - `void sendToChannel(String channel, String nick, String message)` — injects a PRIVMSG as if from an external user

- [ ] **Step 1: Write a smoke test for EmbeddedIrcServer**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/test/EmbeddedIrcServerTest.java`:

```java
package io.casehub.connectors.chat.irc.test;

import static org.assertj.core.api.Assertions.assertThat;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class EmbeddedIrcServerTest {

    private EmbeddedIrcServer server;

    @BeforeEach
    void setUp() {
        server = new EmbeddedIrcServer(0);
        server.start();
    }

    @AfterEach
    void tearDown() {
        server.stop();
    }

    @Test
    void connectAndReceiveWelcome() throws Exception {
        try (Socket socket = new Socket("localhost", server.getPort());
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream()))) {
            out.println("NICK testbot");
            out.println("USER testbot 0 * :Test Bot");
            String welcome = in.readLine();
            assertThat(welcome).contains("001").contains("testbot");
        }
    }

    @Test
    void joinChannelReceivesNamesReply() throws Exception {
        try (Socket socket = new Socket("localhost", server.getPort());
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream()))) {
            out.println("NICK testbot");
            out.println("USER testbot 0 * :Test Bot");
            in.readLine(); // 001 welcome
            out.println("JOIN #test");
            String namReply = in.readLine();
            assertThat(namReply).contains("353");
            String endOfNames = in.readLine();
            assertThat(endOfNames).contains("366");
        }
    }

    @Test
    void privmsgIsRecorded() throws Exception {
        try (Socket socket = new Socket("localhost", server.getPort());
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream()))) {
            out.println("NICK testbot");
            out.println("USER testbot 0 * :Test Bot");
            in.readLine(); // 001
            out.println("JOIN #test");
            in.readLine(); // 353
            in.readLine(); // 366
            out.println("PRIVMSG #test :hello world");
            Thread.sleep(100);
            assertThat(server.getReceivedMessages()).hasSize(1);
            assertThat(server.getReceivedMessages().get(0).text()).isEqualTo("hello world");
        }
    }

    @Test
    void listReturnsJoinedChannels() throws Exception {
        try (Socket socket = new Socket("localhost", server.getPort());
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream()))) {
            out.println("NICK testbot");
            out.println("USER testbot 0 * :Test Bot");
            in.readLine(); // 001
            out.println("JOIN #alpha");
            in.readLine(); in.readLine(); // 353, 366
            out.println("JOIN #beta");
            in.readLine(); in.readLine(); // 353, 366
            out.println("LIST");
            String list1 = in.readLine();
            assertThat(list1).contains("322");
            String list2 = in.readLine();
            assertThat(list2).contains("322");
            String listEnd = in.readLine();
            assertThat(listEnd).contains("323");
        }
    }

    @Test
    void sendToChannelDeliversToClient() throws Exception {
        try (Socket socket = new Socket("localhost", server.getPort());
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream()))) {
            out.println("NICK testbot");
            out.println("USER testbot 0 * :Test Bot");
            in.readLine(); // 001
            out.println("JOIN #test");
            in.readLine(); in.readLine(); // 353, 366
            server.sendToChannel("#test", "externaluser", "injected message");
            String received = in.readLine();
            assertThat(received).contains("PRIVMSG").contains("#test").contains("injected message");
        }
    }

    @Test
    void pingReceivesPong() throws Exception {
        try (Socket socket = new Socket("localhost", server.getPort());
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream()))) {
            out.println("NICK testbot");
            out.println("USER testbot 0 * :Test Bot");
            in.readLine(); // 001
            out.println("PING :test123");
            String pong = in.readLine();
            assertThat(pong).isEqualTo("PONG :test123");
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=EmbeddedIrcServerTest -f pom.xml`
Expected: compilation failure — `EmbeddedIrcServer` does not exist.

- [ ] **Step 3: Implement EmbeddedIrcServer**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/test/EmbeddedIrcServer.java`.

This is the largest single file in the module. Key implementation notes:
- `ServerSocket` on the given port (0 for random)
- `start()` spawns an acceptor thread; each client connection gets a virtual thread
- Per-client state: `nick`, `Set<String> joinedChannels`
- Server state: `Map<String, Set<ClientHandler>> channelMembers`, `Map<String, String> channelTopics`
- NICK/USER → store nick, send `001 nick :Welcome`
- JOIN → add to channel members, send `353` (NAMES reply with all nicks in channel) + `366` (end of names)
- PART → remove from channel members
- PRIVMSG → record in `receivedMessages` list, echo to other clients in that channel (not back to sender)
- NAMES → send `353` + `366` for the requested channel
- LIST → send `322` per channel (name, member count, topic) + `323` (list end)
- PING → respond with `PONG :<param>`
- QUIT → close client connection
- `sendToChannel()` → format a PRIVMSG from the given nick and write it to all clients in that channel
- `getReceivedMessages()` → return `List.copyOf(receivedMessages)`
- `record ReceivedPrivmsg(String from, String channel, String text)` — inner record
- `stop()` → close server socket, interrupt all client threads

Use `IrcParser.parse()` and `IrcParser.format()` from the protocol package for all line processing. Synchronize on channel membership maps for thread safety.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=EmbeddedIrcServerTest -f pom.xml`
Expected: all 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add chat-irc/src/test/java/io/casehub/connectors/chat/irc/test/
git commit -m "test(chat-irc): add EmbeddedIrcServer with NICK/USER/JOIN/PART/PRIVMSG/LIST/NAMES/PING/QUIT — Refs #25"
```

---

### Task 3: IrcClient — TCP transport

The raw TCP client. Connects, sends commands, reads responses, maintains the read loop. Pure transport — no reconnect, no backoff. Tests run against EmbeddedIrcServer.

**Files:**
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcClient.java`
- Create: `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcClientTest.java`

**Interfaces:**
- Consumes: `IrcParser`, `IrcMessage`, `IrcCommand`, `ChannelInfo` (from Task 1), `EmbeddedIrcServer` (from Task 2)
- Produces:
  - `boolean connect()` — opens socket, NICK/USER handshake, starts read thread
  - `void disconnect()` — QUIT, close socket
  - `boolean isConnected()`
  - `boolean join(String channel)` — JOIN + wait for NAMES
  - `void part(String channel)` — PART
  - `boolean send(String channel, String message)` — PRIVMSG
  - `List<ChannelInfo> listChannels()` — LIST + collect
  - `List<String> names(String channel)` — NAMES + collect
  - `void setMessageCallback(Consumer<IrcMessage> callback)` — for PRIVMSG delivery

- [ ] **Step 1: Write IrcClientTest**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcClientTest.java`:

```java
package io.casehub.connectors.chat.irc;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.irc.protocol.ChannelInfo;
import io.casehub.connectors.chat.irc.protocol.IrcMessage;
import io.casehub.connectors.chat.irc.test.EmbeddedIrcServer;

class IrcClientTest {

    private EmbeddedIrcServer server;
    private IrcClient client;

    @BeforeEach
    void setUp() {
        server = new EmbeddedIrcServer(0);
        server.start();
        client = new IrcClient("localhost", server.getPort(), "testbot");
    }

    @AfterEach
    void tearDown() {
        client.disconnect();
        server.stop();
    }

    @Test
    void connectAndDisconnect() {
        assertThat(client.connect()).isTrue();
        assertThat(client.isConnected()).isTrue();
        client.disconnect();
        assertThat(client.isConnected()).isFalse();
    }

    @Test
    void joinChannel() {
        client.connect();
        assertThat(client.join("#test")).isTrue();
    }

    @Test
    void sendMessage() {
        client.connect();
        client.join("#test");
        assertThat(client.send("#test", "hello")).isTrue();
        assertThat(server.getReceivedMessages()).hasSize(1);
        assertThat(server.getReceivedMessages().get(0).text()).isEqualTo("hello");
    }

    @Test
    void listChannels() {
        client.connect();
        client.join("#alpha");
        client.join("#beta");
        List<ChannelInfo> channels = client.listChannels();
        assertThat(channels).extracting(ChannelInfo::name)
                .containsExactlyInAnyOrder("#alpha", "#beta");
    }

    @Test
    void namesReturnsChannelMembers() {
        client.connect();
        client.join("#test");
        List<String> nicks = client.names("#test");
        assertThat(nicks).contains("testbot");
    }

    @Test
    void receiveCallback() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);
        List<IrcMessage> received = new CopyOnWriteArrayList<>();
        client.setMessageCallback(msg -> {
            received.add(msg);
            latch.countDown();
        });
        client.connect();
        client.join("#test");
        server.sendToChannel("#test", "other", "hello from other");
        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(1);
        assertThat(received.get(0).params().get(1)).isEqualTo("hello from other");
    }

    @Test
    void sendReturnsFalseWhenNotConnected() {
        assertThat(client.send("#test", "hello")).isFalse();
    }

    @Test
    void listChannelsReturnsEmptyWhenNotConnected() {
        assertThat(client.listChannels()).isEmpty();
    }

    @Test
    void joinReturnsFalseWhenNotConnected() {
        assertThat(client.join("#test")).isFalse();
    }

    @Test
    void readLoopExitsOnServerDisconnect() throws Exception {
        client.connect();
        assertThat(client.isConnected()).isTrue();
        server.stop();
        Thread.sleep(500);
        assertThat(client.isConnected()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=IrcClientTest -f pom.xml`
Expected: compilation failure — `IrcClient` does not exist.

- [ ] **Step 3: Implement IrcClient**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcClient.java`.

Key implementation notes:
- Constructor: `IrcClient(String host, int port, String nick)` — plain Java, NOT CDI-injected in this constructor (CDI wiring comes in Task 4). For now, this is a plain class for testability.
- `connect()`: open `Socket`, create `BufferedReader`/`PrintWriter`, send `NICK nick` + `USER nick 0 * :nick`, wait for `001` reply (10s timeout via CompletableFuture), start read thread, return true/false
- `disconnect()`: if connected, send `QUIT`, close socket, interrupt read thread, set `connected = false`
- `isConnected()`: return `volatile boolean connected`
- `join(channel)`: if not connected return false. Register a collector for `366` (end of names) keyed by channel. Send `JOIN channel`. Wait on the CompletableFuture (10s timeout). Return true if completed, false on timeout.
- `part(channel)`: if connected, send `PART channel`
- `send(channel, message)`: if not connected return false. Send `PRIVMSG channel :message`. Return true.
- `listChannels()`: if not connected return empty list. Register a `List<ChannelInfo>` collector for `323` (list end). Send `LIST`. Read loop feeds `322` replies into the collector. Wait on CompletableFuture. Return the list.
- `names(channel)`: if not connected return empty list. Register a `List<String>` collector for `366` keyed by channel. Send `NAMES channel`. Wait. Return.
- `setMessageCallback(Consumer<IrcMessage>)`: store the consumer
- Read thread: loop reading lines from BufferedReader, parse each with `IrcParser.parse()`. Route by command:
  - `PING` → write `PONG :` + first param
  - `PRIVMSG` → call the message callback if set
  - `001`, `322`, `323`, `353`, `366` → feed into collectors
  - On `IOException` → set `connected = false`, exit loop
- Write lock: `ReentrantLock` around all `PrintWriter.println()` calls
- Collector: `ConcurrentHashMap<String, CompletableFuture<...>>` — keyed by a combination of command and channel. The specific collector design: for LIST, one pending future that accumulates `322` replies until `323` completes it. For NAMES/JOIN, keyed by channel, accumulates `353` until `366`. These are short-lived — registered before send, consumed by reply, removed after completion or timeout.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=IrcClientTest -f pom.xml`
Expected: all 10 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcClient.java \
  chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcClientTest.java
git commit -m "feat(chat-irc): add IrcClient TCP transport with connect/send/join/list/names — Refs #25"
```

---

### Task 4: CDI beans — IrcInboundTranslator, IrcInboundConnector, IrcChatPlatform

The three CDI beans that integrate IrcClient with the platform. IrcClient gets `@ApplicationScoped` + `@ConfigProperty` injection. IrcInboundConnector owns the reconnect loop. IrcChatPlatform maps IrcClient to the ChatPlatform SPI. IrcInboundTranslator bridges L2→L7.

**Files:**
- Modify: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcClient.java` — add `@ApplicationScoped`, `@ConfigProperty` injection, keep the `(String, int, String)` constructor for tests
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcInboundConnector.java`
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcInboundTranslator.java`
- Create: `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcChatPlatform.java`
- Create: `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcInboundTranslatorTest.java`
- Create: `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcChatPlatformTest.java`
- Create: `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcInboundConnectorTest.java`
- Create: `chat-irc/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `IrcClient` (Task 3), `EmbeddedIrcServer` (Task 2), `ChatPlatform`, `InboundConnector`, `InboundTranslator`, `InboundMessage`, `ReceivedMessage`, `ChatInboundAdapter`, all Chat SPI model types and degraded implementations
- Produces:
  - `IrcInboundTranslator implements InboundTranslator` — `connectorType()` → `"irc"`, `translate(InboundMessage)` → `ReceivedMessage`
  - `IrcInboundConnector implements InboundConnector` — `id()` → `"irc-inbound"`, `start(sink)/stop()`
  - `IrcChatPlatform implements ChatPlatform` — `id()` → `"irc"`, all six capability methods

- [ ] **Step 1: Write IrcInboundTranslatorTest**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcInboundTranslatorTest.java`:

```java
package io.casehub.connectors.chat.irc;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ReceivedMessage;

class IrcInboundTranslatorTest {

    private final IrcInboundTranslator translator = new IrcInboundTranslator();

    @Test
    void connectorTypeIsIrc() {
        assertThat(translator.connectorType()).isEqualTo(InboundConnectorTypes.IRC);
    }

    @Test
    void translateMapsAllFields() {
        Instant now = Instant.now();
        InboundMessage msg = new InboundMessage(
                InboundConnectorIds.IRC,
                InboundConnectorTypes.IRC,
                "alice",
                "#general",
                "hello world",
                List.of(),
                now,
                Map.of("nick-prefix", "alice!user@host"),
                null);

        ReceivedMessage result = translator.translate(msg);

        assertThat(result.platformId()).isEqualTo("irc");
        assertThat(result.channel().id()).isEqualTo("#general");
        assertThat(result.messageRef().channel().id()).isEqualTo("#general");
        assertThat(result.messageRef().messageId()).isNotBlank();
        assertThat(result.parentRef()).isNull();
        assertThat(result.sender().id()).isEqualTo("alice");
        assertThat(result.content().text()).isEqualTo("hello world");
        assertThat(result.content().attachments()).isEmpty();
        assertThat(result.receivedAt()).isEqualTo(now);
    }

    @Test
    void syntheticMessageIdIsUniquePerCall() {
        InboundMessage msg = new InboundMessage(
                InboundConnectorIds.IRC, InboundConnectorTypes.IRC,
                "alice", "#general", "text", List.of(),
                Instant.now(), Map.of(), null);

        String id1 = translator.translate(msg).messageRef().messageId();
        String id2 = translator.translate(msg).messageRef().messageId();
        assertThat(id1).isNotEqualTo(id2);
    }
}
```

- [ ] **Step 2: Implement IrcInboundTranslator**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcInboundTranslator.java`:

```java
package io.casehub.connectors.chat.irc;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

@ApplicationScoped
public class IrcInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return InboundConnectorTypes.IRC;
    }

    @Override
    public ReceivedMessage translate(final InboundMessage msg) {
        ChatChannelRef channel = new ChatChannelRef(msg.externalChannelRef());
        ChatMessageRef messageRef = new ChatMessageRef(channel,
                UUID.randomUUID().toString());
        return new ReceivedMessage(
                InboundConnectorTypes.IRC,
                channel,
                messageRef,
                null,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content(), null, msg.attachments()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 3: Run IrcInboundTranslatorTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -Dtest=IrcInboundTranslatorTest -f pom.xml`
Expected: all 3 tests PASS.

- [ ] **Step 4: Add CDI wiring to IrcClient**

Modify `IrcClient.java`: add `@ApplicationScoped`, add a CDI constructor with `@ConfigProperty` injection for `casehub.connectors.chat-irc.host`, `casehub.connectors.chat-irc.port` (default `6667`), `casehub.connectors.chat-irc.nick`. Keep the existing `(String, int, String)` constructor for test use (package-private or annotated appropriately for ArC to ignore).

- [ ] **Step 5: Implement IrcInboundConnector**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcInboundConnector.java`:

```java
package io.casehub.connectors.chat.irc;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.logging.Level;
import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.casehub.connectors.InboundConnector;
import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.InboundMessageSink;
import io.casehub.connectors.chat.irc.protocol.IrcMessage;

@ApplicationScoped
public class IrcInboundConnector implements InboundConnector {

    private static final Logger LOG = Logger.getLogger(
            IrcInboundConnector.class.getName());

    private final IrcClient client;
    private final Optional<List<String>> channels;
    private volatile boolean stopping = false;
    private ExecutorService executor;

    @Inject
    public IrcInboundConnector(
            final IrcClient client,
            @ConfigProperty(name = "casehub.connectors.chat-irc.channels")
            final Optional<List<String>> channels) {
        this.client = client;
        this.channels = channels;
    }

    @Override
    public String id() {
        return InboundConnectorIds.IRC;
    }

    @Override
    public void start(final InboundMessageSink sink) {
        if (executor != null) return;
        executor = Executors.newVirtualThreadPerTaskExecutor();
        executor.submit(() -> connectLoop(sink));
    }

    @Override
    public void stop() {
        stopping = true;
        client.disconnect();
        if (executor != null) {
            executor.shutdownNow();
        }
    }

    private void connectLoop(final InboundMessageSink sink) {
        int backoffSeconds = 1;
        int consecutiveFailures = 0;

        while (!stopping) {
            try {
                client.setMessageCallback(msg -> deliverToSink(msg, sink));
                if (!client.connect()) {
                    throw new java.io.IOException("IRC connect failed");
                }
                backoffSeconds = 1;
                consecutiveFailures = 0;
                LOG.info("irc-inbound: connected");
                joinConfiguredChannels();
                awaitDisconnect();
            } catch (final Exception e) {
                if (!stopping) {
                    consecutiveFailures++;
                    Level level = consecutiveFailures >= 5
                            ? Level.SEVERE : Level.WARNING;
                    LOG.log(level, "irc-inbound: connection failed (attempt "
                            + consecutiveFailures + "): " + e.getMessage());
                    sleepQuietly(backoffSeconds * 1000L);
                    backoffSeconds = Math.min(backoffSeconds * 2, 60);
                }
            }
        }
    }

    private void joinConfiguredChannels() {
        channels.ifPresent(list -> list.forEach(ch -> {
            if (!client.join(ch)) {
                LOG.warning("irc-inbound: failed to join " + ch);
            }
        }));
    }

    private void awaitDisconnect() {
        while (client.isConnected() && !stopping) {
            sleepQuietly(1000);
        }
    }

    private void deliverToSink(final IrcMessage msg,
                                final InboundMessageSink sink) {
        String nick = extractNick(msg.prefix());
        String channel = msg.params().get(0);
        String text = msg.params().get(1);
        try {
            sink.receive(new InboundMessage(
                    InboundConnectorIds.IRC,
                    InboundConnectorTypes.IRC,
                    nick,
                    channel,
                    text,
                    List.of(),
                    Instant.now(),
                    Map.of("nick-prefix",
                            msg.prefix() != null ? msg.prefix() : ""),
                    null));
        } catch (final Exception e) {
            LOG.log(Level.SEVERE, "irc-inbound: sink threw", e);
        }
    }

    static String extractNick(final String prefix) {
        if (prefix == null) return "";
        int bang = prefix.indexOf('!');
        return bang >= 0 ? prefix.substring(0, bang) : prefix;
    }

    private static void sleepQuietly(final long millis) {
        try { Thread.sleep(millis); }
        catch (final InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

- [ ] **Step 6: Implement IrcChatPlatform**

Create `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcChatPlatform.java`:

```java
package io.casehub.connectors.chat.irc;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.connectors.chat.degraded.ChannelFallbackThreading;
import io.casehub.connectors.chat.degraded.NoOpReactions;
import io.casehub.connectors.chat.degraded.UnknownPresence;
import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.Member;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;

@ApplicationScoped
public class IrcChatPlatform implements ChatPlatform {

    private static final Set<Class<?>> NATIVE_CAPABILITIES = Set.of(
            Messaging.class, Discovery.class, Members.class);

    private final IrcClient client;
    private final Messaging messaging;
    private final Threading threading;
    private final Discovery discovery;
    private final Members members;

    @Inject
    public IrcChatPlatform(final IrcClient client) {
        this.client = client;
        this.messaging = (channel, content) -> {
            if (!client.send(channel.id(), content.text())) {
                return SendResult.failure("not connected");
            }
            return SendResult.success(
                    new ChatMessageRef(channel, UUID.randomUUID().toString()),
                    Instant.now());
        };
        this.threading = new ChannelFallbackThreading(messaging);
        this.discovery = () -> client.listChannels().stream()
                .map(ci -> new Channel(
                        new ChatChannelRef(ci.name()),
                        ci.name(),
                        ci.topic(),
                        false))
                .toList();
        this.members = channel -> client.names(channel.id()).stream()
                .map(nick -> new Member(new MemberRef(nick), nick))
                .toList();
    }

    @Override public String id() { return "irc"; }
    @Override public Messaging messaging() { return messaging; }
    @Override public Threading threading() { return threading; }
    @Override public Discovery discovery() { return discovery; }
    @Override public Reactions reactions() { return new NoOpReactions(); }
    @Override public Presence presence() { return new UnknownPresence(); }
    @Override public Members members() { return members; }

    @Override
    public boolean supports(final Class<?> capability) {
        return NATIVE_CAPABILITIES.contains(capability);
    }
}
```

- [ ] **Step 7: Create test application.properties**

Create `chat-irc/src/test/resources/application.properties`:

```properties
# Overridden per-test via @TestProfile or QuarkusTestResource
casehub.connectors.chat-irc.host=localhost
casehub.connectors.chat-irc.port=0
casehub.connectors.chat-irc.nick=ircbot
```

- [ ] **Step 8: Write IrcChatPlatformTest (contract test)**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcChatPlatformTest.java`.

This test mirrors `RefChatPlatformContractTest`. It uses `EmbeddedIrcServer` started in a `@BeforeAll`, with the port injected via config overrides. The test creates an `IrcClient` directly (not CDI) and an `IrcChatPlatform` wrapping it, to test the ChatPlatform contract without the full CDI lifecycle.

Tests:
- `idIsIrc()` — `assertThat(platform.id()).isEqualTo("irc")`
- `supportsMessagingDiscoveryMembers()` — true for Messaging, Discovery, Members; false for Threading, Reactions, Presence
- `sendToChannelReturnsSuccess()` — send a message, verify `SendResult.ok()`, `messageRef()` is not null, `messageRef().messageId()` is not blank
- `sendWhenDisconnectedReturnsFailure()` — disconnect client, send, verify `!SendResult.ok()`, error contains "not connected"
- `listChannelsReturnsJoinedChannels()` — join two channels, `discovery().listChannels()` returns both, `isPrivate` is false for all
- `listMembersReturnsNicks()` — join a channel, `members().list(channel)` includes the bot nick
- `threadingDegradesToChannelSend()` — send a reply via `threading().reply()`, verify it succeeds (sent to channel)
- `reactionsAreNoOp()` — add/remove reaction doesn't throw
- `presenceReturnsUnknown()` — `presence().of(member)` returns `UNKNOWN`

- [ ] **Step 9: Write IrcInboundConnectorTest (L2 integration)**

Create `chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcInboundConnectorTest.java`.

This test verifies the L2 path without full CDI. It creates `EmbeddedIrcServer`, `IrcClient`, `IrcInboundConnector` directly, and passes a recording `InboundMessageSink`.

Tests:
- `inboundPrivmsgDeliveredToSink()` — start connector, server sends a PRIVMSG via `sendToChannel()`, verify sink received an `InboundMessage` with correct connectorId, connectorType, externalSenderId, externalChannelRef, content
- `reconnectsAfterServerDrop()` — start connector, verify connected, stop server, start a new server on same port, verify connector reconnects and re-joins channels, send another message, verify it's delivered
- `stopPreventsReconnect()` — start connector, stop it, verify no reconnect attempts

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-irc -f pom.xml`
Expected: all tests PASS.

- [ ] **Step 11: Full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcClient.java \
  chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcInboundConnector.java \
  chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcInboundTranslator.java \
  chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcChatPlatform.java \
  chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcInboundTranslatorTest.java \
  chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcChatPlatformTest.java \
  chat-irc/src/test/java/io/casehub/connectors/chat/irc/IrcInboundConnectorTest.java \
  chat-irc/src/test/resources/application.properties
git commit -m "feat(chat-irc): add IrcChatPlatform, IrcInboundConnector, IrcInboundTranslator with CDI wiring — Refs #25"
```
