# Consent Token Database Persistence — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #107 — TrueLayer: Database-backed consent token persistence
**Issue group:** #107

**Goal:** Replace in-memory ConcurrentHashMap consent token storage with
database-backed persistence following the platform's persistence SPI
pattern.

**Architecture:** Extract a `ConsentTokenStore` SPI interface with two
implementations: `JpaConsentTokenStore` (Tier 1, EntityManager) and
`InMemoryConsentTokenStore` (Tier 3, ConcurrentHashMap). Refactor
`TrueLayerConsentService` to depend on the SPI. Add scheduled cleanup
for expired consents.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (EntityManager, no Panache),
quarkus-hibernate-orm, PostgreSQL (Dev Services for tests)

## Global Constraints

- No Panache — use `EntityManager` directly for all JPA operations
- `StoredConsent` record is unchanged — a separate `ConsentTokenEntity`
  JPA entity handles persistence
- `ConsentTokenEntityMapper` converts between domain record and entity
- CDI Tier 1 (`@ApplicationScoped`) for JPA, Tier 3
  (`@Alternative @Priority(100)`) for in-memory
- Framework-agnostic core — no CDI/Spring annotations on SPI or
  cleanup job
- `pendingStates` stays in-memory (CSRF, 10-min TTL)

---

## Batch 1: SPI + In-Memory Store

After this batch: `ConsentTokenStore` SPI exists,
`InMemoryConsentTokenStore` passes all contract tests,
`TrueLayerConsentService` uses the store SPI, existing tests pass.

### Task 1: ConsentTokenStore SPI + InMemoryConsentTokenStore + contract tests

**Files:**
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentTokenStore.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/InMemoryConsentTokenStore.java`
- Create: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/ConsentTokenStoreContractTest.java`
- Create: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/InMemoryConsentTokenStoreTest.java`

**Interfaces:**
- Consumes: `StoredConsent` record, `ConsentScope` enum (existing)
- Produces: `ConsentTokenStore` interface — `store(String, StoredConsent)`,
  `find(String) → StoredConsent`, `remove(String)`,
  `removeExpiredBefore(Instant) → int`. Used by Task 2, 3, 4.

- [ ] **Step 1: Write ConsentTokenStore SPI interface**

```java
package io.casehub.connectors.bank.truelayer;

import java.time.Instant;

public interface ConsentTokenStore {

    void store(String userId, StoredConsent consent);

    StoredConsent find(String userId);

    void remove(String userId);

    int removeExpiredBefore(Instant cutoff);
}
```

- [ ] **Step 2: Write ConsentTokenStoreContractTest (abstract)**

Defines the behavioural contract. Concrete test classes provide the
store instance.

```java
package io.casehub.connectors.bank.truelayer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

abstract class ConsentTokenStoreContractTest {

    protected abstract ConsentTokenStore createStore();

    private ConsentTokenStore store;

    @BeforeEach
    void setUp() {
        store = createStore();
    }

    @Test
    void storeAndFind_roundTrip() {
        StoredConsent consent = consent("token-1", "refresh-1",
                Instant.now().plusSeconds(3600));
        store.store("user-1", consent);
        StoredConsent found = store.find("user-1");
        assertThat(found.accessToken()).isEqualTo("token-1");
        assertThat(found.refreshToken()).isEqualTo("refresh-1");
        assertThat(found.scopes()).containsExactly(
                ConsentScope.ACCOUNTS, ConsentScope.BALANCE);
    }

    @Test
    void find_unknownUser_returnsNull() {
        assertThat(store.find("nonexistent")).isNull();
    }

    @Test
    void store_overwrites_existingEntry() {
        store.store("user-1", consent("old-token", "refresh",
                Instant.now().plusSeconds(3600)));
        store.store("user-1", consent("new-token", "refresh",
                Instant.now().plusSeconds(3600)));
        assertThat(store.find("user-1").accessToken())
                .isEqualTo("new-token");
    }

    @Test
    void remove_deletesEntry() {
        store.store("user-1", consent("token", "refresh",
                Instant.now().plusSeconds(3600)));
        store.remove("user-1");
        assertThat(store.find("user-1")).isNull();
    }

    @Test
    void remove_unknownUser_noError() {
        store.remove("nonexistent");
    }

    @Test
    void removeExpiredBefore_removesOnlyExpired() {
        Instant now = Instant.now();
        store.store("expired-user", consent("t1", "r1",
                now.minusSeconds(3600)));
        store.store("active-user", consent("t2", "r2",
                now.plusSeconds(86400)));

        int removed = store.removeExpiredBefore(now);

        assertThat(removed).isEqualTo(1);
        assertThat(store.find("expired-user")).isNull();
        assertThat(store.find("active-user")).isNotNull();
    }

    @Test
    void removeExpiredBefore_noneExpired_returnsZero() {
        store.store("active-user", consent("t1", "r1",
                Instant.now().plusSeconds(86400)));
        assertThat(store.removeExpiredBefore(Instant.now())).isZero();
    }

    private StoredConsent consent(String accessToken, String refreshToken,
                                  Instant consentExpiry) {
        return new StoredConsent(accessToken, refreshToken,
                Instant.now().plusSeconds(3600), consentExpiry,
                List.of(ConsentScope.ACCOUNTS, ConsentScope.BALANCE),
                Instant.now());
    }
}
```

- [ ] **Step 3: Write InMemoryConsentTokenStore**

```java
package io.casehub.connectors.bank.truelayer;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Alternative
@Priority(100)
@ApplicationScoped
public class InMemoryConsentTokenStore implements ConsentTokenStore {

    private final Map<String, StoredConsent> store = new ConcurrentHashMap<>();

    @Override
    public void store(String userId, StoredConsent consent) {
        store.put(userId, consent);
    }

    @Override
    public StoredConsent find(String userId) {
        return store.get(userId);
    }

    @Override
    public void remove(String userId) {
        store.remove(userId);
    }

    @Override
    public int removeExpiredBefore(Instant cutoff) {
        int[] count = {0};
        store.entrySet().removeIf(e -> {
            if (e.getValue().consentExpiry().isBefore(cutoff)) {
                count[0]++;
                return true;
            }
            return false;
        });
        return count[0];
    }
}
```

- [ ] **Step 4: Write InMemoryConsentTokenStoreTest**

```java
package io.casehub.connectors.bank.truelayer;

class InMemoryConsentTokenStoreTest extends ConsentTokenStoreContractTest {

    @Override
    protected ConsentTokenStore createStore() {
        return new InMemoryConsentTokenStore();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="InMemoryConsentTokenStoreTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 7 contract tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentTokenStore.java bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/InMemoryConsentTokenStore.java bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/ConsentTokenStoreContractTest.java bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/InMemoryConsentTokenStoreTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#107): add ConsentTokenStore SPI and InMemoryConsentTokenStore"
```

---

### Task 2: Refactor TrueLayerConsentService to use ConsentTokenStore

**Files:**
- Modify: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentService.java`
- Modify: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerBeans.java`
- Modify: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentServiceTest.java`

**Interfaces:**
- Consumes: `ConsentTokenStore` (from Task 1)
- Produces: Updated `TrueLayerConsentService` constructor that takes
  `ConsentTokenStore` as a parameter. Used by `TrueLayerBeans`.

- [ ] **Step 1: Update TrueLayerConsentServiceTest constructor call**

In `TrueLayerConsentServiceTest.setUp()`, add `InMemoryConsentTokenStore`
to the constructor:

```java
@BeforeEach
void setUp() {
    service = new TrueLayerConsentService(
            "test-client-id", "test-client-secret",
            "https://auth.truelayer-sandbox.com", null,
            new InMemoryConsentTokenStore());
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="TrueLayerConsentServiceTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — constructor doesn't accept `ConsentTokenStore` yet

- [ ] **Step 2: Refactor TrueLayerConsentService**

Remove the `consents` `ConcurrentHashMap` field. Add `ConsentTokenStore`
constructor parameter. Replace all `consents.put/get/remove` with
`store.store/find/remove`.

Changes to `TrueLayerConsentService.java`:
1. Remove field: `private final Map<String, StoredConsent> consents = new ConcurrentHashMap<>();`
2. Add field: `private final ConsentTokenStore store;`
3. Add constructor parameter: `ConsentTokenStore store`
4. In constructor body: `this.store = store;`
5. In `exchangeCode()`: `consents.put(userId, consent)` → `store.store(userId, consent)`
6. In `consentStatus()`: `consents.get(userId)` → `store.find(userId)`
7. In `revokeConsent()`: `consents.remove(userId)` → `store.remove(userId)`
8. In `getUserToken()`: `consents.get(userId)` → `store.find(userId)`
9. In `refreshToken()`:
   - `consents.get(userId)` → `store.find(userId)` (double-check read)
   - `consents.remove(userId)` → `store.remove(userId)` (refresh failure)
   - `consents.put(userId, refreshed)` → `store.store(userId, refreshed)`
10. In `storeConsent()`: `consents.put(userId, consent)` → `store.store(userId, consent)`

- [ ] **Step 3: Update TrueLayerBeans to wire ConsentTokenStore**

In `TrueLayerBeans.trueLayerConsentService()`, add `ConsentTokenStore`
parameter:

```java
@Produces
@ApplicationScoped
public TrueLayerConsentService trueLayerConsentService(
        OidcClient oidcClient,
        ConsentTokenStore consentTokenStore,
        @ConfigProperty(name = "casehub.connectors.bank.truelayer.client-id",
                        defaultValue = "") String clientId,
        @ConfigProperty(name = "casehub.connectors.bank.truelayer.client-secret",
                        defaultValue = "") String clientSecret,
        @ConfigProperty(name = "casehub.connectors.bank.truelayer.auth-url",
                        defaultValue = "https://auth.truelayer.com") String authUrl) {
    return new TrueLayerConsentService(clientId, clientSecret, authUrl,
            oidcClient, consentTokenStore);
}
```

- [ ] **Step 4: Run all existing tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS (TrueLayerConsentServiceTest,
InMemoryConsentTokenStoreTest, TrueLayerClientTest,
TrueLayerBankPlatformTest)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentService.java bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerBeans.java bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentServiceTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#107): refactor TrueLayerConsentService to use ConsentTokenStore SPI"
```

---

## Batch 2: JPA Implementation + Scheduled Cleanup

After this batch: `JpaConsentTokenStore` passes contract tests against
PostgreSQL (Dev Services), `ConsentCleanupJob` purges expired consents
on schedule, full module builds clean.

### Task 3: JPA store — entity, mapper, implementation, integration test

**Files:**
- Modify: `bank-truelayer/pom.xml`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentTokenEntity.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentTokenEntityMapper.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/JpaConsentTokenStore.java`
- Create: `bank-truelayer/src/test/resources/application.properties`
- Create: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/ConsentTokenEntityMapperTest.java`
- Create: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/JpaConsentTokenStoreTest.java`

**Interfaces:**
- Consumes: `ConsentTokenStore` interface (Task 1), `StoredConsent`,
  `ConsentScope`
- Produces: `JpaConsentTokenStore` — `@ApplicationScoped` (Tier 1),
  injected by CDI when no `@Alternative` is present.
  `ConsentTokenEntity` JPA entity mapped to `truelayer_consent_token`
  table. `ConsentTokenEntityMapper` with `toDomain(ConsentTokenEntity)`
  and `toEntity(String, StoredConsent)`.

- [ ] **Step 1: Add dependencies to bank-truelayer/pom.xml**

Add before the `<!-- Testing -->` comment:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-postgresql</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Write ConsentTokenEntityMapper tests (failing)**

```java
package io.casehub.connectors.bank.truelayer;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ConsentTokenEntityMapperTest {

    @Test
    void toEntity_and_toDomain_roundTrip() {
        Instant now = Instant.now();
        Instant accessExpiry = now.plusSeconds(3600);
        Instant consentExpiry = now.plusSeconds(86400L * 90);
        StoredConsent consent = new StoredConsent(
                "access-token", "refresh-token",
                accessExpiry, consentExpiry,
                List.of(ConsentScope.ACCOUNTS, ConsentScope.BALANCE), now);

        ConsentTokenEntity entity = ConsentTokenEntityMapper.toEntity(
                "user-1", consent);
        StoredConsent result = ConsentTokenEntityMapper.toDomain(entity);

        assertThat(result.accessToken()).isEqualTo("access-token");
        assertThat(result.refreshToken()).isEqualTo("refresh-token");
        assertThat(result.accessTokenExpiry()).isEqualTo(accessExpiry);
        assertThat(result.consentExpiry()).isEqualTo(consentExpiry);
        assertThat(result.scopes()).containsExactly(
                ConsentScope.ACCOUNTS, ConsentScope.BALANCE);
        assertThat(result.grantedAt()).isEqualTo(now);
    }

    @Test
    void toEntity_scopesSerialization() {
        StoredConsent consent = new StoredConsent(
                "t", "r", Instant.now(), Instant.now(),
                List.of(ConsentScope.ACCOUNTS, ConsentScope.PAYMENTS),
                Instant.now());
        ConsentTokenEntity entity = ConsentTokenEntityMapper.toEntity(
                "user-1", consent);
        assertThat(entity.getScopes()).isEqualTo("ACCOUNTS,PAYMENTS");
    }

    @Test
    void toDomain_nullRefreshToken() {
        StoredConsent consent = new StoredConsent(
                "t", null, Instant.now(), Instant.now(),
                List.of(ConsentScope.ACCOUNTS), Instant.now());
        ConsentTokenEntity entity = ConsentTokenEntityMapper.toEntity(
                "user-1", consent);
        StoredConsent result = ConsentTokenEntityMapper.toDomain(entity);
        assertThat(result.refreshToken()).isNull();
    }
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="ConsentTokenEntityMapperTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `ConsentTokenEntityMapper` does not exist

- [ ] **Step 3: Write ConsentTokenEntity**

```java
package io.casehub.connectors.bank.truelayer;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.time.Instant;

@Entity
@Table(name = "truelayer_consent_token")
public class ConsentTokenEntity {

    @Id
    @Column(name = "user_id")
    private String userId;

    @Column(name = "access_token", nullable = false)
    private String accessToken;

    @Column(name = "refresh_token")
    private String refreshToken;

    @Column(name = "access_token_expiry", nullable = false)
    private Instant accessTokenExpiry;

    @Column(name = "consent_expiry", nullable = false)
    private Instant consentExpiry;

    @Column(name = "scopes", nullable = false)
    private String scopes;

    @Column(name = "granted_at", nullable = false)
    private Instant grantedAt;

    protected ConsentTokenEntity() {}

    public ConsentTokenEntity(String userId, String accessToken,
                               String refreshToken, Instant accessTokenExpiry,
                               Instant consentExpiry, String scopes,
                               Instant grantedAt) {
        this.userId = userId;
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
        this.accessTokenExpiry = accessTokenExpiry;
        this.consentExpiry = consentExpiry;
        this.scopes = scopes;
        this.grantedAt = grantedAt;
    }

    public String getUserId() { return userId; }
    public String getAccessToken() { return accessToken; }
    public String getRefreshToken() { return refreshToken; }
    public Instant getAccessTokenExpiry() { return accessTokenExpiry; }
    public Instant getConsentExpiry() { return consentExpiry; }
    public String getScopes() { return scopes; }
    public Instant getGrantedAt() { return grantedAt; }

    public void setAccessToken(String accessToken) { this.accessToken = accessToken; }
    public void setRefreshToken(String refreshToken) { this.refreshToken = refreshToken; }
    public void setAccessTokenExpiry(Instant accessTokenExpiry) { this.accessTokenExpiry = accessTokenExpiry; }
    public void setConsentExpiry(Instant consentExpiry) { this.consentExpiry = consentExpiry; }
    public void setScopes(String scopes) { this.scopes = scopes; }
    public void setGrantedAt(Instant grantedAt) { this.grantedAt = grantedAt; }
}
```

- [ ] **Step 4: Write ConsentTokenEntityMapper**

```java
package io.casehub.connectors.bank.truelayer;

import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class ConsentTokenEntityMapper {

    public static StoredConsent toDomain(ConsentTokenEntity entity) {
        List<ConsentScope> scopes = Arrays.stream(entity.getScopes().split(","))
                .map(String::trim)
                .map(ConsentScope::valueOf)
                .toList();
        return new StoredConsent(
                entity.getAccessToken(),
                entity.getRefreshToken(),
                entity.getAccessTokenExpiry(),
                entity.getConsentExpiry(),
                scopes,
                entity.getGrantedAt());
    }

    public static ConsentTokenEntity toEntity(String userId,
                                               StoredConsent consent) {
        String scopes = consent.scopes().stream()
                .map(Enum::name)
                .collect(Collectors.joining(","));
        return new ConsentTokenEntity(
                userId,
                consent.accessToken(),
                consent.refreshToken(),
                consent.accessTokenExpiry(),
                consent.consentExpiry(),
                scopes,
                consent.grantedAt());
    }
}
```

- [ ] **Step 5: Run mapper tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="ConsentTokenEntityMapperTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 3 mapper tests PASS

- [ ] **Step 6: Write JpaConsentTokenStore**

```java
package io.casehub.connectors.bank.truelayer;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import java.time.Instant;

@ApplicationScoped
public class JpaConsentTokenStore implements ConsentTokenStore {

    @Inject
    EntityManager em;

    @Override
    @Transactional
    public void store(String userId, StoredConsent consent) {
        em.merge(ConsentTokenEntityMapper.toEntity(userId, consent));
    }

    @Override
    public StoredConsent find(String userId) {
        ConsentTokenEntity entity = em.find(ConsentTokenEntity.class, userId);
        return entity == null ? null
                : ConsentTokenEntityMapper.toDomain(entity);
    }

    @Override
    @Transactional
    public void remove(String userId) {
        ConsentTokenEntity entity = em.find(ConsentTokenEntity.class, userId);
        if (entity != null) em.remove(entity);
    }

    @Override
    @Transactional
    public int removeExpiredBefore(Instant cutoff) {
        return em.createQuery(
                "DELETE FROM ConsentTokenEntity e WHERE e.consentExpiry < :cutoff")
                .setParameter("cutoff", cutoff)
                .executeUpdate();
    }
}
```

- [ ] **Step 7: Create test application.properties**

Create `bank-truelayer/src/test/resources/application.properties`:

```properties
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.datasource.devservices.enabled=true
```

- [ ] **Step 8: Write JpaConsentTokenStoreTest**

```java
package io.casehub.connectors.bank.truelayer;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;

@QuarkusTest
class JpaConsentTokenStoreTest extends ConsentTokenStoreContractTest {

    @Inject
    JpaConsentTokenStore jpaStore;

    @Inject
    EntityManager em;

    @Override
    protected ConsentTokenStore createStore() {
        return jpaStore;
    }

    @BeforeEach
    @Transactional
    void cleanTable() {
        em.createQuery("DELETE FROM ConsentTokenEntity").executeUpdate();
    }
}
```

Add the required import for `EntityManager`:
```java
import jakarta.persistence.EntityManager;
```

- [ ] **Step 9: Add quarkus-junit5 test dependency to pom.xml**

Add to the testing section of `bank-truelayer/pom.xml`:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 10: Run JPA integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="JpaConsentTokenStoreTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Dev Services starts PostgreSQL, all 7 contract tests PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add bank-truelayer/pom.xml bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentTokenEntity.java bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentTokenEntityMapper.java bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/JpaConsentTokenStore.java bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/ConsentTokenEntityMapperTest.java bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/JpaConsentTokenStoreTest.java bank-truelayer/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#107): add JpaConsentTokenStore with ConsentTokenEntity and mapper"
```

---

### Task 4: ConsentCleanupJob + scheduler wiring

**Files:**
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentCleanupJob.java`
- Create: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/ConsentCleanupJobTest.java`
- Modify: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerBeans.java`
- Modify: `bank-truelayer/pom.xml` (add `quarkus-scheduler`)

**Interfaces:**
- Consumes: `ConsentTokenStore.removeExpiredBefore(Instant)` (Task 1)
- Produces: `ConsentCleanupJob.run()` — called by `TrueLayerBeans`
  scheduler method

- [ ] **Step 1: Write ConsentCleanupJobTest (failing)**

```java
package io.casehub.connectors.bank.truelayer;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ConsentCleanupJobTest {

    @Test
    void run_removesExpiredConsents_leavesActiveOnes() {
        InMemoryConsentTokenStore store = new InMemoryConsentTokenStore();
        Instant now = Instant.now();

        store.store("expired-1", new StoredConsent(
                "t1", "r1", now.minusSeconds(7200),
                now.minusSeconds(3600),
                List.of(ConsentScope.ACCOUNTS), now.minusSeconds(86400)));
        store.store("expired-2", new StoredConsent(
                "t2", "r2", now.minusSeconds(7200),
                now.minusSeconds(60),
                List.of(ConsentScope.ACCOUNTS), now.minusSeconds(86400)));
        store.store("active", new StoredConsent(
                "t3", "r3", now.plusSeconds(3600),
                now.plusSeconds(86400L * 90),
                List.of(ConsentScope.ACCOUNTS, ConsentScope.BALANCE),
                now));

        ConsentCleanupJob job = new ConsentCleanupJob(store);
        job.run();

        assertThat(store.find("expired-1")).isNull();
        assertThat(store.find("expired-2")).isNull();
        assertThat(store.find("active")).isNotNull();
        assertThat(store.find("active").accessToken()).isEqualTo("t3");
    }

    @Test
    void run_noExpired_noErrors() {
        InMemoryConsentTokenStore store = new InMemoryConsentTokenStore();
        store.store("active", new StoredConsent(
                "t1", "r1", Instant.now().plusSeconds(3600),
                Instant.now().plusSeconds(86400L * 90),
                List.of(ConsentScope.ACCOUNTS), Instant.now()));

        ConsentCleanupJob job = new ConsentCleanupJob(store);
        job.run();

        assertThat(store.find("active")).isNotNull();
    }
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="ConsentCleanupJobTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `ConsentCleanupJob` does not exist

- [ ] **Step 2: Write ConsentCleanupJob**

```java
package io.casehub.connectors.bank.truelayer;

import org.jboss.logging.Logger;

import java.time.Instant;

public class ConsentCleanupJob {

    private static final Logger LOG = Logger.getLogger(ConsentCleanupJob.class);

    private final ConsentTokenStore store;

    public ConsentCleanupJob(ConsentTokenStore store) {
        this.store = store;
    }

    public void run() {
        int removed = store.removeExpiredBefore(Instant.now());
        if (removed > 0) {
            LOG.infof("Cleaned up %d expired consent tokens", removed);
        }
    }
}
```

- [ ] **Step 3: Run cleanup tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest="ConsentCleanupJobTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: both tests PASS

- [ ] **Step 4: Add quarkus-scheduler dependency to pom.xml**

Add before the `<!-- Testing -->` comment in `bank-truelayer/pom.xml`:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-scheduler</artifactId>
    </dependency>
```

- [ ] **Step 5: Wire ConsentCleanupJob + scheduler in TrueLayerBeans**

Add to `TrueLayerBeans.java`:

```java
@Produces
@ApplicationScoped
public ConsentCleanupJob consentCleanupJob(ConsentTokenStore store) {
    return new ConsentCleanupJob(store);
}

@Scheduled(every = "24h")
void cleanupExpiredConsents() {
    Arc.container().instance(ConsentCleanupJob.class).get().run();
}
```

Add imports:
```java
import io.quarkus.arc.Arc;
import io.quarkus.scheduler.Scheduled;
```

Note: `Arc.container().instance()` is used because `@Scheduled` methods
run on the bean that declares them (`TrueLayerBeans`), which is the
producer — it does not directly inject the cleanup job. Alternatively,
inject `ConsentCleanupJob` as a field.

Alternative (simpler — inject as field):
```java
@Inject
ConsentCleanupJob cleanupJob;

@Scheduled(every = "24h")
void cleanupExpiredConsents() {
    cleanupJob.run();
}
```

Use the field injection approach — it is simpler and idiomatic.

- [ ] **Step 6: Run full module build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all tests pass (contract tests × 2 impls,
mapper tests, cleanup tests, existing service/client/platform tests)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add bank-truelayer/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#107): add ConsentCleanupJob with daily scheduled purge"
```

---

## References

- [2026-09-23-consent-db-persistence-design.md] — design spec this plan implements
- [decisions.md] — D1 (store SPI pattern), D2 (scheduled cleanup)
- [bank-truelayer/TrueLayerConsentService.java] — current in-memory implementation
- [bank-truelayer/StoredConsent.java] — domain record (unchanged)
- [bank-truelayer/TrueLayerBeans.java] — CDI wiring to modify
- [bank-truelayer/TrueLayerConsentServiceTest.java] — existing tests to update
- [bank-truelayer/pom.xml] — dependencies to add
- [persistence-backend-cdi-priority.md] — CDI tier protocol
- [GitHub #107] — focal issue
