# Voice Config Manifest — Design Spec

**Issue:** casehubio/connectors#203
**Branch:** issue-188-real-time-inbound-channels
**Date:** 2026-11-09

## Summary

Voice processing needs the same manifest + routing pattern as LLM agent config. Voice models, providers, and voice identities are declared in `voice-config.yaml`, loaded from the same priority chain as `agent-config.yaml`, accumulated and merged. A `VoiceRegistry` catalogs `VoiceDescriptor` entries. A `RoutingVoicePlatformService` resolves voice references (by name, alias, or criteria query) to the correct `VoicePlatform` engine — mirroring `RoutingAgentProvider`'s relationship with `ModelRegistry` and `BackendInstanceRegistry`.

## Architecture

### Pattern Mapping

| Agent Pattern | Voice Equivalent | Location |
|---|---|---|
| `ModelDescriptor` | `VoiceDescriptor` | `voice-spi` |
| `ModelRegistry` | `VoiceRegistry` | `voice-spi` |
| `ModelQuery` | `VoiceQuery` | `voice-spi` |
| `MutableModelRegistry` | `MutableVoiceRegistry` | `voice-spi` |
| `ManifestLoader` | `VoiceManifestLoader` | `voice-spi` |
| `ManifestProcessor` | `VoiceManifestProcessor` | `voice-spi` |
| `Manifest` (agent-config) | `VoiceManifest` | `voice-spi` |
| `ProviderDeclaration` | `VoiceProviderDeclaration` | `voice-spi` |
| `AliasDeclaration` | `VoiceAliasDeclaration` | `voice-spi` |
| `BackendInstanceRegistry` | `VoicePlatformService` (existing) | `voice-spi` |
| `RoutingAgentProvider` | `RoutingVoicePlatformService` | `voice-spi` |

All new types live in `voice-spi` (D14). No new modules.

### Data Flow

```
voice-config.yaml (priority chain)
        │
        ▼
VoiceManifestLoader ──► VoiceManifest (merged)
        │
        ▼
VoiceManifestProcessor ──► populates MutableVoiceRegistry
                           (via replaceSource)
        │
        ▼
RoutingVoicePlatformService
   ├── VoiceRegistry (voice catalog)
   ├── VoicePlatformService (engine registry)
   ├── aliases map
   └── defaults
        │
        ▼
TtsSynthesisePlugin calls:
   routing.resolve("af_heart")
   → ResolvedVoice(kokoroVoicePlatform, descriptor)
   → voicePlatform.textToSpeech().synthesise(text, options)
```

## voice-config.yaml Schema

```yaml
providers:
  - vendor: kokoro
    model-dir: ${KOKORO_MODEL_DIR:/models/kokoro-multi-lang-v1_0}
  - vendor: whisper
    model-dir: ${WHISPER_MODEL_DIR:/models/whisper-large-v3}
  - vendor: openai
    credential: ${OPENAI_API_KEY}

voices:
  af_heart:
    vendor: kokoro
    voice-id: "0"
    name: Heart
    gender: female
    accent: american
    quality: 4.5
    languages: [en-US]
    capability: tts
  bm_daniel:
    vendor: kokoro
    voice-id: "4"
    name: Daniel
    gender: male
    accent: british
    quality: 4.3
    languages: [en-GB]
    style: newscast
    capability: tts
  whisper-large:
    vendor: whisper
    voice-id: large-v3
    name: Whisper Large v3
    capability: stt
    languages: [en, es, fr, de, ja, zh]

defaults:
  tts-vendor: kokoro
  stt-vendor: whisper

aliases:
  alice:
    voice: af_heart
  bob:
    voice: bm_daniel
  high-quality:
    min-quality: 4.0
    prefer-vendor: kokoro
    capability: tts
```

### Priority Chain (mirrors agent-config)

| Priority | Source |
|----------|--------|
| 0 | Classpath seed catalog (`voices/seed-catalog.yaml`) |
| 10 | `/etc/casehub/voice-config.yaml` |
| 20 | `~/.casehub/voice-config.yaml` |
| 25 | `~/.casehub/voice-config-{profile}.yaml` |
| 30 | `{project}/voice-config.yaml` |
| 35 | `{project}/voice-config-{profile}.yaml` |

Higher priority overrides lower, keyed by voice name (for voices), vendor (for providers), alias name (for aliases). Last-wins merge via `LinkedHashMap`, sorted by priority — identical to `ManifestLoader.merge()`.

## Types

### VoiceDescriptor

```java
public record VoiceDescriptor(
    String id,           // voice key in YAML (e.g. "af_heart")
    String vendor,       // provider vendor (e.g. "kokoro")
    String voiceId,      // vendor-specific voice identifier
    String name,         // display name (e.g. "Heart")
    String gender,       // female, male, neutral
    String accent,       // american, british, australian, etc.
    double quality,      // quality rating (0.0 - 5.0)
    List<String> languages,  // supported language codes
    String style,        // optional: newscast, cheerful, empathetic
    Integer sampleRate,  // optional: preferred sample rate (Hz)
    Integer bitDepth,    // optional: preferred bit depth
    Integer channels,    // optional: mono=1, stereo=2
    VoiceCapability capability, // TTS, STT, or BOTH
    Map<String, String> properties  // vendor-specific extras
) {}

public enum VoiceCapability { TTS, STT, BOTH }
```

### VoiceQuery

```java
public record VoiceQuery(
    String vendor,
    String gender,
    String accent,
    Double minQuality,
    String language,
    String style,
    VoiceCapability capability,
    String preferVendor
) {
    public static VoiceQuery all() { ... }
    public static Builder builder() { ... }
}
```

### VoiceRegistry / MutableVoiceRegistry

```java
public interface VoiceRegistry {
    Optional<VoiceDescriptor> resolveById(String voiceId);
    List<VoiceDescriptor> query(VoiceQuery query);
    List<VoiceDescriptor> all();
    static VoiceRegistry empty() { ... }
}

public interface MutableVoiceRegistry extends VoiceRegistry {
    record CatalogDelta(Set<String> addedIds, Set<String> removedIds, Set<String> updatedIds) {
        public boolean hasChanges() { ... }
    }
    CatalogDelta replaceSource(String sourceId, int priority, List<VoiceDescriptor> voices);
}
```

### VoiceManifest

```java
public record VoiceManifest(
    List<VoiceProviderDeclaration> providers,
    Map<String, VoiceDescriptor> voices,
    Map<String, VoiceAliasDeclaration> aliases,
    VoiceManifestDefaults defaults
) {}
```

### VoiceProviderDeclaration

```java
public record VoiceProviderDeclaration(
    String vendor,
    String credential,
    Map<String, String> properties  // model-dir, endpoint, etc.
) {}
```

### VoiceAliasDeclaration

```java
public record VoiceAliasDeclaration(
    String voice,          // direct voice reference (e.g. "af_heart")
    // or criteria-based:
    Double minQuality,
    String preferVendor,
    String gender,
    String accent,
    String language,
    String style,
    VoiceCapability capability
) {}
```

### VoiceManifestDefaults

```java
public record VoiceManifestDefaults(
    @JsonProperty("tts-vendor") String ttsVendor,
    @JsonProperty("stt-vendor") String sttVendor
) {}
```

### ResolvedVoice

```java
public record ResolvedVoice(
    VoicePlatform platform,
    VoiceDescriptor descriptor
) {}
```

## RoutingVoicePlatformService

```java
public class RoutingVoicePlatformService {

    private final VoicePlatformService platformService;  // engine registry
    private final VoiceRegistry voiceRegistry;           // voice catalog
    private final Map<String, VoiceAliasDeclaration> aliases;
    private final VoiceManifestDefaults defaults;

    // Resolve by voice name
    public ResolvedVoice resolve(String voiceRef) {
        // 1. Check aliases
        if (aliases.containsKey(voiceRef)) {
            var alias = aliases.get(voiceRef);
            if (alias.voice() != null) {
                return resolve(alias.voice());  // follow alias → voice name
            }
            return resolveQuery(toVoiceQuery(alias));  // criteria alias
        }
        // 2. Check registry by ID
        var descriptor = voiceRegistry.resolveById(voiceRef);
        if (descriptor.isPresent()) {
            var d = descriptor.get();
            var platform = platformService.platform(d.vendor());
            return new ResolvedVoice(platform, d);
        }
        // 3. Check as vendor key (default voice for vendor)
        if (platformService.supports(voiceRef)) {
            return new ResolvedVoice(platformService.platform(voiceRef), null);
        }
        throw new IllegalArgumentException("No voice or alias: " + voiceRef);
    }

    // Resolve by criteria query
    public ResolvedVoice resolveQuery(VoiceQuery query) {
        var candidates = voiceRegistry.query(query);
        if (candidates.isEmpty()) {
            throw new IllegalArgumentException("No voice matching query: " + query);
        }
        // Prefer default vendor, then highest quality
        var selected = selectBest(candidates, query.preferVendor());
        var platform = platformService.platform(selected.vendor());
        return new ResolvedVoice(platform, selected);
    }

    // Default platform for TTS/STT
    public VoicePlatform defaultTts() {
        return platformService.platform(defaults.ttsVendor());
    }

    public VoicePlatform defaultStt() {
        return platformService.platform(defaults.sttVendor());
    }
}
```

## VoiceManifestLoader

Mirrors `ManifestLoader.load(Path projectDir, String profile)`:

```java
public class VoiceManifestLoader {

    public VoiceManifest load(Path projectDir, String profile) {
        var entries = new ArrayList<PrioritizedManifest>();

        addClasspathResource(entries, "voices/seed-catalog.yaml", 0);
        addFileIfExists(entries, Path.of("/etc/casehub/voice-config.yaml"), 10);
        addFileIfExists(entries, Path.of(home(), ".casehub", "voice-config.yaml"), 20);
        if (profile != null) {
            addFileIfExists(entries, Path.of(home(), ".casehub",
                "voice-config-" + profile + ".yaml"), 25);
        }
        if (projectDir != null) {
            addFileIfExists(entries, projectDir.resolve("voice-config.yaml"), 30);
            if (profile != null) {
                addFileIfExists(entries, projectDir.resolve(
                    "voice-config-" + profile + ".yaml"), 35);
            }
        }

        return merge(entries);
    }

    VoiceManifest merge(List<PrioritizedManifest> entries) {
        entries.sort(Comparator.comparingInt(PrioritizedManifest::priority));
        var providers = new LinkedHashMap<String, VoiceProviderDeclaration>();
        var voices = new LinkedHashMap<String, VoiceDescriptor>();
        var aliases = new LinkedHashMap<String, VoiceAliasDeclaration>();
        VoiceManifestDefaults defaults = null;
        for (var entry : entries) {
            var m = entry.manifest();
            for (var p : m.providers()) providers.put(p.vendor(), p);
            voices.putAll(m.voices());
            aliases.putAll(m.aliases());
            if (m.defaults() != null) defaults = m.defaults();
        }
        return new VoiceManifest(
            new ArrayList<>(providers.values()), voices, aliases, defaults);
    }

    record PrioritizedManifest(VoiceManifest manifest, int priority) {}
}
```

## VoiceManifestProcessor

Processes merged manifest, populates registry:

```java
public class VoiceManifestProcessor {

    private final MutableVoiceRegistry voiceRegistry;

    public VoiceManifestResult process(VoiceManifest manifest) {
        // Populate registry with voice descriptors
        voiceRegistry.replaceSource("manifest", 0,
            new ArrayList<>(manifest.voices().values()));

        return new VoiceManifestResult(
            manifest.aliases(),
            manifest.defaults(),
            manifest.providers().stream()
                .collect(Collectors.toMap(
                    VoiceProviderDeclaration::vendor,
                    VoiceProviderDeclaration::properties)));
    }
}

public record VoiceManifestResult(
    Map<String, VoiceAliasDeclaration> aliases,
    VoiceManifestDefaults defaults,
    Map<String, Map<String, String>> providers
) {}
```

## Integration with TtsSynthesisePlugin

Current plugin:
```java
@Plugin(value = "tts-synthesise", streaming = true)
public record TtsSynthesisePlugin(@Optional String voice, @Optional String language) {
    @Execute
    public SynthesisResult apply(String text, VoicePlatform voicePlatform) {
        var options = new SynthesisOptions(voice, language, "wav", false);
        return voicePlatform.textToSpeech().synthesise(text, options);
    }
}
```

Updated plugin (injects `RoutingVoicePlatformService` instead of `VoicePlatform`):
```java
@Plugin(value = "tts-synthesise", streaming = true)
public record TtsSynthesisePlugin(@Optional String voice, @Optional String language) {
    @Execute
    public SynthesisResult apply(String text, RoutingVoicePlatformService routing) {
        if (voice != null) {
            var resolved = routing.resolve(voice);
            var options = new SynthesisOptions(
                resolved.descriptor().voiceId(), language, "wav", false);
            return resolved.platform().textToSpeech().synthesise(text, options);
        }
        var options = new SynthesisOptions(null, language, "wav", false);
        return routing.defaultTts().textToSpeech().synthesise(text, options);
    }
}
```

Similarly for `SttTranscribePlugin`:
```java
@Plugin(value = "stt-transcribe", streaming = true)
public record SttTranscribePlugin(@Optional String model) {
    @Execute
    public TranscriptionResult apply(byte[] audio, RoutingVoicePlatformService routing) {
        if (model != null) {
            var resolved = routing.resolve(model);
            var options = new TranscriptionOptions(null, null, resolved.descriptor().voiceId());
            return resolved.platform().speechToText().transcribe(audio, options);
        }
        var options = TranscriptionOptions.defaults();
        return routing.defaultStt().speechToText().transcribe(audio, options);
    }
}
```

## Seed Catalog

The classpath seed catalog (`voices/seed-catalog.yaml`) ships with voice-spi and provides the Kokoro voice entries that playbooks already reference:

```yaml
providers:
  - vendor: ref
voices:
  af_heart:
    vendor: ref
    voice-id: af_heart
    name: Heart
    gender: female
    accent: american
    quality: 4.5
    languages: [en-US]
    capability: tts
  bm_daniel:
    vendor: ref
    voice-id: bm_daniel
    name: Daniel
    gender: male
    accent: british
    quality: 4.3
    languages: [en-GB]
    capability: tts
defaults:
  tts-vendor: ref
  stt-vendor: ref
```

The seed catalog maps to the `ref` vendor (ReferenceVoicePlatform). Real deployments override with project-level `voice-config.yaml` that maps the same voice names to real vendor implementations (Kokoro, OpenAI, etc.).

## Credential Resolution

Provider credentials follow the same `${ENV_VAR}` expansion pattern as agent-config. The `VoiceManifestLoader` expands `${VAR}` and `${VAR:default}` in provider `credential` and `properties` values during YAML loading. Full `CredentialResolver` integration (credential store, managed credentials) is deferred until a real voice provider implementation needs it — the ref platform requires no credentials.

## CDI Wiring

```java
@ApplicationScoped
public class VoiceConfigProducer {

    @ConfigProperty(name = "casehub.voice.profile", defaultValue = "")
    String profile;

    @Inject
    List<VoicePlatform> platforms;

    @Produces @ApplicationScoped
    public RoutingVoicePlatformService routingVoicePlatformService() {
        var platformService = new VoicePlatformService(platforms);
        var registry = new InMemoryVoiceRegistry();
        var loader = new VoiceManifestLoader();
        var manifest = loader.load(projectDir(), profileOrNull());
        var processor = new VoiceManifestProcessor(registry);
        var result = processor.process(manifest);
        return new RoutingVoicePlatformService(
            platformService, registry, result.aliases(), manifest.defaults());
    }
}
```

## Testing

- `VoiceManifestLoaderTest` — priority-chain merge (higher overrides lower), profile handling, missing files
- `VoiceRegistryTest` — resolveById, query by criteria, empty registry
- `RoutingVoicePlatformServiceTest` — resolve by name, by alias (direct + criteria), by vendor key, default TTS/STT
- `VoiceManifestProcessorTest` — populates registry from manifest
- Existing playbook tests continue to pass — seed catalog maps existing voice names to ref platform

## References

- `ManifestLoader.java` lines 89-111 — priority chain pattern
- `ManifestLoader.merge()` lines 341-372 — LinkedHashMap merge
- `RoutingAgentProvider.java` — resolve by name/alias/query, dispatch to backend
- `ModelRegistry.java`, `MutableModelRegistry.java` — registry interfaces
- `ModelQuery.java` — criteria-based query with builder
- `ModelDescriptor.java` — descriptor record with properties map
- `AliasDeclaration.java` — alias → criteria or direct ref
- `VoicePlatform.java`, `VoicePlatformService.java` — existing voice infrastructure
- `TtsSynthesisePlugin.java` — current plugin (consumer of routing)
- `SynthesisOptions.java`, `TranscriptionOptions.java` — options records
- casehubio/connectors#203 — issue spec
