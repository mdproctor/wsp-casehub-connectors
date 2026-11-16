# Teams Adaptive Card Action Handling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #235 — chat-teams: Adaptive Card action handling (invoke activities)
**Issue group:** #235

**Goal:** Handle Bot Framework invoke activities (Adaptive Card actions,
task modules) by adding `WebhookResult.Responded` transport variant, an
`InvokeHandler` CDI SPI for response construction, and invoke routing
in the Teams inbound connector.

**Architecture:** Three layers — (1) transport: `WebhookResult.Responded`
sealed variant that delivers InboundMessages AND returns an HTTP response
body, (2) handler SPI: `InvokeHandler`/`InvokeService` following
`CommandHandler`/`CommandService` pattern with sealed `InvokeResponse`
variants, (3) connector wiring: `case "invoke"` in
`TeamsInboundConnector.doHandle()` dispatching to the handler and
returning `Responded`.

**Tech Stack:** Java 21, Quarkus 3.32.2, jakarta.json, JUnit 5,
AssertJ

## Global Constraints

- Java source level 21, running on JDK 26
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All new types in `chat-teams` module except `WebhookResult.Responded`
  (in `connectors-api`) and its router handling (in `webhook`)
- No business logic in connectors — pure delivery infrastructure
- Follow existing patterns: sealed types, CDI bean discovery, `@All List<T>`

---

## Batch 1: Transport Layer + Type Definitions

After this batch: `WebhookResult.Responded` exists and is handled by the
router. Invoke SPI types exist but are not wired to the connector.

### Task 1: WebhookResult.Responded variant + WebhookRouter handling

**Files:**
- Modify: `connectors-api/src/main/java/io/casehub/connectors/WebhookResult.java:61-63` (add after DeferredDelivery)
- Modify: `webhook/src/main/java/io/casehub/connectors/webhook/WebhookRouter.java:92-127` (add case to switch)
- Modify: `webhook/src/test/java/io/casehub/connectors/webhook/WebhookRouterTest.java` (add test)
- Create: `webhook/src/test/java/io/casehub/connectors/webhook/RespondedTestConnector.java`

**Interfaces:**
- Produces: `WebhookResult.Responded(String responseBody, String contentType, List<InboundMessage> messages)` — used by Task 5

- [ ] **Step 1: Write the failing test in WebhookRouterTest**

Add a test connector that returns `Responded` and a test that verifies
the router both delivers messages AND returns the response body.

First, create `RespondedTestConnector.java`:

```java
package io.casehub.connectors.webhook;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.WebhookInboundConnector;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;
import jakarta.enterprise.context.ApplicationScoped;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class RespondedTestConnector extends WebhookInboundConnector {

    @Override
    public String id() {
        return "responded-test";
    }

    @Override
    public WebhookResult handle(WebhookRequest request) {
        var msg = new InboundMessage(
                "responded-test", "test",
                "user1", "channel1", "invoke-event",
                List.of(), Instant.now(), Map.of(), null);
        return new WebhookResult.Responded(
                "{\"statusCode\":200}", "application/json",
                List.of(msg));
    }
}
```

Then add the test in `WebhookRouterTest`:

```java
@Test
void post_respondedConnector_returns200WithBodyAndFiresEvent() throws InterruptedException {
    given()
        .contentType("application/json")
        .body("{}")
        .when().post("/connectors/responded-test/webhook")
        .then()
        .statusCode(200)
        .contentType("application/json")
        .body(equalTo("{\"statusCode\":200}"));

    InboundMessage msg = capture.poll(2, TimeUnit.SECONDS);
    assertThat(msg).isNotNull();
    assertThat(msg.content()).isEqualTo("invoke-event");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook -Dtest=WebhookRouterTest#post_respondedConnector_returns200WithBodyAndFiresEvent -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: Compilation failure — `WebhookResult.Responded` does not exist.

- [ ] **Step 3: Add Responded variant to WebhookResult**

In `WebhookResult.java`, add after the `DeferredDelivery` record (after line 63):

```java
/**
 * Messages are delivered AND a response body is returned to the caller.
 *
 * <p>Used for Bot Framework invoke activities (Adaptive Card actions, task modules)
 * where the HTTP response body determines what happens in the client
 * (card update, task module dialog, message display).
 */
record Responded(String responseBody, String contentType,
                 List<InboundMessage> messages) implements WebhookResult {}
```

- [ ] **Step 4: Add Responded case to WebhookRouter.dispatch()**

In `WebhookRouter.java`, add a new case in the switch at line 92. Insert
before the `Challenged` case (line 108):

```java
case WebhookResult.Responded(var responseBody, var contentType, var msgs) -> {
    msgs.forEach(service::receive);
    yield Response.ok(responseBody).type(contentType).build();
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook -Dtest=WebhookRouterTest#post_respondedConnector_returns200WithBodyAndFiresEvent -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/WebhookResult.java
git add webhook/src/main/java/io/casehub/connectors/webhook/WebhookRouter.java
git add webhook/src/test/java/io/casehub/connectors/webhook/RespondedTestConnector.java
git add webhook/src/test/java/io/casehub/connectors/webhook/WebhookRouterTest.java
git commit -m "feat(#235): add WebhookResult.Responded variant and router handling"
```

### Task 2: Invoke SPI types — InvokeRequest, InvokeResponse, InvokeHandler

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeRequest.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java`
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java`

**Interfaces:**
- Produces: `InvokeRequest(String name, JsonObject value, String senderId, String channelRef, Map<String, String> metadata)` — used by Tasks 4, 5
- Produces: `InvokeResponse` sealed interface with `Accepted`, `CardUpdate`, `Message`, `TaskModuleContinue`, `TaskModuleMessage`, `Error` — used by Tasks 3, 4, 5
- Produces: `InvokeHandler` interface with `String name()`, `InvokeResponse handle(InvokeRequest request)` — used by Task 4

- [ ] **Step 1: Create InvokeRequest**

```java
package io.casehub.connectors.chat.teams;

import jakarta.json.JsonObject;
import java.util.Map;

public record InvokeRequest(
        String name,
        JsonObject value,
        String senderId,
        String channelRef,
        Map<String, String> metadata) {}
```

- [ ] **Step 2: Create InvokeResponse**

```java
package io.casehub.connectors.chat.teams;

public sealed interface InvokeResponse {

    record Accepted() implements InvokeResponse {}

    record CardUpdate(String cardJson) implements InvokeResponse {}

    record Message(String text) implements InvokeResponse {}

    record TaskModuleContinue(String title, String cardJson) implements InvokeResponse {}

    record TaskModuleMessage(String text) implements InvokeResponse {}

    record Error(int statusCode, String message) implements InvokeResponse {}
}
```

- [ ] **Step 3: Create InvokeHandler**

```java
package io.casehub.connectors.chat.teams;

public interface InvokeHandler {
    String name();
    InvokeResponse handle(InvokeRequest request);
}
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl chat-teams -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeRequest.java
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java
git commit -m "feat(#235): add InvokeRequest, InvokeResponse, InvokeHandler types"
```

---

## Batch 2: Handler Infrastructure

After this batch: `InvokeResponseSerializer` and `InvokeService` exist
with tests. Not yet wired to the connector.

### Task 3: InvokeResponseSerializer + tests

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java`
- Create: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeResponseSerializerTest.java`

**Interfaces:**
- Consumes: `InvokeResponse` sealed interface (from Task 2)
- Produces: `InvokeResponseSerializer.toJson(InvokeResponse)` → `String` — used by Task 5

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.io.StringReader;

class InvokeResponseSerializerTest {

    @Test
    void accepted_returnsStatusCode200() {
        String json = InvokeResponseSerializer.toJson(new InvokeResponse.Accepted());
        JsonObject obj = parse(json);
        assertThat(obj.getInt("statusCode")).isEqualTo(200);
        assertThat(obj.size()).isEqualTo(1);
    }

    @Test
    void cardUpdate_returnsAdaptiveCardType() {
        String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[]}";
        String json = InvokeResponseSerializer.toJson(new InvokeResponse.CardUpdate(cardJson));
        JsonObject obj = parse(json);
        assertThat(obj.getInt("statusCode")).isEqualTo(200);
        assertThat(obj.getString("type")).isEqualTo("application/vnd.microsoft.card.adaptive");
        assertThat(obj.getJsonObject("value").getString("type")).isEqualTo("AdaptiveCard");
    }

    @Test
    void message_returnsActivityMessageType() {
        String json = InvokeResponseSerializer.toJson(new InvokeResponse.Message("Done!"));
        JsonObject obj = parse(json);
        assertThat(obj.getInt("statusCode")).isEqualTo(200);
        assertThat(obj.getString("type")).isEqualTo("application/vnd.microsoft.activity.message");
        assertThat(obj.getString("value")).isEqualTo("Done!");
    }

    @Test
    void taskModuleContinue_returnsTaskContinueEnvelope() {
        String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[]}";
        String json = InvokeResponseSerializer.toJson(
                new InvokeResponse.TaskModuleContinue("My Task", cardJson));
        JsonObject obj = parse(json);
        JsonObject task = obj.getJsonObject("task");
        assertThat(task.getString("type")).isEqualTo("continue");
        JsonObject value = task.getJsonObject("value");
        assertThat(value.getString("title")).isEqualTo("My Task");
        JsonObject card = value.getJsonObject("card");
        assertThat(card.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
        assertThat(card.getJsonObject("content").getString("type")).isEqualTo("AdaptiveCard");
    }

    @Test
    void taskModuleMessage_returnsTaskMessageEnvelope() {
        String json = InvokeResponseSerializer.toJson(
                new InvokeResponse.TaskModuleMessage("Thanks for submitting!"));
        JsonObject obj = parse(json);
        JsonObject task = obj.getJsonObject("task");
        assertThat(task.getString("type")).isEqualTo("message");
        assertThat(task.getString("value")).isEqualTo("Thanks for submitting!");
    }

    @Test
    void error_returnsErrorType() {
        String json = InvokeResponseSerializer.toJson(
                new InvokeResponse.Error(400, "Invalid input"));
        JsonObject obj = parse(json);
        assertThat(obj.getInt("statusCode")).isEqualTo(400);
        assertThat(obj.getString("type")).isEqualTo("application/vnd.microsoft.error");
        JsonObject value = obj.getJsonObject("value");
        assertThat(value.getString("code")).isEqualTo("Error");
        assertThat(value.getString("message")).isEqualTo("Invalid input");
    }

    private static JsonObject parse(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InvokeResponseSerializerTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: Compilation failure — `InvokeResponseSerializer` does not exist.

- [ ] **Step 3: Implement InvokeResponseSerializer**

```java
package io.casehub.connectors.chat.teams;

import jakarta.json.Json;
import jakarta.json.JsonObject;
import jakarta.json.JsonObjectBuilder;

import java.io.StringReader;

class InvokeResponseSerializer {

    static String toJson(InvokeResponse response) {
        return switch (response) {
            case InvokeResponse.Accepted() ->
                Json.createObjectBuilder()
                    .add("statusCode", 200)
                    .build().toString();

            case InvokeResponse.CardUpdate(var cardJson) ->
                Json.createObjectBuilder()
                    .add("statusCode", 200)
                    .add("type", "application/vnd.microsoft.card.adaptive")
                    .add("value", parseJson(cardJson))
                    .build().toString();

            case InvokeResponse.Message(var text) ->
                Json.createObjectBuilder()
                    .add("statusCode", 200)
                    .add("type", "application/vnd.microsoft.activity.message")
                    .add("value", text)
                    .build().toString();

            case InvokeResponse.TaskModuleContinue(var title, var cardJson) ->
                Json.createObjectBuilder()
                    .add("task", Json.createObjectBuilder()
                        .add("type", "continue")
                        .add("value", Json.createObjectBuilder()
                            .add("title", title)
                            .add("card", Json.createObjectBuilder()
                                .add("contentType", "application/vnd.microsoft.card.adaptive")
                                .add("content", parseJson(cardJson)))))
                    .build().toString();

            case InvokeResponse.TaskModuleMessage(var text) ->
                Json.createObjectBuilder()
                    .add("task", Json.createObjectBuilder()
                        .add("type", "message")
                        .add("value", text))
                    .build().toString();

            case InvokeResponse.Error(var statusCode, var message) ->
                Json.createObjectBuilder()
                    .add("statusCode", statusCode)
                    .add("type", "application/vnd.microsoft.error")
                    .add("value", Json.createObjectBuilder()
                        .add("code", "Error")
                        .add("message", message))
                    .build().toString();
        };
    }

    private static JsonObject parseJson(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InvokeResponseSerializerTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: All 6 tests PASS

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeResponseSerializerTest.java
git commit -m "feat(#235): InvokeResponseSerializer with tests for all 6 response variants"
```

### Task 4: InvokeService + tests

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeService.java`
- Create: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeServiceTest.java`

**Interfaces:**
- Consumes: `InvokeHandler` interface (from Task 2), `InvokeRequest` record (from Task 2), `InvokeResponse` sealed interface (from Task 2)
- Produces: `InvokeService(List<InvokeHandler>)` constructor, `InvokeResponse dispatch(InvokeRequest)` method — used by Task 5

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.connectors.chat.teams;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

class InvokeServiceTest {

    private static final JsonObject EMPTY_VALUE = Json.createObjectBuilder().build();

    @Test
    void dispatch_withHandler_delegatesToHandler() {
        var handler = stubHandler("adaptiveCard/action", new InvokeResponse.Message("OK"));
        var service = new InvokeService(List.of(handler));

        var request = new InvokeRequest("adaptiveCard/action", EMPTY_VALUE,
                "user1", "channel1", Map.of());
        InvokeResponse result = service.dispatch(request);

        assertThat(result).isInstanceOf(InvokeResponse.Message.class);
        assertThat(((InvokeResponse.Message) result).text()).isEqualTo("OK");
    }

    @Test
    void dispatch_noHandler_returnsAccepted() {
        var service = new InvokeService(List.of());

        var request = new InvokeRequest("unknown/action", EMPTY_VALUE,
                "user1", "channel1", Map.of());
        InvokeResponse result = service.dispatch(request);

        assertThat(result).isInstanceOf(InvokeResponse.Accepted.class);
    }

    @Test
    void dispatch_handlerThrows_returnsAccepted() {
        InvokeHandler throwing = new InvokeHandler() {
            @Override public String name() { return "adaptiveCard/action"; }
            @Override public InvokeResponse handle(InvokeRequest r) {
                throw new RuntimeException("boom");
            }
        };
        var service = new InvokeService(List.of(throwing));

        var request = new InvokeRequest("adaptiveCard/action", EMPTY_VALUE,
                "user1", "channel1", Map.of());
        InvokeResponse result = service.dispatch(request);

        assertThat(result).isInstanceOf(InvokeResponse.Accepted.class);
    }

    @Test
    void constructor_duplicateHandlerName_throws() {
        var h1 = stubHandler("adaptiveCard/action", new InvokeResponse.Accepted());
        var h2 = stubHandler("adaptiveCard/action", new InvokeResponse.Accepted());

        assertThatThrownBy(() -> new InvokeService(List.of(h1, h2)))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("adaptiveCard/action");
    }

    @Test
    void dispatch_multipleHandlers_routesByName() {
        var cardHandler = stubHandler("adaptiveCard/action", new InvokeResponse.CardUpdate("{}"));
        var taskHandler = stubHandler("task/fetch", new InvokeResponse.TaskModuleMessage("Hi"));
        var service = new InvokeService(List.of(cardHandler, taskHandler));

        var cardRequest = new InvokeRequest("adaptiveCard/action", EMPTY_VALUE,
                "user1", "channel1", Map.of());
        assertThat(service.dispatch(cardRequest)).isInstanceOf(InvokeResponse.CardUpdate.class);

        var taskRequest = new InvokeRequest("task/fetch", EMPTY_VALUE,
                "user1", "channel1", Map.of());
        assertThat(service.dispatch(taskRequest)).isInstanceOf(InvokeResponse.TaskModuleMessage.class);
    }

    private static InvokeHandler stubHandler(String name, InvokeResponse response) {
        return new InvokeHandler() {
            @Override public String name() { return name; }
            @Override public InvokeResponse handle(InvokeRequest r) { return response; }
        };
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InvokeServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: Compilation failure — `InvokeService` does not exist.

- [ ] **Step 3: Implement InvokeService**

```java
package io.casehub.connectors.chat.teams;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.stream.Collectors;

public class InvokeService {

    private static final Logger LOG = Logger.getLogger(InvokeService.class.getName());

    private final Map<String, InvokeHandler> handlers;

    public InvokeService(List<InvokeHandler> handlers) {
        this.handlers = handlers.stream()
                .collect(Collectors.toMap(
                        InvokeHandler::name,
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate invoke handler name: '" + a.name() + "'");
                        }));
    }

    public InvokeResponse dispatch(InvokeRequest request) {
        InvokeHandler handler = handlers.get(request.name());
        if (handler == null) {
            LOG.fine("teams-invoke: no handler for: " + request.name());
            return new InvokeResponse.Accepted();
        }
        try {
            return handler.handle(request);
        } catch (Exception e) {
            LOG.log(Level.WARNING, "teams-invoke: handler failed for: " + request.name(), e);
            return new InvokeResponse.Accepted();
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=InvokeServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeService.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeServiceTest.java
git commit -m "feat(#235): InvokeService with handler dispatch, fallback, and exception safety"
```

---

## Batch 3: Connector Wiring + Build Verification

After this batch: invoke activities are fully handled end-to-end. Full
build passes.

### Task 5: TeamsInboundConnector invoke handling + TeamsBeans wiring

**Files:**
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java:29-100` (add field, constructors, handleInvoke, switch case)
- Modify: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java:57-74` (add InvokeService parameter)
- Modify: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java` (add invoke tests)

**Interfaces:**
- Consumes: `WebhookResult.Responded` (from Task 1), `InvokeService.dispatch(InvokeRequest)` (from Task 4), `InvokeResponseSerializer.toJson(InvokeResponse)` (from Task 3), `InvokeRequest` (from Task 2)

- [ ] **Step 1: Write the failing tests**

Add to `TeamsInboundConnectorTest.java`:

```java
@Test
void invokeActivity_withHandler_returnsRespondedWithHandlerResponse() {
    var handler = new InvokeHandler() {
        @Override public String name() { return "adaptiveCard/action"; }
        @Override public InvokeResponse handle(InvokeRequest r) {
            return new InvokeResponse.Message("Action received");
        }
    };
    var invokeService = new InvokeService(List.of(handler));
    var connector = new TeamsInboundConnector(
            stubAuth(), stubClient(), stubCommandService(), "bot-id", null, invokeService);

    String body = invokeActivity("adaptiveCard/action",
            "{\"actionVerb\":\"approve\"}");
    WebhookResult result = connector.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responded = (WebhookResult.Responded) result;
    assertThat(responded.contentType()).isEqualTo("application/json");
    assertThat(responded.messages()).hasSize(1);

    var msg = responded.messages().get(0);
    assertThat(msg.metadata().get("teams-invoke-name")).isEqualTo("adaptiveCard/action");

    // Verify response body contains the handler's message response
    var responseObj = parseJson(responded.responseBody());
    assertThat(responseObj.getInt("statusCode")).isEqualTo(200);
    assertThat(responseObj.getString("type"))
            .isEqualTo("application/vnd.microsoft.activity.message");
}

@Test
void invokeActivity_noHandler_returnsRespondedWithAccept() {
    var invokeService = new InvokeService(List.of());
    var connector = new TeamsInboundConnector(
            stubAuth(), stubClient(), stubCommandService(), "bot-id", null, invokeService);

    String body = invokeActivity("unknown/type", "{}");
    WebhookResult result = connector.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responded = (WebhookResult.Responded) result;
    assertThat(responded.messages()).hasSize(1);

    var responseObj = parseJson(responded.responseBody());
    assertThat(responseObj.getInt("statusCode")).isEqualTo(200);
    assertThat(responseObj.size()).isEqualTo(1);
}

@Test
void invokeActivity_nullInvokeService_returnsRespondedWithAccept() {
    var connector = new TeamsInboundConnector(
            stubAuth(), stubClient(), stubCommandService(), "bot-id", null, null);

    String body = invokeActivity("adaptiveCard/action", "{\"actionVerb\":\"ok\"}");
    WebhookResult result = connector.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Responded.class);
    var responded = (WebhookResult.Responded) result;
    assertThat(responded.messages()).hasSize(1);
}

@Test
void invokeActivity_selfMessage_returnsIgnored() {
    var connector = new TeamsInboundConnector(
            stubAuth(), stubClient(), stubCommandService(), "bot-id", null, null);

    String body = invokeActivityFromBot("adaptiveCard/action", "{}", "bot-id");
    WebhookResult result = connector.handle(postRequest(body));

    assertThat(result).isInstanceOf(WebhookResult.Ignored.class);
}

@Test
void invokeActivity_metadataContainsInvokeName() {
    var invokeService = new InvokeService(List.of());
    var connector = new TeamsInboundConnector(
            stubAuth(), stubClient(), stubCommandService(), "bot-id", null, invokeService);

    String body = invokeActivity("task/fetch", "{\"data\":\"test\"}");
    WebhookResult result = connector.handle(postRequest(body));

    var responded = (WebhookResult.Responded) result;
    var msg = responded.messages().get(0);
    assertThat(msg.metadata().get("teams-activity-type")).isEqualTo("invoke");
    assertThat(msg.metadata().get("teams-invoke-name")).isEqualTo("task/fetch");
    assertThat(msg.metadata().get("teams-invoke-data")).isEqualTo("{\"data\":\"test\"}");
}
```

Add helper methods to `TeamsInboundConnectorTest`:

```java
private static String invokeActivity(String name, String valueJson) {
    return """
            {"type":"invoke","name":"%s","value":%s,\
            "from":{"id":"user-aad-id","aadObjectId":"user-aad-id","name":"Test User"},\
            "conversation":{"id":"conv-1"},\
            "channelData":{"tenant":{"id":"tenant-1"}},\
            "serviceUrl":"https://smba.trafficmanager.net/uk/",\
            "timestamp":"2026-11-16T10:00:00Z","id":"act-1"}""".formatted(name, valueJson);
}

private static String invokeActivityFromBot(String name, String valueJson, String botId) {
    return """
            {"type":"invoke","name":"%s","value":%s,\
            "from":{"id":"%s","aadObjectId":"%s","name":"Bot"},\
            "conversation":{"id":"conv-1"},\
            "channelData":{"tenant":{"id":"tenant-1"}},\
            "serviceUrl":"https://smba.trafficmanager.net/uk/",\
            "timestamp":"2026-11-16T10:00:00Z","id":"act-1"}""".formatted(name, valueJson, botId, botId);
}

private static jakarta.json.JsonObject parseJson(String json) {
    return jakarta.json.Json.createReader(new java.io.StringReader(json)).readObject();
}
```

Note: the existing `setUp()` method creates the connector with the 5-arg
constructor. Existing tests are unaffected since `invokeService` defaults
to `null` (backward compat). The new invoke tests create their own
connector instances with the 6-arg constructor.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: Compilation failure — 6-arg constructor does not exist,
`handleInvoke` method does not exist.

- [ ] **Step 3: Add InvokeService field and constructors to TeamsInboundConnector**

Add field at line 37 (after `conversationReferenceStore`):

```java
private final InvokeService invokeService;
```

Update the existing 5-arg constructor (lines 43-51) to chain to a new
6-arg constructor:

```java
TeamsInboundConnector(BotFrameworkAuth auth, BotFrameworkClient client,
                      CommandService commandService, String appId,
                      ConversationReferenceStore conversationReferenceStore) {
    this(auth, client, commandService, appId, conversationReferenceStore, null);
}

TeamsInboundConnector(BotFrameworkAuth auth, BotFrameworkClient client,
                      CommandService commandService, String appId,
                      ConversationReferenceStore conversationReferenceStore,
                      InvokeService invokeService) {
    this.auth                       = auth;
    this.client                     = client;
    this.commandService             = commandService;
    this.appId                      = appId;
    this.conversationReferenceStore = conversationReferenceStore;
    this.invokeService              = invokeService;
}
```

Keep the existing 4-arg constructor chaining to the 5-arg (which now
chains to the 6-arg).

- [ ] **Step 4: Add handleInvoke method and switch case**

Add `case "invoke"` to the switch in `doHandle()` (line 91):

```java
return switch (type) {
    case "message" -> handleMessage(activity);
    case "messageReaction" -> handleMessageReaction(activity);
    case "conversationUpdate" -> handleConversationUpdate(activity);
    case "invoke" -> handleInvoke(activity);
    default -> {
        LOG.fine("teams-inbound: ignoring activity type: " + type);
        yield new WebhookResult.Ignored();
    }
};
```

Add `handleInvoke` method (after `handleConversationUpdate`):

```java
private WebhookResult handleInvoke(JsonObject activity) {
    String invokeName = activity.getString("name", "");
    JsonObject value = activity.containsKey("value") && !activity.isNull("value")
                       ? activity.getJsonObject("value")
                       : Json.createObjectBuilder().build();

    Map<String, String> metadata = buildBaseMetadata(activity);
    metadata.put("teams-activity-type", "invoke");
    metadata.put("teams-invoke-name", invokeName);
    metadata.put("teams-invoke-data", value.toString());
    String fromName = extractFromName(activity);
    if (fromName != null) {
        metadata.put("teams-from-name", fromName);
    }

    String timestamp = activity.getString("timestamp", null);
    Instant ts = timestamp != null ? Instant.parse(timestamp) : Instant.now();

    var msg = new InboundMessage(
            InboundConnectorIds.TEAMS_INBOUND,
            InboundConnectorTypes.TEAMS,
            extractSenderId(activity),
            extractChannelRef(activity),
            "",
            List.of(),
            ts,
            metadata,
            extractTenancyId(activity));

    InvokeResponse response;
    if (invokeService != null) {
        var request = new InvokeRequest(invokeName, value,
                extractSenderId(activity), extractChannelRef(activity), metadata);
        response = invokeService.dispatch(request);
    } else {
        response = new InvokeResponse.Accepted();
    }

    String responseJson = InvokeResponseSerializer.toJson(response);
    return new WebhookResult.Responded(responseJson, "application/json", List.of(msg));
}
```

- [ ] **Step 5: Update TeamsBeans producer**

In `TeamsBeans.java`, add `InvokeService` parameter and pass it to the
constructor. Modify the producer method (lines 57-74):

```java
@Produces
@ApplicationScoped
io.casehub.connectors.WebhookInboundConnector teamsInboundConnector(
        @ConfigProperty(name = "casehub.connectors.teams-bot.app-id", defaultValue = "") String appId,
        @ConfigProperty(name = "casehub.connectors.teams-bot.dev-mode", defaultValue = "false") boolean devMode,
        @ConfigProperty(name = "casehub.connectors.teams-bot.service-url-allowlist",
                        defaultValue = "*.botframework.com,smba.trafficmanager.net") String allowlist,
        @ConfigProperty(name = "casehub.connectors.teams-bot.openid-metadata-url",
                        defaultValue = "https://login.botframework.com/v1/.well-known/openidconfiguration") String metadataUrl,
        @ConfigProperty(name = "casehub.connectors.teams-bot.issuer",
                        defaultValue = "https://api.botframework.com") String issuer,
        io.casehub.connectors.chat.command.CommandService commandService,
        ServiceConnectionProvider connectionProvider,
        ConversationReferenceStore conversationReferenceStore,
        InvokeService invokeService) {
    var auth   = new BotFrameworkAuth(appId, devMode, metadataUrl, issuer);
    var client = new BotFrameworkClient(connectionProvider, allowlist);
    return new TeamsInboundConnector(auth, client, commandService, appId,
            conversationReferenceStore, invokeService);
}
```

Add `InvokeService` producer method to `TeamsBeans.java` (CDI needs to
construct it from discovered `InvokeHandler` beans):

```java
@Produces
@ApplicationScoped
InvokeService invokeService(@io.quarkus.arc.All List<InvokeHandler> handlers) {
    return new InvokeService(handlers);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=TeamsInboundConnectorTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: All tests PASS (existing + 5 new invoke tests)

- [ ] **Step 7: Full build verification**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: BUILD SUCCESS — all modules compile, all tests pass.

- [ ] **Step 8: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java
git commit -m "feat(#235): wire invoke activity handling in TeamsInboundConnector"
```

---

## References

- [2026-11-16-teams-adaptive-card-actions-design.md] — design spec this plan implements
- [decisions.md] — 6 captured design decisions
- `connectors-api/src/main/java/io/casehub/connectors/WebhookResult.java:23` — sealed interface to extend
- `webhook/src/main/java/io/casehub/connectors/webhook/WebhookRouter.java:92` — dispatch switch to update
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsInboundConnector.java:90` — activity type switch
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java:73` — CDI producer to update
- `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandHandler.java:3` — prior art for handler SPI
- `chat-spi/src/main/java/io/casehub/connectors/chat/command/CommandService.java:13` — prior art for dispatch
- `webhook/src/test/java/io/casehub/connectors/webhook/WebhookRouterTest.java:19` — router test to extend
- `chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java:16` — connector test to extend
- [GitHub #235] — focal issue
- [GitHub #220 decisions D4, D6] — pre-validated Responded variant
