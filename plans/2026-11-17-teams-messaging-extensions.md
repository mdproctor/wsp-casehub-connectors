# Teams Messaging Extensions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #236 — chat-teams: Messaging extensions support
**Issue group:** #236

**Goal:** Add 3 new InvokeResponse variants (ComposeResult, ComposeAuth, ComposeContinue) with serialization and round-trip tests to support all 7 `composeExtension/*` Bot Framework invoke activities.

**Architecture:** Extend the existing InvokeHandler → InvokeService → InvokeResponse → InvokeResponseSerializer pipeline from #235. No new modules, SPIs, or webhook handlers. All changes are within `chat-teams`. The existing `handleInvoke()` in `TeamsInboundConnector` already dispatches all invoke activities through `InvokeService` and returns `WebhookResult.Responded` — the new response variants flow through unchanged.

**Tech Stack:** Java 21, Jakarta JSON-P, JUnit 5, AssertJ

## Global Constraints

- Java 21 source (Java 26 JVM)
- Use `jakarta.json` for JSON construction (matches existing code)
- Raw JSON strings for card/attachment content — no typed Adaptive Card model
- `HttpHelper.CLIENT` for any outbound HTTP (protocol: shared-http-client)
- Connector IDs via `InboundConnectorIds` constants (protocol: inbound-connector-id-constants)

---

## Batch 1: Response Model, Serialization, and Integration Tests

### Task 1: InvokeResponse variants and serialization

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java`
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java`
- Modify: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeResponseSerializerTest.java`

**Interfaces:**
- Consumes: Existing `InvokeResponse` sealed interface, `InvokeResponseSerializer.toJson()`, `InvokeResponseSerializer.parseJson()`
- Produces: `InvokeResponse.ComposeResult(String attachmentLayout, List<String> attachmentsJson)`, `InvokeResponse.ComposeAuth(String suggestedActionsJson)`, `InvokeResponse.ComposeContinue(String title, String cardJson)` — used by Task 2's round-trip tests and by any future `InvokeHandler` implementations

- [ ] **Step 1: Write failing serializer tests for ComposeResult**

Add to `InvokeResponseSerializerTest.java`:

```java
@Test
void composeResult_serializesWithAttachments() {
    String att1 = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\"},\"preview\":{\"contentType\":\"application/vnd.microsoft.card.thumbnail\",\"content\":{\"title\":\"Result 1\"}}}";
    String att2 = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\"},\"preview\":{\"contentType\":\"application/vnd.microsoft.card.thumbnail\",\"content\":{\"title\":\"Result 2\"}}}";
    String json = InvokeResponseSerializer.toJson(
            new InvokeResponse.ComposeResult("list", List.of(att1, att2)));
    JsonObject obj = parse(json);
    JsonObject ext = obj.getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
    assertThat(ext.getString("attachmentLayout")).isEqualTo("list");
    assertThat(ext.getJsonArray("attachments")).hasSize(2);
    assertThat(ext.getJsonArray("attachments").getJsonObject(0)
            .getJsonObject("preview").getJsonObject("content").getString("title"))
            .isEqualTo("Result 1");
}

@Test
void composeResult_emptyAttachments() {
    String json = InvokeResponseSerializer.toJson(
            new InvokeResponse.ComposeResult("list", List.of()));
    JsonObject obj = parse(json);
    JsonObject ext = obj.getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
    assertThat(ext.getJsonArray("attachments")).isEmpty();
}

@Test
void composeResult_gridLayout() {
    String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\"}}";
    String json = InvokeResponseSerializer.toJson(
            new InvokeResponse.ComposeResult("grid", List.of(att)));
    JsonObject obj = parse(json);
    assertThat(obj.getJsonObject("composeExtension").getString("attachmentLayout"))
            .isEqualTo("grid");
}
```

Add `import java.util.List;` to the test file imports.

- [ ] **Step 2: Write failing serializer tests for ComposeAuth**

Add to `InvokeResponseSerializerTest.java`:

```java
@Test
void composeAuth_serializesWithSuggestedActions() {
    String actions = "{\"actions\":[{\"type\":\"openUrl\",\"value\":\"https://auth.example.com/oauth\",\"title\":\"Sign in\"}]}";
    String json = InvokeResponseSerializer.toJson(
            new InvokeResponse.ComposeAuth(actions));
    JsonObject obj = parse(json);
    JsonObject ext = obj.getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("auth");
    assertThat(ext.getJsonObject("suggestedActions")
            .getJsonArray("actions").getJsonObject(0)
            .getString("type")).isEqualTo("openUrl");
}
```

- [ ] **Step 3: Write failing serializer test for ComposeContinue**

Add to `InvokeResponseSerializerTest.java`:

```java
@Test
void composeContinue_serializesWithTitleAndCard() {
    String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Enter details\"}]}";
    String json = InvokeResponseSerializer.toJson(
            new InvokeResponse.ComposeContinue("Action Form", cardJson));
    JsonObject obj = parse(json);
    JsonObject ext = obj.getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("continue");
    JsonObject value = ext.getJsonObject("value");
    assertThat(value.getString("title")).isEqualTo("Action Form");
    JsonObject card = value.getJsonObject("card");
    assertThat(card.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
    assertThat(card.getJsonObject("content").getString("type")).isEqualTo("AdaptiveCard");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InvokeResponseSerializerTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: compilation failure — `ComposeResult`, `ComposeAuth`, `ComposeContinue` do not exist yet.

- [ ] **Step 5: Add 3 new variants to InvokeResponse**

Add to `InvokeResponse.java` after the `Error` record (before the closing `}`):

```java
record ComposeResult(String attachmentLayout, java.util.List<String> attachmentsJson)
    implements InvokeResponse {}

record ComposeAuth(String suggestedActionsJson) implements InvokeResponse {}

record ComposeContinue(String title, String cardJson) implements InvokeResponse {}
```

- [ ] **Step 6: Add 3 serialization cases to InvokeResponseSerializer**

Add to `InvokeResponseSerializer.toJson()` switch expression, before the closing `;`, after the `Error` case. Add `import jakarta.json.JsonArrayBuilder;` to the imports.

```java
case InvokeResponse.ComposeResult(var attachmentLayout, var attachmentsJson) -> {
    var attachmentsArray = Json.createArrayBuilder();
    for (String att : attachmentsJson) {
        attachmentsArray.add(parseJson(att));
    }
    yield Json.createObjectBuilder()
        .add("composeExtension", Json.createObjectBuilder()
            .add("type", "result")
            .add("attachmentLayout", attachmentLayout)
            .add("attachments", attachmentsArray))
        .build().toString();
}

case InvokeResponse.ComposeAuth(var suggestedActionsJson) ->
    Json.createObjectBuilder()
        .add("composeExtension", Json.createObjectBuilder()
            .add("type", "auth")
            .add("suggestedActions", parseJson(suggestedActionsJson)))
        .build().toString();

case InvokeResponse.ComposeContinue(var title, var cardJson) ->
    Json.createObjectBuilder()
        .add("composeExtension", Json.createObjectBuilder()
            .add("type", "continue")
            .add("value", Json.createObjectBuilder()
                .add("title", title)
                .add("card", Json.createObjectBuilder()
                    .add("contentType", "application/vnd.microsoft.card.adaptive")
                    .add("content", parseJson(cardJson)))))
        .build().toString();
```

- [ ] **Step 7: Run serializer tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InvokeResponseSerializerTest`

Expected: all 11 tests PASS (6 existing + 5 new).

- [ ] **Step 8: Run all chat-teams tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams`

Expected: all tests PASS. The new sealed variants don't break existing code — the switch in `InvokeResponseSerializer` is exhaustive, and no other switch matches on `InvokeResponse`.

- [ ] **Step 9: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeResponseSerializerTest.java
git commit -m "feat(#236): add ComposeResult, ComposeAuth, ComposeContinue invoke response variants

Add 3 new InvokeResponse variants for messaging extension responses:
- ComposeResult: search results, link unfurling, selectItem
- ComposeAuth: SSO/OAuth auth flow initiation
- ComposeContinue: task module for fetchTask/submitAction

Include serialization in InvokeResponseSerializer and 5 new tests.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Round-trip integration tests

**Files:**
- Modify: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java`

**Interfaces:**
- Consumes: `InvokeResponse.ComposeResult`, `InvokeResponse.ComposeAuth`, `InvokeResponse.ComposeContinue` from Task 1; existing `InvokeHandler`, `InvokeService`, `TeamsInboundConnector`, `WebhookResult.Responded`
- Produces: Verification that the full invoke pipeline works for all `composeExtension/*` invoke names

- [ ] **Step 1: Write round-trip test for composeExtension/query**

Add to `TeamsInboundConnectorTest.java` after the existing invoke tests section:

```java
// ── Messaging extension (composeExtension) activities ────────────────

@Test
void composeExtensionQuery_withHandler_returnsRespondedWithResults() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "composeExtension/query"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\"},\"preview\":{\"contentType\":\"application/vnd.microsoft.card.thumbnail\",\"content\":{\"title\":\"Result\"}}}";
            return new InvokeResponse.ComposeResult("list", List.of(att));
        }
    };
    var conn = connectorWithInvoke(new InvokeService(List.of(handler)));

    String body = invokeActivity("composeExtension/query",
            "{\"queryText\":\"search term\",\"queryOptions\":{\"skip\":0,\"count\":25}}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responded = (WebhookResult.Responded) result;
    var responseObj = parseJson(responded.responseBody());
    var ext = responseObj.getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
    assertThat(ext.getString("attachmentLayout")).isEqualTo("list");
    assertThat(ext.getJsonArray("attachments")).hasSize(1);
    assertThat(responded.messages()).hasSize(1);
    assertThat(responded.messages().get(0).metadata().get("teams-invoke-name"))
            .isEqualTo("composeExtension/query");
}
```

- [ ] **Step 2: Write round-trip test for composeExtension/queryLink**

```java
@Test
void composeExtensionQueryLink_returnsUnfurlCard() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "composeExtension/queryLink"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Unfurled\"}]}}";
            return new InvokeResponse.ComposeResult("list", List.of(att));
        }
    };
    var conn = connectorWithInvoke(new InvokeService(List.of(handler)));

    String body = invokeActivity("composeExtension/queryLink",
            "{\"url\":\"https://example.com/item/42\"}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responded = (WebhookResult.Responded) result;
    var ext = parseJson(responded.responseBody()).getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
    assertThat(ext.getJsonArray("attachments")).hasSize(1);
}
```

- [ ] **Step 3: Write round-trip test for composeExtension/selectItem**

```java
@Test
void composeExtensionSelectItem_returnsSingleCard() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "composeExtension/selectItem"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\"}}";
            return new InvokeResponse.ComposeResult("list", List.of(att));
        }
    };
    var conn = connectorWithInvoke(new InvokeService(List.of(handler)));

    String body = invokeActivity("composeExtension/selectItem",
            "{\"id\":\"item-42\"}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var ext = parseJson(((WebhookResult.Responded) result).responseBody())
            .getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
    assertThat(ext.getJsonArray("attachments")).hasSize(1);
}
```

- [ ] **Step 4: Write round-trip test for composeExtension/fetchTask**

```java
@Test
void composeExtensionFetchTask_returnsTaskModule() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "composeExtension/fetchTask"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            return new InvokeResponse.ComposeContinue("Action Form",
                    "{\"type\":\"AdaptiveCard\",\"body\":[]}");
        }
    };
    var conn = connectorWithInvoke(new InvokeService(List.of(handler)));

    String body = invokeActivity("composeExtension/fetchTask",
            "{\"commandId\":\"createTicket\"}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var ext = parseJson(((WebhookResult.Responded) result).responseBody())
            .getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("continue");
    assertThat(ext.getJsonObject("value").getString("title")).isEqualTo("Action Form");
}
```

- [ ] **Step 5: Write round-trip test for composeExtension/submitAction**

```java
@Test
void composeExtensionSubmitAction_returnsResult() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "composeExtension/submitAction"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            String att = "{\"contentType\":\"application/vnd.microsoft.card.adaptive\",\"content\":{\"type\":\"AdaptiveCard\"}}";
            return new InvokeResponse.ComposeResult("list", List.of(att));
        }
    };
    var conn = connectorWithInvoke(new InvokeService(List.of(handler)));

    String body = invokeActivity("composeExtension/submitAction",
            "{\"commandId\":\"createTicket\",\"data\":{\"title\":\"Bug report\"}}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var ext = parseJson(((WebhookResult.Responded) result).responseBody())
            .getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
}
```

- [ ] **Step 6: Write round-trip test for composeExtension/setting (Accepted)**

```java
@Test
void composeExtensionSetting_returnsAccepted() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "composeExtension/setting"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            return new InvokeResponse.Accepted();
        }
    };
    var conn = connectorWithInvoke(new InvokeService(List.of(handler)));

    String body = invokeActivity("composeExtension/setting",
            "{\"state\":\"saved-settings-json\"}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responded = (WebhookResult.Responded) result;
    var responseObj = parseJson(responded.responseBody());
    assertThat(responseObj.getInt("statusCode")).isEqualTo(200);
    assertThat(responseObj.size()).isEqualTo(1);
}
```

- [ ] **Step 7: Write round-trip test for no-handler fallback**

```java
@Test
void composeExtension_noHandler_returnsAccepted() {
    var conn = connectorWithInvoke(new InvokeService(List.of()));

    String body = invokeActivity("composeExtension/query",
            "{\"queryText\":\"test\"}");
    WebhookResult result = conn.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responseObj = parseJson(((WebhookResult.Responded) result).responseBody());
    assertThat(responseObj.getInt("statusCode")).isEqualTo(200);
    assertThat(responseObj.size()).isEqualTo(1);
}
```

- [ ] **Step 8: Run all round-trip tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest`

Expected: all tests PASS. No production code changes needed — `handleInvoke()` already dispatches through `InvokeService` for all invoke activities.

- [ ] **Step 9: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams`

Expected: all tests PASS.

- [ ] **Step 10: Commit**

```bash
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java
git commit -m "test(#236): add round-trip tests for composeExtension invoke activities

Verify the full invoke pipeline for all 7 messaging extension names:
query, queryLink, selectItem, fetchTask, submitAction, setting, and
no-handler fallback. No production code changes — handleInvoke()
already dispatches through InvokeService for all invoke activities.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-11-17-teams-messaging-extensions-design.md] — design spec this plan implements
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java] — existing sealed interface (6 variants)
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java] — existing serializer (6 switch cases)
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java:225-271] — existing handleInvoke
- [chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeResponseSerializerTest.java] — existing serializer tests (6 tests)
- [chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java] — existing connector tests
- [docs/protocols/connectors/shared-http-client.md] — HttpHelper.CLIENT protocol
- [docs/protocols/connectors/inbound-connector-id-constants.md] — connector ID constants protocol
- [GitHub #236] — chat-teams: Messaging extensions support
- [GitHub #235] — chat-teams: Adaptive Card action handling (invoke activities)
- [GitHub #220] — chat-teams: Teams inbound via Bot Framework
