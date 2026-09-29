# ContactsPlatform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #125 — feat: ContactsPlatform SPI — identity sync from external providers
**Issue group:** #125

**Goal:** Add a ContactsPlatform SPI with capability sub-interfaces (ContactRead, GroupRead, ContactWrite), sync support, an in-memory reference implementation, a Google People API provider, and GraphQL/MCP integration.

**Architecture:** Capability sub-interface pattern (DocumentPlatform style) with user-scoped accessors (BankPlatform style). Sync types (SyncRequest/SyncResult/SyncTokenExpiredException) in connectors-api as cross-SPI primitives. Google provider uses per-user credential resolution via GoogleCredentialResolver CDI SPI. GraphQL API resolves userId from SecurityIdentity.

**Tech Stack:** Java 21 (Java 26 JVM), Quarkus 3.32.2, Google People API v1 (`google-api-services-people`), Maven multi-module

## Global Constraints

- Java source level: 21, JVM: 26. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All artifacts: `0.2-SNAPSHOT`
- Parent POM: `casehub-connectors-parent:0.2-SNAPSHOT`
- Jandex plugin required in every module POM (`jandex-maven-plugin:3.3.1`)
- SPI identifiers named `id()` per protocol PP-20260609-e3a2bd
- Pagination methods use `PageRequest pagination` parameter name (not `page`)
- Paginating methods return partial results + WARNING on failure per PP-20260610-83747b
- IntelliJ MCP for all code navigation and structural editing

---

## Batch 1: Sync types + contacts-spi

### Task 1: Add sync types to connectors-api

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/SyncRequest.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/SyncResult.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/SyncRequestTest.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/SyncResultTest.java`

**Interfaces:**
- Produces: `SyncRequest(String syncToken, int pageSize)` with `SyncRequest.initial(int pageSize)` factory
- Produces: `SyncResult<T>(List<T> items, List<String> deletedIds, String syncToken, boolean hasMore)`
- Produces: `SyncTokenExpiredException(String expiredToken)` with `expiredToken()` accessor

- [ ] **Step 1: Write failing tests for SyncRequest**

Create `connectors-api/src/test/java/io/casehub/connectors/SyncRequestTest.java`:

```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SyncRequestTest {

    @Test
    void initialRequestHasNullSyncToken() {
        var request = SyncRequest.initial(50);
        assertThat(request.syncToken()).isNull();
        assertThat(request.pageSize()).isEqualTo(50);
    }

    @Test
    void requestWithSyncToken() {
        var request = new SyncRequest("token-abc", 100);
        assertThat(request.syncToken()).isEqualTo("token-abc");
        assertThat(request.pageSize()).isEqualTo(100);
    }
}
```

- [ ] **Step 2: Write failing tests for SyncResult**

Create `connectors-api/src/test/java/io/casehub/connectors/SyncResultTest.java`:

```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class SyncResultTest {

    @Test
    void fullSyncResult() {
        var result = new SyncResult<>(
            List.of("a", "b"),
            List.of("deleted-1"),
            "next-token",
            false
        );
        assertThat(result.items()).containsExactly("a", "b");
        assertThat(result.deletedIds()).containsExactly("deleted-1");
        assertThat(result.syncToken()).isEqualTo("next-token");
        assertThat(result.hasMore()).isFalse();
    }

    @Test
    void emptySyncResult() {
        var result = new SyncResult<>(List.of(), List.of(), "token", true);
        assertThat(result.items()).isEmpty();
        assertThat(result.deletedIds()).isEmpty();
        assertThat(result.hasMore()).isTrue();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest="SyncRequestTest,SyncResultTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — classes do not exist

- [ ] **Step 4: Implement SyncRequest**

Create `connectors-api/src/main/java/io/casehub/connectors/SyncRequest.java`:

```java
package io.casehub.connectors;

public record SyncRequest(String syncToken, int pageSize) {
    public static SyncRequest initial(int pageSize) {
        return new SyncRequest(null, pageSize);
    }
}
```

- [ ] **Step 5: Implement SyncResult**

Create `connectors-api/src/main/java/io/casehub/connectors/SyncResult.java`:

```java
package io.casehub.connectors;

import java.util.List;

public record SyncResult<T>(
    List<T> items,
    List<String> deletedIds,
    String syncToken,
    boolean hasMore
) {}
```

- [ ] **Step 6: Implement SyncTokenExpiredException**

Create `connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java`:

```java
package io.casehub.connectors;

public class SyncTokenExpiredException extends RuntimeException {
    private final String expiredToken;

    public SyncTokenExpiredException(String expiredToken) {
        super("Sync token expired — full resync required");
        this.expiredToken = expiredToken;
    }

    public String expiredToken() {
        return expiredToken;
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest="SyncRequestTest,SyncResultTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/SyncRequest.java connectors-api/src/main/java/io/casehub/connectors/SyncResult.java connectors-api/src/main/java/io/casehub/connectors/SyncTokenExpiredException.java connectors-api/src/test/java/io/casehub/connectors/SyncRequestTest.java connectors-api/src/test/java/io/casehub/connectors/SyncResultTest.java
git commit -m "feat(#125): add SyncRequest, SyncResult, SyncTokenExpiredException to connectors-api"
```

---

### Task 2: Create contacts-spi module

**Files:**
- Create: `contacts-spi/pom.xml`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/Contact.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/ContactName.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/Address.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/LabelledValue.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/Group.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/GroupType.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatformService.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsBeans.java`
- Create: `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/NoOpContactsPlatform.java`
- Modify: `pom.xml` (parent — add module)
- Test: `contacts-spi/src/test/java/io/casehub/connectors/contacts/spi/ContactsPlatformServiceTest.java`

**Interfaces:**
- Consumes: `Page<T>`, `PageRequest` from `connectors-api`; `SyncRequest`, `SyncResult` from Task 1
- Produces: `ContactsPlatform` interface with `contactRead(userId)`, `groupRead(userId)`, `contactWrite(userId)`, `supports(Class<?>)`, `id()`
- Produces: `ContactsPlatformService` registry with `platform(id)`, `supports(id)`, `ids()`
- Produces: Model records: `Contact`, `ContactName`, `Address`, `LabelledValue<T>`, `Group`, `GroupType`

- [ ] **Step 1: Create contacts-spi/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-connectors-contacts-spi</artifactId>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-simulation-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add contacts-spi module to parent pom.xml**

Add `<module>contacts-spi</module>` after the `document-google` module entry (line 51) in the parent `pom.xml`.

- [ ] **Step 3: Create model records**

Create all model records in `contacts-spi/src/main/java/io/casehub/connectors/contacts/model/`:

**GroupType.java:**
```java
package io.casehub.connectors.contacts.model;

public enum GroupType {
    SYSTEM,
    USER_CREATED
}
```

**LabelledValue.java:**
```java
package io.casehub.connectors.contacts.model;

public record LabelledValue<T>(String label, T value, boolean primary) {}
```

**ContactName.java:**
```java
package io.casehub.connectors.contacts.model;

public record ContactName(String displayName, String givenName, String familyName) {}
```

**Address.java:**
```java
package io.casehub.connectors.contacts.model;

public record Address(String street, String city, String region, String postalCode, String country) {}
```

**Group.java:**
```java
package io.casehub.connectors.contacts.model;

public record Group(String id, String name, GroupType groupType, int memberCount) {}
```

**Contact.java:**
```java
package io.casehub.connectors.contacts.model;

import java.util.List;
import java.util.Map;

public record Contact(
    String id,
    ContactName name,
    List<LabelledValue<String>> emails,
    List<LabelledValue<String>> phones,
    List<LabelledValue<Address>> addresses,
    String company,
    String jobTitle,
    String photoUrl,
    String notes,
    Map<String, String> metadata
) {}
```

- [ ] **Step 4: Create ContactsPlatform SPI interface**

Create `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java`:

```java
package io.casehub.connectors.contacts.spi;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.contacts.model.Contact;
import io.casehub.connectors.contacts.model.Group;
import io.casehub.platform.simulation.SimulationEligible;

import java.util.List;

@SimulationEligible(name = "contacts-platform",
    capabilities = {"contactRead", "groupRead", "contactWrite"})
public interface ContactsPlatform {

    String id();

    boolean supports(Class<?> capability);

    ContactRead contactRead(String userId);

    GroupRead groupRead(String userId);

    ContactWrite contactWrite(String userId);

    interface ContactRead {
        Page<Contact> list(PageRequest pagination);
        SyncResult<Contact> listSync(SyncRequest request);
        Contact get(String contactId);
        Page<Contact> search(String query, PageRequest pagination);
    }

    interface GroupRead {
        List<Group> list();
        Page<Contact> listContacts(String groupId, PageRequest pagination);
    }

    interface ContactWrite {
        Contact create(Contact contact);
        Contact update(String contactId, Contact contact);
        void delete(String contactId);
    }
}
```

- [ ] **Step 5: Create ContactsPlatformService**

Create `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatformService.java`:

Follow the pattern from `DocumentPlatformService`. Registry with `platform(id)`, `supports(id)`, `ids()`.

```java
package io.casehub.connectors.contacts.spi;

import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class ContactsPlatformService {

    private final Map<String, ContactsPlatform> platforms;

    public ContactsPlatformService(java.util.List<ContactsPlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(ContactsPlatform::id, p -> p));
    }

    public ContactsPlatform platform(String id) {
        var platform = platforms.get(id);
        if (platform == null) {
            throw new IllegalArgumentException("No contacts platform: " + id);
        }
        return platform;
    }

    public boolean supports(String id) {
        return platforms.containsKey(id);
    }

    public Set<String> ids() {
        return platforms.keySet();
    }
}
```

- [ ] **Step 6: Create ContactsBeans CDI producer**

Create `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsBeans.java`:

```java
package io.casehub.connectors.contacts.spi;

import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

public class ContactsBeans {

    @Produces
    @ApplicationScoped
    ContactsPlatformService contactsPlatformService(@All List<ContactsPlatform> platforms) {
        return new ContactsPlatformService(platforms);
    }
}
```

- [ ] **Step 7: Create NoOpContactsPlatform**

Create `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/NoOpContactsPlatform.java`:

Follow the NoOp pattern — `@DefaultBean`, returns "none" id, all capabilities throw `UnsupportedCapabilityException`.

```java
package io.casehub.connectors.contacts.spi;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.contacts.model.Contact;
import io.casehub.connectors.contacts.model.Group;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;

@DefaultBean
@ApplicationScoped
public class NoOpContactsPlatform implements ContactsPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return false;
    }

    @Override
    public ContactRead contactRead(String userId) {
        return NoOpContactRead.INSTANCE;
    }

    @Override
    public GroupRead groupRead(String userId) {
        return NoOpGroupRead.INSTANCE;
    }

    @Override
    public ContactWrite contactWrite(String userId) {
        return NoOpContactWrite.INSTANCE;
    }

    private enum NoOpContactRead implements ContactRead {
        INSTANCE;

        @Override
        public Page<Contact> list(PageRequest pagination) {
            throw new UnsupportedCapabilityException("list", "ContactRead", "none", List.of());
        }

        @Override
        public SyncResult<Contact> listSync(SyncRequest request) {
            throw new UnsupportedCapabilityException("listSync", "ContactRead", "none", List.of());
        }

        @Override
        public Contact get(String contactId) {
            throw new UnsupportedCapabilityException("get", "ContactRead", "none", List.of());
        }

        @Override
        public Page<Contact> search(String query, PageRequest pagination) {
            throw new UnsupportedCapabilityException("search", "ContactRead", "none", List.of());
        }
    }

    private enum NoOpGroupRead implements GroupRead {
        INSTANCE;

        @Override
        public List<Group> list() {
            throw new UnsupportedCapabilityException("list", "GroupRead", "none", List.of());
        }

        @Override
        public Page<Contact> listContacts(String groupId, PageRequest pagination) {
            throw new UnsupportedCapabilityException("listContacts", "GroupRead", "none", List.of());
        }
    }

    private enum NoOpContactWrite implements ContactWrite {
        INSTANCE;

        @Override
        public Contact create(Contact contact) {
            throw new UnsupportedCapabilityException("create", "ContactWrite", "none", List.of());
        }

        @Override
        public Contact update(String contactId, Contact contact) {
            throw new UnsupportedCapabilityException("update", "ContactWrite", "none", List.of());
        }

        @Override
        public void delete(String contactId) {
            throw new UnsupportedCapabilityException("delete", "ContactWrite", "none", List.of());
        }
    }
}
```

- [ ] **Step 8: Write ContactsPlatformService test**

Create `contacts-spi/src/test/java/io/casehub/connectors/contacts/spi/ContactsPlatformServiceTest.java`:

```java
package io.casehub.connectors.contacts.spi;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ContactsPlatformServiceTest {

    @Test
    void registersAndRetrieves() {
        var noop = new NoOpContactsPlatform();
        var service = new ContactsPlatformService(List.of(noop));
        assertThat(service.platform("none")).isSameAs(noop);
        assertThat(service.supports("none")).isTrue();
        assertThat(service.ids()).containsExactly("none");
    }

    @Test
    void throwsForUnknownPlatform() {
        var service = new ContactsPlatformService(List.of());
        assertThatThrownBy(() -> service.platform("missing"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("No contacts platform");
    }

    @Test
    void noOpPlatformReportsNoCapabilities() {
        var noop = new NoOpContactsPlatform();
        assertThat(noop.id()).isEqualTo("none");
        assertThat(noop.supports(ContactsPlatform.ContactRead.class)).isFalse();
        assertThat(noop.supports(ContactsPlatform.GroupRead.class)).isFalse();
        assertThat(noop.supports(ContactsPlatform.ContactWrite.class)).isFalse();
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-spi -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add contacts-spi/ pom.xml
git commit -m "feat(#125): add contacts-spi module — ContactsPlatform SPI with capability sub-interfaces"
```

---

## Batch 2: Reference implementation — contacts-ref

### Task 3: Create contacts-ref module

**Files:**
- Create: `contacts-ref/pom.xml`
- Create: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/ContactsBackend.java`
- Create: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/InMemoryContactsBackend.java`
- Create: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/RefContactsPlatform.java`
- Create: `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/ContactsRefBeans.java`
- Modify: `pom.xml` (parent — add module)
- Test: `contacts-ref/src/test/java/io/casehub/connectors/contacts/ref/RefContactsPlatformTest.java`

**Interfaces:**
- Consumes: `ContactsPlatform`, `ContactsPlatform.ContactRead`, `ContactsPlatform.GroupRead`, `ContactsPlatform.ContactWrite` from Task 2; `Page<T>`, `PageRequest`, `SyncRequest`, `SyncResult` from connectors-api
- Produces: `RefContactsPlatform` implementing `ContactsPlatform` with `id()` = `"ref"`; `ContactsBackend` interface; `InMemoryContactsBackend` with pre-loaded test data and sync tracking

- [ ] **Step 1: Create contacts-ref/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-connectors-contacts-ref</artifactId>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-contacts-spi</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add contacts-ref module to parent pom.xml**

Add `<module>contacts-ref</module>` after `contacts-spi` in the parent `pom.xml`.

- [ ] **Step 3: Write failing test for RefContactsPlatform**

Create `contacts-ref/src/test/java/io/casehub/connectors/contacts/ref/RefContactsPlatformTest.java`:

```java
package io.casehub.connectors.contacts.ref;

import io.casehub.connectors.PageRequest;
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.contacts.model.GroupType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RefContactsPlatformTest {

    private RefContactsPlatform platform;

    @BeforeEach
    void setUp() {
        platform = new RefContactsPlatform(new InMemoryContactsBackend());
    }

    @Test
    void id() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsAllCapabilities() {
        assertThat(platform.supports(io.casehub.connectors.contacts.spi.ContactsPlatform.ContactRead.class)).isTrue();
        assertThat(platform.supports(io.casehub.connectors.contacts.spi.ContactsPlatform.GroupRead.class)).isTrue();
        assertThat(platform.supports(io.casehub.connectors.contacts.spi.ContactsPlatform.ContactWrite.class)).isTrue();
    }

    @Test
    void listContactsReturnsPaginatedResults() {
        var page = platform.contactRead("user1").list(new PageRequest(null, 5));
        assertThat(page.items()).isNotEmpty();
        assertThat(page.items().size()).isLessThanOrEqualTo(5);
    }

    @Test
    void getContactById() {
        var contacts = platform.contactRead("user1").list(new PageRequest(null, 10));
        var first = contacts.items().getFirst();
        var retrieved = platform.contactRead("user1").get(first.id());
        assertThat(retrieved).isEqualTo(first);
    }

    @Test
    void searchContactsByName() {
        var results = platform.contactRead("user1").search("Smith", new PageRequest(null, 10));
        assertThat(results.items()).allSatisfy(c ->
            assertThat(c.name().displayName().toLowerCase()).contains("smith")
        );
    }

    @Test
    void listGroups() {
        var groups = platform.groupRead("user1").list();
        assertThat(groups).isNotEmpty();
        assertThat(groups).anyMatch(g -> g.groupType() == GroupType.SYSTEM);
        assertThat(groups).anyMatch(g -> g.groupType() == GroupType.USER_CREATED);
    }

    @Test
    void listContactsInGroup() {
        var groups = platform.groupRead("user1").list();
        var group = groups.getFirst();
        var page = platform.groupRead("user1").listContacts(group.id(), new PageRequest(null, 10));
        assertThat(page.items()).isNotEmpty();
    }

    @Test
    void fullSyncThenIncrementalSync() {
        var fullSync = platform.contactRead("user1").listSync(SyncRequest.initial(100));
        assertThat(fullSync.items()).isNotEmpty();
        assertThat(fullSync.syncToken()).isNotNull();
        assertThat(fullSync.deletedIds()).isEmpty();

        var incrementalSync = platform.contactRead("user1").listSync(
            new SyncRequest(fullSync.syncToken(), 100));
        assertThat(incrementalSync.items()).isEmpty();
        assertThat(incrementalSync.deletedIds()).isEmpty();
    }

    @Test
    void syncDetectsChangesAfterCreate() {
        var fullSync = platform.contactRead("user1").listSync(SyncRequest.initial(100));
        var token = fullSync.syncToken();

        var created = platform.contactWrite("user1").create(
            new io.casehub.connectors.contacts.model.Contact(
                null, new io.casehub.connectors.contacts.model.ContactName("New Person", "New", "Person"),
                java.util.List.of(), java.util.List.of(), java.util.List.of(),
                null, null, null, null, java.util.Map.of()));

        var incrementalSync = platform.contactRead("user1").listSync(new SyncRequest(token, 100));
        assertThat(incrementalSync.items()).hasSize(1);
        assertThat(incrementalSync.items().getFirst().id()).isEqualTo(created.id());
    }

    @Test
    void syncDetectsDeletes() {
        var contacts = platform.contactRead("user1").list(new PageRequest(null, 10));
        var toDelete = contacts.items().getFirst();

        var fullSync = platform.contactRead("user1").listSync(SyncRequest.initial(100));
        var token = fullSync.syncToken();

        platform.contactWrite("user1").delete(toDelete.id());

        var incrementalSync = platform.contactRead("user1").listSync(new SyncRequest(token, 100));
        assertThat(incrementalSync.deletedIds()).contains(toDelete.id());
    }

    @Test
    void createUpdateDeleteContact() {
        var contact = new io.casehub.connectors.contacts.model.Contact(
            null, new io.casehub.connectors.contacts.model.ContactName("Test User", "Test", "User"),
            java.util.List.of(), java.util.List.of(), java.util.List.of(),
            "TestCo", "Engineer", null, null, java.util.Map.of());

        var created = platform.contactWrite("user1").create(contact);
        assertThat(created.id()).isNotNull();
        assertThat(created.name().displayName()).isEqualTo("Test User");

        var updated = platform.contactWrite("user1").update(created.id(),
            new io.casehub.connectors.contacts.model.Contact(
                created.id(), new io.casehub.connectors.contacts.model.ContactName("Updated User", "Updated", "User"),
                java.util.List.of(), java.util.List.of(), java.util.List.of(),
                "TestCo", "Senior Engineer", null, null, java.util.Map.of()));
        assertThat(updated.name().displayName()).isEqualTo("Updated User");

        platform.contactWrite("user1").delete(created.id());
        assertThatThrownBy(() -> platform.contactRead("user1").get(created.id()));
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-ref -Dtest="RefContactsPlatformTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — classes do not exist

- [ ] **Step 5: Implement ContactsBackend interface**

Create `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/ContactsBackend.java`:

```java
package io.casehub.connectors.contacts.ref;

import io.casehub.connectors.contacts.model.Contact;
import io.casehub.connectors.contacts.model.Group;

import java.util.List;

interface ContactsBackend {
    List<Contact> allContacts();
    Contact contact(String id);
    List<Contact> search(String query);
    List<Group> allGroups();
    List<Contact> contactsInGroup(String groupId);
    Contact create(Contact contact);
    Contact update(String id, Contact contact);
    void delete(String id);
    long currentVersion();
    List<Contact> changedSince(long version);
    List<String> deletedSince(long version);
}
```

- [ ] **Step 6: Implement InMemoryContactsBackend**

Create `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/InMemoryContactsBackend.java`:

Thread-safe in-memory backend with:
- Pre-loaded test data: ~10 contacts across 3 groups (myContacts/SYSTEM, work/USER_CREATED, starred/SYSTEM)
- Monotonic version counter for sync tracking
- Version-stamped entries: each contact tracks its create/update version
- Deletion tracking: deleted IDs with version stamps
- `changedSince(version)` returns contacts with version > given version
- `deletedSince(version)` returns IDs deleted after given version

```java
package io.casehub.connectors.contacts.ref;

import io.casehub.connectors.contacts.model.*;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

class InMemoryContactsBackend implements ContactsBackend {

    private final Map<String, VersionedContact> contacts = new ConcurrentHashMap<>();
    private final Map<String, Group> groups = new ConcurrentHashMap<>();
    private final Map<String, Set<String>> groupMembers = new ConcurrentHashMap<>();
    private final Map<String, Long> deletedVersions = new ConcurrentHashMap<>();
    private final AtomicLong version = new AtomicLong(0);
    private final AtomicLong idSeq = new AtomicLong(0);

    InMemoryContactsBackend() {
        seed();
    }

    private void seed() {
        groups.put("g-mycontacts", new Group("g-mycontacts", "myContacts", GroupType.SYSTEM, 0));
        groups.put("g-starred", new Group("g-starred", "Starred", GroupType.SYSTEM, 0));
        groups.put("g-work", new Group("g-work", "Work", GroupType.USER_CREATED, 0));
        groupMembers.put("g-mycontacts", ConcurrentHashMap.newKeySet());
        groupMembers.put("g-starred", ConcurrentHashMap.newKeySet());
        groupMembers.put("g-work", ConcurrentHashMap.newKeySet());

        addSeed("Alice Smith", "Alice", "Smith", "alice@example.com", "+1234567001", "Acme Corp", "Engineer", "g-mycontacts", "g-work");
        addSeed("Bob Jones", "Bob", "Jones", "bob@example.com", "+1234567002", "Acme Corp", "Manager", "g-mycontacts", "g-work");
        addSeed("Carol Smith", "Carol", "Smith", "carol@example.com", "+1234567003", "Beta Inc", "Designer", "g-mycontacts", "g-starred");
        addSeed("Dave Wilson", "Dave", "Wilson", "dave@example.com", "+1234567004", "Beta Inc", "CEO", "g-mycontacts");
        addSeed("Eve Brown", "Eve", "Brown", "eve@example.com", "+1234567005", "Gamma LLC", "CTO", "g-mycontacts", "g-work");
        addSeed("Frank Lee", "Frank", "Lee", "frank@example.com", "+1234567006", "Gamma LLC", "Intern", "g-mycontacts");
        addSeed("Grace Chen", "Grace", "Chen", "grace@example.com", "+1234567007", "Delta Co", "VP", "g-mycontacts", "g-starred");
        addSeed("Hank Miller", "Hank", "Miller", "hank@example.com", "+1234567008", "Acme Corp", "Analyst", "g-mycontacts", "g-work");
        addSeed("Ivy Davis", "Ivy", "Davis", "ivy@example.com", "+1234567009", "Beta Inc", "PM", "g-mycontacts");
        addSeed("Jack Taylor", "Jack", "Taylor", "jack@example.com", "+1234567010", "Delta Co", "Director", "g-mycontacts", "g-starred");

        updateGroupCounts();
    }

    private void addSeed(String display, String given, String family, String email, String phone,
                         String company, String title, String... memberOfGroups) {
        var id = "c-" + idSeq.incrementAndGet();
        var contact = new Contact(id,
            new ContactName(display, given, family),
            List.of(new LabelledValue<>("work", email, true)),
            List.of(new LabelledValue<>("mobile", phone, true)),
            List.of(),
            company, title, null, null, Map.of());
        contacts.put(id, new VersionedContact(contact, version.incrementAndGet()));
        for (var g : memberOfGroups) {
            groupMembers.get(g).add(id);
        }
    }

    private void updateGroupCounts() {
        groupMembers.forEach((gId, members) -> {
            var g = groups.get(gId);
            groups.put(gId, new Group(g.id(), g.name(), g.groupType(), members.size()));
        });
    }

    @Override
    public List<Contact> allContacts() {
        return contacts.values().stream().map(VersionedContact::contact).toList();
    }

    @Override
    public Contact contact(String id) {
        var vc = contacts.get(id);
        if (vc == null) throw new NoSuchElementException("Contact not found: " + id);
        return vc.contact();
    }

    @Override
    public List<Contact> search(String query) {
        var q = query.toLowerCase();
        return contacts.values().stream()
            .map(VersionedContact::contact)
            .filter(c -> matchesQuery(c, q))
            .toList();
    }

    private boolean matchesQuery(Contact c, String query) {
        if (c.name() != null && c.name().displayName() != null
                && c.name().displayName().toLowerCase().contains(query)) return true;
        if (c.company() != null && c.company().toLowerCase().contains(query)) return true;
        return c.emails().stream().anyMatch(e -> e.value().toLowerCase().contains(query));
    }

    @Override
    public List<Group> allGroups() {
        return List.copyOf(groups.values());
    }

    @Override
    public List<Contact> contactsInGroup(String groupId) {
        var members = groupMembers.get(groupId);
        if (members == null) throw new NoSuchElementException("Group not found: " + groupId);
        return members.stream().map(id -> contacts.get(id).contact()).toList();
    }

    @Override
    public Contact create(Contact contact) {
        var id = "c-" + idSeq.incrementAndGet();
        var created = new Contact(id, contact.name(), contact.emails(), contact.phones(),
            contact.addresses(), contact.company(), contact.jobTitle(),
            contact.photoUrl(), contact.notes(), contact.metadata());
        contacts.put(id, new VersionedContact(created, version.incrementAndGet()));
        groupMembers.get("g-mycontacts").add(id);
        updateGroupCounts();
        return created;
    }

    @Override
    public Contact update(String id, Contact contact) {
        if (!contacts.containsKey(id)) throw new NoSuchElementException("Contact not found: " + id);
        var updated = new Contact(id, contact.name(), contact.emails(), contact.phones(),
            contact.addresses(), contact.company(), contact.jobTitle(),
            contact.photoUrl(), contact.notes(), contact.metadata());
        contacts.put(id, new VersionedContact(updated, version.incrementAndGet()));
        return updated;
    }

    @Override
    public void delete(String id) {
        if (contacts.remove(id) == null) throw new NoSuchElementException("Contact not found: " + id);
        deletedVersions.put(id, version.incrementAndGet());
        groupMembers.values().forEach(members -> members.remove(id));
        updateGroupCounts();
    }

    @Override
    public long currentVersion() {
        return version.get();
    }

    @Override
    public List<Contact> changedSince(long sinceVersion) {
        return contacts.values().stream()
            .filter(vc -> vc.version() > sinceVersion)
            .map(VersionedContact::contact)
            .toList();
    }

    @Override
    public List<String> deletedSince(long sinceVersion) {
        return deletedVersions.entrySet().stream()
            .filter(e -> e.getValue() > sinceVersion)
            .map(Map.Entry::getKey)
            .toList();
    }

    private record VersionedContact(Contact contact, long version) {}
}
```

- [ ] **Step 7: Implement RefContactsPlatform**

Create `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/RefContactsPlatform.java`:

```java
package io.casehub.connectors.contacts.ref;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.contacts.model.Contact;
import io.casehub.connectors.contacts.model.Group;
import io.casehub.connectors.contacts.spi.ContactsPlatform;

import java.util.List;

public class RefContactsPlatform implements ContactsPlatform {

    private final ContactsBackend backend;

    public RefContactsPlatform(ContactsBackend backend) {
        this.backend = backend;
    }

    @Override
    public String id() {
        return "ref";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == ContactRead.class
            || capability == GroupRead.class
            || capability == ContactWrite.class;
    }

    @Override
    public ContactRead contactRead(String userId) {
        return new RefContactRead();
    }

    @Override
    public GroupRead groupRead(String userId) {
        return new RefGroupRead();
    }

    @Override
    public ContactWrite contactWrite(String userId) {
        return new RefContactWrite();
    }

    private class RefContactRead implements ContactRead {
        @Override
        public Page<Contact> list(PageRequest pagination) {
            return paginate(backend.allContacts(), pagination);
        }

        @Override
        public SyncResult<Contact> listSync(SyncRequest request) {
            long sinceVersion = request.syncToken() != null
                ? Long.parseLong(request.syncToken()) : 0;
            var changed = backend.changedSince(sinceVersion);
            var deleted = backend.deletedSince(sinceVersion);
            return new SyncResult<>(changed, deleted,
                String.valueOf(backend.currentVersion()), false);
        }

        @Override
        public Contact get(String contactId) {
            return backend.contact(contactId);
        }

        @Override
        public Page<Contact> search(String query, PageRequest pagination) {
            return paginate(backend.search(query), pagination);
        }
    }

    private class RefGroupRead implements GroupRead {
        @Override
        public List<Group> list() {
            return backend.allGroups();
        }

        @Override
        public Page<Contact> listContacts(String groupId, PageRequest pagination) {
            return paginate(backend.contactsInGroup(groupId), pagination);
        }
    }

    private class RefContactWrite implements ContactWrite {
        @Override
        public Contact create(Contact contact) {
            return backend.create(contact);
        }

        @Override
        public Contact update(String contactId, Contact contact) {
            return backend.update(contactId, contact);
        }

        @Override
        public void delete(String contactId) {
            backend.delete(contactId);
        }
    }

    private static <T> Page<T> paginate(List<T> all, PageRequest pagination) {
        int start = 0;
        if (pagination.cursor() != null) {
            start = Integer.parseInt(pagination.cursor());
        }
        int size = pagination.size() > 0 ? pagination.size() : 20;
        int end = Math.min(start + size, all.size());
        var items = all.subList(start, end);
        boolean hasMore = end < all.size();
        String nextCursor = hasMore ? String.valueOf(end) : null;
        return new Page<>(items, nextCursor, hasMore);
    }
}
```

- [ ] **Step 8: Create ContactsRefBeans CDI producer**

Create `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/ContactsRefBeans.java`:

```java
package io.casehub.connectors.contacts.ref;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

public class ContactsRefBeans {

    @Produces
    @ApplicationScoped
    RefContactsPlatform refContactsPlatform() {
        return new RefContactsPlatform(new InMemoryContactsBackend());
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-ref -Dtest="RefContactsPlatformTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add contacts-ref/ pom.xml
git commit -m "feat(#125): add contacts-ref module — in-memory reference ContactsPlatform with sync tracking"
```

---

## Batch 3: Google provider — contacts-google

### Task 4: Create contacts-google module

**Files:**
- Create: `contacts-google/pom.xml`
- Create: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleCredentialResolver.java`
- Create: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleOAuthConfig.java`
- Create: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/ConfigGoogleCredentialResolver.java`
- Create: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleContactsPlatform.java`
- Create: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/ContactsGoogleBeans.java`
- Modify: `pom.xml` (parent — add module)
- Test: `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactsPlatformTest.java`

**Interfaces:**
- Consumes: `ContactsPlatform`, model records from Task 2; `SyncRequest`, `SyncResult`, `SyncTokenExpiredException` from Task 1
- Produces: `GoogleCredentialResolver` CDI SPI with `resolve(String userId)` returning `GoogleOAuthConfig`; `GoogleContactsPlatform` implementing `ContactsPlatform` with `id()` = `"google"`

- [ ] **Step 1: Create contacts-google/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-connectors-contacts-google</artifactId>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-contacts-spi</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.apis</groupId>
            <artifactId>google-api-services-people</artifactId>
            <version>v1-rev20260917-2.0.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.auth</groupId>
            <artifactId>google-auth-library-oauth2-http</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.api-client</groupId>
            <artifactId>google-api-client</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.http-client</groupId>
            <artifactId>google-http-client-gson</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.wiremock</groupId>
            <artifactId>wiremock-standalone</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Note: check exact `google-api-services-people` version available. The version above may need adjustment — look at what `google-api-services-calendar` and `google-api-services-gmail` use for the `google-api-client` BOM version and match it.

- [ ] **Step 2: Add contacts-google module to parent pom.xml**

Add `<module>contacts-google</module>` after `contacts-ref` in the parent `pom.xml`.

- [ ] **Step 3: Create GoogleCredentialResolver SPI and GoogleOAuthConfig**

Create `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleOAuthConfig.java`:

```java
package io.casehub.connectors.contacts.google;

public record GoogleOAuthConfig(String refreshToken, String clientId, String clientSecret) {}
```

Create `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleCredentialResolver.java`:

```java
package io.casehub.connectors.contacts.google;

public interface GoogleCredentialResolver {
    GoogleOAuthConfig resolve(String userId);
}
```

- [ ] **Step 4: Create ConfigGoogleCredentialResolver default bean**

Create `contacts-google/src/main/java/io/casehub/connectors/contacts/google/ConfigGoogleCredentialResolver.java`:

```java
package io.casehub.connectors.contacts.google;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.ConfigProvider;

@DefaultBean
@ApplicationScoped
public class ConfigGoogleCredentialResolver implements GoogleCredentialResolver {

    @Override
    public GoogleOAuthConfig resolve(String userId) {
        var config = ConfigProvider.getConfig();
        var prefix = "casehub.contacts.google.credentials." + userId;
        var refreshToken = config.getValue(prefix + ".refresh-token", String.class);
        var clientId = config.getValue(prefix + ".client-id", String.class);
        var clientSecret = config.getValue(prefix + ".client-secret", String.class);
        return new GoogleOAuthConfig(refreshToken, clientId, clientSecret);
    }
}
```

- [ ] **Step 5: Implement GoogleContactsPlatform**

Create `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleContactsPlatform.java`:

This is the most complex file. Key mapping logic:
- Google `Person` → `Contact` (map `names`, `emailAddresses`, `phoneNumbers`, `addresses`, `organizations`, `photos`, `biographies`)
- Google `ContactGroup` → `Group` (map `name`, `groupType`, `memberCount`)
- Sync via `requestSyncToken=true` / passing `syncToken` on `people().connections().list("people/me")`
- HTTP 410 Gone → throw `SyncTokenExpiredException`
- Pagination via `pageToken`/`nextPageToken`
- Search via `people().searchContacts()`

```java
package io.casehub.connectors.contacts.google;

import com.google.api.client.googleapis.javanet.GoogleNetHttpTransport;
import com.google.api.client.json.gson.GsonFactory;
import com.google.api.services.people.v1.PeopleService;
import com.google.api.services.people.v1.model.*;
import com.google.auth.http.HttpCredentialsAdapter;
import com.google.auth.oauth2.UserCredentials;
import io.casehub.connectors.*;
import io.casehub.connectors.contacts.model.*;
import io.casehub.connectors.contacts.model.Address;
import io.casehub.connectors.contacts.spi.ContactsPlatform;

import java.io.IOException;
import java.security.GeneralSecurityException;
import java.util.List;
import java.util.Map;
import java.util.logging.Level;
import java.util.logging.Logger;

public class GoogleContactsPlatform implements ContactsPlatform {

    private static final Logger LOG = Logger.getLogger(GoogleContactsPlatform.class.getName());
    private static final String PERSON_FIELDS = "names,emailAddresses,phoneNumbers,addresses,organizations,photos,biographies,metadata";
    private static final String GROUP_FIELDS = "name,groupType,memberCount";

    private final GoogleCredentialResolver resolver;

    public GoogleContactsPlatform(GoogleCredentialResolver resolver) {
        this.resolver = resolver;
    }

    @Override
    public String id() {
        return "google";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == ContactRead.class
            || capability == GroupRead.class
            || capability == ContactWrite.class;
    }

    @Override
    public ContactRead contactRead(String userId) {
        return new GoogleContactRead(buildService(userId));
    }

    @Override
    public GroupRead groupRead(String userId) {
        return new GoogleGroupRead(buildService(userId));
    }

    @Override
    public ContactWrite contactWrite(String userId) {
        return new GoogleContactWrite(buildService(userId));
    }

    private PeopleService buildService(String userId) {
        var config = resolver.resolve(userId);
        try {
            var credentials = UserCredentials.newBuilder()
                .setClientId(config.clientId())
                .setClientSecret(config.clientSecret())
                .setRefreshToken(config.refreshToken())
                .build();
            var transport = GoogleNetHttpTransport.newTrustedTransport();
            return new PeopleService.Builder(transport, GsonFactory.getDefaultInstance(),
                    new HttpCredentialsAdapter(credentials))
                .setApplicationName("casehub-connectors")
                .build();
        } catch (GeneralSecurityException | IOException e) {
            throw new RuntimeException("Failed to build Google People service", e);
        }
    }

    private class GoogleContactRead implements ContactRead {
        private final PeopleService service;

        GoogleContactRead(PeopleService service) {
            this.service = service;
        }

        @Override
        public Page<Contact> list(PageRequest pagination) {
            try {
                var request = service.people().connections().list("people/me")
                    .setPersonFields(PERSON_FIELDS)
                    .setPageSize(pagination.size() > 0 ? pagination.size() : 20);
                if (pagination.cursor() != null) {
                    request.setPageToken(pagination.cursor());
                }
                var response = request.execute();
                var contacts = mapContacts(response.getConnections());
                return new Page<>(contacts, response.getNextPageToken(),
                    response.getNextPageToken() != null);
            } catch (IOException e) {
                LOG.log(Level.WARNING, "Partial results — Google People API list failed", e);
                return new Page<>(List.of(), null, false);
            }
        }

        @Override
        public SyncResult<Contact> listSync(SyncRequest request) {
            try {
                var req = service.people().connections().list("people/me")
                    .setPersonFields(PERSON_FIELDS)
                    .setRequestSyncToken(true)
                    .setPageSize(request.pageSize() > 0 ? request.pageSize() : 100);
                if (request.syncToken() != null) {
                    req.setSyncToken(request.syncToken());
                }
                var response = req.execute();
                var contacts = mapContacts(response.getConnections());
                var deletedIds = List.<String>of(); // Google marks deleted via metadata.deleted
                if (response.getConnections() != null) {
                    deletedIds = response.getConnections().stream()
                        .filter(p -> p.getMetadata() != null && Boolean.TRUE.equals(p.getMetadata().getDeleted()))
                        .map(p -> p.getResourceName())
                        .toList();
                    contacts = contacts.stream()
                        .filter(c -> !deletedIds.contains(c.id()))
                        .toList();
                }
                return new SyncResult<>(contacts, deletedIds,
                    response.getNextSyncToken(), response.getNextPageToken() != null);
            } catch (com.google.api.client.googleapis.json.GoogleJsonResponseException e) {
                if (e.getStatusCode() == 410) {
                    throw new SyncTokenExpiredException(request.syncToken());
                }
                throw new RuntimeException("Google People API sync failed", e);
            } catch (IOException e) {
                throw new RuntimeException("Google People API sync failed", e);
            }
        }

        @Override
        public Contact get(String contactId) {
            try {
                var person = service.people().get(contactId)
                    .setPersonFields(PERSON_FIELDS)
                    .execute();
                return mapContact(person);
            } catch (IOException e) {
                throw new RuntimeException("Failed to get contact: " + contactId, e);
            }
        }

        @Override
        public Page<Contact> search(String query, PageRequest pagination) {
            try {
                var request = service.people().searchContacts()
                    .setQuery(query)
                    .setReadMask(PERSON_FIELDS)
                    .setPageSize(pagination.size() > 0 ? pagination.size() : 20);
                var response = request.execute();
                var contacts = response.getResults() == null ? List.<Contact>of()
                    : response.getResults().stream()
                        .map(r -> mapContact(r.getPerson()))
                        .toList();
                return new Page<>(contacts, null, false);
            } catch (IOException e) {
                LOG.log(Level.WARNING, "Partial results — Google People API search failed", e);
                return new Page<>(List.of(), null, false);
            }
        }
    }

    private class GoogleGroupRead implements GroupRead {
        private final PeopleService service;

        GoogleGroupRead(PeopleService service) {
            this.service = service;
        }

        @Override
        public List<Group> list() {
            try {
                var response = service.contactGroups().list()
                    .setGroupFields(GROUP_FIELDS)
                    .execute();
                if (response.getContactGroups() == null) return List.of();
                return response.getContactGroups().stream()
                    .map(this::mapGroup)
                    .toList();
            } catch (IOException e) {
                LOG.log(Level.WARNING, "Partial results — Google People API group list failed", e);
                return List.of();
            }
        }

        @Override
        public Page<Contact> listContacts(String groupId, PageRequest pagination) {
            try {
                var group = service.contactGroups().get(groupId)
                    .setGroupFields(GROUP_FIELDS)
                    .setMaxMembers(pagination.size() > 0 ? pagination.size() : 100)
                    .execute();
                if (group.getMemberResourceNames() == null) {
                    return new Page<>(List.of(), null, false);
                }
                var batchGet = service.people().getBatchGet()
                    .setResourceNames(group.getMemberResourceNames())
                    .setPersonFields(PERSON_FIELDS)
                    .execute();
                var contacts = batchGet.getResponses() == null ? List.<Contact>of()
                    : batchGet.getResponses().stream()
                        .map(r -> mapContact(r.getPerson()))
                        .toList();
                return new Page<>(contacts, null, false);
            } catch (IOException e) {
                LOG.log(Level.WARNING, "Partial results — Google People API group contacts failed", e);
                return new Page<>(List.of(), null, false);
            }
        }

        private Group mapGroup(ContactGroup cg) {
            var groupType = "SYSTEM_CONTACT_GROUP".equals(cg.getGroupType())
                ? GroupType.SYSTEM : GroupType.USER_CREATED;
            var memberCount = cg.getMemberCount() != null ? cg.getMemberCount() : 0;
            return new Group(cg.getResourceName(), cg.getName(), groupType, memberCount);
        }
    }

    private class GoogleContactWrite implements ContactWrite {
        private final PeopleService service;

        GoogleContactWrite(PeopleService service) {
            this.service = service;
        }

        @Override
        public Contact create(Contact contact) {
            try {
                var person = mapToPerson(contact);
                var created = service.people().createContact(person).execute();
                return mapContact(created);
            } catch (IOException e) {
                throw new RuntimeException("Failed to create contact", e);
            }
        }

        @Override
        public Contact update(String contactId, Contact contact) {
            try {
                var person = mapToPerson(contact);
                var updated = service.people().updateContact(contactId, person)
                    .setUpdatePersonFields(PERSON_FIELDS)
                    .execute();
                return mapContact(updated);
            } catch (IOException e) {
                throw new RuntimeException("Failed to update contact: " + contactId, e);
            }
        }

        @Override
        public void delete(String contactId) {
            try {
                service.people().deleteContact(contactId).execute();
            } catch (IOException e) {
                throw new RuntimeException("Failed to delete contact: " + contactId, e);
            }
        }

        private Person mapToPerson(Contact contact) {
            var person = new Person();
            if (contact.name() != null) {
                person.setNames(List.of(new Name()
                    .setGivenName(contact.name().givenName())
                    .setFamilyName(contact.name().familyName())
                    .setDisplayName(contact.name().displayName())));
            }
            if (contact.emails() != null && !contact.emails().isEmpty()) {
                person.setEmailAddresses(contact.emails().stream()
                    .map(e -> new EmailAddress().setValue(e.value()).setType(e.label()))
                    .toList());
            }
            if (contact.phones() != null && !contact.phones().isEmpty()) {
                person.setPhoneNumbers(contact.phones().stream()
                    .map(p -> new PhoneNumber().setValue(p.value()).setType(p.label()))
                    .toList());
            }
            if (contact.company() != null) {
                person.setOrganizations(List.of(new Organization()
                    .setName(contact.company())
                    .setTitle(contact.jobTitle())));
            }
            return person;
        }
    }

    private static List<Contact> mapContacts(List<Person> persons) {
        if (persons == null) return List.of();
        return persons.stream()
            .filter(p -> p.getMetadata() == null || !Boolean.TRUE.equals(p.getMetadata().getDeleted()))
            .map(GoogleContactsPlatform::mapContact)
            .toList();
    }

    static Contact mapContact(Person person) {
        var name = person.getNames() != null && !person.getNames().isEmpty()
            ? new ContactName(
                person.getNames().getFirst().getDisplayName(),
                person.getNames().getFirst().getGivenName(),
                person.getNames().getFirst().getFamilyName())
            : new ContactName(null, null, null);

        var emails = person.getEmailAddresses() == null ? List.<LabelledValue<String>>of()
            : person.getEmailAddresses().stream()
                .map(e -> new LabelledValue<>(e.getType(), e.getValue(),
                    e.getMetadata() != null && Boolean.TRUE.equals(e.getMetadata().getPrimary())))
                .toList();

        var phones = person.getPhoneNumbers() == null ? List.<LabelledValue<String>>of()
            : person.getPhoneNumbers().stream()
                .map(p -> new LabelledValue<>(p.getType(), p.getValue(),
                    p.getMetadata() != null && Boolean.TRUE.equals(p.getMetadata().getPrimary())))
                .toList();

        var addresses = person.getAddresses() == null ? List.<LabelledValue<Address>>of()
            : person.getAddresses().stream()
                .map(a -> new LabelledValue<>(a.getType(),
                    new Address(a.getStreetAddress(), a.getCity(), a.getRegion(),
                        a.getPostalCode(), a.getCountry()),
                    a.getMetadata() != null && Boolean.TRUE.equals(a.getMetadata().getPrimary())))
                .toList();

        var company = person.getOrganizations() != null && !person.getOrganizations().isEmpty()
            ? person.getOrganizations().getFirst().getName() : null;
        var jobTitle = person.getOrganizations() != null && !person.getOrganizations().isEmpty()
            ? person.getOrganizations().getFirst().getTitle() : null;

        var photoUrl = person.getPhotos() != null && !person.getPhotos().isEmpty()
            ? person.getPhotos().getFirst().getUrl() : null;

        var notes = person.getBiographies() != null && !person.getBiographies().isEmpty()
            ? person.getBiographies().getFirst().getValue() : null;

        return new Contact(person.getResourceName(), name, emails, phones, addresses,
            company, jobTitle, photoUrl, notes, Map.of());
    }
}
```

- [ ] **Step 6: Create ContactsGoogleBeans CDI producer**

Create `contacts-google/src/main/java/io/casehub/connectors/contacts/google/ContactsGoogleBeans.java`:

```java
package io.casehub.connectors.contacts.google;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

public class ContactsGoogleBeans {

    @Produces
    @ApplicationScoped
    GoogleContactsPlatform googleContactsPlatform(GoogleCredentialResolver resolver) {
        return new GoogleContactsPlatform(resolver);
    }
}
```

- [ ] **Step 7: Write unit test for mapping logic**

Create `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactsPlatformTest.java`:

Tests the `mapContact` static method (mapping Google `Person` → `Contact`). Does not require live API.

```java
package io.casehub.connectors.contacts.google;

import com.google.api.services.people.v1.model.*;
import io.casehub.connectors.contacts.model.GroupType;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class GoogleContactsPlatformTest {

    @Test
    void mapsPersonToContact() {
        var person = new Person()
            .setResourceName("people/123")
            .setNames(List.of(new Name()
                .setDisplayName("Alice Smith")
                .setGivenName("Alice")
                .setFamilyName("Smith")))
            .setEmailAddresses(List.of(
                new EmailAddress().setValue("alice@work.com").setType("work")
                    .setMetadata(new FieldMetadata().setPrimary(true)),
                new EmailAddress().setValue("alice@home.com").setType("home")))
            .setPhoneNumbers(List.of(
                new PhoneNumber().setValue("+1234567890").setType("mobile")
                    .setMetadata(new FieldMetadata().setPrimary(true))))
            .setOrganizations(List.of(
                new Organization().setName("Acme Corp").setTitle("Engineer")))
            .setPhotos(List.of(new Photo().setUrl("https://photo.url/alice")))
            .setBiographies(List.of(new Biography().setValue("A note")));

        var contact = GoogleContactsPlatform.mapContact(person);

        assertThat(contact.id()).isEqualTo("people/123");
        assertThat(contact.name().displayName()).isEqualTo("Alice Smith");
        assertThat(contact.name().givenName()).isEqualTo("Alice");
        assertThat(contact.name().familyName()).isEqualTo("Smith");
        assertThat(contact.emails()).hasSize(2);
        assertThat(contact.emails().getFirst().value()).isEqualTo("alice@work.com");
        assertThat(contact.emails().getFirst().primary()).isTrue();
        assertThat(contact.phones()).hasSize(1);
        assertThat(contact.phones().getFirst().label()).isEqualTo("mobile");
        assertThat(contact.company()).isEqualTo("Acme Corp");
        assertThat(contact.jobTitle()).isEqualTo("Engineer");
        assertThat(contact.photoUrl()).isEqualTo("https://photo.url/alice");
        assertThat(contact.notes()).isEqualTo("A note");
    }

    @Test
    void mapsPersonWithMinimalFields() {
        var person = new Person().setResourceName("people/456");

        var contact = GoogleContactsPlatform.mapContact(person);

        assertThat(contact.id()).isEqualTo("people/456");
        assertThat(contact.name().displayName()).isNull();
        assertThat(contact.emails()).isEmpty();
        assertThat(contact.phones()).isEmpty();
        assertThat(contact.addresses()).isEmpty();
        assertThat(contact.company()).isNull();
    }

    @Test
    void mapsAddresses() {
        var person = new Person()
            .setResourceName("people/789")
            .setAddresses(List.of(
                new com.google.api.services.people.v1.model.Address()
                    .setType("home")
                    .setStreetAddress("123 Main St")
                    .setCity("Springfield")
                    .setRegion("IL")
                    .setPostalCode("62701")
                    .setCountry("US")));

        var contact = GoogleContactsPlatform.mapContact(person);

        assertThat(contact.addresses()).hasSize(1);
        var addr = contact.addresses().getFirst();
        assertThat(addr.label()).isEqualTo("home");
        assertThat(addr.value().street()).isEqualTo("123 Main St");
        assertThat(addr.value().city()).isEqualTo("Springfield");
        assertThat(addr.value().country()).isEqualTo("US");
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-google -Dtest="GoogleContactsPlatformTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add contacts-google/ pom.xml
git commit -m "feat(#125): add contacts-google module — Google People API ContactsPlatform with sync support"
```

---

## Batch 4: Platform integration — GraphQL/MCP

### Task 5: Add ConnectorContactsApi and update ConnectorOperationsImpl

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorContactsApi.java`
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java` (add contacts to `ALL_SCOPES` and constructor, add contacts capability block)
- Modify: `graphql/pom.xml` (add `casehub-connectors-contacts-spi` dependency)
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorContactsApiTest.java`

**Interfaces:**
- Consumes: `ContactsPlatformService`, `ContactsPlatform`, model records from Task 2; `SyncRequest`, `SyncResult` from Task 1; `SecurityIdentity` from Quarkus
- Produces: GraphQL/MCP endpoints for contacts operations; updated `connectorsReport` with "contacts" scope

- [ ] **Step 1: Add contacts-spi dependency to graphql/pom.xml**

Add to `graphql/pom.xml` dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-contacts-spi</artifactId>
</dependency>
```

- [ ] **Step 2: Write failing test for ConnectorContactsApi**

Create `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorContactsApiTest.java`:

Unit test using mock `ContactsPlatformService` and `SecurityIdentity`. Verifies delegation to platform.

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.contacts.model.*;
import io.casehub.connectors.contacts.spi.ContactsPlatform;
import io.casehub.connectors.contacts.spi.ContactsPlatformService;
import io.quarkus.security.identity.SecurityIdentity;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.security.Principal;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ConnectorContactsApiTest {

    private ConnectorContactsApi api;
    private ContactsPlatform.ContactRead contactRead;
    private ContactsPlatform.GroupRead groupRead;
    private ContactsPlatform.ContactWrite contactWrite;

    @BeforeEach
    void setUp() {
        var platform = mock(ContactsPlatform.class);
        contactRead = mock(ContactsPlatform.ContactRead.class);
        groupRead = mock(ContactsPlatform.GroupRead.class);
        contactWrite = mock(ContactsPlatform.ContactWrite.class);

        when(platform.contactRead("test-user")).thenReturn(contactRead);
        when(platform.groupRead("test-user")).thenReturn(groupRead);
        when(platform.contactWrite("test-user")).thenReturn(contactWrite);
        when(platform.supports(ContactsPlatform.ContactRead.class)).thenReturn(true);
        when(platform.supports(ContactsPlatform.GroupRead.class)).thenReturn(true);
        when(platform.supports(ContactsPlatform.ContactWrite.class)).thenReturn(true);

        var platformService = new ContactsPlatformService(List.of(platform));
        when(platform.id()).thenReturn("test");

        var identity = mock(SecurityIdentity.class);
        var principal = mock(Principal.class);
        when(identity.getPrincipal()).thenReturn(principal);
        when(principal.getName()).thenReturn("test-user");

        api = new ConnectorContactsApi();
        api.platformService = platformService;
        api.identity = identity;
    }

    @Test
    void listContactsDelegatesToPlatform() {
        var contact = testContact("c1");
        when(contactRead.list(any())).thenReturn(new Page<>(List.of(contact), null, false));

        var result = api.listContacts("test", null, null);

        assertThat(result.items()).hasSize(1);
        assertThat(result.items().getFirst().id()).isEqualTo("c1");
    }

    @Test
    void syncContactsDelegatesToPlatform() {
        when(contactRead.listSync(any())).thenReturn(
            new SyncResult<>(List.of(), List.of("d1"), "token-2", false));

        var result = api.syncContacts("test", "token-1", 50);

        assertThat(result.deletedIds()).containsExactly("d1");
        assertThat(result.syncToken()).isEqualTo("token-2");
    }

    @Test
    void listGroupsDelegatesToPlatform() {
        when(groupRead.list()).thenReturn(List.of(
            new Group("g1", "Work", GroupType.USER_CREATED, 5)));

        var result = api.listGroups("test");

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().name()).isEqualTo("Work");
    }

    private Contact testContact(String id) {
        return new Contact(id, new ContactName("Test", "Test", "User"),
            List.of(), List.of(), List.of(), null, null, null, null, Map.of());
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest="ConnectorContactsApiTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement ConnectorContactsApi**

Create `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorContactsApi.java`:

Follow `ConnectorBankApi` pattern exactly — `@McpDomain`, `@ApplicationScoped`, `SecurityIdentity` for userId, `requireCapability()` helper.

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.*;
import io.casehub.connectors.contacts.model.Contact;
import io.casehub.connectors.contacts.model.Group;
import io.casehub.connectors.contacts.spi.ContactsPlatform;
import io.casehub.connectors.contacts.spi.ContactsPlatformService;
import io.quarkus.security.identity.SecurityIdentity;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.QueryParam;

import java.util.List;

@McpDomain(value = "connectors/contacts", app = "connectors",
    basePath = "/api/connectors/contacts",
    summary = "Contacts connector — contacts, groups, sync")
@ApplicationScoped
public class ConnectorContactsApi {

    @Inject
    ContactsPlatformService platformService;

    @Inject
    SecurityIdentity identity;

    private String userId() {
        return identity.getPrincipal().getName();
    }

    private <T> T requireCapability(String platformId, Class<T> capability, String operation) {
        var platform = platformService.platform(platformId);
        if (!platform.supports(capability)) {
            throw new UnsupportedCapabilityException(operation, capability.getSimpleName(),
                platformId, List.of());
        }
        if (capability == ContactsPlatform.ContactRead.class) {
            return capability.cast(platform.contactRead(userId()));
        } else if (capability == ContactsPlatform.GroupRead.class) {
            return capability.cast(platform.groupRead(userId()));
        } else if (capability == ContactsPlatform.ContactWrite.class) {
            return capability.cast(platform.contactWrite(userId()));
        }
        throw new IllegalArgumentException("Unknown capability: " + capability);
    }

    @PlatformQuery("List contacts from a provider")
    @RestPath("/contacts")
    public Page<Contact> listContacts(
            @QueryParam("platform") String platformId,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        var read = requireCapability(platformId, ContactsPlatform.ContactRead.class, "listContacts");
        return read.list(new PageRequest(cursor, pageSize != null ? pageSize : 20));
    }

    @PlatformQuery("Incremental sync of contacts")
    @RestPath("/contacts/sync")
    public SyncResult<Contact> syncContacts(
            @QueryParam("platform") String platformId,
            @QueryParam("syncToken") String syncToken,
            @QueryParam("pageSize") Integer pageSize) {
        var read = requireCapability(platformId, ContactsPlatform.ContactRead.class, "syncContacts");
        var request = syncToken != null
            ? new SyncRequest(syncToken, pageSize != null ? pageSize : 100)
            : SyncRequest.initial(pageSize != null ? pageSize : 100);
        return read.listSync(request);
    }

    @PlatformQuery("Get a specific contact by ID")
    @RestPath("/contacts/{contactId}")
    public Contact getContact(
            @QueryParam("platform") String platformId,
            @PathParam String contactId) {
        var read = requireCapability(platformId, ContactsPlatform.ContactRead.class, "getContact");
        return read.get(contactId);
    }

    @PlatformQuery("Search contacts by query")
    @RestPath("/contacts/search")
    public Page<Contact> searchContacts(
            @QueryParam("platform") String platformId,
            @QueryParam("query") String query,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        var read = requireCapability(platformId, ContactsPlatform.ContactRead.class, "searchContacts");
        return read.search(query, new PageRequest(cursor, pageSize != null ? pageSize : 20));
    }

    @PlatformQuery("List contact groups")
    @RestPath("/groups")
    public List<Group> listGroups(@QueryParam("platform") String platformId) {
        var read = requireCapability(platformId, ContactsPlatform.GroupRead.class, "listGroups");
        return read.list();
    }

    @PlatformQuery("List contacts in a group")
    @RestPath("/groups/{groupId}/contacts")
    public Page<Contact> listGroupContacts(
            @QueryParam("platform") String platformId,
            @PathParam String groupId,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        var read = requireCapability(platformId, ContactsPlatform.GroupRead.class, "listGroupContacts");
        return read.listContacts(groupId, new PageRequest(cursor, pageSize != null ? pageSize : 20));
    }

    @PlatformMutation("Create a new contact")
    @RestPath("/contacts")
    public Contact createContact(
            @QueryParam("platform") String platformId,
            Contact contact) {
        var write = requireCapability(platformId, ContactsPlatform.ContactWrite.class, "createContact");
        return write.create(contact);
    }

    @PlatformMutation("Update an existing contact")
    @RestPath("/contacts/{contactId}")
    public Contact updateContact(
            @QueryParam("platform") String platformId,
            @PathParam String contactId,
            Contact contact) {
        var write = requireCapability(platformId, ContactsPlatform.ContactWrite.class, "updateContact");
        return write.update(contactId, contact);
    }

    @PlatformMutation("Delete a contact")
    @RestPath("/contacts/{contactId}/delete")
    public void deleteContact(
            @QueryParam("platform") String platformId,
            @PathParam String contactId) {
        var write = requireCapability(platformId, ContactsPlatform.ContactWrite.class, "deleteContact");
        write.delete(contactId);
    }
}
```

Note: import `@PlatformQuery`, `@PlatformMutation`, `@RestPath`, `@McpDomain` from the graphql module's existing packages — check exact imports by looking at `ConnectorBankApi`.

- [ ] **Step 5: Update ConnectorOperationsImpl**

Modify `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java`:

1. Add `"contacts"` to `ALL_SCOPES` set
2. Add `ContactsPlatformService` to constructor injection
3. Add contacts capability enumeration block (following the document block pattern at lines 219-227):

```java
// In ALL_SCOPES:
private static final Set<String> ALL_SCOPES = Set.of("chat", "calendar", "bank", "email", "document", "contacts");

// In constructor: add ContactsPlatformService parameter

// Add contacts block after document block:
for (var id : contactsPlatformService.ids()) {
    var platform = contactsPlatformService.platform(id);
    var caps = new ArrayList<String>();
    if (platform.supports(ContactsPlatform.ContactRead.class)) caps.add("ContactRead");
    if (platform.supports(ContactsPlatform.GroupRead.class)) caps.add("GroupRead");
    if (platform.supports(ContactsPlatform.ContactWrite.class)) caps.add("ContactWrite");
    // Build PlatformInfo and add to contacts scope report
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest="ConnectorContactsApiTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full build to verify everything compiles together**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add graphql/
git commit -m "feat(#125): add ConnectorContactsApi — GraphQL/MCP contacts endpoints + connectorsReport integration"
```

---

## References

- `specs/issue-125-contacts-platform-spi/2026-09-29-contacts-platform-spi-design.md` — design spec
- `document-spi/` — SPI module template
- `document-ref/` — reference implementation template
- `document-google/` — Google provider template
- `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorBankApi.java` — GraphQL API template
- `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java:69` — ALL_SCOPES constant
- `connectors-api/src/main/java/io/casehub/connectors/Page.java` — pagination types
- Protocol PP-20260609-e3a2bd — SPI id() naming
- Protocol PP-20260610-83747b — pagination partial results
- casehubio/connectors#125
