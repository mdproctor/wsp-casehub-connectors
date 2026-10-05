# Seed Data YAML & Simulation Corpus Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #143 — refactor: extract seed data to YAML and wire simulation corpus (Phase 2b)
**Issue group:** #143

**Goal:** Extract hardcoded seed data from 7 ref implementations into YAML files, wire seed loaders, and ship simulation corpus files for interpretive capabilities.

**Architecture:** Each ref module's `InMemory*Backend` currently hardcodes seed data in Java. This refactoring extracts that data to YAML files on the classpath, read by a per-module `SeedLoader` utility. The constructor still calls seed (per design decision — `@PostConstruct` breaks non-CDI test usage), but `seed()` now delegates to the YAML-backed loader. Simulation corpus files (InvocationRecord format) are shipped separately for interpretive capabilities. Two modules (chat-ref, calendar-ref) have no seed data and are excluded.

**Tech Stack:** Jackson `jackson-dataformat-yaml` + `jackson-datatype-jsr310` for YAML→record deserialization. All domain model types are Java records — Jackson handles these natively. Quarkus BOM manages Jackson versions.

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All domain records are immutable — YAML deserialization must use Jackson's record support, not setter-based
- Constructor must still call `seed()` — `@PostConstruct` breaks non-CDI test usage (design decision from #138)
- No new modules — seed loaders live in the same package as their backend
- YAML files go under `<module>/src/main/resources/seed/`
- Existing tests must pass unchanged — same data, different source
- Use `mvn` not `./mvnw`

---

## Batch 1: Commerce-ref exemplar

Establishes the pattern for all other modules. Commerce-ref is the most representative case (8 products with nested reviews, images, specs) and the E2E verification target for DataRealism.

### Task 1: Commerce-ref YAML seed extraction

**Files:**
- Create: `commerce-ref/src/main/resources/seed/products.yaml`
- Create: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/SeedLoader.java`
- Modify: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/InMemoryCommerceBackend.java`
- Modify: `commerce-ref/pom.xml`
- Test: `commerce-ref/src/test/java/io/casehub/connectors/commerce/ref/RefCommercePlatformTest.java` (existing — verify unchanged)
- Test: `commerce-ref/src/test/java/io/casehub/connectors/commerce/ref/SeedLoaderTest.java` (new)

**Interfaces:**
- Produces: `SeedLoader.loadProducts()` → `List<ProductDetail>` — used by `InMemoryCommerceBackend.seed()`

- [ ] **Step 1: Add jackson-dataformat-yaml and jackson-datatype-jsr310 dependencies to commerce-ref pom.xml**

Add after the existing `quarkus-arc` dependency:

```xml
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>
  <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
<dependency>
  <groupId>com.fasterxml.jackson.datatype</groupId>
  <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

No version needed — Quarkus BOM manages Jackson versions.

- [ ] **Step 2: Create products.yaml seed file**

Create `commerce-ref/src/main/resources/seed/products.yaml` containing the 8 products currently hardcoded in `InMemoryCommerceBackend.seed()`. Each entry maps to `ProductDetail` record fields:

```yaml
- id: "prod-1"
  name: "Sony WH-1000XM5 Headphones"
  brand: "Sony"
  sku: "prod-sony-xm5"
  category: "electronics"
  description: "Premium noise-cancelling wireless headphones with 30-hour battery life."
  price:
    amount: 299.99
    currency: "GBP"
  rating: 4.7
  reviewCount: 2350
  inStock: true
  stockQuantity: 45
  images:
    - url: "https://example.com/sony-xm5-1.jpg"
      altText: "Sony WH-1000XM5"
      width: 800
      height: 600
  reviews:
    - author: "Alice S."
      rating: 5.0
      text: "Best noise cancelling I've ever used"
      timestampMs: 1727900000000
    - author: "Bob J."
      rating: 4.0
      text: "Great sound, slightly tight fit"
      timestampMs: 1727800000000
  specifications:
    type: "Over-ear"
    connectivity: "Bluetooth 5.2"
    battery: "30 hours"
    weight: "250g"
  url: null
- id: "prod-2"
  name: "Kindle Paperwhite"
  # ... (all 8 products, matching existing seed data exactly)
```

Use explicit IDs (`prod-1` through `prod-8`) matching the auto-generated sequence from the current `addProduct` method.

- [ ] **Step 3: Write SeedLoader test**

Create `commerce-ref/src/test/java/io/casehub/connectors/commerce/ref/SeedLoaderTest.java`:

```java
package io.casehub.connectors.commerce.ref;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SeedLoaderTest {

    @Test
    void loadsAllProducts() {
        var products = SeedLoader.loadProducts();
        assertThat(products).hasSize(8);
    }

    @Test
    void firstProductHasCorrectFields() {
        var products = SeedLoader.loadProducts();
        var sony = products.stream()
            .filter(p -> p.id().equals("prod-1"))
            .findFirst().orElseThrow();
        assertThat(sony.name()).isEqualTo("Sony WH-1000XM5 Headphones");
        assertThat(sony.brand()).isEqualTo("Sony");
        assertThat(sony.category()).isEqualTo("electronics");
        assertThat(sony.price().amount()).isEqualByComparingTo("299.99");
        assertThat(sony.price().currency()).isEqualTo("GBP");
        assertThat(sony.reviews()).hasSize(2);
        assertThat(sony.specifications()).containsEntry("connectivity", "Bluetooth 5.2");
    }

    @Test
    void outOfStockProductLoadsCorrectly() {
        var products = SeedLoader.loadProducts();
        var marshall = products.stream()
            .filter(p -> p.name().contains("Marshall"))
            .findFirst().orElseThrow();
        assertThat(marshall.inStock()).isFalse();
        assertThat(marshall.stockQuantity()).isZero();
    }
}
```

- [ ] **Step 4: Run SeedLoader test — verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref -Dtest=SeedLoaderTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compilation error — `SeedLoader` class doesn't exist yet.

- [ ] **Step 5: Create SeedLoader class**

Create `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/SeedLoader.java`:

```java
package io.casehub.connectors.commerce.ref;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.connectors.commerce.model.ProductDetail;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.util.List;

final class SeedLoader {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory())
            .registerModule(new JavaTimeModule());

    static List<ProductDetail> loadProducts() {
        try (var is = SeedLoader.class.getResourceAsStream("/seed/products.yaml")) {
            if (is == null) throw new IllegalStateException("Missing /seed/products.yaml");
            return YAML.readValue(is, new TypeReference<>() {});
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private SeedLoader() {}
}
```

- [ ] **Step 6: Run SeedLoader test — verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref -Dtest=SeedLoaderTest
```

Expected: all 3 tests PASS.

- [ ] **Step 7: Modify InMemoryCommerceBackend to use SeedLoader**

Replace the `seed()` method body. Remove the 8 `addProduct(...)` calls. Replace with:

```java
private void seed() {
    SeedLoader.loadProducts().forEach(d -> {
        products.put(d.id(), new Product(d.id(), d.name(), d.brand(), d.category(),
                d.price(), d.rating(), d.reviewCount(), d.inStock(),
                d.images().isEmpty() ? null : d.images().getFirst().url()));
        details.put(d.id(), d);
        reviews.put(d.id(), new ArrayList<>(d.reviews()));
    });
    idSeq = products.size();
}
```

Remove the `addProduct` private method (no longer needed). Update `idSeq` field to non-final if needed.

- [ ] **Step 8: Run all commerce-ref tests — verify unchanged behaviour**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref
```

Expected: all tests PASS (RefCommercePlatformTest, CommerceIntegrationTest, SeedLoaderTest).

- [ ] **Step 9: Commit**

```bash
git add commerce-ref/pom.xml commerce-ref/src/main/resources/seed/ commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/SeedLoader.java commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/InMemoryCommerceBackend.java commerce-ref/src/test/java/io/casehub/connectors/commerce/ref/SeedLoaderTest.java
git commit -m "$(cat <<'EOF'
refactor(#143): extract commerce-ref seed data to YAML

Replace hardcoded product data in InMemoryCommerceBackend.seed() with
YAML-backed SeedLoader. Products, reviews, images, and specs are now
in commerce-ref/src/main/resources/seed/products.yaml.

Establishes the pattern for remaining 6 ref modules.

Refs #143
EOF
)"
```

---

## Batch 2: Remaining ref modules (contacts, document, email)

Same pattern as commerce-ref. Each module: add dependencies, create YAML, create SeedLoader, modify backend.

### Task 2: Contacts-ref seed extraction

**Files:**
- Create: `contacts-ref/src/main/resources/seed/contacts.yaml`
- Create: `contacts-ref/src/main/resources/seed/groups.yaml`
- Create: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/SeedLoader.java`
- Modify: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/InMemoryContactsBackend.java`
- Modify: `contacts-ref/pom.xml`
- Test: `contacts-ref/src/test/java/io/casehub/connectors/contacts/ref/RefContactsPlatformTest.java` (existing)
- Test: `contacts-ref/src/test/java/io/casehub/connectors/contacts/ref/SeedLoaderTest.java` (new)

**Interfaces:**
- Produces: `SeedLoader.loadContacts()` → `List<SeedLoader.ContactSeed>` — flattened contact entry with group membership
- Produces: `SeedLoader.loadGroups()` → `List<Group>`

The contacts YAML uses a flattened seed format because the `Contact` record uses `LabelledValue<String>` wrappers for emails/phones, and the current `addSeed` method takes raw strings and wraps them. A `ContactSeed` record bridges the gap:

```java
record ContactSeed(String displayName, String givenName, String familyName,
                   String email, String phone, String company, String title,
                   List<String> groups) {}
```

- [ ] **Step 1: Add jackson-dataformat-yaml and jackson-datatype-jsr310 dependencies to contacts-ref pom.xml**

Same pattern as commerce-ref (no version, BOM-managed).

- [ ] **Step 2: Create contacts.yaml and groups.yaml seed files**

`contacts-ref/src/main/resources/seed/groups.yaml`:

```yaml
- id: "g-mycontacts"
  name: "myContacts"
  groupType: "SYSTEM"
  memberCount: 0
- id: "g-starred"
  name: "Starred"
  groupType: "SYSTEM"
  memberCount: 0
- id: "g-work"
  name: "Work"
  groupType: "USER_CREATED"
  memberCount: 0
```

`contacts-ref/src/main/resources/seed/contacts.yaml`:

```yaml
- displayName: "Alice Smith"
  givenName: "Alice"
  familyName: "Smith"
  email: "alice@example.com"
  phone: "+1234567001"
  company: "Acme Corp"
  title: "Engineer"
  groups: ["g-mycontacts", "g-work"]
# ... (all 10 contacts matching existing seed data)
```

- [ ] **Step 3: Write SeedLoader test**

Test `loadGroups()` returns 3 groups and `loadContacts()` returns 10 contacts with correct field mapping.

- [ ] **Step 4: Run test — verify it fails**

- [ ] **Step 5: Create SeedLoader class**

Reads both YAML files. `loadGroups()` returns `List<Group>`. `loadContacts()` returns `List<ContactSeed>`.

- [ ] **Step 6: Run test — verify it passes**

- [ ] **Step 7: Modify InMemoryContactsBackend.seed() to use SeedLoader**

Replace hardcoded `addSeed(...)` calls. Loader provides ContactSeed records; `seed()` converts them to Contact objects with LabelledValue wrapping and populates maps and groups.

- [ ] **Step 8: Run all contacts-ref tests — verify unchanged behaviour**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-ref
```

- [ ] **Step 9: Commit**

### Task 3: Document-ref and email-ref seed extraction

Two modules in one task — both use `loadData()` pattern and have similar complexity.

**Files (document-ref):**
- Create: `document-ref/src/main/resources/seed/folders.yaml`
- Create: `document-ref/src/main/resources/seed/files.yaml`
- Create: `document-ref/src/main/java/io/casehub/connectors/document/ref/SeedLoader.java`
- Modify: `document-ref/src/main/java/io/casehub/connectors/document/ref/InMemoryDocumentBackend.java`
- Modify: `document-ref/pom.xml`
- Test: `document-ref/src/test/java/io/casehub/connectors/document/ref/SeedLoaderTest.java` (new)

**Files (email-ref):**
- Create: `email-ref/src/main/resources/seed/messages.yaml`
- Create: `email-ref/src/main/java/io/casehub/connectors/email/ref/SeedLoader.java`
- Modify: `email-ref/src/main/java/io/casehub/connectors/email/ref/InMemoryEmailBackend.java`
- Modify: `email-ref/pom.xml`
- Test: `email-ref/src/test/java/io/casehub/connectors/email/ref/SeedLoaderTest.java` (new)

**Interfaces:**
- document-ref `SeedLoader`: `loadFolders()` → `List<SeedLoader.FolderSeed>`, `loadFiles()` → `List<SeedLoader.FileSeed>`
- email-ref `SeedLoader`: `loadMessages()` → `List<SeedLoader.MailboxSeed>` (mailbox name + list of message entries)

Note for document-ref: file content is zero-filled byte arrays based on declared size — only metadata goes in YAML. The seed loader creates `byte[size]` at load time.

Note for email-ref: messages are organized by mailbox (inbox, sent, archive). The YAML uses a mailbox-grouped structure. Attachment content is placeholder bytes — only metadata in YAML.

- [ ] **Step 1: Add dependencies to both pom.xml files**
- [ ] **Step 2: Create document-ref YAML seed files (folders.yaml, files.yaml)**
- [ ] **Step 3: Write document-ref SeedLoader test**
- [ ] **Step 4: Run test — verify it fails**
- [ ] **Step 5: Create document-ref SeedLoader + modify InMemoryDocumentBackend**
- [ ] **Step 6: Run document-ref tests — verify passes**
- [ ] **Step 7: Create email-ref YAML seed file (messages.yaml)**
- [ ] **Step 8: Write email-ref SeedLoader test**
- [ ] **Step 9: Run test — verify it fails**
- [ ] **Step 10: Create email-ref SeedLoader + modify InMemoryEmailBackend**
- [ ] **Step 11: Run email-ref tests — verify passes**
- [ ] **Step 12: Commit both modules together**

```bash
git commit -m "refactor(#143): extract document-ref and email-ref seed data to YAML

Refs #143"
```

---

## Batch 3: Remaining ref modules (bank, location, project)

### Task 4: Bank-ref seed extraction

**Files:**
- Create: `bank-ref/src/main/resources/seed/accounts.yaml`
- Create: `bank-ref/src/main/resources/seed/balances.yaml`
- Create: `bank-ref/src/main/resources/seed/transactions.yaml`
- Create: `bank-ref/src/main/java/io/casehub/connectors/bank/ref/SeedLoader.java`
- Modify: `bank-ref/src/main/java/io/casehub/connectors/bank/ref/InMemoryBankBackend.java`
- Modify: `bank-ref/pom.xml`
- Test: `bank-ref/src/test/java/io/casehub/connectors/bank/ref/SeedLoaderTest.java` (new)

**Interfaces:**
- Produces: `SeedLoader.loadAccounts()` → `List<AccountInfo>`
- Produces: `SeedLoader.loadBalances()` → `Map<String, AccountBalance>` (keyed by account ID)
- Produces: `SeedLoader.loadTransactions()` → `List<Transaction>`

Bank-ref is unique: data is currently `private static final` fields, not a `seed()` method. The refactoring changes these to instance fields populated from YAML in the constructor. The `AccountBalance` record uses `Instant` — `JavaTimeModule` handles ISO-8601 format. The `Transaction` record uses `LocalDate` — same module handles it.

- [ ] **Step 1: Add dependencies to bank-ref pom.xml**
- [ ] **Step 2: Create bank-ref YAML seed files (accounts.yaml, balances.yaml, transactions.yaml)**

`accounts.yaml`:
```yaml
- id: "acc-100"
  name: "Current Account"
  type: "CURRENT"
  currency: "GBP"
# ...
```

`balances.yaml`:
```yaml
acc-100:
  accountId: "acc-100"
  available: 2450.00
  current: 2450.00
  currency: "GBP"
  asOf: "2026-09-25T00:00:00Z"
# ...
```

`transactions.yaml`:
```yaml
- id: "txn-001"
  accountId: "acc-100"
  amount: 3.50
  direction: "DEBIT"
  currency: "GBP"
  description: "Grocery purchase"
  merchantName: "Tesco Express"
  category: "Groceries"
  date: "2026-09-01"
  status: "BOOKED"
# ...
```

- [ ] **Step 3: Write SeedLoader test**
- [ ] **Step 4: Run test — verify it fails**
- [ ] **Step 5: Create SeedLoader + modify InMemoryBankBackend**

Change `private static final` fields to `private final` fields. Add constructor that calls `seed()`, which uses `SeedLoader`.

- [ ] **Step 6: Run all bank-ref tests — verify passes**
- [ ] **Step 7: Commit**

### Task 5: Location-ref seed extraction

**Files:**
- Create: `location-ref/src/main/resources/seed/places.yaml`
- Create: `location-ref/src/main/resources/seed/geocoding.yaml`
- Create: `location-ref/src/main/java/io/casehub/connectors/location/ref/SeedLoader.java`
- Modify: `location-ref/src/main/java/io/casehub/connectors/location/ref/InMemoryLocationBackend.java`
- Modify: `location-ref/pom.xml`
- Test: `location-ref/src/test/java/io/casehub/connectors/location/ref/SeedLoaderTest.java` (new)

**Interfaces:**
- Produces: `SeedLoader.loadPlaces()` → `List<SeedLoader.PlaceSeed>` — contains both `Place` summary and `PlaceDetail` fields
- Produces: `SeedLoader.loadGeocoding()` → `List<SeedLoader.GeocodingSeed>`

Location-ref is the most complex seed data — each place has coordinates, opening hours, reviews, photos, price level. The YAML format uses a unified `PlaceSeed` that the loader splits into `Place` + `PlaceDetail`. Geocoding entries are separate (address ↔ coordinates mappings).

- [ ] **Step 1: Add dependencies to location-ref pom.xml**
- [ ] **Step 2: Create YAML seed files (places.yaml, geocoding.yaml)**
- [ ] **Step 3: Write SeedLoader test**
- [ ] **Step 4: Run test — verify it fails**
- [ ] **Step 5: Create SeedLoader + modify InMemoryLocationBackend**
- [ ] **Step 6: Run all location-ref tests — verify passes**
- [ ] **Step 7: Commit**

### Task 6: Project-ref seed extraction

**Files:**
- Create: `project-ref/src/main/resources/seed/project-data.yaml`
- Create: `project-ref/src/main/java/io/casehub/connectors/project/ref/SeedLoader.java`
- Modify: `project-ref/src/main/java/io/casehub/connectors/project/ref/ProjectBackend.java`
- Modify: `project-ref/pom.xml`
- Test: `project-ref/src/test/java/io/casehub/connectors/project/ref/SeedLoaderTest.java` (new)

**Interfaces:**
- Produces: `SeedLoader.loadProjectData()` → `SeedLoader.ProjectSeed` — wraps labels, milestones, issues, comments, board

Project-ref is unique: `withTestData()` static factory uses the backend's public API (createLabel, createIssue, etc.) to build state. The refactored `withTestData()` reads YAML and calls the same API methods. Uses a single `project-data.yaml` with sections because the entities are cross-referencing (issues reference labels and milestones).

```yaml
repo:
  owner: "test-org"
  name: "test-repo"

labels:
  - name: "bug"
    color: "d73a4a"
    description: "Something isn't working"
  # ...

milestones:
  - title: "v1.0"
    description: "First release"
    state: "open"
  - title: "v0.9"
    description: "Beta release"
    state: "closed"

issues:
  - title: "Fix login bug"
    body: "Login fails on Safari"
    state: "open"
    labels: ["bug"]
    milestone: "v1.0"
  # ...

comments:
  - issueIndex: 0
    body: "Reproduced on Safari 17"
    author: "dev1"
  # ...

boards:
  - id: "board-1"
    name: "Project Board"
    columns:
      - id: "col-1"
        name: "To Do"
        position: 0
      # ...
```

- [ ] **Step 1: Add dependencies to project-ref pom.xml**
- [ ] **Step 2: Create project-data.yaml seed file**
- [ ] **Step 3: Write SeedLoader test**
- [ ] **Step 4: Run test — verify it fails**
- [ ] **Step 5: Create SeedLoader + modify ProjectBackend.withTestData()**
- [ ] **Step 6: Run all project-ref tests — verify passes**
- [ ] **Step 7: Commit**

---

## Batch 4: Full build verification + simulation corpus

### Task 7: Full build verification and simulation corpus files

**Files:**
- Create: `commerce-ref/src/main/resources/simulation/commerce/product-search-corpus.yaml`
- Create: `commerce-ref/src/main/resources/simulation/commerce/product-details-corpus.yaml`
- Create: `location-ref/src/main/resources/simulation/location/place-search-corpus.yaml`
- Create: `contacts-ref/src/main/resources/simulation/contacts/search-corpus.yaml`
- Create: `document-ref/src/main/resources/simulation/document/search-corpus.yaml`
- Create: `email-ref/src/main/resources/simulation/email/search-corpus.yaml`
- Create: `project-ref/src/main/resources/simulation/project/search-corpus.yaml`

**Interfaces:**
- Consumes: existing simulation corpus YAML format (InvocationRecord pattern from chat-spi, bank-spi, etc.)

Simulation corpus files follow the established InvocationRecord format seen in `chat-spi/src/main/resources/simulation/chat/`. Each file is keyed by SPI method and contains input/output pairs that simulation strategies can replay.

- [ ] **Step 1: Run full project build to verify all modules compile and test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS. All existing tests pass with YAML-sourced seed data.

- [ ] **Step 2: Create simulation corpus files for interpretive capabilities**

Follow the existing corpus pattern from `chat-spi`. Each corpus file maps SPI method → input/output pairs:

```yaml
commerce-platform.productSearch.search:
  - tenancy-id: default
    input:
      query: "headphones"
      pagination: { cursor: null, pageSize: 10 }
    output:
      items:
        - id: "prod-1"
          name: "Sony WH-1000XM5 Headphones"
          # ... (realistic search results)
      hasMore: false
      nextCursor: null
```

Create corpus files for all 7 interpretive capability areas:
1. CommercePlatform: ProductSearch, ProductDetails
2. LocationPlatform: PlaceSearch, PlaceDetails, Geocoding, Directions
3. ContactsPlatform: ContactRead.search
4. DocumentPlatform: SearchOperations
5. EmailPlatform: search
6. ProjectPlatform: Issues.search

- [ ] **Step 3: Commit simulation corpus files**

```bash
git commit -m "feat(#143): ship simulation corpus files for interpretive capabilities

InvocationRecord-format YAML for 7 interpretive capability areas
across commerce, location, contacts, document, email, and project
ref modules. Follows established corpus pattern from chat-spi.

Refs #143"
```

- [ ] **Step 4: Final full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS.

---

## References

- `docs/specs/issue-138-design-unify-ref-simulation/2026-10-04-unify-ref-simulation-design.md` — design spec
- `docs/specs/issue-138-design-unify-ref-simulation/decisions.md` — design decisions
- `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/InMemoryCommerceBackend.java` — exemplar backend
- `chat-spi/src/main/resources/simulation/chat/messages-corpus.yaml` — existing InvocationRecord format
- GitHub #143 — focal issue
- GitHub #138 — parent design issue
- GitHub #141 — Phase 1 (merged)
- casehubio/platform#512 — Phase 2a (landed)
