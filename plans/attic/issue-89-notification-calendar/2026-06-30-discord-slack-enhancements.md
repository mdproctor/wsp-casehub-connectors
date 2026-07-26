# Discord & Slack Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Discord message history attachment downloading (#36), advanced embed MCP parameters (#38), and SlackChatPlatform with all 9 native capabilities (#40), plus fix a bug where DiscordInboundTranslator drops attachments.

**Architecture:** Four features on one branch. #36 and the bug fix are small changes in chat-discord. #38 extends the MCP tool with validation. #40 is a new chat-slack module backed by expanded SlackBotClient methods, following the same CDI pattern as chat-discord. All Slack HTTP calls go through `SlackBotClient` using `HttpHelper.CLIENT` per protocol.

**Tech Stack:** Java 21, Quarkus 3.32.2, jakarta.json (SlackBotClient), Jackson (DiscordClient), WireMock (tests), AssertJ

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All HTTP calls use `HttpHelper.CLIENT` — never `HttpClient.newHttpClient()` (protocol: `shared-http-client`)
- Credentials passed at call time, not stored on shared clients (protocol: `credential-config-ownership`)
- SPI identifier methods named `id()` (protocol: `spi-id-method-naming`)
- `@Blocking` on every `@Tool` method calling blocking HTTP (protocol: `mcp-tool-blocking-annotation`)
- Paginating methods return partial results + WARNING on failure (protocol: `paginating-client-fail-soft`)
- IntelliJ MCP for all code navigation — never bash grep/find for classes
- TDD: red → green → refactor → commit

---

### Task 1: Discord attachment bug fix + message history downloading (#36)

**Files:**
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundTranslator.java:36`
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java:319-333`
- Test: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInboundTranslatorTest.java`
- Test: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java`

**Interfaces:**
- Consumes: `DiscordClient.downloadAttachment(DiscordAttachment) → Attachment` (existing)
- Produces: No new interfaces — existing contract, wired up

- [ ] **Step 1: Write failing test — translator drops attachments**

In `DiscordInboundTranslatorTest.java`, add:

```java
@Test
void translate_forwardsAttachments() {
    Attachment att = new Attachment("file.pdf", "application/pdf", new byte[]{1, 2, 3});
    InboundMessage inbound = new InboundMessage(
            "discord-inbound", "discord", "user-123", "channel-456",
            "See attached", List.of(att), Instant.now(),
            Map.of("discord-message-id", "msg-789", "discord-guild-id", "guild-999"),
            null);

    ReceivedMessage received = translator.translate(inbound);

    assertThat(received.content().attachments()).hasSize(1);
    assertThat(received.content().attachments().get(0).filename()).isEqualTo("file.pdf");
    assertThat(received.content().attachments().get(0).contentType()).isEqualTo("application/pdf");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest=DiscordInboundTranslatorTest#translate_forwardsAttachments -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `assertThat(received.content().attachments()).hasSize(1)` fails because attachments are `List.of()`

- [ ] **Step 3: Fix DiscordInboundTranslator**

In `DiscordInboundTranslator.java`, change line 36 from:
```java
new ChatContent(msg.content())
```
to:
```java
new ChatContent(msg.content(), null, msg.attachments())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest=DiscordInboundTranslatorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Write failing test — message history downloads attachments**

In `DiscordChatPlatformTest.java`, add a test that stubs a message with attachments and a CDN download endpoint:

```java
@Test
void messageHistory_downloadsAttachments() {
    // Stub message with attachment metadata
    wireMock.stubFor(get(urlMatching("/channels/chan-123/messages\\?limit=100&after=\\d+"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            [{
                              "id": "msg-att",
                              "channel_id": "chan-123",
                              "content": "See file",
                              "author": {"id": "u1", "username": "alice"},
                              "timestamp": "2026-06-29T10:00:00Z",
                              "type": 0,
                              "attachments": [{
                                "id": "att-1",
                                "filename": "report.pdf",
                                "content_type": "application/pdf",
                                "size": 1024,
                                "url": "%s/cdn/report.pdf"
                              }]
                            }]
                            """.formatted(wireMock.baseUrl()))));

    // Stub CDN download
    wireMock.stubFor(get(urlEqualTo("/cdn/report.pdf"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Length", "3")
                    .withBody(new byte[]{1, 2, 3})));

    ChatChannelRef channel = new ChatChannelRef("chan-123");
    Instant since = Instant.parse("2026-06-29T09:00:00Z");

    List<ReceivedMessage> messages = platform.messageHistory().messages(channel, since);

    assertThat(messages).hasSize(1);
    assertThat(messages.get(0).content().attachments()).hasSize(1);
    assertThat(messages.get(0).content().attachments().get(0).filename()).isEqualTo("report.pdf");
    assertThat(messages.get(0).content().attachments().get(0).content()).containsExactly(1, 2, 3);
}

@Test
void messageHistory_attachmentDownloadFailure_gracefulSkip() {
    wireMock.stubFor(get(urlMatching("/channels/chan-123/messages\\?limit=100&after=\\d+"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            [{
                              "id": "msg-att2",
                              "channel_id": "chan-123",
                              "content": "Bad file",
                              "author": {"id": "u1", "username": "alice"},
                              "timestamp": "2026-06-29T10:00:00Z",
                              "type": 0,
                              "attachments": [{
                                "id": "att-2",
                                "filename": "missing.pdf",
                                "content_type": "application/pdf",
                                "size": 1024,
                                "url": "%s/cdn/missing.pdf"
                              }]
                            }]
                            """.formatted(wireMock.baseUrl()))));

    // CDN returns 403 (expired URL)
    wireMock.stubFor(get(urlEqualTo("/cdn/missing.pdf"))
            .willReturn(aResponse().withStatus(403)));

    ChatChannelRef channel = new ChatChannelRef("chan-123");
    Instant since = Instant.parse("2026-06-29T09:00:00Z");

    List<ReceivedMessage> messages = platform.messageHistory().messages(channel, since);

    assertThat(messages).hasSize(1);
    assertThat(messages.get(0).content().attachments()).isEmpty();
    assertThat(messages.get(0).content().text()).isEqualTo("Bad file");
}
```

Note: The CDN host for the test URL must be `localhost` which needs to be in `allowedCdnHosts`. Check the test `setUp()` — the `DiscordClient` test field `allowedCdnHosts` should include `"localhost"`. If not, add it via reflection in setUp: `setField(client, "allowedCdnHosts", Set.of("cdn.discordapp.com", "media.discordapp.net", "localhost"))`.

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest=DiscordChatPlatformTest#messageHistory_downloadsAttachments -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — attachments are `List.of()` because `toReceivedMessage()` doesn't download them

- [ ] **Step 7: Implement attachment downloading in toReceivedMessage**

In `DiscordChatPlatform.java`, replace `toReceivedMessage()`:

```java
private ReceivedMessage toReceivedMessage(final ChatChannelRef channel, final DiscordMessage dm) {
    final ChatMessageRef messageRef = new ChatMessageRef(channel, dm.id());
    final ChatMessageRef parentRef = dm.type() == 19 && dm.referencedMessageId() != null
            ? new ChatMessageRef(channel, dm.referencedMessageId())
            : null;

    final List<Attachment> attachments = new ArrayList<>();
    for (final DiscordAttachment da : dm.attachments()) {
        final Attachment downloaded = client.downloadAttachment(da);
        if (downloaded != null) {
            attachments.add(downloaded);
        }
    }

    return new ReceivedMessage(
            InboundConnectorTypes.DISCORD,
            channel,
            messageRef,
            parentRef,
            new MemberRef(dm.author().id()),
            new ChatContent(dm.content(), null, attachments),
            dm.timestamp());
}
```

Add import: `import java.util.ArrayList;`
Add import: `import io.casehub.connectors.Attachment;`
Add import: `import io.casehub.connectors.discord.model.DiscordAttachment;`

- [ ] **Step 8: Run all chat-discord tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat(chat-discord): download attachments in message history + fix translator forwarding — Refs #36

DiscordChatPlatform.toReceivedMessage() now downloads each DiscordAttachment
via DiscordClient.downloadAttachment() with existing SSRF defense.
DiscordInboundTranslator.translate() now forwards msg.attachments() to
ChatContent instead of discarding them.
```

---

### Task 2: Advanced embed MCP parameters (#38)

**Files:**
- Modify: `mcp/src/main/java/io/casehub/connectors/mcp/DiscordMcpTool.java`
- Test: `mcp/src/test/java/io/casehub/connectors/mcp/DiscordMcpToolTest.java`

**Interfaces:**
- Consumes: `DiscordClient.sendMessage(token, channelId, content, embeds)`, `DiscordEmbed` (existing)
- Produces: No new interfaces — extends existing MCP tool parameters

- [ ] **Step 1: Write failing tests for new embed parameters**

In `DiscordMcpToolTest.java`, add tests. Note: `sendDiscord()` signature will grow from 6 to 12 parameters. Update ALL existing test calls to pass the 6 new `null` args.

```java
@Test
void sendDiscord_fullEmbed_allParams() {
    wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    final String result = tool.sendDiscord("ch1", null, null,
            "Title", "Description", "255",
            "https://example.com",
            "https://example.com/thumb.png",
            "https://example.com/image.png",
            "Footer text", "Author Name",
            "[{\"name\":\"Field 1\",\"value\":\"Value 1\",\"inline\":true}]");

    assertThat(result).startsWith("Posted to");
    wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.embeds[0].url", equalTo("https://example.com")))
            .withRequestBody(matchingJsonPath("$.embeds[0].thumbnail.url", equalTo("https://example.com/thumb.png")))
            .withRequestBody(matchingJsonPath("$.embeds[0].image.url", equalTo("https://example.com/image.png")))
            .withRequestBody(matchingJsonPath("$.embeds[0].footer.text", equalTo("Footer text")))
            .withRequestBody(matchingJsonPath("$.embeds[0].author.name", equalTo("Author Name")))
            .withRequestBody(matchingJsonPath("$.embeds[0].fields[0].name", equalTo("Field 1")))
            .withRequestBody(matchingJsonPath("$.embeds[0].fields[0].inline", equalTo("true"))));
}

@Test
void sendDiscord_embedFieldsMalformedJson_returnsFailed() {
    final String result = tool.sendDiscord("ch1", null, null,
            "Title", null, null, null, null, null, null, null, "not-json");

    assertThat(result).startsWith("Failed: embedFields must be");
}

@Test
void sendDiscord_embedFieldMissingName_returnsFailed() {
    final String result = tool.sendDiscord("ch1", null, null,
            "Title", null, null, null, null, null, null, null,
            "[{\"value\":\"val\"}]");

    assertThat(result).startsWith("Failed: embedFields");
}

@Test
void sendDiscord_embedTitleExceedsLimit_returnsFailed() {
    final String result = tool.sendDiscord("ch1", null, null,
            "x".repeat(257), null, null, null, null, null, null, null, null);

    assertThat(result).isEqualTo("Failed: embedTitle exceeds 256 characters");
}

@Test
void sendDiscord_embedDescriptionExceedsLimit_returnsFailed() {
    final String result = tool.sendDiscord("ch1", null, null,
            "Title", "x".repeat(4097), null, null, null, null, null, null, null);

    assertThat(result).isEqualTo("Failed: embedDescription exceeds 4096 characters");
}

@Test
void sendDiscord_embedFieldsExceeds25_returnsFailed() {
    StringBuilder fields = new StringBuilder("[");
    for (int i = 0; i < 26; i++) {
        if (i > 0) fields.append(",");
        fields.append("{\"name\":\"f").append(i).append("\",\"value\":\"v\"}");
    }
    fields.append("]");

    final String result = tool.sendDiscord("ch1", null, null,
            "Title", null, null, null, null, null, null, null, fields.toString());

    assertThat(result).isEqualTo("Failed: embedFields exceeds 25 fields");
}

@Test
void sendDiscord_totalEmbedContentExceedsLimit_returnsFailed() {
    final String result = tool.sendDiscord("ch1", null, null,
            "Title", "x".repeat(4096), null, null, null, null,
            "x".repeat(2000), null, null);

    assertThat(result).startsWith("Failed: total embed content exceeds 6000 characters");
}

@Test
void sendDiscord_embedUrlWithoutTitle_returnsFailed() {
    final String result = tool.sendDiscord("ch1", "text", null,
            null, "Desc", null, "https://example.com", null, null, null, null, null);

    assertThat(result).isEqualTo("Failed: embedUrl requires embedTitle");
}

@Test
void sendDiscord_onlyFooter_isValidEmbed() {
    wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    final String result = tool.sendDiscord("ch1", null, null,
            null, null, null, null, null, null, "Just footer", null, null);

    assertThat(result).startsWith("Posted to");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=DiscordMcpToolTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: COMPILATION FAILURE — `sendDiscord()` has wrong parameter count

- [ ] **Step 3: Implement the new parameters and validation**

Update `DiscordMcpTool.sendDiscord()` — add 6 new `@ToolArg` parameters, embed field parsing, and Discord API limit validation. The method signature becomes:

```java
@Tool(name = "send_discord",
      description = "Posts a message to a Discord channel via bot token. ...")
@Blocking
public String sendDiscord(
        @ToolArg(description = "Discord channel ID (snowflake).")
        final String channel,
        @ToolArg(description = "Message text (max 2000 chars). Optional if embed args provided.",
                 required = false)
        final String text,
        @ToolArg(description = "Discord message ID to reply to.", required = false)
        final String replyToMessageId,
        @ToolArg(description = "Embed title (max 256 chars).", required = false)
        final String embedTitle,
        @ToolArg(description = "Embed description (max 4096 chars).", required = false)
        final String embedDescription,
        @ToolArg(description = "Embed color as decimal integer RGB.", required = false)
        final String embedColor,
        @ToolArg(description = "Embed URL — makes title a hyperlink. Requires embedTitle.", required = false)
        final String embedUrl,
        @ToolArg(description = "Embed thumbnail URL — small image top-right.", required = false)
        final String embedThumbnailUrl,
        @ToolArg(description = "Embed image URL — full-width image below description.", required = false)
        final String embedImageUrl,
        @ToolArg(description = "Embed footer text (max 2048 chars).", required = false)
        final String embedFooter,
        @ToolArg(description = "Embed author name (max 256 chars).", required = false)
        final String embedAuthor,
        @ToolArg(description = "Embed fields as JSON array: [{\"name\":\"...\",\"value\":\"...\",\"inline\":true}]. Max 25 fields.", required = false)
        final String embedFields)
```

Implementation details:
- Parse `embedFields` using `jakarta.json`: `Json.createReader(new StringReader(embedFields)).readArray()`
- Catch `Exception` on parse → return `"Failed: embedFields must be a JSON array of {name, value, inline} objects"`
- Validate each field has `name` and `value` (both non-null strings)
- `inline` defaults to `false` if absent
- Validate Discord limits before constructing `DiscordEmbed`
- `hasEmbed` extends to check all 9 embed parameters
- `embedUrl` requires `embedTitle` (check before sending)
- Construct `DiscordEmbed` with all parameters including `Footer` and `Author` sub-records

- [ ] **Step 4: Run all MCP tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: ALL PASS (including updated existing tests with new null args)

- [ ] **Step 5: Commit**

```
feat(mcp): add advanced embed parameters to send_discord — Refs #38

Adds embedUrl, embedThumbnailUrl, embedImageUrl, embedFooter,
embedAuthor, embedFields to send_discord MCP tool with full Discord
API limit validation (256 title, 4096 desc, 25 fields, 6000 total).
```

---

### Task 3: SlackBotClient expansion

**Files:**
- Modify: `slack-bot/src/main/java/io/casehub/connectors/slack/bot/SlackBotClient.java`
- Test: `slack-bot/src/test/java/io/casehub/connectors/slack/bot/SlackBotClientTest.java`

**Interfaces:**
- Consumes: `HttpHelper.CLIENT` (existing), `DiscoveredTarget` (existing)
- Produces: All new methods and result records consumed by Task 4 (SlackChatPlatform):
  - `listConversations(String token) → List<ConversationInfo>`
  - `addReaction(String token, String channel, String timestamp, String emoji) → ReactionResult`
  - `removeReaction(String token, String channel, String timestamp, String emoji) → ReactionResult`
  - `getReactions(String token, String channel, String timestamp) → ReactionListResult`
  - `getPresence(String token, String userId) → PresenceResult`
  - `listConversationMembers(String token, String channelId) → List<String>`
  - `listUsers(String token) → List<UserInfo>`
  - `createConversation(String token, String name, boolean isPrivate) → ConversationResult`
  - `getConversationInfo(String token, String channelId) → ConversationResult`
  - `inviteToConversation(String token, String channelId, String userId) → ReactionResult`
  - `kickFromConversation(String token, String channelId, String userId) → ReactionResult`
  - `getHistory(String token, String channelId, String oldest, int limit) → HistoryResult`

This is a large task because every method follows the identical HTTP pattern. Group tests by API concern.

- [ ] **Step 1: Add result records to SlackBotClient**

Add these inner records to `SlackBotClient.java`:

```java
public record ConversationInfo(String id, String name, String topic, String purpose, boolean isPrivate) {}
public record ReactionResult(boolean ok, String error) {}
public record ReactionListResult(boolean ok, List<String> emojis, String error) {}
public record PresenceResult(boolean ok, String presence, String error) {}
public record UserInfo(String id, String displayName, String realName) {}
public record HistoryMessage(String ts, String user, String text, String threadTs) {}
public record HistoryResult(boolean ok, List<HistoryMessage> messages, String error) {}
```

- [ ] **Step 2: Write failing test — listConversations + listChannels delegation**

```java
@Test
void listConversations_returnsFullDetail() {
    wireMock.stubFor(get(urlMatching("/api/conversations\\.list.*"))
            .willReturn(okJson("""
                    {"ok":true,"channels":[
                      {"id":"C1","name":"general","topic":{"value":"Main"},"purpose":{"value":"General discussion"},"is_private":false},
                      {"id":"C2","name":"secret","topic":{"value":""},"purpose":{"value":"Private stuff"},"is_private":true}
                    ],"response_metadata":{"next_cursor":""}}
                    """)));

    List<SlackBotClient.ConversationInfo> convos = client.listConversations("tok");

    assertThat(convos).hasSize(2);
    assertThat(convos.get(0).id()).isEqualTo("C1");
    assertThat(convos.get(0).name()).isEqualTo("general");
    assertThat(convos.get(0).topic()).isEqualTo("Main");
    assertThat(convos.get(0).purpose()).isEqualTo("General discussion");
    assertThat(convos.get(0).isPrivate()).isFalse();
    assertThat(convos.get(1).isPrivate()).isTrue();
}

@Test
void listChannels_delegatesToListConversations() {
    wireMock.stubFor(get(urlMatching("/api/conversations\\.list.*"))
            .willReturn(okJson("""
                    {"ok":true,"channels":[
                      {"id":"C1","name":"general","topic":{"value":""},"purpose":{"value":""},"is_private":false}
                    ],"response_metadata":{"next_cursor":""}}
                    """)));

    List<DiscoveredTarget> targets = client.listChannels("tok");

    assertThat(targets).hasSize(1);
    assertThat(targets.get(0).id()).isEqualTo("C1");
    assertThat(targets.get(0).displayName()).isEqualTo("#general");
}
```

- [ ] **Step 3: Implement listConversations + refactor listChannels**

Implement `listConversations(token)` with the same pagination loop as existing `listChannels`. Parse `topic.value`, `purpose.value`, `is_private` from each channel object. Refactor `listChannels` to delegate:

```java
public List<DiscoveredTarget> listChannels(final String token) {
    return listConversations(token).stream()
            .map(c -> new DiscoveredTarget(c.id(), "#" + c.name()))
            .toList();
}
```

- [ ] **Step 4: Write failing tests — reaction methods**

```java
@Test
void addReaction_success() {
    wireMock.stubFor(post(urlEqualTo("/api/reactions.add"))
            .willReturn(okJson("{\"ok\":true}")));

    SlackBotClient.ReactionResult result = client.addReaction("tok", "C1", "1234567890.123456", "thumbsup");

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/api/reactions.add"))
            .withRequestBody(matchingJsonPath("$.channel", equalTo("C1")))
            .withRequestBody(matchingJsonPath("$.timestamp", equalTo("1234567890.123456")))
            .withRequestBody(matchingJsonPath("$.name", equalTo("thumbsup"))));
}

@Test
void removeReaction_success() {
    wireMock.stubFor(post(urlEqualTo("/api/reactions.remove"))
            .willReturn(okJson("{\"ok\":true}")));

    SlackBotClient.ReactionResult result = client.removeReaction("tok", "C1", "1234567890.123456", "thumbsup");

    assertThat(result.ok()).isTrue();
}

@Test
void getReactions_success() {
    wireMock.stubFor(get(urlMatching("/api/reactions\\.get.*"))
            .willReturn(okJson("""
                    {"ok":true,"message":{"reactions":[
                      {"name":"thumbsup","count":3},
                      {"name":"heart","count":1}
                    ]}}
                    """)));

    SlackBotClient.ReactionListResult result = client.getReactions("tok", "C1", "1234567890.123456");

    assertThat(result.ok()).isTrue();
    assertThat(result.emojis()).containsExactly("thumbsup", "heart");
}

@Test
void addReaction_apiError() {
    wireMock.stubFor(post(urlEqualTo("/api/reactions.add"))
            .willReturn(okJson("{\"ok\":false,\"error\":\"already_reacted\"}")));

    SlackBotClient.ReactionResult result = client.addReaction("tok", "C1", "ts", "thumbsup");

    assertThat(result.ok()).isFalse();
    assertThat(result.error()).isEqualTo("already_reacted");
}
```

- [ ] **Step 5: Implement reaction methods**

Three methods, all following the same pattern:
- `addReaction`: POST to `/api/reactions.add` with JSON body `{channel, timestamp, name}`
- `removeReaction`: POST to `/api/reactions.remove` with JSON body `{channel, timestamp, name}`
- `getReactions`: GET `/api/reactions.get?channel={c}&timestamp={ts}&full=true`, parse `message.reactions[].name`

- [ ] **Step 6: Write failing tests — presence, members, users**

```java
@Test
void getPresence_active() {
    wireMock.stubFor(get(urlMatching("/api/users\\.getPresence.*"))
            .willReturn(okJson("{\"ok\":true,\"presence\":\"active\"}")));

    SlackBotClient.PresenceResult result = client.getPresence("tok", "U123");

    assertThat(result.ok()).isTrue();
    assertThat(result.presence()).isEqualTo("active");
}

@Test
void listConversationMembers_paginates() {
    wireMock.stubFor(get(urlMatching("/api/conversations\\.members\\?channel=C1&limit=200$"))
            .willReturn(okJson("""
                    {"ok":true,"members":["U1","U2"],"response_metadata":{"next_cursor":"page2"}}
                    """)));
    wireMock.stubFor(get(urlMatching("/api/conversations\\.members\\?channel=C1&limit=200&cursor=page2"))
            .willReturn(okJson("""
                    {"ok":true,"members":["U3"],"response_metadata":{"next_cursor":""}}
                    """)));

    List<String> members = client.listConversationMembers("tok", "C1");

    assertThat(members).containsExactly("U1", "U2", "U3");
}

@Test
void listUsers_returnsDisplayNames() {
    wireMock.stubFor(get(urlMatching("/api/users\\.list.*"))
            .willReturn(okJson("""
                    {"ok":true,"members":[
                      {"id":"U1","profile":{"display_name":"Alice","real_name":"Alice Smith"}},
                      {"id":"U2","profile":{"display_name":"","real_name":"Bob Jones"}}
                    ],"response_metadata":{"next_cursor":""}}
                    """)));

    List<SlackBotClient.UserInfo> users = client.listUsers("tok");

    assertThat(users).hasSize(2);
    assertThat(users.get(0).displayName()).isEqualTo("Alice");
    assertThat(users.get(1).displayName()).isEmpty();
    assertThat(users.get(1).realName()).isEqualTo("Bob Jones");
}
```

- [ ] **Step 7: Implement presence, members, users**

- `getPresence`: GET `/api/users.getPresence?user={userId}`, return `presence` field
- `listConversationMembers`: paginating GET `/api/conversations.members?channel={c}&limit=200`, accumulate `members[]` arrays
- `listUsers`: paginating GET `/api/users.list?limit=200`, parse `members[].id`, `members[].profile.display_name`, `members[].profile.real_name`

- [ ] **Step 8: Write failing tests — channel management + member management**

```java
@Test
void createConversation_success() {
    wireMock.stubFor(post(urlEqualTo("/api/conversations.create"))
            .willReturn(okJson("""
                    {"ok":true,"channel":{"id":"C99","name":"new-chan","topic":{"value":""},"purpose":{"value":""},"is_private":false}}
                    """)));

    SlackBotClient.ConversationResult result = client.createConversation("tok", "new-chan", false);

    assertThat(result.ok()).isTrue();
    assertThat(result.info().id()).isEqualTo("C99");
    assertThat(result.info().name()).isEqualTo("new-chan");
}

@Test
void getConversationInfo_success() {
    wireMock.stubFor(get(urlMatching("/api/conversations\\.info.*"))
            .willReturn(okJson("""
                    {"ok":true,"channel":{"id":"C1","name":"general","topic":{"value":"Main topic"},"purpose":{"value":"General chat"},"is_private":false}}
                    """)));

    SlackBotClient.ConversationResult result = client.getConversationInfo("tok", "C1");

    assertThat(result.ok()).isTrue();
    assertThat(result.info().topic()).isEqualTo("Main topic");
}

@Test
void inviteToConversation_success() {
    wireMock.stubFor(post(urlEqualTo("/api/conversations.invite"))
            .willReturn(okJson("{\"ok\":true}")));

    SlackBotClient.ReactionResult result = client.inviteToConversation("tok", "C1", "U1");

    assertThat(result.ok()).isTrue();
}

@Test
void kickFromConversation_success() {
    wireMock.stubFor(post(urlEqualTo("/api/conversations.kick"))
            .willReturn(okJson("{\"ok\":true}")));

    SlackBotClient.ReactionResult result = client.kickFromConversation("tok", "C1", "U1");

    assertThat(result.ok()).isTrue();
}
```

- [ ] **Step 9: Implement channel management + member management**

- `createConversation`: POST `/api/conversations.create` with `{name, is_private}`
- `getConversationInfo`: GET `/api/conversations.info?channel={id}`
- `inviteToConversation`: POST `/api/conversations.invite` with `{channel, users}`
- `kickFromConversation`: POST `/api/conversations.kick` with `{channel, user}`

- [ ] **Step 10: Write failing test — message history**

```java
@Test
void getHistory_success() {
    wireMock.stubFor(get(urlMatching("/api/conversations\\.history.*"))
            .willReturn(okJson("""
                    {"ok":true,"messages":[
                      {"ts":"1234567890.123456","user":"U1","text":"Hello","thread_ts":null},
                      {"ts":"1234567891.654321","user":"U2","text":"Reply","thread_ts":"1234567890.123456"}
                    ]}
                    """)));

    SlackBotClient.HistoryResult result = client.getHistory("tok", "C1", "1234567889.000000", 100);

    assertThat(result.ok()).isTrue();
    assertThat(result.messages()).hasSize(2);
    assertThat(result.messages().get(0).ts()).isEqualTo("1234567890.123456");
    assertThat(result.messages().get(0).text()).isEqualTo("Hello");
    assertThat(result.messages().get(1).threadTs()).isEqualTo("1234567890.123456");
}

@Test
void getHistory_apiError() {
    wireMock.stubFor(get(urlMatching("/api/conversations\\.history.*"))
            .willReturn(okJson("{\"ok\":false,\"error\":\"channel_not_found\"}")));

    SlackBotClient.HistoryResult result = client.getHistory("tok", "C1", "0", 100);

    assertThat(result.ok()).isFalse();
    assertThat(result.error()).isEqualTo("channel_not_found");
}
```

- [ ] **Step 11: Implement getHistory**

GET `/api/conversations.history?channel={c}&oldest={oldest}&limit={limit}`, parse messages array.

- [ ] **Step 12: Run all slack-bot tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl slack-bot -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: ALL PASS

- [ ] **Step 13: Commit**

```
feat(slack-bot): expand SlackBotClient with 12 new Slack Web API methods — Refs #40

Adds listConversations (refactors listChannels to delegate), reactions
(add/remove/get), presence, members, users, channel management
(create/info), member management (invite/kick), and message history.
All follow credential-config-ownership and paginating-client-fail-soft.
```

---

### Task 4: chat-slack module — SlackChatPlatform + SlackInboundTranslator (#40)

**Files:**
- Create: `chat-slack/pom.xml`
- Create: `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackChatPlatform.java`
- Create: `chat-slack/src/main/java/io/casehub/connectors/chat/slack/SlackInboundTranslator.java`
- Create: `chat-slack/src/test/java/io/casehub/connectors/chat/slack/SlackChatPlatformTest.java`
- Create: `chat-slack/src/test/java/io/casehub/connectors/chat/slack/SlackInboundTranslatorTest.java`
- Modify: `pom.xml` (parent) — add `<module>chat-slack</module>`

**Interfaces:**
- Consumes: All `SlackBotClient` methods from Task 3, `ChatPlatform` SPI, `InboundTranslator` SPI, `InboundConnectorTypes.SLACK`
- Produces: `SlackChatPlatform` CDI bean (discoverable at startup), `SlackInboundTranslator` CDI bean

- [ ] **Step 1: Create module scaffold**

Create `chat-slack/pom.xml` mirroring `chat-discord/pom.xml`:

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

  <artifactId>casehub-connectors-chat-slack</artifactId>
  <name>CaseHub Connectors — Chat Slack</name>
  <description>ChatPlatform SPI implementation for Slack. All 9 capabilities native.</description>

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
      <artifactId>casehub-connectors-slack-bot</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
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

Add `<module>chat-slack</module>` to parent `pom.xml` after `<module>chat-discord</module>`.

- [ ] **Step 2: Write SlackInboundTranslator + test**

Create `SlackInboundTranslatorTest.java`:

```java
package io.casehub.connectors.chat.slack;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.Attachment;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ReceivedMessage;

class SlackInboundTranslatorTest {

    private SlackInboundTranslator translator;

    @BeforeEach
    void setUp() {
        translator = new SlackInboundTranslator();
    }

    @Test
    void connectorType_returnsSlack() {
        assertThat(translator.connectorType()).isEqualTo("slack");
    }

    @Test
    void translate_basicMessage() {
        Instant now = Instant.now();
        InboundMessage inbound = new InboundMessage(
                "slack-inbound", "slack", "U123", "C456",
                "Hello world", List.of(), now,
                Map.of("slack-ts", "1234567890.123456", "workspace-id", "T1"),
                null);

        ReceivedMessage received = translator.translate(inbound);

        assertThat(received.platformId()).isEqualTo("slack");
        assertThat(received.channel().id()).isEqualTo("C456");
        assertThat(received.messageRef().messageId()).isEqualTo("1234567890.123456");
        assertThat(received.parentRef()).isNull();
        assertThat(received.sender().id()).isEqualTo("U123");
        assertThat(received.content().text()).isEqualTo("Hello world");
    }

    @Test
    void translate_threadedMessage() {
        InboundMessage inbound = new InboundMessage(
                "slack-inbound", "slack", "U123", "C456",
                "Reply", List.of(), Instant.now(),
                Map.of("slack-ts", "1234567891.654321",
                       "slack-thread-ts", "1234567890.123456"),
                null);

        ReceivedMessage received = translator.translate(inbound);

        assertThat(received.parentRef()).isNotNull();
        assertThat(received.parentRef().messageId()).isEqualTo("1234567890.123456");
    }

    @Test
    void translate_forwardsAttachments() {
        Attachment att = new Attachment("doc.pdf", "application/pdf", new byte[]{1, 2});
        InboundMessage inbound = new InboundMessage(
                "slack-inbound", "slack", "U123", "C456",
                "See attached", List.of(att), Instant.now(),
                Map.of("slack-ts", "ts1"), null);

        ReceivedMessage received = translator.translate(inbound);

        assertThat(received.content().attachments()).hasSize(1);
        assertThat(received.content().attachments().get(0).filename()).isEqualTo("doc.pdf");
    }
}
```

Create `SlackInboundTranslator.java`:

```java
package io.casehub.connectors.chat.slack;

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
public class SlackInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return InboundConnectorTypes.SLACK;
    }

    @Override
    public ReceivedMessage translate(final InboundMessage msg) {
        final var channel = new ChatChannelRef(msg.externalChannelRef());
        final var messageRef = new ChatMessageRef(channel, msg.metadata().get("slack-ts"));
        final String threadTs = msg.metadata().get("slack-thread-ts");
        final ChatMessageRef parentRef = threadTs != null
                ? new ChatMessageRef(channel, threadTs) : null;
        return new ReceivedMessage(
                InboundConnectorTypes.SLACK, channel, messageRef, parentRef,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content(), null, msg.attachments()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 3: Write SlackChatPlatform test + implementation**

Create `SlackChatPlatformTest.java` with WireMock-based tests covering all 9 capabilities. Follow `DiscordChatPlatformTest` structure. Create `SlackChatPlatform.java` as `@ApplicationScoped` CDI bean.

Key implementation points for `SlackChatPlatform`:
- Constructor takes `SlackBotClient`, `@ConfigProperty(name = "casehub.slack.token", defaultValue = "") String token`
- `@PostConstruct init()` — if token blank, degrade all capabilities (same as Discord pattern)
- `id()` returns `InboundConnectorTypes.SLACK`
- `NATIVE_CAPABILITIES = Set.of(all 9 classes)` when token is configured
- Messaging: `client.postMessage(token, channelId, text, null)` → map `PostResult` to `SendResult`
- Threading: `client.postMessage(token, channelId, text, parent.messageId())` — `thread_ts` = parent `ts`
- Discovery: `client.listConversations(token)` → map `ConversationInfo` → `Channel`
- Reactions: delegate to `client.addReaction/removeReaction/getReactions`
- Presence: `client.getPresence(token, userId)` → map `active`→ONLINE, `away`→AWAY. `set()` logs warning, no-op.
- Members: `client.listConversationMembers(token, channelId)` → IDs, `client.listUsers(token)` → map, join locally
- ChannelManagement: `client.createConversation/getConversationInfo` → map to `Channel`
- MemberManagement: `client.inviteToConversation/kickFromConversation`
- MessageHistory: `client.getHistory(token, channelId, oldest, 100)` → map `HistoryMessage` to `ReceivedMessage`
- Slack `ts` → `Instant` parsing: split on `"."`, epoch seconds + microseconds → nanos

The test class should cover:
- `idIsSlack()`
- `supportsNineNativeCapabilities()`
- `messaging_send()`
- `threading_reply()`
- `discovery_listChannels()` (returns `Channel` with topic/purpose/isPrivate)
- `reactions_addRemoveList()`
- `presence_ofMember()` (active→ONLINE, away→AWAY)
- `presence_setLogsWarning()`
- `members_listWithBatchUserFetch()` (verifies local join, fallback to userId as displayName)
- `channelManagement_createAndFind()`
- `memberManagement_addAndRemove()`
- `messageHistory_messagesSince()` (ts precision, threadTs→parentRef)
- `messaging_blankTokenReturnsFailure()`
- `degradedMode_blankToken()`

- [ ] **Step 4: Run all chat-slack tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-slack -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all modules pass

- [ ] **Step 6: Commit**

```
feat(chat-slack): add SlackChatPlatform with 9 native capabilities — Refs #40

New chat-slack module with SlackChatPlatform @ApplicationScoped implementing
all 9 ChatPlatform capabilities natively via SlackBotClient. Includes
SlackInboundTranslator for webhook→ChatSPI translation. Uses
casehub.slack.token config with degraded fallback when unconfigured.
```

---

## Post-Implementation

After all 4 tasks pass and are committed:
1. Run `superpowers:requesting-code-review` — fix Important+, file Minor as GitHub issue
2. Run `implementation-doc-sync` — update ARC42STORIES.MD, PLATFORM.md capability table
3. Update `CLAUDE.md` "What This Project Is" section if needed
