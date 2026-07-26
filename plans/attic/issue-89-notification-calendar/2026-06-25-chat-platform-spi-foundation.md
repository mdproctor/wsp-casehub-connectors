# Chat Platform SPI Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the chat-spi module (model types, capability interfaces, builder, degradation types, ChatPlatformService, ChatInboundAdapter) and chat-ref module (in-memory reference implementation with full Messaging + Threading + Discovery + Reactions + Presence + Members), with SPI contract tests that validate the entire model.

**Architecture:** Composed capabilities with auto-degrading builder (Approach D from spec). `ChatPlatform` is a composition of focused capability interfaces. The builder auto-provides degradation defaults for any capability the platform doesn't natively support. `ChatPlatformService` mirrors `ConnectorService` — CDI-discovered via `@All List<ChatPlatform>`, indexed by `id()`. Inbound uses existing `InboundConnector` SPI; `ChatInboundAdapter` translates `InboundMessage` → `ReceivedMessage` via per-platform `InboundTranslator` beans.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`quarkus-arc`), JUnit5 + AssertJ

**Spec:** `specs/2026-06-25-chat-platform-spi-design.md`

## Global Constraints

- Java 21 (on Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Quarkus 3.32.2 — all casehubio repos aligned
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw` — maven wrapper not configured
- Parent POM: `io.casehub:casehub-connectors-parent:0.2-SNAPSHOT`
- `core` module: no casehubio runtime dependencies
- chat-spi depends on: `casehub-connectors-core`
- chat-ref depends on: `chat-spi`
- Fail-soft: `send()` and `reply()` must not throw — return `SendResult.failure()`
- Reactions/Presence/Members must not throw — fire-and-forget / sentinel values
- `id()` pattern: lowercase, URL-safe, `[a-z0-9][a-z0-9\-]*`
- Jandex plugin required in each module for CDI bean discovery
- Use `HttpHelper.CLIENT` singleton (from core) for any HTTP calls — never create new `HttpClient()`
- InboundConnectorTypes constants: add `DISCORD = "discord"` and `IRC = "irc"` to core

---

### Task 1: Create chat-spi Module — Model Types

**Files:**
- Create: `chat-spi/pom.xml`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/ChatChannelRef.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/ChatMessageRef.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/MemberRef.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/ChatContent.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/SendResult.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/Channel.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/Member.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/PresenceStatus.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/ReceivedMessage.java`
- Modify: `pom.xml` (parent) — add `<module>chat-spi</module>`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/model/ChatContentTest.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/model/SendResultTest.java`

**Interfaces:**
- Consumes: `io.casehub.connectors.Attachment` from core
- Produces: All model types used by every subsequent task

- [ ] **Step 1: Create `chat-spi/pom.xml`**

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

  <artifactId>casehub-connectors-chat-spi</artifactId>
  <name>CaseHub Connectors — Chat SPI</name>
  <description>Chat Platform SPI: capability interfaces, model types, degradation defaults,
and ChatInboundAdapter for translating InboundMessage to ReceivedMessage.</description>

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

- [ ] **Step 2: Add `chat-spi` module to parent POM**

In `pom.xml` (root), add `<module>chat-spi</module>` to the `<modules>` section, after `slack-bot`.

- [ ] **Step 3: Create model record types**

Create the following records. Each is a single file in `chat-spi/src/main/java/io/casehub/connectors/chat/model/`:

`ChatChannelRef.java`:
```java
package io.casehub.connectors.chat.model;

public record ChatChannelRef(String id) {}
```

`ChatMessageRef.java`:
```java
package io.casehub.connectors.chat.model;

public record ChatMessageRef(ChatChannelRef channel, String messageId) {}
```

`MemberRef.java`:
```java
package io.casehub.connectors.chat.model;

public record MemberRef(String id) {}
```

`ChatContent.java`:
```java
package io.casehub.connectors.chat.model;

import java.util.List;
import java.util.Objects;

import io.casehub.connectors.Attachment;

public record ChatContent(
        String text,
        String markdown,
        List<Attachment> attachments) {

    public ChatContent {
        Objects.requireNonNull(text, "text");
        attachments = attachments == null ? List.of() : List.copyOf(attachments);
    }

    public ChatContent(final String text) {
        this(text, null, List.of());
    }
}
```

`SendResult.java`:
```java
package io.casehub.connectors.chat.model;

import java.time.Instant;
import java.util.Objects;

public record SendResult(boolean ok, ChatMessageRef messageRef, Instant timestamp, String error) {

    public static SendResult success(final ChatMessageRef ref, final Instant ts) {
        Objects.requireNonNull(ref, "messageRef");
        return new SendResult(true, ref, ts, null);
    }

    public static SendResult failure(final String error) {
        return new SendResult(false, null, null, error);
    }
}
```

`Channel.java`:
```java
package io.casehub.connectors.chat.model;

public record Channel(ChatChannelRef ref, String name, String topic, boolean isPrivate) {}
```

`Member.java`:
```java
package io.casehub.connectors.chat.model;

public record Member(MemberRef ref, String displayName) {}
```

`PresenceStatus.java`:
```java
package io.casehub.connectors.chat.model;

public enum PresenceStatus { ONLINE, OFFLINE, AWAY, DND, UNKNOWN }
```

`ReceivedMessage.java`:
```java
package io.casehub.connectors.chat.model;

import java.time.Instant;

public record ReceivedMessage(
        String platformId,
        ChatChannelRef channel,
        ChatMessageRef messageRef,
        ChatMessageRef parentRef,
        MemberRef sender,
        ChatContent content,
        Instant receivedAt) {}
```

- [ ] **Step 4: Write tests for ChatContent defensive constructor**

`chat-spi/src/test/java/io/casehub/connectors/chat/model/ChatContentTest.java`:
```java
package io.casehub.connectors.chat.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.Attachment;

class ChatContentTest {

    @Test
    void textIsRequired() {
        assertThatThrownBy(() -> new ChatContent(null))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("text");
    }

    @Test
    void convenienceConstructorSetsDefaults() {
        ChatContent content = new ChatContent("hello");
        assertThat(content.text()).isEqualTo("hello");
        assertThat(content.markdown()).isNull();
        assertThat(content.attachments()).isEmpty();
    }

    @Test
    void nullAttachmentsBecomesEmptyList() {
        ChatContent content = new ChatContent("hello", null, null);
        assertThat(content.attachments()).isEmpty();
    }

    @Test
    void attachmentsAreDefensivelyCopied() {
        List<Attachment> mutable = new ArrayList<>();
        mutable.add(new Attachment("f.txt", "text/plain", new byte[]{1}));
        ChatContent content = new ChatContent("hello", null, mutable);
        mutable.add(new Attachment("g.txt", "text/plain", new byte[]{2}));
        assertThat(content.attachments()).hasSize(1);
    }
}
```

- [ ] **Step 5: Write tests for SendResult factory methods**

`chat-spi/src/test/java/io/casehub/connectors/chat/model/SendResultTest.java`:
```java
package io.casehub.connectors.chat.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;

import org.junit.jupiter.api.Test;

class SendResultTest {

    @Test
    void successRequiresNonNullRef() {
        assertThatThrownBy(() -> SendResult.success(null, Instant.now()))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("messageRef");
    }

    @Test
    void successCarriesRefAndTimestamp() {
        ChatChannelRef ch = new ChatChannelRef("C1");
        ChatMessageRef ref = new ChatMessageRef(ch, "ts1");
        Instant now = Instant.now();
        SendResult result = SendResult.success(ref, now);
        assertThat(result.ok()).isTrue();
        assertThat(result.messageRef()).isEqualTo(ref);
        assertThat(result.timestamp()).isEqualTo(now);
        assertThat(result.error()).isNull();
    }

    @Test
    void failureCarriesErrorMessage() {
        SendResult result = SendResult.failure("timeout");
        assertThat(result.ok()).isFalse();
        assertThat(result.messageRef()).isNull();
        assertThat(result.timestamp()).isNull();
        assertThat(result.error()).isEqualTo("timeout");
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```
feat(chat-spi): add model types — ChatChannelRef, ChatMessageRef, ChatContent, SendResult, ReceivedMessage

Refs #<issue>
```

---

### Task 2: Create chat-spi Module — Capability Interfaces + Degradation Types

**Files:**
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Messaging.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Threading.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Discovery.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Reactions.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Presence.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Members.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/ChannelFallbackThreading.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/NoOpReactions.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/UnknownPresence.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/EmptyMembers.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/EmptyDiscovery.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/degraded/ChannelFallbackThreadingTest.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/degraded/DegradationTypesTest.java`

**Interfaces:**
- Consumes: Model types from Task 1
- Produces: All capability interfaces and degradation types used by ChatPlatform (Task 3) and all platform implementations

- [ ] **Step 1: Create capability interfaces**

Each is a single file in `chat-spi/src/main/java/io/casehub/connectors/chat/spi/`:

`Messaging.java`:
```java
package io.casehub.connectors.chat.spi;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.SendResult;

public interface Messaging {
    SendResult send(ChatChannelRef channel, ChatContent content);
}
```

`Threading.java`:
```java
package io.casehub.connectors.chat.spi;

import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.SendResult;

public interface Threading {
    SendResult reply(ChatMessageRef parent, ChatContent content);
}
```

`Discovery.java`:
```java
package io.casehub.connectors.chat.spi;

import java.util.List;

import io.casehub.connectors.chat.model.Channel;

public interface Discovery {
    List<Channel> listChannels();
}
```

`Reactions.java`:
```java
package io.casehub.connectors.chat.spi;

import io.casehub.connectors.chat.model.ChatMessageRef;

public interface Reactions {
    void add(ChatMessageRef message, String emoji);
    void remove(ChatMessageRef message, String emoji);
}
```

`Presence.java`:
```java
package io.casehub.connectors.chat.spi;

import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;

public interface Presence {
    PresenceStatus of(MemberRef member);
}
```

`Members.java`:
```java
package io.casehub.connectors.chat.spi;

import java.util.List;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.Member;

public interface Members {
    List<Member> list(ChatChannelRef channel);
}
```

- [ ] **Step 2: Create degradation types**

Each in `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/`:

`ChannelFallbackThreading.java`:
```java
package io.casehub.connectors.chat.degraded;

import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Threading;

public class ChannelFallbackThreading implements Threading {

    private final Messaging messaging;

    public ChannelFallbackThreading(final Messaging messaging) {
        this.messaging = messaging;
    }

    @Override
    public SendResult reply(final ChatMessageRef parent, final ChatContent content) {
        return messaging.send(parent.channel(), content);
    }
}
```

`NoOpReactions.java`:
```java
package io.casehub.connectors.chat.degraded;

import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.spi.Reactions;

public class NoOpReactions implements Reactions {
    @Override public void add(final ChatMessageRef message, final String emoji) {}
    @Override public void remove(final ChatMessageRef message, final String emoji) {}
}
```

`UnknownPresence.java`:
```java
package io.casehub.connectors.chat.degraded;

import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;
import io.casehub.connectors.chat.spi.Presence;

public class UnknownPresence implements Presence {
    @Override
    public PresenceStatus of(final MemberRef member) {
        return PresenceStatus.UNKNOWN;
    }
}
```

`EmptyMembers.java`:
```java
package io.casehub.connectors.chat.degraded;

import java.util.List;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.Member;
import io.casehub.connectors.chat.spi.Members;

public class EmptyMembers implements Members {
    @Override
    public List<Member> list(final ChatChannelRef channel) {
        return List.of();
    }
}
```

`EmptyDiscovery.java`:
```java
package io.casehub.connectors.chat.degraded;

import java.util.List;

import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.spi.Discovery;

public class EmptyDiscovery implements Discovery {
    @Override
    public List<Channel> listChannels() {
        return List.of();
    }
}
```

- [ ] **Step 3: Write ChannelFallbackThreading test**

`chat-spi/src/test/java/io/casehub/connectors/chat/degraded/ChannelFallbackThreadingTest.java`:
```java
package io.casehub.connectors.chat.degraded;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.Messaging;

class ChannelFallbackThreadingTest {

    @Test
    void replyDelegatesToMessagingSendWithParentChannel() {
        ChatChannelRef channel = new ChatChannelRef("general");
        ChatMessageRef parent = new ChatMessageRef(channel, "msg-1");
        ChatContent content = new ChatContent("reply text");
        SendResult expected = SendResult.success(
                new ChatMessageRef(channel, "msg-2"), Instant.now());

        Messaging mockMessaging = (ch, c) -> {
            assertThat(ch).isEqualTo(channel);
            assertThat(c).isEqualTo(content);
            return expected;
        };

        ChannelFallbackThreading threading = new ChannelFallbackThreading(mockMessaging);
        SendResult result = threading.reply(parent, content);

        assertThat(result).isEqualTo(expected);
    }
}
```

- [ ] **Step 4: Write degradation sentinel tests**

`chat-spi/src/test/java/io/casehub/connectors/chat/degraded/DegradationTypesTest.java`:
```java
package io.casehub.connectors.chat.degraded;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;

class DegradationTypesTest {

    @Test
    void noOpReactionsDoNotThrow() {
        NoOpReactions reactions = new NoOpReactions();
        ChatMessageRef ref = new ChatMessageRef(new ChatChannelRef("ch"), "msg");
        assertThatCode(() -> reactions.add(ref, "thumbsup")).doesNotThrowAnyException();
        assertThatCode(() -> reactions.remove(ref, "thumbsup")).doesNotThrowAnyException();
    }

    @Test
    void unknownPresenceAlwaysReturnsUnknown() {
        UnknownPresence presence = new UnknownPresence();
        assertThat(presence.of(new MemberRef("user1"))).isEqualTo(PresenceStatus.UNKNOWN);
    }

    @Test
    void emptyMembersReturnsEmptyList() {
        EmptyMembers members = new EmptyMembers();
        assertThat(members.list(new ChatChannelRef("ch"))).isEmpty();
    }

    @Test
    void emptyDiscoveryReturnsEmptyList() {
        EmptyDiscovery discovery = new EmptyDiscovery();
        assertThat(discovery.listChannels()).isEmpty();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
feat(chat-spi): add capability interfaces and degradation types

Six capability interfaces: Messaging, Threading, Discovery, Reactions, Presence, Members.
Five degradation defaults: ChannelFallbackThreading, NoOpReactions, UnknownPresence,
EmptyMembers, EmptyDiscovery. — Refs #<issue>
```

---

### Task 3: Create ChatPlatform Interface, Builder, and ChatPlatformService

**Files:**
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/DefaultChatPlatform.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/ChatPlatformService.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/spi/ChatPlatformBuilderTest.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/ChatPlatformServiceTest.java`

**Interfaces:**
- Consumes: All capability interfaces and degradation types from Task 2
- Produces: `ChatPlatform` interface (the core SPI), `ChatPlatform.Builder`, `ChatPlatformService`

- [ ] **Step 1: Create ChatPlatform interface with Builder**

`chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java`:
```java
package io.casehub.connectors.chat.spi;

import java.util.EnumSet;
import java.util.Objects;
import java.util.Set;

import io.casehub.connectors.chat.degraded.ChannelFallbackThreading;
import io.casehub.connectors.chat.degraded.EmptyDiscovery;
import io.casehub.connectors.chat.degraded.EmptyMembers;
import io.casehub.connectors.chat.degraded.NoOpReactions;
import io.casehub.connectors.chat.degraded.UnknownPresence;

public interface ChatPlatform {

    String id();
    Messaging messaging();
    Threading threading();
    Discovery discovery();
    Reactions reactions();
    Presence presence();
    Members members();
    boolean supports(Class<?> capability);

    static Builder builder(final String id) {
        return new Builder(id);
    }

    class Builder {
        private final String id;
        private Messaging messaging;
        private Threading threading;
        private Discovery discovery;
        private Reactions reactions;
        private Presence presence;
        private Members members;
        private final Set<Class<?>> nativeCapabilities = new java.util.HashSet<>();

        Builder(final String id) {
            this.id = Objects.requireNonNull(id, "id");
        }

        public Builder messaging(final Messaging m) { this.messaging = m; nativeCapabilities.add(Messaging.class); return this; }
        public Builder threading(final Threading t) { this.threading = t; nativeCapabilities.add(Threading.class); return this; }
        public Builder discovery(final Discovery d) { this.discovery = d; nativeCapabilities.add(Discovery.class); return this; }
        public Builder reactions(final Reactions r) { this.reactions = r; nativeCapabilities.add(Reactions.class); return this; }
        public Builder presence(final Presence p) { this.presence = p; nativeCapabilities.add(Presence.class); return this; }
        public Builder members(final Members m) { this.members = m; nativeCapabilities.add(Members.class); return this; }

        public ChatPlatform build() {
            Objects.requireNonNull(messaging, "messaging is required");
            return new DefaultChatPlatform(
                    id,
                    messaging,
                    threading != null ? threading : new ChannelFallbackThreading(messaging),
                    discovery != null ? discovery : new EmptyDiscovery(),
                    reactions != null ? reactions : new NoOpReactions(),
                    presence != null ? presence : new UnknownPresence(),
                    members != null ? members : new EmptyMembers(),
                    Set.copyOf(nativeCapabilities));
        }
    }
}
```

- [ ] **Step 2: Create DefaultChatPlatform record**

`chat-spi/src/main/java/io/casehub/connectors/chat/spi/DefaultChatPlatform.java`:
```java
package io.casehub.connectors.chat.spi;

import java.util.Set;

record DefaultChatPlatform(
        String id,
        Messaging messaging,
        Threading threading,
        Discovery discovery,
        Reactions reactions,
        Presence presence,
        Members members,
        Set<Class<?>> nativeCapabilities) implements ChatPlatform {

    @Override
    public boolean supports(final Class<?> capability) {
        return nativeCapabilities.contains(capability);
    }
}
```

- [ ] **Step 3: Create ChatPlatformService**

`chat-spi/src/main/java/io/casehub/connectors/chat/ChatPlatformService.java`:
```java
package io.casehub.connectors.chat;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkus.arc.All;

import io.casehub.connectors.chat.spi.ChatPlatform;

@ApplicationScoped
public class ChatPlatformService {

    private final Map<String, ChatPlatform> registry;

    public ChatPlatformService(@All final List<ChatPlatform> platforms) {
        this.registry = platforms.stream()
                .collect(Collectors.toMap(
                        ChatPlatform::id,
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate chat platform id: '" + a.id() + "'");
                        }));
    }

    public ChatPlatform platform(final String id) {
        final ChatPlatform platform = registry.get(id);
        if (platform == null) {
            throw new IllegalArgumentException(
                    "No chat platform registered for id '" + id
                    + "'. Available: " + registry.keySet());
        }
        return platform;
    }

    public boolean supports(final String id) {
        return registry.containsKey(id);
    }

    public Set<String> ids() {
        return Set.copyOf(registry.keySet());
    }
}
```

- [ ] **Step 4: Write Builder tests**

`chat-spi/src/test/java/io/casehub/connectors/chat/spi/ChatPlatformBuilderTest.java`:
```java
package io.casehub.connectors.chat.spi;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.degraded.ChannelFallbackThreading;
import io.casehub.connectors.chat.degraded.EmptyDiscovery;
import io.casehub.connectors.chat.degraded.EmptyMembers;
import io.casehub.connectors.chat.degraded.NoOpReactions;
import io.casehub.connectors.chat.degraded.UnknownPresence;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.SendResult;

class ChatPlatformBuilderTest {

    private static final Messaging STUB_MESSAGING = (ch, c) ->
            SendResult.success(new ChatMessageRef(ch, "stub"), Instant.now());

    @Test
    void buildFailsWithoutMessaging() {
        assertThatThrownBy(() -> ChatPlatform.builder("test").build())
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("messaging");
    }

    @Test
    void buildWithOnlyMessagingAutoDegrades() {
        ChatPlatform platform = ChatPlatform.builder("irc")
                .messaging(STUB_MESSAGING)
                .build();

        assertThat(platform.id()).isEqualTo("irc");
        assertThat(platform.messaging()).isEqualTo(STUB_MESSAGING);
        assertThat(platform.threading()).isInstanceOf(ChannelFallbackThreading.class);
        assertThat(platform.discovery()).isInstanceOf(EmptyDiscovery.class);
        assertThat(platform.reactions()).isInstanceOf(NoOpReactions.class);
        assertThat(platform.presence()).isInstanceOf(UnknownPresence.class);
        assertThat(platform.members()).isInstanceOf(EmptyMembers.class);
    }

    @Test
    void supportsReturnsTrueForExplicitlyProvided() {
        ChatPlatform platform = ChatPlatform.builder("test")
                .messaging(STUB_MESSAGING)
                .build();

        assertThat(platform.supports(Messaging.class)).isTrue();
        assertThat(platform.supports(Threading.class)).isFalse();
        assertThat(platform.supports(Discovery.class)).isFalse();
        assertThat(platform.supports(Reactions.class)).isFalse();
        assertThat(platform.supports(Presence.class)).isFalse();
        assertThat(platform.supports(Members.class)).isFalse();
    }

    @Test
    void supportsReturnsTrueForAllWhenFullyProvided() {
        Threading threading = (parent, content) -> SendResult.success(
                new ChatMessageRef(parent.channel(), "reply"), Instant.now());

        ChatPlatform platform = ChatPlatform.builder("full")
                .messaging(STUB_MESSAGING)
                .threading(threading)
                .discovery(java.util.List::of)
                .reactions(new NoOpReactions())
                .presence(m -> io.casehub.connectors.chat.model.PresenceStatus.ONLINE)
                .members(ch -> java.util.List.of())
                .build();

        assertThat(platform.supports(Messaging.class)).isTrue();
        assertThat(platform.supports(Threading.class)).isTrue();
        assertThat(platform.supports(Discovery.class)).isTrue();
        assertThat(platform.supports(Reactions.class)).isTrue();
        assertThat(platform.supports(Presence.class)).isTrue();
        assertThat(platform.supports(Members.class)).isTrue();
    }

    @Test
    void channelFallbackThreadingDelegatesToMessaging() {
        ChatChannelRef channel = new ChatChannelRef("ch1");
        ChatMessageRef parent = new ChatMessageRef(channel, "msg1");
        ChatContent content = new ChatContent("hello");

        ChatPlatform platform = ChatPlatform.builder("degraded")
                .messaging(STUB_MESSAGING)
                .build();

        SendResult result = platform.threading().reply(parent, content);
        assertThat(result.ok()).isTrue();
        assertThat(result.messageRef().channel()).isEqualTo(channel);
    }
}
```

- [ ] **Step 5: Write ChatPlatformService tests**

`chat-spi/src/test/java/io/casehub/connectors/chat/ChatPlatformServiceTest.java`:
```java
package io.casehub.connectors.chat;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;
import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Messaging;

class ChatPlatformServiceTest {

    private static final Messaging STUB = (ch, c) ->
            SendResult.success(new ChatMessageRef(ch, "s"), Instant.now());

    @Test
    void routesByPlatformId() {
        ChatPlatform p = ChatPlatform.builder("test").messaging(STUB).build();
        ChatPlatformService service = new ChatPlatformService(List.of(p));

        assertThat(service.platform("test")).isEqualTo(p);
        assertThat(service.supports("test")).isTrue();
        assertThat(service.ids()).containsExactly("test");
    }

    @Test
    void throwsOnUnknownId() {
        ChatPlatformService service = new ChatPlatformService(List.of());
        assertThatThrownBy(() -> service.platform("missing"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("missing");
    }

    @Test
    void throwsOnDuplicateId() {
        ChatPlatform a = ChatPlatform.builder("dup").messaging(STUB).build();
        ChatPlatform b = ChatPlatform.builder("dup").messaging(STUB).build();
        assertThatThrownBy(() -> new ChatPlatformService(List.of(a, b)))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Duplicate");
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```
feat(chat-spi): add ChatPlatform interface, builder, and ChatPlatformService

ChatPlatform composes six capability interfaces. Builder auto-provides degradation
defaults; messaging is required, all others auto-degrade. supports(Class<?>) reports
native capabilities. ChatPlatformService mirrors ConnectorService. — Refs #<issue>
```

---

### Task 4: Create ChatInboundAdapter and InboundTranslator

**Files:**
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/InboundTranslator.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/ChatInboundAdapter.java`
- Modify: `core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java` — add DISCORD and IRC constants
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/ChatInboundAdapterTest.java`

**Interfaces:**
- Consumes: `InboundMessage` from core, `ReceivedMessage` from Task 1, model types from Task 1
- Produces: `InboundTranslator` SPI (implemented per platform), `ChatInboundAdapter` CDI bean

- [ ] **Step 1: Add DISCORD and IRC constants to InboundConnectorTypes**

In `core/src/main/java/io/casehub/connectors/InboundConnectorTypes.java`, add:
```java
public static final String DISCORD  = "discord";
public static final String IRC      = "irc";
```

- [ ] **Step 2: Create InboundTranslator interface**

`chat-spi/src/main/java/io/casehub/connectors/chat/spi/InboundTranslator.java`:
```java
package io.casehub.connectors.chat.spi;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ReceivedMessage;

public interface InboundTranslator {
    String connectorType();
    ReceivedMessage translate(InboundMessage msg);
}
```

- [ ] **Step 3: Create ChatInboundAdapter**

`chat-spi/src/main/java/io/casehub/connectors/chat/ChatInboundAdapter.java`:
```java
package io.casehub.connectors.chat;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.logging.Logger;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import io.quarkus.arc.All;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

@ApplicationScoped
public class ChatInboundAdapter {

    private static final Logger LOG = Logger.getLogger(ChatInboundAdapter.class.getName());

    private final Map<String, InboundTranslator> translators;
    private final Event<ReceivedMessage> receivedEvent;

    @Inject
    ChatInboundAdapter(@All final List<InboundTranslator> translators,
                       final Event<ReceivedMessage> receivedEvent) {
        this.translators = translators.stream()
                .collect(Collectors.toMap(
                        InboundTranslator::connectorType,
                        Function.identity()));
        this.receivedEvent = receivedEvent;
    }

    public void onMessage(@ObservesAsync final InboundMessage msg) {
        final InboundTranslator translator = translators.get(msg.connectorType());
        if (translator != null) {
            try {
                receivedEvent.fireAsync(translator.translate(msg));
            } catch (final Exception e) {
                LOG.warning("ChatInboundAdapter: translation failed for "
                        + msg.connectorType() + ": " + e.getMessage());
            }
        }
    }
}
```

- [ ] **Step 4: Write ChatInboundAdapter tests**

`chat-spi/src/test/java/io/casehub/connectors/chat/ChatInboundAdapterTest.java`:
```java
package io.casehub.connectors.chat;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

import jakarta.enterprise.event.Event;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

class ChatInboundAdapterTest {

    @Test
    void translatesMatchingConnectorType() {
        List<ReceivedMessage> received = new ArrayList<>();
        Event<ReceivedMessage> mockEvent = createRecordingEvent(received);

        InboundTranslator translator = new InboundTranslator() {
            @Override public String connectorType() { return "test-chat"; }
            @Override public ReceivedMessage translate(InboundMessage msg) {
                return new ReceivedMessage("test-chat",
                        new ChatChannelRef(msg.externalChannelRef()),
                        new ChatMessageRef(new ChatChannelRef(msg.externalChannelRef()), "m1"),
                        null, new MemberRef(msg.externalSenderId()),
                        new ChatContent(msg.content()), msg.receivedAt());
            }
        };

        ChatInboundAdapter adapter = new ChatInboundAdapter(List.of(translator), mockEvent);

        InboundMessage msg = new InboundMessage("test-inbound", "test-chat",
                "user1", "channel1", "hello", List.of(), Instant.now(), Map.of(), null);

        adapter.onMessage(msg);

        assertThat(received).hasSize(1);
        assertThat(received.get(0).platformId()).isEqualTo("test-chat");
        assertThat(received.get(0).content().text()).isEqualTo("hello");
    }

    @Test
    void ignoresNonMatchingConnectorType() {
        List<ReceivedMessage> received = new ArrayList<>();
        Event<ReceivedMessage> mockEvent = createRecordingEvent(received);

        InboundTranslator translator = new InboundTranslator() {
            @Override public String connectorType() { return "test-chat"; }
            @Override public ReceivedMessage translate(InboundMessage msg) {
                throw new AssertionError("Should not be called");
            }
        };

        ChatInboundAdapter adapter = new ChatInboundAdapter(List.of(translator), mockEvent);

        InboundMessage msg = new InboundMessage("email-inbound", "email",
                "sender", "inbox", "hi", List.of(), Instant.now(), Map.of(), null);

        adapter.onMessage(msg);

        assertThat(received).isEmpty();
    }

    @Test
    void translationFailureIsSwallowed() {
        List<ReceivedMessage> received = new ArrayList<>();
        Event<ReceivedMessage> mockEvent = createRecordingEvent(received);

        InboundTranslator translator = new InboundTranslator() {
            @Override public String connectorType() { return "broken"; }
            @Override public ReceivedMessage translate(InboundMessage msg) {
                throw new RuntimeException("parse error");
            }
        };

        ChatInboundAdapter adapter = new ChatInboundAdapter(List.of(translator), mockEvent);

        InboundMessage msg = new InboundMessage("broken-inbound", "broken",
                "user", "ch", "text", List.of(), Instant.now(), Map.of(), null);

        adapter.onMessage(msg);

        assertThat(received).isEmpty();
    }

    @SuppressWarnings("unchecked")
    private static Event<ReceivedMessage> createRecordingEvent(List<ReceivedMessage> received) {
        return new Event<>() {
            @Override public void fire(ReceivedMessage event) { received.add(event); }
            @Override public <U extends ReceivedMessage> CompletableFuture<U> fireAsync(U event) {
                received.add(event);
                return (CompletableFuture<U>) CompletableFuture.completedFuture(event);
            }
            @Override public <U extends ReceivedMessage> CompletableFuture<U> fireAsync(U event, java.lang.annotation.Annotation... qualifiers) {
                return fireAsync(event);
            }
            @Override public Event<ReceivedMessage> select(java.lang.annotation.Annotation... qualifiers) { return this; }
            @Override public <U extends ReceivedMessage> Event<U> select(Class<U> subtype, java.lang.annotation.Annotation... qualifiers) { throw new UnsupportedOperationException(); }
            @Override public <U extends ReceivedMessage> Event<U> select(jakarta.enterprise.util.TypeLiteral<U> subtype, java.lang.annotation.Annotation... qualifiers) { throw new UnsupportedOperationException(); }
        };
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core,chat-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests PASS (core tests still pass with new constants; chat-spi adapter tests pass)

- [ ] **Step 6: Commit**

```
feat(chat-spi): add ChatInboundAdapter + InboundTranslator for typed inbound events

ChatInboundAdapter observes @ObservesAsync InboundMessage, delegates to matching
InboundTranslator, fires ReceivedMessage on CDI event bus. Fail-soft: translation
errors logged and swallowed. Also adds DISCORD and IRC constants to
InboundConnectorTypes in core. — Refs #<issue>
```

---

### Task 5: Create chat-ref Module — In-Memory Reference Implementation

**Files:**
- Create: `chat-ref/pom.xml`
- Create: `chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java`
- Create: `chat-ref/src/main/java/io/casehub/connectors/chat/ref/InMemoryStore.java`
- Create: `chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefInboundTranslator.java`
- Modify: `pom.xml` (parent) — add `<module>chat-ref</module>`
- Test: `chat-ref/src/test/java/io/casehub/connectors/chat/ref/RefChatPlatformContractTest.java`

**Interfaces:**
- Consumes: `ChatPlatform`, all capability interfaces, model types from Tasks 1-3
- Produces: Full-fidelity in-memory `ChatPlatform` implementation — SPI contract test target

- [ ] **Step 1: Create `chat-ref/pom.xml`**

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

  <artifactId>casehub-connectors-chat-ref</artifactId>
  <name>CaseHub Connectors — Chat Reference Implementation</name>
  <description>In-memory ChatPlatform implementation for SPI contract testing.
Full fidelity — all capabilities native. No external infrastructure.</description>

  <dependencies>
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

- [ ] **Step 2: Add `chat-ref` module to parent POM**

In `pom.xml` (root), add `<module>chat-ref</module>` after `<module>chat-spi</module>`.

- [ ] **Step 3: Create InMemoryStore — shared state for the ref impl**

`chat-ref/src/main/java/io/casehub/connectors/chat/ref/InMemoryStore.java`:
```java
package io.casehub.connectors.chat.ref;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.Member;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;
import io.casehub.connectors.chat.model.SendResult;

class InMemoryStore {

    record StoredMessage(ChatMessageRef ref, ChatChannelRef channel, ChatContent content,
                         ChatMessageRef parentRef, Instant timestamp) {}

    final Map<String, Channel> channels = new ConcurrentHashMap<>();
    final Map<String, List<StoredMessage>> messagesByChannel = new ConcurrentHashMap<>();
    final Map<String, List<String>> reactionsByMessage = new ConcurrentHashMap<>();
    final Map<String, PresenceStatus> presenceByMember = new ConcurrentHashMap<>();
    final Map<String, List<MemberRef>> membersByChannel = new ConcurrentHashMap<>();

    SendResult store(final ChatChannelRef channel, final ChatContent content,
                     final ChatMessageRef parentRef) {
        String messageId = UUID.randomUUID().toString();
        ChatMessageRef ref = new ChatMessageRef(channel, messageId);
        Instant now = Instant.now();
        StoredMessage stored = new StoredMessage(ref, channel, content, parentRef, now);
        messagesByChannel.computeIfAbsent(channel.id(), k -> new ArrayList<>()).add(stored);
        return SendResult.success(ref, now);
    }

    void addChannel(final Channel channel) {
        channels.put(channel.ref().id(), channel);
    }

    void addMember(final String channelId, final MemberRef member) {
        membersByChannel.computeIfAbsent(channelId, k -> new ArrayList<>()).add(member);
    }
}
```

- [ ] **Step 4: Create RefChatPlatform**

`chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java`:
```java
package io.casehub.connectors.chat.ref;

import java.util.List;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.Member;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;

@ApplicationScoped
public class RefChatPlatform implements ChatPlatform {

    private static final Set<Class<?>> ALL_CAPABILITIES = Set.of(
            Messaging.class, Threading.class, Discovery.class,
            Reactions.class, Presence.class, Members.class);

    final InMemoryStore store = new InMemoryStore();

    @Override public String id() { return "ref"; }

    @Override
    public Messaging messaging() {
        return (channel, content) -> store.store(channel, content, null);
    }

    @Override
    public Threading threading() {
        return (parent, content) -> store.store(parent.channel(), content, parent);
    }

    @Override
    public Discovery discovery() {
        return () -> List.copyOf(store.channels.values());
    }

    @Override
    public Reactions reactions() {
        return new Reactions() {
            @Override public void add(ChatMessageRef message, String emoji) {
                store.reactionsByMessage
                        .computeIfAbsent(message.messageId(), k -> new java.util.ArrayList<>())
                        .add(emoji);
            }
            @Override public void remove(ChatMessageRef message, String emoji) {
                List<String> reactions = store.reactionsByMessage.get(message.messageId());
                if (reactions != null) reactions.remove(emoji);
            }
        };
    }

    @Override
    public Presence presence() {
        return member -> store.presenceByMember.getOrDefault(member.id(), PresenceStatus.UNKNOWN);
    }

    @Override
    public Members members() {
        return channel -> {
            List<MemberRef> refs = store.membersByChannel.get(channel.id());
            if (refs == null) return List.of();
            return refs.stream()
                    .map(ref -> new Member(ref, ref.id()))
                    .toList();
        };
    }

    @Override
    public boolean supports(final Class<?> capability) {
        return ALL_CAPABILITIES.contains(capability);
    }

    public void addChannel(final Channel channel) { store.addChannel(channel); }
    public void addMember(final String channelId, final MemberRef member) { store.addMember(channelId, member); }
    public void setPresence(final String memberId, final PresenceStatus status) { store.presenceByMember.put(memberId, status); }
    public List<String> getReactions(final String messageId) { return store.reactionsByMessage.getOrDefault(messageId, List.of()); }
}
```

- [ ] **Step 5: Create RefInboundTranslator**

`chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefInboundTranslator.java`:
```java
package io.casehub.connectors.chat.ref;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

@ApplicationScoped
public class RefInboundTranslator implements InboundTranslator {

    static final String CONNECTOR_TYPE = "ref";

    @Override
    public String connectorType() {
        return CONNECTOR_TYPE;
    }

    @Override
    public ReceivedMessage translate(final InboundMessage msg) {
        ChatChannelRef channel = new ChatChannelRef(msg.externalChannelRef());
        String messageId = msg.metadata().getOrDefault("message-id",
                java.util.UUID.randomUUID().toString());
        String parentId = msg.metadata().get("parent-id");
        ChatMessageRef messageRef = new ChatMessageRef(channel, messageId);
        ChatMessageRef parentRef = parentId != null
                ? new ChatMessageRef(channel, parentId) : null;
        return new ReceivedMessage(CONNECTOR_TYPE, channel, messageRef, parentRef,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content(), null, msg.attachments()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 6: Write SPI contract tests**

`chat-ref/src/test/java/io/casehub/connectors/chat/ref/RefChatPlatformContractTest.java`:
```java
package io.casehub.connectors.chat.ref;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.PresenceStatus;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;

class RefChatPlatformContractTest {

    private RefChatPlatform platform;
    private ChatChannelRef channel;

    @BeforeEach
    void setUp() {
        platform = new RefChatPlatform();
        channel = new ChatChannelRef("general");
        platform.addChannel(new Channel(channel, "general", "General chat", false));
    }

    @Test
    void idIsRef() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsAllCapabilities() {
        assertThat(platform.supports(Messaging.class)).isTrue();
        assertThat(platform.supports(Threading.class)).isTrue();
        assertThat(platform.supports(Discovery.class)).isTrue();
        assertThat(platform.supports(Reactions.class)).isTrue();
        assertThat(platform.supports(Presence.class)).isTrue();
        assertThat(platform.supports(Members.class)).isTrue();
    }

    @Test
    void sendToChannelReturnsSuccessWithMessageRef() {
        SendResult result = platform.messaging().send(channel, new ChatContent("hello"));
        assertThat(result.ok()).isTrue();
        assertThat(result.messageRef()).isNotNull();
        assertThat(result.messageRef().channel()).isEqualTo(channel);
        assertThat(result.messageRef().messageId()).isNotBlank();
        assertThat(result.timestamp()).isNotNull();
    }

    @Test
    void replyToMessageReturnsSuccessWithParentChannel() {
        SendResult original = platform.messaging().send(channel, new ChatContent("original"));
        SendResult reply = platform.threading().reply(
                original.messageRef(), new ChatContent("reply"));

        assertThat(reply.ok()).isTrue();
        assertThat(reply.messageRef().channel()).isEqualTo(channel);
        assertThat(reply.messageRef().messageId())
                .isNotEqualTo(original.messageRef().messageId());
    }

    @Test
    void replyToReplyCreatesChain() {
        SendResult msg1 = platform.messaging().send(channel, new ChatContent("msg1"));
        SendResult msg2 = platform.threading().reply(msg1.messageRef(), new ChatContent("msg2"));
        SendResult msg3 = platform.threading().reply(msg2.messageRef(), new ChatContent("msg3"));

        assertThat(msg3.ok()).isTrue();
        assertThat(msg3.messageRef().channel()).isEqualTo(channel);
    }

    @Test
    void listChannelsReturnsAddedChannels() {
        ChatChannelRef ch2 = new ChatChannelRef("random");
        platform.addChannel(new Channel(ch2, "random", "Random chat", false));

        assertThat(platform.discovery().listChannels()).hasSize(2);
    }

    @Test
    void addAndRemoveReaction() {
        SendResult msg = platform.messaging().send(channel, new ChatContent("react to me"));
        platform.reactions().add(msg.messageRef(), "thumbsup");
        assertThat(platform.getReactions(msg.messageRef().messageId()))
                .containsExactly("thumbsup");

        platform.reactions().remove(msg.messageRef(), "thumbsup");
        assertThat(platform.getReactions(msg.messageRef().messageId())).isEmpty();
    }

    @Test
    void presenceReflectsSetState() {
        MemberRef member = new MemberRef("user1");
        assertThat(platform.presence().of(member)).isEqualTo(PresenceStatus.UNKNOWN);

        platform.setPresence("user1", PresenceStatus.ONLINE);
        assertThat(platform.presence().of(member)).isEqualTo(PresenceStatus.ONLINE);
    }

    @Test
    void listMembersReturnsAddedMembers() {
        MemberRef m1 = new MemberRef("user1");
        MemberRef m2 = new MemberRef("user2");
        platform.addMember("general", m1);
        platform.addMember("general", m2);

        assertThat(platform.members().list(channel)).hasSize(2);
    }

    @Test
    void listMembersReturnsEmptyForUnknownChannel() {
        assertThat(platform.members().list(new ChatChannelRef("nonexistent"))).isEmpty();
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi,chat-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: All tests PASS

- [ ] **Step 8: Run full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all existing modules still pass

- [ ] **Step 9: Commit**

```
feat(chat-ref): add in-memory ChatPlatform reference implementation

Full-fidelity in-memory implementation with all six capabilities native.
InMemoryStore holds channels, messages, reactions, presence, members.
RefInboundTranslator translates InboundMessage → ReceivedMessage.
SPI contract tests validate the complete ChatPlatform model. — Refs #<issue>
```

---

## Plan Scope and Next Steps

This plan covers the **chat-spi** and **chat-ref** modules — the SPI foundation. After completing these tasks, the following separate plans are needed:

1. **chat-slack** — SlackChatPlatform adapting SlackBotClient, SlackInboundTranslator, SlackConnectorDiscovery
2. **chat-discord** — DiscordChatPlatform, DiscordClient, DiscordInboundConnector, DiscordInboundTranslator
3. **chat-irc** — IrcChatPlatform, IrcClient, IrcInboundConnector, IrcInboundTranslator

Each plan follows the spec's layer build order (messaging first, then discovery, threading, inbound, reactions, presence, members) within the platform module.
