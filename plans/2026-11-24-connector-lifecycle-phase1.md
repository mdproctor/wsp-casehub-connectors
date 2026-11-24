# ConnectorLifecycle SPI — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #356 — Epic: ConnectorLifecycle SPI — declarative connector setup, auth, and health
**Issue group:** #356, platform#966

**Goal:** Replace bespoke `*Beans.java` CDI producers with manifest-driven synthetic bean registration for Numbrrs, Telegram, and Signal — enabling agent-driven connector discovery, configuration, and setup.

**Architecture:** Each connector module declares its requirements in a `connector-manifest.yaml`. A Quarkus build extension discovers these manifests and registers synthetic CDI beans, resolving credentials via a `ProviderCredentialFacade`. MCP tools expose the lifecycle to agents. Setup playbooks handle multi-step init flows like Signal phone registration via Numbrrs SMS.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Jackson YAML, Quarkus SyntheticBeanBuildItem, Jandex

## Global Constraints

- Java source level 21 on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Quarkus 3.32.2 — all SyntheticBeanBuildItem usage must match this version's API
- YAML keys: kebab-case, `@JsonProperty("kebab-name")` mapping, `YamlMappers.create()` or `new ObjectMapper(new YAMLFactory())` with `FAIL_ON_UNKNOWN_PROPERTIES = false`
- Platform dependency: `ProviderCredentialFacade` interface (platform#966) — stub implementation in connectors until platform delivers
- No connectors-spring changes in Phase 1
- Build: `mvn clean install` from root, or per-module via `-f <module>/pom.xml`
- Use `ide_create_file` for new Java files, `ide_edit_member`/`ide_replace_member`/`ide_insert_member` for modifications, `ide_refactor_safe_delete` for removals

---

## Batch 1: Foundation — Manifest Records, Loader, Credential Facade Stub

After this batch: `ConnectorManifest` can be parsed from YAML, the loader discovers manifests via merge cascade, and a stub credential facade resolves fields from a simple in-memory map. All tested.

### Task 1: ConnectorManifest Record and Supporting Types

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifest.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorBean.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/ConnectorManifestTest.java`

**Interfaces:**
- Consumes: Jackson YAML (`com.fasterxml.jackson.dataformat.yaml.YAMLFactory`)
- Produces: `ConnectorManifest` record with nested `ConnectorDeclaration`, `AuthDeclaration`, `FieldDeclaration`, `BeanDeclaration`, `ConstructorArg`, `HealthDeclaration`; `@ConnectorBean` qualifier annotation

- [ ] **Step 1: Write failing test — parse a minimal manifest YAML**

```java
@Test
void parses_minimal_manifest() throws Exception {
    var yaml = """
            connector:
              id: test
              platform: chat
              description: "Test connector"
              auth:
                type: api-key
                fields:
                  - name: api-key
                    description: "The API key"
              produces:
                - class: com.example.TestPlatform
                  type: platform
            """;
    var mapper = new ObjectMapper(new YAMLFactory())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    var manifest = mapper.readValue(yaml, ConnectorManifest.class);

    assertThat(manifest.connector().id()).isEqualTo("test");
    assertThat(manifest.connector().platform()).isEqualTo("chat");
    assertThat(manifest.connector().auth().type()).isEqualTo("api-key");
    assertThat(manifest.connector().auth().fields()).hasSize(1);
    assertThat(manifest.connector().auth().fields().getFirst().name()).isEqualTo("api-key");
    assertThat(manifest.connector().auth().fields().getFirst().required()).isTrue();
    assertThat(manifest.connector().auth().fields().getFirst().sensitive()).isTrue();
    assertThat(manifest.connector().produces()).hasSize(1);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=ConnectorManifestTest#parses_minimal_manifest -pl connectors-api`
Expected: FAIL — `ConnectorManifest` class does not exist

- [ ] **Step 3: Implement ConnectorManifest record**

Use `ide_create_file` for the record. Full implementation per spec — all nested records with `@JsonProperty` annotations. `FieldDeclaration` defaults: `required=true`, `sensitive=true`, `defaultValue=null`. `ConstructorArg` validates exactly-one-of `field`/`bean`.

```java
package io.casehub.connectors.lifecycle;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

public record ConnectorManifest(ConnectorDeclaration connector) {

    public record ConnectorDeclaration(
            String id, String platform, String description,
            AuthDeclaration auth, List<BeanDeclaration> produces,
            @JsonProperty("setup-playbook") String setupPlaybook,
            HealthDeclaration health) {}

    public record AuthDeclaration(String type, String family, List<FieldDeclaration> fields) {}

    public record FieldDeclaration(
            String name,
            boolean required,
            String description,
            boolean sensitive,
            @JsonProperty("default") String defaultValue) {
        public FieldDeclaration {
            if (name == null || name.isBlank()) throw new IllegalArgumentException("field name required");
        }
        public FieldDeclaration(String name, String description) { this(name, true, description, true, null); }
    }

    public record BeanDeclaration(
            @JsonProperty("class") String className, String scope, String type,
            @JsonProperty("constructor-args") List<ConstructorArg> constructorArgs) {}

    public record ConstructorArg(String field, String bean) {
        public ConstructorArg {
            if ((field == null) == (bean == null))
                throw new IllegalArgumentException("exactly one of field or bean required");
        }
    }

    public record HealthDeclaration(String endpoint, String interval) {}
}
```

- [ ] **Step 4: Implement @ConnectorBean qualifier**

```java
package io.casehub.connectors.lifecycle;

import jakarta.inject.Qualifier;
import java.lang.annotation.*;

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER})
public @interface ConnectorBean {
    String id();
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=ConnectorManifestTest#parses_minimal_manifest`
Expected: PASS

- [ ] **Step 6: Write additional tests — field defaults, constructor-arg validation, full manifest**

```java
@Test
void field_defaults_applied() throws Exception {
    var yaml = """
            connector:
              id: test
              platform: mobile
              auth:
                type: api-key
                fields:
                  - name: api-key
                    description: "key"
                  - name: optional-field
                    required: false
                    sensitive: false
                    default: "fallback"
                    description: "optional"
              produces: []
            """;
    var manifest = parse(yaml);
    var required = manifest.connector().auth().fields().get(0);
    assertThat(required.required()).isTrue();
    assertThat(required.sensitive()).isTrue();
    assertThat(required.defaultValue()).isNull();

    var optional = manifest.connector().auth().fields().get(1);
    assertThat(optional.required()).isFalse();
    assertThat(optional.sensitive()).isFalse();
    assertThat(optional.defaultValue()).isEqualTo("fallback");
}

@Test
void constructor_arg_requires_exactly_one() {
    assertThatThrownBy(() -> new ConnectorManifest.ConstructorArg(null, null))
            .isInstanceOf(IllegalArgumentException.class);
    assertThatThrownBy(() -> new ConnectorManifest.ConstructorArg("field", "bean"))
            .isInstanceOf(IllegalArgumentException.class);
    assertThatCode(() -> new ConnectorManifest.ConstructorArg("field", null)).doesNotThrowAnyException();
    assertThatCode(() -> new ConnectorManifest.ConstructorArg(null, "bean")).doesNotThrowAnyException();
}

@Test
void parses_numbrrs_manifest() throws Exception {
    var yaml = Files.readString(Path.of("../mobile-numbrrs/src/main/resources/META-INF/connector-manifest.yaml"));
    var manifest = parse(yaml);
    assertThat(manifest.connector().id()).isEqualTo("numbrrs");
    assertThat(manifest.connector().auth().fields()).hasSize(3);
    assertThat(manifest.connector().setupPlaybook()).isNull();
}
```

- [ ] **Step 7: Run all tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=ConnectorManifestTest`
Expected: PASS (all tests)

- [ ] **Step 8: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/ connectors-api/src/test/java/io/casehub/connectors/lifecycle/
git commit -m "feat(#356): ConnectorManifest record + @ConnectorBean qualifier

TDD: manifest YAML parsing, field defaults, constructor-arg validation.
Refs #356"
```

### Task 2: ConnectorManifestLoader with Merge Cascade

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifestLoader.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/ConnectorManifestLoaderTest.java`
- Create: `connectors-api/src/test/resources/META-INF/connector-manifest.yaml` (test fixture)

**Interfaces:**
- Consumes: `ConnectorManifest` record (Task 1)
- Produces: `ConnectorManifestLoader.discover()` → `List<ConnectorManifest>`, `ConnectorManifestLoader.discoverFromClasspath()` → `List<ConnectorManifest>`

- [ ] **Step 1: Write failing test — discover manifest from classpath**

```java
@Test
void discovers_manifest_from_classpath() {
    var manifests = ConnectorManifestLoader.discoverFromClasspath();
    assertThat(manifests).isNotEmpty();
    assertThat(manifests).anyMatch(m -> m.connector().id().equals("test-fixture"));
}
```

Create test fixture at `connectors-api/src/test/resources/META-INF/connector-manifest.yaml`:
```yaml
connector:
  id: test-fixture
  platform: test
  description: "Test fixture connector"
  auth:
    type: api-key
    fields:
      - name: api-key
        description: "Test API key"
  produces:
    - class: io.casehub.connectors.lifecycle.TestPlatform
      type: platform
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `ConnectorManifestLoader` class does not exist

- [ ] **Step 3: Implement ConnectorManifestLoader**

Follow `VoiceManifestLoader` pattern. Key methods:
- `discoverFromClasspath()` — scans `META-INF/connector-manifest.yaml` via `ClassLoader.getResources()`
- `discover()` — full merge cascade: classpath → /etc/casehub → ~/.casehub → project dir → profile variants
- `merge(List<ConnectorManifest>)` — merges by connector `id`, later sources override earlier ones field-by-field

```java
package io.casehub.connectors.lifecycle;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.logging.Logger;

public class ConnectorManifestLoader {

    private static final Logger LOG = Logger.getLogger(ConnectorManifestLoader.class.getName());
    private static final String RESOURCE_PATH = "META-INF/connector-manifest.yaml";
    private static final ObjectMapper MAPPER = new ObjectMapper(new YAMLFactory())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    public static List<ConnectorManifest> discoverFromClasspath() {
        var result = new ArrayList<ConnectorManifest>();
        try {
            var resources = Thread.currentThread().getContextClassLoader().getResources(RESOURCE_PATH);
            while (resources.hasMoreElements()) {
                var url = resources.nextElement();
                try (var is = url.openStream()) {
                    var manifest = MAPPER.readValue(is, ConnectorManifest.class);
                    if (manifest != null && manifest.connector() != null) result.add(manifest);
                }
            }
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
        return List.copyOf(result);
    }

    public static List<ConnectorManifest> discover() {
        return discover(null, null);
    }

    public static List<ConnectorManifest> discover(Path projectDir, String profile) {
        var all = new ArrayList<>(discoverFromClasspath());
        addFileIfExists(all, Path.of("/etc/casehub/connector-manifest.yaml"));
        var home = Path.of(System.getProperty("user.home"));
        addFileIfExists(all, home.resolve(".casehub/connector-manifest.yaml"));
        if (profile != null) addFileIfExists(all, home.resolve(".casehub/connector-manifest-" + profile + ".yaml"));
        if (projectDir != null) {
            addFileIfExists(all, projectDir.resolve("connector-manifest.yaml"));
            if (profile != null) addFileIfExists(all, projectDir.resolve("connector-manifest-" + profile + ".yaml"));
        }
        return merge(all);
    }

    private static void addFileIfExists(List<ConnectorManifest> list, Path path) {
        if (Files.isReadable(path)) {
            try (var is = Files.newInputStream(path)) {
                var manifest = MAPPER.readValue(is, ConnectorManifest.class);
                if (manifest != null) list.add(manifest);
            } catch (IOException e) {
                LOG.warning("Failed to load connector manifest from " + path + ": " + e.getMessage());
            }
        }
    }

    static List<ConnectorManifest> merge(List<ConnectorManifest> manifests) {
        var byId = new LinkedHashMap<String, ConnectorManifest>();
        for (var m : manifests) {
            byId.put(m.connector().id(), m);
        }
        return List.copyOf(byId.values());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Write additional tests — merge override, multiple manifests**

```java
@Test
void later_manifest_overrides_earlier_by_id() {
    var first = parse("connector:\n  id: x\n  platform: chat\n  auth:\n    type: api-key\n    fields: []\n  produces: []\n");
    var second = parse("connector:\n  id: x\n  platform: mobile\n  auth:\n    type: bot-token\n    fields: []\n  produces: []\n");
    var merged = ConnectorManifestLoader.merge(List.of(first, second));
    assertThat(merged).hasSize(1);
    assertThat(merged.getFirst().connector().platform()).isEqualTo("mobile");
}
```

- [ ] **Step 6: Run all tests, commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifestLoader.java \
       connectors-api/src/test/java/io/casehub/connectors/lifecycle/ConnectorManifestLoaderTest.java \
       connectors-api/src/test/resources/META-INF/connector-manifest.yaml
git commit -m "feat(#356): ConnectorManifestLoader with merge cascade

Discovers connector-manifest.yaml from classpath + file-system cascade.
Merges by connector id — later sources override earlier ones.
Refs #356"
```

### Task 3: ProviderCredentialFacade Stub

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ProviderCredentialFacade.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/InMemoryCredentialFacade.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/InMemoryCredentialFacadeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `ProviderCredentialFacade` interface (matches platform#966 spec), `InMemoryCredentialFacade` test/dev implementation

- [ ] **Step 1: Write failing test — resolve credential from in-memory store**

```java
@Test
void resolves_credential_from_store() {
    var facade = new InMemoryCredentialFacade(Map.of(
            "numbrrs", Map.of("api-key", "test-key-123", "signing-secret", "secret-456")));

    assertThat(facade.resolve("numbrrs", "api-key")).contains("test-key-123");
    assertThat(facade.resolve("numbrrs", "signing-secret")).contains("secret-456");
    assertThat(facade.resolve("numbrrs", "missing")).isEmpty();
    assertThat(facade.resolve("unknown", "api-key")).isEmpty();
}

@Test
void isConfigured_checks_required_fields() {
    var facade = new InMemoryCredentialFacade(Map.of(
            "numbrrs", Map.of("api-key", "key")));

    assertThat(facade.isConfigured("numbrrs", List.of("api-key"))).isTrue();
    assertThat(facade.isConfigured("numbrrs", List.of("api-key", "signing-secret"))).isFalse();
}

@Test
void source_returns_credential_store() {
    var facade = new InMemoryCredentialFacade(Map.of(
            "numbrrs", Map.of("api-key", "key")));
    assertThat(facade.source("numbrrs", "api-key")).isEqualTo(ProviderCredentialFacade.CredentialSource.CREDENTIAL_STORE);
    assertThat(facade.source("numbrrs", "missing")).isEqualTo(ProviderCredentialFacade.CredentialSource.NONE);
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — classes don't exist

- [ ] **Step 3: Implement ProviderCredentialFacade interface and InMemoryCredentialFacade**

```java
package io.casehub.connectors.lifecycle;

import java.util.List;
import java.util.Map;
import java.util.Optional;

public interface ProviderCredentialFacade {
    Optional<String> resolve(String providerId, String field);
    Map<String, String> resolveAll(String providerId);
    boolean isConfigured(String providerId, List<String> requiredFields);
    CredentialSource source(String providerId, String field);

    enum CredentialSource { CREDENTIAL_STORE, CONFIG, ENVIRONMENT, SERVICE_CONNECTION, NONE }
}
```

```java
package io.casehub.connectors.lifecycle;

import java.util.*;

public class InMemoryCredentialFacade implements ProviderCredentialFacade {
    private final Map<String, Map<String, String>> credentials;

    public InMemoryCredentialFacade(Map<String, Map<String, String>> credentials) {
        this.credentials = Map.copyOf(credentials);
    }

    @Override
    public Optional<String> resolve(String providerId, String field) {
        return Optional.ofNullable(credentials.getOrDefault(providerId, Map.of()).get(field));
    }

    @Override
    public Map<String, String> resolveAll(String providerId) {
        return credentials.getOrDefault(providerId, Map.of());
    }

    @Override
    public boolean isConfigured(String providerId, List<String> requiredFields) {
        var providerCreds = credentials.getOrDefault(providerId, Map.of());
        return requiredFields.stream().allMatch(providerCreds::containsKey);
    }

    @Override
    public CredentialSource source(String providerId, String field) {
        return resolve(providerId, field).isPresent() ? CredentialSource.CREDENTIAL_STORE : CredentialSource.NONE;
    }
}
```

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/ProviderCredentialFacade.java \
       connectors-api/src/main/java/io/casehub/connectors/lifecycle/InMemoryCredentialFacade.java \
       connectors-api/src/test/java/io/casehub/connectors/lifecycle/InMemoryCredentialFacadeTest.java
git commit -m "feat(#356): ProviderCredentialFacade interface + InMemory stub

Stub implementation for dev/test until platform#966 delivers the real facade.
Refs #356, Refs platform#966"
```

---

## Batch 2: Build Extension — Synthetic Bean Registration

After this batch: the Quarkus build extension discovers manifests, registers synthetic CDI beans, and populates a ConnectorRegistry. Tested with a synthetic test manifest.

### Task 4: ConnectorRegistry and Status Model

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorRegistry.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorStatus.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/ConnectorRegistryTest.java`

**Interfaces:**
- Consumes: `ConnectorManifest` (Task 1), `ProviderCredentialFacade` (Task 3)
- Produces: `ConnectorRegistry` with `register()`, `status()`, `all()`, `configured()`, `unconfigured()` methods; `ConnectorStatus` enum (`READY`, `UNCONFIGURED`, `ERROR`, `DEGRADED`)

- [ ] **Step 1: Write failing test**

```java
@Test
void registry_tracks_connector_status() {
    var registry = new ConnectorRegistry();
    var manifest = parse(NUMBRRS_YAML);
    registry.register(manifest, ConnectorStatus.READY);

    assertThat(registry.status("numbrrs")).isEqualTo(ConnectorStatus.READY);
    assertThat(registry.all()).hasSize(1);
    assertThat(registry.configured()).hasSize(1);
    assertThat(registry.unconfigured()).isEmpty();
}

@Test
void unconfigured_connectors_reported() {
    var registry = new ConnectorRegistry();
    registry.register(parse(NUMBRRS_YAML), ConnectorStatus.READY);
    registry.register(parse(SIGNAL_YAML), ConnectorStatus.UNCONFIGURED);

    assertThat(registry.configured()).hasSize(1);
    assertThat(registry.unconfigured()).hasSize(1);
}
```

- [ ] **Step 2: Implement ConnectorRegistry and ConnectorStatus**

```java
public enum ConnectorStatus { READY, UNCONFIGURED, ERROR, DEGRADED }
```

```java
public class ConnectorRegistry {
    private final Map<String, RegisteredConnector> connectors = new ConcurrentHashMap<>();

    public record RegisteredConnector(ConnectorManifest manifest, ConnectorStatus status) {}

    public void register(ConnectorManifest manifest, ConnectorStatus status) {
        connectors.put(manifest.connector().id(), new RegisteredConnector(manifest, status));
    }

    public ConnectorStatus status(String connectorId) {
        var reg = connectors.get(connectorId);
        return reg != null ? reg.status() : null;
    }

    public List<RegisteredConnector> all() { return List.copyOf(connectors.values()); }
    public List<RegisteredConnector> configured() { return all().stream().filter(r -> r.status() == ConnectorStatus.READY).toList(); }
    public List<RegisteredConnector> unconfigured() { return all().stream().filter(r -> r.status() == ConnectorStatus.UNCONFIGURED).toList(); }
}
```

- [ ] **Step 3: Run tests, commit**

```bash
git commit -m "feat(#356): ConnectorRegistry + ConnectorStatus

Tracks manifest-declared connectors with their lifecycle state.
Refs #356"
```

### Task 5: Quarkus Build Extension — Manifest Discovery + Synthetic Beans

**Files:**
- Create: `connector-lifecycle/pom.xml` (new module — Quarkus deployment extension)
- Create: `connector-lifecycle/src/main/java/io/casehub/connectors/lifecycle/deployment/ConnectorLifecycleProcessor.java`
- Modify: `pom.xml` (add `<module>connector-lifecycle</module>`)
- Test: `connector-lifecycle/src/test/java/io/casehub/connectors/lifecycle/deployment/ConnectorLifecycleProcessorTest.java`
- Create: `connector-lifecycle/src/test/resources/META-INF/connector-manifest.yaml` (test manifest with simple test bean)
- Create: `connector-lifecycle/src/test/java/io/casehub/connectors/lifecycle/deployment/TestChatPlatform.java` (simple test bean)

**Interfaces:**
- Consumes: `ConnectorManifestLoader` (Task 2), `ProviderCredentialFacade` (Task 3), `ConnectorRegistry` (Task 4)
- Produces: `ConnectorLifecycleProcessor` build extension with `@BuildStep` methods: `discoverManifests()`, `registerSyntheticBeans()`, `registerRegistry()`

- [ ] **Step 1: Create connector-lifecycle module pom.xml**

The module needs Quarkus deployment dependencies for `SyntheticBeanBuildItem`, `BeanDefiningAnnotationBuildItem`, Jandex for type introspection.

```xml
<parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
</parent>

<artifactId>casehub-connectors-lifecycle</artifactId>
<name>CaseHub Connectors — Lifecycle Build Extension</name>

<dependencies>
    <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-arc-deployment</artifactId>
    </dependency>
    <!-- Test -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-junit5-internal</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

- [ ] **Step 2: Write failing test — synthetic bean is injectable**

```java
@QuarkusTest
class ConnectorLifecycleProcessorTest {

    @Inject
    ConnectorRegistry registry;

    @Test
    void registry_is_injectable() {
        assertThat(registry).isNotNull();
        assertThat(registry.all()).isNotEmpty();
    }
}
```

- [ ] **Step 3: Implement ConnectorLifecycleProcessor**

The build extension has three `@BuildStep` methods:

1. `discoverManifests()` — produces `ConnectorManifestBuildItem` list
2. `registerSyntheticBeans()` — for each manifest bean, registers `SyntheticBeanBuildItem` with creation function that injects `ProviderCredentialFacade`, resolves fields, constructs via reflection
3. `registerRegistry()` — registers `ConnectorRegistry` as a synthetic singleton

The synthetic bean creation function:
- For `type: translator` — no-arg constructor
- For `type: client` — constructor with credential field values
- For `type: platform` / `type: inbound-connector` — constructor with mix of credential fields and bean references

Uses Jandex to introspect constructor parameter types at build time for `String` vs `Optional<String>` adaptation.

- [ ] **Step 4: Run test to verify synthetic bean injection works**

- [ ] **Step 5: Write test — credential resolution flows through to beans**

```java
@Test
void registry_reports_unconfigured_when_credentials_missing() {
    var unconfigured = registry.unconfigured();
    // test-fixture connector has no credentials configured
    assertThat(unconfigured).anyMatch(r -> r.manifest().connector().id().equals("test-fixture"));
}
```

- [ ] **Step 6: Run all tests, commit**

```bash
git add connector-lifecycle/ pom.xml
git commit -m "feat(#356): Quarkus build extension — manifest discovery + synthetic beans

ConnectorLifecycleProcessor discovers connector-manifest.yaml files,
registers synthetic CDI beans from constructor-args declarations,
populates ConnectorRegistry with lifecycle status.
Refs #356"
```

---

## Batch 3: First Mover Migrations — Numbrrs, Telegram, Signal

After this batch: all three connectors are manifest-driven. Bespoke `*Beans.java` producers are deleted. Existing tests pass as regression.

### Task 6: Migrate Numbrrs to Manifest

**Files:**
- Create: `mobile-numbrrs/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `mobile-numbrrs/src/main/java/.../NumbrrsMobilePlatform.java` — new constructor `(String apiKey, String numbers)`
- Delete: `mobile-numbrrs/src/main/java/.../NumbrrsMobileBeans.java` (use `ide_refactor_safe_delete`)
- Modify: `mobile-numbrrs/pom.xml` — add `connector-lifecycle` dependency
- Test: existing `NumbrrsMobilePlatformTest.java` and `NumbrrsInboundConnectorTest.java` as regression

- [ ] **Step 1: Create connector-manifest.yaml for Numbrrs**

Per spec — `id: numbrrs`, `platform: mobile`, `auth.type: api-key`, three fields (`api-key` required, `signing-secret` optional, `numbers` optional non-sensitive).

- [ ] **Step 2: Add new constructor to NumbrrsMobilePlatform**

Use `ide_insert_member` to add:
```java
public NumbrrsMobilePlatform(String apiKey, String numbers) {
    this(new NumbrrsClient(apiKey), parseNumbers(numbers != null ? numbers : ""));
}

private static List<String> parseNumbers(String numbers) {
    if (numbers.isBlank()) return List.of();
    return Arrays.stream(numbers.split(",")).map(String::trim).filter(s -> !s.isEmpty()).toList();
}
```

Also add degraded mode: when `apiKey` is blank, construct with no-op client.

- [ ] **Step 3: Run existing tests to verify regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f mobile-numbrrs/pom.xml test`
Expected: all 17 tests pass

- [ ] **Step 4: Delete NumbrrsMobileBeans.java**

Use `ide_refactor_safe_delete` — verify no usages.

- [ ] **Step 5: Run full module tests again**

Expected: all pass (beans now produced by lifecycle extension)

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#356): migrate Numbrrs to manifest-driven wiring

Add connector-manifest.yaml, new String-based constructor,
delete NumbrrsMobileBeans. Refs #356"
```

### Task 7: Migrate Telegram to Manifest

**Files:**
- Create: `chat-telegram/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `telegram-bot/src/main/java/.../TelegramBotClient.java` — remove `@ApplicationScoped`, `@Inject`, `@ConfigProperty`; plain constructor `(String baseUrl)`
- Modify: `chat-telegram/src/main/java/.../TelegramChatPlatform.java` — remove `@ApplicationScoped`, `@Inject`, `@ConfigProperty`; plain constructor `(TelegramBotClient client, String token, String webhookSecret)`
- Modify: `chat-telegram/src/main/java/.../TelegramInboundConnector.java` — remove CDI annotations; fix `String.equals()` → `MessageDigest.isEqual()` for webhook secret comparison
- Delete: `chat-telegram/src/main/java/.../ChatTelegramBeans.java` (use `ide_refactor_safe_delete`)
- Modify: `chat-telegram/pom.xml`, `telegram-bot/pom.xml` — dependency adjustments
- Test: existing tests as regression + new test for constant-time secret comparison

- [ ] **Step 1: Create connector-manifest.yaml**

Per spec — `id: telegram`, `platform: chat`, 4 `produces` entries with `constructor-args`, `setup-playbook`, `health` endpoint.

- [ ] **Step 2: Strip CDI annotations from TelegramBotClient**

Remove `@ApplicationScoped`, `@Inject`, `@ConfigProperty`. Constructor becomes `public TelegramBotClient(String baseUrl)`.

- [ ] **Step 3: Strip CDI annotations from TelegramChatPlatform**

Remove `@ApplicationScoped`, `@Inject`, `@ConfigProperty`. Constructor becomes `public TelegramChatPlatform(TelegramBotClient client, String token, String webhookSecret)`. Retain `@PostConstruct init()` logic — the build extension invokes it.

- [ ] **Step 4: Strip CDI annotations from TelegramInboundConnector + fix secret comparison**

Remove CDI annotations. Constructor becomes `public TelegramInboundConnector(String secretToken, TelegramChatPlatform chatPlatform)`. Fix webhook secret comparison:

```java
// Before (timing-vulnerable):
if (!secretToken.equals(headerToken)) return new WebhookResult.Unauthorized();

// After (constant-time):
if (!MessageDigest.isEqual(
        secretToken.getBytes(StandardCharsets.UTF_8),
        headerToken.getBytes(StandardCharsets.UTF_8))) {
    return new WebhookResult.Unauthorized();
}
```

- [ ] **Step 5: Delete ChatTelegramBeans.java**

- [ ] **Step 6: Run all telegram tests as regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f chat-telegram/pom.xml test`

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#356): migrate Telegram to manifest-driven wiring

Strip CDI annotations from TelegramBotClient, TelegramChatPlatform,
TelegramInboundConnector. Fix timing-vulnerable String.equals() webhook
secret comparison to MessageDigest.isEqual(). Delete ChatTelegramBeans.
Refs #356"
```

### Task 8: Migrate Signal to Manifest

**Files:**
- Create: `chat-signal/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `signal-cli/src/main/java/.../SignalClient.java` — remove CDI annotations; constructor `(String apiUrl)`
- Modify: `chat-signal/src/main/java/.../SignalChatPlatform.java` — remove CDI annotations; constructor `(SignalClient client, String apiUrl, String number)`
- Modify: `chat-signal/src/main/java/.../SignalInboundConnector.java` — remove CDI annotations; constructor `(SignalClient client, String apiUrl, String number)`
- Delete: `chat-signal/src/main/java/.../ChatSignalBeans.java`
- Modify: `chat-signal/pom.xml`, `signal-cli/pom.xml` — dependency adjustments

- [ ] **Step 1: Create connector-manifest.yaml**

Per spec — `id: signal`, `platform: chat`, 4 `produces` entries, `setup-playbook`, `health` endpoint.

- [ ] **Step 2: Strip CDI annotations from SignalClient**

Constructor becomes `public SignalClient(String apiUrl)`. Remove `@ApplicationScoped`, `@Inject`, `@ConfigProperty`. Keep no-arg constructor for backward compat.

- [ ] **Step 3: Strip CDI annotations from SignalChatPlatform**

Constructor becomes `public SignalChatPlatform(SignalClient client, String apiUrl, String number)`. Retain `@PostConstruct init()`.

- [ ] **Step 4: Strip CDI annotations from SignalInboundConnector**

Constructor becomes `public SignalInboundConnector(SignalClient client, String apiUrl, String number)`.

- [ ] **Step 5: Delete ChatSignalBeans.java**

- [ ] **Step 6: Run all signal tests as regression**

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#356): migrate Signal to manifest-driven wiring

Strip CDI annotations from SignalClient, SignalChatPlatform,
SignalInboundConnector. Delete ChatSignalBeans.
Refs #356"
```

---

## Batch 4: MCP Tools — Agent-Facing Lifecycle API

After this batch: agents can discover connectors, check status, trigger setup, and check health via MCP tools.

### Task 9: ConnectorLifecycleOperations

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorLifecycleOperations.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorLifecycleOperationsTest.java`

**Interfaces:**
- Consumes: `ConnectorRegistry` (Task 4), `ProviderCredentialFacade` (Task 3)
- Produces: `@McpDomain("connectors/lifecycle")` operations: `connector_discover`, `connector_status`, `connector_setup`, `connector_health`

- [ ] **Step 1: Write failing test — connector_discover returns registered connectors**

```java
@Test
void discover_returns_all_manifests() {
    var result = operations.connectorDiscover(null);
    assertThat(result).containsKey("connectors");
    var connectors = (List<?>) result.get("connectors");
    assertThat(connectors).isNotEmpty();
}
```

- [ ] **Step 2: Implement ConnectorLifecycleOperations**

Follow `ConnectorOperationsImpl` pattern for `@McpDomain`. Four operations returning structured JSON maps.

- [ ] **Step 3: Write tests for status, setup, health**

- [ ] **Step 4: Run tests, commit**

```bash
git commit -m "feat(#356): MCP lifecycle tools — discover, status, setup, health

@McpDomain('connectors/lifecycle') with 4 operations for agent-driven
connector management. Refs #356"
```

---

## Batch 5: Setup Playbooks — Step Types and Connector Setup Flows

After this batch: Telegram webhook registration and Signal phone registration are executable as playbooks. The `wait-for-sms` step type enables cross-connector flows.

### Task 10: New Playbook Step Types

**Files:**
- Create: `playbooks/src/main/java/io/casehub/connectors/playbook/steps/HealthCheckStep.java`
- Create: `playbooks/src/main/java/io/casehub/connectors/playbook/steps/HttpPostStep.java`
- Create: `playbooks/src/main/java/io/casehub/connectors/playbook/steps/ConnectorStatusStep.java`
- Create: `playbooks/src/main/java/io/casehub/connectors/playbook/steps/WaitForSmsStep.java`
- Test: `playbooks/src/test/java/io/casehub/connectors/playbook/steps/HealthCheckStepTest.java`
- Test: `playbooks/src/test/java/io/casehub/connectors/playbook/steps/WaitForSmsStepTest.java`

**Interfaces:**
- Consumes: `ConnectorRegistry` (Task 4), `MobilePlatformService` (existing SPI)
- Produces: Four `@Plugin` step type implementations per spec contracts

- [ ] **Step 1: Implement health-check step — HTTP GET with retry**

Per spec: 3 retries, 2s backoff, fails playbook on exhaustion.

- [ ] **Step 2: Implement http-post step — HTTP POST with JSON body**

Per spec: `Content-Type: application/json`, fails on non-2xx.

- [ ] **Step 3: Implement connector-status step — registry query**

Per spec: queries `ConnectorRegistry`, fails if status doesn't match `require`.

- [ ] **Step 4: Implement wait-for-sms step — poll with regex matching**

Per spec: polls `MobilePlatformService.platform(provider).sms().listMessages()` every 5s until pattern match or timeout.

- [ ] **Step 5: Write tests for all step types**

Use WireMock for health-check and http-post. Use `mobile-ref` for wait-for-sms.

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#356): playbook step types — health-check, http-post, connector-status, wait-for-sms

Four new @Plugin step types for connector setup playbooks.
Refs #356"
```

### Task 11: Telegram and Signal Setup Playbooks

**Files:**
- Create: `playbooks/src/main/playbooks/telegram-webhook-setup.yaml`
- Create: `playbooks/src/main/playbooks/signal-registration.yaml`
- Test: `playbooks/src/test/java/io/casehub/connectors/playbook/ConnectorSetupPlaybookTest.java`

**Interfaces:**
- Consumes: Step types from Task 10, `ProviderCredentialFacade` for template variable resolution
- Produces: Two executable setup playbooks

- [ ] **Step 1: Create telegram-webhook-setup.yaml**

Per spec — health check + setWebhook POST.

- [ ] **Step 2: Create signal-registration.yaml**

Per spec — health check → connector-status (Numbrrs READY) → register POST → wait-for-sms → verify POST.

- [ ] **Step 3: Write integration test — telegram playbook executes against WireMock**

- [ ] **Step 4: Write integration test — signal playbook executes against WireMock + mobile-ref**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#356): setup playbooks — Telegram webhook + Signal registration

Telegram: health check + setWebhook via http-post.
Signal: cross-connector flow — verify via Numbrrs SMS, register with signal-cli.
Refs #356"
```

---

## References

- [2026-11-24-connector-lifecycle-phase1-design.md] — Phase 1 design spec
- [decisions.md] — D1-D12 architectural decisions
- [NumbrrsMobileBeans.java] — `mobile-numbrrs/src/main/java/.../NumbrrsMobileBeans.java`
- [NumbrrsMobilePlatform.java] — `mobile-numbrrs/src/main/java/.../NumbrrsMobilePlatform.java`
- [NumbrrsInboundConnector.java] — `mobile-numbrrs/src/main/java/.../NumbrrsInboundConnector.java`
- [TelegramBotClient.java:34-39] — constructor with `@ConfigProperty`
- [TelegramChatPlatform.java:75-82] — constructor with `@ConfigProperty`
- [TelegramInboundConnector.java:32-38] — constructor with `@ConfigProperty`
- [SignalClient.java:31-36] — constructor with `@ConfigProperty`
- [SignalChatPlatform.java:71-79] — constructor with `@ConfigProperty`
- [SignalInboundConnector.java:39-45] — constructor with `@ConfigProperty`
- [VoiceManifestLoader.java:25-41] — merge cascade pattern
- [ConnectorOperationsImpl.java] — `@McpDomain("connectors")` pattern
- [GitHub #356] — Epic: ConnectorLifecycle SPI
- [GitHub platform#966] — ProviderCredentialFacade
