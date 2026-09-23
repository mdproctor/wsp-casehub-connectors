# TrueLayer Consent Token Database Persistence — Design Spec

> **Issue:** casehubio/connectors#107
> **Date:** 2026-09-23
> **Status:** Draft
> **Parent:** casehubio/connectors#106 (TrueLayer BankPlatform Implementation)

## Overview

Replace the in-memory `ConcurrentHashMap<String, StoredConsent>` in
`TrueLayerConsentService` with database-backed persistence following the
platform's persistence SPI pattern (work/ledger). The service API is
unchanged — only the storage implementation changes.

A JVM restart currently loses all consent tokens, forcing every user
through PSD2 re-consent (bank redirect, SCA authentication). With
90-day AISP consent windows, this is significant operational friction.

## Scope

**In scope:**
- `ConsentTokenStore` SPI interface
- `JpaConsentTokenStore` implementation (Tier 1)
- `InMemoryConsentTokenStore` implementation (Tier 3)
- `ConsentTokenEntity` JPA entity + `ConsentTokenEntityMapper`
- `ConsentCleanupJob` scheduled cleanup
- Dependencies: `quarkus-hibernate-orm`, JDBC driver
- Tests for both store implementations

**Out of scope:**
- `pendingStates` persistence (CSRF state, 10-min TTL — ephemeral by design)
- `TrueLayerConsentService` API changes (none needed)
- Multi-instance locking (see §Thread-safe refresh)

## ConsentTokenStore SPI

```java
public interface ConsentTokenStore {

    void store(String userId, StoredConsent consent);

    StoredConsent find(String userId);

    void remove(String userId);

    int removeExpiredBefore(Instant cutoff);
}
```

Plain Java interface, no framework annotations. Keyed by `userId` —
one consent per user (matching the current `ConcurrentHashMap` semantics).

`removeExpiredBefore()` returns the count of removed entries for logging.
The cutoff is compared against `StoredConsent.consentExpiry()` — the
PSD2 consent grant expiry, not the short-lived access token expiry.

## JPA Implementation (Tier 1)

### ConsentTokenEntity

```java
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
}
```

`scopes` is stored as a comma-separated string (`ACCOUNTS,BALANCE`).
`ConsentScope` is a simple enum with 4 values — a join table is
unnecessary overhead. The mapper handles `List<ConsentScope>` ↔ `String`
conversion.

`userId` is the primary key — one row per user, `store()` does an
upsert (merge).

### ConsentTokenEntityMapper

```java
public class ConsentTokenEntityMapper {

    public static StoredConsent toDomain(ConsentTokenEntity entity) { ... }

    public static ConsentTokenEntity toEntity(String userId, StoredConsent consent) { ... }
}
```

Stateless utility class. Handles `List<ConsentScope>` ↔ comma-separated
string conversion for the `scopes` field.

### JpaConsentTokenStore

```java
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
        return entity == null ? null : ConsentTokenEntityMapper.toDomain(entity);
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

`@Transactional` on write methods. Read methods don't need a
transaction — `EntityManager.find()` works outside a transaction in
Quarkus (returns a detached entity).

## In-Memory Implementation (Tier 3)

```java
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

Thread-safe via `ConcurrentHashMap`. Wins over JPA when on the
classpath (CDI Tier 3, `@Priority(100)`).

## Scheduled Cleanup

```java
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

Framework-agnostic — no `@Scheduled` annotation. Scheduling wired
externally:

- **Quarkus:** `TrueLayerBeans` adds a `@Scheduled(every = "24h")`
  method that calls `cleanupJob.run()`
- **Spring:** Auto-config adds a `@Scheduled(cron = "0 0 3 * * *")`
  method

## TrueLayerConsentService Changes

Minimal refactoring — replace direct map access with store calls:

| Current | After |
|---------|-------|
| `consents.put(userId, consent)` | `store.store(userId, consent)` |
| `consents.get(userId)` | `store.find(userId)` |
| `consents.remove(userId)` | `store.remove(userId)` |

The `consents` field and its `ConcurrentHashMap` are removed. The
constructor takes `ConsentTokenStore` as a new parameter. `TrueLayerBeans`
wires it.

The `storeConsent()` package-private method (used by tests) delegates
to `store.store()`.

## Thread-safe Token Refresh

The existing per-userId `synchronized` block in `refreshToken()` works
unchanged. The lock protects against concurrent refresh within a single
JVM:

1. Acquire per-userId lock (same `refreshLocks` ConcurrentHashMap)
2. Double-check: `store.find(userId)` — if another thread refreshed,
   return that token
3. Call TrueLayer's token endpoint
4. `store.store(userId, refreshedConsent)` — JPA impl merges within
   `@Transactional`

For multi-instance deployments (multiple pods), the database provides
eventual consistency — worst case, two pods refresh simultaneously and
both write valid tokens. Last write wins, which is correct since both
new tokens are valid. The token endpoint is idempotent for refresh
operations with the same refresh token.

## Dependencies

Added to `bank-truelayer/pom.xml`:

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

Both managed by the platform parent BOM — no version needed.

`quarkus-hibernate-orm` is a compile dependency (provides
`EntityManager`). `quarkus-jdbc-postgresql` is test-scope only —
`bank-truelayer` is a library; the consuming application provides the
JDBC driver and datasource configuration.

Dev Services auto-provisions a PostgreSQL instance for
`@QuarkusTest` integration tests.

## Schema Management

Pre-release: Hibernate auto-DDL (`quarkus.hibernate-orm.database.generation=drop-and-create`)
in dev mode. The consuming application owns production schema management
(Flyway or equivalent). The entity mapping is the schema definition —
a single table with 7 columns, no relationships.

## Testing

### ConsentTokenStore contract tests

A shared test class `ConsentTokenStoreContractTest` defines the
behavioural contract:
- `store()` and `find()` round-trip
- `find()` returns null for unknown userId
- `store()` overwrites existing entry (upsert)
- `remove()` deletes entry
- `remove()` is idempotent (no error for unknown userId)
- `removeExpiredBefore()` removes only expired entries, returns count
- `removeExpiredBefore()` leaves non-expired entries untouched

Two concrete test classes extend it:
- `InMemoryConsentTokenStoreTest` — plain JUnit, instantiates
  `InMemoryConsentTokenStore` directly
- `JpaConsentTokenStoreTest` — `@QuarkusTest` with Dev Services
  PostgreSQL, injects `JpaConsentTokenStore`

### TrueLayerConsentServiceTest updates

Current tests construct the service directly with no CDI. After the
change, the constructor takes `ConsentTokenStore`. Tests pass
`InMemoryConsentTokenStore` — no change to test behaviour, just the
construction call.

### ConsentCleanupJob test

Plain JUnit. Creates `InMemoryConsentTokenStore`, stores entries with
mixed expiry times, runs the job, asserts correct entries remain.

## Deliverables

| Artifact | Description |
|----------|-------------|
| `ConsentTokenStore.java` | SPI interface |
| `JpaConsentTokenStore.java` | JPA implementation (Tier 1) |
| `InMemoryConsentTokenStore.java` | In-memory implementation (Tier 3) |
| `ConsentTokenEntity.java` | JPA entity |
| `ConsentTokenEntityMapper.java` | Entity ↔ domain mapper |
| `ConsentCleanupJob.java` | Scheduled cleanup job |
| `TrueLayerConsentService.java` | Refactored to use `ConsentTokenStore` |
| `TrueLayerBeans.java` | Updated CDI wiring + scheduler |
| `ConsentTokenStoreContractTest.java` | Shared contract tests |
| `InMemoryConsentTokenStoreTest.java` | In-memory store tests |
| `JpaConsentTokenStoreTest.java` | JPA store integration tests |
| `ConsentCleanupJobTest.java` | Cleanup job tests |
| `TrueLayerConsentServiceTest.java` | Updated constructor |
| `bank-truelayer/pom.xml` | Added `quarkus-hibernate-orm`, `quarkus-jdbc-postgresql` |

## References

- `bank-truelayer/TrueLayerConsentService.java` — current in-memory implementation
- `bank-truelayer/StoredConsent.java` — domain record (unchanged)
- `bank-truelayer/TrueLayerBeans.java` — CDI wiring
- `docs/specs/issue-106-truelayer-bankfeed/2026-09-22-truelayer-bankplatform-design.md` — parent design (§Consent token persistence defers to #107)
- `docs/specs/issue-106-truelayer-bankfeed/decisions.md` — D3 (consent as provider-internal), D5 (explicit token passing, consent store as provider concern)
- `docs/specs/2026-06-06-persistence-memory-module-design.md` — CDI tier ladder, in-memory store pattern
- `work/api/WorkItemStore.java` — store SPI pattern reference
- `work/runtime/JpaWorkItemStore.java` — EntityManager-based JPA implementation reference
- `work/persistence-memory/InMemoryWorkItemStore.java` — in-memory tier reference
- `persistence-backend-cdi-priority.md` — CDI tier protocol
