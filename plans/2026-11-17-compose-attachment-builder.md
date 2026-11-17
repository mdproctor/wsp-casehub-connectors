# ComposeAttachment Builder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #259 — chat-teams: Messaging extension preview card model
**Issue group:** #236, #257, #259, #258

**Goal:** Add a `ComposeAttachment` builder that eliminates manual JSON construction for Teams messaging extension attachments.

**Architecture:** Single utility class in `chat-teams` with a static factory method `adaptive()` returning a builder. The builder produces JSON strings compatible with the existing `ComposeResult(String attachmentLayout, List<String> attachmentsJson)` contract. No changes to existing types.

**Tech Stack:** Java 21, Jakarta JSON-P (`jakarta.json`), JUnit 5, AssertJ

## Global Constraints

- Java 21 source, Java 26 JVM
- No new dependencies — `jakarta.json` already in `chat-teams`
- No changes to `InvokeResponse.ComposeResult` signature
- Raw JSON path always remains available
- This is delivery infrastructure — no business logic

---

## Batch 1: ComposeAttachment builder

### Task 1: ComposeAttachment utility class with TDD

**Files:**
- Create: `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java`
- Test: `chat-teams/src/test/java/io/casehub/connectors/chat/teams/ComposeAttachmentTest.java`

**Interfaces:**
- Consumes: `jakarta.json.Json`, `jakarta.json.JsonObjectBuilder` (Jakarta JSON-P)
- Produces:
  - `ComposeAttachment.adaptive(String contentJson)` → `ComposeAttachment.Builder`
  - `ComposeAttachment.Builder.preview(String title, String subtitle)` → `Builder`
  - `ComposeAttachment.Builder.preview(String title, String subtitle, String imageUrl)` → `Builder`
  - `ComposeAttachment.Builder.build()` → `String` (JSON)

- [ ] **Step 1: Write failing test — adaptive card without preview**

```java
package io.casehub.connectors.chat.teams;

import jakarta.json.Json;
import jakarta.json.JsonObject;
import org.junit.jupiter.api.Test;

import java.io.StringReader;

import static org.assertj.core.api.Assertions.assertThat;

class ComposeAttachmentTest {

    @Test
    void adaptive_withoutPreview_producesContentOnly() {
        String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Hello\"}]}";

        String result = ComposeAttachment.adaptive(cardJson).build();

        JsonObject obj = parse(result);
        assertThat(obj.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
        assertThat(obj.getJsonObject("content").getString("type")).isEqualTo("AdaptiveCard");
        assertThat(obj.containsKey("preview")).isFalse();
    }

    private static JsonObject parse(String json) {
        return Json.createReader(new StringReader(json)).readObject();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=ComposeAttachmentTest#adaptive_withoutPreview_producesContentOnly -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `ComposeAttachment` class does not exist

- [ ] **Step 3: Implement ComposeAttachment with minimal code**

Create `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java`:

```java
package io.casehub.connectors.chat.teams;

import jakarta.json.Json;
import jakarta.json.JsonObjectBuilder;

import java.io.StringReader;

public final class ComposeAttachment {

    private ComposeAttachment() {}

    public static Builder adaptive(String contentJson) {
        return new Builder(contentJson);
    }

    public static final class Builder {
        private final String contentJson;
        private String previewTitle;
        private String previewSubtitle;
        private String previewImageUrl;

        Builder(String contentJson) {
            this.contentJson = contentJson;
        }

        public Builder preview(String title, String subtitle) {
            this.previewTitle = title;
            this.previewSubtitle = subtitle;
            return this;
        }

        public Builder preview(String title, String subtitle, String imageUrl) {
            this.previewTitle = title;
            this.previewSubtitle = subtitle;
            this.previewImageUrl = imageUrl;
            return this;
        }

        public String build() {
            var content = Json.createReader(new StringReader(contentJson)).readObject();
            var builder = Json.createObjectBuilder()
                    .add("contentType", "application/vnd.microsoft.card.adaptive")
                    .add("content", content);

            if (previewTitle != null) {
                var previewContent = Json.createObjectBuilder()
                        .add("title", previewTitle);
                if (previewSubtitle != null) {
                    previewContent.add("subtitle", previewSubtitle);
                }
                if (previewImageUrl != null) {
                    previewContent.add("images", Json.createArrayBuilder()
                            .add(Json.createObjectBuilder().add("url", previewImageUrl)));
                }
                builder.add("preview", Json.createObjectBuilder()
                        .add("contentType", "application/vnd.microsoft.card.thumbnail")
                        .add("content", previewContent));
            }

            return builder.build().toString();
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=ComposeAttachmentTest#adaptive_withoutPreview_producesContentOnly -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write failing test — preview with title and subtitle**

Add to `ComposeAttachmentTest.java`:

```java
@Test
void adaptive_withPreview_includesThumbnailPreview() {
    String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[]}";

    String result = ComposeAttachment.adaptive(cardJson)
            .preview("Product Name", "£29.99")
            .build();

    JsonObject obj = parse(result);
    assertThat(obj.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
    JsonObject preview = obj.getJsonObject("preview");
    assertThat(preview.getString("contentType")).isEqualTo("application/vnd.microsoft.card.thumbnail");
    JsonObject previewContent = preview.getJsonObject("content");
    assertThat(previewContent.getString("title")).isEqualTo("Product Name");
    assertThat(previewContent.getString("subtitle")).isEqualTo("£29.99");
    assertThat(previewContent.containsKey("images")).isFalse();
}
```

- [ ] **Step 6: Run test to verify it passes (already implemented)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=ComposeAttachmentTest#adaptive_withPreview_includesThumbnailPreview -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — preview support already implemented in Step 3

- [ ] **Step 7: Write failing test — preview with image**

Add to `ComposeAttachmentTest.java`:

```java
@Test
void adaptive_withPreviewAndImage_includesImagesArray() {
    String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[]}";

    String result = ComposeAttachment.adaptive(cardJson)
            .preview("Product", "Details", "https://img.example.com/thumb.png")
            .build();

    JsonObject obj = parse(result);
    JsonObject preview = obj.getJsonObject("preview");
    JsonObject previewContent = preview.getJsonObject("content");
    assertThat(previewContent.getString("title")).isEqualTo("Product");
    assertThat(previewContent.getString("subtitle")).isEqualTo("Details");
    assertThat(previewContent.getJsonArray("images")).hasSize(1);
    assertThat(previewContent.getJsonArray("images").getJsonObject(0)
            .getString("url")).isEqualTo("https://img.example.com/thumb.png");
}
```

- [ ] **Step 8: Run test to verify it passes (already implemented)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=ComposeAttachmentTest#adaptive_withPreviewAndImage_includesImagesArray -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — image support already implemented in Step 3

- [ ] **Step 9: Write failing test — content JSON preserved verbatim**

Add to `ComposeAttachmentTest.java`:

```java
@Test
void adaptive_preservesContentJsonVerbatim() {
    String cardJson = "{\"type\":\"AdaptiveCard\",\"$schema\":\"http://adaptivecards.io/schemas/adaptive-card.json\",\"version\":\"1.5\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Complex card\"},{\"type\":\"Image\",\"url\":\"https://img.example.com/photo.png\"}]}";

    String result = ComposeAttachment.adaptive(cardJson).build();

    JsonObject obj = parse(result);
    JsonObject content = obj.getJsonObject("content");
    assertThat(content.getString("type")).isEqualTo("AdaptiveCard");
    assertThat(content.getString("version")).isEqualTo("1.5");
    assertThat(content.getJsonArray("body")).hasSize(2);
}
```

- [ ] **Step 10: Run test to verify it passes (already implemented)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=ComposeAttachmentTest#adaptive_preservesContentJsonVerbatim -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 11: Write failing test — round-trip through InvokeResponseSerializer**

Add to `ComposeAttachmentTest.java`:

```java
@Test
void adaptive_roundTrip_throughInvokeResponseSerializer() {
    String cardJson = "{\"type\":\"AdaptiveCard\",\"body\":[]}";
    String att = ComposeAttachment.adaptive(cardJson)
            .preview("Search Result", "Found item")
            .build();

    String serialized = InvokeResponseSerializer.toJson(
            new InvokeResponse.ComposeResult("list", java.util.List.of(att)));

    JsonObject obj = parse(serialized);
    JsonObject ext = obj.getJsonObject("composeExtension");
    assertThat(ext.getString("type")).isEqualTo("result");
    assertThat(ext.getString("attachmentLayout")).isEqualTo("list");
    assertThat(ext.getJsonArray("attachments")).hasSize(1);
    JsonObject attachment = ext.getJsonArray("attachments").getJsonObject(0);
    assertThat(attachment.getString("contentType")).isEqualTo("application/vnd.microsoft.card.adaptive");
    assertThat(attachment.getJsonObject("preview").getJsonObject("content")
            .getString("title")).isEqualTo("Search Result");
}
```

- [ ] **Step 12: Run test to verify it passes (already implemented)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams -Dtest=ComposeAttachmentTest#adaptive_roundTrip_throughInvokeResponseSerializer -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 13: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-teams`
Expected: All tests PASS

- [ ] **Step 14: Commit**

```bash
git add chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java
git add chat-teams/src/test/java/io/casehub/connectors/chat/teams/ComposeAttachmentTest.java
git commit -m "feat(#259): add ComposeAttachment builder for messaging extension attachments"
```

## References

- [2026-11-17-compose-attachment-builder-design.md] — design spec this plan implements
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java:17] — ComposeResult record
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java:58-68] — ComposeResult serialization
- [chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeResponseSerializerTest.java:80-112] — existing ComposeResult tests
- [chat-teams/src/test/java/io/casehub/connectors/chat/teams/TeamsInboundConnectorTest.java:435-561] — messaging extension handler tests
- [chat-spi/src/main/java/io/casehub/connectors/chat/model/RichCard.java:25-56] — builder pattern precedent
- [GitHub #259] — issue: messaging extension preview card model
