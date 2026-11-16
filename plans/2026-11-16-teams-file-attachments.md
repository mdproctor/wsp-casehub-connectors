# Teams File Attachment Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #229 — chat-teams: file attachment support in toMessageBody

**Goal:** Upload `ChatContent.attachments()` to the Teams channel's SharePoint drive and reference them as `"reference"` attachments in the message body.

**Architecture:** `TeamsChatPlatform.sendMessage/replyToMessage` performs a pre-send upload step (get the channel's filesFolder, upload each attachment, collect `UploadedRef` values) before calling `TeamsMapper.toMessageBody(content, uploadedRefs)`. All upload logic lives in `TeamsClient` using the existing `MicrosoftGraphClient`; `TeamsMapper` stays a pure static mapper.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jakarta JSON-P, `MicrosoftGraphClient`/`GraphErrorHandler` from `microsoft-graph-client`, `HttpHelper.CLIENT` for pre-authenticated chunk PUTs.

## Global Constraints

- No new Maven dependencies — all within `casehub-connectors-chat-teams`
- Simple upload: ≤4MB (`content.length <= 4 * 1024 * 1024`) — PUT to Graph drive endpoint
- Resumable upload: >4MB — `createUploadSession` POST then chunked PUTs of `16 * 320 * 1024` bytes
- Resumable chunk PUTs must NOT carry `Authorization` header — uploadUrl is pre-authenticated; build requests directly via `HttpHelper.CLIENT`, not via any wrapper that adds the token
- Filename: null-safe (generate `"attachment-{i}.bin"` or typed extension), index-prefixed (`"{i}-{name}"`), URL-encoded before embedding in Graph path
- Duplicate filename collision avoided by the index prefix
- Upload failure fails the whole send — `ProviderApiException` propagates, no partial delivery
- Application token used for all upload calls (same as existing `sendMessage`/`replyToMessage`)
- `UploadedRef` is package-private — not part of any SPI

---

## File Map

| File | Action |
|------|--------|
| `chat-teams/src/main/java/io/casehub/connectors/chat/teams/UploadedRef.java` | Create — package-private record |
| `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsMapper.java` | Modify — add 2-arg `toMessageBody`, make 1-arg delegate |
| `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java` | Modify — add `getChannelFilesFolder`, `uploadToChannelFolder` |
| `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java` | Modify — pre-upload in `sendMessage` and `replyToMessage` |
| `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsMapperTest.java` | Modify — add uploaded-ref tests |
| `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsClientTest.java` | Create — new test class |
| `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java` | Modify — add attachment send tests |
| `docs/guides/consumer-guide.md` | Modify — add `Files.ReadWrite.All` permission note |

---

## Batch 1: Pure types and mapper extension

After this batch: `UploadedRef` exists; `toMessageBody` can render file reference attachments; all existing mapper tests pass unchanged.

### Task 1: UploadedRef + TeamsMapper two-arg toMessageBody

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/UploadedRef.java`
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsMapper.java`
- Modify: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsMapperTest.java`

**Interfaces:**
- Produces: `record UploadedRef(String name, String contentUrl)` (package-private)
- Produces: `static JsonObject TeamsMapper.toMessageBody(ChatContent content, List<UploadedRef> uploadedRefs)` — existing 1-arg method becomes a delegate to this

- [ ] **Step 1: Write failing tests in TeamsMapperTest**

Add these three tests at the end of `TeamsMapperTest.java` (before the closing `}`):

Use `ide_insert_member` targeting `TeamsMapperTest` to add:
```java
@Test
void toMessageBodyWithUploadedRefAddsAttachment() {
    var ref     = new UploadedRef("0-report.pdf", "https://sharepoint.com/files/0-report.pdf");
    var content = new ChatContent("See attachment");
    var body    = TeamsMapper.toMessageBody(content, List.of(ref));
    assertThat(body.getJsonObject("body").getString("contentType")).isEqualTo("html");
    assertThat(body.getJsonObject("body").getString("content"))
            .contains("<attachment id=\"file-0\">");
    var attachments = body.getJsonArray("attachments");
    assertThat(attachments).hasSize(1);
    var att = attachments.getJsonObject(0);
    assertThat(att.getString("id")).isEqualTo("file-0");
    assertThat(att.getString("contentType")).isEqualTo("reference");
    assertThat(att.getString("contentUrl")).isEqualTo("https://sharepoint.com/files/0-report.pdf");
    assertThat(att.getString("name")).isEqualTo("0-report.pdf");
}

@Test
void toMessageBodyWithMultipleRefsGeneratesUniqueIds() {
    var refs = List.of(
            new UploadedRef("0-a.pdf", "https://sharepoint.com/files/0-a.pdf"),
            new UploadedRef("1-b.png", "https://sharepoint.com/files/1-b.png"));
    var content = new ChatContent("See files");
    var body    = TeamsMapper.toMessageBody(content, refs);
    var html    = body.getJsonObject("body").getString("content");
    assertThat(html).contains("<attachment id=\"file-0\">").contains("<attachment id=\"file-1\">");
    assertThat(body.getJsonArray("attachments")).hasSize(2);
    assertThat(body.getJsonArray("attachments").getJsonObject(0).getString("id")).isEqualTo("file-0");
    assertThat(body.getJsonArray("attachments").getJsonObject(1).getString("id")).isEqualTo("file-1");
}

@Test
void toMessageBodyWithRefsAndCardsBuildsUnifiedAttachmentsArray() {
    var card = RichCard.builder().title("Title").build();
    var ref  = new UploadedRef("0-doc.pdf", "https://sharepoint.com/files/0-doc.pdf");
    var content = new ChatContent("text", null, List.of(), List.of(card));
    var body    = TeamsMapper.toMessageBody(content, List.of(ref));
    var attachments = body.getJsonArray("attachments");
    assertThat(attachments).hasSize(2);
    // card comes first
    assertThat(attachments.getJsonObject(0).getString("id")).isEqualTo("card-0");
    assertThat(attachments.getJsonObject(0).getString("contentType"))
            .isEqualTo("application/vnd.microsoft.card.adaptive");
    // file ref comes second
    assertThat(attachments.getJsonObject(1).getString("id")).isEqualTo("file-0");
    assertThat(attachments.getJsonObject(1).getString("contentType")).isEqualTo("reference");
}
```

Also add `import java.util.List;` if not already present (check the existing imports).

- [ ] **Step 2: Run tests to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsMapperTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -15
```

Expected: compile error — `UploadedRef` not found and 2-arg `toMessageBody` not found.

- [ ] **Step 3: Create UploadedRef record**

Use `ide_create_file` to create `chat-teams/src/main/java/io/casehub/connectors/chat/teams/UploadedRef.java`:

```java
package io.casehub.connectors.chat.teams;

record UploadedRef(String name, String contentUrl) {}
```

- [ ] **Step 4: Add 2-arg toMessageBody and update 1-arg to delegate**

In `TeamsMapper.java`, use `ide_replace_member` to replace the existing `toMessageBody(ChatContent content)` method with:

```java
static JsonObject toMessageBody(ChatContent content) {
    return toMessageBody(content, java.util.List.of());
}

static JsonObject toMessageBody(ChatContent content, java.util.List<UploadedRef> uploadedRefs) {
    boolean hasCards = !content.cards().isEmpty();
    boolean hasFiles = !uploadedRefs.isEmpty();
    boolean useHtml  = content.markdown() != null || hasCards || hasFiles;

    String bodyText;
    String contentType;
    if (content.markdown() != null) {
        bodyText    = markdownToHtml(content.markdown());
        contentType = "html";
    } else if (useHtml) {
        bodyText    = escapeHtml(content.text());
        contentType = "html";
    } else {
        bodyText    = content.text();
        contentType = "text";
    }

    var msg = Json.createObjectBuilder();

    if (hasCards || hasFiles) {
        var attachments = Json.createArrayBuilder();
        var refs = new StringBuilder(bodyText);

        int idx = 0;
        for (var card : content.cards()) {
            String attachId = "card-" + idx;
            attachments.add(Json.createObjectBuilder()
                                .add("id", attachId)
                                .add("contentType", "application/vnd.microsoft.card.adaptive")
                                .addNull("contentUrl")
                                .add("content", toAdaptiveCardJson(card)));
            refs.append("<attachment id=\"").append(attachId).append("\"></attachment>");
            idx++;
        }

        for (int i = 0; i < uploadedRefs.size(); i++) {
            var ref = uploadedRefs.get(i);
            String attachId = "file-" + i;
            attachments.add(Json.createObjectBuilder()
                                .add("id", attachId)
                                .add("contentType", "reference")
                                .add("contentUrl", ref.contentUrl())
                                .add("name", ref.name()));
            refs.append("<attachment id=\"").append(attachId).append("\"></attachment>");
        }

        msg.add("attachments", attachments);
        bodyText = refs.toString();
    }

    msg.add("body", Json.createObjectBuilder()
                        .add("contentType", contentType)
                        .add("content", bodyText));
    return msg.build();
}
```

Also add to `TeamsMapper.java` imports (use `ide_replace_text_in_file` to add after existing imports):
```java
import java.util.List;
```

- [ ] **Step 5: Run all mapper tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsMapperTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -15
```

Expected: `Tests run: 22, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-teams/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#229): add UploadedRef + toMessageBody(content, refs) in TeamsMapper

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: TeamsClient file upload methods

After this batch: `TeamsClient` can get a channel's filesFolder and upload files (simple ≤4MB and resumable >4MB) to SharePoint. Fully tested with stubbed `HttpSender`.

### Task 2: getChannelFilesFolder + simple upload

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java`
- Create: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsClientTest.java`

**Interfaces:**
- Produces: `JsonObject TeamsClient.getChannelFilesFolder(String teamId, String channelId, String token)` — calls `GET /teams/{teamId}/channels/{channelId}/filesFolder`, returns raw Graph DriveItem JsonObject
- Produces: `JsonObject TeamsClient.uploadToChannelFolder(String driveId, String folderId, String filename, String contentType, byte[] content, String token)` — returns Graph DriveItem JsonObject containing `webUrl` and `name`

- [ ] **Step 1: Create TeamsClientTest with failing tests**

Use `ide_create_file` to create `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsClientTest.java`:

```java
package io.casehub.connectors.chat.teams;

import io.casehub.connectors.microsoft.MicrosoftGraphClient;
import org.junit.jupiter.api.Test;

import java.net.http.HttpRequest;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class TeamsClientTest {

    @Test
    void getChannelFilesFolderCallsCorrectPath() {
        var capturedUris = new ArrayList<String>();
        var graph = new MicrosoftGraphClient(request -> {
            capturedUris.add(request.uri().toString());
            return new StubHttpResponse(200, """
                    {"id":"folder-1","parentReference":{"driveId":"drive-1"}}""");
        });
        var client = new TeamsClient(graph);

        var result = client.getChannelFilesFolder("team-1", "ch-1", "token");

        assertThat(capturedUris).anyMatch(u -> u.contains("/teams/team-1/channels/ch-1/filesFolder"));
        assertThat(result.getString("id")).isEqualTo("folder-1");
        assertThat(result.getJsonObject("parentReference").getString("driveId")).isEqualTo("drive-1");
    }

    @Test
    void uploadToChannelFolderSimplePathPutsToCorrectUrl() {
        var capturedRequests = new ArrayList<HttpRequest>();
        var graph = new MicrosoftGraphClient(request -> {
            capturedRequests.add(request);
            return new StubHttpResponse(201, """
                    {"id":"item-1","name":"0-report.pdf",\
                    "webUrl":"https://sharepoint.com/files/0-report.pdf"}""");
        });
        var client = new TeamsClient(graph);
        byte[] content = "PDF content".getBytes(StandardCharsets.UTF_8);

        var result = client.uploadToChannelFolder(
                "drive-1", "folder-1", "0-report.pdf", "application/pdf", content, "token");

        assertThat(capturedRequests).hasSize(1);
        String uri = capturedRequests.get(0).uri().toString();
        assertThat(uri).contains("/drives/drive-1/items/folder-1:/0-report.pdf:/content");
        assertThat(capturedRequests.get(0).method()).isEqualTo("PUT");
        assertThat(result.getString("webUrl")).isEqualTo("https://sharepoint.com/files/0-report.pdf");
        assertThat(result.getString("name")).isEqualTo("0-report.pdf");
    }

    @Test
    void uploadToChannelFolderUrlEncodesFilename() {
        var capturedUris = new ArrayList<String>();
        var graph = new MicrosoftGraphClient(request -> {
            capturedUris.add(request.uri().toString());
            return new StubHttpResponse(201, """
                    {"id":"item-1","name":"0-my file.pdf",\
                    "webUrl":"https://sharepoint.com/files/0-my+file.pdf"}""");
        });
        var client = new TeamsClient(graph);

        client.uploadToChannelFolder("drive-1", "folder-1", "0-my file.pdf",
                "application/pdf", new byte[1], "token");

        assertThat(capturedUris.get(0)).contains("0-my%20file.pdf");
        assertThat(capturedUris.get(0)).doesNotContain("0-my file.pdf");
    }
}
```

- [ ] **Step 2: Run to confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsClientTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -10
```

Expected: compile error — `getChannelFilesFolder` and `uploadToChannelFolder` not found.

- [ ] **Step 3: Add getChannelFilesFolder and simple upload to TeamsClient**

Use `ide_insert_member` to add after the `removeMember` method in `TeamsClient.java`:

```java
JsonObject getChannelFilesFolder(String teamId, String channelId, String token) {
    return graph.get("/teams/" + teamId + "/channels/" + channelId + "/filesFolder", token);
}

JsonObject uploadToChannelFolder(String driveId, String folderId, String filename,
        String contentType, byte[] content, String token) {
    String encodedName = java.net.URLEncoder.encode(filename, java.nio.charset.StandardCharsets.UTF_8)
            .replace("+", "%20");
    if (content.length <= SIMPLE_UPLOAD_LIMIT) {
        return uploadSimple(driveId, folderId, encodedName, contentType, content, token);
    }
    return uploadResumable(driveId, folderId, encodedName, contentType, content, token);
}

private static final int SIMPLE_UPLOAD_LIMIT = 4 * 1024 * 1024;
private static final int CHUNK_SIZE = 16 * 320 * 1024;

private JsonObject uploadSimple(String driveId, String folderId, String encodedName,
        String contentType, byte[] content, String token) {
    var request = java.net.http.HttpRequest.newBuilder(
                    java.net.URI.create(io.casehub.connectors.microsoft.MicrosoftGraphClient.BASE_URL
                            + "/drives/" + driveId + "/items/" + folderId
                            + ":/" + encodedName + ":/content"))
            .header("Authorization", "Bearer " + token)
            .header("Content-Type", contentType)
            .PUT(java.net.http.HttpRequest.BodyPublishers.ofByteArray(content))
            .build();
    var response = graph.send(request);
    io.casehub.connectors.microsoft.GraphErrorHandler.handle(response, "microsoft");
    return parseJsonObject(response.body());
}

private static JsonObject parseJsonObject(String body) {
    try (var reader = jakarta.json.Json.createReader(new java.io.StringReader(body))) {
        return reader.readObject();
    }
}
```

Add these imports at the top of `TeamsClient.java` using `ide_replace_text_in_file` to add after the existing imports:
```java
import io.casehub.connectors.microsoft.GraphErrorHandler;
import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpRequest;
import java.nio.charset.StandardCharsets;
import java.io.StringReader;
```

- [ ] **Step 4: Run Task 2 tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsClientTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -10
```

Expected: `Tests run: 3, Failures: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-teams/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#229): add getChannelFilesFolder + simple upload in TeamsClient

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Resumable upload path

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java`
- Modify: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsClientTest.java`

**Interfaces:**
- Extends `uploadToChannelFolder` — the >4MB branch calls `uploadResumable` which uses pre-auth uploadUrl chunks WITHOUT Authorization header

- [ ] **Step 1: Add failing resumable tests to TeamsClientTest**

Use `ide_insert_member` to add to `TeamsClientTest`:

```java
@Test
void uploadToChannelFolderResumablePathUsesUploadSession() {
    int bigSize = 5 * 1024 * 1024; // 5MB > 4MB threshold
    byte[] bigContent = new byte[bigSize];

    var capturedRequests = new ArrayList<HttpRequest>();
    var graph = new MicrosoftGraphClient(request -> {
        capturedRequests.add(request);
        String uri = request.uri().toString();
        // Upload session creation
        if (uri.contains("createUploadSession")) {
            return new StubHttpResponse(200,
                    "{\"uploadUrl\":\"https://upload.example.com/session123\"}");
        }
        // Chunk PUT to pre-auth uploadUrl
        if (uri.contains("upload.example.com")) {
            if (capturedRequests.stream()
                    .filter(r -> r.uri().toString().contains("upload.example.com"))
                    .count() < 2) {
                // intermediate chunk
                return new StubHttpResponse(202, "");
            }
            // final chunk
            return new StubHttpResponse(201, """
                    {"id":"item-1","name":"0-big.pdf",\
                    "webUrl":"https://sharepoint.com/files/0-big.pdf"}""");
        }
        return new StubHttpResponse(200, "{}");
    });
    var client = new TeamsClient(graph);

    var result = client.uploadToChannelFolder(
            "drive-1", "folder-1", "0-big.pdf", "application/pdf", bigContent, "token");

    assertThat(result.getString("webUrl")).isEqualTo("https://sharepoint.com/files/0-big.pdf");
    // Session creation request
    assertThat(capturedRequests.get(0).uri().toString()).contains("createUploadSession");
    // Chunk PUTs to uploadUrl — MUST NOT carry Authorization header
    var chunkRequests = capturedRequests.stream()
            .filter(r -> r.uri().toString().contains("upload.example.com"))
            .toList();
    assertThat(chunkRequests).isNotEmpty();
    for (var chunk : chunkRequests) {
        assertThat(chunk.headers().firstValue("Authorization")).isEmpty();
        assertThat(chunk.headers().firstValue("Content-Range")).isPresent();
    }
}
```

- [ ] **Step 2: Run to confirm new test fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsClientTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 1` (the resumable test fails — 5MB currently takes the simple path and 404s or returns wrong result).

- [ ] **Step 3: Add uploadResumable to TeamsClient**

Use `ide_insert_member` to add `uploadResumable` after `uploadSimple` in `TeamsClient.java`:

```java
private JsonObject uploadResumable(String driveId, String folderId, String encodedName,
        String contentType, byte[] content, String token) {
    var sessionBody = jakarta.json.Json.createObjectBuilder()
            .add("item", jakarta.json.Json.createObjectBuilder()
                    .add("@microsoft.graph.conflictBehavior", "replace")
                    .add("name", encodedName))
            .build();
    var session = graph.post("/drives/" + driveId + "/items/" + folderId
            + ":/" + encodedName + ":/createUploadSession", sessionBody, token);
    String uploadUrl = session.getString("uploadUrl");

    int totalSize = content.length;
    int offset = 0;
    JsonObject result = null;
    while (offset < totalSize) {
        int end = Math.min(offset + CHUNK_SIZE, totalSize);
        byte[] chunk = java.util.Arrays.copyOfRange(content, offset, end);
        var request = java.net.http.HttpRequest.newBuilder(java.net.URI.create(uploadUrl))
                .header("Content-Type", contentType)
                .header("Content-Range",
                        "bytes " + offset + "-" + (end - 1) + "/" + totalSize)
                .PUT(java.net.http.HttpRequest.BodyPublishers.ofByteArray(chunk))
                .build();
        var response = graph.send(request);
        if (response.statusCode() >= 300) {
            io.casehub.connectors.microsoft.GraphErrorHandler.handle(response, "microsoft");
        }
        if (end >= totalSize) {
            result = parseJsonObject(response.body());
        }
        offset = end;
    }
    return result;
}
```

Add `import java.util.Arrays;` using `ide_replace_text_in_file`.

- [ ] **Step 4: Run all TeamsClient tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsClientTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0`

- [ ] **Step 5: Run full module test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test 2>&1 | tail -10
```

Expected: `Tests run: N, Failures: 0`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-teams/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#229): add resumable upload path in TeamsClient

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Orchestration + documentation

After this batch: `TeamsChatPlatform.sendMessage` and `replyToMessage` upload attachments before sending; consumer guide documents the required permission.

### Task 4: Pre-upload step in TeamsChatPlatform

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java`
- Modify: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java`

**Interfaces:**
- Consumes: `TeamsClient.getChannelFilesFolder(teamId, channelId, token)` → JsonObject with `id` and `parentReference.driveId`
- Consumes: `TeamsClient.uploadToChannelFolder(driveId, folderId, filename, contentType, bytes, token)` → JsonObject with `webUrl`, `name`
- Consumes: `TeamsMapper.toMessageBody(content, uploadedRefs)` — 2-arg form from Task 1
- Consumes: `UploadedRef(name, contentUrl)` from Task 1

- [ ] **Step 1: Add failing attachment test to TeamsChatPlatformTest**

Add a new nested class `AttachmentTest` and a helper method. Use `ide_insert_member` to add before the closing `}` of `TeamsChatPlatformTest`:

```java
@Nested
class AttachmentTest {

    @Test
    void sendMessageWithAttachmentUploadsAndReferences() {
        var requestedUris = new java.util.ArrayList<String>();
        var graph = new MicrosoftGraphClient(request -> {
            String uri = request.uri().toString();
            String method = request.method();
            requestedUris.add(method + " " + uri);

            if (uri.contains("/filesFolder")) {
                return new StubHttpResponse(200, """
                        {"id":"folder-1","parentReference":{"driveId":"drive-1"}}""");
            }
            if (uri.contains("/items/folder-1:/")) {
                return new StubHttpResponse(201, """
                        {"id":"item-1","name":"0-test.pdf",\
                        "webUrl":"https://sharepoint.com/files/0-test.pdf"}""");
            }
            if (uri.contains("/messages") && method.equals("POST")) {
                return new StubHttpResponse(201, messageResponseJson("msg-with-att"));
            }
            return new StubHttpResponse(200, "{\"value\":[]}");
        });
        var platformWithAtt = new TeamsChatPlatform(new TeamsClient(graph)).build();

        var att = new io.casehub.connectors.Attachment(
                "test.pdf", "application/pdf", new byte[]{1, 2, 3});
        var content = new ChatContent("See attachment", null,
                java.util.List.of(att), java.util.List.of());
        var ref = new ChatChannelRef("team-1:ch-1");
        var result = platformWithAtt.messaging().send(ref, content);

        assertThat(result.ok()).isTrue();
        assertThat(requestedUris).anyMatch(u -> u.contains("/filesFolder"));
        assertThat(requestedUris).anyMatch(u -> u.contains("/items/folder-1:/"));
        assertThat(requestedUris).anyMatch(u -> u.contains("/messages") && u.startsWith("POST"));
    }

    @Test
    void sendMessageUploadFailurePropagates() {
        var graph = new MicrosoftGraphClient(request -> {
            if (request.uri().toString().contains("/filesFolder")) {
                return new StubHttpResponse(200, """
                        {"id":"folder-1","parentReference":{"driveId":"drive-1"}}""");
            }
            if (request.uri().toString().contains("/items/")) {
                return new StubHttpResponse(403, "{\"error\":{\"code\":\"Forbidden\"}}");
            }
            return new StubHttpResponse(200, "{\"value\":[]}");
        });
        var platformWithAtt = new TeamsChatPlatform(new TeamsClient(graph)).build();

        var att = new io.casehub.connectors.Attachment(
                "secret.pdf", "application/pdf", new byte[]{1});
        var content = new ChatContent("text", null,
                java.util.List.of(att), java.util.List.of());
        var ref = new ChatChannelRef("team-1:ch-1");

        org.assertj.core.api.Assertions.assertThatThrownBy(
                () -> platformWithAtt.messaging().send(ref, content))
                .isInstanceOf(io.casehub.connectors.ProviderApiException.class);
    }
}
```

Add these imports to `TeamsChatPlatformTest.java` using `ide_replace_text_in_file` to add after existing imports:
```java
import io.casehub.connectors.Attachment;
import io.casehub.connectors.ProviderApiException;
import io.casehub.connectors.chat.model.ChatContent;
```

- [ ] **Step 2: Run to confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test -Dtest=TeamsChatPlatformTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -10
```

Expected: failures — attachments not yet uploaded, `toMessageBody` still ignores them.

- [ ] **Step 3: Add safeAttachmentName helper to TeamsChatPlatform**

Use `ide_insert_member` to add before the existing `resolveDelegatedToken` method in `TeamsChatPlatform.java`:

```java
private static String safeAttachmentName(int index, io.casehub.connectors.Attachment attachment) {
    String name = attachment.filename();
    if (name == null || name.isBlank()) {
        name = "attachment-" + index + extensionFromMimeType(attachment.contentType());
    }
    return index + "-" + name;
}

private static String extensionFromMimeType(String contentType) {
    if (contentType == null) return ".bin";
    return switch (contentType) {
        case "application/pdf" -> ".pdf";
        case "image/png"       -> ".png";
        case "image/jpeg"      -> ".jpg";
        case "image/gif"       -> ".gif";
        case "text/plain"      -> ".txt";
        case "application/msword" -> ".doc";
        case "application/vnd.openxmlformats-officedocument.wordprocessingml.document" -> ".docx";
        default -> ".bin";
    };
}

private java.util.List<UploadedRef> uploadAttachments(String teamId, String channelId,
        java.util.List<io.casehub.connectors.Attachment> attachments, String token) {
    if (attachments.isEmpty()) return java.util.List.of();
    var folder = client.getChannelFilesFolder(teamId, channelId, token);
    String folderId = folder.getString("id");
    String driveId  = folder.getJsonObject("parentReference").getString("driveId");
    var refs = new java.util.ArrayList<UploadedRef>();
    for (int i = 0; i < attachments.size(); i++) {
        var att = attachments.get(i);
        String name = safeAttachmentName(i, att);
        var item = client.uploadToChannelFolder(
                driveId, folderId, name, att.contentType(), att.content(), token);
        refs.add(new UploadedRef(item.getString("name"), item.getString("webUrl")));
    }
    return java.util.List.copyOf(refs);
}
```

- [ ] **Step 4: Update sendMessage to use 2-arg toMessageBody with pre-upload**

Use `ide_replace_member` to replace the existing `sendMessage` method in `TeamsChatPlatform.java`:

```java
private SendResult sendMessage(ChatChannelRef channel, ChatContent content) {
    var tc = TeamsMapper.parseTeamAndChannel(channel.id());
    return TokenRetry.withTokenRetry(this::resolveApplicationToken,
            token -> {
                var uploadedRefs = uploadAttachments(tc[0], tc[1],
                        content.attachments(), token);
                var result = client.sendMessage(tc[0], tc[1],
                        TeamsMapper.toMessageBody(content, uploadedRefs), token);
                return TeamsMapper.toSendResult(tc[0], tc[1], result);
            });
}
```

- [ ] **Step 5: Update replyToMessage to use 2-arg toMessageBody with pre-upload**

Use `ide_replace_member` to replace the existing `replyToMessage` method in `TeamsChatPlatform.java`:

```java
private SendResult replyToMessage(ChatMessageRef parent, ChatContent content) {
    var tc = TeamsMapper.parseTeamAndChannel(parent.channel().id());
    return TokenRetry.withTokenRetry(this::resolveApplicationToken,
            token -> {
                var uploadedRefs = uploadAttachments(tc[0], tc[1],
                        content.attachments(), token);
                var result = client.replyToMessage(tc[0], tc[1],
                        parent.messageId(), TeamsMapper.toMessageBody(content, uploadedRefs), token);
                return TeamsMapper.toSendResult(tc[0], tc[1], result);
            });
}
```

- [ ] **Step 6: Run all chat-teams tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test 2>&1 | tail -10
```

Expected: `Tests run: N, Failures: 0`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-teams/src/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#229): pre-upload attachments in TeamsChatPlatform.sendMessage/replyToMessage

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: Consumer guide — document Files.ReadWrite.All requirement

**Files:**
- Modify: `docs/guides/consumer-guide.md`

**Interfaces:** None — documentation only.

- [ ] **Step 1: Find the Teams section in consumer-guide.md**

```bash
grep -n "chat-teams\|microsoft-teams\|Teams" /Users/mdproctor/claude/casehub/connectors/docs/guides/consumer-guide.md | head -20
```

- [ ] **Step 2: Add permission note to the Teams section**

Use `ide_replace_text_in_file` or Edit to add after the Teams capabilities description:

```markdown
> **Required Graph permissions for file attachments:** The Teams app registration must have
> the `Files.ReadWrite.All` **application** permission (admin-granted). Without it, `sendMessage`
> calls that include `ChatContent.attachments()` will throw `ProviderAuthException` (403).
> Existing messaging without attachments is unaffected.
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add docs/guides/consumer-guide.md
git -C /Users/mdproctor/claude/casehub/connectors commit -m "docs(#229): document Files.ReadWrite.All requirement for Teams file attachments

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Verification

After all tasks complete:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/connectors/chat-teams/pom.xml test 2>&1 | tail -5
```

Expected: all tests green. Run `git log --oneline -6` to confirm all 5 task commits are present.

---

## References

- `specs/issue-229-teams-file-attachments/2026-11-16-teams-file-attachments-design.md` — design spec
- `specs/issue-229-teams-file-attachments/decisions.md` — D1, D2, D3
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsMapper.java` — mapper being extended
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java` — client being extended
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java` — orchestration
- `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsChatPlatformTest.java` — existing test patterns
- `chat-teams/src/test/java/io/casehub/connectors/chat/teams/StubHttpResponse.java` — test stub reused
- `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/MicrosoftDriveClient.java` — resumable upload pattern source
- `microsoft-graph-client/src/main/java/io/casehub/connectors/microsoft/MicrosoftGraphClient.java` — `send()` passthrough + `HttpSender`
- casehubio/connectors#229 — issue
