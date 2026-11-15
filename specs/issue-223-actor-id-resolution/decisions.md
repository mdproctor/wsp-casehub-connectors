# Decisions — #223 actorId resolution

## D1: Explicit userId on SPI methods

**Choice:** Add explicit userId parameters to flat SPIs (CalendarPlatform, DocumentPlatform, EmailPlatform) rather than using implicit CurrentPrincipal injection.
**Alternatives:**
- Implicit via CurrentPrincipal — simpler but breaks in async/background contexts, hides runtime dependency
- Hybrid (CurrentPrincipal fallback) — adds complexity, two paths to maintain
**Rationale:** SPI signatures should express their dependencies. User-scoped SPIs already prove this pattern works (ContactsPlatform, BankPlatform, ProjectPlatform). Explicit parameters work in any execution context.
**Trade-offs:** More SPI breaking changes; every caller site needs updating.
**Sources:** ContactsPlatform SPI (contacts-spi), CurrentPrincipal interface (platform-api), ServiceConnectionProvider interface (platform-api)
**Exploration:** quick
**Status:** captured

## D2: User-scoped accessor pattern for CalendarPlatform

**Choice:** Restructure CalendarPlatform with a `CalendarOperations` inner interface behind a `calendarOps(String userId)` accessor.
**Alternatives:**
- userId as first parameter on each method — simpler but noisy, inconsistent with capability-accessor pattern
**Rationale:** Consistent with ContactsPlatform, BankPlatform patterns. One userId entry point for all operations. Natural boundary for per-user client construction.
**Trade-offs:** Structural SPI change (bigger diff than adding a parameter).
**Sources:** ContactsPlatform.contactRead(String userId) pattern, CalendarPlatform.java:15
**Exploration:** quick
**Status:** captured

## D3: Scope includes all three flat SPIs

**Choice:** Retrofit CalendarPlatform, DocumentPlatform, and EmailPlatform in this issue.
**Alternatives:**
- Calendar only, follow-up issues for rest — smaller PR but same work deferred
- Calendar + Document, defer Email — arbitrary grouping
**Rationale:** All three have the same problem and the fix is mechanical once the pattern is established. One issue avoids three near-identical design+review cycles.
**Trade-offs:** Larger PR; more modules touched.
**Sources:** GitHub issue #223 body
**Exploration:** quick
**Status:** captured

## D4: userId on existing DocumentPlatform accessors

**Choice:** Add userId parameter to existing capability accessors: `files(String userId)`, `folders(String userId)`, etc.
**Alternatives:**
- Single DocumentOperations entry point — adds unnecessary wrapping layer when capability sub-interfaces already exist
**Rationale:** DocumentPlatform already has capability sub-interfaces. Adding userId to their accessors is the minimum change. Consistent with ContactsPlatform.contactRead(userId) which also returns a capability sub-interface.
**Trade-offs:** None significant — this is the natural fit.
**Sources:** DocumentPlatform.java, ContactsPlatform.java
**Exploration:** quick
**Depends on:** D1 (explicit userId strategy)
**Status:** captured

## D5: CurrentPrincipal at MCP/GraphQL boundary only

**Choice:** The MCP/GraphQL resolvers (ConnectorOperationsImpl) extract `currentPrincipal.actorId()` and pass it as userId to user-scoped SPI accessors. CurrentPrincipal is only used at the request boundary, not inside SPI implementations.
**Alternatives:**
- Explicit userId parameter on MCP tool definitions — over-exposes internal identity model to external callers
**Rationale:** The MCP layer is the request boundary where CurrentPrincipal is always available. The SPI stays explicit and testable; CurrentPrincipal stays at the edge.
**Trade-offs:** MCP tools can't act on behalf of a different user without CurrentPrincipal supporting impersonation.
**Sources:** ConnectorOperationsImpl.java:85 (currentPrincipal field), CurrentPrincipal.java:24
**Exploration:** quick
**Depends on:** D1 (explicit userId strategy)
**Status:** captured
