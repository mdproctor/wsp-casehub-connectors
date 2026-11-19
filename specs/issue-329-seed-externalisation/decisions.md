## D1: SeedLoader architecture

**Choice:** Shared `SeedDataResolver` in `connectors-api` with thin per-module SeedLoader wrappers
**Alternatives:**
- Refactor each SeedLoader independently — zero coupling but 12x duplicated resolution logic
- Single generic SeedLoader in `connectors-api` — too rigid for health-ref's polymorphic parsing and chat-ref's missing JavaTimeModule
**Rationale:** Resolution logic (file dir → classpath → empty) is identical across all 12 modules. Type-specific deserialization varies (health-ref uses readTree, chat-ref skips JavaTimeModule), so each module keeps a thin wrapper. All modules already share connectors-api and YamlMappers.
**Trade-offs:** Adds a cross-module dependency on SeedDataResolver, but connectors-api is already a shared dependency for all ref modules.
**Sources:** calendar-ref/SeedLoader.java, commerce-ref/SeedLoader.java, mobile-ref/SeedLoader.java, health-ref/SeedLoader.java
**Exploration:** quick
**Status:** captured

## D2: SeedDataResolver API shape

**Choice:** `Optional<InputStream> resolve(String domain, String filename)` — static utility, not CDI
**Alternatives:**
- Resolver handles deserialization too (`<T> List<T> resolve(...)` with TypeReference) — breaks for health-ref's readTree and chat-ref's missing JavaTimeModule
- CDI bean injected into each Backend — over-engineered for a file lookup, adds constructor injection to 12 classes
**Rationale:** Resolver answers "where are the bytes?" only. Each module retains full deserialization control. Static utility with ConfigProvider avoids CDI wiring changes. Domain subdirectory gives clean corpus layout.
**Trade-offs:** Static utility is less testable than CDI injection, but the resolver is trivial logic (check file, fall back to classpath).
**Sources:** calendar-ref/SeedLoader.java, health-ref/SeedLoader.java (readTree usage)
**Exploration:** quick
**Depends on:** D1 (shared resolver in connectors-api)
**Status:** captured

## D3: Where seed YAML lives after the move

**Choice:** Move to `src/test/resources/seed/` in ref modules; MCP server copies seed files into its JAR via Maven resource plugin
**Alternatives:**
- Keep in main resources with profile-based filtering — fragile Maven configuration
- New `seed-corpus` module — overkill for static YAML that rarely changes
**Rationale:** Moving to test scope is the simplest strip from production JARs. Test classpath still includes test resources, so existing tests work unchanged. Maven copy-resources pulls seed files into MCP server JAR at build time.
**Trade-offs:** Build-time file copying means the MCP server pom must list each ref module's seed directory explicitly. If a new ref module is added, the copy config must be updated.
**Sources:** mcp-server-connectors/pom.xml, issue #329 spec
**Exploration:** quick
**Status:** captured

## D4: Corpus directory layout

**Choice:** Domain-namespaced flat — `<seed-dir>/<domain>/<file>.yaml`
**Alternatives:**
- Flat, all files in one directory — collision risk (bank/accounts.yaml vs future domains)
- Module-namespaced (`calendar-ref/seed/events.yaml`) — leaks internal module naming to users
**Rationale:** Domain names are stable, user-facing concepts. Self-documenting for users authoring custom corpus files. No collision risk.
**Trade-offs:** Resolver must map domain names consistently — each ref module must agree on its domain string.
**Sources:** Issue #329 spec (corpus directory example)
**Exploration:** quick
**Depends on:** D2 (resolver takes domain parameter)
**Status:** captured
