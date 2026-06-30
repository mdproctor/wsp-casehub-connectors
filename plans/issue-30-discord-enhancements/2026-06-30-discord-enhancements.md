# Discord Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Discord attachment downloading (#30), rich embed support (#33), and MCP tools (#34) to casehub-connectors.

**Architecture:** Extends `discord` module (shared HTTP client + models) with attachment and embed types, and `mcp` module with Discord-specific tools. Gateway inbound path downloads attachments on virtual threads. MCP tools follow the SlackBotMcpTool pattern.

**Tech Stack:** Java 21, Quarkus 3.32.2, WireMock (tests), JUnit 5, AssertJ

**Spec:** `specs/issue-30-discord-enhancements/2026-06-30-discord-enhancements-design.md` (design-reviewed, 19 issues resolved)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All HTTP calls via `HttpHelper.CLIENT` (protocol: shared-http-client)
- `@Blocking` on all `@Tool` methods (protocol: mcp-tool-blocking-annotation)
- Credentials passed at call time, not stored on shared clients (protocol: credential-config-ownership)
- Every commit references an issue

---

### Task 1: DiscordAttachment model + parseAttachments + DiscordMessage extension

**Files:**
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordAttachment.java`
- Modify: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordMessage.java`
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java` — add public `parseAttachments()`, update `parseMessage()`
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java` — add attachment parsing tests, fix broken constructors
- Test: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`

**Interfaces:**
- Produces: `DiscordAttachment(String id, String filename, String contentType, long size, String url)` record
- Produces: `DiscordClient.parseAttachments(JsonNode attachmentsArray) → List<DiscordAttachment>` (public)
- Produces: `DiscordMessage` now has 8th field `List<DiscordAttachment> attachments`

- [ ] **Step 1: Create DiscordAttachment record**

```java
// discord/src/main/java/io/casehub/connectors/discord/model/DiscordAttachment.java
package io.casehub.connectors.discord.model;

public record DiscordAttachment(String id, String filename,
        String contentType, long size, String url) {}
```

- [ ] **Step 2: Update DiscordMessage to include attachments**

Add 8th field to the record. This breaks existing constructor calls — fix them in Step 5.

```java
// discord/src/main/java/io/casehub/connectors/discord/model/DiscordMessage.java
package io.casehub.connectors.discord.model;

import java.time.Instant;
import java.util.List;

public record DiscordMessage(String id, String channelId, DiscordUser author,
                             String content, Instant timestamp,
                             String referencedMessageId, int type,
                             List<DiscordAttachment> attachments) {}
```

- [ ] **Step 3: Write failing tests for attachment parsing**

Add to `DiscordClientTest.java`:

```java
@Test
void parseAttachments_parsesArrayWithMultipleAttachments() throws Exception {
    final ObjectMapper mapper = new ObjectMapper();
    final String json = """
            [
              {"id":"att1","filename":"image.png","content_type":"image/png","size":12345,"url":"https://cdn.discordapp.com/attachments/1/2/image.png"},
              {"id":"att2","filename":"doc.pdf","content_type":"application/pdf","size":67890,"url":"https://cdn.discordapp.com/attachments/1/2/doc.pdf"}
            ]""";
    final JsonNode array = mapper.readTree(json);

    final List<DiscordAttachment> result = client.parseAttachments(array);

    assertThat(result).hasSize(2);
    assertThat(result.get(0).id()).isEqualTo("att1");
    assertThat(result.get(0).filename()).isEqualTo("image.png");
    assertThat(result.get(0).contentType()).isEqualTo("image/png");
    assertThat(result.get(0).size()).isEqualTo(12345L);
    assertThat(result.get(0).url()).isEqualTo("https://cdn.discordapp.com/attachments/1/2/image.png");
    assertThat(result.get(1).id()).isEqualTo("att2");
}

@Test
void parseAttachments_emptyArray_returnsEmptyList() throws Exception {
    final ObjectMapper mapper = new ObjectMapper();
    final JsonNode array = mapper.readTree("[]");

    final List<DiscordAttachment> result = client.parseAttachments(array);

    assertThat(result).isEmpty();
}

@Test
void parseAttachments_nullArray_returnsEmptyList() {
    final List<DiscordAttachment> result = client.parseAttachments(null);

    assertThat(result).isEmpty();
}

@Test
void getMessages_includesAttachments() {
    wireMock.stubFor(get(urlPathEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("""
                    [{"id":"msg1","channel_id":"ch1","content":"hello",
                      "timestamp":"2026-06-01T00:00:00Z","type":0,
                      "author":{"id":"u1","username":"user1","bot":false},
                      "attachments":[{"id":"a1","filename":"f.png","content_type":"image/png","size":100,"url":"https://cdn.discordapp.com/a1"}]}]
                    """)));

    final List<DiscordMessage> msgs = client.getMessages(TOKEN, "ch1", null, 100);

    assertThat(msgs).hasSize(1);
    assertThat(msgs.get(0).attachments()).hasSize(1);
    assertThat(msgs.get(0).attachments().get(0).filename()).isEqualTo("f.png");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `parseAttachments` does not exist, `DiscordMessage` constructor calls have wrong arity.

- [ ] **Step 5: Implement parseAttachments and update parseMessage**

In `DiscordClient.java`:
1. Add `public List<DiscordAttachment> parseAttachments(JsonNode attachmentsArray)` method
2. Update `parseMessage()` to call `parseAttachments()` and pass result to `DiscordMessage` constructor
3. Fix all existing test `DiscordMessage` constructors to include the 8th `List.of()` argument

```java
// In DiscordClient.java — new public method
public List<DiscordAttachment> parseAttachments(final JsonNode attachmentsArray) {
    if (attachmentsArray == null || !attachmentsArray.isArray() || attachmentsArray.isEmpty()) {
        return List.of();
    }
    final List<DiscordAttachment> result = new ArrayList<>();
    for (final JsonNode node : attachmentsArray) {
        result.add(new DiscordAttachment(
                node.get("id").asText(),
                node.has("filename") ? node.get("filename").asText() : null,
                node.has("content_type") ? node.get("content_type").asText() : null,
                node.has("size") ? node.get("size").asLong() : 0,
                node.has("url") ? node.get("url").asText() : null));
    }
    return List.copyOf(result);
}
```

Update `parseMessage()` — add attachment parsing before the return:
```java
final List<DiscordAttachment> attachments = parseAttachments(
        node.has("attachments") ? node.get("attachments") : null);
// ...include attachments as 8th arg in DiscordMessage constructor
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest`
Expected: All tests PASS

- [ ] **Step 7: Fix downstream compilation — chat-discord module**

`DiscordChatPlatform.toReceivedMessage()` uses `dm.content()`, `dm.id()` etc. — accessor-based, no constructor call. Verify it compiles:

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl chat-discord`

If any test files in `chat-discord` construct `DiscordMessage` directly, add the 8th `List.of()` argument.

- [ ] **Step 8: Commit**

```
feat(discord): add DiscordAttachment model and parseAttachments — Refs #30

DiscordMessage now carries List<DiscordAttachment> metadata.
DiscordClient.parseAttachments() is public for cross-package use.
```

---

### Task 2: downloadAttachment on DiscordClient

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java` — add `downloadAttachment()`, `guildId()`, config property
- Test: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`

**Interfaces:**
- Consumes: `DiscordAttachment` from Task 1
- Consumes: `Attachment` from `core` (already a dependency)
- Consumes: `HttpHelper.CLIENT` from `core`
- Produces: `DiscordClient.downloadAttachment(DiscordAttachment) → Attachment` (nullable)
- Produces: `DiscordClient.guildId() → String`

- [ ] **Step 1: Write failing tests for downloadAttachment**

Add to `DiscordClientTest.java`. These tests use a second WireMock for CDN simulation (the existing wireMock simulates the Discord API; CDN is a different host).

```java
@Test
void downloadAttachment_success() {
    final byte[] content = "hello world".getBytes();
    wireMock.stubFor(get(urlPathEqualTo("/cdn/file.png"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Length", String.valueOf(content.length))
                    .withBody(content)));

    final var att = new DiscordAttachment("a1", "file.png", "image/png",
            content.length, wireMock.baseUrl() + "/cdn/file.png");
    // Override allowed hosts for test — production validates cdn.discordapp.com
    final Attachment result = client.downloadAttachment(att,
            Set.of("localhost"));

    assertThat(result).isNotNull();
    assertThat(result.filename()).isEqualTo("file.png");
    assertThat(result.contentType()).isEqualTo("image/png");
    assertThat(result.content()).isEqualTo(content);
}

@Test
void downloadAttachment_ssrfRejected_nonAllowedHost() {
    final var att = new DiscordAttachment("a1", "f.png", "image/png",
            100, "https://evil.com/payload");

    final Attachment result = client.downloadAttachment(att);

    assertThat(result).isNull();
}

@Test
void downloadAttachment_contentLengthExceedsLimit_returnsNull() {
    wireMock.stubFor(get(urlPathEqualTo("/cdn/huge.bin"))
            .willReturn(aResponse().withStatus(200)
                    .withHeader("Content-Length", "999999999")));

    final var att = new DiscordAttachment("a1", "huge.bin", "application/octet-stream",
            999999999, wireMock.baseUrl() + "/cdn/huge.bin");
    final Attachment result = client.downloadAttachment(att,
            Set.of("localhost"));

    assertThat(result).isNull();
}

@Test
void downloadAttachment_cdn403_returnsNull() {
    wireMock.stubFor(get(urlPathEqualTo("/cdn/expired.png"))
            .willReturn(aResponse().withStatus(403)));

    final var att = new DiscordAttachment("a1", "expired.png", "image/png",
            100, wireMock.baseUrl() + "/cdn/expired.png");
    final Attachment result = client.downloadAttachment(att,
            Set.of("localhost"));

    assertThat(result).isNull();
}

@Test
void downloadAttachment_streamingAbortOnOversizedChunkedResponse() {
    // No Content-Length header — chunked transfer
    final byte[] oversized = new byte[1024 * 1024 + 1]; // 1MB + 1 byte
    wireMock.stubFor(get(urlPathEqualTo("/cdn/chunked.bin"))
            .willReturn(aResponse().withStatus(200).withBody(oversized)));

    client.maxAttachmentBytes = 1024 * 1024; // 1MB limit for this test
    final var att = new DiscordAttachment("a1", "chunked.bin", "application/octet-stream",
            0, wireMock.baseUrl() + "/cdn/chunked.bin");
    final Attachment result = client.downloadAttachment(att,
            Set.of("localhost"));

    assertThat(result).isNull();
}

@Test
void guildId_returnsConfiguredValue() {
    assertThat(client.guildId()).isEqualTo(GUILD_ID);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest`
Expected: Compilation failure — `downloadAttachment` and `guildId` do not exist.

- [ ] **Step 3: Implement downloadAttachment and guildId**

In `DiscordClient.java`:

Add config property:
```java
@ConfigProperty(name = "casehub.discord.attachment.max-bytes",
                defaultValue = "8388608")
long maxAttachmentBytes;
```

Add allowed CDN hosts constant:
```java
private static final Set<String> ALLOWED_CDN_HOSTS = Set.of(
        "cdn.discordapp.com", "media.discordapp.net");
```

Add `guildId()` accessor:
```java
public String guildId() { return guildId; }
```

Add `downloadAttachment` with two signatures (production and test-overridable host set):
```java
public Attachment downloadAttachment(final DiscordAttachment attachment) {
    return downloadAttachment(attachment, ALLOWED_CDN_HOSTS);
}

Attachment downloadAttachment(final DiscordAttachment attachment,
                              final Set<String> allowedHosts) {
    // 1. SSRF validation
    final URI uri;
    try {
        uri = URI.create(attachment.url());
    } catch (final Exception e) {
        LOG.warning("DiscordClient: invalid attachment URL — " + attachment.url());
        return null;
    }
    if (!allowedHosts.contains(uri.getHost())) {
        LOG.warning("DiscordClient: attachment URL host rejected (SSRF defense) — " + uri.getHost());
        return null;
    }

    try {
        final HttpRequest request = HttpRequest.newBuilder()
                .uri(uri).timeout(REQUEST_TIMEOUT).GET().build();
        final HttpResponse<InputStream> response = HttpHelper.CLIENT.send(
                request, HttpResponse.BodyHandlers.ofInputStream());

        if (response.statusCode() == 403) {
            LOG.warning("DiscordClient: CDN 403 (expired URL) — " + attachment.filename());
            return null;
        }
        if (response.statusCode() != 200) {
            LOG.warning("DiscordClient: attachment download HTTP " + response.statusCode());
            return null;
        }

        // 2. Content-Length pre-check
        final long contentLength = response.headers()
                .firstValueAsLong("Content-Length").orElse(-1);
        if (contentLength > maxAttachmentBytes) {
            LOG.warning(String.format(
                    "DiscordClient: attachment too large (%d bytes, limit %d) — %s",
                    contentLength, maxAttachmentBytes, attachment.filename()));
            return null;
        }

        // 3. Streaming byte-count enforcement
        try (final InputStream in = response.body()) {
            final ByteArrayOutputStream out = new ByteArrayOutputStream();
            final byte[] buf = new byte[8192];
            long total = 0;
            int read;
            while ((read = in.read(buf)) != -1) {
                total += read;
                if (total > maxAttachmentBytes) {
                    LOG.warning(String.format(
                            "DiscordClient: attachment stream exceeded limit (%d bytes) — %s",
                            maxAttachmentBytes, attachment.filename()));
                    return null;
                }
                out.write(buf, 0, read);
            }
            return new Attachment(attachment.filename(),
                    attachment.contentType(), out.toByteArray());
        }
    } catch (final InterruptedException e) {
        Thread.currentThread().interrupt();
        return null;
    } catch (final Exception e) {
        LOG.warning("DiscordClient: attachment download error — " + e.getMessage());
        return null;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
feat(discord): add downloadAttachment with SSRF defense and streaming size limit — Refs #30

- URL host validation (cdn.discordapp.com, media.discordapp.net)
- Content-Length pre-check + streaming byte-count abort
- CDN 403 treated as non-retryable (expired signed URL)
- Configurable via casehub.discord.attachment.max-bytes (default 8MB)
```

---

### Task 3: DiscordInboundConnector attachment integration

**Files:**
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundConnector.java`
- Create: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInboundConnectorTest.java`

**Interfaces:**
- Consumes: `DiscordClient.parseAttachments(JsonNode)` from Task 1
- Consumes: `DiscordClient.downloadAttachment(DiscordAttachment)` from Task 2
- Consumes: `InboundMessageSink.receive(InboundMessage)` from core
- Produces: `InboundMessage` with populated `attachments` list and metadata keys `discord-attachment-count`, `discord-attachment-download-failures`

- [ ] **Step 1: Write failing tests**

Create `DiscordInboundConnectorTest.java`. Test the connector by calling `handleMessageCreate()` via `start()` + simulated Gateway events. Use WireMock for the Discord API (getGatewayUrl) and CDN (attachment downloads). Use a recording `InboundMessageSink` to capture results.

Key test cases:
1. `messageWithAttachments_downloadsAndPopulatesInboundMessage` — MESSAGE_CREATE with 1 attachment, CDN returns 200 → InboundMessage has 1 Attachment
2. `messageWithAttachments_partialFailure_includesSuccessfulDownloads` — 2 attachments, one CDN 403 → InboundMessage has 1 Attachment, metadata shows 1 failure
3. `messageWithNoAttachments_emptyAttachmentList` — MESSAGE_CREATE with empty attachments → InboundMessage has 0 attachments, no attachment metadata
4. `messageWithAttachments_metadata_includesCountAndFailures` — verify `discord-attachment-count` and `discord-attachment-download-failures` metadata keys

The test should construct JSON payloads matching Discord MESSAGE_CREATE format and invoke the connector's event handling. Since `handleMessageCreate` is private, the test exercises it via the `GatewayEventListener` callback — create the connector, mock `DiscordClient` methods directly (using constructor injection or field injection in test), and call the listener.

Alternative approach: since `DiscordInboundConnector` uses `DiscordClient` (CDI bean), the test constructs a connector with a real DiscordClient pointed at WireMock, stubs both the Gateway URL and CDN attachment endpoints.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest=DiscordInboundConnectorTest`
Expected: FAIL — current implementation passes `List.of()` for attachments.

- [ ] **Step 3: Implement attachment handling in handleMessageCreate**

In `DiscordInboundConnector.handleMessageCreate()`:

```java
private void handleMessageCreate(final JsonNode data, final InboundMessageSink sink) {
    // ... existing filtering (bot, type) ...

    // Parse attachments metadata
    final List<DiscordAttachment> discordAttachments =
            client.parseAttachments(data.get("attachments"));

    if (discordAttachments.isEmpty()) {
        // No attachments — stay on event loop (non-blocking path)
        deliverMessage(data, List.of(), 0, 0, sink);
    } else {
        // Offload to virtual thread — downloads are blocking
        final JsonNode capturedData = data;
        Thread.ofVirtual().name("discord-attachment-download").start(() ->
                downloadAndDeliver(capturedData, discordAttachments, sink));
    }
}

private void downloadAndDeliver(final JsonNode data,
        final List<DiscordAttachment> discordAttachments,
        final InboundMessageSink sink) {
    final List<Attachment> downloaded = new ArrayList<>();
    int failures = 0;
    for (final DiscordAttachment da : discordAttachments) {
        final Attachment att = client.downloadAttachment(da);
        if (att != null) {
            downloaded.add(att);
        } else {
            failures++;
        }
    }
    deliverMessage(data, downloaded, discordAttachments.size(), failures, sink);
}

private void deliverMessage(final JsonNode data, final List<Attachment> attachments,
        final int attachmentCount, final int downloadFailures,
        final InboundMessageSink sink) {
    // ... existing field extraction (messageId, channelId, content, senderId, metadata) ...
    
    if (attachmentCount > 0) {
        metadata.put("discord-attachment-count", String.valueOf(attachmentCount));
        metadata.put("discord-attachment-download-failures", String.valueOf(downloadFailures));
    }
    
    final InboundMessage msg = new InboundMessage(
            InboundConnectorIds.DISCORD_INBOUND,
            InboundConnectorTypes.DISCORD,
            senderId, channelId, content,
            attachments,  // was List.of()
            Instant.now(), metadata, null);
    
    try {
        sink.receive(msg);
    } catch (final Exception e) {
        LOG.log(Level.SEVERE, "discord-inbound: sink threw", e);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
feat(discord): download Gateway attachments on virtual threads — Closes #30

- Parse attachments[] from MESSAGE_CREATE, download each via CDN
- Virtual thread offloading prevents Vert.x event loop blocking
- Partial download failures logged + skipped (partial results preferred)
- Metadata: discord-attachment-count, discord-attachment-download-failures
```

---

### Task 4: DiscordEmbed model + DiscordClient embed overloads

**Files:**
- Create: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordEmbed.java`
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java` — add embed overloads, `buildMessageBody()` helper
- Test: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`

**Interfaces:**
- Produces: `DiscordEmbed` record with nested `Field`, `Footer`, `Author`
- Produces: `DiscordClient.sendMessage(token, channelId, content, List<DiscordEmbed> embeds) → PostResult`
- Produces: `DiscordClient.sendReply(token, channelId, content, replyToMessageId, List<DiscordEmbed> embeds) → PostResult`
- Existing 3-arg `sendMessage` and 4-arg `sendReply` delegate to new overloads with `List.of()`

- [ ] **Step 1: Create DiscordEmbed record**

```java
// discord/src/main/java/io/casehub/connectors/discord/model/DiscordEmbed.java
package io.casehub.connectors.discord.model;

import java.util.List;

public record DiscordEmbed(
        String title, String description, String url, Integer color,
        List<Field> fields, String thumbnailUrl, String imageUrl,
        Footer footer, Author author) {

    public record Field(String name, String value, boolean inline) {}
    public record Footer(String text) {}
    public record Author(String name) {}

    public DiscordEmbed {
        fields = fields == null ? List.of() : List.copyOf(fields);
    }
}
```

- [ ] **Step 2: Write failing tests for embed sending**

Add to `DiscordClientTest.java`:

```java
@Test
void sendMessage_withEmbed_includesEmbedsArray() {
    wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    final var embed = new DiscordEmbed("Title", "Desc", null, 16711680,
            List.of(new DiscordEmbed.Field("f1", "v1", true)),
            null, null, new DiscordEmbed.Footer("foot"), null);

    final PostResult result = client.sendMessage(TOKEN, "ch1", "text", List.of(embed));

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.content", equalTo("text")))
            .withRequestBody(matchingJsonPath("$.embeds[0].title", equalTo("Title")))
            .withRequestBody(matchingJsonPath("$.embeds[0].description", equalTo("Desc")))
            .withRequestBody(matchingJsonPath("$.embeds[0].color", equalTo("16711680")))
            .withRequestBody(matchingJsonPath("$.embeds[0].fields[0].name", equalTo("f1")))
            .withRequestBody(matchingJsonPath("$.embeds[0].footer.text", equalTo("foot"))));
}

@Test
void sendMessage_embedOnly_noContentField() {
    wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    final var embed = new DiscordEmbed("Title", "Desc", null, null,
            List.of(), null, null, null, null);

    final PostResult result = client.sendMessage(TOKEN, "ch1", null, List.of(embed));

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.embeds[0].title", equalTo("Title")))
            .withoutRequestBody(matchingJsonPath("$.content")));
}

@Test
void sendReply_withEmbed_includesEmbedAndReference() {
    wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg2\",\"channel_id\":\"ch1\"}")));

    final var embed = new DiscordEmbed("T", "D", null, 65280,
            List.of(), null, null, null, null);

    final PostResult result = client.sendReply(TOKEN, "ch1", "reply",
            "parent1", List.of(embed));

    assertThat(result.ok()).isTrue();
    wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.embeds[0].title", equalTo("T")))
            .withRequestBody(matchingJsonPath("$.message_reference.message_id",
                    equalTo("parent1"))));
}

@Test
void sendMessage_existingThreeArgDelegates_noEmbeds() {
    wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
            .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

    client.sendMessage(TOKEN, "ch1", "text");

    wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
            .withRequestBody(matchingJsonPath("$.content", equalTo("text")))
            .withoutRequestBody(matchingJsonPath("$.embeds")));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest`
Expected: Compilation failure — new overloads don't exist.

- [ ] **Step 4: Implement embed overloads and buildMessageBody**

In `DiscordClient.java`:

1. Convert existing `sendMessage(token, channelId, content)` to delegate:
```java
public PostResult sendMessage(final String token, final String channelId,
                              final String content) {
    return sendMessage(token, channelId, content, List.of());
}
```

2. New overload:
```java
public PostResult sendMessage(final String token, final String channelId,
                              final String content, final List<DiscordEmbed> embeds) {
    try {
        final ObjectNode body = buildMessageBody(content, embeds);
        final HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(apiBaseUrl + "/channels/" + channelId + "/messages"))
                .header("Authorization", "Bot " + token)
                .header("Content-Type", "application/json")
                .timeout(REQUEST_TIMEOUT)
                .POST(HttpRequest.BodyPublishers.ofString(mapper.writeValueAsString(body)))
                .build();
        return sendWithRetry(request);
    } catch (final Exception e) {
        LOG.warning("DiscordClient: sendMessage error — " + e.getMessage());
        return PostResult.failure("json-error");
    }
}
```

3. Same pattern for `sendReply` — delegate 4-arg to 5-arg, add `message_reference` to body.

4. Private `buildMessageBody`:
```java
private ObjectNode buildMessageBody(final String content,
                                    final List<DiscordEmbed> embeds) {
    final ObjectNode body = mapper.createObjectNode();
    if (content != null && !content.isEmpty()) {
        body.put("content", content);
    }
    if (embeds != null && !embeds.isEmpty()) {
        final ArrayNode embedsArray = body.putArray("embeds");
        for (final DiscordEmbed embed : embeds) {
            serializeEmbed(embedsArray.addObject(), embed);
        }
    }
    return body;
}

private void serializeEmbed(final ObjectNode node, final DiscordEmbed embed) {
    if (embed.title() != null) node.put("title", embed.title());
    if (embed.description() != null) node.put("description", embed.description());
    if (embed.url() != null) node.put("url", embed.url());
    if (embed.color() != null) node.put("color", embed.color());
    if (!embed.fields().isEmpty()) {
        final ArrayNode fields = node.putArray("fields");
        for (final DiscordEmbed.Field f : embed.fields()) {
            final ObjectNode fn = fields.addObject();
            fn.put("name", f.name());
            fn.put("value", f.value());
            fn.put("inline", f.inline());
        }
    }
    if (embed.thumbnailUrl() != null)
        node.putObject("thumbnail").put("url", embed.thumbnailUrl());
    if (embed.imageUrl() != null)
        node.putObject("image").put("url", embed.imageUrl());
    if (embed.footer() != null)
        node.putObject("footer").put("text", embed.footer().text());
    if (embed.author() != null)
        node.putObject("author").put("name", embed.author().name());
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest`
Expected: All tests PASS

- [ ] **Step 6: Full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord,chat-discord`
Expected: All tests PASS — existing callers use the 3-arg/4-arg delegating methods.

- [ ] **Step 7: Commit**

```
feat(discord): add rich embed support to DiscordClient — Closes #33

- DiscordEmbed record with Field, Footer, Author nested types
- sendMessage/sendReply overloads accepting List<DiscordEmbed>
- Existing 3-arg/4-arg methods delegate with List.of()
- buildMessageBody extracts shared JSON body construction
```

---

### Task 5: DiscordMcpTool — send_discord + list_discord_channels

**Files:**
- Modify: `mcp/pom.xml` — add `casehub-connectors-discord` dependency
- Create: `mcp/src/main/java/io/casehub/connectors/mcp/DiscordMcpTool.java`
- Create: `mcp/src/test/java/io/casehub/connectors/mcp/DiscordMcpToolTest.java`

**Interfaces:**
- Consumes: `DiscordClient.sendMessage(token, channelId, content, List<DiscordEmbed>)` from Task 4
- Consumes: `DiscordClient.sendReply(token, channelId, content, replyToMessageId, List<DiscordEmbed>)` from Task 4
- Consumes: `DiscordClient.listGuildChannels(token)` (existing)
- Consumes: `DiscordClient.guildId()` from Task 2
- Consumes: `ConnectorMeshBridge.notifyDelivered()` from core
- Consumes: `McpContentSanitizer.sanitize()` (package-private in mcp)
- Consumes: `DiscordDiscovery.ID` ("discord") from discord module

- [ ] **Step 1: Add discord dependency to mcp/pom.xml**

Add to `mcp/pom.xml` `<dependencies>`:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-connectors-discord</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

- [ ] **Step 2: Write failing tests**

Create `mcp/src/test/java/io/casehub/connectors/mcp/DiscordMcpToolTest.java`:

```java
package io.casehub.connectors.mcp;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.*;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import io.casehub.connectors.discord.DiscordClient;
import io.casehub.connectors.discord.DiscordDiscovery;

class DiscordMcpToolTest {

    private WireMockServer wireMock;
    private DiscordClient client;
    private McpToolTestSupport.RecordingBridge bridge;
    private DiscordMcpTool tool;

    @BeforeEach
    void start() {
        wireMock = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMock.start();
        client = new DiscordClient();
        client.apiBaseUrl = wireMock.baseUrl();
        client.guildId = "guild1";
        bridge = new McpToolTestSupport.RecordingBridge();
        tool = new DiscordMcpTool(client, bridge, "test-token");
    }

    @AfterEach
    void stop() { wireMock.stop(); }

    // send_discord tests
    @Test
    void sendDiscord_success_returnsPostedWithId() {
        wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
                .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

        final String result = tool.sendDiscord("ch1", "Hello", null, null, null, null);

        assertThat(result).isEqualTo("Posted to ch1 (id=msg1)");
    }

    @Test
    void sendDiscord_success_bridgeCalled() {
        wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
                .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

        tool.sendDiscord("ch1", "Hello", null, null, null, null);

        assertThat(bridge.lastConnectorId).isEqualTo(DiscordDiscovery.ID);
        assertThat(bridge.lastDestination).isEqualTo("ch1");
        assertThat(bridge.lastContent).isEqualTo("Hello");
    }

    @Test
    void sendDiscord_blankToken_returnsFailedNoBridgeCall() {
        final var blankTool = new DiscordMcpTool(client, bridge, "");
        final String result = blankTool.sendDiscord("ch1", "Hello", null, null, null, null);

        assertThat(result).isEqualTo("Failed: casehub.connectors.discord.token is not configured");
        assertThat(bridge.lastConnectorId).isNull();
    }

    @Test
    void sendDiscord_withReplyToMessageId_callsSendReply() {
        wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
                .willReturn(okJson("{\"id\":\"msg2\",\"channel_id\":\"ch1\"}")));

        tool.sendDiscord("ch1", "reply", "parent1", null, null, null);

        wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
                .withRequestBody(matchingJsonPath("$.message_reference.message_id",
                        equalTo("parent1"))));
    }

    @Test
    void sendDiscord_withEmbedArgs_sendsEmbed() {
        wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
                .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

        tool.sendDiscord("ch1", "text", null, "Embed Title", "Embed Desc", "16711680");

        wireMock.verify(postRequestedFor(urlEqualTo("/channels/ch1/messages"))
                .withRequestBody(matchingJsonPath("$.embeds[0].title", equalTo("Embed Title")))
                .withRequestBody(matchingJsonPath("$.embeds[0].color", equalTo("16711680"))));
    }

    @Test
    void sendDiscord_embedOnly_noTextRequired() {
        wireMock.stubFor(post(urlEqualTo("/channels/ch1/messages"))
                .willReturn(okJson("{\"id\":\"msg1\",\"channel_id\":\"ch1\"}")));

        final String result = tool.sendDiscord("ch1", null, null, "Title", null, null);

        assertThat(result).startsWith("Posted to");
        assertThat(bridge.lastContent).isEqualTo("Title");
    }

    @Test
    void sendDiscord_noTextNoEmbed_returnsFailed() {
        final String result = tool.sendDiscord("ch1", null, null, null, null, null);

        assertThat(result).isEqualTo("Failed: text or embed required");
    }

    // list_discord_channels tests
    @Test
    void listDiscordChannels_success() {
        wireMock.stubFor(get(urlPathEqualTo("/guilds/guild1/channels"))
                .willReturn(okJson("""
                        [{"id":"ch1","name":"general","topic":"Main channel","type":0,
                          "permission_overwrites":[]},
                         {"id":"ch2","name":"announcements","topic":null,"type":5,
                          "permission_overwrites":[]}]
                        """)));

        final String result = tool.listDiscordChannels();

        assertThat(result).contains("general").contains("ch1").contains("announcements");
    }

    @Test
    void listDiscordChannels_blankToken_returnsFailed() {
        final var blankTool = new DiscordMcpTool(client, bridge, "");
        assertThat(blankTool.listDiscordChannels())
                .isEqualTo("Failed: casehub.connectors.discord.token is not configured");
    }

    @Test
    void listDiscordChannels_blankGuildId_returnsFailed() {
        client.guildId = "";
        assertThat(tool.listDiscordChannels())
                .isEqualTo("Failed: casehub.discord.guild-id is not configured");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=DiscordMcpToolTest`
Expected: Compilation failure — `DiscordMcpTool` does not exist.

- [ ] **Step 4: Implement DiscordMcpTool**

Create `mcp/src/main/java/io/casehub/connectors/mcp/DiscordMcpTool.java`:

```java
package io.casehub.connectors.mcp;

import java.util.List;
import java.util.Set;

import org.jboss.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.smallrye.common.annotation.Blocking;

import io.casehub.connectors.ConnectorMeshBridge;
import io.casehub.connectors.discord.DiscordClient;
import io.casehub.connectors.discord.DiscordDiscovery;
import io.casehub.connectors.discord.model.DiscordEmbed;
import io.casehub.connectors.discord.model.PostResult;

@ApplicationScoped
public class DiscordMcpTool {

    private static final Logger LOG = Logger.getLogger(DiscordMcpTool.class);
    private static final Set<Integer> TEXT_CHANNEL_TYPES = Set.of(0, 5, 10, 11, 12);

    private final DiscordClient client;
    private final ConnectorMeshBridge meshBridge;
    private final String token;

    @Inject
    public DiscordMcpTool(final DiscordClient client,
                          final ConnectorMeshBridge meshBridge,
                          @ConfigProperty(name = "casehub.connectors.discord.token",
                                          defaultValue = "") final String token) {
        this.client = client;
        this.meshBridge = meshBridge;
        this.token = token;
    }

    @Tool(name = "send_discord",
          description = "Posts a message to a Discord channel via bot token. "
                      + "Returns 'Posted to <channel> (id=<messageId>)' on success "
                      + "or 'Failed: <reason>' on error. At least text or an embed "
                      + "arg is required. Use list_discord_channels to discover IDs.")
    @Blocking
    public String sendDiscord(
            @ToolArg(description = "Discord channel ID (snowflake).")
            final String channel,
            @ToolArg(description = "Message text (max 2000 chars). "
                                 + "Optional if embed args are provided.",
                     required = false)
            final String text,
            @ToolArg(description = "Discord message ID to reply to. "
                                 + "Use discord-message-id from inbound metadata.",
                     required = false)
            final String replyToMessageId,
            @ToolArg(description = "Embed title.", required = false)
            final String embedTitle,
            @ToolArg(description = "Embed description.", required = false)
            final String embedDescription,
            @ToolArg(description = "Embed color as decimal integer RGB "
                                 + "(e.g. 16711680 for red #FF0000, "
                                 + "65280 for green #00FF00).",
                     required = false)
            final String embedColor) {
        try {
            if (token.isBlank()) {
                return "Failed: casehub.connectors.discord.token is not configured";
            }

            final boolean hasText = text != null && !text.isBlank();
            final boolean hasEmbed = (embedTitle != null && !embedTitle.isBlank())
                    || (embedDescription != null && !embedDescription.isBlank());

            if (!hasText && !hasEmbed) {
                return "Failed: text or embed required";
            }

            final List<DiscordEmbed> embeds;
            if (hasEmbed) {
                final Integer color = embedColor != null && !embedColor.isBlank()
                        ? Integer.parseInt(embedColor) : null;
                embeds = List.of(new DiscordEmbed(
                        embedTitle, embedDescription, null, color,
                        List.of(), null, null, null, null));
            } else {
                embeds = List.of();
            }

            final PostResult result;
            if (replyToMessageId != null && !replyToMessageId.isBlank()) {
                result = client.sendReply(token, channel,
                        hasText ? text : null, replyToMessageId, embeds);
            } else {
                result = client.sendMessage(token, channel,
                        hasText ? text : null, embeds);
            }

            if (!result.ok()) {
                return "Failed: " + result.error();
            }

            // Bridge content: text first, fall back to embed title/description
            final String bridgeContent;
            if (hasText) {
                bridgeContent = McpContentSanitizer.sanitize(text);
            } else if (embedTitle != null && !embedTitle.isBlank()) {
                bridgeContent = McpContentSanitizer.sanitize(embedTitle);
            } else {
                bridgeContent = McpContentSanitizer.sanitize(embedDescription);
            }
            meshBridge.notifyDelivered(DiscordDiscovery.ID, channel, bridgeContent);

            return "Posted to " + channel + " (id=" + result.messageId() + ")";
        } catch (final Exception e) {
            LOG.warnf("send_discord failed [%s]: %s",
                    e.getClass().getSimpleName(), e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }

    @Tool(name = "list_discord_channels",
          description = "Lists text channels in the configured Discord guild "
                      + "with name, ID, topic, and type. For Discord-specific "
                      + "detail use this tool. For a cross-platform overview "
                      + "across all connectors, use list_channels.")
    @Blocking
    public String listDiscordChannels() {
        if (token.isBlank()) {
            return "Failed: casehub.connectors.discord.token is not configured";
        }
        if (client.guildId() == null || client.guildId().isBlank()) {
            return "Failed: casehub.discord.guild-id is not configured";
        }

        try {
            final var channels = client.listGuildChannels(token);
            if (channels == null || channels.isEmpty()) {
                return "No channels found.";
            }

            final StringBuilder sb = new StringBuilder();
            for (final var ch : channels) {
                if (!TEXT_CHANNEL_TYPES.contains(ch.type())) continue;
                sb.append("#").append(ch.name())
                  .append(" (").append(ch.id()).append(")");
                if (ch.topic() != null && !ch.topic().isBlank()) {
                    sb.append(" — ").append(ch.topic());
                }
                sb.append("\n");
            }
            return sb.isEmpty() ? "No text channels found." : sb.toString().stripTrailing();
        } catch (final Exception e) {
            LOG.warnf("list_discord_channels failed: %s", e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp -Dtest=DiscordMcpToolTest`
Expected: All tests PASS

- [ ] **Step 6: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: All modules compile and all tests pass.

- [ ] **Step 7: Commit**

```
feat(mcp): add send_discord and list_discord_channels MCP tools — Closes #34

- send_discord: text, reply, embed support with ConnectorMeshBridge
- list_discord_channels: richer output than generic list_channels
- Credential ownership: casehub.connectors.discord.token
- @Blocking on all @Tool methods per protocol
```

---

## Verification

After all tasks complete:

1. `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` — all modules green
2. `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo` — demo module green
3. Review ARC42STORIES.MD for updates needed (§4, §5, §12)
4. Code review via `superpowers:requesting-code-review`
5. Doc sync via `implementation-doc-sync`
