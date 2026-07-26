# Rich Content Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a platform-agnostic `RichCard` model to `ChatContent`, implement Block Kit support for Slack, consolidate MCP tools into `send_chat`/`list_chat_channels`, and enrich the `Channel` model with member count.

**Architecture:** `RichCard` is a pure-Java record in `chat-spi` with a `Builder`. `ChatContent` gains a `List<RichCard> cards` field. Each `ChatPlatform` implementation translates `RichCard` to its native format (Discord embeds, Slack Block Kit). MCP tools consolidate from platform-specific (`send_slack_bot`, `send_discord`) to cross-platform (`send_chat`, `list_chat_channels`) routing through `ChatPlatformService`.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, jakarta.json for Block Kit serialization, WireMock for HTTP client tests, AssertJ for assertions.

**Spec:** `specs/2026-06-30-rich-content-model-design.md`

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo`
- Use `mvn` not `./mvnw`
- All commits reference an issue (`Refs #37`, `Refs #41`, `Refs #42`)
- IntelliJ MCP for all class/symbol lookups — never bash grep/find for code navigation
- `chat-spi` is Tier 2 (depends on `quarkus-arc`); `RichCard` itself has no CDI annotations
- No platform-specific limit enforcement in `RichCard` — each translator validates its own limits

---

### Task 1: RichCard Model + ChatContent Extension + Channel Enrichment

Core model types in `chat-spi`. All downstream compilation breakage is fixed in this task — every 3-arg `ChatContent` and 5-arg `Channel` call site is updated.

**Files:**
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/model/RichCard.java`
- Create: `chat-spi/src/test/java/io/casehub/connectors/chat/model/RichCardTest.java`
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/model/ChatContent.java`
- Modify: `chat-spi/src/test/java/io/casehub/connectors/chat/model/ChatContentTest.java`
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/model/Channel.java`
- Modify: 5 production files with 3-arg ChatContent (DiscordChatPlatform, DiscordInboundTranslator, IrcInboundTranslator, RefInboundTranslator, SlackInboundTranslator)
- Modify: 2 production files with 5-arg Channel (IrcChatPlatform, SlackChatPlatform)
- Modify: 2 production files with 5-arg Channel in chat-ref/chat-demo (InMemoryChatBackend, SqliteChatBackend)
- Modify: 17 test files with 3-arg ChatContent
- Modify: DiscordChatPlatform toChannel() for null memberCount

**Interfaces:**
- Produces: `RichCard(title, description, url, color, fields, thumbnailUrl, imageUrl, footer, author)` with `Builder` and `Field(name, value, inline)` nested record
- Produces: `ChatContent(text, markdown, attachments, cards)` with convenience `ChatContent(String text)` constructor
- Produces: `Channel(ref, name, topic, description, isPrivate, memberCount)` with 5-arg backward-compatible constructor

- [ ] **Step 1: Write RichCard tests**

Create `chat-spi/src/test/java/io/casehub/connectors/chat/model/RichCardTest.java`:

```java
package io.casehub.connectors.chat.model;

import java.util.List;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RichCardTest {

    @Test
    void requiresTitleOrDescription() {
        assertThatThrownBy(() -> new RichCard(null, null, null, null, null, null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("title or description");
    }

    @Test
    void titleOnlyIsValid() {
        final var card = new RichCard("title", null, null, null, null, null, null, null, null);
        assertThat(card.title()).isEqualTo("title");
        assertThat(card.description()).isNull();
        assertThat(card.fields()).isEmpty();
    }

    @Test
    void descriptionOnlyIsValid() {
        final var card = new RichCard(null, "desc", null, null, null, null, null, null, null);
        assertThat(card.description()).isEqualTo("desc");
    }

    @Test
    void fieldsDefensiveCopy() {
        final var mutable = new java.util.ArrayList<>(List.of(
                new RichCard.Field("k", "v", false)));
        final var card = new RichCard("t", null, null, null, mutable, null, null, null, null);
        mutable.add(new RichCard.Field("k2", "v2", true));
        assertThat(card.fields()).hasSize(1);
    }

    @Test
    void nullFieldsBecomesEmptyList() {
        final var card = new RichCard("t", null, null, null, null, null, null, null, null);
        assertThat(card.fields()).isEmpty();
    }

    @Test
    void builderProducesValidCard() {
        final var card = RichCard.builder()
                .title("Deploy Summary")
                .description("3 services updated")
                .color(0x00FF00)
                .fields(List.of(new RichCard.Field("env", "prod", true)))
                .footer("footer")
                .author("bot")
                .build();

        assertThat(card.title()).isEqualTo("Deploy Summary");
        assertThat(card.description()).isEqualTo("3 services updated");
        assertThat(card.color()).isEqualTo(0x00FF00);
        assertThat(card.fields()).hasSize(1);
        assertThat(card.fields().getFirst().name()).isEqualTo("env");
        assertThat(card.footer()).isEqualTo("footer");
        assertThat(card.author()).isEqualTo("bot");
    }

    @Test
    void builderRequiresTitleOrDescription() {
        assertThatThrownBy(() -> RichCard.builder().build())
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void builderWithAllFields() {
        final var card = RichCard.builder()
                .title("t")
                .description("d")
                .url("https://example.com")
                .color(16711680)
                .thumbnailUrl("https://img/thumb.png")
                .imageUrl("https://img/full.png")
                .footer("ft")
                .author("au")
                .fields(List.of(
                        new RichCard.Field("a", "1", true),
                        new RichCard.Field("b", "2", false)))
                .build();

        assertThat(card.url()).isEqualTo("https://example.com");
        assertThat(card.thumbnailUrl()).isEqualTo("https://img/thumb.png");
        assertThat(card.imageUrl()).isEqualTo("https://img/full.png");
        assertThat(card.fields()).hasSize(2);
    }
}
```

- [ ] **Step 2: Implement RichCard**

Create `chat-spi/src/main/java/io/casehub/connectors/chat/model/RichCard.java`:

```java
package io.casehub.connectors.chat.model;

import java.util.List;

public record RichCard(
        String title,
        String description,
        String url,
        Integer color,
        List<Field> fields,
        String thumbnailUrl,
        String imageUrl,
        String footer,
        String author) {

    public record Field(String name, String value, boolean inline) {}

    public RichCard {
        if (title == null && description == null) {
            throw new IllegalArgumentException("RichCard requires at least title or description");
        }
        fields = fields == null ? List.of() : List.copyOf(fields);
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String title;
        private String description;
        private String url;
        private Integer color;
        private List<Field> fields;
        private String thumbnailUrl;
        private String imageUrl;
        private String footer;
        private String author;

        Builder() {}

        public Builder title(final String title) { this.title = title; return this; }
        public Builder description(final String description) { this.description = description; return this; }
        public Builder url(final String url) { this.url = url; return this; }
        public Builder color(final Integer color) { this.color = color; return this; }
        public Builder fields(final List<Field> fields) { this.fields = fields; return this; }
        public Builder thumbnailUrl(final String thumbnailUrl) { this.thumbnailUrl = thumbnailUrl; return this; }
        public Builder imageUrl(final String imageUrl) { this.imageUrl = imageUrl; return this; }
        public Builder footer(final String footer) { this.footer = footer; return this; }
        public Builder author(final String author) { this.author = author; return this; }

        public RichCard build() {
            return new RichCard(title, description, url, color, fields,
                    thumbnailUrl, imageUrl, footer, author);
        }
    }
}
```

- [ ] **Step 3: Update ChatContent — add cards field**

Modify `chat-spi/src/main/java/io/casehub/connectors/chat/model/ChatContent.java`:

```java
package io.casehub.connectors.chat.model;

import java.util.List;
import java.util.Objects;

import io.casehub.connectors.Attachment;

public record ChatContent(
        String text,
        String markdown,
        List<Attachment> attachments,
        List<RichCard> cards) {

    public ChatContent {
        Objects.requireNonNull(text, "text");
        attachments = attachments == null ? List.of() : List.copyOf(attachments);
        cards = cards == null ? List.of() : List.copyOf(cards);
    }

    public ChatContent(final String text) {
        this(text, null, List.of(), List.of());
    }
}
```

- [ ] **Step 4: Update ChatContentTest**

Modify `chat-spi/src/test/java/io/casehub/connectors/chat/model/ChatContentTest.java` — update the two 3-arg constructions to 4-arg, and add a test for cards:

At line 32: `new ChatContent("hello", null, null)` → `new ChatContent("hello", null, null, null)`
At line 40: `new ChatContent("hello", null, mutable)` → `new ChatContent("hello", null, mutable, null)`

Add test:
```java
@Test
void cardsDefaultsToEmptyList() {
    final var content = new ChatContent("hello");
    assertThat(content.cards()).isEmpty();
}

@Test
void cardsDefensiveCopy() {
    final var mutable = new java.util.ArrayList<>(List.of(
            RichCard.builder().title("t").build()));
    final var content = new ChatContent("hello", null, List.of(), mutable);
    mutable.add(RichCard.builder().title("t2").build());
    assertThat(content.cards()).hasSize(1);
}
```

- [ ] **Step 5: Update Channel — add memberCount**

Modify `chat-spi/src/main/java/io/casehub/connectors/chat/model/Channel.java`:

```java
package io.casehub.connectors.chat.model;

public record Channel(ChatChannelRef ref, String name, String topic,
                      String description, boolean isPrivate, Integer memberCount) {

    public Channel(final ChatChannelRef ref, final String name, final String topic,
                   final String description, final boolean isPrivate) {
        this(ref, name, topic, description, isPrivate, null);
    }
}
```

- [ ] **Step 6: Fix all 3-arg ChatContent call sites**

Update all 5 production files — add `List.of()` as 4th argument:

1. `chat-discord/src/main/java/.../DiscordChatPlatform.java:341` — `new ChatContent(dm.content(), null, attachments)` → `new ChatContent(dm.content(), null, attachments, List.of())`
2. `chat-discord/src/main/java/.../DiscordInboundTranslator.java:35` — add `, List.of())`
3. `chat-irc/src/main/java/.../IrcInboundTranslator.java:35` — add `, List.of())`
4. `chat-ref/src/main/java/.../RefInboundTranslator.java:34` — add `, List.of())`
5. `chat-slack/src/main/java/.../SlackInboundTranslator.java:35` — add `, List.of())`

Update all 17 test files with 3-arg ChatContent — same mechanical addition of `, List.of()` as 4th argument. Files:
- `DiscordChatPlatformTest.java` lines 118, 132, 151, 171, 464
- `IrcChatPlatformTest.java` lines 74, 87, 124
- `SlackChatPlatformTest.java` lines 82, 108, 413
- `ChatContentTest.java` lines 32, 40

The 1-arg `new ChatContent("text")` call sites (27 total) need no change — the convenience constructor handles them.

- [ ] **Step 7: Run tests to verify compilation and backward compatibility**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo`
Expected: BUILD SUCCESS — all existing tests pass, new RichCard tests pass.

- [ ] **Step 8: Commit**

```
feat(chat-spi): add RichCard model, extend ChatContent with cards, add Channel memberCount — Refs #37, #42
```

---

### Task 2: SlackBotClient Block Kit Support + ConversationInfo Member Count

Extend `SlackBotClient` with a `blocks` parameter on `postMessage` and parse `num_members` in `listConversations`. Pure HTTP client changes — no SPI involvement.

**Files:**
- Modify: `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java`
- Modify: `slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotClientTest.java`

**Interfaces:**
- Consumes: nothing from Task 1
- Produces: `postMessage(token, channelId, text, threadTs, blocksJson)` overload; `ConversationInfo(id, name, topic, purpose, isPrivate, numMembers)` with `numMembers` field

- [ ] **Step 1: Write test for postMessage with blocks**

In `SlackBotClientTest.java`, add a test that sends a message with blocks JSON and verifies the HTTP payload includes the `blocks` field:

```java
@Test
void postMessageWithBlocks() {
    wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
            .willReturn(okJson("{\"ok\":true,\"ts\":\"1234567890.123456\"}")));

    final String blocksJson = "[{\"type\":\"header\",\"text\":{\"type\":\"plain_text\",\"text\":\"Title\"}}]";
    final var result = client.postMessage("xoxb-test", "C123", "fallback", null, blocksJson);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/chat.postMessage"))
            .withRequestBody(matchingJsonPath("$.blocks")));
}
```

- [ ] **Step 2: Write test for postMessage without blocks delegates correctly**

```java
@Test
void postMessageWithoutBlocksDelegates() {
    wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
            .willReturn(okJson("{\"ok\":true,\"ts\":\"1234567890.123456\"}")));

    final var result = client.postMessage("xoxb-test", "C123", "hello", null);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/chat.postMessage"))
            .withRequestBody(not(matchingJsonPath("$.blocks"))));
}
```

- [ ] **Step 3: Write test for ConversationInfo numMembers parsing**

```java
@Test
void listConversationsParsesMemberCount() {
    wireMock.stubFor(get(urlPathEqualTo("/api/conversations.list"))
            .willReturn(okJson("{\"ok\":true,\"channels\":[{\"id\":\"C1\",\"name\":\"general\","
                    + "\"topic\":{\"value\":\"t\"},\"purpose\":{\"value\":\"p\"},"
                    + "\"is_private\":false,\"num_members\":42}],"
                    + "\"response_metadata\":{\"next_cursor\":\"\"}}")));

    final var result = client.listConversations("xoxb-test");

    assertThat(result).hasSize(1);
    assertThat(result.getFirst().numMembers()).isEqualTo(42);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot`
Expected: compilation errors — `postMessage` has no 5-arg overload, `ConversationInfo` has no `numMembers` field.

- [ ] **Step 5: Implement postMessage with blocks**

In `SlackBotClient.java`, add the 5-arg overload and update `buildPayload`:

Change the existing `postMessage` to delegate:
```java
public PostResult postMessage(final String token, final String channelId,
                              final String text, final String threadTs) {
    return postMessage(token, channelId, text, threadTs, null);
}

public PostResult postMessage(final String token, final String channelId,
                              final String text, final String threadTs,
                              final String blocksJson) {
    final String json = buildPayload(channelId, text, threadTs, blocksJson);
    final HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(apiBaseUrl + API_PATH))
            .header("Authorization", "Bearer " + token)
            .header("Content-Type", "application/json")
            .timeout(REQUEST_TIMEOUT)
            .POST(HttpRequest.BodyPublishers.ofString(json))
            .build();
    return sendWithRetry(request);
}
```

Update `buildPayload` to accept and include blocks:
```java
private static String buildPayload(final String channelId, final String text,
                                   final String threadTs, final String blocksJson) {
    final JsonObjectBuilder builder = Json.createObjectBuilder()
            .add("channel", channelId)
            .add("text", text);
    if (threadTs != null) {
        builder.add("thread_ts", threadTs);
    }
    if (blocksJson != null) {
        builder.add("blocks", Json.createReader(new StringReader(blocksJson)).readArray());
    }
    return builder.build().toString();
}
```

Add import: `import java.io.StringReader;`

- [ ] **Step 6: Update ConversationInfo and parsing**

Change the `ConversationInfo` record:
```java
public record ConversationInfo(String id, String name, String topic,
                               String purpose, boolean isPrivate, Integer numMembers) {}
```

In `parseConversationPage`, add `num_members` parsing before the `return new ConversationInfo(...)`:
```java
final Integer numMembers = ch.containsKey("num_members")
        ? ch.getInt("num_members") : null;
return new ConversationInfo(id, name, topic, purpose, isPrivate, numMembers);
```

- [ ] **Step 7: Fix all ConversationInfo construction sites**

Search for all places that construct `ConversationInfo` — the `parseConversationPage` method is the only place (verified in the exploration). But `ConversationResult` uses `ConversationInfo`, and `getConversationInfo`/`createConversation` also construct `ConversationInfo` via `parseConversationResult`. Check `parseConversationResult` — it constructs `ConversationInfo` from a single channel object. Update it too to parse `num_members`.

In `parseConversationResult` (around line 634), update the ConversationInfo construction:
```java
final Integer numMembers = channel.containsKey("num_members")
        ? channel.getInt("num_members") : null;
return new ConversationResult(true,
        new ConversationInfo(id, name, topic, purpose, isPrivate, numMembers), null);
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot`
Expected: All tests pass including the 3 new tests.

- [ ] **Step 9: Commit**

```
feat(slack-bot): add Block Kit blocks parameter to postMessage, parse num_members in conversations — Refs #41, #42
```

---

### Task 3: DiscordChatPlatform RichCard Translation + Member Count

Teach `DiscordChatPlatform` to translate `RichCard` objects from `ChatContent.cards()` into `DiscordEmbed` objects, enforce Discord-specific limits, and provide member count in channel discovery.

**Files:**
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java`
- Modify: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java`

**Interfaces:**
- Consumes: `RichCard` and `ChatContent.cards()` from Task 1; `DiscordEmbed` and `DiscordClient.sendMessage(token, channelId, content, embeds)` already exist
- Produces: `Messaging.send()` and `Threading.reply()` that render `RichCard` → `DiscordEmbed`; `Discovery.listChannels()` returning `Channel` with `memberCount`

- [ ] **Step 1: Write test for sending message with RichCard**

In `DiscordChatPlatformTest.java`:

```java
@Test
void sendMessageWithRichCard() {
    wireMock.stubFor(post(urlEqualTo("/api/v10/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    final var card = RichCard.builder()
            .title("Deploy")
            .description("3 services")
            .color(0x00FF00)
            .fields(List.of(new RichCard.Field("env", "prod", true)))
            .footer("bot v2")
            .author("CI")
            .build();

    final var content = new ChatContent("fallback", null, List.of(), List.of(card));
    final var result = platform.messaging().send(new ChatChannelRef("ch1"), content);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/v10/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.embeds[0].title", equalTo("Deploy")))
            .withRequestBody(matchingJsonPath("$.embeds[0].description", equalTo("3 services")))
            .withRequestBody(matchingJsonPath("$.embeds[0].color", equalTo("65280")))
            .withRequestBody(matchingJsonPath("$.embeds[0].footer.text", equalTo("bot v2")))
            .withRequestBody(matchingJsonPath("$.embeds[0].author.name", equalTo("CI")))
            .withRequestBody(matchingJsonPath("$.embeds[0].fields[0].name", equalTo("env"))));
}
```

- [ ] **Step 2: Write test for Discord title length limit**

```java
@Test
void sendMessageRejectsOversizedTitle() {
    final var card = RichCard.builder()
            .title("x".repeat(257))
            .build();
    final var content = new ChatContent("fallback", null, List.of(), List.of(card));
    final var result = platform.messaging().send(new ChatChannelRef("ch1"), content);

    assertThat(result.ok()).isFalse();
    assertThat(result.error()).contains("256");
}
```

- [ ] **Step 3: Write test for text-only message (no cards) is unchanged**

```java
@Test
void sendMessageWithoutCardsUsesTextOnly() {
    wireMock.stubFor(post(urlEqualTo("/api/v10/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    final var content = new ChatContent("just text");
    final var result = platform.messaging().send(new ChatChannelRef("ch1"), content);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/v10/channels/ch1/messages"))
            .withRequestBody(not(matchingJsonPath("$.embeds"))));
}
```

- [ ] **Step 4: Write test for reply with RichCard**

```java
@Test
void replyWithRichCard() {
    wireMock.stubFor(post(urlEqualTo("/api/v10/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg2\",\"channel_id\":\"ch1\"}")));

    final var card = RichCard.builder().title("Update").build();
    final var content = new ChatContent("fallback", null, List.of(), List.of(card));
    final var parent = new ChatMessageRef(new ChatChannelRef("ch1"), "msg1");
    final var result = platform.threading().reply(parent, content);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/v10/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.embeds[0].title", equalTo("Update")))
            .withRequestBody(matchingJsonPath("$.message_reference.message_id", equalTo("msg1"))));
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord`
Expected: Tests fail — sendMessage doesn't pass embeds from ChatContent.

- [ ] **Step 6: Implement RichCard → DiscordEmbed translation**

In `DiscordChatPlatform.java`, add a private translation method:

```java
private List<DiscordEmbed> toEmbeds(final List<RichCard> cards) {
    return cards.stream().map(this::toEmbed).toList();
}

private DiscordEmbed toEmbed(final RichCard card) {
    final List<DiscordEmbed.Field> fields = card.fields().stream()
            .map(f -> new DiscordEmbed.Field(f.name(), f.value(), f.inline()))
            .toList();
    final DiscordEmbed.Footer footer = card.footer() != null
            ? new DiscordEmbed.Footer(card.footer()) : null;
    final DiscordEmbed.Author author = card.author() != null
            ? new DiscordEmbed.Author(card.author()) : null;
    return new DiscordEmbed(card.title(), card.description(), card.url(), card.color(),
            fields, card.thumbnailUrl(), card.imageUrl(), footer, author);
}
```

Add a validation method:

```java
private SendResult validateEmbeds(final List<RichCard> cards) {
    if (cards.size() > 10) {
        return SendResult.failure("Discord allows at most 10 embeds per message");
    }
    long totalChars = 0;
    for (final RichCard card : cards) {
        if (card.title() != null && card.title().length() > 256)
            return SendResult.failure("Embed title exceeds 256 characters");
        if (card.description() != null && card.description().length() > 4096)
            return SendResult.failure("Embed description exceeds 4096 characters");
        if (card.footer() != null && card.footer().length() > 2048)
            return SendResult.failure("Embed footer exceeds 2048 characters");
        if (card.author() != null && card.author().length() > 256)
            return SendResult.failure("Embed author exceeds 256 characters");
        if (card.fields().size() > 25)
            return SendResult.failure("Embed exceeds 25 fields");
        if (card.url() != null && card.title() == null)
            return SendResult.failure("Embed url requires title");
        totalChars += (card.title() != null ? card.title().length() : 0)
                + (card.description() != null ? card.description().length() : 0)
                + (card.footer() != null ? card.footer().length() : 0)
                + (card.author() != null ? card.author().length() : 0);
        for (final RichCard.Field f : card.fields()) {
            totalChars += f.name().length() + f.value().length();
        }
    }
    if (totalChars > 6000) {
        return SendResult.failure("Total embed content exceeds 6000 characters");
    }
    return null;
}
```

Update `sendMessage` and `sendReply`:

```java
private SendResult sendMessage(final ChatChannelRef channel, final ChatContent content) {
    final String messageContent = extractContent(content);
    final List<DiscordEmbed> embeds;
    if (!content.cards().isEmpty()) {
        final SendResult validation = validateEmbeds(content.cards());
        if (validation != null) return validation;
        embeds = toEmbeds(content.cards());
    } else {
        embeds = List.of();
        if (messageContent.length() > MAX_CONTENT_LENGTH) {
            return SendResult.failure("Content exceeds Discord's 2000-character limit");
        }
    }

    final PostResult result = embeds.isEmpty()
            ? client.sendMessage(token, channel.id(), messageContent)
            : client.sendMessage(token, channel.id(), messageContent, embeds);

    if (!result.ok()) {
        return SendResult.failure(result.error());
    }
    return SendResult.success(new ChatMessageRef(channel, result.messageId()), Instant.now());
}
```

Apply the same pattern to `sendReply`.

- [ ] **Step 7: Add member count to discovery**

In `DiscordChatPlatform`, update `listChannels()` to fetch the guild member count:

```java
private List<Channel> listChannels() {
    final List<DiscordChannel> channels = client.listGuildChannels(token);
    final DiscordGuild guild = client.getGuild(token, true);
    final Integer memberCount = guild != null ? guild.approximateMemberCount() : null;

    return channels.stream()
            .filter(ch -> isTextChannel(ch.type()))
            .map(ch -> toChannel(ch, memberCount))
            .toList();
}
```

Update `toChannel` to accept memberCount:
```java
private Channel toChannel(final DiscordChannel dc, final Integer memberCount) {
    return new Channel(
            new ChatChannelRef(dc.id()),
            dc.name(),
            dc.topic(),
            null,
            isPrivateChannel(dc),
            memberCount);
}
```

Note: Discord member count is guild-level (not per-channel), so all channels get the same count. This is accurate for single-guild bots.

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```
feat(chat-discord): translate RichCard to DiscordEmbed with limit enforcement, add member count to discovery — Refs #37, #42
```

---

### Task 4: SlackChatPlatform RichCard → Block Kit Translation + Member Count

Teach `SlackChatPlatform` to translate `RichCard` objects into Slack Block Kit JSON, pass it through `SlackBotClient.postMessage`, and provide member count in channel discovery.

**Files:**
- Modify: `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackChatPlatform.java`
- Modify: `chat-slack/src/test/java/io/casehub/connectors/chat/slack/SlackChatPlatformTest.java`

**Interfaces:**
- Consumes: `RichCard` from Task 1; `SlackBotClient.postMessage(token, channelId, text, threadTs, blocksJson)` from Task 2; `ConversationInfo.numMembers()` from Task 2
- Produces: `Messaging.send()` that renders `RichCard` → Block Kit; `Discovery.listChannels()` returning `Channel` with `memberCount`

- [ ] **Step 1: Write test for sending message with RichCard**

In `SlackChatPlatformTest.java`:

```java
@Test
void sendMessageWithRichCard() {
    wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
            .willReturn(okJson("{\"ok\":true,\"ts\":\"1234567890.123456\"}")));

    final var card = RichCard.builder()
            .title("Deploy Summary")
            .description("3 services updated")
            .fields(List.of(new RichCard.Field("env", "prod", true)))
            .footer("bot v2")
            .author("CI")
            .build();

    final var content = new ChatContent("fallback", null, List.of(), List.of(card));
    final var result = platform.messaging().send(new ChatChannelRef("C123"), content);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/chat.postMessage"))
            .withRequestBody(matchingJsonPath("$.text", equalTo("fallback")))
            .withRequestBody(matchingJsonPath("$.blocks[0].type", equalTo("header")))
            .withRequestBody(matchingJsonPath("$.blocks[0].text.text", equalTo("Deploy Summary")))
            .withRequestBody(matchingJsonPath("$.blocks[1].type", equalTo("section")))
            .withRequestBody(matchingJsonPath("$.blocks[1].text.text", equalTo("3 services updated"))));
}
```

- [ ] **Step 2: Write test for message without cards is unchanged**

```java
@Test
void sendMessageWithoutCardsNoBlocks() {
    wireMock.stubFor(post(urlEqualTo("/api/chat.postMessage"))
            .willReturn(okJson("{\"ok\":true,\"ts\":\"1234567890.123456\"}")));

    final var content = new ChatContent("just text");
    final var result = platform.messaging().send(new ChatChannelRef("C123"), content);

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/chat.postMessage"))
            .withRequestBody(not(matchingJsonPath("$.blocks"))));
}
```

- [ ] **Step 3: Write test for discovery with member count**

```java
@Test
void listChannelsIncludesMemberCount() {
    wireMock.stubFor(get(urlPathEqualTo("/api/conversations.list"))
            .willReturn(okJson("{\"ok\":true,\"channels\":[{\"id\":\"C1\",\"name\":\"general\","
                    + "\"topic\":{\"value\":\"t\"},\"purpose\":{\"value\":\"p\"},"
                    + "\"is_private\":false,\"num_members\":42}],"
                    + "\"response_metadata\":{\"next_cursor\":\"\"}}")));

    final var channels = platform.discovery().listChannels();

    assertThat(channels).hasSize(1);
    assertThat(channels.getFirst().memberCount()).isEqualTo(42);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-slack`
Expected: Compilation failures or assertion failures — no block rendering yet.

- [ ] **Step 5: Implement RichCard → Block Kit serialization**

In `SlackChatPlatform.java`, add a private serialization method:

```java
private String serializeToBlocks(final List<RichCard> cards) {
    final var arrayBuilder = Json.createArrayBuilder();
    boolean first = true;
    for (final RichCard card : cards) {
        if (!first) {
            arrayBuilder.add(Json.createObjectBuilder().add("type", "divider"));
        }
        first = false;

        if (card.title() != null) {
            arrayBuilder.add(Json.createObjectBuilder()
                    .add("type", "header")
                    .add("text", Json.createObjectBuilder()
                            .add("type", "plain_text")
                            .add("text", card.title())));
        }

        if (card.description() != null) {
            final String descText = card.url() != null && card.title() != null
                    ? "<" + card.url() + "|" + card.title() + ">\n" + card.description()
                    : card.description();
            final var sectionBuilder = Json.createObjectBuilder()
                    .add("type", "section")
                    .add("text", Json.createObjectBuilder()
                            .add("type", "mrkdwn")
                            .add("text", descText));
            if (card.thumbnailUrl() != null) {
                sectionBuilder.add("accessory", Json.createObjectBuilder()
                        .add("type", "image")
                        .add("image_url", card.thumbnailUrl())
                        .add("alt_text", "thumbnail"));
            }
            arrayBuilder.add(sectionBuilder);
        }

        if (!card.fields().isEmpty()) {
            final var fieldsArray = Json.createArrayBuilder();
            for (final RichCard.Field f : card.fields()) {
                fieldsArray.add(Json.createObjectBuilder()
                        .add("type", "mrkdwn")
                        .add("text", "*" + f.name() + "*\n" + f.value()));
            }
            arrayBuilder.add(Json.createObjectBuilder()
                    .add("type", "section")
                    .add("fields", fieldsArray));
        }

        if (card.imageUrl() != null) {
            arrayBuilder.add(Json.createObjectBuilder()
                    .add("type", "image")
                    .add("image_url", card.imageUrl())
                    .add("alt_text", "image"));
        }

        if (card.footer() != null || card.author() != null) {
            final var elements = Json.createArrayBuilder();
            if (card.author() != null) {
                elements.add(Json.createObjectBuilder()
                        .add("type", "mrkdwn")
                        .add("text", card.author()));
            }
            if (card.footer() != null) {
                elements.add(Json.createObjectBuilder()
                        .add("type", "mrkdwn")
                        .add("text", card.footer()));
            }
            arrayBuilder.add(Json.createObjectBuilder()
                    .add("type", "context")
                    .add("elements", elements));
        }
    }
    return arrayBuilder.build().toString();
}
```

Add imports: `import jakarta.json.Json;`

- [ ] **Step 6: Update sendMessage and sendReply to pass blocks**

Update `sendMessage`:
```java
private SendResult sendMessage(final ChatChannelRef channel, final ChatContent content) {
    final String blocksJson = content.cards().isEmpty() ? null : serializeToBlocks(content.cards());
    final PostResult result = client.postMessage(
            token, channel.id(), content.text(), null, blocksJson);
    if (!result.ok()) {
        return SendResult.failure(result.error());
    }
    return SendResult.success(
            new ChatMessageRef(channel, result.ts()),
            parseTs(result.ts()));
}
```

Update `sendReply`:
```java
private SendResult sendReply(final ChatMessageRef parent, final ChatContent content) {
    final String blocksJson = content.cards().isEmpty() ? null : serializeToBlocks(content.cards());
    final PostResult result = client.postMessage(
            token, parent.channel().id(), content.text(), parent.messageId(), blocksJson);
    if (!result.ok()) {
        return SendResult.failure(result.error());
    }
    return SendResult.success(
            new ChatMessageRef(parent.channel(), result.ts()),
            parseTs(result.ts()));
}
```

- [ ] **Step 7: Update discovery to include member count**

Update `toChannel`:
```java
private static Channel toChannel(final ConversationInfo c) {
    return new Channel(
            new ChatChannelRef(c.id()),
            c.name(),
            c.topic(),
            c.purpose(),
            c.isPrivate(),
            c.numMembers());
}
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-slack`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```
feat(chat-slack): translate RichCard to Block Kit, add member count to discovery — Refs #37, #41, #42
```

---

### Task 5: MCP Tool Consolidation — send_chat + list_chat_channels

Create `ChatPlatformMcpTool` with `send_chat` and `list_chat_channels`, delete `SlackBotMcpTool` and `DiscordMcpTool`, update module dependencies.

**Files:**
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/ChatPlatformMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/ChatPlatformMcpToolTest.java`
- Delete: `mcp/src/main/java/io/casehub/connectors/mcp/SlackBotMcpTool.java`
- Delete: `mcp/src/main/java/io/casehub/connectors/mcp/DiscordMcpTool.java`
- Delete: `mcp/src/test/java/io/casehub/connectors/slack/bot/SlackBotMcpToolTest.java`
- Delete: `mcp/src/test/java/io/casehub/connectors/mcp/DiscordMcpToolTest.java`
- Modify: `mcp/pom.xml`

**Interfaces:**
- Consumes: `ChatPlatformService.platform(id)` from `chat-spi`; `ChatPlatform.messaging().send()`, `.threading().reply()`, `.discovery().listChannels()` from `chat-spi`; `ConnectorMeshBridge.notifyDelivered()` from `core`; `McpContentSanitizer.sanitize()` from `mcp`
- Produces: `send_chat` MCP tool, `list_chat_channels` MCP tool

- [ ] **Step 1: Update mcp/pom.xml dependencies**

Replace:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-slack-bot</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-discord</artifactId>
</dependency>
```

With:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-chat-slack</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-chat-discord</artifactId>
</dependency>
```

- [ ] **Step 2: Write test for send_chat routing to Slack**

Create `mcp/src/test/java/io/casehub/connectors/mcp/ChatPlatformMcpToolTest.java`:

```java
package io.casehub.connectors.mcp;

import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.chat.ChatPlatformService;
import io.casehub.connectors.chat.model.*;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.ref.InMemoryChatBackend;
import io.casehub.connectors.chat.ref.RefChatPlatform;

import static org.assertj.core.api.Assertions.assertThat;

class ChatPlatformMcpToolTest {

    private ChatPlatformMcpTool tool;
    private InMemoryChatBackend backend;
    private RecordingBridge bridge;

    @BeforeEach
    void setUp() {
        backend = new InMemoryChatBackend();
        backend.createChannel("general", "topic", "desc", false);
        final var refPlatform = new RefChatPlatform(backend);
        final var service = new ChatPlatformService(List.of(refPlatform));
        bridge = new RecordingBridge();
        tool = new ChatPlatformMcpTool(service, bridge);
    }

    @Test
    void sendChatPlainText() {
        final var channelId = backend.listChannels().getFirst().ref().id();
        final var result = tool.sendChat("ref", channelId, "hello", null,
                null, null, null, null, null, null, null, null, null);

        assertThat(result).startsWith("Sent to ");
        assertThat(backend.messages(new ChatChannelRef(channelId),
                java.time.Instant.EPOCH)).hasSize(1);
    }

    @Test
    void sendChatWithRichCard() {
        final var channelId = backend.listChannels().getFirst().ref().id();
        final var result = tool.sendChat("ref", channelId, "fallback", null,
                "Deploy", "3 services", null, null, null, null, null, null, null);

        assertThat(result).startsWith("Sent to ");
        final var messages = backend.messages(new ChatChannelRef(channelId),
                java.time.Instant.EPOCH);
        assertThat(messages).hasSize(1);
        assertThat(messages.getFirst().content().cards()).hasSize(1);
        assertThat(messages.getFirst().content().cards().getFirst().title())
                .isEqualTo("Deploy");
    }

    @Test
    void sendChatUnknownPlatform() {
        final var result = tool.sendChat("nonexistent", "ch", "hi", null,
                null, null, null, null, null, null, null, null, null);
        assertThat(result).startsWith("Failed:");
    }

    @Test
    void sendChatWithThread() {
        final var channelId = backend.listChannels().getFirst().ref().id();
        final var firstResult = tool.sendChat("ref", channelId, "parent", null,
                null, null, null, null, null, null, null, null, null);
        assertThat(firstResult).contains("messageId=");

        final var messageId = firstResult.replaceAll(".*messageId=([^)]+)\\).*", "$1");
        final var replyResult = tool.sendChat("ref", channelId, "reply", messageId,
                null, null, null, null, null, null, null, null, null);
        assertThat(replyResult).startsWith("Sent to ");
    }

    @Test
    void listChatChannels() {
        final var result = tool.listChatChannels("ref");
        assertThat(result).contains("general");
    }

    @Test
    void listChatChannelsUnknownPlatform() {
        final var result = tool.listChatChannels("nonexistent");
        assertThat(result).startsWith("Failed:");
    }

    @Test
    void meshBridgeNotified() {
        final var channelId = backend.listChannels().getFirst().ref().id();
        tool.sendChat("ref", channelId, "hello", null,
                null, null, null, null, null, null, null, null, null);
        assertThat(bridge.calls).hasSize(1);
        assertThat(bridge.calls.getFirst().connectorId()).isEqualTo("ref");
    }

    static class RecordingBridge implements ConnectorMeshBridge {
        final java.util.List<Call> calls = new java.util.ArrayList<>();
        record Call(String connectorId, String destination, String content) {}
        @Override
        public void notifyDelivered(String connectorId, String destination, String content) {
            calls.add(new Call(connectorId, destination, content));
        }
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp`
Expected: Compilation failure — `ChatPlatformMcpTool` doesn't exist.

- [ ] **Step 4: Implement ChatPlatformMcpTool**

Create `mcp/src/main/java/io/casehub/connectors/mcp/ChatPlatformMcpTool.java`:

```java
package io.casehub.connectors.mcp;

import java.io.StringReader;
import java.util.ArrayList;
import java.util.List;

import org.jboss.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.json.Json;

import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.smallrye.common.annotation.Blocking;

import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.chat.ChatPlatformService;
import io.casehub.connectors.chat.model.*;
import io.casehub.connectors.chat.spi.ChatPlatform;

@ApplicationScoped
public class ChatPlatformMcpTool {

    private static final Logger LOG = Logger.getLogger(ChatPlatformMcpTool.class);

    private final ChatPlatformService platformService;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    public ChatPlatformMcpTool(final ChatPlatformService platformService,
                               final ConnectorMeshBridge meshBridge) {
        this.platformService = platformService;
        this.meshBridge = meshBridge;
    }

    @Tool(name = "send_chat",
          description = "Sends a message to a chat channel on any configured platform "
                      + "(slack, discord, irc, ref). Supports optional rich content via "
                      + "card parameters (title, description, fields, images). "
                      + "Returns 'Sent to <channel> (messageId=<id>)' on success.")
    @Blocking
    public String sendChat(
            @ToolArg(description = "Chat platform id: slack, discord, irc, ref.")
            final String platform,
            @ToolArg(description = "Channel ID (e.g. C123ABC for Slack, snowflake for Discord).")
            final String channel,
            @ToolArg(description = "Message text — required. Serves as notification fallback "
                                 + "when rich cards are present.")
            final String text,
            @ToolArg(description = "Parent message ID for threaded replies. "
                                 + "Slack: ts value. Discord: message ID. Omit for new message.",
                     required = false)
            final String parentMessageId,
            @ToolArg(description = "Rich card title (max 256 on Discord).", required = false)
            final String cardTitle,
            @ToolArg(description = "Rich card description/body text.", required = false)
            final String cardDescription,
            @ToolArg(description = "Card color as decimal RGB (e.g. 16711680 = red). "
                                 + "Discord only — ignored on Slack.",
                     required = false)
            final String cardColor,
            @ToolArg(description = "URL — makes card title a hyperlink.", required = false)
            final String cardUrl,
            @ToolArg(description = "Thumbnail URL — small image.", required = false)
            final String cardThumbnailUrl,
            @ToolArg(description = "Image URL — full-width image.", required = false)
            final String cardImageUrl,
            @ToolArg(description = "Card footer text.", required = false)
            final String cardFooter,
            @ToolArg(description = "Card author text.", required = false)
            final String cardAuthor,
            @ToolArg(description = "Card fields as JSON array: "
                                 + "[{\"name\":\"...\",\"value\":\"...\",\"inline\":true}].",
                     required = false)
            final String cardFields) {
        try {
            final ChatPlatform p = platformService.platform(platform);

            final List<RichCard> cards;
            if (hasAnyCardParam(cardTitle, cardDescription, cardColor, cardUrl,
                    cardThumbnailUrl, cardImageUrl, cardFooter, cardAuthor, cardFields)) {
                final var builder = RichCard.builder()
                        .title(cardTitle)
                        .description(cardDescription)
                        .url(cardUrl)
                        .thumbnailUrl(cardThumbnailUrl)
                        .imageUrl(cardImageUrl)
                        .footer(cardFooter)
                        .author(cardAuthor);

                if (cardColor != null && !cardColor.isBlank()) {
                    try {
                        builder.color(Integer.parseInt(cardColor));
                    } catch (final NumberFormatException e) {
                        return "Failed: cardColor must be a decimal integer";
                    }
                }

                if (cardFields != null && !cardFields.isBlank()) {
                    try {
                        final var jsonArray = Json.createReader(
                                new StringReader(cardFields)).readArray();
                        final List<RichCard.Field> fields = new ArrayList<>();
                        for (final var jv : jsonArray) {
                            final var jo = jv.asJsonObject();
                            fields.add(new RichCard.Field(
                                    jo.getString("name"),
                                    jo.getString("value"),
                                    jo.getBoolean("inline", false)));
                        }
                        builder.fields(fields);
                    } catch (final Exception e) {
                        return "Failed: cardFields must be a JSON array of "
                             + "{name, value, inline} objects";
                    }
                }

                cards = List.of(builder.build());
            } else {
                cards = List.of();
            }

            final var content = new ChatContent(text, null, List.of(), cards);
            final SendResult result;
            if (parentMessageId != null && !parentMessageId.isBlank()) {
                final var parentRef = new ChatMessageRef(
                        new ChatChannelRef(channel), parentMessageId);
                result = p.threading().reply(parentRef, content);
            } else {
                result = p.messaging().send(new ChatChannelRef(channel), content);
            }

            if (!result.ok()) {
                return "Failed: " + result.error();
            }

            meshBridge.notifyDelivered(p.id(), channel,
                    McpContentSanitizer.sanitize(text));

            return "Sent to " + channel + " (messageId="
                    + result.messageRef().messageId() + ")";
        } catch (final IllegalArgumentException e) {
            return "Failed: " + e.getMessage();
        } catch (final Exception e) {
            LOG.warnf("send_chat failed [%s]: %s",
                    e.getClass().getSimpleName(), e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }

    @Tool(name = "list_chat_channels",
          description = "Lists channels on a chat platform with rich detail: "
                      + "name, ID, topic, description, private flag, member count. "
                      + "Use list_channels for a thin cross-connector overview.")
    @Blocking
    public String listChatChannels(
            @ToolArg(description = "Chat platform id: slack, discord, irc, ref.")
            final String platform) {
        try {
            final ChatPlatform p = platformService.platform(platform);
            final var channels = p.discovery().listChannels();

            if (channels.isEmpty()) {
                return "No channels found on " + platform + ".";
            }

            final var sb = new StringBuilder();
            for (final Channel ch : channels) {
                sb.append("#").append(ch.name())
                  .append(" (").append(ch.ref().id()).append(")");
                if (ch.isPrivate()) sb.append(" [private]");
                if (ch.memberCount() != null) sb.append(" [").append(ch.memberCount()).append(" members]");
                if (ch.topic() != null && !ch.topic().isBlank())
                    sb.append(" — ").append(ch.topic());
                if (ch.description() != null && !ch.description().isBlank())
                    sb.append(" | ").append(ch.description());
                sb.append("\n");
            }
            return sb.toString().stripTrailing();
        } catch (final IllegalArgumentException e) {
            return "Failed: " + e.getMessage();
        } catch (final Exception e) {
            LOG.warnf("list_chat_channels failed [%s]: %s",
                    e.getClass().getSimpleName(), e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }

    private static boolean hasAnyCardParam(final String... params) {
        for (final String p : params) {
            if (p != null && !p.isBlank()) return true;
        }
        return false;
    }
}
```

- [ ] **Step 5: Delete old MCP tools**

Delete these files:
- `mcp/src/main/java/io/casehub/connectors/mcp/SlackBotMcpTool.java`
- `mcp/src/main/java/io/casehub/connectors/mcp/DiscordMcpTool.java`
- `mcp/src/test/java/io/casehub/connectors/slack/bot/SlackBotMcpToolTest.java`
- `mcp/src/test/java/io/casehub/connectors/mcp/DiscordMcpToolTest.java`

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo`
Expected: Full build succeeds — all modules compile and tests pass.

- [ ] **Step 7: Commit**

```
feat(mcp): consolidate send_slack_bot + send_discord into send_chat, add list_chat_channels — Refs #37, #41, #42
```

---

### Task 6: Full Integration Verification + CLAUDE.md Update

Run the full build, verify everything compiles and tests pass, update CLAUDE.md to reflect the new tool surface.

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/connectors/CLAUDE.md` — update "What This Project Is" section with new tools

**Interfaces:**
- Consumes: all prior tasks
- Produces: updated project documentation

- [ ] **Step 1: Full build with demo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo`
Expected: BUILD SUCCESS

- [ ] **Step 2: Update CLAUDE.md**

Update the "What This Project Is" section to reflect:
- `RichCard` model in `chat-spi` for platform-agnostic rich content
- `send_chat` replacing `send_slack_bot` and `send_discord`
- `list_chat_channels` replacing `list_discord_channels` and satisfying #42
- `Channel` now includes `memberCount`
- MCP tools list: replace `send_slack_bot`, `send_discord`, `list_discord_channels` with `send_chat`, `list_chat_channels`

- [ ] **Step 3: Commit**

```
docs: update CLAUDE.md for rich content model and MCP tool consolidation — Refs #37, #41, #42
```
