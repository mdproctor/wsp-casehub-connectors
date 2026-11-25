# Manifest Requires Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #365 — Manifest requires declaration — backward-chaining setup dependencies
**Issue group:** #365

**Goal:** Add `requires` and `provides` capability declarations to connector manifests, build a DependencyResolver that walks the graph and returns resolution options, and update Signal's playbook to use capability-based requirements.

**Architecture:** New `CapabilityDeclaration`, `RequirementDeclaration` records added to `ConnectorManifest` in `connectors-api`. A `DependencyResolver` class (pure function over manifests + registry) provides backward-chaining resolution. Signal and Numbrrs manifests updated with the new fields. Signal's registration playbook changed from hardcoded `connector-status: numbrrs` to capability-based `require-capability: sms[receive]`.

**Tech Stack:** Java 21, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Java source level 21, run on Java 26 JVM
- All new types go in `connectors-api` module under `io.casehub.connectors.lifecycle`
- Jackson YAML with `FAIL_ON_UNKNOWN_PROPERTIES=false` — new fields must not break old loaders
- Capability strings are convention-based, no enum
- Connectors never reference other connectors — requires uses capability names only

---

## Batch 1: Schema — requires/provides in manifest records and parser

### Task 1: CapabilityDeclaration and RequirementDeclaration records + manifest parsing

**Files:**
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifest.java`
- Modify: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/ConnectorManifestTest.java`

**Interfaces:**
- Produces: `ConnectorManifest.CapabilityDeclaration(String capability, List<String> sub)`, `ConnectorManifest.RequirementDeclaration(String capability, List<String> sub, String reason)`, updated `ConnectorDeclaration` with `List<RequirementDeclaration> requires` and `List<CapabilityDeclaration> provides` fields

- [ ] **Step 1: Write failing test — parses manifest with requires and provides**

```java
@Test
void parses_manifest_with_requires_and_provides() throws Exception {
    var yaml = """
            connector:
              id: test-requires
              platform: chat
              description: "Test with requires"
              auth:
                type: api-key
                fields:
                  - name: api-key
                    description: "Key"
              produces:
                - class: com.example.TestPlatform
                  type: platform
              requires:
                - capability: sms
                  sub: [receive]
                  reason: "Verification code"
              provides:
                - capability: chat-messaging
                  sub: [send, receive]
            """;
    var manifest = MAPPER.readValue(yaml, ConnectorManifest.class);

    assertThat(manifest.connector().requires()).hasSize(1);
    var req = manifest.connector().requires().getFirst();
    assertThat(req.capability()).isEqualTo("sms");
    assertThat(req.sub()).containsExactly("receive");
    assertThat(req.reason()).isEqualTo("Verification code");

    assertThat(manifest.connector().provides()).hasSize(1);
    var prov = manifest.connector().provides().getFirst();
    assertThat(prov.capability()).isEqualTo("chat-messaging");
    assertThat(prov.sub()).containsExactly("send", "receive");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=ConnectorManifestTest#parses_manifest_with_requires_and_provides`
Expected: Compilation error — `requires()` and `provides()` don't exist on `ConnectorDeclaration`

- [ ] **Step 3: Write failing test — missing requires/provides defaults to null or empty**

```java
@Test
void missing_requires_and_provides_default_gracefully() throws Exception {
    var yaml = """
            connector:
              id: no-caps
              platform: chat
              description: "No capabilities"
              auth:
                type: api-key
                fields:
                  - name: api-key
                    description: "Key"
              produces:
                - class: com.example.TestPlatform
                  type: platform
            """;
    var manifest = MAPPER.readValue(yaml, ConnectorManifest.class);

    assertThat(manifest.connector().requires()).isNullOrEmpty();
    assertThat(manifest.connector().provides()).isNullOrEmpty();
}
```

- [ ] **Step 4: Write failing test — provides with no sub-qualifiers**

```java
@Test
void parses_provides_without_sub_qualifiers() throws Exception {
    var yaml = """
            connector:
              id: simple-provider
              platform: mobile
              description: "Simple provider"
              auth:
                type: api-key
                fields:
                  - name: api-key
                    description: "Key"
              produces:
                - class: com.example.TestPlatform
                  type: platform
              provides:
                - capability: number-info
            """;
    var manifest = MAPPER.readValue(yaml, ConnectorManifest.class);

    assertThat(manifest.connector().provides()).hasSize(1);
    var prov = manifest.connector().provides().getFirst();
    assertThat(prov.capability()).isEqualTo("number-info");
    assertThat(prov.sub()).isEmpty();
}
```

- [ ] **Step 5: Implement CapabilityDeclaration, RequirementDeclaration, and updated ConnectorDeclaration**

Add to `ConnectorManifest.java` using `ide_insert_member`:

```java
public record CapabilityDeclaration(String capability, List<String> sub) {
    public CapabilityDeclaration {
        if (capability == null || capability.isBlank()) throw new IllegalArgumentException("capability required");
        if (sub == null) sub = List.of();
    }
    public CapabilityDeclaration(String capability) {
        this(capability, List.of());
    }
}

public record RequirementDeclaration(String capability, List<String> sub, String reason) {
    public RequirementDeclaration {
        if (capability == null || capability.isBlank()) throw new IllegalArgumentException("capability required");
        if (sub == null) sub = List.of();
    }
}
```

Update `ConnectorDeclaration` using `ide_replace_member` to add `requires` and `provides` fields:

```java
public record ConnectorDeclaration(
        String id,
        String platform,
        String description,
        AuthDeclaration auth,
        List<BeanDeclaration> produces,
        List<RequirementDeclaration> requires,
        List<CapabilityDeclaration> provides,
        @JsonProperty("setup-playbook") String setupPlaybook,
        HealthDeclaration health) {}
```

- [ ] **Step 6: Run all ConnectorManifestTest tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=ConnectorManifestTest`
Expected: All tests PASS (new tests + existing tests unchanged)

- [ ] **Step 7: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifest.java
git add connectors-api/src/test/java/io/casehub/connectors/lifecycle/ConnectorManifestTest.java
git commit -m "feat(#365): add requires/provides capability declarations to ConnectorManifest"
```

---

## Batch 2: DependencyResolver — matching and resolution

### Task 2: DependencyResolver with capability matching and resolution options

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/DependencyResolver.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/UnmetRequirement.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ResolutionOption.java`
- Create: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/DependencyResolverTest.java`

**Interfaces:**
- Consumes: `ConnectorManifest`, `ConnectorManifest.CapabilityDeclaration`, `ConnectorManifest.RequirementDeclaration`, `ConnectorRegistry`, `ConnectorStatus`
- Produces: `DependencyResolver(List<ConnectorManifest>)`, `DependencyResolver.unmetRequirements(String, ConnectorRegistry)`, `DependencyResolver.resolutionOptions(String, List<String>, ConnectorRegistry)`, `UnmetRequirement(String, List<String>, String, List<ResolutionOption>)`, `ResolutionOption.ProviderOption(String, ConnectorStatus, String)`, `ResolutionOption.ManualOption(String)`

- [ ] **Step 1: Write failing test — qualified requirement matched by qualified provider**

```java
@Test
void qualified_requirement_matched_by_qualified_provider() {
    var signal = manifest("signal", "chat",
            List.of(new RequirementDeclaration("sms", List.of("receive"), "Verification")),
            List.of());
    var numbrrs = manifest("numbrrs", "mobile",
            List.of(),
            List.of(new CapabilityDeclaration("sms", List.of("receive"))));

    var resolver = new DependencyResolver(List.of(signal, numbrrs));
    var registry = registryWith(Map.of("numbrrs", ConnectorStatus.READY));

    var unmet = resolver.unmetRequirements("signal", registry);
    assertThat(unmet).isEmpty();
}
```

- [ ] **Step 2: Write failing test — unqualified provider satisfies qualified requirement**

```java
@Test
void unqualified_provider_satisfies_qualified_requirement() {
    var signal = manifest("signal", "chat",
            List.of(new RequirementDeclaration("sms", List.of("receive"), "Verification")),
            List.of());
    var twilio = manifest("twilio", "mobile",
            List.of(),
            List.of(new CapabilityDeclaration("sms")));

    var resolver = new DependencyResolver(List.of(signal, twilio));
    var registry = registryWith(Map.of("twilio", ConnectorStatus.READY));

    var unmet = resolver.unmetRequirements("signal", registry);
    assertThat(unmet).isEmpty();
}
```

- [ ] **Step 3: Write failing test — no provider returns unmet with resolution options**

```java
@Test
void no_ready_provider_returns_unmet_with_options() {
    var signal = manifest("signal", "chat",
            List.of(new RequirementDeclaration("sms", List.of("receive"), "Verification")),
            List.of());
    var numbrrs = manifest("numbrrs", "mobile",
            List.of(),
            List.of(new CapabilityDeclaration("sms", List.of("receive"))));

    var resolver = new DependencyResolver(List.of(signal, numbrrs));
    var registry = registryWith(Map.of("numbrrs", ConnectorStatus.UNCONFIGURED));

    var unmet = resolver.unmetRequirements("signal", registry);
    assertThat(unmet).hasSize(1);
    assertThat(unmet.getFirst().capability()).isEqualTo("sms");

    var options = unmet.getFirst().options();
    assertThat(options).hasAtLeastOneElementOfType(ResolutionOption.ProviderOption.class);
    assertThat(options).hasAtLeastOneElementOfType(ResolutionOption.ManualOption.class);

    var provider = options.stream()
            .filter(o -> o instanceof ResolutionOption.ProviderOption)
            .map(o -> (ResolutionOption.ProviderOption) o)
            .findFirst().orElseThrow();
    assertThat(provider.connectorId()).isEqualTo("numbrrs");
    assertThat(provider.status()).isEqualTo(ConnectorStatus.UNCONFIGURED);
}
```

- [ ] **Step 4: Write failing test — manual option always present**

```java
@Test
void manual_option_always_present_in_resolution() {
    var signal = manifest("signal", "chat",
            List.of(new RequirementDeclaration("sms", List.of("receive"), "Verification")),
            List.of());

    var resolver = new DependencyResolver(List.of(signal));
    var options = resolver.resolutionOptions("sms", List.of("receive"), registryWith(Map.of()));

    assertThat(options).hasSize(1);
    assertThat(options.getFirst()).isInstanceOf(ResolutionOption.ManualOption.class);
    assertThat(((ResolutionOption.ManualOption) options.getFirst()).capability()).isEqualTo("sms");
}
```

- [ ] **Step 5: Write failing test — connector with no requirements has no unmet**

```java
@Test
void connector_with_no_requirements_has_no_unmet() {
    var telegram = manifest("telegram", "chat", List.of(), List.of());
    var resolver = new DependencyResolver(List.of(telegram));

    var unmet = resolver.unmetRequirements("telegram", registryWith(Map.of()));
    assertThat(unmet).isEmpty();
}
```

- [ ] **Step 6: Write failing test — provider with partial sub-capabilities does not match**

```java
@Test
void provider_missing_required_sub_capability_does_not_match() {
    var consumer = manifest("consumer", "chat",
            List.of(new RequirementDeclaration("sms", List.of("receive", "send"), null)),
            List.of());
    var numbrrs = manifest("numbrrs", "mobile",
            List.of(),
            List.of(new CapabilityDeclaration("sms", List.of("receive"))));

    var resolver = new DependencyResolver(List.of(consumer, numbrrs));
    var registry = registryWith(Map.of("numbrrs", ConnectorStatus.READY));

    var unmet = resolver.unmetRequirements("consumer", registry);
    assertThat(unmet).hasSize(1);
}
```

- [ ] **Step 7: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=DependencyResolverTest`
Expected: Compilation errors — classes don't exist yet

- [ ] **Step 8: Implement ResolutionOption sealed interface**

Create `ResolutionOption.java`:

```java
package io.casehub.connectors.lifecycle;

public sealed interface ResolutionOption {
    record ProviderOption(String connectorId, ConnectorStatus status, String setupPlaybook) implements ResolutionOption {}
    record ManualOption(String capability) implements ResolutionOption {}
}
```

- [ ] **Step 9: Implement UnmetRequirement record**

Create `UnmetRequirement.java`:

```java
package io.casehub.connectors.lifecycle;

import java.util.List;

public record UnmetRequirement(String capability, List<String> sub, String reason, List<ResolutionOption> options) {}
```

- [ ] **Step 10: Implement DependencyResolver**

Create `DependencyResolver.java`:

```java
package io.casehub.connectors.lifecycle;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class DependencyResolver {

    private final Map<String, ConnectorManifest> manifestsById;
    private final Map<String, List<ProviderEntry>> providersByCapability;

    private record ProviderEntry(String connectorId, List<String> sub, String setupPlaybook) {}

    public DependencyResolver(List<ConnectorManifest> manifests) {
        this.manifestsById = new HashMap<>();
        this.providersByCapability = new HashMap<>();
        for (var m : manifests) {
            var decl = m.connector();
            manifestsById.put(decl.id(), m);
            if (decl.provides() != null) {
                for (var cap : decl.provides()) {
                    providersByCapability
                            .computeIfAbsent(cap.capability(), k -> new ArrayList<>())
                            .add(new ProviderEntry(decl.id(), cap.sub(), decl.setupPlaybook()));
                }
            }
        }
    }

    public List<UnmetRequirement> unmetRequirements(String connectorId, ConnectorRegistry registry) {
        var manifest = manifestsById.get(connectorId);
        if (manifest == null || manifest.connector().requires() == null) return List.of();

        var result = new ArrayList<UnmetRequirement>();
        for (var req : manifest.connector().requires()) {
            var options = resolutionOptions(req.capability(), req.sub(), registry);
            boolean satisfied = options.stream()
                    .filter(o -> o instanceof ResolutionOption.ProviderOption p)
                    .map(o -> (ResolutionOption.ProviderOption) o)
                    .anyMatch(p -> p.status() == ConnectorStatus.READY);
            if (!satisfied) {
                result.add(new UnmetRequirement(req.capability(), req.sub(), req.reason(), options));
            }
        }
        return List.copyOf(result);
    }

    public List<ResolutionOption> resolutionOptions(String capability, List<String> sub, ConnectorRegistry registry) {
        var options = new ArrayList<ResolutionOption>();
        var providers = providersByCapability.getOrDefault(capability, List.of());
        for (var entry : providers) {
            if (matches(entry.sub(), sub)) {
                var status = registry.status(entry.connectorId());
                options.add(new ResolutionOption.ProviderOption(
                        entry.connectorId(),
                        status != null ? status : ConnectorStatus.UNCONFIGURED,
                        entry.setupPlaybook()));
            }
        }
        options.add(new ResolutionOption.ManualOption(capability));
        return List.copyOf(options);
    }

    private static boolean matches(List<String> providerSub, List<String> requiredSub) {
        if (requiredSub == null || requiredSub.isEmpty()) return true;
        if (providerSub == null || providerSub.isEmpty()) return true; // unqualified = full
        return providerSub.containsAll(requiredSub);
    }
}
```

- [ ] **Step 11: Add test helper methods to DependencyResolverTest**

```java
private static ConnectorManifest manifest(String id, String platform,
        List<ConnectorManifest.RequirementDeclaration> requires,
        List<ConnectorManifest.CapabilityDeclaration> provides) {
    return new ConnectorManifest(new ConnectorManifest.ConnectorDeclaration(
            id, platform, id + " connector", null, null,
            requires.isEmpty() ? null : requires,
            provides.isEmpty() ? null : provides,
            null, null));
}

private static ConnectorRegistry registryWith(Map<String, ConnectorStatus> entries) {
    var registry = new ConnectorRegistry();
    entries.forEach((id, status) ->
            registry.register(new ConnectorManifest(new ConnectorManifest.ConnectorDeclaration(
                    id, "test", "", null, null, null, null, null, null)), status));
    return registry;
}
```

- [ ] **Step 12: Run all DependencyResolverTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=DependencyResolverTest`
Expected: All 6 tests PASS

- [ ] **Step 13: Run full connectors-api test suite for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test`
Expected: All tests PASS

- [ ] **Step 14: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/DependencyResolver.java
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/UnmetRequirement.java
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/ResolutionOption.java
git add connectors-api/src/test/java/io/casehub/connectors/lifecycle/DependencyResolverTest.java
git commit -m "feat(#365): DependencyResolver with capability matching and resolution options"
```

---

## Batch 3: Manifests + playbook update — end-to-end proof

### Task 3: Update Signal and Numbrrs manifests with requires/provides

**Files:**
- Modify: `chat-signal/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `mobile-numbrrs/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `playbooks/src/main/playbooks/signal-registration.yaml`

**Interfaces:**
- Consumes: `CapabilityDeclaration`, `RequirementDeclaration` schema from Task 1

- [ ] **Step 1: Add requires to Signal manifest**

Add after the `produces` section in `chat-signal/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
  requires:
    - capability: sms
      sub: [receive]
      reason: "SMS verification code during registration"
```

- [ ] **Step 2: Add provides to Numbrrs manifest**

Add after the `produces` section in `mobile-numbrrs/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
  provides:
    - capability: sms
      sub: [receive]
    - capability: number-info
```

- [ ] **Step 3: Update Signal playbook from hardcoded connector to capability-based**

In `playbooks/src/main/playbooks/signal-registration.yaml`, replace:

```yaml
- connector-status:
    connector-id: numbrrs
    require: READY
```

with:

```yaml
- require-capability:
    capability: sms
    sub: [receive]
```

- [ ] **Step 4: Verify manifests parse correctly**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors-api/pom.xml test -Dtest=ConnectorManifestLoaderTest`
Expected: All tests PASS (existing test-fixture manifest has no requires/provides — should still work)

- [ ] **Step 5: Run full build to verify nothing breaks**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f pom.xml clean install -pl connectors-api,chat-signal,mobile-numbrrs -am`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add chat-signal/src/main/resources/META-INF/connector-manifest.yaml
git add mobile-numbrrs/src/main/resources/META-INF/connector-manifest.yaml
git add playbooks/src/main/playbooks/signal-registration.yaml
git commit -m "feat(#365): add requires/provides to Signal and Numbrrs manifests, update playbook"
```

### Task 4: Integration test — resolver with real manifests

**Files:**
- Modify: `integration-tests/pom.xml` (add `chat-signal` dependency for Signal manifest on classpath)
- Modify: `integration-tests/src/test/java/io/casehub/connectors/it/ManifestWiringIntegrationTest.java`

**Interfaces:**
- Consumes: `DependencyResolver`, `ConnectorRegistry`, `ConnectorManifestLoader`, `ResolutionOption`, `UnmetRequirement`

- [ ] **Step 1: Write failing test — resolver discovers Signal requires sms from real manifests**

Add to `ManifestWiringIntegrationTest.java`:

```java
@Inject
ConnectorRegistry registry;

@Test
void resolver_discovers_signal_requires_sms_from_real_manifests() {
    var manifests = ConnectorManifestLoader.discoverFromClasspath();
    var resolver = new DependencyResolver(manifests);

    var unmet = resolver.unmetRequirements("signal", registry);
    // numbrrs is READY in this test (credentials configured) so signal's sms requirement is met
    assertThat(unmet).isEmpty();
}

@Test
void resolver_reports_unmet_when_provider_unconfigured() {
    var manifests = ConnectorManifestLoader.discoverFromClasspath();
    var resolver = new DependencyResolver(manifests);

    // Use an empty registry — no connectors are READY
    var emptyRegistry = new ConnectorRegistry();
    var unmet = resolver.unmetRequirements("signal", emptyRegistry);

    assertThat(unmet).hasSize(1);
    assertThat(unmet.getFirst().capability()).isEqualTo("sms");
    assertThat(unmet.getFirst().options()).anyMatch(o -> o instanceof ResolutionOption.ProviderOption p
            && p.connectorId().equals("numbrrs"));
    assertThat(unmet.getFirst().options()).anyMatch(o -> o instanceof ResolutionOption.ManualOption);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f integration-tests/pom.xml test -Dtest=ManifestWiringIntegrationTest#resolver_discovers_signal_requires_sms_from_real_manifests`
Expected: Compilation error — new imports needed, then FAIL or PASS depending on classpath

- [ ] **Step 3: Add chat-signal dependency and TestProfile exclusion**

Add `chat-signal` test dependency to `integration-tests/pom.xml` (for Signal manifest on classpath).
Add `io.casehub.connectors.chat.signal.**` and `io.casehub.connectors.signal.**` to the TestProfile's `quarkus.arc.exclude-types` (Signal CDI beans need dependencies not present in test).
Add imports for `DependencyResolver`, `ConnectorManifestLoader`, `ResolutionOption`, `UnmetRequirement`.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f integration-tests/pom.xml test -Dtest=ManifestWiringIntegrationTest`
Expected: All tests PASS (both existing 5 tests + 2 new)

- [ ] **Step 4: Commit**

```bash
git add integration-tests/src/test/java/io/casehub/connectors/it/ManifestWiringIntegrationTest.java
git commit -m "test(#365): integration test — resolver with real Signal/Numbrrs manifests"
```

## References

- [2026-11-25-manifest-requires-design.md] — design spec this plan implements
- [connectors-api/.../ConnectorManifest.java] — existing manifest records
- [connectors-api/.../ConnectorManifestTest.java] — existing manifest parser tests
- [connectors-api/.../ConnectorManifestLoaderTest.java] — existing loader tests
- [connectors-api/.../ConnectorRegistry.java] — runtime status tracking
- [chat-signal/.../connector-manifest.yaml] — Signal manifest
- [mobile-numbrrs/.../connector-manifest.yaml] — Numbrrs manifest
- [playbooks/signal-registration.yaml] — Signal registration playbook
- [integration-tests/.../ManifestWiringIntegrationTest.java] — #363 integration test
- [GitHub #365] — focal issue
