# Seed Externalisation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #329 — Ref providers: externalise seed data — configurable corpus directory, strip from production JARs
**Issue group:** #329

**Goal:** Externalise ref provider seed YAML from production JARs into a configurable file-based corpus directory, with classpath fallback for tests and graceful empty-data degradation.

**Architecture:** A shared `SeedDataResolver` utility in `connectors-api` resolves seed data via three-tier fallback: file-based directory → classpath → empty. Each ref module's `SeedLoader` becomes a thin wrapper. Seed YAML moves from `src/main/resources/` to `src/test/resources/`. The MCP server bundles corpus files as loose files alongside `quarkus-run.jar` and uses a CDI startup observer to configure the seed directory.

**Tech Stack:** Java 21, Jackson YAML (via YamlMappers), Quarkus CDI, Maven resources plugin

## Global Constraints

- Java source level 21, run on Java 26 JVM
- `connectors-api` must remain zero-dependency (no CDI, no MicroProfile Config)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All ref modules use `YamlMappers.create()` from `casehub-platform-yaml-jackson`

---

## Batch 1: SeedDataResolver foundation

### Task 1: Create SeedDataResolver in connectors-api

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/seed/SeedDataResolver.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/seed/SeedDataResolverTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `SeedDataResolver.resolve(String domain, String filename): Optional<InputStream>`, `SeedDataResolver.setSeedDir(Path dir): void`

- [ ] **Step 1: Write failing tests for SeedDataResolver**

Create the test class covering three-tier resolution:

```java
package io.casehub.connectors.seed;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class SeedDataResolverTest {

    @AfterEach
    void resetSeedDir() {
        SeedDataResolver.setSeedDir(null);
    }

    @Test
    void resolve_returnsEmpty_whenNoSeedDirAndNoClasspath() {
        assertThat(SeedDataResolver.resolve("calendar", "nonexistent.yaml")).isEmpty();
    }

    @Test
    void resolve_returnsClasspathResource_whenNoSeedDir() {
        // src/test/resources/seed/test-data.yaml must exist for this test
        var result = SeedDataResolver.resolve("test", "test-data.yaml");
        assertThat(result).isPresent();
    }

    @Test
    void resolve_returnsFileBasedResource_whenSeedDirSet(@TempDir Path tempDir) throws IOException {
        Path domainDir = tempDir.resolve("calendar");
        Files.createDirectories(domainDir);
        Files.writeString(domainDir.resolve("events.yaml"), "- name: test");

        SeedDataResolver.setSeedDir(tempDir);

        var result = SeedDataResolver.resolve("calendar", "events.yaml");
        assertThat(result).isPresent();
        try (var is = result.get()) {
            assertThat(new String(is.readAllBytes())).contains("test");
        }
    }

    @Test
    void resolve_fileBasedTakesPrecedenceOverClasspath(@TempDir Path tempDir) throws IOException {
        Path domainDir = tempDir.resolve("test");
        Files.createDirectories(domainDir);
        Files.writeString(domainDir.resolve("test-data.yaml"), "file-based-content");

        SeedDataResolver.setSeedDir(tempDir);

        var result = SeedDataResolver.resolve("test", "test-data.yaml");
        assertThat(result).isPresent();
        try (var is = result.get()) {
            assertThat(new String(is.readAllBytes())).isEqualTo("file-based-content");
        }
    }

    @Test
    void resolve_fallsBackToClasspath_whenFileNotInSeedDir(@TempDir Path tempDir) {
        SeedDataResolver.setSeedDir(tempDir);
        // tempDir exists but has no "test/test-data.yaml"
        var result = SeedDataResolver.resolve("test", "test-data.yaml");
        assertThat(result).isPresent();
    }

    @Test
    void resolve_returnsEmpty_whenSeedDirSetButNoFileAndNoClasspath(@TempDir Path tempDir) {
        SeedDataResolver.setSeedDir(tempDir);
        assertThat(SeedDataResolver.resolve("calendar", "nonexistent.yaml")).isEmpty();
    }
}
```

- [ ] **Step 2: Create test resource file for classpath fallback tests**

Create `connectors-api/src/test/resources/seed/test-data.yaml`:

```yaml
- name: classpath-test-entry
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=SeedDataResolverTest -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: compilation error — `SeedDataResolver` class does not exist

- [ ] **Step 4: Implement SeedDataResolver**

```java
package io.casehub.connectors.seed;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Optional;

public final class SeedDataResolver {

    private static volatile Path seedDir;

    public static void setSeedDir(Path dir) {
        seedDir = dir;
    }

    public static Optional<InputStream> resolve(String domain, String filename) {
        Path dir = seedDir;
        if (dir != null) {
            Path file = dir.resolve(domain).resolve(filename);
            if (Files.exists(file)) {
                try {
                    return Optional.of(Files.newInputStream(file));
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            }
        }
        InputStream is = SeedDataResolver.class.getResourceAsStream("/seed/" + filename);
        return Optional.ofNullable(is);
    }

    private SeedDataResolver() {}
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=SeedDataResolverTest -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: all 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/seed/SeedDataResolver.java connectors-api/src/test/java/io/casehub/connectors/seed/SeedDataResolverTest.java connectors-api/src/test/resources/seed/test-data.yaml
git commit -m "feat(connectors-api): add SeedDataResolver with three-tier fallback Refs #329"
```

---

## Batch 2: Migrate SeedLoaders and move YAML to test scope

### Task 2: Migrate standard SeedLoaders (10 modules)

Migrate all SeedLoaders that follow the standard pattern (all except mobile-ref and health-ref). For each module: change SeedLoader to use `SeedDataResolver.resolve()` with graceful empty fallback, then move seed YAML from `src/main/resources/seed/` to `src/test/resources/seed/`.

**Files (per module — 10 modules listed below):**
- Modify: `<module>/src/main/java/.../SeedLoader.java`
- Move: `<module>/src/main/resources/seed/*.yaml` → `<module>/src/test/resources/seed/*.yaml` (use bash `mv` — these are data files, not source code)

**Interfaces:**
- Consumes: `SeedDataResolver.resolve(String domain, String filename): Optional<InputStream>`
- Produces: same method signatures as before, but returning empty collections instead of throwing when no seed data is found

**The pattern — applied to all 10 modules:**

Each SeedLoader method changes from:

```java
static List<CalendarInfo> loadCalendars() {
    try (var is = SeedLoader.class.getResourceAsStream("/seed/calendars.yaml")) {
        if (is == null) throw new IllegalStateException("Missing /seed/calendars.yaml");
        return YAML.readValue(is, new TypeReference<>() {});
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

to:

```java
static List<CalendarInfo> loadCalendars() {
    var maybeIs = SeedDataResolver.resolve("calendar", "calendars.yaml");
    if (maybeIs.isEmpty()) return List.of();
    try (var is = maybeIs.get()) {
        return YAML.readValue(is, new TypeReference<>() {});
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

Add import: `import io.casehub.connectors.seed.SeedDataResolver;`

**Module-by-module details:**

| Module | Domain | SeedLoader methods → return type on empty | YAML files to move |
|--------|--------|------------------------------------------|-------------------|
| `calendar-ref` | `calendar` | `loadCalendars()` → `List.of()`, `loadEvents()` → `List.of()` | calendars.yaml, events.yaml |
| `bank-ref` | `bank` | `loadAccounts()` → `List.of()`, `loadBalances()` → `Map.of()`, `loadTransactions()` → `List.of()` | accounts.yaml, balances.yaml, transactions.yaml |
| `chat-ref` | `chat` | `loadChannels()` → `List.of()`, `loadMembers()` → `List.of()` | channels.yaml, members.yaml |
| `commerce-ref` | `commerce` | `loadProducts()` → `List.of()` | products.yaml |
| `contacts-ref` | `contacts` | `loadContacts()` → `List.of()`, `loadGroups()` → `List.of()` | contacts.yaml, groups.yaml |
| `document-ref` | `document` | `loadFolders()` → `List.of()`, `loadFiles()` → `List.of()` | files.yaml, folders.yaml |
| `email-ref` | `email` | `loadMessages()` → `List.of()` | messages.yaml |
| `location-ref` | `location` | `loadPlaces()` → `List.of()`, `loadGeocoding()` → `List.of()` | geocoding.yaml, places.yaml |
| `project-ref` | `project` | `load()` → `null` (returns `ProjectSeed` not a list — caller must null-check) | project-data.yaml |
| `travel-ref` | `travel` | `loadTransport()` → `List.of()`, `loadAccommodation()` → `List.of()` | accommodation.yaml, transport.yaml |

**Special case — project-ref:** `SeedLoader.load()` returns a single `ProjectSeed` object (not a list). Change to:

```java
static ProjectSeed load() {
    var maybeIs = SeedDataResolver.resolve("project", "project-data.yaml");
    if (maybeIs.isEmpty()) return null;
    try (var is = maybeIs.get()) {
        return YAML.readValue(is, ProjectSeed.class);
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

The caller (`InMemoryProjectBackend` or similar) must handle `null` — guard with `if (data != null) { ... }` around the seeding logic.

**Special case — bank-ref `loadBalances()`:** Returns `Map<String, AccountBalance>`. Change to:

```java
static Map<String, AccountBalance> loadBalances() {
    var maybeIs = SeedDataResolver.resolve("bank", "balances.yaml");
    if (maybeIs.isEmpty()) return Map.of();
    try (var is = maybeIs.get()) {
        List<AccountBalance> list = YAML.readValue(is, new TypeReference<>() {});
        return list.stream().collect(Collectors.toMap(AccountBalance::accountId, b -> b));
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

- [ ] **Step 1: Migrate calendar-ref SeedLoader (reference implementation)**

Modify `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/SeedLoader.java`:
- Add `import io.casehub.connectors.seed.SeedDataResolver;`
- Change `loadCalendars()` and `loadEvents()` to use `SeedDataResolver.resolve("calendar", ...)` with `List.of()` fallback

- [ ] **Step 2: Run calendar-ref tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-ref -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: PASS (seed YAML is still in main resources, classpath resolves it)

- [ ] **Step 3: Move calendar-ref seed YAML to test scope**

```bash
mkdir -p calendar-ref/src/test/resources/seed
mv calendar-ref/src/main/resources/seed/calendars.yaml calendar-ref/src/test/resources/seed/
mv calendar-ref/src/main/resources/seed/events.yaml calendar-ref/src/test/resources/seed/
rmdir calendar-ref/src/main/resources/seed
```

- [ ] **Step 4: Run calendar-ref tests again to verify classpath fallback works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-ref -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: PASS (test classpath includes test resources)

- [ ] **Step 5: Migrate remaining 9 standard SeedLoaders**

Apply the same pattern to: bank-ref, chat-ref, commerce-ref, contacts-ref, document-ref, email-ref, location-ref, project-ref, travel-ref. For each:
1. Modify `SeedLoader.java` — add `SeedDataResolver` import, change each method to use `resolve()` with empty fallback
2. Move YAML files from `src/main/resources/seed/` to `src/test/resources/seed/`

For project-ref: also modify the caller of `SeedLoader.load()` to handle `null` return.

- [ ] **Step 6: Run all ref module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-ref,bank-ref,chat-ref,commerce-ref,contacts-ref,document-ref,email-ref,location-ref,project-ref,travel-ref -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(ref): migrate 10 SeedLoaders to SeedDataResolver and move YAML to test scope Refs #329"
```

### Task 3: Migrate health-ref SeedLoader (readTree variant)

health-ref's `loadRecords()` uses `readTree()` for polymorphic parsing. The migration is the same resolve pattern but preserves the `readTree` logic.

**Files:**
- Modify: `health-ref/src/main/java/io/casehub/connectors/health/ref/SeedLoader.java`
- Move: `health-ref/src/main/resources/seed/*.yaml` → `health-ref/src/test/resources/seed/*.yaml`

**Interfaces:**
- Consumes: `SeedDataResolver.resolve(String domain, String filename): Optional<InputStream>`
- Produces: same signatures, empty fallback

- [ ] **Step 1: Modify health-ref SeedLoader**

Add `import io.casehub.connectors.seed.SeedDataResolver;`

Change `loadProviders()`:
```java
static List<Provider> loadProviders() {
    var maybeIs = SeedDataResolver.resolve("health", "providers.yaml");
    if (maybeIs.isEmpty()) return List.of();
    try (var is = maybeIs.get()) {
        return YAML.readValue(is, new TypeReference<>() {});
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

Change `loadAppointments()`:
```java
static Map<String, List<Appointment>> loadAppointments() {
    var maybeIs = SeedDataResolver.resolve("health", "appointments.yaml");
    if (maybeIs.isEmpty()) return Map.of();
    try (var is = maybeIs.get()) {
        return YAML.readValue(is, new TypeReference<>() {});
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

Change `loadRecords()` (preserving `readTree` logic):
```java
static Map<String, List<MedicalRecord>> loadRecords() {
    var maybeIs = SeedDataResolver.resolve("health", "records.yaml");
    if (maybeIs.isEmpty()) return Map.of();
    try (var is = maybeIs.get()) {
        JsonNode root = YAML.readTree(is);
        Map<String, List<MedicalRecord>> result = new HashMap<>();
        var fields = root.fields();
        while (fields.hasNext()) {
            var entry = fields.next();
            String userId = entry.getKey();
            List<MedicalRecord> userRecords = new ArrayList<>();
            for (JsonNode node : entry.getValue()) {
                userRecords.add(parseRecord(node));
            }
            result.put(userId, userRecords);
        }
        return result;
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

- [ ] **Step 2: Move health-ref seed YAML to test scope**

```bash
mkdir -p health-ref/src/test/resources/seed
mv health-ref/src/main/resources/seed/appointments.yaml health-ref/src/test/resources/seed/
mv health-ref/src/main/resources/seed/providers.yaml health-ref/src/test/resources/seed/
mv health-ref/src/main/resources/seed/records.yaml health-ref/src/test/resources/seed/
rmdir health-ref/src/main/resources/seed
```

- [ ] **Step 3: Run health-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl health-ref -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(health-ref): migrate SeedLoader readTree variant to SeedDataResolver Refs #329"
```

### Task 4: Migrate mobile-ref SeedLoader (normalise path convention)

mobile-ref uses a package-scoped classpath path (`/io/casehub/connectors/mobile/ref/seed/`). Normalise to `/seed/` convention and move YAML to test scope.

**Files:**
- Modify: `mobile-ref/src/main/java/io/casehub/connectors/mobile/ref/SeedLoader.java`
- Move: `mobile-ref/src/main/resources/io/casehub/connectors/mobile/ref/seed/*.yaml` → `mobile-ref/src/test/resources/seed/*.yaml`

**Interfaces:**
- Consumes: `SeedDataResolver.resolve(String domain, String filename): Optional<InputStream>`
- Produces: same signatures, empty fallback

- [ ] **Step 1: Modify mobile-ref SeedLoader**

Add `import io.casehub.connectors.seed.SeedDataResolver;`

Change the private `load()` helper and all public methods. The existing helper:
```java
private static <T> List<T> load(String resource, TypeReference<List<T>> type) {
    try (var is = SeedLoader.class.getResourceAsStream(resource)) {
        if (is == null) {
            throw new IllegalStateException("Seed data not found: " + resource);
        }
        return YAML.readValue(is, type);
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

Replace with:
```java
private static <T> List<T> load(String filename, TypeReference<List<T>> type) {
    var maybeIs = SeedDataResolver.resolve("mobile", filename);
    if (maybeIs.isEmpty()) return List.of();
    try (var is = maybeIs.get()) {
        return YAML.readValue(is, type);
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}
```

Change callers from full package paths to just filenames:
```java
static List<SeedNumber> loadNumbers() {
    return load("numbers.yaml", new TypeReference<>() {});
}

static List<SeedConversation> loadConversations() {
    return load("conversations.yaml", new TypeReference<>() {});
}

static List<SeedVoicemail> loadVoicemails() {
    return load("voicemails.yaml", new TypeReference<>() {});
}

static List<SeedMessage> loadMessages() {
    return load("messages.yaml", new TypeReference<>() {});
}
```

- [ ] **Step 2: Move mobile-ref seed YAML to normalised location**

```bash
mkdir -p mobile-ref/src/test/resources/seed
mv mobile-ref/src/main/resources/io/casehub/connectors/mobile/ref/seed/numbers.yaml mobile-ref/src/test/resources/seed/
mv mobile-ref/src/main/resources/io/casehub/connectors/mobile/ref/seed/conversations.yaml mobile-ref/src/test/resources/seed/
mv mobile-ref/src/main/resources/io/casehub/connectors/mobile/ref/seed/voicemails.yaml mobile-ref/src/test/resources/seed/
mv mobile-ref/src/main/resources/io/casehub/connectors/mobile/ref/seed/messages.yaml mobile-ref/src/test/resources/seed/
```

Remove the now-empty directory tree:
```bash
rm -rf mobile-ref/src/main/resources/io
```

- [ ] **Step 3: Run mobile-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mobile-ref -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(mobile-ref): normalise seed path convention and migrate to SeedDataResolver Refs #329"
```

---

## Batch 3: MCP server corpus bundling

### Task 5: Configure MCP server corpus and seed directory

Bundle seed YAML as loose files alongside `quarkus-run.jar` and configure `SeedDataResolver` at startup.

**Files:**
- Modify: `mcp-server-connectors/pom.xml` (add maven-resources-plugin executions)
- Create: `mcp-server-connectors/src/main/java/io/casehub/mcp/server/SeedDataResolverConfig.java`
- Modify: `mcp-server-connectors/src/main/resources/application.properties`
- Test: `mcp-server-connectors/src/test/java/io/casehub/mcp/server/SeedDataResolverConfigTest.java`

**Interfaces:**
- Consumes: `SeedDataResolver.setSeedDir(Path dir): void`
- Produces: CDI startup observer that configures the seed directory

- [ ] **Step 1: Write failing test for SeedDataResolverConfig**

```java
package io.casehub.mcp.server;

import io.casehub.connectors.seed.SeedDataResolver;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class SeedDataResolverConfigTest {

    @Test
    void seedDataResolver_resolves_corpusFiles() {
        // The test application.properties sets casehub.ref.seed-dir to a test corpus
        // SeedDataResolverConfig should have set the seed dir at startup
        var result = SeedDataResolver.resolve("calendar", "calendars.yaml");
        assertThat(result).isPresent();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors -Dtest=SeedDataResolverConfigTest -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: FAIL — `SeedDataResolverConfig` does not exist

- [ ] **Step 3: Create SeedDataResolverConfig**

```java
package io.casehub.mcp.server;

import io.casehub.connectors.seed.SeedDataResolver;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.nio.file.Path;
import java.util.Optional;

@ApplicationScoped
public class SeedDataResolverConfig {

    @ConfigProperty(name = "casehub.ref.seed-dir")
    Optional<String> seedDir;

    void onStart(@Observes StartupEvent ev) {
        seedDir.map(Path::of).ifPresent(SeedDataResolver::setSeedDir);
    }
}
```

- [ ] **Step 4: Add maven-resources-plugin executions to mcp-server-connectors/pom.xml**

Add to the `<build><plugins>` section, after the quarkus-maven-plugin:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-seed-corpus</id>
            <phase>prepare-package</phase>
            <goals><goal>copy-resources</goal></goals>
            <configuration>
                <outputDirectory>${project.build.directory}/quarkus-app/corpus</outputDirectory>
                <resources>
                    <resource><directory>${project.basedir}/../bank-ref/src/test/resources/seed</directory><targetPath>bank</targetPath></resource>
                    <resource><directory>${project.basedir}/../calendar-ref/src/test/resources/seed</directory><targetPath>calendar</targetPath></resource>
                    <resource><directory>${project.basedir}/../chat-ref/src/test/resources/seed</directory><targetPath>chat</targetPath></resource>
                    <resource><directory>${project.basedir}/../commerce-ref/src/test/resources/seed</directory><targetPath>commerce</targetPath></resource>
                    <resource><directory>${project.basedir}/../contacts-ref/src/test/resources/seed</directory><targetPath>contacts</targetPath></resource>
                    <resource><directory>${project.basedir}/../document-ref/src/test/resources/seed</directory><targetPath>document</targetPath></resource>
                    <resource><directory>${project.basedir}/../email-ref/src/test/resources/seed</directory><targetPath>email</targetPath></resource>
                    <resource><directory>${project.basedir}/../health-ref/src/test/resources/seed</directory><targetPath>health</targetPath></resource>
                    <resource><directory>${project.basedir}/../location-ref/src/test/resources/seed</directory><targetPath>location</targetPath></resource>
                    <resource><directory>${project.basedir}/../mobile-ref/src/test/resources/seed</directory><targetPath>mobile</targetPath></resource>
                    <resource><directory>${project.basedir}/../project-ref/src/test/resources/seed</directory><targetPath>project</targetPath></resource>
                    <resource><directory>${project.basedir}/../travel-ref/src/test/resources/seed</directory><targetPath>travel</targetPath></resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

- [ ] **Step 5: Update application.properties**

Add to `mcp-server-connectors/src/main/resources/application.properties`:

```properties
# Seed corpus directory — resolved relative to working directory
casehub.ref.seed-dir=corpus/
```

- [ ] **Step 6: Update native profile resource includes**

In `mcp-server-connectors/pom.xml`, change the native profile from:

```xml
<quarkus.native.resources.includes>seed/*.yaml,seed/*.json</quarkus.native.resources.includes>
```

to:

```xml
<quarkus.native.resources.includes>corpus/**/*.yaml</quarkus.native.resources.includes>
```

- [ ] **Step 7: Set up test corpus for SeedDataResolverConfigTest**

Add to `mcp-server-connectors/src/test/resources/application.properties`:

```properties
casehub.ref.seed-dir=src/test/resources/test-corpus/
```

Create `mcp-server-connectors/src/test/resources/test-corpus/calendar/calendars.yaml`:

```yaml
- id: test-cal
  name: Test Calendar
  primary: true
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: PASS

- [ ] **Step 9: Run full build to verify everything compiles and corpus is bundled**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: BUILD SUCCESS

Verify corpus files are in the distribution:
```bash
ls mcp-server-connectors/target/quarkus-app/corpus/calendar/
```
Expected: `calendars.yaml  events.yaml`

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(mcp-server): bundle seed corpus and configure SeedDataResolver at startup Refs #329"
```

---

## Batch 4: Cleanup and verification

### Task 6: File the first-run bootstrap issue and final verification

**Files:**
- No code changes — issue filing and full test run

- [ ] **Step 1: File the first-run seed bootstrap issue**

Create a GitHub issue on casehubio/connectors:
- Title: "MCP server: first-run seed corpus bootstrap — CLI/web prompt to extract defaults"
- Body: Reference #329 (infrastructure delivered), #324 (MCP server packaging epic). Describe the first-run experience: detect no corpus directory, prompt user to extract defaults to a chosen location.
- Labels: enhancement, scale: S, complexity: Med

- [ ] **Step 2: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/slots/225/connectors/pom.xml`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 3: Verify production JARs are clean**

Check that no ref module production JAR contains seed YAML:
```bash
jar tf calendar-ref/target/casehub-connectors-calendar-ref-0.4-SNAPSHOT.jar | grep seed
```
Expected: no output (no seed files in production JAR)

- [ ] **Step 4: Commit any remaining changes**

If any cleanup was needed, commit it.

## References

- [specs/issue-329-seed-externalisation/2026-11-19-seed-externalisation-design.md] — design spec
- [connectors-api/src/main/java/io/casehub/connectors/] — SeedDataResolver location
- [calendar-ref/src/main/java/.../SeedLoader.java] — representative standard pattern
- [health-ref/src/main/java/.../SeedLoader.java] — readTree variant
- [mobile-ref/src/main/java/.../SeedLoader.java] — package-scoped path variant
- [mcp-server-connectors/pom.xml] — MCP server packaging
- [GitHub #329] — focal issue
- [GitHub #324] — MCP server packaging epic
