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
