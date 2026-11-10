# Voice Config Manifest Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #203 — Voice config manifest — provider routing and voice registry following agent-config pattern
**Issue group:** #188, #203

**Goal:** Add priority-chain YAML config loading, a voice registry, and a routing voice platform service to voice-spi — mirroring the agent-config pattern.

**Architecture:** All new types live in `voice-spi`. `VoiceManifestLoader` reads `voice-config.yaml` from a 6-level priority chain and merges by key. `VoiceManifestProcessor` populates a `MutableVoiceRegistry`. `RoutingVoicePlatformService` resolves voice references (name, alias, criteria) to a `VoicePlatform` + `VoiceDescriptor`, mirroring `RoutingAgentProvider`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson (YAML via `jackson-dataformat-yaml`), JUnit 5, AssertJ

## Global Constraints

- Java source level 21 (on Java 26 JVM)
- All new production code in `voice-spi/src/main/java/io/casehub/connectors/voice/`
- All new test code in `voice-spi/src/test/java/io/casehub/connectors/voice/`
- Jackson YAML for config parsing (add `jackson-dataformat-yaml` to voice-spi pom.xml)
- Package: `io.casehub.connectors.voice.config` for config loading types, `.spi` for registry/routing, `.model` for descriptors
- Use `ide_create_file` for new Java files, `ide_edit_member`/`ide_replace_member` for modifications
- Commit after each task

---

## Batch 1: Model Types and Registry

### Task 1: VoiceDescriptor, VoiceCapability, VoiceQuery

**Files:**
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceDescriptor.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceCapability.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceQuery.java`
- Test: `voice-spi/src/test/java/io/casehub/connectors/voice/model/VoiceQueryTest.java`

**Interfaces:**
- Produces: `VoiceDescriptor(String id, String vendor, String voiceId, String name, String gender, String accent, double quality, List<String> languages, String style, Integer sampleRate, Integer bitDepth, Integer channels, VoiceCapability capability, Map<String, String> properties)`
- Produces: `VoiceCapability { TTS, STT, BOTH }`
- Produces: `VoiceQuery` with `all()`, `Builder` pattern, matching fields (vendor, gender, accent, minQuality, language, style, capability, preferVendor)

- [ ] **Step 1: Write VoiceCapability enum**

```java
package io.casehub.connectors.voice.model;

public enum VoiceCapability { TTS, STT, BOTH }
```

Use `ide_create_file` to create `voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceCapability.java`.

- [ ] **Step 2: Write VoiceDescriptor record**

```java
package io.casehub.connectors.voice.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public record VoiceDescriptor(
    String id,
    String vendor,
    @JsonProperty("voice-id") String voiceId,
    String name,
    String gender,
    String accent,
    double quality,
    List<String> languages,
    String style,
    @JsonProperty("sample-rate") Integer sampleRate,
    @JsonProperty("bit-depth") Integer bitDepth,
    Integer channels,
    VoiceCapability capability,
    Map<String, String> properties
) {
    public VoiceDescriptor {
        Objects.requireNonNull(id, "id");
        Objects.requireNonNull(vendor, "vendor");
        Objects.requireNonNull(voiceId, "voiceId");
        languages = languages != null ? List.copyOf(languages) : List.of();
        properties = properties != null ? Map.copyOf(properties) : Map.of();
        if (capability == null) capability = VoiceCapability.BOTH;
    }
}
```

Use `ide_create_file`.

- [ ] **Step 3: Write VoiceQuery record with builder**

```java
package io.casehub.connectors.voice.model;

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
    public static VoiceQuery all() {
        return new VoiceQuery(null, null, null, null, null, null, null, null);
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String vendor;
        private String gender;
        private String accent;
        private Double minQuality;
        private String language;
        private String style;
        private VoiceCapability capability;
        private String preferVendor;

        public Builder vendor(String v) { this.vendor = v; return this; }
        public Builder gender(String v) { this.gender = v; return this; }
        public Builder accent(String v) { this.accent = v; return this; }
        public Builder minQuality(Double v) { this.minQuality = v; return this; }
        public Builder language(String v) { this.language = v; return this; }
        public Builder style(String v) { this.style = v; return this; }
        public Builder capability(VoiceCapability v) { this.capability = v; return this; }
        public Builder preferVendor(String v) { this.preferVendor = v; return this; }

        public VoiceQuery build() {
            return new VoiceQuery(vendor, gender, accent, minQuality,
                language, style, capability, preferVendor);
        }
    }
}
```

- [ ] **Step 4: Write VoiceQuery filter test**

```java
package io.casehub.connectors.voice.model;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class VoiceQueryTest {

    static final VoiceDescriptor HEART = new VoiceDescriptor(
        "af_heart", "kokoro", "0", "Heart", "female", "american",
        4.5, List.of("en-US"), null, null, null, null, VoiceCapability.TTS, Map.of());

    static final VoiceDescriptor DANIEL = new VoiceDescriptor(
        "bm_daniel", "kokoro", "4", "Daniel", "male", "british",
        4.3, List.of("en-GB"), "newscast", null, null, null, VoiceCapability.TTS, Map.of());

    static final VoiceDescriptor WHISPER = new VoiceDescriptor(
        "whisper-large", "whisper", "large-v3", "Whisper Large",
        null, null, 0.0, List.of("en", "es", "fr"), null, null, null, null,
        VoiceCapability.STT, Map.of());

    @Test
    void allMatchesEverything() {
        var query = VoiceQuery.all();
        var voices = List.of(HEART, DANIEL, WHISPER);
        assertThat(filter(voices, query)).hasSize(3);
    }

    @Test
    void filterByGender() {
        var query = VoiceQuery.builder().gender("female").build();
        assertThat(filter(List.of(HEART, DANIEL, WHISPER), query))
            .containsExactly(HEART);
    }

    @Test
    void filterByCapability() {
        var query = VoiceQuery.builder().capability(VoiceCapability.STT).build();
        assertThat(filter(List.of(HEART, DANIEL, WHISPER), query))
            .containsExactly(WHISPER);
    }

    @Test
    void filterByMinQuality() {
        var query = VoiceQuery.builder().minQuality(4.4).build();
        assertThat(filter(List.of(HEART, DANIEL, WHISPER), query))
            .containsExactly(HEART);
    }

    @Test
    void filterByLanguage() {
        var query = VoiceQuery.builder().language("en-GB").build();
        assertThat(filter(List.of(HEART, DANIEL, WHISPER), query))
            .containsExactly(DANIEL);
    }

    // Helper — mirrors the filtering logic that InMemoryVoiceRegistry will use
    static List<VoiceDescriptor> filter(List<VoiceDescriptor> voices, VoiceQuery q) {
        return voices.stream()
            .filter(v -> q.vendor() == null || q.vendor().equals(v.vendor()))
            .filter(v -> q.gender() == null || q.gender().equals(v.gender()))
            .filter(v -> q.accent() == null || q.accent().equals(v.accent()))
            .filter(v -> q.minQuality() == null || v.quality() >= q.minQuality())
            .filter(v -> q.language() == null || v.languages().contains(q.language()))
            .filter(v -> q.style() == null || q.style().equals(v.style()))
            .filter(v -> q.capability() == null || v.capability() == q.capability()
                || v.capability() == VoiceCapability.BOTH)
            .toList();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl voice-spi test -Dtest=VoiceQueryTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceCapability.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceDescriptor.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/model/VoiceQuery.java
git add voice-spi/src/test/java/io/casehub/connectors/voice/model/VoiceQueryTest.java
git commit -m "feat(#203): VoiceDescriptor, VoiceCapability, VoiceQuery model types"
```

### Task 2: VoiceRegistry, MutableVoiceRegistry, InMemoryVoiceRegistry

**Files:**
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoiceRegistry.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/MutableVoiceRegistry.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/InMemoryVoiceRegistry.java`
- Test: `voice-spi/src/test/java/io/casehub/connectors/voice/spi/InMemoryVoiceRegistryTest.java`

**Interfaces:**
- Consumes: `VoiceDescriptor`, `VoiceQuery`, `VoiceCapability` from Task 1
- Produces: `VoiceRegistry { resolveById(String), query(VoiceQuery), all() }`
- Produces: `MutableVoiceRegistry extends VoiceRegistry { replaceSource(String, int, List<VoiceDescriptor>) → CatalogDelta }`
- Produces: `InMemoryVoiceRegistry implements MutableVoiceRegistry`

- [ ] **Step 1: Write VoiceRegistry interface**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import java.util.List;
import java.util.Optional;

public interface VoiceRegistry {
    Optional<VoiceDescriptor> resolveById(String voiceId);
    List<VoiceDescriptor> query(VoiceQuery query);
    List<VoiceDescriptor> all();

    static VoiceRegistry empty() {
        return new VoiceRegistry() {
            @Override public Optional<VoiceDescriptor> resolveById(String id) { return Optional.empty(); }
            @Override public List<VoiceDescriptor> query(VoiceQuery q) { return List.of(); }
            @Override public List<VoiceDescriptor> all() { return List.of(); }
        };
    }
}
```

- [ ] **Step 2: Write MutableVoiceRegistry interface**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.VoiceDescriptor;
import java.util.List;
import java.util.Set;

public interface MutableVoiceRegistry extends VoiceRegistry {

    record CatalogDelta(Set<String> addedIds, Set<String> removedIds, Set<String> updatedIds) {
        public boolean hasChanges() {
            return !addedIds.isEmpty() || !removedIds.isEmpty() || !updatedIds.isEmpty();
        }
    }

    CatalogDelta replaceSource(String sourceId, int priority, List<VoiceDescriptor> voices);
}
```

- [ ] **Step 3: Write InMemoryVoiceRegistry**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.VoiceCapability;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import java.util.HashSet;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryVoiceRegistry implements MutableVoiceRegistry {

    private final ConcurrentHashMap<String, List<VoiceDescriptor>> sources = new ConcurrentHashMap<>();
    private volatile ConcurrentHashMap<String, VoiceDescriptor> view = new ConcurrentHashMap<>();

    private void rebuildView() {
        var merged = new ConcurrentHashMap<String, VoiceDescriptor>();
        sources.values().forEach(list -> list.forEach(v -> merged.put(v.id(), v)));
        this.view = merged;
    }

    @Override
    public Optional<VoiceDescriptor> resolveById(String voiceId) {
        return Optional.ofNullable(view.get(voiceId));
    }

    @Override
    public List<VoiceDescriptor> query(VoiceQuery query) {
        return view.values().stream()
            .filter(v -> query.vendor() == null || query.vendor().equals(v.vendor()))
            .filter(v -> query.gender() == null || query.gender().equals(v.gender()))
            .filter(v -> query.accent() == null || query.accent().equals(v.accent()))
            .filter(v -> query.minQuality() == null || v.quality() >= query.minQuality())
            .filter(v -> query.language() == null || v.languages().contains(query.language()))
            .filter(v -> query.style() == null || query.style().equals(v.style()))
            .filter(v -> query.capability() == null || v.capability() == query.capability()
                || v.capability() == VoiceCapability.BOTH)
            .toList();
    }

    @Override
    public List<VoiceDescriptor> all() {
        return List.copyOf(view.values());
    }

    @Override
    public CatalogDelta replaceSource(String sourceId, int priority, List<VoiceDescriptor> voices) {
        var oldIds = new HashSet<>(view.keySet());
        sources.put(sourceId, List.copyOf(voices));
        rebuildView();
        var newIds = new HashSet<>(view.keySet());
        var added = new HashSet<>(newIds);
        added.removeAll(oldIds);
        var removed = new HashSet<>(oldIds);
        removed.removeAll(newIds);
        return new CatalogDelta(added, removed, java.util.Set.of());
    }
}
```

- [ ] **Step 4: Write InMemoryVoiceRegistryTest**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.VoiceCapability;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class InMemoryVoiceRegistryTest {

    static final VoiceDescriptor HEART = new VoiceDescriptor(
        "af_heart", "kokoro", "0", "Heart", "female", "american",
        4.5, List.of("en-US"), null, null, null, null, VoiceCapability.TTS, Map.of());

    static final VoiceDescriptor DANIEL = new VoiceDescriptor(
        "bm_daniel", "kokoro", "4", "Daniel", "male", "british",
        4.3, List.of("en-GB"), null, null, null, null, VoiceCapability.TTS, Map.of());

    @Test
    void emptyRegistryResolvesNothing() {
        var reg = new InMemoryVoiceRegistry();
        assertThat(reg.resolveById("af_heart")).isEmpty();
        assertThat(reg.all()).isEmpty();
    }

    @Test
    void replaceSourcePopulatesRegistry() {
        var reg = new InMemoryVoiceRegistry();
        var delta = reg.replaceSource("manifest", 0, List.of(HEART, DANIEL));
        assertThat(delta.addedIds()).containsExactlyInAnyOrder("af_heart", "bm_daniel");
        assertThat(reg.resolveById("af_heart")).contains(HEART);
        assertThat(reg.all()).hasSize(2);
    }

    @Test
    void replaceSourceReplacesExisting() {
        var reg = new InMemoryVoiceRegistry();
        reg.replaceSource("s1", 0, List.of(HEART));
        var updated = new VoiceDescriptor(
            "af_heart", "openai", "nova", "Nova", "female", "american",
            4.8, List.of("en-US"), null, null, null, null, VoiceCapability.TTS, Map.of());
        reg.replaceSource("s1", 0, List.of(updated));
        assertThat(reg.resolveById("af_heart").get().vendor()).isEqualTo("openai");
    }

    @Test
    void queryFilters() {
        var reg = new InMemoryVoiceRegistry();
        reg.replaceSource("m", 0, List.of(HEART, DANIEL));
        assertThat(reg.query(VoiceQuery.builder().gender("female").build()))
            .containsExactly(HEART);
        assertThat(reg.query(VoiceQuery.builder().accent("british").build()))
            .containsExactly(DANIEL);
        assertThat(reg.query(VoiceQuery.all())).hasSize(2);
    }

    @Test
    void deltaTracksRemovals() {
        var reg = new InMemoryVoiceRegistry();
        reg.replaceSource("s1", 0, List.of(HEART, DANIEL));
        var delta = reg.replaceSource("s1", 0, List.of(HEART));
        assertThat(delta.removedIds()).containsExactly("bm_daniel");
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl voice-spi test -Dtest=InMemoryVoiceRegistryTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoiceRegistry.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/spi/MutableVoiceRegistry.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/spi/InMemoryVoiceRegistry.java
git add voice-spi/src/test/java/io/casehub/connectors/voice/spi/InMemoryVoiceRegistryTest.java
git commit -m "feat(#203): VoiceRegistry, MutableVoiceRegistry, InMemoryVoiceRegistry"
```

## Batch 2: Manifest Loading

### Task 3: VoiceManifest types and VoiceManifestLoader

**Files:**
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifest.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceProviderDeclaration.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceAliasDeclaration.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifestDefaults.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifestLoader.java`
- Modify: `voice-spi/pom.xml` — add `jackson-dataformat-yaml` dependency
- Test: `voice-spi/src/test/java/io/casehub/connectors/voice/config/VoiceManifestLoaderTest.java`

**Interfaces:**
- Consumes: `VoiceDescriptor`, `VoiceCapability` from Task 1
- Produces: `VoiceManifest(List<VoiceProviderDeclaration>, Map<String, VoiceDescriptor>, Map<String, VoiceAliasDeclaration>, VoiceManifestDefaults)`
- Produces: `VoiceManifestLoader { load(Path projectDir, String profile) → VoiceManifest, loadResource(String classpath) → VoiceManifest }`

- [ ] **Step 1: Add jackson-dataformat-yaml dependency to voice-spi/pom.xml**

Add to `<dependencies>` section:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```

Use Edit tool (pom.xml is not a Java file).

- [ ] **Step 2: Write VoiceProviderDeclaration**

```java
package io.casehub.connectors.voice.config;

import com.fasterxml.jackson.annotation.JsonAnySetter;
import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.LinkedHashMap;
import java.util.Map;

public record VoiceProviderDeclaration(
    String vendor,
    String credential,
    @JsonProperty("model-dir") String modelDir,
    Map<String, String> properties
) {
    public VoiceProviderDeclaration {
        properties = properties != null ? Map.copyOf(properties) : Map.of();
    }

    @JsonAnySetter
    private void setProperty(String key, String value) {
        // Jackson doesn't support @JsonAnySetter on records — use a custom deserializer
        // or build properties from model-dir + extras in a factory method
    }
}
```

Note: Since Java records don't support `@JsonAnySetter`, use a simpler approach — typed fields (`vendor`, `credential`, `model-dir`) plus Jackson's `@JsonIgnoreProperties(ignoreUnknown = true)`. The `properties` map can be populated by a custom deserializer or by the processor copying known fields into the map. For simplicity, add `@JsonProperty` fields for known vendor config and let the processor build the properties map from the declaration.

Revised implementation:

```java
package io.casehub.connectors.voice.config;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;

@JsonIgnoreProperties(ignoreUnknown = true)
public record VoiceProviderDeclaration(
    String vendor,
    String credential,
    @JsonProperty("model-dir") String modelDir
) {}
```

- [ ] **Step 3: Write VoiceAliasDeclaration**

```java
package io.casehub.connectors.voice.config;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.casehub.connectors.voice.model.VoiceCapability;

public record VoiceAliasDeclaration(
    String voice,
    @JsonProperty("min-quality") Double minQuality,
    @JsonProperty("prefer-vendor") String preferVendor,
    String gender,
    String accent,
    String language,
    String style,
    VoiceCapability capability
) {}
```

- [ ] **Step 4: Write VoiceManifestDefaults**

```java
package io.casehub.connectors.voice.config;

import com.fasterxml.jackson.annotation.JsonProperty;

public record VoiceManifestDefaults(
    @JsonProperty("tts-vendor") String ttsVendor,
    @JsonProperty("stt-vendor") String sttVendor
) {}
```

- [ ] **Step 5: Write VoiceManifest**

```java
package io.casehub.connectors.voice.config;

import io.casehub.connectors.voice.model.VoiceDescriptor;
import java.util.List;
import java.util.Map;

public record VoiceManifest(
    List<VoiceProviderDeclaration> providers,
    Map<String, VoiceDescriptor> voices,
    Map<String, VoiceAliasDeclaration> aliases,
    VoiceManifestDefaults defaults
) {
    public VoiceManifest {
        providers = providers != null ? List.copyOf(providers) : List.of();
        voices = voices != null ? Map.copyOf(voices) : Map.of();
        aliases = aliases != null ? Map.copyOf(aliases) : Map.of();
    }
}
```

Note: The YAML `voices` section is a map keyed by voice name. Jackson deserializes this into `Map<String, VoiceDescriptor>`. But `VoiceDescriptor.id` must be set from the map key. Handle this in `VoiceManifestLoader` after deserialization — iterate the map and create new `VoiceDescriptor` instances with the `id` field set from the key.

- [ ] **Step 6: Write VoiceManifestLoader**

```java
package io.casehub.connectors.voice.config;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import org.jboss.logging.Logger;
import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class VoiceManifestLoader {

    private static final Logger LOG = Logger.getLogger(VoiceManifestLoader.class);
    private final ObjectMapper mapper = new ObjectMapper(new YAMLFactory())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    public VoiceManifest load(Path projectDir, String profile) {
        var entries = new ArrayList<PrioritizedManifest>();
        addClasspathResource(entries, "voices/seed-catalog.yaml", 0);
        addFileIfExists(entries, Path.of("/etc/casehub/voice-config.yaml"), 10);
        var home = Path.of(System.getProperty("user.home"));
        addFileIfExists(entries, home.resolve(".casehub/voice-config.yaml"), 20);
        if (profile != null) {
            addFileIfExists(entries, home.resolve(".casehub/voice-config-" + profile + ".yaml"), 25);
        }
        if (projectDir != null) {
            addFileIfExists(entries, projectDir.resolve("voice-config.yaml"), 30);
            if (profile != null) {
                addFileIfExists(entries, projectDir.resolve("voice-config-" + profile + ".yaml"), 35);
            }
        }
        return merge(entries);
    }

    public VoiceManifest loadResource(String classpath) {
        try (var is = getClass().getClassLoader().getResourceAsStream(classpath)) {
            if (is == null) return emptyManifest();
            return injectIds(mapper.readValue(is, VoiceManifest.class));
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    public VoiceManifest loadFile(Path path) {
        try (var is = Files.newInputStream(path)) {
            return injectIds(mapper.readValue(is, VoiceManifest.class));
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    VoiceManifest merge(List<PrioritizedManifest> entries) {
        entries.sort(Comparator.comparingInt(PrioritizedManifest::priority));
        var providers = new LinkedHashMap<String, VoiceProviderDeclaration>();
        var voices = new LinkedHashMap<String, VoiceDescriptor>();
        var aliases = new LinkedHashMap<String, VoiceAliasDeclaration>();
        VoiceManifestDefaults defaults = null;
        for (var entry : entries) {
            var m = entry.manifest;
            for (var p : m.providers()) providers.put(p.vendor(), p);
            voices.putAll(m.voices());
            aliases.putAll(m.aliases());
            if (m.defaults() != null) defaults = m.defaults();
        }
        return new VoiceManifest(new ArrayList<>(providers.values()), voices, aliases, defaults);
    }

    private VoiceManifest injectIds(VoiceManifest manifest) {
        if (manifest.voices().isEmpty()) return manifest;
        var withIds = new LinkedHashMap<String, VoiceDescriptor>();
        for (var e : manifest.voices().entrySet()) {
            var v = e.getValue();
            withIds.put(e.getKey(), new VoiceDescriptor(
                e.getKey(), v.vendor(), v.voiceId(), v.name(), v.gender(), v.accent(),
                v.quality(), v.languages(), v.style(), v.sampleRate(), v.bitDepth(),
                v.channels(), v.capability(), v.properties()));
        }
        return new VoiceManifest(manifest.providers(), withIds, manifest.aliases(), manifest.defaults());
    }

    private void addClasspathResource(List<PrioritizedManifest> entries, String resource, int priority) {
        try (var is = getClass().getClassLoader().getResourceAsStream(resource)) {
            if (is != null) {
                var manifest = injectIds(mapper.readValue(is, VoiceManifest.class));
                entries.add(new PrioritizedManifest(manifest, priority));
                LOG.debugf("Loaded voice config from classpath: %s (priority %d)", resource, priority);
            }
        } catch (IOException e) {
            LOG.warnf("Failed to load voice config from classpath %s: %s", resource, e.getMessage());
        }
    }

    private void addFileIfExists(List<PrioritizedManifest> entries, Path path, int priority) {
        if (Files.isRegularFile(path)) {
            entries.add(new PrioritizedManifest(loadFile(path), priority));
            LOG.debugf("Loaded voice config from file: %s (priority %d)", path, priority);
        }
    }

    private static VoiceManifest emptyManifest() {
        return new VoiceManifest(List.of(), Map.of(), Map.of(), null);
    }

    record PrioritizedManifest(VoiceManifest manifest, int priority) {}
}
```

- [ ] **Step 7: Write VoiceManifestLoaderTest**

```java
package io.casehub.connectors.voice.config;

import io.casehub.connectors.voice.model.VoiceCapability;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import static org.assertj.core.api.Assertions.assertThat;

class VoiceManifestLoaderTest {

    @Test
    void loadFromClasspath() {
        var loader = new VoiceManifestLoader();
        var manifest = loader.loadResource("voices/seed-catalog.yaml");
        assertThat(manifest.voices()).isNotEmpty();
        assertThat(manifest.voices().get("af_heart").vendor()).isEqualTo("ref");
    }

    @Test
    void mergeHigherPriorityOverrides(@TempDir Path tempDir) throws IOException {
        var loader = new VoiceManifestLoader();

        var low = """
            voices:
              af_heart:
                vendor: kokoro
                voice-id: "0"
                name: Heart
                quality: 4.0
                capability: tts
            defaults:
              tts-vendor: kokoro
              stt-vendor: whisper
            """;
        var high = """
            voices:
              af_heart:
                vendor: openai
                voice-id: nova
                name: Nova
                quality: 4.8
                capability: tts
            """;

        Files.writeString(tempDir.resolve("low.yaml"), low);
        Files.writeString(tempDir.resolve("high.yaml"), high);

        var lowManifest = loader.loadFile(tempDir.resolve("low.yaml"));
        var highManifest = loader.loadFile(tempDir.resolve("high.yaml"));

        var merged = loader.merge(java.util.List.of(
            new VoiceManifestLoader.PrioritizedManifest(lowManifest, 10),
            new VoiceManifestLoader.PrioritizedManifest(highManifest, 30)));

        assertThat(merged.voices().get("af_heart").vendor()).isEqualTo("openai");
        assertThat(merged.voices().get("af_heart").voiceId()).isEqualTo("nova");
        assertThat(merged.defaults().ttsVendor()).isEqualTo("kokoro");
    }

    @Test
    void injectsIdFromMapKey(@TempDir Path tempDir) throws IOException {
        var yaml = """
            voices:
              my_voice:
                vendor: kokoro
                voice-id: "7"
                name: Custom
                quality: 3.5
                capability: tts
            """;
        Files.writeString(tempDir.resolve("voice-config.yaml"), yaml);
        var loader = new VoiceManifestLoader();
        var manifest = loader.loadFile(tempDir.resolve("voice-config.yaml"));
        assertThat(manifest.voices().get("my_voice").id()).isEqualTo("my_voice");
    }

    @Test
    void parsesAliases(@TempDir Path tempDir) throws IOException {
        var yaml = """
            aliases:
              alice:
                voice: af_heart
              high-quality:
                min-quality: 4.0
                prefer-vendor: kokoro
                capability: tts
            """;
        Files.writeString(tempDir.resolve("voice-config.yaml"), yaml);
        var loader = new VoiceManifestLoader();
        var manifest = loader.loadFile(tempDir.resolve("voice-config.yaml"));
        assertThat(manifest.aliases().get("alice").voice()).isEqualTo("af_heart");
        assertThat(manifest.aliases().get("high-quality").minQuality()).isEqualTo(4.0);
        assertThat(manifest.aliases().get("high-quality").capability()).isEqualTo(VoiceCapability.TTS);
    }

    @Test
    void parsesProviders(@TempDir Path tempDir) throws IOException {
        var yaml = """
            providers:
              - vendor: kokoro
                model-dir: /models/kokoro
              - vendor: whisper
                model-dir: /models/whisper
            """;
        Files.writeString(tempDir.resolve("voice-config.yaml"), yaml);
        var loader = new VoiceManifestLoader();
        var manifest = loader.loadFile(tempDir.resolve("voice-config.yaml"));
        assertThat(manifest.providers()).hasSize(2);
        assertThat(manifest.providers().get(0).vendor()).isEqualTo("kokoro");
        assertThat(manifest.providers().get(0).modelDir()).isEqualTo("/models/kokoro");
    }

    @Test
    void loadFromProjectDirOverridesDefaults(@TempDir Path tempDir) throws IOException {
        var yaml = """
            voices:
              af_heart:
                vendor: openai
                voice-id: nova
                name: Nova
                quality: 5.0
                capability: tts
            """;
        Files.writeString(tempDir.resolve("voice-config.yaml"), yaml);
        var loader = new VoiceManifestLoader();
        var manifest = loader.load(tempDir, null);
        var heart = manifest.voices().get("af_heart");
        assertThat(heart.vendor()).isEqualTo("openai");
    }
}
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl voice-spi test -Dtest=VoiceManifestLoaderTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add voice-spi/pom.xml
git add voice-spi/src/main/java/io/casehub/connectors/voice/config/
git add voice-spi/src/test/java/io/casehub/connectors/voice/config/
git commit -m "feat(#203): VoiceManifestLoader with priority-chain loading and merge"
```

## Batch 3: Routing and Integration

### Task 4: VoiceManifestProcessor, VoiceManifestResult, RoutingVoicePlatformService, ResolvedVoice

**Files:**
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifestProcessor.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifestResult.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/ResolvedVoice.java`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformService.java`
- Test: `voice-spi/src/test/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformServiceTest.java`

**Interfaces:**
- Consumes: `VoiceManifest`, `VoiceAliasDeclaration`, `VoiceManifestDefaults` from Task 3; `VoiceRegistry`, `MutableVoiceRegistry`, `InMemoryVoiceRegistry` from Task 2; `VoiceDescriptor`, `VoiceQuery`, `VoiceCapability` from Task 1; `VoicePlatformService`, `VoicePlatform` (existing)
- Produces: `VoiceManifestProcessor { process(VoiceManifest) → VoiceManifestResult }`
- Produces: `VoiceManifestResult(Map<String, VoiceAliasDeclaration>, VoiceManifestDefaults, Map<String, Map<String, String>>)`
- Produces: `ResolvedVoice(VoicePlatform, VoiceDescriptor)`
- Produces: `RoutingVoicePlatformService { resolve(String) → ResolvedVoice, resolveQuery(VoiceQuery) → ResolvedVoice, defaultTts() → VoicePlatform, defaultStt() → VoicePlatform }`

- [ ] **Step 1: Write VoiceManifestResult**

```java
package io.casehub.connectors.voice.config;

import java.util.Map;

public record VoiceManifestResult(
    Map<String, VoiceAliasDeclaration> aliases,
    VoiceManifestDefaults defaults,
    Map<String, Map<String, String>> providers
) {
    public VoiceManifestResult {
        aliases = aliases != null ? Map.copyOf(aliases) : Map.of();
        providers = providers != null
            ? providers.entrySet().stream().collect(
                java.util.stream.Collectors.toUnmodifiableMap(
                    Map.Entry::getKey, e -> Map.copyOf(e.getValue())))
            : Map.of();
    }

    public static VoiceManifestResult empty() {
        return new VoiceManifestResult(Map.of(), null, Map.of());
    }
}
```

- [ ] **Step 2: Write VoiceManifestProcessor**

```java
package io.casehub.connectors.voice.config;

import io.casehub.connectors.voice.spi.MutableVoiceRegistry;
import java.util.ArrayList;
import java.util.Map;
import java.util.stream.Collectors;

public class VoiceManifestProcessor {

    private final MutableVoiceRegistry voiceRegistry;

    public VoiceManifestProcessor(MutableVoiceRegistry voiceRegistry) {
        this.voiceRegistry = voiceRegistry;
    }

    public VoiceManifestResult process(VoiceManifest manifest) {
        voiceRegistry.replaceSource("manifest", 0,
            new ArrayList<>(manifest.voices().values()));

        var providerProps = manifest.providers().stream()
            .collect(Collectors.toMap(
                VoiceProviderDeclaration::vendor,
                p -> {
                    var props = new java.util.LinkedHashMap<String, String>();
                    if (p.modelDir() != null) props.put("model-dir", p.modelDir());
                    if (p.credential() != null) props.put("credential", p.credential());
                    return Map.copyOf(props);
                }));

        return new VoiceManifestResult(manifest.aliases(), manifest.defaults(), providerProps);
    }
}
```

- [ ] **Step 3: Write ResolvedVoice**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.model.VoiceDescriptor;

public record ResolvedVoice(VoicePlatform platform, VoiceDescriptor descriptor) {}
```

- [ ] **Step 4: Write RoutingVoicePlatformService**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.config.VoiceAliasDeclaration;
import io.casehub.connectors.voice.config.VoiceManifestDefaults;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import org.jboss.logging.Logger;
import java.util.Comparator;
import java.util.List;
import java.util.Map;

public class RoutingVoicePlatformService {

    private static final Logger LOG = Logger.getLogger(RoutingVoicePlatformService.class);

    private final VoicePlatformService platformService;
    private final VoiceRegistry voiceRegistry;
    private final Map<String, VoiceAliasDeclaration> aliases;
    private final VoiceManifestDefaults defaults;

    public RoutingVoicePlatformService(VoicePlatformService platformService,
                                       VoiceRegistry voiceRegistry,
                                       Map<String, VoiceAliasDeclaration> aliases,
                                       VoiceManifestDefaults defaults) {
        this.platformService = platformService;
        this.voiceRegistry = voiceRegistry;
        this.aliases = aliases != null ? Map.copyOf(aliases) : Map.of();
        this.defaults = defaults;
        LOG.infof("Voice router initialized: %d voices, %d aliases, tts=%s, stt=%s",
            voiceRegistry.all().size(), this.aliases.size(),
            defaults != null ? defaults.ttsVendor() : "(none)",
            defaults != null ? defaults.sttVendor() : "(none)");
    }

    public ResolvedVoice resolve(String voiceRef) {
        if (aliases.containsKey(voiceRef)) {
            var alias = aliases.get(voiceRef);
            if (alias.voice() != null) {
                return resolve(alias.voice());
            }
            return resolveQuery(toVoiceQuery(alias));
        }
        var descriptor = voiceRegistry.resolveById(voiceRef);
        if (descriptor.isPresent()) {
            var d = descriptor.get();
            var platform = platformService.platform(d.vendor());
            return new ResolvedVoice(platform, d);
        }
        if (platformService.supports(voiceRef)) {
            return new ResolvedVoice(platformService.platform(voiceRef), null);
        }
        throw new IllegalArgumentException("No voice or alias: " + voiceRef);
    }

    public ResolvedVoice resolveQuery(VoiceQuery query) {
        var candidates = voiceRegistry.query(query);
        if (candidates.isEmpty()) {
            throw new IllegalArgumentException("No voice matching query: " + query);
        }
        var selected = selectBest(candidates, query.preferVendor());
        var platform = platformService.platform(selected.vendor());
        return new ResolvedVoice(platform, selected);
    }

    public VoicePlatform defaultTts() {
        if (defaults == null || defaults.ttsVendor() == null) {
            throw new IllegalStateException("No default TTS vendor configured");
        }
        return platformService.platform(defaults.ttsVendor());
    }

    public VoicePlatform defaultStt() {
        if (defaults == null || defaults.sttVendor() == null) {
            throw new IllegalStateException("No default STT vendor configured");
        }
        return platformService.platform(defaults.sttVendor());
    }

    public VoiceRegistry voiceRegistry() {
        return voiceRegistry;
    }

    private VoiceDescriptor selectBest(List<VoiceDescriptor> candidates, String preferVendor) {
        if (preferVendor != null) {
            var preferred = candidates.stream()
                .filter(v -> v.vendor().equals(preferVendor))
                .findFirst();
            if (preferred.isPresent()) return preferred.get();
        }
        return candidates.stream()
            .max(Comparator.comparingDouble(VoiceDescriptor::quality))
            .orElse(candidates.get(0));
    }

    private static VoiceQuery toVoiceQuery(VoiceAliasDeclaration alias) {
        return VoiceQuery.builder()
            .minQuality(alias.minQuality())
            .preferVendor(alias.preferVendor())
            .gender(alias.gender())
            .accent(alias.accent())
            .language(alias.language())
            .style(alias.style())
            .capability(alias.capability())
            .build();
    }
}
```

- [ ] **Step 5: Write RoutingVoicePlatformServiceTest**

```java
package io.casehub.connectors.voice.spi;

import io.casehub.connectors.voice.config.VoiceAliasDeclaration;
import io.casehub.connectors.voice.config.VoiceManifestDefaults;
import io.casehub.connectors.voice.model.VoiceCapability;
import io.casehub.connectors.voice.model.VoiceDescriptor;
import io.casehub.connectors.voice.model.VoiceQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RoutingVoicePlatformServiceTest {

    static final VoiceDescriptor HEART = new VoiceDescriptor(
        "af_heart", "ref", "af_heart", "Heart", "female", "american",
        4.5, List.of("en-US"), null, null, null, null, VoiceCapability.TTS, Map.of());

    static final VoiceDescriptor DANIEL = new VoiceDescriptor(
        "bm_daniel", "ref", "bm_daniel", "Daniel", "male", "british",
        4.3, List.of("en-GB"), null, null, null, null, VoiceCapability.TTS, Map.of());

    private RoutingVoicePlatformService routing;
    private VoicePlatform refPlatform;

    @BeforeEach
    void setUp() {
        refPlatform = new NoOpVoicePlatform();
        var platformService = new VoicePlatformService(List.of(refPlatform));
        var registry = new InMemoryVoiceRegistry();
        registry.replaceSource("test", 0, List.of(HEART, DANIEL));
        var aliases = Map.of(
            "alice", new VoiceAliasDeclaration("af_heart", null, null, null, null, null, null, null),
            "high-quality", new VoiceAliasDeclaration(null, 4.4, null, null, null, null, null, VoiceCapability.TTS)
        );
        var defaults = new VoiceManifestDefaults("ref", "ref");
        routing = new RoutingVoicePlatformService(platformService, registry, aliases, defaults);
    }

    @Test
    void resolveByVoiceName() {
        var resolved = routing.resolve("af_heart");
        assertThat(resolved.descriptor()).isEqualTo(HEART);
        assertThat(resolved.platform().id()).isEqualTo("ref");
    }

    @Test
    void resolveByDirectAlias() {
        var resolved = routing.resolve("alice");
        assertThat(resolved.descriptor()).isEqualTo(HEART);
    }

    @Test
    void resolveByCriteriaAlias() {
        var resolved = routing.resolve("high-quality");
        assertThat(resolved.descriptor()).isEqualTo(HEART);
        assertThat(resolved.descriptor().quality()).isGreaterThanOrEqualTo(4.4);
    }

    @Test
    void resolveByVendorKey() {
        var resolved = routing.resolve("ref");
        assertThat(resolved.platform().id()).isEqualTo("ref");
        assertThat(resolved.descriptor()).isNull();
    }

    @Test
    void resolveUnknownThrows() {
        assertThatThrownBy(() -> routing.resolve("nonexistent"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("No voice or alias");
    }

    @Test
    void resolveQueryByGender() {
        var query = VoiceQuery.builder().gender("male").build();
        var resolved = routing.resolveQuery(query);
        assertThat(resolved.descriptor()).isEqualTo(DANIEL);
    }

    @Test
    void resolveQueryNoMatchThrows() {
        var query = VoiceQuery.builder().vendor("openai").build();
        assertThatThrownBy(() -> routing.resolveQuery(query))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void defaultTts() {
        assertThat(routing.defaultTts().id()).isEqualTo("ref");
    }

    @Test
    void defaultStt() {
        assertThat(routing.defaultStt().id()).isEqualTo("ref");
    }

    @Test
    void selectBestPrefersHighestQuality() {
        var query = VoiceQuery.builder().capability(VoiceCapability.TTS).build();
        var resolved = routing.resolveQuery(query);
        assertThat(resolved.descriptor().quality()).isEqualTo(4.5);
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl voice-spi test -Dtest=RoutingVoicePlatformServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifestProcessor.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceManifestResult.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/spi/ResolvedVoice.java
git add voice-spi/src/main/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformService.java
git add voice-spi/src/test/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformServiceTest.java
git commit -m "feat(#203): RoutingVoicePlatformService with alias and query resolution"
```

### Task 5: Seed catalog, CDI wiring, and plugin integration

**Files:**
- Create: `voice-spi/src/main/resources/voices/seed-catalog.yaml`
- Create: `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceConfigProducer.java`
- Modify: `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/TtsSynthesisePlugin.java`
- Modify: `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/SttTranscribePlugin.java`
- Modify: `voice-pipeline-steps/pom.xml` — may need voice-spi dependency update
- Test: `voice-spi/src/test/java/io/casehub/connectors/voice/config/VoiceConfigProducerTest.java`

**Interfaces:**
- Consumes: `VoiceManifestLoader`, `VoiceManifestProcessor`, `RoutingVoicePlatformService`, `InMemoryVoiceRegistry`, `VoicePlatformService` from Tasks 1-4; `VoicePlatform` (existing CDI beans)
- Produces: `VoiceConfigProducer` CDI bean producing `RoutingVoicePlatformService`
- Modifies: `TtsSynthesisePlugin.apply()` signature — `VoicePlatform` → `RoutingVoicePlatformService`
- Modifies: `SttTranscribePlugin.apply()` signature — `VoicePlatform` → `RoutingVoicePlatformService`

- [ ] **Step 1: Write seed-catalog.yaml**

Create `voice-spi/src/main/resources/voices/seed-catalog.yaml`:

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

Use Edit tool or Write tool (YAML is not a Java file).

- [ ] **Step 2: Write VoiceConfigProducer**

```java
package io.casehub.connectors.voice.config;

import io.casehub.connectors.voice.spi.InMemoryVoiceRegistry;
import io.casehub.connectors.voice.spi.RoutingVoicePlatformService;
import io.casehub.connectors.voice.spi.VoicePlatform;
import io.casehub.connectors.voice.spi.VoicePlatformService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import java.util.Optional;

@ApplicationScoped
public class VoiceConfigProducer {

    @Inject
    Instance<VoicePlatform> platforms;

    @ConfigProperty(name = "casehub.voice.profile")
    Optional<String> profile;

    @Produces
    @ApplicationScoped
    public RoutingVoicePlatformService routingVoicePlatformService() {
        var platformService = new VoicePlatformService(platforms.stream().toList());
        var registry = new InMemoryVoiceRegistry();
        var loader = new VoiceManifestLoader();
        var manifest = loader.load(null, profile.orElse(null));
        var processor = new VoiceManifestProcessor(registry);
        var result = processor.process(manifest);
        return new RoutingVoicePlatformService(
            platformService, registry, result.aliases(), manifest.defaults());
    }
}
```

- [ ] **Step 3: Update TtsSynthesisePlugin to use RoutingVoicePlatformService**

Use `ide_replace_member` on `TtsSynthesisePlugin.apply`:

```java
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
```

Update imports: add `RoutingVoicePlatformService`, `ResolvedVoice`; remove `VoicePlatform`.

- [ ] **Step 4: Update SttTranscribePlugin to use RoutingVoicePlatformService**

Use `ide_replace_member` on `SttTranscribePlugin.apply`:

```java
@Execute
public TranscriptionResult apply(byte[] audio, RoutingVoicePlatformService routing) {
    if (model != null) {
        var resolved = routing.resolve(model);
        var options = TranscriptionOptions.defaults()
            .withLanguageHint(language)
            .withModelSize(resolved.descriptor().voiceId());
        return resolved.platform().speechToText().transcribe(audio, options);
    }
    var options = TranscriptionOptions.defaults().withLanguageHint(language);
    return routing.defaultStt().speechToText().transcribe(audio, options);
}
```

Update imports: add `RoutingVoicePlatformService`; remove `VoicePlatform`.

- [ ] **Step 5: Write VoiceConfigProducerTest**

```java
package io.casehub.connectors.voice.config;

import io.casehub.connectors.voice.model.VoiceCapability;
import io.casehub.connectors.voice.ref.ReferenceVoicePlatform;
import io.casehub.connectors.voice.spi.InMemoryVoiceRegistry;
import io.casehub.connectors.voice.spi.VoicePlatformService;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class VoiceConfigProducerTest {

    @Test
    void seedCatalogLoadsVoices() {
        var loader = new VoiceManifestLoader();
        var manifest = loader.load(null, null);
        assertThat(manifest.voices()).containsKey("af_heart");
        assertThat(manifest.voices()).containsKey("bm_daniel");
        assertThat(manifest.voices().get("af_heart").vendor()).isEqualTo("ref");
        assertThat(manifest.voices().get("af_heart").capability()).isEqualTo(VoiceCapability.TTS);
    }

    @Test
    void endToEndRoutingWithSeedCatalog() {
        var refPlatform = new ReferenceVoicePlatform();
        var platformService = new VoicePlatformService(List.of(refPlatform));
        var registry = new InMemoryVoiceRegistry();
        var loader = new VoiceManifestLoader();
        var manifest = loader.load(null, null);
        var processor = new VoiceManifestProcessor(registry);
        var result = processor.process(manifest);
        var routing = new io.casehub.connectors.voice.spi.RoutingVoicePlatformService(
            platformService, registry, result.aliases(), manifest.defaults());

        var resolved = routing.resolve("af_heart");
        assertThat(resolved.platform().id()).isEqualTo("ref");
        assertThat(resolved.descriptor().voiceId()).isEqualTo("af_heart");
        assertThat(resolved.descriptor().name()).isEqualTo("Heart");
    }
}
```

- [ ] **Step 6: Run full voice-spi and voice-pipeline-steps test suites**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl voice-spi,voice-pipeline-steps,voice-ref test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — all existing tests continue to pass, new tests pass

- [ ] **Step 7: Run playbook integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl integration-tests test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS — existing playbook tests work with RoutingVoicePlatformService via seed catalog

- [ ] **Step 8: Commit**

```bash
git add voice-spi/src/main/resources/voices/seed-catalog.yaml
git add voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceConfigProducer.java
git add voice-spi/src/test/java/io/casehub/connectors/voice/config/VoiceConfigProducerTest.java
git add voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/TtsSynthesisePlugin.java
git add voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/SttTranscribePlugin.java
git commit -m "feat(#203): VoiceConfigProducer CDI wiring, seed catalog, plugin integration"
```

## References

- [2026-11-09-voice-config-manifest-design.md] — design spec this plan implements
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatform.java` — existing SPI interface
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatformService.java` — existing engine registry
- `voice-spi/src/main/java/io/casehub/connectors/voice/model/SynthesisOptions.java` — existing TTS options
- `voice-spi/src/main/java/io/casehub/connectors/voice/model/TranscriptionOptions.java` — existing STT options
- `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/TtsSynthesisePlugin.java` — TTS plugin to update
- `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/SttTranscribePlugin.java` — STT plugin to update
- `platform/agent-config-core/src/main/java/io/casehub/platform/agent/config/ManifestLoader.java:89-111` — priority chain pattern
- `platform/agent-config-core/src/main/java/io/casehub/platform/agent/config/ManifestLoader.java:341-372` — merge pattern
- `platform/agent-router-core/src/main/java/io/casehub/platform/agent/router/RoutingAgentProvider.java` — routing pattern
- `platform/platform-api/src/main/java/io/casehub/platform/api/model/ModelRegistry.java` — registry interface pattern
- `platform/platform-api/src/main/java/io/casehub/platform/api/model/ModelQuery.java` — query pattern
- GitHub #203 — focal issue
- GitHub #188 — parent branch issue
