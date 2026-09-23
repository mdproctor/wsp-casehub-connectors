## D1: Store SPI in bank-truelayer with platform persistence pattern

**Choice:** Extract a `ConsentTokenStore` interface (plain Java, no framework annotations) inside `bank-truelayer`. Two implementations: `JpaConsentTokenStore` (`@ApplicationScoped`, Tier 1, `EntityManager` directly) and `InMemoryConsentTokenStore` (`@Alternative @Priority(100)`, Tier 3, `ConcurrentHashMap`). `TrueLayerConsentService` takes `ConsentTokenStore` as a constructor dependency. Separate `ConsentTokenEntity` JPA entity + `ConsentTokenEntityMapper`. `pendingStates` (CSRF, 10-min TTL) stays in-memory.
**Alternatives:**
- JPA in separate `bank-truelayer-jpa` module — extra module for one entity, no concrete use case for running without persistence in production
- No in-memory tier, JPA always required — breaks platform tier pattern, makes unit tests slower
**Rationale:** Follows the platform persistence SPI pattern exactly (work/ledger). Framework-agnostic SPI enables Spring/Quarkus parity. In-memory tier keeps tests fast and enables ephemeral deployments. The three extra classes (SPI + 2 impls) plus entity + mapper are mechanical, not complex.
**Trade-offs:** Adds ~5 new classes for what is currently 2 lines of ConcurrentHashMap. Introduces JPA (`quarkus-hibernate-orm`) and JDBC driver dependencies into `bank-truelayer`.
**Sources:** `work/api/WorkItemStore.java` (SPI pattern), `work/runtime/JpaWorkItemStore.java` (EntityManager usage), `work/persistence-memory/InMemoryWorkItemStore.java` (in-memory tier), `persistence-memory-module-design.md` (CDI tier ladder), `persistence-backend-cdi-priority.md` (tier protocol)
**Exploration:** quick
**Status:** captured

## D2: Scheduled cleanup for expired consents

**Choice:** `ConsentTokenStore` includes `removeExpiredBefore(Instant cutoff)`. A `ConsentCleanupJob` (plain Java, no framework annotations) runs daily to purge consents whose `consentExpiry` has passed. Framework-specific scheduling wired externally — CDI `@Scheduled` in `TrueLayerBeans`, Spring `@Scheduled` in auto-config. The job takes `ConsentTokenStore` as a constructor arg and exposes a `run()` method.
**Alternatives:**
- Lazy cleanup on read only — accumulates dead rows over time, defers a known problem
**Rationale:** Consent tokens carry sensitive data (access/refresh tokens). Expired entries have no business value and accumulate indefinitely without cleanup. Daily purge is trivial to implement now and prevents a predictable operational issue. The framework-agnostic job pattern maintains Spring/Quarkus parity.
**Trade-offs:** One additional class + scheduling wiring. Negligible complexity.
**Sources:** PSD2 consent lifecycle (90-day AISP window), D1 (store SPI design)
**Exploration:** quick
**Depends on:** D1 (store SPI)
**Status:** captured
