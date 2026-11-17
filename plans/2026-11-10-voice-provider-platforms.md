# Voice Provider VoicePlatform Implementations Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #202 — Voice provider manifest and routing — match LLM pattern
**Issue group:** #202

**Goal:** Implement `KokoroVoicePlatform` (TTS) and `WhisperVoicePlatform` (STT) as `VoicePlatform` composition roots in voice-sherpa, and update the seed catalog to wire them as defaults.

**Architecture:** Each engine gets its own `VoicePlatform` implementation — single-capability, single-responsibility. The routing layer (`RoutingVoicePlatformService`) already supports separate `tts-vendor`/`stt-vendor` defaults and handles mix-and-match. Platform classes are plain Java (no CDI) in voice-sherpa.

**Tech Stack:** Java 22 (FFM/Panama), JUnit 5, AssertJ, sherpa-onnx native bindings

## Global Constraints

- voice-sherpa is pure Java — no Jakarta CDI annotations, no Quarkus dependencies
- Java 22 compiler release (set in voice-sherpa pom.xml)
- Tests use `@EnabledIf` guards for native library availability — no test should fail without native libs
- `UnsupportedCapabilityException` from `connectors-api` for unsupported capability accessors

---

## Batch 1: VoicePlatform implementations + seed catalog

### Task 1: KokoroVoicePlatform

**Files:**
- Create: `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/KokoroVoicePlatform.java`
- Test: `voice-sherpa/src/test/java/io/casehub/connectors/voice/sherpa/KokoroVoicePlatformTest.java`

**Interfaces:**
- Consumes: `VoicePlatform` (voice-spi), `KokoroTextToSpeech` (voice-sherpa), `KokoroConfig` (voice-sherpa), `UnsupportedCapabilityException` (connectors-api), `SpeechToText` (voice-spi), `TextToSpeech` (voice-spi)
- Produces: `KokoroVoicePlatform implements VoicePlatform, AutoCloseable` — `id()` returns `"kokoro"`, `withDefaults()` factory, `close()` delegates to engine

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.connectors.voice.sherpa;

import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class KokoroVoicePlatformTest {

    @Test
    void id_returnsKokoro() {
        var config = KokoroConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new KokoroVoicePlatform(config);
        assertThat(platform.id()).isEqualTo("kokoro");
    }

    @Test
    void supports_ttsTrue() {
        var config = KokoroConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new KokoroVoicePlatform(config);
        assertThat(platform.supports(TextToSpeech.class)).isTrue();
    }

    @Test
    void supports_sttFalse() {
        var config = KokoroConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new KokoroVoicePlatform(config);
        assertThat(platform.supports(SpeechToText.class)).isFalse();
    }

    @Test
    void speechToText_throwsUnsupported() {
        var config = KokoroConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new KokoroVoicePlatform(config);
        assertThatThrownBy(platform::speechToText)
            .isInstanceOf(UnsupportedCapabilityException.class);
    }

    @Test
    @org.junit.jupiter.api.condition.EnabledIf("isSherpaAvailable")
    void textToSpeech_returnsNonNull() {
        var platform = KokoroVoicePlatform.withDefaults();
        try {
            assertThat(platform.textToSpeech()).isNotNull();
            assertThat(platform.textToSpeech()).isInstanceOf(TextToSpeech.class);
        } finally {
            platform.close();
        }
    }

    @Test
    void implementsVoicePlatform() {
        var config = KokoroConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new KokoroVoicePlatform(config);
        assertThat(platform).isInstanceOf(VoicePlatform.class);
        assertThat(platform).isInstanceOf(AutoCloseable.class);
    }

    static boolean isSherpaAvailable() {
        return SherpaLibrary.isAvailable();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-sherpa -Dtest=KokoroVoicePlatformTest -DfailIfNoTests=false`
Expected: FAIL — `KokoroVoicePlatform` class not found

- [ ] **Step 3: Implement KokoroVoicePlatform**

Use `ide_create_file` to create the class:

```java
package io.casehub.connectors.voice.sherpa;

import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;

import java.util.List;

public final class KokoroVoicePlatform implements VoicePlatform, AutoCloseable {

    private final KokoroTextToSpeech tts;

    public KokoroVoicePlatform(KokoroConfig config) {
        this.tts = new KokoroTextToSpeech(config);
    }

    KokoroVoicePlatform(KokoroTextToSpeech tts) {
        this.tts = tts;
    }

    public static KokoroVoicePlatform withDefaults() {
        Provisioner.ensureNativeLibrary();
        var modelDir = Provisioner.ensureKokoroModel("kokoro-multi-lang-v1_0");
        return new KokoroVoicePlatform(KokoroConfig.defaults(modelDir));
    }

    @Override
    public String id() {
        return "kokoro";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == TextToSpeech.class;
    }

    @Override
    public SpeechToText speechToText() {
        throw new UnsupportedCapabilityException(
            "speechToText", "SpeechToText", "kokoro", List.of("TextToSpeech"));
    }

    @Override
    public TextToSpeech textToSpeech() {
        return tts;
    }

    @Override
    public void close() {
        tts.close();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-sherpa -Dtest=KokoroVoicePlatformTest -DfailIfNoTests=false`
Expected: PASS (native-guarded test may skip if sherpa unavailable)

- [ ] **Step 5: Commit**

```bash
git add voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/KokoroVoicePlatform.java
git add voice-sherpa/src/test/java/io/casehub/connectors/voice/sherpa/KokoroVoicePlatformTest.java
git commit -m "feat(#202): add KokoroVoicePlatform — TTS-only VoicePlatform composition root"
```

### Task 2: WhisperVoicePlatform

**Files:**
- Create: `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/WhisperVoicePlatform.java`
- Test: `voice-sherpa/src/test/java/io/casehub/connectors/voice/sherpa/WhisperVoicePlatformTest.java`

**Interfaces:**
- Consumes: `VoicePlatform` (voice-spi), `SherpaOnnxSpeechToText` (voice-sherpa), `SherpaConfig` (voice-sherpa), `UnsupportedCapabilityException` (connectors-api), `SpeechToText` (voice-spi), `TextToSpeech` (voice-spi)
- Produces: `WhisperVoicePlatform implements VoicePlatform, AutoCloseable` — `id()` returns `"whisper"`, `withDefaults()` factory, `close()` delegates to engine

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.connectors.voice.sherpa;

import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class WhisperVoicePlatformTest {

    @Test
    void id_returnsWhisper() {
        var config = SherpaConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new WhisperVoicePlatform(config);
        assertThat(platform.id()).isEqualTo("whisper");
    }

    @Test
    void supports_sttTrue() {
        var config = SherpaConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new WhisperVoicePlatform(config);
        assertThat(platform.supports(SpeechToText.class)).isTrue();
    }

    @Test
    void supports_ttsFalse() {
        var config = SherpaConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new WhisperVoicePlatform(config);
        assertThat(platform.supports(TextToSpeech.class)).isFalse();
    }

    @Test
    void textToSpeech_throwsUnsupported() {
        var config = SherpaConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new WhisperVoicePlatform(config);
        assertThatThrownBy(platform::textToSpeech)
            .isInstanceOf(UnsupportedCapabilityException.class);
    }

    @Test
    @org.junit.jupiter.api.condition.EnabledIf("isSherpaAvailable")
    void speechToText_returnsNonNull() {
        var platform = WhisperVoicePlatform.withDefaults();
        try {
            assertThat(platform.speechToText()).isNotNull();
            assertThat(platform.speechToText()).isInstanceOf(SpeechToText.class);
        } finally {
            platform.close();
        }
    }

    @Test
    void implementsVoicePlatform() {
        var config = SherpaConfig.defaults(java.nio.file.Path.of("/tmp/fake-model"));
        var platform = new WhisperVoicePlatform(config);
        assertThat(platform).isInstanceOf(VoicePlatform.class);
        assertThat(platform).isInstanceOf(AutoCloseable.class);
    }

    static boolean isSherpaAvailable() {
        return SherpaLibrary.isAvailable();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-sherpa -Dtest=WhisperVoicePlatformTest -DfailIfNoTests=false`
Expected: FAIL — `WhisperVoicePlatform` class not found

- [ ] **Step 3: Implement WhisperVoicePlatform**

Use `ide_create_file` to create the class:

```java
package io.casehub.connectors.voice.sherpa;

import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;

import java.util.List;

public final class WhisperVoicePlatform implements VoicePlatform, AutoCloseable {

    private final SherpaOnnxSpeechToText stt;

    public WhisperVoicePlatform(SherpaConfig config) {
        this.stt = new SherpaOnnxSpeechToText(config);
    }

    WhisperVoicePlatform(SherpaOnnxSpeechToText stt) {
        this.stt = stt;
    }

    public static WhisperVoicePlatform withDefaults() {
        Provisioner.ensureNativeLibrary();
        if (Provisioner.isAutoDownloadEnabled()) {
            Provisioner.ensureModel("sherpa-onnx-whisper-tiny");
        }
        return new WhisperVoicePlatform(SherpaConfig.defaults());
    }

    @Override
    public String id() {
        return "whisper";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == SpeechToText.class;
    }

    @Override
    public SpeechToText speechToText() {
        return stt;
    }

    @Override
    public TextToSpeech textToSpeech() {
        throw new UnsupportedCapabilityException(
            "textToSpeech", "TextToSpeech", "whisper", List.of("SpeechToText"));
    }

    @Override
    public void close() {
        stt.close();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-sherpa -Dtest=WhisperVoicePlatformTest -DfailIfNoTests=false`
Expected: PASS (native-guarded test may skip if sherpa unavailable)

- [ ] **Step 5: Commit**

```bash
git add voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/WhisperVoicePlatform.java
git add voice-sherpa/src/test/java/io/casehub/connectors/voice/sherpa/WhisperVoicePlatformTest.java
git commit -m "feat(#202): add WhisperVoicePlatform — STT-only VoicePlatform composition root"
```

### Task 3: Update seed catalog

**Files:**
- Modify: `voice-spi/src/main/resources/voices/seed-catalog.yaml`
- Modify: `voice-spi/src/test/java/io/casehub/connectors/voice/config/VoiceManifestLoaderTest.java` (verify updated catalog loads)

**Interfaces:**
- Consumes: `VoiceManifestLoader.loadResource()`, `VoiceManifest` record
- Produces: Updated seed catalog with kokoro/whisper providers and defaults

- [ ] **Step 1: Write failing test for updated catalog**

Add a test to the existing `VoiceManifestLoaderTest` that verifies the seed catalog declares kokoro and whisper providers:

```java
@Test
void seedCatalog_declaresKokoroAndWhisperProviders() {
    var loader = new VoiceManifestLoader();
    var manifest = loader.loadResource("voices/seed-catalog.yaml");
    var vendors = manifest.providers().stream()
        .map(VoiceProviderDeclaration::vendor)
        .toList();
    assertThat(vendors).contains("kokoro", "whisper");
}

@Test
void seedCatalog_defaultsToKokoroTtsAndWhisperStt() {
    var loader = new VoiceManifestLoader();
    var manifest = loader.loadResource("voices/seed-catalog.yaml");
    assertThat(manifest.defaults()).isNotNull();
    assertThat(manifest.defaults().ttsVendor()).isEqualTo("kokoro");
    assertThat(manifest.defaults().sttVendor()).isEqualTo("whisper");
}

@Test
void seedCatalog_heartVoiceIsKokoroVendor() {
    var loader = new VoiceManifestLoader();
    var manifest = loader.loadResource("voices/seed-catalog.yaml");
    assertThat(manifest.voices()).containsKey("af_heart");
    assertThat(manifest.voices().get("af_heart").vendor()).isEqualTo("kokoro");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-spi -Dtest=VoiceManifestLoaderTest -DfailIfNoTests=false`
Expected: FAIL — seed catalog currently has vendor `ref` and no kokoro/whisper providers

- [ ] **Step 3: Update seed catalog**

Replace `voice-spi/src/main/resources/voices/seed-catalog.yaml`:

```yaml
providers:
  - vendor: ref
  - vendor: kokoro
  - vendor: whisper
voices:
  af_heart:
    vendor: kokoro
    voice-id: af_heart
    name: Heart
    gender: female
    accent: american
    quality: 4.5
    languages: [en-US]
    capability: tts
  bm_daniel:
    vendor: kokoro
    voice-id: bm_daniel
    name: Daniel
    gender: male
    accent: british
    quality: 4.3
    languages: [en-GB]
    capability: tts
defaults:
  tts-vendor: kokoro
  stt-vendor: whisper
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl voice-spi -DfailIfNoTests=false`
Expected: PASS — all existing tests should still pass with updated catalog

- [ ] **Step 5: Commit**

```bash
git add voice-spi/src/main/resources/voices/seed-catalog.yaml
git add voice-spi/src/test/java/io/casehub/connectors/voice/config/VoiceManifestLoaderTest.java
git commit -m "feat(#202): update seed catalog with kokoro/whisper providers and defaults"
```

## References

- `specs/issue-202-voice-provider-manifest-routing/2026-11-10-voice-provider-platforms-design.md` — design spec
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatform.java:7-16` — SPI interface
- `voice-ref/src/main/java/io/casehub/connectors/voice/ref/ReferenceVoicePlatform.java` — reference pattern
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/NoOpVoicePlatform.java` — UnsupportedCapabilityException usage
- `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/KokoroTextToSpeech.java` — Kokoro engine
- `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/SherpaOnnxSpeechToText.java` — Whisper STT engine
- `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/KokoroConfig.java` — config record
- `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/SherpaConfig.java` — config record
- `connectors-api/src/main/java/io/casehub/connectors/UnsupportedCapabilityException.java` — capability exception
- GitHub #202 — Voice provider manifest and routing
