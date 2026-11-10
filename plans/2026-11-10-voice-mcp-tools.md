# Voice MCP Tools Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** connectors#187 — epic: VoicePlatform SPI — bidirectional voice for agent automation
**Issue group:** connectors#187

**Goal:** Add MCP tools (`voice_speak`, `voice_transcribe`, `voice_list_voices`, `voice_status`) to the connectors graphql module, exposing the VoicePlatform SPI to LLM agents via the platform's MCP dispatcher.

**Architecture:** A single `ConnectorVoiceApi` class in the graphql module, annotated with `@McpDomain("connectors/voice")`, delegates to `RoutingVoicePlatformService` for smart voice routing and `VoicePlatformService` for direct platform access. A `VoiceDomainReportProvider` provides runtime status via the `DomainReport` pattern. Audio data is base64-encoded in DTOs, following the MCP AudioContent spec pattern.

**Tech Stack:** Java 21, Quarkus, casehub-platform mcp-api annotations, casehub-connectors voice-spi

## Global Constraints

- Use platform MCP annotations (`@McpDomain`, `@PlatformQuery`, `@PlatformMutation`, `@DefaultValue`, `@RestPath`, `@QueryParam` from `io.casehub.platform.api.mcp`) — NOT JAX-RS annotations
- Use `McpCapabilityException` for structured error responses (NOT `UnsupportedCapabilityException`)
- Audio data as base64 strings in DTOs (standard MCP pattern)
- Tests: plain JUnit 5 with inline stubs (no CDI, no Mockito) — following existing graphql test patterns
- All paths relative to `/Users/mdproctor/claude/casehub/slots/223/connectors`

---

## Batch 1: Voice MCP Tools

### Task 1: ConnectorVoiceApi — pom, DTO, API class, tests

**Files:**
- Modify: `graphql/pom.xml` — add voice-spi dependency + domain filter
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/dto/VoiceSynthesisResult.java`
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorVoiceApi.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorVoiceApiTest.java`

**Interfaces:**
- Consumes: `VoicePlatformService` (voice-spi), `RoutingVoicePlatformService` (voice-spi), `VoiceRegistry` (voice-spi), `SpeechToText` (voice-spi), `TextToSpeech` (voice-spi), `VoiceDescriptor` (voice-spi), `TranscriptionResult` (voice-spi), `SynthesisResult` (voice-spi), `TranscriptionOptions` (voice-spi), `SynthesisOptions` (voice-spi), `VoiceQuery` (voice-spi), `VoiceCapability` (voice-spi), `ResolvedVoice` (voice-spi)
- Produces: `ConnectorVoiceApi` class with methods: `speak(String text, String voice, String language, String audioFormat, boolean includePhonemes) → VoiceSynthesisResult`, `transcribe(String audioBase64, String audioFormat, String platform, String languageHint) → TranscriptionResult`, `listVoices(String gender, String accent, String language, String style, String capability, String vendor) → List<VoiceDescriptor>`. `VoiceSynthesisResult` record: `(String audioBase64, String audioFormat, String voiceId, String voiceName, String vendor, List<PhonemeTiming> phonemes)`.

- [ ] **Step 1: Add voice-spi dependency to graphql pom.xml**

Add to the `<dependencies>` section after the health-spi dependency:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-connectors-voice-spi</artifactId>
  <version>${project.version}</version>
</dependency>
```

Add `connectors/voice` to the `-AdomainFilter` compiler arg. The existing value is:
```
connectors,connectors/messaging,connectors/chat,connectors/calendar,connectors/bank-feed,connectors/email,connectors/contacts,connectors/project,connectors/commerce,connectors/travel,connectors/location,connectors/health
```
Append `,connectors/voice` to the end.

- [ ] **Step 2: Create VoiceSynthesisResult DTO**

Create `graphql/src/main/java/io/casehub/connectors/graphql/dto/VoiceSynthesisResult.java`:

```java
package io.casehub.connectors.graphql.dto;

import io.casehub.connectors.voice.model.PhonemeTiming;
import java.util.List;

public record VoiceSynthesisResult(
        String audioBase64,
        String audioFormat,
        String voiceId,
        String voiceName,
        String vendor,
        List<PhonemeTiming> phonemes) {

    public VoiceSynthesisResult {
        phonemes = phonemes != null ? List.copyOf(phonemes) : List.of();
    }
}
```

- [ ] **Step 3: Write the test class with stub infrastructure**

Create `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorVoiceApiTest.java` with the stub voice platform and initial test:

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.voice.model.PhonemeTiming;
import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.SynthesisResult;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.model.VoiceCapability;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import io.casehub.connectors.voice.spi.InMemoryVoiceRegistry;
import io.casehub.connectors.voice.spi.ResolvedVoice;
import io.casehub.connectors.voice.spi.RoutingVoicePlatformService;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;
import io.casehub.connectors.voice.spi.VoicePlatformService;
import org.junit.jupiter.api.Test;

import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ConnectorVoiceApiTest {

    private static final byte[] SAMPLE_AUDIO = "fake-wav-data".getBytes(StandardCharsets.UTF_8);
    private static final String SAMPLE_BASE64 = Base64.getEncoder().encodeToString(SAMPLE_AUDIO);

    @Test
    void speak_withDefaultVoice_returnsSynthesisResult() {
        var api = apiWithDefaults();

        var result = api.speak("Hello world", null, null, "wav", false);

        assertThat(result.audioBase64()).isNotEmpty();
        assertThat(result.audioFormat()).isEqualTo("wav");
        assertThat(result.vendor()).isEqualTo("stub-voice");
        assertThat(result.phonemes()).isEmpty();
    }

    @Test
    void speak_withNamedVoice_resolvesAndSynthesises() {
        var api = apiWithDefaults();

        var result = api.speak("Hello", "narrator", null, "wav", false);

        assertThat(result.audioBase64()).isNotEmpty();
        assertThat(result.voiceId()).isEqualTo("narrator");
    }

    @Test
    void speak_withLanguageQuery_resolvesViaQuery() {
        var api = apiWithDefaults();

        var result = api.speak("Bonjour", null, "fr", "wav", false);

        assertThat(result.audioBase64()).isNotEmpty();
        assertThat(result.vendor()).isEqualTo("stub-voice");
    }

    @Test
    void speak_withPhonemes_includesPhonemeTiming() {
        var api = apiWithDefaults();

        var result = api.speak("Hello", null, null, "wav", true);

        assertThat(result.phonemes()).isNotEmpty();
        assertThat(result.phonemes().getFirst().phoneme()).isEqualTo("h");
    }

    @Test
    void transcribe_withDefaultPlatform_returnsTranscription() {
        var api = apiWithDefaults();

        var result = api.transcribe(SAMPLE_BASE64, "wav", null, null);

        assertThat(result.text()).isEqualTo("hello world");
        assertThat(result.confidence()).isGreaterThan(0.0);
    }

    @Test
    void transcribe_withNamedPlatform_delegatesToPlatform() {
        var api = apiWithDefaults();

        var result = api.transcribe(SAMPLE_BASE64, "wav", "stub-voice", null);

        assertThat(result.text()).isEqualTo("hello world");
    }

    @Test
    void listVoices_noFilters_returnsAll() {
        var api = apiWithDefaults();

        var result = api.listVoices(null, null, null, null, null, null);

        assertThat(result).hasSize(2);
    }

    @Test
    void listVoices_filteredByGender_returnsMatching() {
        var api = apiWithDefaults();

        var result = api.listVoices("female", null, null, null, null, null);

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().gender()).isEqualTo("female");
    }

    @Test
    void speak_unsupportedCapability_throwsMcpCapabilityException() {
        var sttOnly = new StubVoicePlatform("stt-only", false, true);
        var api = apiWith(sttOnly);

        org.junit.jupiter.api.Assertions.assertThrows(
                io.casehub.platform.api.mcp.McpCapabilityException.class,
                () -> api.speak("Hello", null, null, "wav", false));
    }

    @Test
    void transcribe_unsupportedCapability_throwsMcpCapabilityException() {
        var ttsOnly = new StubVoicePlatform("tts-only", true, false);
        var api = apiWith(ttsOnly);

        org.junit.jupiter.api.Assertions.assertThrows(
                io.casehub.platform.api.mcp.McpCapabilityException.class,
                () -> api.transcribe(SAMPLE_BASE64, "wav", "tts-only", null));
    }

    // --- test infrastructure ---

    private ConnectorVoiceApi apiWithDefaults() {
        return apiWith(new StubVoicePlatform("stub-voice", true, true));
    }

    private ConnectorVoiceApi apiWith(VoicePlatform platform) {
        var voices = List.of(
                new VoiceDescriptor("narrator", platform.id(), "narrator",
                        "Narrator", "male", "british", 0.9,
                        List.of("en"), "narrative", 24000, 16, 1,
                        VoiceCapability.TTS, Map.of()),
                new VoiceDescriptor("alice", platform.id(), "alice",
                        "Alice", "female", "american", 0.85,
                        List.of("en", "fr"), "conversational", 24000, 16, 1,
                        VoiceCapability.TTS, Map.of()));

        var registry = new InMemoryVoiceRegistry(voices);
        var platformService = new VoicePlatformService(List.of(platform));
        var routingService = new RoutingVoicePlatformService(
                platformService, registry, Map.of(), null);

        var api = new ConnectorVoiceApi();
        api.routingService = routingService;
        api.platformService = platformService;
        api.voiceRegistry = registry;
        return api;
    }

    static class StubVoicePlatform implements VoicePlatform {
        private final String id;
        private final boolean supportsTts;
        private final boolean supportsStt;

        StubVoicePlatform(String id, boolean supportsTts, boolean supportsStt) {
            this.id = id;
            this.supportsTts = supportsTts;
            this.supportsStt = supportsStt;
        }

        @Override public String id() { return id; }

        @Override public boolean supports(Class<?> capability) {
            if (capability == TextToSpeech.class) return supportsTts;
            if (capability == SpeechToText.class) return supportsStt;
            return false;
        }

        @Override public SpeechToText speechToText() {
            return (audio, options) -> new TranscriptionResult("hello world", "en", 0.95);
        }

        @Override public TextToSpeech textToSpeech() {
            return (text, options) -> {
                var phonemes = options.includePhonemes()
                        ? List.of(new PhonemeTiming("h", 0, 50),
                                  new PhonemeTiming("E", 50, 100))
                        : List.<PhonemeTiming>of();
                return new SynthesisResult(SAMPLE_AUDIO, options.audioFormat() != null
                        ? options.audioFormat() : "wav", phonemes);
            };
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorVoiceApiTest -f /Users/mdproctor/claude/casehub/slots/223/connectors/pom.xml`
Expected: compilation failure — `ConnectorVoiceApi` class doesn't exist yet.

- [ ] **Step 5: Create ConnectorVoiceApi implementation**

Create `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorVoiceApi.java`:

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.graphql.dto.VoiceSynthesisResult;
import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.model.VoiceCapability;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import io.casehub.connectors.voice.spi.ResolvedVoice;
import io.casehub.connectors.voice.spi.RoutingVoicePlatformService;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatform;
import io.casehub.connectors.voice.spi.VoicePlatformService;
import io.casehub.connectors.voice.spi.VoiceRegistry;
import io.casehub.platform.api.mcp.DefaultValue;
import io.casehub.platform.api.mcp.McpCapabilityException;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.QueryParam;
import io.casehub.platform.api.mcp.RestPath;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.Base64;
import java.util.List;

@McpDomain(value = "connectors/voice", app = "connectors",
    basePath = "/api/connectors/voice",
    summary = "Voice connector — speech-to-text, text-to-speech, voice discovery")
@ApplicationScoped
public class ConnectorVoiceApi {

    @Inject RoutingVoicePlatformService routingService;
    @Inject VoicePlatformService platformService;
    @Inject VoiceRegistry voiceRegistry;

    @PlatformMutation("Synthesise text to speech audio")
    @RestPath("/speak")
    public VoiceSynthesisResult speak(
            @QueryParam("text") String text,
            @QueryParam("voice") String voice,
            @QueryParam("language") String language,
            @QueryParam("audioFormat") @DefaultValue("wav") String audioFormat,
            @QueryParam("includePhonemes") @DefaultValue("false") boolean includePhonemes) {

        ResolvedVoice resolved = resolveVoice(voice, language);
        VoicePlatform platform = resolved.platform();
        requireCapability(platform, TextToSpeech.class, "speak");

        var options = new SynthesisOptions(
                resolved.descriptor() != null ? resolved.descriptor().voiceId() : null,
                language, audioFormat, includePhonemes);
        var result = platform.textToSpeech().synthesise(text, options);

        String base64 = Base64.getEncoder().encodeToString(result.audioData());
        VoiceDescriptor desc = resolved.descriptor();
        return new VoiceSynthesisResult(
                base64, result.audioFormat(),
                desc != null ? desc.id() : null,
                desc != null ? desc.name() : null,
                platform.id(),
                result.phonemes());
    }

    @PlatformMutation("Transcribe audio to text")
    @RestPath("/transcribe")
    public TranscriptionResult transcribe(
            @QueryParam("audioBase64") String audioBase64,
            @QueryParam("audioFormat") String audioFormat,
            @QueryParam("platform") String platform,
            @QueryParam("languageHint") String languageHint) {

        VoicePlatform p = platform != null
                ? platformService.platform(platform)
                : routingService.defaultStt();
        requireCapability(p, SpeechToText.class, "transcribe");

        byte[] audio = Base64.getDecoder().decode(audioBase64);
        var options = new TranscriptionOptions(audioFormat, languageHint, null);
        return p.speechToText().transcribe(audio, options);
    }

    @PlatformQuery("List available voices with optional filters")
    @RestPath("/voices")
    public List<VoiceDescriptor> listVoices(
            @QueryParam("gender") String gender,
            @QueryParam("accent") String accent,
            @QueryParam("language") String language,
            @QueryParam("style") String style,
            @QueryParam("capability") String capability,
            @QueryParam("vendor") String vendor) {

        VoiceCapability cap = capability != null
                ? VoiceCapability.valueOf(capability.toUpperCase())
                : null;
        var query = new VoiceQuery(vendor, gender, accent, null,
                language, style, cap, null);
        return voiceRegistry.query(query);
    }

    private ResolvedVoice resolveVoice(String voice, String language) {
        if (voice != null) {
            return routingService.resolve(voice);
        }
        if (language != null) {
            return routingService.resolveQuery(
                    VoiceQuery.builder().language(language).build());
        }
        return new ResolvedVoice(routingService.defaultTts(), null);
    }

    private static void requireCapability(VoicePlatform platform,
                                           Class<?> capability, String operation) {
        if (!platform.supports(capability)) {
            var supported = new ArrayList<String>();
            if (platform.supports(TextToSpeech.class)) supported.add("TextToSpeech");
            if (platform.supports(SpeechToText.class)) supported.add("SpeechToText");
            throw McpCapabilityException.unsupported(
                    capability.getSimpleName(), platform.id(), supported,
                    operation + " requires " + capability.getSimpleName()
                    + " but platform '" + platform.id() + "' does not support it");
        }
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorVoiceApiTest -f /Users/mdproctor/claude/casehub/slots/223/connectors/pom.xml`
Expected: all 10 tests PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/223/connectors add graphql/pom.xml graphql/src/main/java/io/casehub/connectors/graphql/ConnectorVoiceApi.java graphql/src/main/java/io/casehub/connectors/graphql/dto/VoiceSynthesisResult.java graphql/src/test/java/io/casehub/connectors/graphql/ConnectorVoiceApiTest.java
git -C /Users/mdproctor/claude/casehub/slots/223/connectors commit -m "feat(#187): add voice MCP tools — speak, transcribe, list voices

ConnectorVoiceApi with @McpDomain(\"connectors/voice\") exposing
VoicePlatform SPI as MCP tools. Smart voice routing via
RoutingVoicePlatformService, base64 audio encoding, structured
McpCapabilityException errors.

Refs casehubio/connectors#187"
```

### Task 2: VoiceDomainReportProvider + ModelEnricher update

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/VoiceDomainReportProvider.java`
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorsModelEnricher.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/VoiceDomainReportProviderTest.java`

**Interfaces:**
- Consumes: `VoicePlatformService` (voice-spi), `RoutingVoicePlatformService` (voice-spi), `VoiceRegistry` (voice-spi), `DomainReportProvider` (mcp-api), `DomainReport` (mcp-api), `ProviderReport` (mcp-api), `ComponentStatus` (mcp-api)
- Produces: `VoiceDomainReportProvider implements DomainReportProvider` — `report(String domain) → DomainReport` with per-platform provider reports showing capabilities, voice count, and configured defaults.

- [ ] **Step 1: Write VoiceDomainReportProviderTest**

Create `graphql/src/test/java/io/casehub/connectors/graphql/VoiceDomainReportProviderTest.java`:

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.voice.config.VoiceManifestDefaults;
import io.casehub.connectors.voice.spi.InMemoryVoiceRegistry;
import io.casehub.connectors.voice.spi.RoutingVoicePlatformService;
import io.casehub.connectors.voice.spi.VoicePlatformService;
import io.casehub.platform.api.mcp.ComponentStatus;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class VoiceDomainReportProviderTest {

    @Test
    void report_includesAllPlatforms() {
        var platform = new ConnectorVoiceApiTest.StubVoicePlatform("kokoro", true, false);
        var platformService = new VoicePlatformService(List.of(platform));
        var registry = new InMemoryVoiceRegistry(List.of());
        var defaults = new VoiceManifestDefaults("kokoro", "whisper");
        var routingService = new RoutingVoicePlatformService(
                platformService, registry, Map.of(), defaults);

        var provider = new VoiceDomainReportProvider();
        provider.platformService = platformService;
        provider.routingService = routingService;
        provider.voiceRegistry = registry;

        var report = provider.report("connectors/voice");

        assertThat(report.domain()).isEqualTo("connectors/voice");
        assertThat(report.status()).isEqualTo(ComponentStatus.AVAILABLE);
        assertThat(report.providers()).containsKey("kokoro");
        assertThat(report.providers().get("kokoro").capabilities())
                .contains("TextToSpeech");
        assertThat(report.metadata()).containsEntry("defaultTtsVendor", "kokoro");
        assertThat(report.metadata()).containsEntry("defaultSttVendor", "whisper");
    }

    @Test
    void report_noPlatforms_statusUnavailable() {
        var platformService = new VoicePlatformService(List.of());
        var registry = new InMemoryVoiceRegistry(List.of());
        var routingService = new RoutingVoicePlatformService(
                platformService, registry, Map.of(), null);

        var provider = new VoiceDomainReportProvider();
        provider.platformService = platformService;
        provider.routingService = routingService;
        provider.voiceRegistry = registry;

        var report = provider.report("connectors/voice");

        assertThat(report.status()).isEqualTo(ComponentStatus.UNAVAILABLE);
        assertThat(report.providers()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=VoiceDomainReportProviderTest -f /Users/mdproctor/claude/casehub/slots/223/connectors/pom.xml`
Expected: compilation failure — `VoiceDomainReportProvider` doesn't exist.

- [ ] **Step 3: Create VoiceDomainReportProvider**

Create `graphql/src/main/java/io/casehub/connectors/graphql/VoiceDomainReportProvider.java`:

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.voice.spi.RoutingVoicePlatformService;
import io.casehub.connectors.voice.spi.SpeechToText;
import io.casehub.connectors.voice.spi.TextToSpeech;
import io.casehub.connectors.voice.spi.VoicePlatformService;
import io.casehub.connectors.voice.spi.VoiceRegistry;
import io.casehub.platform.api.mcp.ComponentStatus;
import io.casehub.platform.api.mcp.DomainReport;
import io.casehub.platform.api.mcp.DomainReportProvider;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.ProviderReport;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@McpDomain(value = "connectors/voice", app = "connectors",
    summary = "Voice connector — speech-to-text, text-to-speech, voice discovery")
@ApplicationScoped
public class VoiceDomainReportProvider implements DomainReportProvider {

    @Inject VoicePlatformService platformService;
    @Inject RoutingVoicePlatformService routingService;
    @Inject VoiceRegistry voiceRegistry;

    @Override
    public DomainReport report(String domain) {
        var providers = new LinkedHashMap<String, ProviderReport>();

        for (String id : platformService.ids()) {
            var platform = platformService.platform(id);
            var capabilities = new ArrayList<String>();
            if (platform.supports(TextToSpeech.class)) capabilities.add("TextToSpeech");
            if (platform.supports(SpeechToText.class)) capabilities.add("SpeechToText");
            providers.put(id, new ProviderReport(id, ComponentStatus.AVAILABLE,
                    capabilities, Map.of()));
        }

        ComponentStatus status = providers.isEmpty()
                ? ComponentStatus.UNAVAILABLE : ComponentStatus.AVAILABLE;

        Map<String, Object> metadata = new LinkedHashMap<>();
        metadata.put("voiceCount", voiceRegistry.all().size());
        try {
            metadata.put("defaultTtsVendor", routingService.defaultTts().id());
        } catch (IllegalStateException e) {
            metadata.put("defaultTtsVendor", null);
        }
        try {
            metadata.put("defaultSttVendor", routingService.defaultStt().id());
        } catch (IllegalStateException e) {
            metadata.put("defaultSttVendor", null);
        }

        return new DomainReport(domain, status, providers, metadata);
    }
}
```

- [ ] **Step 4: Update ConnectorsModelEnricher to include voice platform count**

Add to `ConnectorsModelEnricher.java`:
- Import: `import io.casehub.connectors.voice.spi.VoicePlatformService;`
- Field: `@Inject VoicePlatformService voicePlatformService;`
- In `state()`: add `"voicePlatforms", voicePlatformService.ids().size()` to the map

Since `Map.of()` has a max arity of 10 entries and we're at 7, this is safe.

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -f /Users/mdproctor/claude/casehub/slots/223/connectors/pom.xml`
Expected: all tests PASS (ConnectorVoiceApiTest + VoiceDomainReportProviderTest + existing tests).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/223/connectors add graphql/src/main/java/io/casehub/connectors/graphql/VoiceDomainReportProvider.java graphql/src/main/java/io/casehub/connectors/graphql/ConnectorsModelEnricher.java graphql/src/test/java/io/casehub/connectors/graphql/VoiceDomainReportProviderTest.java
git -C /Users/mdproctor/claude/casehub/slots/223/connectors commit -m "feat(#187): add voice domain report + model enricher

VoiceDomainReportProvider for runtime voice status via MCP report
tool. ConnectorsModelEnricher updated with voice platform count.

Refs casehubio/connectors#187"
```

## References

- [2026-11-10-voice-mcp-tools-design.md] — design spec this plan implements
- [connectors/graphql/ConnectorCalendarApi.java] — API class pattern
- [connectors/graphql/ConnectorHealthApi.java] — capability checking pattern
- [connectors/graphql/ConnectorCalendarApiTest.java] — test pattern (inline stubs, field injection)
- [connectors/voice-spi] — VoicePlatform, SpeechToText, TextToSpeech, VoiceRegistry, RoutingVoicePlatformService
- [platform/mcp-api] — McpDomain, McpCapabilityException, DefaultValue, DomainReportProvider
- [platform/mcp/DynamicToolRegistrar.java] — domain report auto-registration
- [connectors#187] — parent epic issue
