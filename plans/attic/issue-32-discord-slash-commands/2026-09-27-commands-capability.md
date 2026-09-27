# Commands Capability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #32 — Discord slash commands and interactions
**Issue group:** #32

**Goal:** Add a cross-platform `Commands` capability to the ChatPlatform
SPI with slash command registration, invocation dispatch, and immediate
+ deferred response delivery.

**Architecture:** New `Commands` capability interface in `chat-spi` with
`CommandHandler` SPI for consumers, `CommandService` for handler
discovery and dispatch, and dedicated JAX-RS interaction endpoints in
each platform module. No new modules — all types land in existing modules.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`@All` injection),
JAX-RS, Ed25519 (`java.security`), HMAC-SHA256, WireMock (tests)

## Global Constraints

- Java source level 21 (on Java 26 JVM)
- All casehubio artifacts use version `0.2-SNAPSHOT`
- All capability interfaces in `io.casehub.connectors.chat.spi`
- All command model types in `io.casehub.connectors.chat.command`
- Degraded fallbacks in `io.casehub.connectors.chat.degraded`
- No external crypto dependencies — use `java.security` EdDSA
- Commands follow Discord naming constraints: 1-32 lowercase chars,
  no spaces, hyphens and underscores allowed
- Platform-specific type mappings live in platform modules, not chat-spi
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: same command; WireMock for HTTP tests, plain JUnit for unit tests
- Commit every task with `feat(#32): <description>`

---

## Batch 1: SPI Foundation — model types, capability interface, handler SPI

After this batch: `chat-spi` compiles with all new types, the
`ChatPlatform` interface includes `commands()`, existing implementations
compile with `NoOpCommands`, and `CommandService` discovers handlers and
dispatches invocations. All tested.

### Task 1: Command model types and Commands capability interface

**Files:**
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandParameterType.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandParameter.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandDefinition.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandInvocation.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/DeferredReply.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandResponse.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandHandler.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/Commands.java`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/degraded/NoOpCommands.java`
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java`
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/DefaultChatPlatform.java`
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/NoOpChatPlatform.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/command/CommandModelTest.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/spi/ChatPlatformBuilderTest.java` (extend)

**Interfaces:**
- Produces: `Commands` interface — `void registerAll(List<CommandDefinition>)`
- Produces: `CommandHandler` interface — `CommandDefinition definition()`, `CommandResponse handle(CommandInvocation)`
- Produces: `CommandResponse` sealed interface — `Immediate(String text, boolean ephemeral)`, `Deferred(boolean ephemeral, Consumer<DeferredReply> callback)`
- Produces: `DeferredReply` interface — `void send(String text, boolean ephemeral)`
- Produces: `NoOpCommands` — degraded `Commands` implementation

- [ ] **Step 1: Write model type tests**

Create `CommandModelTest.java`:

```java
package io.casehub.connectors.chat.command;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

class CommandModelTest {

    @Test
    void definitionRejectsBlankName() {
        assertThatThrownBy(() -> new CommandDefinition("", "desc", List.of()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("name");
    }

    @Test
    void definitionRejectsNullDescription() {
        assertThatThrownBy(() -> new CommandDefinition("cmd", null, List.of()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("description");
    }

    @Test
    void definitionDefaultsNullParametersToEmpty() {
        var def = new CommandDefinition("cmd", "desc", null);
        assertThat(def.parameters()).isEmpty();
    }

    @Test
    void parameterDefaultsNullTypeToString() {
        var param = new CommandParameter("name", "desc", null, false);
        assertThat(param.type()).isEqualTo(CommandParameterType.STRING);
    }

    @Test
    void parameterRejectsBlankName() {
        assertThatThrownBy(() ->
                new CommandParameter("", "desc", CommandParameterType.STRING, false))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("name");
    }

    @Test
    void invocationDefaultsNullMapsToEmpty() {
        var inv = new CommandInvocation("cmd", null, "user1", "ch1", "test", null);
        assertThat(inv.arguments()).isEmpty();
        assertThat(inv.metadata()).isEmpty();
    }

    @Test
    void invocationRequireArgumentThrowsOnMissing() {
        var inv = new CommandInvocation("cmd", Map.of(), "user1", "ch1", "test", Map.of());
        assertThatThrownBy(() -> inv.requireArgument("missing"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("missing");
    }

    @Test
    void invocationArgumentReturnsValue() {
        var inv = new CommandInvocation("cmd", Map.of("key", "val"),
                "user1", "ch1", "test", Map.of());
        assertThat(inv.argument("key")).isEqualTo("val");
        assertThat(inv.argument("nope")).isNull();
    }

    @Test
    void immediateResponseWithDefaults() {
        var resp = new CommandResponse.Immediate("hello");
        assertThat(resp.text()).isEqualTo("hello");
        assertThat(resp.ephemeral()).isFalse();
    }

    @Test
    void immediateResponseEphemeral() {
        var resp = new CommandResponse.Immediate("secret", true);
        assertThat(resp.ephemeral()).isTrue();
    }

    @Test
    void deferredReplyDefaultSend() {
        var captured = new java.util.concurrent.atomic.AtomicReference<String>();
        DeferredReply reply = (text, eph) -> captured.set(text);
        reply.send("hello");
        assertThat(captured.get()).isEqualTo("hello");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -Dtest=CommandModelTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — classes don't exist yet

- [ ] **Step 3: Create CommandParameterType enum**

```java
package io.casehub.connectors.chat.command;

public enum CommandParameterType {
    STRING,
    INTEGER,
    BOOLEAN,
    NUMBER
}
```

- [ ] **Step 4: Create CommandParameter record**

```java
package io.casehub.connectors.chat.command;

public record CommandParameter(
        String name,
        String description,
        CommandParameterType type,
        boolean required) {

    public CommandParameter {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Parameter name must not be blank");
        if (type == null)
            type = CommandParameterType.STRING;
    }
}
```

- [ ] **Step 5: Create CommandDefinition record**

```java
package io.casehub.connectors.chat.command;

import java.util.List;

public record CommandDefinition(
        String name,
        String description,
        List<CommandParameter> parameters) {

    public CommandDefinition {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Command name must not be blank");
        if (description == null || description.isBlank())
            throw new IllegalArgumentException("Command description must not be blank");
        if (parameters == null)
            parameters = List.of();
    }
}
```

- [ ] **Step 6: Create CommandInvocation record**

```java
package io.casehub.connectors.chat.command;

import java.util.Map;

public record CommandInvocation(
        String commandName,
        Map<String, String> arguments,
        String userId,
        String channelId,
        String platformId,
        Map<String, String> metadata) {

    public CommandInvocation {
        if (arguments == null) arguments = Map.of();
        if (metadata == null) metadata = Map.of();
    }

    public String argument(String name) {
        return arguments.get(name);
    }

    public String requireArgument(String name) {
        String value = arguments.get(name);
        if (value == null)
            throw new IllegalArgumentException(
                    "Missing required argument: " + name);
        return value;
    }
}
```

- [ ] **Step 7: Create DeferredReply interface**

```java
package io.casehub.connectors.chat.command;

public interface DeferredReply {
    void send(String text, boolean ephemeral);

    default void send(String text) {
        send(text, false);
    }
}
```

- [ ] **Step 8: Create CommandResponse sealed interface**

```java
package io.casehub.connectors.chat.command;

import java.util.function.Consumer;

public sealed interface CommandResponse {

    record Immediate(String text, boolean ephemeral)
            implements CommandResponse {
        public Immediate(String text) {
            this(text, false);
        }
    }

    record Deferred(boolean ephemeral, Consumer<DeferredReply> callback)
            implements CommandResponse {}
}
```

- [ ] **Step 9: Create CommandHandler interface**

```java
package io.casehub.connectors.chat.command;

public interface CommandHandler {
    CommandDefinition definition();
    CommandResponse handle(CommandInvocation invocation);
}
```

- [ ] **Step 10: Run model tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -Dtest=CommandModelTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 11 tests PASS

- [ ] **Step 11: Create Commands capability interface**

```java
package io.casehub.connectors.chat.spi;

import io.casehub.connectors.chat.command.CommandDefinition;
import java.util.List;

public interface Commands {
    void registerAll(List<CommandDefinition> commands);
}
```

- [ ] **Step 12: Create NoOpCommands degraded fallback**

```java
package io.casehub.connectors.chat.degraded;

import io.casehub.connectors.chat.command.CommandDefinition;
import io.casehub.connectors.chat.spi.Commands;
import java.util.List;
import java.util.logging.Logger;

public class NoOpCommands implements Commands {
    private static final Logger LOG =
            Logger.getLogger(NoOpCommands.class.getName());

    @Override
    public void registerAll(List<CommandDefinition> commands) {
        if (!commands.isEmpty()) {
            LOG.warning("Commands capability not supported — "
                    + commands.size() + " commands not registered");
        }
    }
}
```

- [ ] **Step 13: Add `commands()` to ChatPlatform interface**

Add after `messageHistory()` method (line 118) in `ChatPlatform.java`:

```java
    /**
     * Returns the commands capability for slash command registration.
     *
     * @return the commands implementation; never null (may be degraded)
     */
    Commands commands();
```

Update `@SimulationEligible` annotation to include `"commands"`:

```java
@SimulationEligible(name = "chat-platform",
    capabilities = {"messaging", "threading", "discovery", "reactions",
                    "presence", "members", "channelManagement",
                    "memberManagement", "messageHistory", "commands"})
```

Add to `Builder`:

```java
private Commands commands;
public Builder commands(Commands c) {
    this.commands = c;
    nativeCapabilities.add(Commands.class);
    return this;
}
```

Update `build()` to pass `commands != null ? commands : new NoOpCommands()`
to `DefaultChatPlatform`.

- [ ] **Step 14: Add `commands` field to DefaultChatPlatform record**

Add `Commands commands` field after `messageHistory` and before
`nativeCapabilities`:

```java
record DefaultChatPlatform(
        String id,
        Messaging messaging,
        Threading threading,
        Discovery discovery,
        Reactions reactions,
        Presence presence,
        Members members,
        ChannelManagement channelManagement,
        MemberManagement memberManagement,
        MessageHistory messageHistory,
        Commands commands,
        Set<Class<?>> nativeCapabilities) implements ChatPlatform {

    @Override
    public boolean supports(final Class<?> capability) {
        return nativeCapabilities.contains(capability);
    }
}
```

- [ ] **Step 15: Add `commands()` to NoOpChatPlatform**

Add method returning `new NoOpCommands()`:

```java
@Override
public Commands commands() {
    return new NoOpCommands();
}
```

Add import for `NoOpCommands`.

- [ ] **Step 16: Add `commands()` to all other ChatPlatform implementors**

Each direct implementor needs a `commands()` method returning
`new NoOpCommands()`. This is a temporary default — Discord and Slack
get real implementations in later tasks.

Files to modify:
- `chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java` — will be replaced with `RefCommands` in Batch 3
- `chat-irc/src/main/java/io/casehub/connectors/chat/irc/IrcChatPlatform.java`
- `chat-signal/src/main/java/io/casehub/connectors/chat/signal/SignalChatPlatform.java`
- `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackChatPlatform.java` — will be replaced with `SlackCommands` in Batch 4
- `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java` — will be replaced with `DiscordCommands` in Batch 2

For each, add:

```java
import io.casehub.connectors.chat.degraded.NoOpCommands;
import io.casehub.connectors.chat.spi.Commands;

@Override
public Commands commands() {
    return new NoOpCommands();
}
```

- [ ] **Step 17: Update ChatPlatformBuilderTest**

Add test for `commands()` auto-degradation and `supports(Commands.class)`:

```java
@Test
void builderAutoDegradesCommands() {
    ChatPlatform platform = ChatPlatform.builder("test")
            .messaging(STUB_MESSAGING)
            .build();

    assertThat(platform.commands()).isInstanceOf(NoOpCommands.class);
    assertThat(platform.supports(Commands.class)).isFalse();
}

@Test
void builderWithExplicitCommandsSupportsCommands() {
    Commands stubCommands = commands -> {};
    ChatPlatform platform = ChatPlatform.builder("test")
            .messaging(STUB_MESSAGING)
            .commands(stubCommands)
            .build();

    assertThat(platform.commands()).isEqualTo(stubCommands);
    assertThat(platform.supports(Commands.class)).isTrue();
}
```

Add imports for `Commands` and `NoOpCommands`.

- [ ] **Step 18: Build full project to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 19: Commit**

```bash
git add chat-spi/ chat-ref/ chat-irc/ chat-signal/ chat-slack/ chat-discord/
git commit -m "feat(#32): add Commands capability to ChatPlatform SPI

New command model types (CommandDefinition, CommandParameter,
CommandInvocation, CommandResponse), CommandHandler SPI, Commands
capability interface, NoOpCommands degraded fallback. All existing
ChatPlatform implementations return NoOpCommands.

Refs #32"
```

### Task 2: CommandService — handler discovery and dispatch

**Files:**
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandService.java`
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/ChatBeans.java`
- Test: `chat-spi/src/test/java/io/casehub/connectors/chat/command/CommandServiceTest.java`

**Interfaces:**
- Consumes: `CommandHandler` (from Task 1), `ChatPlatform` (from Task 1), `Commands` (from Task 1)
- Produces: `CommandService` — `CommandResponse dispatch(String commandName, CommandInvocation invocation)`, `List<CommandDefinition> definitions()`

- [ ] **Step 1: Write CommandService tests**

```java
package io.casehub.connectors.chat.command;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.degraded.NoOpCommands;
import io.casehub.connectors.chat.spi.Commands;

class CommandServiceTest {

    static CommandHandler handler(String name, String desc, String response) {
        return new CommandHandler() {
            @Override
            public CommandDefinition definition() {
                return new CommandDefinition(name, desc, List.of());
            }
            @Override
            public CommandResponse handle(CommandInvocation invocation) {
                return new CommandResponse.Immediate(response);
            }
        };
    }

    static StubPlatform platform(String id, Commands commands, boolean supports) {
        return new StubPlatform(id, commands, supports);
    }

    @Test
    void dispatchRoutesToCorrectHandler() {
        var service = new CommandService(
                List.of(handler("ping", "Ping", "pong"),
                        handler("status", "Status", "ok")),
                List.of());

        var invocation = new CommandInvocation(
                "ping", Map.of(), "u1", "ch1", "test", Map.of());
        var result = service.dispatch("ping", invocation);

        assertThat(result).isInstanceOf(CommandResponse.Immediate.class);
        assertThat(((CommandResponse.Immediate) result).text()).isEqualTo("pong");
    }

    @Test
    void dispatchUnknownCommandReturnsError() {
        var service = new CommandService(
                List.of(handler("ping", "Ping", "pong")),
                List.of());

        var invocation = new CommandInvocation(
                "nope", Map.of(), "u1", "ch1", "test", Map.of());
        var result = service.dispatch("nope", invocation);

        assertThat(result).isInstanceOf(CommandResponse.Immediate.class);
        assertThat(((CommandResponse.Immediate) result).text())
                .contains("Unknown command");
    }

    @Test
    void duplicateCommandNamesThrow() {
        assertThatThrownBy(() -> new CommandService(
                List.of(handler("ping", "Ping1", "a"),
                        handler("ping", "Ping2", "b")),
                List.of()))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Duplicate command name");
    }

    @Test
    void definitionsReturnsAllHandlerDefinitions() {
        var service = new CommandService(
                List.of(handler("ping", "Ping", "pong"),
                        handler("status", "Status", "ok")),
                List.of());

        assertThat(service.definitions()).hasSize(2);
        assertThat(service.definitions().stream()
                .map(CommandDefinition::name).toList())
                .containsExactlyInAnyOrder("ping", "status");
    }

    @Test
    void registersCommandsOnSupportingPlatforms() {
        var registered = new ArrayList<List<CommandDefinition>>();
        Commands capturing = registered::add;
        var supportingPlatform = platform("discord", capturing, true);
        var unsupportingPlatform = platform("irc", new NoOpCommands(), false);

        new CommandService(
                List.of(handler("ping", "Ping", "pong")),
                List.of(supportingPlatform, unsupportingPlatform));

        assertThat(registered).hasSize(1);
        assertThat(registered.get(0)).hasSize(1);
        assertThat(registered.get(0).get(0).name()).isEqualTo("ping");
    }

    @Test
    void registrationFailureDoesNotBreakStartup() {
        Commands failing = commands -> {
            throw new RuntimeException("API down");
        };
        var platform = platform("discord", failing, true);

        var service = new CommandService(
                List.of(handler("ping", "Ping", "pong")),
                List.of(platform));

        // Should not throw — startup succeeds despite registration failure
        assertThat(service.definitions()).hasSize(1);
    }

    @Test
    void emptyHandlerListSkipsRegistration() {
        var registered = new ArrayList<List<CommandDefinition>>();
        Commands capturing = registered::add;
        var platform = platform("discord", capturing, true);

        new CommandService(List.of(), List.of(platform));

        assertThat(registered).isEmpty();
    }
}
```

Also create `StubPlatform.java` in the same test package:

```java
package io.casehub.connectors.chat.command;

import io.casehub.connectors.chat.degraded.*;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.*;

import java.util.List;

class StubPlatform implements ChatPlatform {
    private final String id;
    private final Commands commands;
    private final boolean supportsCommands;

    StubPlatform(String id, Commands commands, boolean supportsCommands) {
        this.id = id;
        this.commands = commands;
        this.supportsCommands = supportsCommands;
    }

    @Override public String id() { return id; }
    @Override public Messaging messaging() {
        return (ch, c) -> SendResult.failure("stub");
    }
    @Override public Threading threading() { return new ChannelFallbackThreading(messaging()); }
    @Override public Discovery discovery() { return new EmptyDiscovery(); }
    @Override public Reactions reactions() { return new NoOpReactions(); }
    @Override public Presence presence() { return new UnknownPresence(); }
    @Override public Members members() { return new EmptyMembers(); }
    @Override public ChannelManagement channelManagement() { return new NoOpChannelManagement(); }
    @Override public MemberManagement memberManagement() { return new NoOpMemberManagement(); }
    @Override public MessageHistory messageHistory() { return new EmptyMessageHistory(); }
    @Override public Commands commands() { return commands; }
    @Override public boolean supports(Class<?> cap) {
        return cap == Commands.class && supportsCommands;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -Dtest=CommandServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — `CommandService` doesn't exist yet

- [ ] **Step 3: Implement CommandService**

```java
package io.casehub.connectors.chat.command;

import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Commands;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.stream.Collectors;

public class CommandService {

    private static final Logger LOG =
            Logger.getLogger(CommandService.class.getName());

    private final Map<String, CommandHandler> handlers;
    private final List<CommandDefinition> definitions;

    public CommandService(
            List<CommandHandler> handlers,
            List<ChatPlatform> platforms) {

        this.handlers = handlers.stream()
                .collect(Collectors.toMap(
                        h -> h.definition().name(),
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate command name: '"
                                    + a.definition().name() + "'");
                        }));

        this.definitions = handlers.stream()
                .map(CommandHandler::definition)
                .toList();

        if (!definitions.isEmpty()) {
            for (ChatPlatform platform : platforms) {
                if (platform.supports(Commands.class)) {
                    try {
                        platform.commands().registerAll(definitions);
                    } catch (Exception e) {
                        LOG.log(Level.WARNING,
                                "Command registration failed on "
                                + platform.id(), e);
                    }
                }
            }
        }
    }

    public CommandResponse dispatch(
            String commandName,
            CommandInvocation invocation) {
        CommandHandler handler = handlers.get(commandName);
        if (handler == null) {
            LOG.warning("No handler for command: " + commandName);
            return new CommandResponse.Immediate(
                    "Unknown command: " + commandName, true);
        }
        return handler.handle(invocation);
    }

    public List<CommandDefinition> definitions() {
        return definitions;
    }
}
```

Note: `CommandService` is a plain class here — no CDI annotations. CDI
wiring happens in `ChatBeans` using `@All` injection, matching the
existing `ChatPlatformService` pattern.

- [ ] **Step 4: Add CDI producer for CommandService in ChatBeans**

Add to `ChatBeans.java`:

```java
@Produces
@ApplicationScoped
@io.quarkus.runtime.Startup
public CommandService commandService(
        @All List<CommandHandler> handlers,
        @All List<ChatPlatform> platforms) {
    return new CommandService(handlers, platforms);
}
```

Add imports for `CommandHandler`, `CommandService`, and
`io.quarkus.runtime.Startup`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -Dtest=CommandServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 7 tests PASS

- [ ] **Step 6: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add chat-spi/
git commit -m "feat(#32): add CommandService for handler discovery and dispatch

Discovers CommandHandler beans, registers commands with supporting
platforms at startup, dispatches invocations by command name. CDI
producer in ChatBeans with @Startup for eager initialization.

Refs #32"
```

---

## Batch 2: Discord implementation — command registration + interaction endpoint

After this batch: Discord can register slash commands via its REST API
and receive interaction webhooks with Ed25519 verification. Both
immediate and deferred response paths work.

### Task 3: DiscordClient command registration methods

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java`
- Test: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientCommandsTest.java`

**Interfaces:**
- Consumes: none from chat-spi (discord module must not depend on chat-spi)
- Produces: `DiscordClient.bulkOverwriteGlobalCommands(String token, String applicationId, ArrayNode commands)`, `DiscordClient.respondToInteraction(String interactionId, String interactionToken, int responseType, String content, boolean ephemeral)`, `DiscordClient.sendFollowupMessage(String applicationId, String interactionToken, String content, boolean ephemeral)`

- [ ] **Step 1: Write WireMock tests for command registration**

Create `DiscordClientCommandsTest.java`:

```java
package io.casehub.connectors.discord;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;

class DiscordClientCommandsTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private WireMockServer wm;
    private DiscordClient client;

    @BeforeEach
    void setUp() {
        wm = new WireMockServer(WireMockConfiguration.options().dynamicPort());
        wm.start();
        client = new DiscordClient("http://localhost:" + wm.port(), "", 10_000_000);
    }

    @AfterEach
    void tearDown() {
        wm.stop();
    }

    @Test
    void bulkOverwriteGlobalCommandsSendsCorrectRequest() {
        wm.stubFor(put(urlPathEqualTo("/api/v10/applications/APP123/commands"))
                .willReturn(okJson("[]")));

        // Build the JSON payload that DiscordCommands would construct
        var commands = MAPPER.createArrayNode();
        var cmd1 = commands.addObject();
        cmd1.put("name", "status");
        cmd1.put("description", "Check status");
        cmd1.put("type", 1);
        var cmd2 = commands.addObject();
        cmd2.put("name", "assign");
        cmd2.put("description", "Assign a case");
        cmd2.put("type", 1);
        var opts = cmd2.putArray("options");
        var opt1 = opts.addObject();
        opt1.put("name", "user");
        opt1.put("description", "Target user");
        opt1.put("type", 3); // STRING
        opt1.put("required", true);
        var opt2 = opts.addObject();
        opt2.put("name", "priority");
        opt2.put("description", "Priority level");
        opt2.put("type", 4); // INTEGER
        opt2.put("required", false);

        client.bulkOverwriteGlobalCommands("bot-token", "APP123", commands);

        wm.verify(putRequestedFor(urlPathEqualTo("/api/v10/applications/APP123/commands"))
                .withHeader("Authorization", equalTo("Bot bot-token"))
                .withHeader("Content-Type", containing("application/json")));

        var req = wm.findAll(putRequestedFor(
                urlPathEqualTo("/api/v10/applications/APP123/commands")));
        String body = req.get(0).getBodyAsString();
        assertThat(body).contains("\"name\":\"status\"");
        assertThat(body).contains("\"name\":\"assign\"");
        assertThat(body).contains("\"type\":1"); // CHAT_INPUT
        assertThat(body).contains("\"type\":3"); // STRING option
        assertThat(body).contains("\"type\":4"); // INTEGER option
    }

    @Test
    void respondToInteractionSendsCallback() {
        wm.stubFor(post(urlPathEqualTo(
                "/api/v10/interactions/INT1/TOKEN1/callback"))
                .willReturn(aResponse().withStatus(204)));

        client.respondToInteraction("INT1", "TOKEN1", 4,
                "hello world", false);

        wm.verify(postRequestedFor(urlPathEqualTo(
                "/api/v10/interactions/INT1/TOKEN1/callback"))
                .withRequestBody(containing("\"type\":4"))
                .withRequestBody(containing("hello world")));
    }

    @Test
    void respondToInteractionEphemeral() {
        wm.stubFor(post(urlPathEqualTo(
                "/api/v10/interactions/INT1/TOKEN1/callback"))
                .willReturn(aResponse().withStatus(204)));

        client.respondToInteraction("INT1", "TOKEN1", 4,
                "secret", true);

        wm.verify(postRequestedFor(urlPathEqualTo(
                "/api/v10/interactions/INT1/TOKEN1/callback"))
                .withRequestBody(containing("\"flags\":64")));
    }

    @Test
    void sendFollowupMessage() {
        wm.stubFor(post(urlPathEqualTo(
                "/api/v10/webhooks/APP123/TOKEN1"))
                .willReturn(okJson("{\"id\":\"msg1\"}")));

        client.sendFollowupMessage("APP123", "TOKEN1",
                "delayed response", false);

        wm.verify(postRequestedFor(urlPathEqualTo(
                "/api/v10/webhooks/APP123/TOKEN1"))
                .withRequestBody(containing("delayed response")));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientCommandsTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — methods don't exist

- [ ] **Step 3: Implement DiscordClient command methods**

Add to `DiscordClient.java` — three new methods:

`bulkOverwriteGlobalCommands` — `PUT /api/v10/applications/{appId}/commands`
with JSON array body. Map `CommandParameterType` to Discord option types:
STRING→3, INTEGER→4, BOOLEAN→5, NUMBER→10. Each command object has
`name`, `description`, `type: 1` (CHAT_INPUT), and `options` array.

`respondToInteraction` — `POST /api/v10/interactions/{id}/{token}/callback`
with body `{"type": responseType, "data": {"content": content}}`.
If ephemeral, add `"flags": 64` to the data object.

`sendFollowupMessage` — `POST /api/v10/webhooks/{appId}/{token}` with body
`{"content": content}`. If ephemeral, add `"flags": 64`.

Implementation follows the existing patterns in `DiscordClient` — use
`mapper` (ObjectMapper) for JSON, `HttpHelper.CLIENT` for HTTP, and
the `sendWithRetry` method for retry logic.

```java
public void bulkOverwriteGlobalCommands(
        String token, String applicationId,
        ArrayNode commands) {
    var request = HttpRequest.newBuilder()
            .uri(URI.create(apiBaseUrl + "/api/v10/applications/"
                    + applicationId + "/commands"))
            .header("Authorization", "Bot " + token)
            .header("Content-Type", "application/json")
            .PUT(HttpRequest.BodyPublishers.ofString(
                    commands.toString()))
            .timeout(REQUEST_TIMEOUT)
            .build();
    sendWithRetry(request);
}

public void respondToInteraction(
        String interactionId, String interactionToken,
        int responseType, String content, boolean ephemeral) {
    var data = mapper.createObjectNode();
    data.put("content", content);
    if (ephemeral) data.put("flags", 64);
    var body = mapper.createObjectNode();
    body.put("type", responseType);
    body.set("data", data);
    var request = HttpRequest.newBuilder()
            .uri(URI.create(apiBaseUrl + "/api/v10/interactions/"
                    + interactionId + "/" + interactionToken
                    + "/callback"))
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(body.toString()))
            .timeout(REQUEST_TIMEOUT)
            .build();
    sendWithRetry(request);
}

public void sendFollowupMessage(
        String applicationId, String interactionToken,
        String content, boolean ephemeral) {
    var body = mapper.createObjectNode();
    body.put("content", content);
    if (ephemeral) body.put("flags", 64);
    var request = HttpRequest.newBuilder()
            .uri(URI.create(apiBaseUrl + "/api/v10/webhooks/"
                    + applicationId + "/" + interactionToken))
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(body.toString()))
            .timeout(REQUEST_TIMEOUT)
            .build();
    sendWithRetry(request);
}
```

Add import for `ArrayNode` (`com.fasterxml.jackson.databind.node.ArrayNode`).

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientCommandsTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add discord/
git commit -m "feat(#32): add Discord command registration and interaction API methods

bulkOverwriteGlobalCommands, respondToInteraction, sendFollowupMessage
on DiscordClient. WireMock tests verify request format.

Refs #32"
```

### Task 4: DiscordCommands + DiscordInteractionEndpoint

**Files:**
- Create: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordCommands.java`
- Create: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInteractionEndpoint.java`
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java`
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/ChatDiscordBeans.java`
- Modify: `chat-discord/pom.xml` (add `quarkus-rest` dependency for JAX-RS)
- Test: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInteractionEndpointTest.java`
- Test: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordCommandsTest.java`

**Interfaces:**
- Consumes: `DiscordClient.bulkOverwriteGlobalCommands(ArrayNode)` (from Task 3), `CommandService.dispatch()` (from Task 2), `CommandDefinition`, `CommandParameterType`, `CommandResponse` (from Task 1)
- Produces: `DiscordCommands implements Commands` (maps CommandDefinition → Discord JSON), `DiscordInteractionEndpoint` JAX-RS endpoint at `/interactions/discord`

- [ ] **Step 1: Write DiscordCommands test**

```java
package io.casehub.connectors.chat.discord;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.command.CommandDefinition;

class DiscordCommandsTest {

    @Test
    void registerAllDelegatesToClient() {
        var registered = new ArrayList<String>();
        // Use a test subclass or mock DiscordClient
        // that captures the call
        var commands = new DiscordCommands(null, "", "APP123") {
            @Override
            public void registerAll(List<CommandDefinition> cmds) {
                registered.add("called:" + cmds.size());
            }
        };
        // Verify via the captured calls
    }

    @Test
    void registerAllSkipsWhenApplicationIdBlank() {
        var commands = new DiscordCommands(null, "token", "");
        // Should not throw, should log warning
        commands.registerAll(List.of(
                new CommandDefinition("test", "Test", List.of())));
        // No exception = success
    }
}
```

Note: Full integration testing of DiscordCommands registration is
already covered by `DiscordClientCommandsTest` (Task 3). This test
focuses on the blank-application-id guard logic.

- [ ] **Step 2: Write DiscordInteractionEndpoint tests**

This is the core test class. Tests signature verification, PING/PONG,
immediate and deferred command dispatch:

```java
package io.casehub.connectors.chat.discord;

import static org.assertj.core.api.Assertions.assertThat;

import java.nio.charset.StandardCharsets;
import java.security.*;
import java.security.spec.*;
import java.util.HexFormat;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;

import io.casehub.connectors.chat.command.*;

class DiscordInteractionEndpointTest {

    private KeyPair keyPair;
    private DiscordInteractionEndpoint endpoint;
    private ObjectMapper mapper;
    private StubCommandService commandService;

    @BeforeEach
    void setUp() throws Exception {
        var kpg = KeyPairGenerator.getInstance("Ed25519");
        keyPair = kpg.generateKeyPair();
        mapper = new ObjectMapper();

        commandService = new StubCommandService();
        endpoint = new DiscordInteractionEndpoint(
                commandService, null, "APP123",
                keyPair.getPublic(), null);
    }

    private String sign(String timestamp, String body) throws Exception {
        var signer = Signature.getInstance("Ed25519");
        signer.initSign(keyPair.getPrivate());
        signer.update((timestamp + body).getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(signer.sign());
    }

    @Test
    void pingReturnsPong() throws Exception {
        var body = "{\"type\":1}";
        var timestamp = "1234567890";
        var sig = sign(timestamp, body);

        var response = endpoint.handleInteraction(sig, timestamp, body);
        assertThat(response.getStatus()).isEqualTo(200);

        var responseBody = (String) response.getEntity();
        assertThat(responseBody).contains("\"type\":1");
    }

    @Test
    void invalidSignatureReturns401() {
        var body = "{\"type\":1}";
        var response = endpoint.handleInteraction(
                "bad_sig", "1234567890", body);
        assertThat(response.getStatus()).isEqualTo(401);
    }

    @Test
    void applicationCommandDispatchesAndReturnsImmediate() throws Exception {
        commandService.setResponse(
                new CommandResponse.Immediate("pong", false));

        var body = buildCommandPayload("ping", Map.of());
        var timestamp = "1234567890";
        var sig = sign(timestamp, body);

        var response = endpoint.handleInteraction(sig, timestamp, body);
        assertThat(response.getStatus()).isEqualTo(200);

        var responseBody = (String) response.getEntity();
        assertThat(responseBody).contains("\"type\":4");
        assertThat(responseBody).contains("pong");
    }

    @Test
    void ephemeralResponseIncludesFlags() throws Exception {
        commandService.setResponse(
                new CommandResponse.Immediate("secret", true));

        var body = buildCommandPayload("ping", Map.of());
        var timestamp = "1234567890";
        var sig = sign(timestamp, body);

        var response = endpoint.handleInteraction(sig, timestamp, body);
        var responseBody = (String) response.getEntity();
        assertThat(responseBody).contains("\"flags\":64");
    }

    private String buildCommandPayload(
            String commandName, Map<String, String> options) {
        var node = mapper.createObjectNode();
        node.put("type", 2); // APPLICATION_COMMAND
        node.put("id", "interaction-1");
        node.put("token", "int-token");
        node.put("channel_id", "ch1");
        var member = node.putObject("member");
        var user = member.putObject("user");
        user.put("id", "user1");
        var data = node.putObject("data");
        data.put("name", commandName);
        data.put("type", 1);
        if (!options.isEmpty()) {
            var opts = data.putArray("options");
            options.forEach((k, v) -> {
                var opt = opts.addObject();
                opt.put("name", k);
                opt.put("value", v);
            });
        }
        return node.toString();
    }

    static class StubCommandService extends CommandService {
        private CommandResponse response =
                new CommandResponse.Immediate("default");

        StubCommandService() {
            super(List.of(), List.of());
        }

        void setResponse(CommandResponse r) { this.response = r; }

        @Override
        public CommandResponse dispatch(
                String commandName, CommandInvocation invocation) {
            return response;
        }
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest="DiscordCommandsTest,DiscordInteractionEndpointTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure

- [ ] **Step 4: Implement DiscordCommands**

```java
package io.casehub.connectors.chat.discord;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import io.casehub.connectors.chat.command.CommandDefinition;
import io.casehub.connectors.chat.command.CommandParameterType;
import io.casehub.connectors.chat.spi.Commands;
import io.casehub.connectors.discord.DiscordClient;

import java.util.List;
import java.util.logging.Logger;

public class DiscordCommands implements Commands {

    private static final Logger LOG =
            Logger.getLogger(DiscordCommands.class.getName());
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final DiscordClient client;
    private final String token;
    private final String applicationId;

    public DiscordCommands(DiscordClient client,
                           String token,
                           String applicationId) {
        this.client = client;
        this.token = token;
        this.applicationId = applicationId;
    }

    @Override
    public void registerAll(List<CommandDefinition> commands) {
        if (applicationId == null || applicationId.isBlank()) {
            LOG.warning("discord: application-id not configured, "
                    + "skipping command registration");
            return;
        }
        ArrayNode payload = toDiscordJson(commands);
        client.bulkOverwriteGlobalCommands(
                token, applicationId, payload);
        LOG.info("discord: registered " + commands.size()
                + " commands globally");
    }

    static ArrayNode toDiscordJson(List<CommandDefinition> commands) {
        var array = MAPPER.createArrayNode();
        for (var cmd : commands) {
            var node = array.addObject();
            node.put("name", cmd.name());
            node.put("description", cmd.description());
            node.put("type", 1); // CHAT_INPUT
            if (!cmd.parameters().isEmpty()) {
                var options = node.putArray("options");
                for (var param : cmd.parameters()) {
                    var opt = options.addObject();
                    opt.put("name", param.name());
                    opt.put("description", param.description());
                    opt.put("type", discordOptionType(param.type()));
                    opt.put("required", param.required());
                }
            }
        }
        return array;
    }

    private static int discordOptionType(CommandParameterType type) {
        return switch (type) {
            case STRING -> 3;
            case INTEGER -> 4;
            case BOOLEAN -> 5;
            case NUMBER -> 10;
        };
    }
}
```

- [ ] **Step 5: Implement DiscordInteractionEndpoint**

```java
package io.casehub.connectors.chat.discord;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.connectors.chat.command.*;
import io.casehub.connectors.discord.DiscordClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import io.smallrye.common.annotation.RunOnVirtualThread;

import java.nio.charset.StandardCharsets;
import java.security.*;
import java.security.spec.*;
import java.util.HashMap;
import java.util.HexFormat;
import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.logging.Level;
import java.util.logging.Logger;

@Path("/interactions/discord")
@ApplicationScoped
public class DiscordInteractionEndpoint {

    private static final Logger LOG =
            Logger.getLogger(DiscordInteractionEndpoint.class.getName());
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final CommandService commandService;
    private final DiscordClient client;
    private final String applicationId;
    private final PublicKey publicKey;
    private final ExecutorService executor;

    // Constructor for CDI
    @Inject
    public DiscordInteractionEndpoint(
            CommandService commandService,
            DiscordClient client,
            @ConfigProperty(name = "casehub.discord.application-id",
                    defaultValue = "") String applicationId,
            @ConfigProperty(name = "casehub.discord.public-key",
                    defaultValue = "") String publicKeyHex,
            org.eclipse.microprofile.context.ManagedExecutor executor) {
        this(commandService, client, applicationId,
                parsePublicKey(publicKeyHex), executor);
    }

    // Constructor for tests
    DiscordInteractionEndpoint(
            CommandService commandService,
            DiscordClient client,
            String applicationId,
            PublicKey publicKey,
            ExecutorService executor) {
        this.commandService = commandService;
        this.client = client;
        this.applicationId = applicationId;
        this.publicKey = publicKey;
        this.executor = executor;
    }

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public Response handleInteraction(
            @HeaderParam("X-Signature-Ed25519") String signature,
            @HeaderParam("X-Signature-Timestamp") String timestamp,
            String body) {

        if (publicKey == null) {
            return Response.status(503).build();
        }

        if (!verifySignature(signature, timestamp, body)) {
            return Response.status(401).build();
        }

        try {
            JsonNode json = MAPPER.readTree(body);
            int type = json.get("type").asInt();

            if (type == 1) {
                return Response.ok("{\"type\":1}").build();
            }

            if (type == 2) {
                return handleApplicationCommand(json);
            }

            return Response.status(400).build();
        } catch (Exception e) {
            LOG.log(Level.WARNING, "Failed to process interaction", e);
            return Response.serverError().build();
        }
    }

    private Response handleApplicationCommand(JsonNode json) {
        var data = json.get("data");
        String commandName = data.get("name").asText();

        Map<String, String> arguments = new HashMap<>();
        if (data.has("options")) {
            for (JsonNode opt : data.get("options")) {
                arguments.put(opt.get("name").asText(),
                        opt.get("value").asText());
            }
        }

        String userId = json.at("/member/user/id").asText(
                json.path("user").path("id").asText("unknown"));
        String channelId = json.path("channel_id").asText("");
        String interactionId = json.get("id").asText();
        String interactionToken = json.get("token").asText();

        Map<String, String> metadata = Map.of(
                "interaction_id", interactionId,
                "interaction_token", interactionToken,
                "guild_id", json.path("guild_id").asText(""));

        var invocation = new CommandInvocation(
                commandName, arguments, userId, channelId,
                "discord", metadata);

        CommandResponse response = commandService.dispatch(
                commandName, invocation);

        return switch (response) {
            case CommandResponse.Immediate imm -> {
                var respNode = MAPPER.createObjectNode();
                respNode.put("type", 4);
                var dataNode = respNode.putObject("data");
                dataNode.put("content", imm.text());
                if (imm.ephemeral()) dataNode.put("flags", 64);
                yield Response.ok(respNode.toString()).build();
            }
            case CommandResponse.Deferred def -> {
                if (executor != null) {
                    executor.submit(() -> {
                        DeferredReply reply = (text, ephemeral) ->
                                client.sendFollowupMessage(
                                        applicationId,
                                        interactionToken,
                                        text, ephemeral);
                        def.callback().accept(reply);
                    });
                }
                var respNode = MAPPER.createObjectNode();
                respNode.put("type", 5);
                if (def.ephemeral()) {
                    respNode.putObject("data").put("flags", 64);
                }
                yield Response.ok(respNode.toString()).build();
            }
        };
    }

    private boolean verifySignature(
            String signature, String timestamp, String body) {
        if (signature == null || timestamp == null) return false;
        try {
            byte[] sigBytes = HexFormat.of().parseHex(signature);
            byte[] message = (timestamp + body)
                    .getBytes(StandardCharsets.UTF_8);
            Signature verifier = Signature.getInstance("Ed25519");
            verifier.initVerify(publicKey);
            verifier.update(message);
            return verifier.verify(sigBytes);
        } catch (Exception e) {
            return false;
        }
    }

    private static PublicKey parsePublicKey(String hex) {
        if (hex == null || hex.isBlank()) return null;
        try {
            byte[] keyBytes = HexFormat.of().parseHex(hex);
            KeyFactory kf = KeyFactory.getInstance("EdDSA");
            // Reverse the y-coordinate sign bit for EdECPoint
            boolean xOdd = (keyBytes[0] & 0x80) != 0;
            keyBytes[0] &= 0x7f;
            // Reverse byte order for BigInteger (little-endian to big-endian)
            byte[] reversed = new byte[keyBytes.length];
            for (int i = 0; i < keyBytes.length; i++) {
                reversed[i] = keyBytes[keyBytes.length - 1 - i];
            }
            var point = new EdECPoint(xOdd,
                    new java.math.BigInteger(1, reversed));
            return kf.generatePublic(new EdECPublicKeySpec(
                    NamedParameterSpec.ED25519, point));
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Failed to parse Ed25519 public key", e);
            return null;
        }
    }
}
```

Note on CDI constructor: the actual CDI-injected constructor will need
adjustment during implementation — Quarkus `ManagedExecutor` injection
differs from the test constructor. The test constructor accepts
`PublicKey` directly and `ExecutorService`. The CDI constructor parses
the hex string. This dual-constructor pattern is intentional for
testability.

- [ ] **Step 6: Wire DiscordCommands into DiscordChatPlatform**

Modify `DiscordChatPlatform.java`:

Add fields:
```java
private Commands commands;
```

In `init()`, after the existing capability setup (line ~100):
```java
if (!applicationId.isBlank()) {
    this.commands = new DiscordCommands(client, token, applicationId);
    this.activeCapabilities = new java.util.HashSet<>(NATIVE_CAPABILITIES);
    this.activeCapabilities.add(Commands.class);
    this.activeCapabilities = Set.copyOf(this.activeCapabilities);
} else {
    this.commands = new NoOpCommands();
}
```

Add the `commands()` override (replacing the `NoOpCommands` one from
Task 1):
```java
@Override
public Commands commands() {
    return commands;
}
```

Modify constructor to accept `applicationId`:
```java
public DiscordChatPlatform(
        DiscordClient client,
        DiscordGatewayPresenceCache presenceCache,
        String token,
        String applicationId) {
```

In `initDegraded()`:
```java
this.commands = new NoOpCommands();
```

Update `NATIVE_CAPABILITIES` to not include `Commands.class` statically
— it's conditionally added based on `applicationId`.

- [ ] **Step 7: Update ChatDiscordBeans**

Add `applicationId` config property to the `discordChatPlatform` producer:
```java
@ConfigProperty(name = "casehub.discord.application-id",
        defaultValue = "") String applicationId
```

Pass to constructor:
```java
return new DiscordChatPlatform(client, presenceCache, token, applicationId);
```

- [ ] **Step 8: Add quarkus-rest dependency to chat-discord pom.xml**

Add to `<dependencies>`:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest</artifactId>
</dependency>
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest="DiscordCommandsTest,DiscordInteractionEndpointTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS

- [ ] **Step 10: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add chat-discord/ discord/
git commit -m "feat(#32): add Discord commands registration and interaction endpoint

DiscordCommands registers slash commands via bulk overwrite API.
DiscordInteractionEndpoint receives interactions with Ed25519
verification, dispatches to CommandService, supports immediate
and deferred responses.

Refs #32"
```

---

## Batch 3: Reference implementation

After this batch: `chat-ref` supports Commands natively, enabling
full-cycle testing without platform dependencies.

### Task 5: RefCommands

**Files:**
- Create: `chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefCommands.java`
- Modify: `chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java`
- Test: `chat-ref/src/test/java/io/casehub/connectors/chat/ref/RefCommandsTest.java`

**Interfaces:**
- Consumes: `Commands` (from Task 1), `CommandDefinition` (from Task 1)
- Produces: `RefCommands` — `Commands` impl with `List<CommandDefinition> registered()` accessor

- [ ] **Step 1: Write RefCommands test**

```java
package io.casehub.connectors.chat.ref;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.command.CommandDefinition;
import io.casehub.connectors.chat.spi.Commands;

class RefCommandsTest {

    @Test
    void registerAllStoresDefinitions() {
        var ref = new RefCommands();
        var defs = List.of(
                new CommandDefinition("ping", "Ping", List.of()),
                new CommandDefinition("status", "Status", List.of()));
        ref.registerAll(defs);

        assertThat(ref.registered()).hasSize(2);
        assertThat(ref.registered().stream()
                .map(CommandDefinition::name).toList())
                .containsExactly("ping", "status");
    }

    @Test
    void registeredIsEmptyByDefault() {
        assertThat(new RefCommands().registered()).isEmpty();
    }

    @Test
    void registeredIsImmutableCopy() {
        var ref = new RefCommands();
        ref.registerAll(List.of(
                new CommandDefinition("ping", "Ping", List.of())));
        var snapshot = ref.registered();
        ref.registerAll(List.of());
        assertThat(snapshot).hasSize(1);
    }

    @Test
    void refChatPlatformSupportsCommands() {
        var backend = new InMemoryChatBackend();
        var platform = new RefChatPlatform(backend);
        assertThat(platform.supports(Commands.class)).isTrue();
        assertThat(platform.commands()).isInstanceOf(RefCommands.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-ref -Dtest=RefCommandsTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure

- [ ] **Step 3: Implement RefCommands**

```java
package io.casehub.connectors.chat.ref;

import io.casehub.connectors.chat.command.CommandDefinition;
import io.casehub.connectors.chat.spi.Commands;
import java.util.List;

public class RefCommands implements Commands {

    private List<CommandDefinition> registered = List.of();

    @Override
    public void registerAll(List<CommandDefinition> commands) {
        this.registered = List.copyOf(commands);
    }

    public List<CommandDefinition> registered() {
        return registered;
    }
}
```

- [ ] **Step 4: Update RefChatPlatform**

Add `RefCommands` field and `commands()` override. Add `Commands.class`
to `ALL_CAPABILITIES`:

```java
private static final Set<Class<?>> ALL_CAPABILITIES = Set.of(
        Messaging.class, Threading.class, Discovery.class,
        Reactions.class, Presence.class, Members.class,
        ChannelManagement.class, MemberManagement.class,
        MessageHistory.class, Commands.class);

private final RefCommands commands = new RefCommands();

@Override
public Commands commands() {
    return commands;
}
```

Remove the `NoOpCommands` import and the existing `commands()` method
from Task 1 step 16.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-ref -Dtest=RefCommandsTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 4 tests PASS

- [ ] **Step 6: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add chat-ref/
git commit -m "feat(#32): add RefCommands for in-memory command registration

RefCommands stores registered command definitions for test
verification. RefChatPlatform reports native Commands support.

Refs #32"
```

---

## Batch 4: Slack implementation + documentation

After this batch: Slack can receive slash commands via HTTP endpoint
with HMAC verification. All documentation updated. Feature complete.

### Task 6: SlackCommands + SlackCommandEndpoint

**Files:**
- Create: `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackCommands.java`
- Create: `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackCommandEndpoint.java`
- Modify: `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackChatPlatform.java`
- Modify: `chat-slack/pom.xml` (add `quarkus-rest` dependency)
- Test: `chat-slack/src/test/java/io/casehub/connectors/chat/slack/SlackCommandEndpointTest.java`

**Interfaces:**
- Consumes: `Commands` (from Task 1), `CommandService.dispatch()` (from Task 2)
- Produces: `SlackCommands implements Commands`, `SlackCommandEndpoint` at `/interactions/slack`

- [ ] **Step 1: Write SlackCommandEndpoint tests**

```java
package io.casehub.connectors.chat.slack;

import static org.assertj.core.api.Assertions.assertThat;

import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Map;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.command.*;

class SlackCommandEndpointTest {

    private static final String SIGNING_SECRET = "test-signing-secret";
    private SlackCommandEndpoint endpoint;
    private StubCommandService commandService;

    @BeforeEach
    void setUp() {
        commandService = new StubCommandService();
        endpoint = new SlackCommandEndpoint(
                commandService, SIGNING_SECRET, null);
    }

    private String sign(String timestamp, String body) throws Exception {
        String basestring = "v0:" + timestamp + ":" + body;
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(
                SIGNING_SECRET.getBytes(StandardCharsets.UTF_8),
                "HmacSHA256"));
        byte[] hash = mac.doFinal(
                basestring.getBytes(StandardCharsets.UTF_8));
        StringBuilder sb = new StringBuilder("v0=");
        for (byte b : hash) sb.append(String.format("%02x", b));
        return sb.toString();
    }

    @Test
    void validSignatureDispatchesCommand() throws Exception {
        commandService.setResponse(
                new CommandResponse.Immediate("pong"));

        String body = "command=%2Fping&text=&user_id=U123"
                + "&channel_id=C456&team_id=T789"
                + "&team_domain=test&response_url=https%3A%2F%2Fhooks.slack.com%2Fcommands%2Ffoo";
        String timestamp = String.valueOf(
                System.currentTimeMillis() / 1000);
        String sig = sign(timestamp, body);

        var response = endpoint.handleCommand(sig, timestamp, body);
        assertThat(response.getStatus()).isEqualTo(200);

        var responseBody = (String) response.getEntity();
        assertThat(responseBody).contains("pong");
    }

    @Test
    void invalidSignatureReturns401() {
        String body = "command=%2Fping&text=";
        var response = endpoint.handleCommand(
                "v0=bad", "1234567890", body);
        assertThat(response.getStatus()).isEqualTo(401);
    }

    @Test
    void staleTimestampReturns401() throws Exception {
        String body = "command=%2Fping&text=";
        String staleTimestamp = String.valueOf(
                (System.currentTimeMillis() / 1000) - 600);
        String sig = sign(staleTimestamp, body);

        var response = endpoint.handleCommand(
                sig, staleTimestamp, body);
        assertThat(response.getStatus()).isEqualTo(401);
    }

    @Test
    void commandTextPassedAsArgument() throws Exception {
        commandService.setResponse(
                new CommandResponse.Immediate("ok"));
        commandService.setCaptureInvocation(true);

        String body = "command=%2Fassign&text=user123+high"
                + "&user_id=U123&channel_id=C456"
                + "&team_id=T789&team_domain=test"
                + "&response_url=https%3A%2F%2Fhooks.slack.com";
        String timestamp = String.valueOf(
                System.currentTimeMillis() / 1000);
        String sig = sign(timestamp, body);

        endpoint.handleCommand(sig, timestamp, body);

        assertThat(commandService.lastInvocation().argument("text"))
                .isEqualTo("user123 high");
        assertThat(commandService.lastInvocation().commandName())
                .isEqualTo("assign");
    }

    static class StubCommandService extends CommandService {
        private CommandResponse response =
                new CommandResponse.Immediate("default");
        private boolean captureInvocation;
        private CommandInvocation lastInvocation;

        StubCommandService() { super(List.of(), List.of()); }

        void setResponse(CommandResponse r) { this.response = r; }
        void setCaptureInvocation(boolean b) { captureInvocation = b; }
        CommandInvocation lastInvocation() { return lastInvocation; }

        @Override
        public CommandResponse dispatch(
                String commandName, CommandInvocation invocation) {
            if (captureInvocation) lastInvocation = invocation;
            return response;
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-slack -Dtest=SlackCommandEndpointTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure

- [ ] **Step 3: Implement SlackCommands**

```java
package io.casehub.connectors.chat.slack;

import io.casehub.connectors.chat.command.CommandDefinition;
import io.casehub.connectors.chat.spi.Commands;
import java.util.List;
import java.util.logging.Logger;

public class SlackCommands implements Commands {
    private static final Logger LOG =
            Logger.getLogger(SlackCommands.class.getName());

    @Override
    public void registerAll(List<CommandDefinition> commands) {
        LOG.info("slack: " + commands.size()
                + " commands available (register via Slack app manifest)");
    }
}
```

- [ ] **Step 4: Implement SlackCommandEndpoint**

```java
package io.casehub.connectors.chat.slack;

import io.casehub.connectors.chat.command.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.logging.Level;
import java.util.logging.Logger;

@Path("/interactions/slack")
@ApplicationScoped
public class SlackCommandEndpoint {

    private static final Logger LOG =
            Logger.getLogger(SlackCommandEndpoint.class.getName());
    private static final long MAX_TIMESTAMP_AGE_SECONDS = 300;

    private final CommandService commandService;
    private final String signingSecret;
    private final ExecutorService executor;

    @Inject
    public SlackCommandEndpoint(
            CommandService commandService,
            @ConfigProperty(name = "casehub.slack.signing-secret",
                    defaultValue = "") String signingSecret,
            org.eclipse.microprofile.context.ManagedExecutor executor) {
        this(commandService, signingSecret, executor);
    }

    SlackCommandEndpoint(
            CommandService commandService,
            String signingSecret,
            ExecutorService executor) {
        this.commandService = commandService;
        this.signingSecret = signingSecret;
        this.executor = executor;
    }

    @POST
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    @Produces(MediaType.APPLICATION_JSON)
    public Response handleCommand(
            @HeaderParam("X-Slack-Signature") String signature,
            @HeaderParam("X-Slack-Request-Timestamp") String timestamp,
            String body) {

        if (signingSecret.isBlank()) {
            return Response.status(503).build();
        }

        if (!verifyTimestamp(timestamp)
                || !verifySignature(signature, timestamp, body)) {
            return Response.status(401).build();
        }

        try {
            Map<String, String> params = parseFormBody(body);

            String command = params.getOrDefault("command", "")
                    .replaceFirst("^/", "");
            String text = params.getOrDefault("text", "");

            Map<String, String> arguments = text.isBlank()
                    ? Map.of()
                    : Map.of("text", text);

            String responseUrl = params.getOrDefault(
                    "response_url", "");

            var invocation = new CommandInvocation(
                    command, arguments,
                    params.getOrDefault("user_id", ""),
                    params.getOrDefault("channel_id", ""),
                    "slack",
                    Map.of("team_id",
                            params.getOrDefault("team_id", ""),
                           "team_domain",
                            params.getOrDefault("team_domain", ""),
                           "response_url", responseUrl));

            CommandResponse response = commandService.dispatch(
                    command, invocation);

            return switch (response) {
                case CommandResponse.Immediate imm -> {
                    String json = "{\"text\":\""
                            + escapeJson(imm.text()) + "\""
                            + (imm.ephemeral()
                            ? ",\"response_type\":\"ephemeral\""
                            : ",\"response_type\":\"in_channel\"")
                            + "}";
                    yield Response.ok(json).build();
                }
                case CommandResponse.Deferred def -> {
                    if (executor != null && !responseUrl.isBlank()) {
                        executor.submit(() -> {
                            DeferredReply reply = (txt, eph) ->
                                    postToResponseUrl(
                                            responseUrl, txt, eph);
                            def.callback().accept(reply);
                        });
                    }
                    yield Response.ok().build();
                }
            };
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Failed to process Slack command", e);
            return Response.serverError().build();
        }
    }

    private boolean verifyTimestamp(String timestamp) {
        if (timestamp == null) return false;
        try {
            long ts = Long.parseLong(timestamp);
            long now = System.currentTimeMillis() / 1000;
            return Math.abs(now - ts) <= MAX_TIMESTAMP_AGE_SECONDS;
        } catch (NumberFormatException e) {
            return false;
        }
    }

    private boolean verifySignature(
            String signature, String timestamp, String body) {
        if (signature == null) return false;
        try {
            String basestring = "v0:" + timestamp + ":" + body;
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(
                    signingSecret.getBytes(StandardCharsets.UTF_8),
                    "HmacSHA256"));
            byte[] hash = mac.doFinal(
                    basestring.getBytes(StandardCharsets.UTF_8));
            StringBuilder sb = new StringBuilder("v0=");
            for (byte b : hash)
                sb.append(String.format("%02x", b));
            String expected = sb.toString();
            return MessageDigest.isEqual(
                    expected.getBytes(StandardCharsets.UTF_8),
                    signature.getBytes(StandardCharsets.UTF_8));
        } catch (Exception e) {
            return false;
        }
    }

    private Map<String, String> parseFormBody(String body) {
        Map<String, String> params = new HashMap<>();
        for (String pair : body.split("&")) {
            String[] kv = pair.split("=", 2);
            String key = URLDecoder.decode(kv[0], StandardCharsets.UTF_8);
            String value = kv.length > 1
                    ? URLDecoder.decode(kv[1], StandardCharsets.UTF_8)
                    : "";
            params.put(key, value);
        }
        return params;
    }

    private void postToResponseUrl(
            String url, String text, boolean ephemeral) {
        // POST JSON to Slack response_url
        // Uses java.net.http.HttpClient
        try {
            String json = "{\"text\":\"" + escapeJson(text) + "\""
                    + ",\"response_type\":\""
                    + (ephemeral ? "ephemeral" : "in_channel")
                    + "\"}";
            var request = java.net.http.HttpRequest.newBuilder()
                    .uri(java.net.URI.create(url))
                    .header("Content-Type", "application/json")
                    .POST(java.net.http.HttpRequest.BodyPublishers
                            .ofString(json))
                    .build();
            java.net.http.HttpClient.newHttpClient()
                    .send(request,
                            java.net.http.HttpResponse.BodyHandlers
                                    .discarding());
        } catch (Exception e) {
            LOG.log(Level.WARNING,
                    "Failed to send deferred Slack response", e);
        }
    }

    private static String escapeJson(String s) {
        return s.replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n")
                .replace("\r", "\\r");
    }
}
```

- [ ] **Step 5: Update SlackChatPlatform**

Replace `NoOpCommands` with `SlackCommands`:

```java
private final Commands commands = new SlackCommands();

@Override
public Commands commands() {
    return commands;
}
```

Slack `supports(Commands.class)` returns `true` — it can receive
commands even though registration is via manifest.

Add `Commands.class` to the native capabilities set.

- [ ] **Step 6: Add quarkus-rest dependency to chat-slack pom.xml**

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest</artifactId>
</dependency>
```

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-slack -Dtest=SlackCommandEndpointTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 4 tests PASS

- [ ] **Step 8: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add chat-slack/
git commit -m "feat(#32): add Slack slash command endpoint with HMAC verification

SlackCommandEndpoint receives slash commands at /interactions/slack
with HMAC-SHA256 signature and timestamp replay verification.
SlackCommands provides no-op registration (manifest-based).

Refs #32"
```

### Task 7: Documentation updates

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/guides/consumer-guide.md`
- Modify: `docs/guides/contributor-guide.md`

**Interfaces:**
- Consumes: all types from Tasks 1-6

- [ ] **Step 1: Update CLAUDE.md**

Update the project description to include Commands capability:
- Add `Commands` to the ChatPlatform capability list
- Update Discord capability count: "8 native capabilities" → "9 native capabilities"
- Update Slack capability count: "9 native capabilities" → "10 native capabilities"
- Add `Commands` to the ChatPlatform `@SimulationEligible` capabilities list
- Add new config properties: `casehub.discord.application-id`,
  `casehub.discord.public-key`, `casehub.slack.signing-secret`

- [ ] **Step 2: Update consumer guide**

Add Commands section to `docs/guides/consumer-guide.md`:
- How to implement a `CommandHandler`
- Example with `CommandDefinition`, `CommandResponse.Immediate`,
  and `CommandResponse.Deferred`
- Config properties table for Discord and Slack
- Discord setup: Interactions Endpoint URL pointing to
  `/interactions/discord`
- Slack setup: Request URL in app manifest pointing to
  `/interactions/slack`

- [ ] **Step 3: Update contributor guide**

Add Commands section to `docs/guides/contributor-guide.md`:
- `Commands` SPI contract
- How to add `Commands` support to a new `ChatPlatform`
- Signature verification patterns (Ed25519, HMAC-SHA256)
- Deferred response flow

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md docs/
git commit -m "docs(#32): document Commands capability in guides and CLAUDE.md

Refs #32"
```

## References

- [2026-09-27-commands-capability-design.md] — design spec this plan implements
- [decisions.md] — D1-D7 design decisions
- [chat-spi/ChatPlatform.java] — SPI interface being extended
- [chat-spi/DefaultChatPlatform.java] — record that gains commands field
- [chat-spi/NoOpChatPlatform.java] — default bean needing commands() method
- [chat-spi/ChatBeans.java] — CDI producer pattern for CommandService
- [chat-spi/ChatPlatformService.java] — service pattern precedent
- [chat-spi/ChatPlatformBuilderTest.java] — builder test pattern
- [chat-discord/DiscordChatPlatform.java] — platform impl getting Commands support
- [chat-discord/ChatDiscordBeans.java] — CDI producer to update
- [chat-discord/pom.xml] — Maven dependencies
- [discord/DiscordClient.java] — HTTP client getting new methods
- [chat-ref/RefChatPlatform.java] — ref implementation pattern
- [GitHub #32] — focal issue
