# Reference InvokeHandler Implementations Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #258 — chat-teams: Reference InvokeHandler implementations for messaging extensions
**Issue group:** #236, #257, #259, #258

**Goal:** Add 4 reference InvokeHandler implementations in chat-teams that demonstrate the messaging extension handler contract with canned data.

**Architecture:** Four public classes in `chat-teams` under a `ref` sub-package. Each implements InvokeHandler with a fixed invoke name and returns hard-coded canned responses using ComposeAttachment (from #259). Plain classes, no CDI annotations — instantiated by consumers.

**Tech Stack:** Java 21, Jakarta JSON-P, JUnit 5, AssertJ

## Global Constraints

- Java 21 source, Java 26 JVM
- No new dependencies — `jakarta.json` already in `chat-teams`
- No new modules — all handlers in `chat-teams`
- Plain classes, no CDI annotations (matches ref implementation pattern)
- Hard-coded canned responses regardless of input

---

## Batch 1: Search and unfurl handlers

### Task 1: RefSearchHandler

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefSearchHandler.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefSearchHandlerTest.java`

**Interfaces:**
- Consumes: `InvokeHandler` (SPI), `InvokeRequest`, `InvokeResponse.ComposeResult`, `ComposeAttachment.adaptive(String).preview(String, String).build()` → `String`
- Produces: `RefSearchHandler implements InvokeHandler` — `name()` returns `"composeExtension/query"`, `handle(InvokeRequest)` returns `ComposeResult("list", List<String>)` with 3 canned results

- [ ] **Step 1: Write failing test**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;
import io.casehub.connectors.chat.teams.InvokeResponseSerializer;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.io.StringReader;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RefSearchHandlerTest {

    private final RefSearchHandler handler = new RefSearchHandler();

    @Test
    void name_returnsComposeExtensionQuery() {
        assertThat(handler.name()).isEqualTo("composeExtension/query");
    }

    @Test
    void handle_returnsComposeResultWithThreeResults() {
        var request = new InvokeRequest("composeExtension/query",
                Json.createObjectBuilder().add("queryText", "test").build(),
                "user-1", "channel-1", Map.of());

        InvokeResponse response = handler.handle(request);

        assertThat(response).isInstanceOf(InvokeResponse.ComposeResult.class);
        var result = (InvokeResponse.ComposeResult) response;
        assertThat(result.attachmentLayout()).isEqualTo("list");
        assertThat(result.attachmentsJson()).hasSize(3);
    }

    @Test
    void handle_attachmentsSerializeCorrectly() {
        var request = new InvokeRequest("composeExtension/query",
                Json.createObjectBuilder().add("queryText", "test").build(),
                "user-1", "channel-1", Map.of());

        var result = (InvokeResponse.ComposeResult) handler.handle(request);
        String serialized = InvokeResponseSerializer.toJson(result);
        JsonObject obj = parse(serialized);
        JsonObject ext = obj.getJsonObject("composeExtension");

        assertThat(ext.getString("type")).isEqualTo("result");
        assertThat(ext.getJsonArray("attachments")).hasSize(3);
        JsonObject first = ext.getJsonArray("attachments").getJsonObject(0);
        assertThat(first.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
        assertThat(first.getJsonObject("preview")).isNotNull();
        assertThat(first.getJsonObject("preview").getJsonObject("content")
                .getString("title")).isNotEmpty();
    }

    private static JsonObject parse(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefSearchHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `RefSearchHandler` class does not exist

- [ ] **Step 3: Implement RefSearchHandler**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.ComposeAttachment;
import io.casehub.connectors.chat.teams.InvokeHandler;
import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;

import java.util.List;

public class RefSearchHandler implements InvokeHandler {

    private static final String CARD_1 = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Contoso Widget\",\"weight\":\"bolder\"},{\"type\":\"TextBlock\",\"text\":\"A premium widget for all your needs.\"}]}";
    private static final String CARD_2 = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Northwind Gadget\",\"weight\":\"bolder\"},{\"type\":\"TextBlock\",\"text\":\"The essential gadget for modern teams.\"}]}";
    private static final String CARD_3 = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Adventure Gizmo\",\"weight\":\"bolder\"},{\"type\":\"TextBlock\",\"text\":\"Take your projects to the next level.\"}]}";

    @Override
    public String name() {
        return "composeExtension/query";
    }

    @Override
    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeResult("list", List.of(
                ComposeAttachment.adaptive(CARD_1).preview("Contoso Widget", "£29.99").build(),
                ComposeAttachment.adaptive(CARD_2).preview("Northwind Gadget", "£49.99").build(),
                ComposeAttachment.adaptive(CARD_3).preview("Adventure Gizmo", "£19.99").build()
        ));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefSearchHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefSearchHandler.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefSearchHandlerTest.java
git commit -m "feat(#258): add RefSearchHandler for composeExtension/query"
```

### Task 2: RefLinkUnfurlHandler

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefLinkUnfurlHandler.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefLinkUnfurlHandlerTest.java`

**Interfaces:**
- Consumes: `InvokeHandler`, `InvokeRequest`, `InvokeResponse.ComposeResult`, `ComposeAttachment.adaptive(String).build()` → `String`
- Produces: `RefLinkUnfurlHandler implements InvokeHandler` — `name()` returns `"composeExtension/queryLink"`, `handle(InvokeRequest)` returns `ComposeResult("list", List<String>)` with 1 unfurl card (no preview)

- [ ] **Step 1: Write failing test**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;
import io.casehub.connectors.chat.teams.InvokeResponseSerializer;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.io.StringReader;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RefLinkUnfurlHandlerTest {

    private final RefLinkUnfurlHandler handler = new RefLinkUnfurlHandler();

    @Test
    void name_returnsComposeExtensionQueryLink() {
        assertThat(handler.name()).isEqualTo("composeExtension/queryLink");
    }

    @Test
    void handle_returnsComposeResultWithOneCard() {
        var request = new InvokeRequest("composeExtension/queryLink",
                Json.createObjectBuilder().add("url", "https://example.com/item/42").build(),
                "user-1", "channel-1", Map.of());

        InvokeResponse response = handler.handle(request);

        assertThat(response).isInstanceOf(InvokeResponse.ComposeResult.class);
        var result = (InvokeResponse.ComposeResult) response;
        assertThat(result.attachmentLayout()).isEqualTo("list");
        assertThat(result.attachmentsJson()).hasSize(1);
    }

    @Test
    void handle_unfurlCardHasNoPreview() {
        var request = new InvokeRequest("composeExtension/queryLink",
                Json.createObjectBuilder().add("url", "https://example.com/item/42").build(),
                "user-1", "channel-1", Map.of());

        var result = (InvokeResponse.ComposeResult) handler.handle(request);
        String serialized = InvokeResponseSerializer.toJson(result);
        JsonObject obj = parse(serialized);
        JsonObject attachment = obj.getJsonObject("composeExtension")
                .getJsonArray("attachments").getJsonObject(0);

        assertThat(attachment.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
        assertThat(attachment.containsKey("preview")).isFalse();
    }

    private static JsonObject parse(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefLinkUnfurlHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `RefLinkUnfurlHandler` class does not exist

- [ ] **Step 3: Implement RefLinkUnfurlHandler**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.ComposeAttachment;
import io.casehub.connectors.chat.teams.InvokeHandler;
import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;

import java.util.List;

public class RefLinkUnfurlHandler implements InvokeHandler {

    private static final String UNFURL_CARD = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Link Preview\",\"weight\":\"bolder\"},{\"type\":\"TextBlock\",\"text\":\"This is a preview of the linked content.\"}],\"actions\":[{\"type\":\"Action.OpenUrl\",\"title\":\"View\",\"url\":\"https://example.com\"}]}";

    @Override
    public String name() {
        return "composeExtension/queryLink";
    }

    @Override
    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeResult("list", List.of(
                ComposeAttachment.adaptive(UNFURL_CARD).build()
        ));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefLinkUnfurlHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefLinkUnfurlHandler.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefLinkUnfurlHandlerTest.java
git commit -m "feat(#258): add RefLinkUnfurlHandler for composeExtension/queryLink"
```

## Batch 2: Action handlers

### Task 3: RefFetchTaskHandler

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefFetchTaskHandler.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefFetchTaskHandlerTest.java`

**Interfaces:**
- Consumes: `InvokeHandler`, `InvokeRequest`, `InvokeResponse.ComposeContinue`
- Produces: `RefFetchTaskHandler implements InvokeHandler` — `name()` returns `"composeExtension/fetchTask"`, `handle(InvokeRequest)` returns `ComposeContinue("Create Item", cardJson)`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;
import io.casehub.connectors.chat.teams.InvokeResponseSerializer;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.io.StringReader;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RefFetchTaskHandlerTest {

    private final RefFetchTaskHandler handler = new RefFetchTaskHandler();

    @Test
    void name_returnsComposeExtensionFetchTask() {
        assertThat(handler.name()).isEqualTo("composeExtension/fetchTask");
    }

    @Test
    void handle_returnsComposeContinueWithTitle() {
        var request = new InvokeRequest("composeExtension/fetchTask",
                Json.createObjectBuilder().add("commandId", "createItem").build(),
                "user-1", "channel-1", Map.of());

        InvokeResponse response = handler.handle(request);

        assertThat(response).isInstanceOf(InvokeResponse.ComposeContinue.class);
        var result = (InvokeResponse.ComposeContinue) response;
        assertThat(result.title()).isEqualTo("Create Item");
    }

    @Test
    void handle_serializedResponseHasTaskModuleCard() {
        var request = new InvokeRequest("composeExtension/fetchTask",
                Json.createObjectBuilder().add("commandId", "createItem").build(),
                "user-1", "channel-1", Map.of());

        String serialized = InvokeResponseSerializer.toJson(handler.handle(request));
        JsonObject obj = parse(serialized);
        JsonObject ext = obj.getJsonObject("composeExtension");

        assertThat(ext.getString("type")).isEqualTo("continue");
        assertThat(ext.getJsonObject("value").getString("title")).isEqualTo("Create Item");
        assertThat(ext.getJsonObject("value").getJsonObject("card")
                .getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
    }

    private static JsonObject parse(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefFetchTaskHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `RefFetchTaskHandler` class does not exist

- [ ] **Step 3: Implement RefFetchTaskHandler**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.InvokeHandler;
import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;

public class RefFetchTaskHandler implements InvokeHandler {

    private static final String FORM_CARD = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Create a new item\",\"weight\":\"bolder\"},{\"type\":\"Input.Text\",\"id\":\"title\",\"label\":\"Title\",\"placeholder\":\"Enter a title\"}],\"actions\":[{\"type\":\"Action.Submit\",\"title\":\"Create\"}]}";

    @Override
    public String name() {
        return "composeExtension/fetchTask";
    }

    @Override
    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeContinue("Create Item", FORM_CARD);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefFetchTaskHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefFetchTaskHandler.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefFetchTaskHandlerTest.java
git commit -m "feat(#258): add RefFetchTaskHandler for composeExtension/fetchTask"
```

### Task 4: RefSubmitActionHandler

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefSubmitActionHandler.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefSubmitActionHandlerTest.java`

**Interfaces:**
- Consumes: `InvokeHandler`, `InvokeRequest`, `InvokeResponse.ComposeResult`, `ComposeAttachment.adaptive(String).preview(String, String).build()` → `String`
- Produces: `RefSubmitActionHandler implements InvokeHandler` — `name()` returns `"composeExtension/submitAction"`, `handle(InvokeRequest)` returns `ComposeResult("list", List<String>)` with 1 confirmation card

- [ ] **Step 1: Write failing test**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;
import io.casehub.connectors.chat.teams.InvokeResponseSerializer;
import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.io.StringReader;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RefSubmitActionHandlerTest {

    private final RefSubmitActionHandler handler = new RefSubmitActionHandler();

    @Test
    void name_returnsComposeExtensionSubmitAction() {
        assertThat(handler.name()).isEqualTo("composeExtension/submitAction");
    }

    @Test
    void handle_returnsComposeResultWithConfirmation() {
        var request = new InvokeRequest("composeExtension/submitAction",
                Json.createObjectBuilder().add("title", "Bug report").build(),
                "user-1", "channel-1", Map.of());

        InvokeResponse response = handler.handle(request);

        assertThat(response).isInstanceOf(InvokeResponse.ComposeResult.class);
        var result = (InvokeResponse.ComposeResult) response;
        assertThat(result.attachmentLayout()).isEqualTo("list");
        assertThat(result.attachmentsJson()).hasSize(1);
    }

    @Test
    void handle_confirmationHasPreview() {
        var request = new InvokeRequest("composeExtension/submitAction",
                Json.createObjectBuilder().add("title", "Bug report").build(),
                "user-1", "channel-1", Map.of());

        var result = (InvokeResponse.ComposeResult) handler.handle(request);
        String serialized = InvokeResponseSerializer.toJson(result);
        JsonObject obj = parse(serialized);
        JsonObject attachment = obj.getJsonObject("composeExtension")
                .getJsonArray("attachments").getJsonObject(0);

        assertThat(attachment.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
        assertThat(attachment.getJsonObject("preview")).isNotNull();
        assertThat(attachment.getJsonObject("preview").getJsonObject("content")
                .getString("title")).isEqualTo("Item Created");
    }

    private static JsonObject parse(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefSubmitActionHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `RefSubmitActionHandler` class does not exist

- [ ] **Step 3: Implement RefSubmitActionHandler**

```java
package io.casehub.connectors.chat.teams.ref;

import io.casehub.connectors.chat.teams.ComposeAttachment;
import io.casehub.connectors.chat.teams.InvokeHandler;
import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;

import java.util.List;

public class RefSubmitActionHandler implements InvokeHandler {

    private static final String CONFIRMATION_CARD = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Item Created Successfully\",\"weight\":\"bolder\"},{\"type\":\"TextBlock\",\"text\":\"Your item has been created and is ready to use.\"}]}";

    @Override
    public String name() {
        return "composeExtension/submitAction";
    }

    @Override
    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeResult("list", List.of(
                ComposeAttachment.adaptive(CONFIRMATION_CARD)
                        .preview("Item Created", "Action completed")
                        .build()
        ));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=io.casehub.connectors.chat.teams.ref.RefSubmitActionHandlerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (3 tests)

- [ ] **Step 5: Run full chat-teams test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams`
Expected: All tests PASS (no regressions)

- [ ] **Step 6: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefSubmitActionHandler.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/RefSubmitActionHandlerTest.java
git commit -m "feat(#258): add RefSubmitActionHandler for composeExtension/submitAction"
```

## References

- [2026-11-17-reference-invoke-handlers-design.md] — design spec this plan implements
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java] — SPI interface
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeService.java] — dispatch by name
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeRequest.java] — request record
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java] — response variants
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java] — builder from #259
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java] — serialization
- [chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java] — ref implementation pattern
- [GitHub #258] — issue
- [GitHub #259] — ComposeAttachment builder dependency
