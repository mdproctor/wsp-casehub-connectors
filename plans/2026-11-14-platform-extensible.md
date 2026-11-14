# PlatformExtensible Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #225 — PlatformExtensible base interface with as()
**Issue group:** #225

**Goal:** Add a `PlatformExtensible` base interface with `as()` to all platform SPIs, enabling type-safe vendor extension access that works through CDI proxies and simulation wrappers.

**Architecture:** `PlatformExtensible` lives in `connectors-api` as a pure Java interface with a single default method `as(Class<T>)`. All 14 platform SPIs extend it. The simulation decorator processor in the platform repo is updated to traverse super-interface methods via its existing `collectAllMethods()`, so inherited default methods like `as()` get correct delegation through the generated `@Decorator`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jandex (annotation processing)

## Global Constraints

- Pre-release platform — breaking changes cost nothing
- `connectors-api` must remain zero-dependency (pure Java)
- All 14 platform SPIs must extend `PlatformExtensible` for consistency
- Simulation decorator processor change is in `casehubio/platform` repo

---

## Batch 1: Foundation — PlatformExtensible interface and processor fix

### Task 1: Create PlatformExtensible interface with test

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/PlatformExtensible.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/PlatformExtensibleTest.java`

**Interfaces:**
- Produces: `PlatformExtensible` interface with `default <T> Optional<T> as(Class<T> extensionType)` — all subsequent SPI tasks consume this

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

class PlatformExtensibleTest {

    interface TestExtension {
        String extra();
    }

    static class PlainPlatform implements PlatformExtensible {}

    static class ExtendedPlatform implements PlatformExtensible, TestExtension {
        @Override
        public String extra() { return "extended"; }
    }

    @Test
    void as_returnsEmpty_whenNotExtended() {
        PlatformExtensible platform = new PlainPlatform();
        Optional<TestExtension> result = platform.as(TestExtension.class);
        assertThat(result).isEmpty();
    }

    @Test
    void as_returnsPresent_whenExtended() {
        PlatformExtensible platform = new ExtendedPlatform();
        Optional<TestExtension> result = platform.as(TestExtension.class);
        assertThat(result).isPresent();
        assertThat(result.get().extra()).isEqualTo("extended");
    }

    @Test
    void as_returnsEmpty_forUnrelatedType() {
        PlatformExtensible platform = new ExtendedPlatform();
        Optional<Runnable> result = platform.as(Runnable.class);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=PlatformExtensibleTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `PlatformExtensible` does not exist yet

- [ ] **Step 3: Write minimal implementation**

Use `ide_create_file` or Write to create:

```java
package io.casehub.connectors;

import java.util.Optional;

public interface PlatformExtensible {
    default <T> Optional<T> as(Class<T> extensionType) {
        if (extensionType.isInstance(this)) {
            return Optional.of(extensionType.cast(this));
        }
        return Optional.empty();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=PlatformExtensibleTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/PlatformExtensible.java connectors-api/src/test/java/io/casehub/connectors/PlatformExtensibleTest.java
git commit -m "feat(#225): add PlatformExtensible interface with as() default method"
```

### Task 2: Update SimulationDecoratorProcessor to traverse super-interface methods

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/platform/simulation-generator/src/main/java/io/casehub/platform/simulation/generator/SimulationDecoratorProcessor.java:212` — change main loop from `spiClass.methods()` to `collectAllMethods(spiClass, index)` with deduplication
- Create: `/Users/mdproctor/claude/casehub/platform/simulation-generator/src/test/java/io/casehub/platform/simulation/generator/test/TestExtensibleBase.java`
- Create: `/Users/mdproctor/claude/casehub/platform/simulation-generator/src/test/java/io/casehub/platform/simulation/generator/test/TestExtensibleSpi.java`
- Modify: `/Users/mdproctor/claude/casehub/platform/simulation-generator/src/test/java/io/casehub/platform/simulation/generator/SimulationDecoratorProcessorTest.java` — add tests for inherited method generation

**Interfaces:**
- Consumes: The pattern from `PlatformExtensible` (a super-interface with a default method returning `Optional<T>`)
- Produces: Updated processor that generates overrides for inherited methods

- [ ] **Step 1: Write test fixtures — super-interface and SPI extending it**

Create `TestExtensibleBase.java`:

```java
package io.casehub.platform.simulation.generator.test;

import java.util.Optional;

public interface TestExtensibleBase {
    default <T> Optional<T> as(Class<T> extensionType) {
        if (extensionType.isInstance(this)) {
            return Optional.of(extensionType.cast(this));
        }
        return Optional.empty();
    }
}
```

Create `TestExtensibleSpi.java`:

```java
package io.casehub.platform.simulation.generator.test;

import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "extensible-spi")
public interface TestExtensibleSpi extends TestExtensibleBase {
    String resolve(String id);
}
```

- [ ] **Step 2: Write the failing test**

Add to `SimulationDecoratorProcessorTest.java`:

```java
@Test
void inheritedMethodsFromSuperInterfaceGenerated() {
    var sources = new SimulationDecoratorProcessor().generateFromIndex(index);
    String source = findSource(sources, "SimulatedTestExtensibleSpi");
    assertThat(source).contains("\"extensible-spi.as\"");
    assertThat(source).contains("delegate.as(");
}

@Test
void inheritedMethodQNConstantsGenerated() {
    var sources = new SimulationDecoratorProcessor().generateFromIndex(index);
    String source = findSource(sources, "TestExtensibleSpiQN");
    assertThat(source).contains("AS = \"extensible-spi.as\"");
    assertThat(source).contains("RESOLVE = \"extensible-spi.resolve\"");
}
```

Also update `buildIndex()` to index the new classes:
```java
indexer.indexClass(TestExtensibleBase.class);
indexer.indexClass(TestExtensibleSpi.class);
```

And update the count assertion in `generatesSourceForAllSimulationEligibleInterfaces`:
```java
assertThat(classNames).hasSize(12); // was 10, +2 for TestExtensibleSpi (decorator + QN)
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl simulation-generator -Dtest=SimulationDecoratorProcessorTest#inheritedMethodsFromSuperInterfaceGenerated -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL — generated source does not contain `"extensible-spi.as"` (only direct methods are generated)

- [ ] **Step 4: Implement — change main loop to use collectAllMethods**

In `SimulationDecoratorProcessor.java`, modify `generateDecoratorSource()`. Replace the main method iteration (around line 212):

```java
// Before:
for (final MethodInfo method : spiClass.methods()) {
    if (!isOverridable(method)) {continue;}
    ...
}

// After:
final Set<String> seen = new HashSet<>();
for (final MethodInfo method : collectAllMethods(spiClass, index)) {
    if (!seen.add(methodSignatureKey(method))) {continue;}
    ...
}
```

Add a helper method for deduplication by name+param-count (handles overloads):

```java
private static String methodSignatureKey(final MethodInfo method) {
    return method.name() + ":" + method.parameterTypes().size();
}
```

Also update `generateQNSource()` to use the same `collectAllMethods` traversal (around line 323) so QN constants include inherited methods. The existing `Set<String> emittedConstants` handles deduplication.

Also update `addParameterEntries()` to use `collectAllMethods` (around line 553) so parameter entries include inherited methods. The existing `Set<String> seen` handles deduplication.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl simulation-generator -Dtest=SimulationDecoratorProcessorTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: ALL PASS — existing tests still pass, new tests pass

- [ ] **Step 6: Commit (platform repo)**

```bash
git -C /Users/mdproctor/claude/casehub/platform add simulation-generator/src/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#225): processor traverses super-interface methods via collectAllMethods"
```

---

## Batch 2: SPI wiring — all platform SPIs extend PlatformExtensible

### Task 3: Add PlatformExtensible to all 14 platform SPIs

**Files:**
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java:48` — add `extends PlatformExtensible`
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java:17` — add `extends PlatformExtensible`
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/CalendarPlatform.java:14` — add `extends PlatformExtensible`
- Modify: `bank-spi/src/main/java/io/casehub/connectors/bank/spi/BankPlatform.java:7` — add `extends PlatformExtensible`
- Modify: `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatform.java:14` — add `extends PlatformExtensible`
- Modify: `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java:16` — add `extends PlatformExtensible`
- Modify: `location-spi/src/main/java/io/casehub/connectors/location/spi/LocationPlatform.java:17` — add `extends PlatformExtensible`
- Modify: `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatform.java:19` — add `extends PlatformExtensible`
- Modify: `travel-spi/src/main/java/io/casehub/connectors/travel/spi/TravelPlatform.java:19` — add `extends PlatformExtensible`
- Modify: `project-spi/src/main/java/io/casehub/connectors/project/spi/ProjectPlatform.java:18` — add `extends PlatformExtensible`
- Modify: `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthPlatform.java:19` — add `extends PlatformExtensible`
- Modify: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatform.java:7` — add `extends PlatformExtensible`
- Modify: `mobile-spi/src/main/java/io/casehub/connectors/mobile/spi/MobilePlatform.java:23` — add `extends PlatformExtensible`
- Modify: `web-spi/src/main/java/io/casehub/connectors/web/spi/WebPlatform.java:16` — add `extends PlatformExtensible`

**Interfaces:**
- Consumes: `PlatformExtensible` from `connectors-api` (Task 1)
- Produces: All platform SPIs now have `as()` available

- [ ] **Step 1: Add `extends PlatformExtensible` to each SPI interface**

For each SPI, use `ide_edit_member` (member = interface name) to add `extends PlatformExtensible` and add `import io.casehub.connectors.PlatformExtensible;`.

Pattern for each file — change:
```java
public interface XxxPlatform {
```
to:
```java
public interface XxxPlatform extends PlatformExtensible {
```

All 14 SPIs follow the same pattern. No existing `extends` clause on any of them.

- [ ] **Step 2: Run `ide_diagnostics` to verify no compilation errors**

Run: `ide_diagnostics` on the connectors project to verify all 14 SPIs compile cleanly.

- [ ] **Step 3: Build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /Users/mdproctor/claude/casehub/connectors/pom.xml -T 4`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/connectors/pom.xml -T 4`
Expected: All tests pass — purely additive change, no existing tests should break

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#225): all 14 platform SPIs extend PlatformExtensible"
```

---

## Batch 3: Verification — build and docs

### Task 4: Full build verification and doc updates

**Files:**
- Modify: `docs/guides/consumer-guide.md` — add PlatformExtensible section
- Modify: `docs/guides/contributor-guide.md` — add vendor extension pattern

**Interfaces:**
- Consumes: All changes from Tasks 1-3

- [ ] **Step 1: Full build across both repos**

Run platform build first (processor change):
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/platform/pom.xml -T 4
```

Then connectors build (picks up new processor):
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml -T 4
```

Expected: Both BUILD SUCCESS

- [ ] **Step 2: Update consumer-guide.md**

Add a section on using `as()` for vendor extensions:

```markdown
### Vendor Extensions

Some providers expose capabilities beyond the general SPI. Use `as()` to access them type-safely:

\```java
documentPlatform.as(MicrosoftDriveExtensions.class).ifPresent(msft ->
    msft.sharePoint().listSites()
);
\```

`as()` returns `Optional.empty()` for providers that don't implement the extension. It works correctly through CDI proxies and simulation wrappers.
```

- [ ] **Step 3: Update contributor-guide.md**

Add a section on implementing vendor extensions:

```markdown
### Creating Vendor Extensions

1. Define an extension interface in your provider module:
   \```java
   public interface MicrosoftDriveExtensions {
       SharePointOperations sharePoint();
   }
   \```

2. Implement both the SPI and the extension on your provider class:
   \```java
   public class MicrosoftDocumentPlatform implements DocumentPlatform, MicrosoftDriveExtensions {
       // ...
   }
   \```

Consumers access extensions via `platform.as(ExtensionType.class)`. No `instanceof` checks — `as()` works through CDI proxies and simulation decorators.
```

- [ ] **Step 4: Commit**

```bash
git add docs/
git commit -m "docs(#225): add PlatformExtensible vendor extension docs to guides"
```

## References

- [2026-11-14-platform-extensible-design.md] — design spec this plan implements
- [SimulationDecoratorProcessor.java:212, 594-615] — main loop and collectAllMethods
- [ChatPlatform.java:48] — representative SPI interface
- [PlatformExtensibleTest fixture pattern] — test uses inner interfaces for extension type
- [Decision review R1-02] — generic collectAllMethods approach
- [GitHub #225] — focal issue
