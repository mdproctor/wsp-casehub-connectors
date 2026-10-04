# Ref Normalisation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #141 — refactor: normalise ref implementations (Phase 1)
**Issue group:** #141, #138

**Goal:** Normalise all 9 ref implementations to consistent patterns (CDI wiring, `supports()`, pagination) and fix DocumentPlatform's `@SimulationEligible` annotation, producing a clean baseline for simulation integration.

**Architecture:** Four independent normalisation concerns applied across 9 ref modules. Each concern is a batch — paginate extraction first (shared utility), then supports() normalisation (no dependencies), then CDI wiring (may interact with backend construction), then the DocumentPlatform annotation fix.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Jakarta)

## Global Constraints

- Java 21 source level, Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All SPI interfaces in connectors use `String id()` (protocol: spi-id-method-naming)
- Shared HTTP clients use `HttpHelper.CLIENT` (protocol: shared-http-client)
- Every commit references an issue

---

## Batch 1: Extract paginate() to shared utility

### Task 1: Extract PaginationHelper to connectors-api

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/PaginationHelper.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/PaginationHelperTest.java`

**Interfaces:**
- Consumes: `Page<T>(List<T> items, String nextCursor, boolean hasMore)`, `PageRequest` (both in `connectors-api`)
- Produces: `PaginationHelper.paginate(List<T> all, PageRequest pagination) → Page<T>` — static utility method

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class PaginationHelperTest {

    @Test
    void paginatesFromStart() {
        var items = List.of("a", "b", "c", "d", "e");
        var page = PaginationHelper.paginate(items, new PageRequest(null, 3));
        assertThat(page.items()).containsExactly("a", "b", "c");
        assertThat(page.hasMore()).isTrue();
        assertThat(page.nextCursor()).isEqualTo("3");
    }

    @Test
    void paginatesFromCursor() {
        var items = List.of("a", "b", "c", "d", "e");
        var page = PaginationHelper.paginate(items, new PageRequest("3", 3));
        assertThat(page.items()).containsExactly("d", "e");
        assertThat(page.hasMore()).isFalse();
        assertThat(page.nextCursor()).isNull();
    }

    @Test
    void defaultsPageSizeTo20() {
        var items = List.of("a", "b", "c");
        var page = PaginationHelper.paginate(items, new PageRequest(null, 0));
        assertThat(page.items()).containsExactly("a", "b", "c");
        assertThat(page.hasMore()).isFalse();
    }

    @Test
    void emptyList() {
        var page = PaginationHelper.paginate(List.of(), new PageRequest(null, 10));
        assertThat(page.items()).isEmpty();
        assertThat(page.hasMore()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=PaginationHelperTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `PaginationHelper` does not exist

- [ ] **Step 3: Write minimal implementation**

```java
package io.casehub.connectors;

import java.util.List;

public final class PaginationHelper {

    private PaginationHelper() {}

    public static <T> Page<T> paginate(List<T> all, PageRequest pagination) {
        int start = 0;
        if (pagination.cursor() != null) {
            start = Integer.parseInt(pagination.cursor());
        }
        int size = pagination.pageSize() > 0 ? pagination.pageSize() : 20;
        int end = Math.min(start + size, all.size());
        var items = all.subList(start, end);
        boolean hasMore = end < all.size();
        String nextCursor = hasMore ? String.valueOf(end) : null;
        return new Page<>(items, nextCursor, hasMore);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=PaginationHelperTest`
Expected: PASS

- [ ] **Step 5: Replace paginate() in all 4 ref modules**

Replace the `private static <T> Page<T> paginate(...)` method in each of these files with calls to `PaginationHelper.paginate()`:

- `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/RefCommercePlatform.java:156` — delete method, replace all `paginate(...)` calls with `PaginationHelper.paginate(...)`
- `location-ref/src/main/java/io/casehub/connectors/location/ref/RefLocationPlatform.java:101` — same
- `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/RefContactsPlatform.java:107` — same
- `project-ref/src/main/java/io/casehub/connectors/project/ref/RefProjectPlatform.java:198` — same

Add `import io.casehub.connectors.PaginationHelper;` to each file.

- [ ] **Step 6: Run full build to verify nothing broke**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/PaginationHelper.java connectors-api/src/test/java/io/casehub/connectors/PaginationHelperTest.java commerce-ref/ location-ref/ contacts-ref/ project-ref/
git commit -m "refactor(#141): extract paginate() to PaginationHelper in connectors-api"
```

---

## Batch 2: Normalise supports() pattern

### Task 2: Standardise supports() to Set.of() across all refs

**Files:**
- Modify: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/RefCommercePlatform.java:30-36`
- Modify: `location-ref/src/main/java/io/casehub/connectors/location/ref/RefLocationPlatform.java:24-29`
- Modify: `bank-ref/src/main/java/io/casehub/connectors/bank/ref/RefBankPlatform.java:31-34`
- Modify: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/RefContactsPlatform.java:27-31`
- Modify: `document-ref/src/main/java/io/casehub/connectors/document/ref/RefDocumentPlatform.java:36-41`
- Modify: `project-ref/src/main/java/io/casehub/connectors/project/ref/RefProjectPlatform.java:30-36`

**Interfaces:**
- Consumes: Each SPI's capability sub-interface classes
- Produces: No interface change — `supports()` signature unchanged

- [ ] **Step 1: Verify existing tests pass (baseline)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref,location-ref,bank-ref,contacts-ref,document-ref,project-ref`
Expected: PASS

- [ ] **Step 2: Replace chained == with Set.of() in all 6 refs**

In each file, add a `private static final Set<Class<?>> SUPPORTED = Set.of(...)` field and replace the `supports()` body with `return SUPPORTED.contains(capability)`.

**commerce-ref — RefCommercePlatform.java:**
```java
private static final Set<Class<?>> SUPPORTED = Set.of(
    ProductSearch.class, ProductDetails.class,
    Cart.class, Checkout.class, OrderTracking.class
);

@Override
public boolean supports(Class<?> capability) {
    return SUPPORTED.contains(capability);
}
```

**location-ref — RefLocationPlatform.java:**
```java
private static final Set<Class<?>> SUPPORTED = Set.of(
    PlaceSearch.class, PlaceDetails.class,
    Geocoding.class, Directions.class
);

@Override
public boolean supports(Class<?> capability) {
    return SUPPORTED.contains(capability);
}
```

**bank-ref — RefBankPlatform.java:**
```java
private static final Set<Class<?>> SUPPORTED = Set.of(
    AccountInformation.class, PaymentInitiation.class
);

@Override
public boolean supports(Class<?> capability) {
    return SUPPORTED.contains(capability);
}
```

**contacts-ref — RefContactsPlatform.java:**
```java
private static final Set<Class<?>> SUPPORTED = Set.of(
    ContactRead.class, GroupRead.class, ContactWrite.class
);

@Override
public boolean supports(Class<?> capability) {
    return SUPPORTED.contains(capability);
}
```

**document-ref — RefDocumentPlatform.java:**
```java
private static final Set<Class<?>> SUPPORTED = Set.of(
    FileOperations.class, FolderOperations.class,
    SearchOperations.class, SharingOperations.class
);

@Override
public boolean supports(Class<?> capability) {
    return SUPPORTED.contains(capability);
}
```

**project-ref — RefProjectPlatform.java:**
```java
private static final Set<Class<?>> SUPPORTED = Set.of(
    Issues.class, Labels.class, Milestones.class,
    Comments.class, Boards.class
);

@Override
public boolean supports(Class<?> capability) {
    return SUPPORTED.contains(capability);
}
```

Add `import java.util.Set;` to each file where missing.

Chat-ref already uses `ALL_CAPABILITIES.contains(capability)` — no change needed.

- [ ] **Step 3: Run tests to verify nothing broke**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref,location-ref,bank-ref,contacts-ref,document-ref,project-ref`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add commerce-ref/ location-ref/ bank-ref/ contacts-ref/ document-ref/ project-ref/
git commit -m "refactor(#141): normalise supports() to Set.of() pattern across 6 refs"
```

---

## Batch 3: Normalise CDI wiring and fix DocumentPlatform annotation

### Task 3: Migrate direct-construction backends to CDI-managed pattern

**Files:**
- Modify: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/InMemoryCommerceBackend.java` — add `@DefaultBean @ApplicationScoped`
- Modify: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/CommerceRefBeans.java` — inject backend instead of `new`
- Modify: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/InMemoryContactsBackend.java` — add `@DefaultBean @ApplicationScoped`
- Modify: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/ContactsRefBeans.java` — inject backend instead of `new`
- Modify: `location-ref/src/main/java/io/casehub/connectors/location/ref/InMemoryLocationBackend.java` — add `@DefaultBean @ApplicationScoped`
- Modify: `location-ref/src/main/java/io/casehub/connectors/location/ref/LocationRefBeans.java` — inject backend instead of `new`
- Rename: `project-ref/src/main/java/io/casehub/connectors/project/ref/ProjectBeans.java` → `ProjectRefBeans.java` (use `ide_refactor_rename`)

**Interfaces:**
- Consumes: `InMemory*Backend` classes, `*RefBeans` producer classes
- Produces: No interface change — same beans produced, different wiring

- [ ] **Step 1: Verify existing tests pass (baseline)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref,contacts-ref,location-ref,project-ref`
Expected: PASS

- [ ] **Step 2: Migrate InMemoryCommerceBackend**

Add annotations to `InMemoryCommerceBackend`:
```java
@DefaultBean
@ApplicationScoped
public class InMemoryCommerceBackend implements CommerceBackend {
```

Move `seed()` call from constructor to `@PostConstruct`:
```java
@PostConstruct
void init() {
    seed();
}
```

Update `CommerceRefBeans` to inject backend:
```java
public class CommerceRefBeans {

    @Produces
    @ApplicationScoped
    RefCommercePlatform commercePlatform(CommerceBackend backend) {
        return new RefCommercePlatform(backend);
    }
}
```

Add required imports: `jakarta.enterprise.inject.Default`, `io.quarkus.arc.DefaultBean`, `jakarta.enterprise.context.ApplicationScoped`, `jakarta.annotation.PostConstruct`.

- [ ] **Step 3: Run commerce-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref`
Expected: PASS

- [ ] **Step 4: Migrate InMemoryContactsBackend (same pattern as Step 2)**

Add `@DefaultBean @ApplicationScoped` to `InMemoryContactsBackend`. Move `seed()` to `@PostConstruct`. Update `ContactsRefBeans` to inject backend.

- [ ] **Step 5: Run contacts-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-ref`
Expected: PASS

- [ ] **Step 6: Migrate InMemoryLocationBackend (same pattern as Step 2)**

Add `@DefaultBean @ApplicationScoped` to `InMemoryLocationBackend`. Move `seed()` to `@PostConstruct`. Update `LocationRefBeans` to inject backend.

- [ ] **Step 7: Run location-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl location-ref`
Expected: PASS

- [ ] **Step 8: Rename ProjectBeans → ProjectRefBeans**

Use `ide_refactor_rename` to rename `ProjectRefBeans` to `ProjectRefBeans`. This updates all references.

- [ ] **Step 9: Run project-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl project-ref`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add commerce-ref/ contacts-ref/ location-ref/ project-ref/
git commit -m "refactor(#141): normalise CDI wiring — backends as @DefaultBean, rename ProjectBeans"
```

### Task 4: Fix DocumentPlatform @SimulationEligible capabilities

**Files:**
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java:14`

**Interfaces:**
- Consumes: `@SimulationEligible` annotation (from `simulation-api`)
- Produces: No runtime change — affects code generation only

- [ ] **Step 1: Update @SimulationEligible annotation**

Change line 14 from:
```java
@SimulationEligible(name = "document-platform")
```
to:
```java
@SimulationEligible(name = "document-platform",
    capabilities = {"fileOperations", "folderOperations", "searchOperations", "sharingOperations"})
```

- [ ] **Step 2: Run full build to verify annotation change compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add document-spi/
git commit -m "fix(#141): add capabilities attribute to DocumentPlatform @SimulationEligible"
```

---

## References

- [2026-10-04-unify-ref-simulation-design.md] — design spec this plan implements
- `connectors-api/src/main/java/io/casehub/connectors/Page.java:5` — Page record
- `connectors-api/src/main/java/io/casehub/connectors/PageRequest.java` — PageRequest record
- `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/RefCommercePlatform.java:156` — existing paginate() to extract
- `chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java:138` — reference supports() pattern (Set-based)
- `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java:14` — missing capabilities attribute
- `project-ref/src/main/java/io/casehub/connectors/project/ref/ProjectBeans.java:9` — naming inconsistency
- casehubio/connectors#141 — Phase 1 issue
- casehubio/connectors#138 — parent design issue
- Protocol: spi-id-method-naming — SPI identifier convention
