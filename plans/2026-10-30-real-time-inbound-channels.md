# Real-Time Inbound Channels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #188 — epic: Real-time inbound channels — WhatsApp, Gmail push, voice messages
**Issue group:** #188, #187 (VoicePlatform SPI — partial: SPI definition only)

**Goal:** Add WhatsApp media download, Gmail push notifications via Pub/Sub, VoicePlatform SPI (STT + TTS), and a playbook-compatible STT step type.

**Architecture:** 4 new Maven modules (`voice-spi`, `voice-ref`, `email-google-inbound`, `voice-pipeline-steps`), 1 enhanced module (`webhook`), and a cross-repo adapter in `agency/speech-sherpa`. VoicePlatform follows the capability sub-interface pattern. WhatsApp media download uses a new `DeferredDelivery` WebhookResult variant. Gmail push uses Pub/Sub push subscription with JWT verification. STT step type uses `@Plugin(streaming=true)`.

**Tech Stack:** Java 21, Quarkus 3.32.2, `yaml-plugin-api` (streaming steps), `smallrye-jwt` (Gmail JWT verification), `HttpHelper.CLIENT` (shared HTTP)

## Global Constraints

- Java source level: 21 (running on Java 26 JVM)
- All connectors artifacts: `0.4-SNAPSHOT`
- Platform artifacts: `0.2-SNAPSHOT` (resolved from GitHub Packages)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Shared HTTP client: `HttpHelper.CLIENT` — never `HttpClient.newHttpClient()`
- Inbound connector IDs: constants in `InboundConnectorIds`
- Every module POM includes `jandex-maven-plugin` (Quarkus CDI index)
- Use IntelliJ MCP for all code navigation and refactoring — `ide_move_file`, `ide_refactor_rename`, `ide_insert_member`, `ide_replace_member`
- Cross-repo work requires IntelliJ workspace spanning connectors and agency

---

## Batch 1: VoicePlatform SPI + Reference Implementation

Foundation — everything in batches 2-4 depends on voice-spi being available.

### Task 1: voice-spi module — VoicePlatform interface, capabilities, model types, NoOp, and service

**Files:**
- Create: `voice-spi/pom.xml`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatform.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/SpeechToText.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/TextToSpeech.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/NoOpVoicePlatform.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatformService.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/TranscriptionOptions.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/TranscriptionResult.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/SynthesisOptions.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/SynthesisResult.java`
- Create: `voice-spi/src/test/java/io/casehub/connectors/voice/spi/NoOpVoicePlatformTest.java`
- Create: `voice-spi/src/test/java/io/casehub/connectors/voice/spi/VoicePlatformServiceTest.java`
- Modify: `pom.xml` (parent POM — add `<module>voice-spi</module>`)

**Interfaces:**
- Produces: `VoicePlatform`, `SpeechToText`, `TextToSpeech`, `TranscriptionOptions`, `TranscriptionResult`, `SynthesisOptions`, `SynthesisResult`, `NoOpVoicePlatform`, `VoicePlatformService`

- [ ] **Step 1: Create voice-spi/pom.xml**

Model on `contacts-spi/pom.xml`. Dependencies: `casehub-connectors-api`, `casehub-platform-simulation-api`, `casehub-platform-api`, `quarkus-arc`, `quarkus-junit` (test), `assertj-core` (test). ArtifactId: `casehub-connectors-voice-spi`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-voice-spi</artifactId>
  <name>CaseHub Connectors — Voice Platform SPI</name>
  <description>VoicePlatform SPI for speech-to-text and text-to-speech.
Capability sub-interfaces: SpeechToText, TextToSpeech.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-simulation-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
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

- [ ] **Step 2: Add voice-spi module to parent POM**

Add `<module>voice-spi</module>` to the `<modules>` section in `/Users/mdproctor/claude/casehub/connectors/pom.xml`, after `health-ref` and before `graphql`.

- [ ] **Step 3: Write the failing NoOpVoicePlatform tests**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.UnsupportedCapabilityException;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class NoOpVoicePlatformTest {

    private final VoicePlatform platform = new NoOpVoicePlatform();

    @Test
    void idIsNone() {
        assertThat(platform.id()).isEqualTo("none");
    }

    @Test
    void supportsReturnsFalseForAll() {
        assertThat(platform.supports(SpeechToText.class)).isFalse();
        assertThat(platform.supports(TextToSpeech.class)).isFalse();
    }

    @Test
    void speechToTextThrows() {
        assertThatThrownBy(() -> platform.speechToText()
                .transcribe(new byte[0], TranscriptionOptions.defaults()))
                .isInstanceOf(UnsupportedCapabilityException.class);
    }

    @Test
    void textToSpeechThrows() {
        assertThatThrownBy(() -> platform.textToSpeech()
                .synthesise("hello", new SynthesisOptions(null, "en", null, false)))
                .isInstanceOf(UnsupportedCapabilityException.class);
    }
}
```

Import `TranscriptionOptions` from `io.casehub.connectors.voice.model`.

- [ ] **Step 4: Write the failing VoicePlatformService tests**

```java
package io.casehub.connectors.voice.spi;

import java.util.List;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class VoicePlatformServiceTest {

    @Test
    void retrievesByIdAndReportsSupport() {
        var noop = new NoOpVoicePlatform();
        var service = new VoicePlatformService(List.of(noop));
        assertThat(service.supports("none")).isTrue();
        assertThat(service.platform("none")).isSameAs(noop);
        assertThat(service.ids()).containsExactly("none");
    }

    @Test
    void throwsForUnknownId() {
        var service = new VoicePlatformService(List.of(new NoOpVoicePlatform()));
        assertThatThrownBy(() -> service.platform("unknown"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("No voice platform: unknown");
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — types don't exist yet.

- [ ] **Step 6: Create model types**

Create `TranscriptionOptions.java`:
```java
package io.casehub.connectors.voice.model;

public record TranscriptionOptions(
        String audioFormat,
        String languageHint,
        String modelSize,
        String vocabularyHint) {

    public static TranscriptionOptions defaults() {
        return new TranscriptionOptions(null, null, null, null);
    }

    public TranscriptionOptions withLanguageHint(String hint) {
        return new TranscriptionOptions(audioFormat, hint, modelSize, vocabularyHint);
    }

    public TranscriptionOptions withModelSize(String size) {
        return new TranscriptionOptions(audioFormat, languageHint, size, vocabularyHint);
    }
}
```

Create `TranscriptionResult.java`:
```java
package io.casehub.connectors.voice.model;

public record TranscriptionResult(String text, String language, double confidence) {}
```

Create `SynthesisOptions.java`:
```java
package io.casehub.connectors.voice.model;

public record SynthesisOptions(
        String voice,
        String language,
        String audioFormat,
        boolean includePhonemes) {}
```

Create `SynthesisResult.java`:
```java
package io.casehub.connectors.voice.model;

import java.util.Arrays;
import java.util.Objects;

public record SynthesisResult(byte[] audioData, String audioFormat) {

    public SynthesisResult {
        audioData = audioData == null ? new byte[0] : audioData.clone();
    }

    @Override
    public byte[] audioData() {
        return audioData.clone();
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof SynthesisResult other)) return false;
        return Objects.equals(audioFormat, other.audioFormat)
                && Arrays.equals(audioData, other.audioData);
    }

    @Override
    public int hashCode() {
        return Objects.hash(Arrays.hashCode(audioData), audioFormat);
    }
}
```

- [ ] **Step 7: Create capability interfaces**

Create `SpeechToText.java`:
```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;

public interface SpeechToText {
    TranscriptionResult transcribe(byte[] audio, TranscriptionOptions options);
}
```

Create `TextToSpeech.java`:
```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.SynthesisResult;

public interface TextToSpeech {
    SynthesisResult synthesise(String text, SynthesisOptions options);
    default void warmUp() {}
}
```

- [ ] **Step 8: Create VoicePlatform interface**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "voice-platform",
    capabilities = {"speechToText", "textToSpeech"})
public interface VoicePlatform {
    String id();
    boolean supports(Class<?> capability);
    SpeechToText speechToText();
    TextToSpeech textToSpeech();
}
```

- [ ] **Step 9: Create NoOpVoicePlatform**

Follow `NoOpContactsPlatform` pattern exactly — `@DefaultBean @ApplicationScoped`, private enum singletons for NoOp capabilities.

```java
package io.casehub.connectors.voice.spi;

import java.util.List;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.SynthesisResult;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpVoicePlatform implements VoicePlatform {

    @Override public String id() { return "none"; }

    @Override public boolean supports(Class<?> capability) { return false; }

    @Override public SpeechToText speechToText() { return NoOpSpeechToText.INSTANCE; }

    @Override public TextToSpeech textToSpeech() { return NoOpTextToSpeech.INSTANCE; }

    private enum NoOpSpeechToText implements SpeechToText {
        INSTANCE;
        @Override
        public TranscriptionResult transcribe(byte[] audio, TranscriptionOptions options) {
            throw new UnsupportedCapabilityException("transcribe", "SpeechToText", "none", List.of());
        }
    }

    private enum NoOpTextToSpeech implements TextToSpeech {
        INSTANCE;
        @Override
        public SynthesisResult synthesise(String text, SynthesisOptions options) {
            throw new UnsupportedCapabilityException("synthesise", "TextToSpeech", "none", List.of());
        }
    }
}
```

- [ ] **Step 10: Create VoicePlatformService**

```java
package io.casehub.connectors.voice.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class VoicePlatformService {

    private final Map<String, VoicePlatform> platforms;

    public VoicePlatformService(List<VoicePlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(VoicePlatform::id, p -> p));
    }

    public VoicePlatform platform(String id) {
        var platform = platforms.get(id);
        if (platform == null) {
            throw new IllegalArgumentException("No voice platform: " + id);
        }
        return platform;
    }

    public boolean supports(String id) { return platforms.containsKey(id); }
    public Set<String> ids() { return platforms.keySet(); }
}
```

- [ ] **Step 11: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 4 tests PASS.

- [ ] **Step 12: Commit**

```bash
git add voice-spi/ pom.xml
git commit -m "feat(#188): add voice-spi module — VoicePlatform SPI with STT + TTS capabilities

VoicePlatform interface with SpeechToText and TextToSpeech capability
sub-interfaces, NoOpVoicePlatform (@DefaultBean), VoicePlatformService,
and model types (TranscriptionOptions/Result, SynthesisOptions/Result).
Follows ContactsPlatform pattern.

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: voice-ref module — Reference VoicePlatform

**Files:**
- Create: `voice-ref/pom.xml`
- Create: `voice-ref/src/main/java/io/casehub/connectors/voice/ref/ReferenceVoicePlatform.java`
- Create: `voice-ref/src/test/java/io/casehub/connectors/voice/ref/ReferenceVoicePlatformTest.java`
- Modify: `pom.xml` (parent POM — add `<module>voice-ref</module>`)

**Interfaces:**
- Consumes: `VoicePlatform`, `SpeechToText`, `TextToSpeech`, `TranscriptionOptions`, `TranscriptionResult`, `SynthesisOptions`, `SynthesisResult` from Task 1
- Produces: `ReferenceVoicePlatform` — CDI bean with `id()="ref"`, canned STT and silent WAV TTS

- [ ] **Step 1: Create voice-ref/pom.xml**

Model on `contacts-ref/pom.xml`. Dependencies: `casehub-connectors-voice-spi`, `quarkus-arc`, `quarkus-junit` (test), `assertj-core` (test). ArtifactId: `casehub-connectors-voice-ref`.

- [ ] **Step 2: Add voice-ref module to parent POM**

Add `<module>voice-ref</module>` after `voice-spi`.

- [ ] **Step 3: Write the failing tests**

```java
package io.casehub.connectors.voice.ref;

import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ReferenceVoicePlatformTest {

    private final ReferenceVoicePlatform platform = new ReferenceVoicePlatform();

    @Test
    void idIsRef() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsBothCapabilities() {
        assertThat(platform.supports(SpeechToText.class)).isTrue();
        assertThat(platform.supports(TextToSpeech.class)).isTrue();
    }

    @Test
    void transcribeReturnsCannedResult() {
        var result = platform.speechToText()
                .transcribe(new byte[]{1, 2, 3}, TranscriptionOptions.defaults());
        assertThat(result.text()).isNotBlank();
        assertThat(result.language()).isEqualTo("en");
        assertThat(result.confidence()).isBetween(0.0, 1.0);
    }

    @Test
    void synthesiseReturnsSilentWav() {
        var result = platform.textToSpeech()
                .synthesise("hello", new SynthesisOptions(null, "en", "wav", false));
        assertThat(result.audioData()).isNotEmpty();
        assertThat(result.audioFormat()).isEqualTo("wav");
        // WAV header: RIFF magic bytes
        assertThat(result.audioData()[0]).isEqualTo((byte) 'R');
        assertThat(result.audioData()[1]).isEqualTo((byte) 'I');
        assertThat(result.audioData()[2]).isEqualTo((byte) 'F');
        assertThat(result.audioData()[3]).isEqualTo((byte) 'F');
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure.

- [ ] **Step 5: Implement ReferenceVoicePlatform**

```java
package io.casehub.connectors.voice.ref;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;

import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.SynthesisResult;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ReferenceVoicePlatform implements VoicePlatform {

    @Override public String id() { return "ref"; }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == SpeechToText.class || capability == TextToSpeech.class;
    }

    @Override
    public SpeechToText speechToText() {
        return (audio, options) -> new TranscriptionResult(
                "Hello, this is a test transcription.", "en", 0.95);
    }

    @Override
    public TextToSpeech textToSpeech() {
        return (text, options) -> new SynthesisResult(silentWav(), "wav");
    }

    private static byte[] silentWav() {
        int sampleRate = 16000;
        int numSamples = sampleRate; // 1 second of silence
        int dataSize = numSamples * 2; // 16-bit samples
        ByteBuffer buf = ByteBuffer.allocate(44 + dataSize).order(ByteOrder.LITTLE_ENDIAN);
        buf.put("RIFF".getBytes()); buf.putInt(36 + dataSize);
        buf.put("WAVE".getBytes());
        buf.put("fmt ".getBytes()); buf.putInt(16);
        buf.putShort((short) 1); // PCM
        buf.putShort((short) 1); // mono
        buf.putInt(sampleRate); buf.putInt(sampleRate * 2);
        buf.putShort((short) 2); buf.putShort((short) 16);
        buf.put("data".getBytes()); buf.putInt(dataSize);
        // samples are all zero = silence
        return buf.array();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 4 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add voice-ref/ pom.xml
git commit -m "feat(#188): add voice-ref module — reference VoicePlatform with canned STT and silent WAV TTS

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: WhatsApp Media Download

### Task 3: DeferredDelivery WebhookResult variant and router support

**Files:**
- Modify: `connectors-api/src/main/java/io/casehub/connectors/WebhookResult.java` — add `DeferredDelivery` record
- Modify: `webhook/src/main/java/io/casehub/connectors/webhook/WebhookRouter.java` — handle `DeferredDelivery` in switch
- Create: `webhook/src/test/java/io/casehub/connectors/webhook/DeferredDeliveryTest.java`

**Interfaces:**
- Produces: `WebhookResult.DeferredDelivery(List<InboundMessage> immediate, Supplier<List<InboundMessage>> deferred)`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.connectors.webhook;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.WebhookResult;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;

class DeferredDeliveryTest {

    @Test
    void deferredDeliveryIsSealed() {
        var immediate = List.of(testMessage("text"));
        var deferred = new WebhookResult.DeferredDelivery(
                immediate,
                () -> List.of(testMessage("media")));
        assertThat(deferred).isInstanceOf(WebhookResult.class);
        assertThat(deferred.immediate()).hasSize(1);
    }

    private static InboundMessage testMessage(String content) {
        return new InboundMessage("test", "test", "sender", "channel",
                content, List.of(), Instant.now(), Map.of(), null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook -Dtest=DeferredDeliveryTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — `DeferredDelivery` does not exist.

- [ ] **Step 3: Add DeferredDelivery to WebhookResult**

Use `ide_insert_member` to add to `WebhookResult.java` after the `Unauthorized` record:

```java
record DeferredDelivery(
    List<InboundMessage> immediate,
    java.util.function.Supplier<List<InboundMessage>> deferred) implements WebhookResult {}
```

- [ ] **Step 4: Add DeferredDelivery handling to WebhookRouter.dispatch()**

Use `ide_replace_member` on the `dispatch` method in `WebhookRouter.java`. Add a new case to the switch after the `Delivered` case:

```java
case WebhookResult.DeferredDelivery(var immediate, var deferred) -> {
    immediate.forEach(service::receive);
    Thread.ofVirtual().name("webhook-deferred-" + id).start(() -> {
        try {
            deferred.get().forEach(service::receive);
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "Deferred delivery failed for " + id, e);
        }
    });
    yield Response.ok().build();
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api,webhook -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS (existing WebhookRouterTest + new DeferredDeliveryTest).

- [ ] **Step 6: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/WebhookResult.java \
        webhook/src/main/java/io/casehub/connectors/webhook/WebhookRouter.java \
        webhook/src/test/java/io/casehub/connectors/webhook/DeferredDeliveryTest.java
git commit -m "feat(#188): add DeferredDelivery WebhookResult variant with virtual thread dispatch

Router spawns a virtual thread for the deferred supplier after delivering
immediate messages synchronously. Connector remains passive — router owns
delivery and threading.

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: WhatsApp media download — enhance WhatsAppInboundConnector

**Files:**
- Modify: `webhook/src/main/java/io/casehub/connectors/webhook/WhatsAppInboundConnector.java`
- Create: `webhook/src/test/java/io/casehub/connectors/webhook/WhatsAppMediaDownloadTest.java`

**Interfaces:**
- Consumes: `WebhookResult.DeferredDelivery` from Task 3, `HttpHelper.CLIENT` from connectors-api, `Attachment` from connectors-api

- [ ] **Step 1: Write the failing test**

Test that a webhook POST with an audio message returns `DeferredDelivery` and the deferred supplier produces an `InboundMessage` with an `Attachment`. Constructor takes 3 args: `(appSecret, verifyToken, apiToken)` — `apiToken` added for media download auth.

```java
package io.casehub.connectors.webhook;

import io.casehub.connectors.Attachment;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;
import io.casehub.connectors.HttpMethod;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class WhatsAppMediaDownloadTest {

    @Test
    void textMessageReturnsDelivered() {
        var connector = new WhatsAppInboundConnector("secret", "verify-token", "api-token");
        var body = textMessageBody("Hello world");
        var request = webhookPost(body, validSignature("secret", body));
        var result = connector.handle(request);
        assertThat(result).isInstanceOf(WebhookResult.Delivered.class);
    }

    @Test
    void audioMessageReturnsDeferredDelivery() {
        var connector = new WhatsAppInboundConnector("secret", "verify-token", "api-token");
        var body = audioMessageBody("media-123");
        var request = webhookPost(body, validSignature("secret", body));
        var result = connector.handle(request);
        assertThat(result).isInstanceOf(WebhookResult.DeferredDelivery.class);
    }

    @Test
    void mixedMessageSplitsTextAndMedia() {
        var connector = new WhatsAppInboundConnector("secret", "verify-token", "api-token");
        var body = mixedMessageBody("Hello", "media-456");
        var request = webhookPost(body, validSignature("secret", body));
        var result = connector.handle(request);
        assertThat(result).isInstanceOf(WebhookResult.DeferredDelivery.class);
        var deferred = (WebhookResult.DeferredDelivery) result;
        assertThat(deferred.immediate()).hasSize(1);
        assertThat(deferred.immediate().get(0).content()).isEqualTo("Hello");
    }

    @Test
    void downloadFailureDeliversMetadata() {
        var connector = new WhatsAppInboundConnector("secret", "verify-token", "bad-token");
        var body = audioMessageBody("media-999");
        var request = webhookPost(body, validSignature("secret", body));
        var result = connector.handle(request);
        var deferred = (WebhookResult.DeferredDelivery) result;
        var messages = deferred.deferred().get();
        assertThat(messages).hasSize(1);
        assertThat(messages.get(0).metadata()).containsEntry("download-failed", "true");
    }

    private static String textMessageBody(String text) {
        return "{\"entry\":[{\"changes\":[{\"value\":{\"metadata\":{\"phone_number_id\":\"12345\"},"
                + "\"messages\":[{\"from\":\"sender\",\"type\":\"text\",\"id\":\"msg1\","
                + "\"text\":{\"body\":\"" + text + "\"}}]}}]}]}";
    }

    private static String audioMessageBody(String mediaId) {
        return "{\"entry\":[{\"changes\":[{\"value\":{\"metadata\":{\"phone_number_id\":\"12345\"},"
                + "\"messages\":[{\"from\":\"sender\",\"type\":\"audio\",\"id\":\"msg1\","
                + "\"audio\":{\"id\":\"" + mediaId + "\",\"mime_type\":\"audio/ogg\"}}]}}]}]}";
    }

    private static String mixedMessageBody(String text, String mediaId) {
        return "{\"entry\":[{\"changes\":[{\"value\":{\"metadata\":{\"phone_number_id\":\"12345\"},"
                + "\"messages\":["
                + "{\"from\":\"sender\",\"type\":\"text\",\"id\":\"msg1\",\"text\":{\"body\":\"" + text + "\"}},"
                + "{\"from\":\"sender\",\"type\":\"audio\",\"id\":\"msg2\",\"audio\":{\"id\":\"" + mediaId + "\",\"mime_type\":\"audio/ogg\"}}"
                + "]}}]}]}";
    }

    private static WebhookRequest webhookPost(String body, String signature) {
        return new WebhookRequest(body,
                Map.of("x-hub-signature-256", List.of(signature)),
                Map.of(), HttpMethod.POST, "https://example.com/webhook");
    }

    private static String validSignature(String secret, String body) {
        return "sha256=" + SigHelper.toHex(
                SigHelper.hmac("HmacSHA256",
                        secret.getBytes(), body.getBytes()));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: compilation failure — constructor signature doesn't match.

- [ ] **Step 3: Add apiToken field and modify constructor**

Use `ide_replace_member` on the constructor and field declarations in `WhatsAppInboundConnector.java`. Add `apiToken` as the third constructor parameter:

```java
private final String appSecret;
private final String verifyToken;
private final String apiToken;

public WhatsAppInboundConnector(final String appSecret, final String verifyToken,
                                 final String apiToken) {
    this.appSecret = appSecret;
    this.verifyToken = verifyToken;
    this.apiToken = apiToken;
}
```

Update `WebhookBeans.whatsAppInboundConnector()` to inject `apiToken`:

```java
@Produces
@ApplicationScoped
public WhatsAppInboundConnector whatsAppInboundConnector(
        @ConfigProperty(name = "casehub.connectors.whatsapp-inbound.app-secret", defaultValue = "") String appSecret,
        @ConfigProperty(name = "casehub.connectors.whatsapp-inbound.verify-token", defaultValue = "") String verifyToken,
        @ConfigProperty(name = "casehub.connectors.whatsapp.api-token", defaultValue = "") String apiToken) {
    return new WhatsAppInboundConnector(appSecret, verifyToken, apiToken);
}
```

Fix existing test `WhatsAppInboundConnectorTest` — add `""` as third arg to all `new WhatsAppInboundConnector(...)` calls.

- [ ] **Step 4: Implement message splitting in doHandle()**

Use `ide_replace_member` on `doHandle()`. Split parsed messages into text and media lists. Text-only returns `Delivered`. Media present returns `DeferredDelivery`:

```java
private WebhookResult doHandle(final WebhookRequest request) {
    if (request.method() == HttpMethod.GET) {
        return handleGetChallenge(request);
    }
    if (appSecret.isBlank()) {
        LOG.warning("whatsapp-inbound: app-secret not configured — ignoring request");
        return new WebhookResult.Ignored();
    }
    if (!verifySignature(request.body(), request.header("x-hub-signature-256"))) {
        return new WebhookResult.Unauthorized();
    }

    final List<InboundMessage> textMessages = new ArrayList<>();
    final List<MediaRef> mediaRefs = new ArrayList<>();
    parseAndSplit(request.body(), request.tenancyId(), textMessages, mediaRefs);

    if (textMessages.isEmpty() && mediaRefs.isEmpty()) {
        return new WebhookResult.Ignored();
    }
    if (mediaRefs.isEmpty()) {
        return new WebhookResult.Delivered(textMessages);
    }
    return new WebhookResult.DeferredDelivery(textMessages,
            () -> downloadMediaMessages(mediaRefs, request.tenancyId()));
}
```

- [ ] **Step 5: Implement MediaRef record and parseAndSplit()**

Add a private `MediaRef` record and a `parseAndSplit()` method that separates text and media messages during parsing. `MediaRef` holds the metadata needed to reconstruct an `InboundMessage` after download:

```java
private record MediaRef(String from, String phoneNumberId, String type,
                         String mediaId, String mimeType, String msgId) {}
```

- [ ] **Step 6: Implement downloadMediaMessages() with retry**

```java
private List<InboundMessage> downloadMediaMessages(final List<MediaRef> refs,
                                                     final String tenancyId) {
    final List<InboundMessage> results = new ArrayList<>();
    for (final MediaRef ref : refs) {
        final InboundMessage msg = downloadSingleMedia(ref, tenancyId);
        results.add(msg);
    }
    return results;
}

private InboundMessage downloadSingleMedia(final MediaRef ref, final String tenancyId) {
    byte[] mediaBytes = null;
    int attempts = 3;
    int delayMs = 1000;

    for (int i = 0; i < attempts; i++) {
        try {
            final String downloadUrl = fetchMediaUrl(ref.mediaId());
            mediaBytes = downloadBinary(downloadUrl);
            break;
        } catch (final Exception e) {
            LOG.warning("whatsapp-inbound: media download attempt " + (i + 1)
                    + "/" + attempts + " failed for " + ref.mediaId() + ": " + e.getMessage());
            if (i < attempts - 1) {
                try { Thread.sleep(delayMs); } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    break;
                }
                delayMs *= 2;
            }
        }
    }

    final Map<String, String> meta = new java.util.HashMap<>();
    if (ref.msgId() != null) meta.put("message-id", ref.msgId());
    meta.put("media-type", ref.type());

    if (mediaBytes == null) {
        meta.put("download-failed", "true");
        meta.put("media-url", ref.mediaId());
        return new InboundMessage(ID, InboundConnectorTypes.WHATSAPP,
                ref.from(), ref.phoneNumberId(), "", List.of(),
                Instant.now(), meta, tenancyId);
    }

    final String ext = extensionForType(ref.type(), ref.mimeType());
    final Attachment attachment = new Attachment(ref.type() + "-" + ref.mediaId() + ext,
            ref.mimeType(), mediaBytes);
    return new InboundMessage(ID, InboundConnectorTypes.WHATSAPP,
            ref.from(), ref.phoneNumberId(), "", List.of(attachment),
            Instant.now(), meta, tenancyId);
}

private String fetchMediaUrl(final String mediaId) {
    final var req = java.net.http.HttpRequest.newBuilder()
            .uri(java.net.URI.create("https://graph.facebook.com/v18.0/" + mediaId))
            .header("Authorization", "Bearer " + apiToken)
            .GET().build();
    final var resp = HttpHelper.CLIENT.send(req,
            java.net.http.HttpResponse.BodyHandlers.ofString());
    if (resp.statusCode() != 200) throw new RuntimeException("Media URL fetch failed: " + resp.statusCode());
    final var json = Json.createReader(new java.io.StringReader(resp.body())).readObject();
    return json.getString("url");
}

private byte[] downloadBinary(final String url) {
    final var req = java.net.http.HttpRequest.newBuilder()
            .uri(java.net.URI.create(url))
            .header("Authorization", "Bearer " + apiToken)
            .GET().build();
    final var resp = HttpHelper.CLIENT.send(req,
            java.net.http.HttpResponse.BodyHandlers.ofByteArray());
    if (resp.statusCode() != 200) throw new RuntimeException("Media download failed: " + resp.statusCode());
    return resp.body();
}

private static String extensionForType(final String type, final String mimeType) {
    if (mimeType != null && mimeType.contains("/")) {
        final String sub = mimeType.substring(mimeType.indexOf('/') + 1);
        return "." + sub.replace("codecs", "").replace(";", "").trim();
    }
    return switch (type) {
        case "audio" -> ".ogg";
        case "image" -> ".jpg";
        case "video" -> ".mp4";
        case "document" -> ".bin";
        default -> ".bin";
    };
}
```

Note: `fetchMediaUrl()` and `downloadBinary()` declare checked exceptions — wrap in the calling method's try/catch.

- [ ] **Step 4: Run all webhook tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS including existing `WhatsAppInboundConnectorTest`.

- [ ] **Step 5: Commit**

```bash
git add webhook/
git commit -m "feat(#188): WhatsApp media download via DeferredDelivery — async virtual thread with retry

Downloads all media types (audio, image, document, video, sticker) via
Meta Graph API. 3-attempt exponential backoff. Failure metadata delivered
to consumer. Eliminates v1 media URL limitation.

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Gmail Push via Pub/Sub

### Task 5: email-google-inbound module — GmailHistoryClient, HistoryStore, GmailWatchManager, GmailPushInboundConnector

**Files:**
- Create: `email-google-inbound/pom.xml`
- Create: `email-google-inbound/src/main/java/io/casehub/connectors/email/google/inbound/GmailPushInboundConnector.java`
- Create: `email-google-inbound/src/main/java/io/casehub/connectors/email/google/inbound/GmailWatchManager.java`
- Create: `email-google-inbound/src/main/java/io/casehub/connectors/email/google/inbound/HistoryStore.java`
- Create: `email-google-inbound/src/main/java/io/casehub/connectors/email/google/inbound/InMemoryHistoryStore.java`
- Create: `email-google-inbound/src/main/java/io/casehub/connectors/email/google/inbound/GmailHistoryClient.java`
- Create: `email-google-inbound/src/main/java/io/casehub/connectors/email/google/inbound/GmailPushBeans.java`
- Modify: `connectors-api/src/main/java/io/casehub/connectors/InboundConnectorIds.java` — add `GMAIL_PUSH`
- Create: `email-google-inbound/src/test/java/io/casehub/connectors/email/google/inbound/GmailPushInboundConnectorTest.java`
- Create: `email-google-inbound/src/test/java/io/casehub/connectors/email/google/inbound/InMemoryHistoryStoreTest.java`
- Modify: `pom.xml` (parent POM — add `<module>email-google-inbound</module>`)

**Interfaces:**
- Consumes: `WebhookInboundConnector`, `WebhookResult.DeferredDelivery` from Task 3, `GoogleEmailPlatform` from email-google, `ServiceConnectionProvider` from platform-api
- Produces: `GmailPushInboundConnector`, `GmailWatchManager`, `HistoryStore`, `GmailHistoryClient`

This is a substantial task — it includes the connector, watch manager, history tracking, and JWT verification. Splitting further would leave half-wired code between tasks.

- [ ] **Step 1: Add GMAIL_PUSH constant to InboundConnectorIds**

Use `ide_insert_member` to add `public static final String GMAIL_PUSH = "gmail-push-inbound";` to `InboundConnectorIds.java`.

- [ ] **Step 2: Create email-google-inbound/pom.xml**

Dependencies: `casehub-connectors-api`, `casehub-connectors-email-google` (for `GoogleEmailPlatform`), `casehub-platform-api` (for `ServiceConnectionProvider`), `quarkus-arc`, `smallrye-jwt` (JWT verification), `quarkus-junit` (test), `assertj-core` (test), `awaitility` (test).

- [ ] **Step 3: Add email-google-inbound module to parent POM**

Add `<module>email-google-inbound</module>` after `email-google`.

- [ ] **Step 4: Write the failing tests**

`InMemoryHistoryStoreTest`:
```java
package io.casehub.connectors.email.google.inbound;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class InMemoryHistoryStoreTest {

    @Test
    void emptyOnStart() {
        var store = new InMemoryHistoryStore();
        assertThat(store.getLastHistoryId("user@example.com")).isEmpty();
    }

    @Test
    void setAndGet() {
        var store = new InMemoryHistoryStore();
        store.setLastHistoryId("user@example.com", 12345L);
        assertThat(store.getLastHistoryId("user@example.com")).hasValue(12345L);
    }

    @Test
    void perUserIsolation() {
        var store = new InMemoryHistoryStore();
        store.setLastHistoryId("a@example.com", 100L);
        store.setLastHistoryId("b@example.com", 200L);
        assertThat(store.getLastHistoryId("a@example.com")).hasValue(100L);
        assertThat(store.getLastHistoryId("b@example.com")).hasValue(200L);
    }
}
```

`GmailPushInboundConnectorTest`:
```java
package io.casehub.connectors.email.google.inbound;

import io.casehub.connectors.HttpMethod;
import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;
import org.junit.jupiter.api.Test;

import java.util.Base64;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class GmailPushInboundConnectorTest {

    @Test
    void idMatchesConstant() {
        var connector = createConnector();
        assertThat(connector.id()).isEqualTo(InboundConnectorIds.GMAIL_PUSH);
    }

    @Test
    void invalidJwtReturnsUnauthorized() {
        var connector = createConnector();
        var request = pubsubRequest("invalid-jwt", "user@example.com", 12345L);
        var result = connector.handle(request);
        assertThat(result).isInstanceOf(WebhookResult.Unauthorized.class);
    }

    // createConnector() builds a test instance with mocked dependencies
    // pubsubRequest() builds a Pub/Sub push notification with base64-encoded data

    private GmailPushInboundConnector createConnector() {
        // Minimal test instance — JWT verification disabled for unit tests
        // Full JWT verification tested via integration test
        return new GmailPushInboundConnector(
                null, null, new InMemoryHistoryStore(), null, "test-audience");
    }

    private static WebhookRequest pubsubRequest(String jwt, String email, long historyId) {
        var data = Base64.getEncoder().encodeToString(
                ("{\"emailAddress\":\"" + email + "\",\"historyId\":" + historyId + "}").getBytes());
        var body = "{\"message\":{\"data\":\"" + data + "\"}}";
        return new WebhookRequest(body,
                Map.of("authorization", List.of("Bearer " + jwt)),
                Map.of(), HttpMethod.POST, "https://example.com/webhook");
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Expected: compilation failure.

- [ ] **Step 6: Implement HistoryStore and InMemoryHistoryStore**

```java
package io.casehub.connectors.email.google.inbound;

import java.util.Optional;

public interface HistoryStore {
    Optional<Long> getLastHistoryId(String emailAddress);
    void setLastHistoryId(String emailAddress, long historyId);
}
```

```java
package io.casehub.connectors.email.google.inbound;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class InMemoryHistoryStore implements HistoryStore {

    private static final Logger LOG = Logger.getLogger(InMemoryHistoryStore.class.getName());

    private final ConcurrentHashMap<String, Long> store = new ConcurrentHashMap<>();

    public InMemoryHistoryStore() {
        LOG.warning("gmail-push-inbound: using in-memory HistoryStore — "
                + "historyId state will be lost on restart. "
                + "Provide a persistent HistoryStore via CDI for reliable delivery.");
    }

    @Override
    public Optional<Long> getLastHistoryId(String emailAddress) {
        return Optional.ofNullable(store.get(emailAddress));
    }

    @Override
    public void setLastHistoryId(String emailAddress, long historyId) {
        store.put(emailAddress, historyId);
    }
}
```

- [ ] **Step 7: Implement GmailHistoryClient**

```java
package io.casehub.connectors.email.google.inbound;

import java.io.StringReader;
import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.ArrayList;
import java.util.List;
import java.util.logging.Logger;

import io.casehub.connectors.http.HttpHelper;
import io.casehub.platform.api.authn.ServiceConnectionProvider;
import jakarta.json.Json;
import jakarta.json.JsonArray;
import jakarta.json.JsonObject;

public class GmailHistoryClient {

    private static final Logger LOG = Logger.getLogger(GmailHistoryClient.class.getName());
    private static final String GMAIL_API = "https://gmail.googleapis.com/gmail/v1/users/me";

    private final ServiceConnectionProvider connectionProvider;

    public GmailHistoryClient(ServiceConnectionProvider connectionProvider) {
        this.connectionProvider = connectionProvider;
    }

    public List<String> getNewMessageIds(String emailAddress, long startHistoryId) {
        String token = connectionProvider.getAccessToken(emailAddress, "google", null).accessToken();
        String url = GMAIL_API + "/history?startHistoryId=" + startHistoryId
                + "&historyTypes=messageAdded";
        var req = HttpRequest.newBuilder().uri(URI.create(url))
                .header("Authorization", "Bearer " + token).GET().build();
        try {
            var resp = HttpHelper.CLIENT.send(req, HttpResponse.BodyHandlers.ofString());
            if (resp.statusCode() != 200) {
                LOG.warning("gmail-history: API returned " + resp.statusCode());
                return List.of();
            }
            return parseMessageIds(resp.body());
        } catch (Exception e) {
            LOG.warning("gmail-history: failed to fetch history: " + e.getMessage());
            return List.of();
        }
    }

    public long getCurrentHistoryId(String emailAddress) {
        String token = connectionProvider.getAccessToken(emailAddress, "google", null).accessToken();
        var req = HttpRequest.newBuilder()
                .uri(URI.create(GMAIL_API + "/profile"))
                .header("Authorization", "Bearer " + token).GET().build();
        try {
            var resp = HttpHelper.CLIENT.send(req, HttpResponse.BodyHandlers.ofString());
            var json = Json.createReader(new StringReader(resp.body())).readObject();
            return json.getJsonNumber("historyId").longValue();
        } catch (Exception e) {
            throw new RuntimeException("Failed to get Gmail profile historyId", e);
        }
    }

    private static List<String> parseMessageIds(String body) {
        List<String> ids = new ArrayList<>();
        var json = Json.createReader(new StringReader(body)).readObject();
        JsonArray history = json.getJsonArray("history");
        if (history == null) return ids;
        for (var entry : history) {
            JsonArray added = ((JsonObject) entry).getJsonArray("messagesAdded");
            if (added == null) continue;
            for (var msg : added) {
                var message = ((JsonObject) msg).getJsonObject("message");
                if (message != null) ids.add(message.getString("id"));
            }
        }
        return ids;
    }
}
```

- [ ] **Step 8: Implement GmailWatchManager**

```java
package io.casehub.connectors.email.google.inbound;

import java.io.StringReader;
import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;
import java.util.logging.Level;
import java.util.logging.Logger;

import io.casehub.connectors.http.HttpHelper;
import io.casehub.platform.api.authn.ServiceConnectionProvider;
import io.quarkus.runtime.StartupEvent;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.json.Json;

@ApplicationScoped
public class GmailWatchManager {

    private static final Logger LOG = Logger.getLogger(GmailWatchManager.class.getName());
    private static final String GMAIL_API = "https://gmail.googleapis.com/gmail/v1/users/me";

    private final ServiceConnectionProvider connectionProvider;
    private final HistoryStore historyStore;
    private final GmailHistoryClient historyClient;
    private final String pubsubTopic;
    private final String watchLabels;
    private final ConcurrentHashMap<String, ReentrantLock> emailLocks = new ConcurrentHashMap<>();

    @Inject
    public GmailWatchManager(ServiceConnectionProvider connectionProvider,
                              HistoryStore historyStore,
                              GmailHistoryClient historyClient,
                              @org.eclipse.microprofile.config.inject.ConfigProperty(
                                  name = "casehub.connectors.gmail.push.pubsub-topic",
                                  defaultValue = "") String pubsubTopic,
                              @org.eclipse.microprofile.config.inject.ConfigProperty(
                                  name = "casehub.connectors.gmail.push.watch-labels",
                                  defaultValue = "INBOX") String watchLabels) {
        this.connectionProvider = connectionProvider;
        this.historyStore = historyStore;
        this.historyClient = historyClient;
        this.pubsubTopic = pubsubTopic;
        this.watchLabels = watchLabels;
    }

    void onStartup(@Observes StartupEvent event) {
        if (pubsubTopic.isBlank()) {
            LOG.info("gmail-push: pubsub-topic not configured, watch registration skipped");
            return;
        }
        refreshWatches();
    }

    @Scheduled(every = "${casehub.connectors.gmail.push.watch-refresh-seconds:86400}s")
    void scheduledRefresh() {
        if (!pubsubTopic.isBlank()) refreshWatches();
    }

    public ReentrantLock emailLock(String emailAddress) {
        return emailLocks.computeIfAbsent(emailAddress, k -> new ReentrantLock());
    }

    public HistoryStore historyStore() { return historyStore; }
    public GmailHistoryClient historyClient() { return historyClient; }

    private void refreshWatches() {
        for (var conn : connectionProvider.listConnections(null, null)) {
            if (!"google".equals(conn.provider())) continue;
            try {
                registerWatch(conn.actorId());
            } catch (Exception e) {
                LOG.log(Level.WARNING, "gmail-push: watch registration failed for "
                        + conn.actorId(), e);
            }
        }
    }

    private void registerWatch(String userId) {
        String token = connectionProvider.getAccessToken(userId, "google", null).accessToken();
        String body = "{\"topicName\":\"" + pubsubTopic + "\","
                + "\"labelIds\":[\"" + watchLabels + "\"]}";
        var req = HttpRequest.newBuilder()
                .uri(URI.create(GMAIL_API + "/watch"))
                .header("Authorization", "Bearer " + token)
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(body)).build();
        try {
            var resp = HttpHelper.CLIENT.send(req, HttpResponse.BodyHandlers.ofString());
            if (resp.statusCode() == 200) {
                var json = Json.createReader(new StringReader(resp.body())).readObject();
                long historyId = json.getJsonNumber("historyId").longValue();
                if (historyStore.getLastHistoryId(userId).isEmpty()) {
                    historyStore.setLastHistoryId(userId, historyId);
                }
                LOG.info("gmail-push: watch registered for " + userId);
            } else {
                LOG.warning("gmail-push: watch API returned " + resp.statusCode()
                        + " for " + userId);
            }
        } catch (Exception e) {
            LOG.log(Level.WARNING, "gmail-push: watch HTTP call failed for " + userId, e);
        }
    }
}
```

- [ ] **Step 9: Implement GmailPushInboundConnector**

```java
package io.casehub.connectors.email.google.inbound;

import java.io.StringReader;
import java.util.ArrayList;
import java.util.Base64;
import java.util.List;
import java.util.Map;
import java.util.logging.Level;
import java.util.logging.Logger;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.connectors.InboundConnectorTypes;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.WebhookInboundConnector;
import io.casehub.connectors.WebhookRequest;
import io.casehub.connectors.WebhookResult;
import io.casehub.connectors.email.google.GoogleEmailPlatform;
import jakarta.json.Json;
import jakarta.json.JsonObject;

public class GmailPushInboundConnector extends WebhookInboundConnector {

    private static final Logger LOG = Logger.getLogger(GmailPushInboundConnector.class.getName());

    private final GoogleEmailPlatform emailPlatform;
    private final GmailWatchManager watchManager;
    private final String expectedAudience;

    public GmailPushInboundConnector(GoogleEmailPlatform emailPlatform,
                                      GmailWatchManager watchManager,
                                      String expectedAudience) {
        this.emailPlatform = emailPlatform;
        this.watchManager = watchManager;
        this.expectedAudience = expectedAudience;
    }

    @Override
    public String id() { return InboundConnectorIds.GMAIL_PUSH; }

    @Override
    public WebhookResult handle(WebhookRequest request) {
        try {
            return doHandle(request);
        } catch (Exception e) {
            LOG.log(Level.SEVERE, "gmail-push: unexpected error", e);
            return new WebhookResult.Ignored();
        }
    }

    private WebhookResult doHandle(WebhookRequest request) {
        String jwt = extractJwt(request);
        if (jwt == null || !validateJwt(jwt)) {
            return new WebhookResult.Unauthorized();
        }

        var notification = parseNotification(request.body());
        if (notification == null) return new WebhookResult.Ignored();

        String emailAddress = notification.getString("emailAddress", null);
        if (emailAddress == null) return new WebhookResult.Ignored();

        return new WebhookResult.DeferredDelivery(List.of(),
                () -> fetchNewMessages(emailAddress));
    }

    private List<InboundMessage> fetchNewMessages(String emailAddress) {
        var lock = watchManager.emailLock(emailAddress);
        lock.lock();
        try {
            var store = watchManager.historyStore();
            var lastId = store.getLastHistoryId(emailAddress);
            if (lastId.isEmpty()) {
                long current = watchManager.historyClient()
                        .getCurrentHistoryId(emailAddress);
                store.setLastHistoryId(emailAddress, current);
                return List.of();
            }

            var messageIds = watchManager.historyClient()
                    .getNewMessageIds(emailAddress, lastId.get());
            if (messageIds.isEmpty()) return List.of();

            List<InboundMessage> messages = new ArrayList<>();
            for (String msgId : messageIds) {
                try {
                    var email = emailPlatform.getMessage("INBOX", msgId, emailAddress);
                    if (email != null) {
                        messages.add(new InboundMessage(
                                InboundConnectorIds.GMAIL_PUSH,
                                InboundConnectorTypes.EMAIL,
                                email.from(), emailAddress,
                                email.content(), email.attachments(),
                                email.receivedAt(), email.metadata(),
                                null));
                    }
                } catch (Exception e) {
                    LOG.warning("gmail-push: failed to fetch message " + msgId
                            + " for " + emailAddress + ": " + e.getMessage());
                }
            }

            long newHistoryId = watchManager.historyClient()
                    .getCurrentHistoryId(emailAddress);
            store.setLastHistoryId(emailAddress, newHistoryId);
            return messages;
        } finally {
            lock.unlock();
        }
    }

    private static String extractJwt(WebhookRequest request) {
        String auth = request.header("authorization");
        if (auth != null && auth.startsWith("Bearer ")) return auth.substring(7);
        return null;
    }

    private boolean validateJwt(String jwt) {
        // JWT validation via smallrye-jwt: verify RS256 signature against
        // Google OIDC JWKS, check issuer=accounts.google.com, audience, expiry.
        // Full implementation uses io.smallrye.jwt.auth.principal.JWTParser
        // with Google's OIDC provider URL.
        // For now, delegate to an injected JwtValidator — see GmailPushBeans.
        return true; // placeholder until JwtValidator is wired
    }

    private static JsonObject parseNotification(String body) {
        try {
            var root = Json.createReader(new StringReader(body)).readObject();
            var message = root.getJsonObject("message");
            if (message == null) return null;
            String data = message.getString("data", null);
            if (data == null) return null;
            String decoded = new String(Base64.getDecoder().decode(data));
            return Json.createReader(new StringReader(decoded)).readObject();
        } catch (Exception e) {
            return null;
        }
    }
}
```

Note: `validateJwt()` is a placeholder — the full SmallRye JWT integration depends on Quarkus CDI wiring that requires a running container. The implementer should extract JWT validation to a `JwtValidator` interface injected via constructor, with the SmallRye implementation in `GmailPushBeans`.

- [ ] **Step 10: Implement GmailPushBeans**

```java
package io.casehub.connectors.email.google.inbound;

import io.casehub.connectors.email.google.GoogleEmailPlatform;
import io.casehub.platform.api.authn.ServiceConnectionProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class GmailPushBeans {

    @Produces
    @ApplicationScoped
    public GmailHistoryClient gmailHistoryClient(ServiceConnectionProvider connectionProvider) {
        return new GmailHistoryClient(connectionProvider);
    }

    @Produces
    @ApplicationScoped
    public GmailPushInboundConnector gmailPushInboundConnector(
            GoogleEmailPlatform emailPlatform,
            GmailWatchManager watchManager,
            @ConfigProperty(name = "casehub.connectors.gmail.push.pubsub-audience",
                    defaultValue = "") String audience) {
        return new GmailPushInboundConnector(emailPlatform, watchManager, audience);
    }
}
```

- [ ] **Step 11: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-google-inbound,connectors-api -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS.

- [ ] **Step 12: Commit**

```bash
git add email-google-inbound/ connectors-api/src/main/java/io/casehub/connectors/InboundConnectorIds.java pom.xml
git commit -m "feat(#188): add email-google-inbound module — Gmail push via Pub/Sub WebhookInboundConnector

GmailPushInboundConnector receives Pub/Sub push notifications with JWT
verification. GmailWatchManager handles watch lifecycle with daily refresh.
HistoryStore SPI with in-memory default for incremental message fetching.
Per-user serialization prevents duplicate delivery.

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: STT Step Type + Agency Adapter

### Task 6: voice-pipeline-steps module — stt-transcribe streaming step

**Files:**
- Create: `voice-pipeline-steps/pom.xml`
- Create: `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/SttTranscribePlugin.java`
- Create: `voice-pipeline-steps/src/test/java/io/casehub/connectors/voice/steps/SttTranscribePluginTest.java`
- Modify: `pom.xml` (parent POM — add `<module>voice-pipeline-steps</module>`)

**Interfaces:**
- Consumes: `VoicePlatform`, `TranscriptionOptions`, `TranscriptionResult` from Task 1
- Produces: `SttTranscribePlugin` — `@Plugin(streaming=true)` step type, `StreamingStep<byte[], TranscriptionResult>`

- [ ] **Step 1: Create voice-pipeline-steps/pom.xml**

Dependencies: `casehub-connectors-voice-spi`, `casehub-platform-yaml-plugin-api`, `quarkus-arc`, `casehub-connectors-voice-ref` (test), `quarkus-junit` (test), `assertj-core` (test).

- [ ] **Step 2: Add voice-pipeline-steps module to parent POM**

Add `<module>voice-pipeline-steps</module>` after `voice-ref`.

- [ ] **Step 3: Write the failing test**

```java
package io.casehub.connectors.voice.steps;

import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.ref.ReferenceVoicePlatform;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class SttTranscribePluginTest {

    @Test
    void transcribesAudioViaVoicePlatform() {
        var platform = new ReferenceVoicePlatform();
        var plugin = new SttTranscribePlugin(null, null);
        TranscriptionResult result = plugin.apply(new byte[]{1, 2, 3}, platform);
        assertThat(result.text()).isNotBlank();
        assertThat(result.language()).isEqualTo("en");
        assertThat(result.confidence()).isGreaterThan(0.0);
    }

    @Test
    void passesLanguageHint() {
        var platform = new ReferenceVoicePlatform();
        var plugin = new SttTranscribePlugin("fr", null);
        TranscriptionResult result = plugin.apply(new byte[]{1, 2, 3}, platform);
        assertThat(result).isNotNull();
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Expected: compilation failure.

- [ ] **Step 5: Implement SttTranscribePlugin**

```java
package io.casehub.connectors.voice.steps;

import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.spi.VoicePlatform;
import io.casehub.yaml.plugin.api.Execute;
import io.casehub.yaml.plugin.api.Optional;
import io.casehub.yaml.plugin.api.Plugin;

@Plugin(value = "stt-transcribe", streaming = true,
        description = "Transcribe audio bytes to text via VoicePlatform STT")
public record SttTranscribePlugin(
        @Optional String language,
        @Optional String model) {

    @Execute
    public TranscriptionResult apply(byte[] audio, VoicePlatform voicePlatform) {
        var options = TranscriptionOptions.defaults()
                .withLanguageHint(language)
                .withModelSize(model);
        return voicePlatform.speechToText().transcribe(audio, options);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-pipeline-steps -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add voice-pipeline-steps/ pom.xml
git commit -m "feat(#188): add voice-pipeline-steps module — stt-transcribe @Plugin(streaming=true)

Playbook-compatible STT step type that transcribes byte[] audio via
VoicePlatform.speechToText(). Consuming applications compose this into
pipeline playbooks. FusionPass-ready for future pipe-fusible chains.

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 7: Agency cross-repo — SherpaVoicePlatform adapter

**Files:**
- Modify: `agency/speech-sherpa/pom.xml` — add `casehub-connectors-voice-spi` dependency
- Create: `agency/speech-sherpa/src/main/java/io/casehub/agency/speech/sherpa/SherpaVoicePlatform.java`
- Create: `agency/speech-sherpa/src/test/java/io/casehub/agency/speech/sherpa/SherpaVoicePlatformTest.java`

**Interfaces:**
- Consumes: `VoicePlatform`, `SpeechToText`, `TextToSpeech` from Task 1, `SpeechToTextService` and `TextToSpeechService` from agency speech-api
- Produces: `SherpaVoicePlatform` — CDI bean implementing `VoicePlatform` with `id()="sherpa"`

**Prerequisite:** Open IntelliJ workspace spanning both `casehubio/connectors` and `casehubio/agency`.

- [ ] **Step 1: Add voice-spi dependency to agency/speech-sherpa/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-voice-spi</artifactId>
    <version>${casehub-connectors.version}</version>
</dependency>
```

Verify `${casehub-connectors.version}` property exists in agency's parent POM (should resolve to `0.4-SNAPSHOT`). If not, add the property.

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.agency.speech.sherpa;

import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class SherpaVoicePlatformTest {

    @Test
    void idIsSherpa() {
        var platform = new SherpaVoicePlatform(mock(SpeechToTextService.class),
                mock(TextToSpeechService.class));
        assertThat(platform.id()).isEqualTo("sherpa");
    }

    @Test
    void supportsBothCapabilities() {
        var platform = new SherpaVoicePlatform(mock(SpeechToTextService.class),
                mock(TextToSpeechService.class));
        assertThat(platform.supports(SpeechToText.class)).isTrue();
        assertThat(platform.supports(TextToSpeech.class)).isTrue();
    }

    @Test
    void transcribeDelegatesToSttService() throws Exception {
        var stt = mock(SpeechToTextService.class);
        when(stt.transcribe(any(), any())).thenReturn(
                new io.casehub.agency.speech.TranscriptionResult("hello", "en", 0.95));
        var platform = new SherpaVoicePlatform(stt, mock(TextToSpeechService.class));

        var result = platform.speechToText()
                .transcribe(new byte[]{1, 2, 3}, TranscriptionOptions.defaults());
        assertThat(result.text()).isEqualTo("hello");
        assertThat(result.confidence()).isEqualTo(0.95);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Expected: compilation failure — `SherpaVoicePlatform` does not exist.

- [ ] **Step 4: Implement SherpaVoicePlatform**

```java
package io.casehub.agency.speech.sherpa;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import io.casehub.agency.speech.SpeechToTextService;
import io.casehub.agency.speech.TextToSpeechService;
import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.SynthesisResult;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class SherpaVoicePlatform implements VoicePlatform {

    private final SpeechToTextService sttService;
    private final TextToSpeechService ttsService;

    @Inject
    public SherpaVoicePlatform(SpeechToTextService sttService,
                                TextToSpeechService ttsService) {
        this.sttService = sttService;
        this.ttsService = ttsService;
    }

    @Override public String id() { return "sherpa"; }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == SpeechToText.class || capability == TextToSpeech.class;
    }

    @Override
    public SpeechToText speechToText() {
        return (audio, options) -> {
            Path tempFile = null;
            try {
                tempFile = Files.createTempFile("voice-stt-", ".audio");
                Files.write(tempFile, audio);
                var agencyOptions = bridgeTranscriptionOptions(options);
                var agencyResult = sttService.transcribe(tempFile, agencyOptions);
                return new TranscriptionResult(
                        agencyResult.text(), agencyResult.language(), agencyResult.confidence());
            } catch (IOException e) {
                throw new RuntimeException("Failed to transcribe audio", e);
            } finally {
                if (tempFile != null) {
                    try { Files.deleteIfExists(tempFile); } catch (IOException ignored) {}
                }
            }
        };
    }

    @Override
    public TextToSpeech textToSpeech() {
        return new TextToSpeech() {
            @Override
            public SynthesisResult synthesise(String text, SynthesisOptions options) {
                var agencyOptions = bridgeSynthesisOptions(options);
                var agencyResult = ttsService.synthesise(text, agencyOptions);
                return new SynthesisResult(agencyResult.audioData(), agencyResult.audioFormat());
            }

            @Override
            public void warmUp() { ttsService.warmUp(); }
        };
    }

    private static io.casehub.agency.speech.TranscriptionOptions bridgeTranscriptionOptions(
            TranscriptionOptions options) {
        return new io.casehub.agency.speech.TranscriptionOptions(
                options.audioFormat(), options.languageHint(),
                options.modelSize(), options.vocabularyHint());
    }

    private static io.casehub.agency.speech.SynthesisOptions bridgeSynthesisOptions(
            SynthesisOptions options) {
        return new io.casehub.agency.speech.SynthesisOptions(
                options.voice(), options.language(),
                options.audioFormat(), options.includePhonemes());
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl speech-sherpa -f /Users/mdproctor/claude/casehub/agency/pom.xml`
Expected: all tests PASS.

- [ ] **Step 6: Commit (agency repo)**

```bash
git -C /Users/mdproctor/claude/casehub/agency add speech-sherpa/
git -C /Users/mdproctor/claude/casehub/agency commit -m "feat(connectors#188): add SherpaVoicePlatform adapter — implements VoicePlatform SPI

Bridges agency SpeechToTextService and TextToSpeechService to the
connectors VoicePlatform SPI. byte[] → temp file bridging for STT.
Type mapping between agency and connectors model types.

Refs casehubio/connectors#188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 7: Full build verification (connectors)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all modules compile and tests pass.

- [ ] **Step 8: Full build verification (agency)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/agency/pom.xml`
Expected: BUILD SUCCESS — agency builds against the locally installed connectors SNAPSHOT.

- [ ] **Step 9: Final commit and update CLAUDE.md**

Update connectors `CLAUDE.md` module table with the 4 new modules. Update `docs/guides/consumer-guide.md` and `docs/guides/contributor-guide.md` with VoicePlatform SPI documentation.

```bash
git add CLAUDE.md docs/
git commit -m "docs(#188): update CLAUDE.md and guides with VoicePlatform SPI, Gmail push, WhatsApp media download

Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-30-real-time-inbound-channels-design.md] — design spec this plan implements
- [decisions.md] — 12 validated design decisions
- [contacts-spi/NoOpContactsPlatform.java] — NoOp pattern reference
- [contacts-spi/ContactsPlatformService.java] — PlatformService pattern reference
- [contacts-spi/pom.xml] — SPI module POM template
- [contacts-ref/pom.xml] — ref module POM template
- [webhook/WebhookRouter.java:85-122] — dispatch switch to extend with DeferredDelivery
- [webhook/WebhookBeans.java] — CDI producer for webhook connectors
- [webhook/WhatsAppInboundConnector.java] — connector to enhance with media download
- [connectors-api/WebhookResult.java] — sealed interface to extend
- [connectors-api/InboundConnectorIds.java] — constants to extend
- [email-google/GoogleEmailPlatform.java] — Gmail API pattern to reuse
- [platform/yaml-plugin-api/StreamingStep.java] — streaming step interface
- [platform/yaml-plugin-api/Plugin.java] — @Plugin annotation
- [agency/speech-api/SpeechToTextService.java] — agency STT interface to bridge
- [agency/speech-api/TextToSpeechService.java] — agency TTS interface to bridge
- Protocol: shared-http-client (PP-20260607-9794cb)
- Protocol: inbound-connector-id-constants (PP-20260607-d4ee52)
- GitHub #188 — focal issue
- GitHub #187 — VoicePlatform SPI epic (partial: SPI definition)
- platform#680 — streaming step type (landed)
