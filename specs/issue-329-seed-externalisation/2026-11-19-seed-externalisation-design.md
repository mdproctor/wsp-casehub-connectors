# Seed Externalisation Design

**Issue:** casehubio/connectors#329
**Date:** 2026-11-19
**Status:** Draft

## Overview

Ref provider seed data (YAML files in `src/main/resources/seed/`) is currently baked into production JARs. This wastes space for every consumer that uses real providers and prevents standalone MCP server users from customising the data without a rebuild.

This design externalises seed data into a configurable file-based corpus directory, with classpath fallback for tests and graceful empty-data degradation when neither source is available.

## Scope

**In scope:**
- Shared `SeedDataResolver` utility in `connectors-api`
- Modify all 12 `SeedLoader` classes to use the resolver
- Move 25 seed YAML files from `src/main/resources/` to `src/test/resources/` (11 modules + mobile-ref normalisation)
- MCP server `pom.xml` — Maven resource copying to bundle corpus under `corpus/`
- File a separate issue for first-run seed bootstrap UX under #324 epic

**Out of scope:**
- voice-ref (no seed YAML — hard-coded canned responses)
- First-run CLI/web bootstrap experience (separate issue)
- Connection homepage (casehubio/casehub-mosaic#328)

## Architecture

### SeedDataResolver (connectors-api)

A static utility class in `io.casehub.connectors.seed` that resolves seed data with a three-tier fallback:

```
1. File-based:  ${casehub.ref.seed-dir}/<domain>/<filename>
2. Classpath:   /seed/<filename>  (present in test scope and MCP server JAR)
3. Empty:       Optional.empty()  (no crash, provider starts with no data)
```

```java
package io.casehub.connectors.seed;

public final class SeedDataResolver {

    public static Optional<InputStream> resolve(String domain, String filename) { ... }

    public static Optional<InputStream> resolve(String domain, String filename,
                                                 Class<?> classpathAnchor) { ... }
}
```

**Config property:** `casehub.ref.seed-dir` — optional. When set, the resolver checks `<seed-dir>/<domain>/<filename>` as a file path first. When unset, skips straight to classpath.

**Classpath anchor:** The two-arg overload uses `SeedDataResolver.class` for classpath lookup. The three-arg overload allows the caller to specify the anchor class — needed for mobile-ref during migration if any classpath-scoped resources remain on a transitional path.

**Thread safety:** The config property is read once and cached. The resolver is stateless beyond that.

### Per-module SeedLoader changes

Each of the 12 `SeedLoader` classes changes from:

```java
// Before
try (var is = SeedLoader.class.getResourceAsStream("/seed/events.yaml")) {
    if (is == null) throw new IllegalStateException("Missing /seed/events.yaml");
    return YAML.readValue(is, new TypeReference<>() {});
}
```

to:

```java
// After
var maybeIs = SeedDataResolver.resolve("calendar", "events.yaml");
if (maybeIs.isEmpty()) return List.of();
try (var is = maybeIs.get()) {
    return YAML.readValue(is, new TypeReference<>() {});
}
```

Each module retains its own `SeedLoader` with domain-specific types, `TypeReference`s, and ObjectMapper configuration (health-ref's `readTree`, chat-ref's missing `JavaTimeModule`).

### mobile-ref normalisation

mobile-ref's seed YAML files move from:
- `src/main/resources/io/casehub/connectors/mobile/ref/seed/*.yaml`

to:
- `src/test/resources/seed/*.yaml`

Its `SeedLoader` changes from package-scoped classpath paths (`/io/casehub/connectors/mobile/ref/seed/numbers.yaml`) to the standard convention (`SeedDataResolver.resolve("mobile", "numbers.yaml")`).

### Seed YAML file moves

All 25 seed YAML files move from `src/main/resources/` to `src/test/resources/`:

| Module | Files | From | To |
|--------|-------|------|----|
| bank-ref | accounts, balances, transactions | `main/resources/seed/` | `test/resources/seed/` |
| calendar-ref | calendars, events | `main/resources/seed/` | `test/resources/seed/` |
| chat-ref | channels, members | `main/resources/seed/` | `test/resources/seed/` |
| commerce-ref | products | `main/resources/seed/` | `test/resources/seed/` |
| contacts-ref | contacts, groups | `main/resources/seed/` | `test/resources/seed/` |
| document-ref | files, folders | `main/resources/seed/` | `test/resources/seed/` |
| email-ref | messages | `main/resources/seed/` | `test/resources/seed/` |
| health-ref | appointments, providers, records | `main/resources/seed/` | `test/resources/seed/` |
| location-ref | geocoding, places | `main/resources/seed/` | `test/resources/seed/` |
| project-ref | project-data | `main/resources/seed/` | `test/resources/seed/` |
| travel-ref | accommodation, transport | `main/resources/seed/` | `test/resources/seed/` |
| mobile-ref | conversations, messages, numbers, voicemails | `main/resources/.../seed/` | `test/resources/seed/` |

### MCP server corpus bundling

The `mcp-server-connectors` module uses `maven-resources-plugin` `copy-resources` executions to pull seed YAML from each ref module's `src/test/resources/seed/` into `target/quarkus-app/corpus/<domain>/` at build time. This places the corpus as loose files alongside `quarkus-run.jar`, not inside the application JAR.

Corpus directory layout in the MCP server distribution:

```
target/quarkus-app/
  quarkus-run.jar
  corpus/
    bank/accounts.yaml, balances.yaml, transactions.yaml
    calendar/calendars.yaml, events.yaml
    chat/channels.yaml, members.yaml
    commerce/products.yaml
    contacts/contacts.yaml, groups.yaml
    document/files.yaml, folders.yaml
    email/messages.yaml
    health/appointments.yaml, providers.yaml, records.yaml
    location/geocoding.yaml, places.yaml
    mobile/conversations.yaml, messages.yaml, numbers.yaml, voicemails.yaml
    project/project-data.yaml
    travel/accommodation.yaml, transport.yaml
```

The MCP server's `application.properties` sets:

```properties
casehub.ref.seed-dir=corpus/
```

This resolves relative to the working directory. Users run the server from the distribution directory (`java -jar quarkus-run.jar`), so `corpus/` resolves to the sibling directory. Users can edit these files directly or point `casehub.ref.seed-dir` to a different location.

### Native image profile

The existing native profile in `mcp-server-connectors/pom.xml`:

```xml
<quarkus.native.resources.includes>seed/*.yaml,seed/*.json</quarkus.native.resources.includes>
```

Changes to:

```xml
<quarkus.native.resources.includes>corpus/**/*.yaml</quarkus.native.resources.includes>
```

## Graceful degradation

When `SeedDataResolver.resolve()` returns `Optional.empty()`:
- Each `SeedLoader` method returns an empty collection (`List.of()`, `Map.of()`)
- Backend classes initialise with no seed data
- The ref provider is fully functional — just empty until data is added via API calls
- No exceptions thrown, no warnings logged (empty is a valid state)

## Testing

- **Existing ref module tests:** Unaffected. Test classpath includes `src/test/resources/`, so seed YAML is available via classpath fallback.
- **SeedDataResolver unit tests:** Test three-tier resolution — file-based override, classpath fallback, empty fallback. Use temp directory for file-based tests.
- **MCP server integration test:** Verify corpus files are bundled and resolvable at runtime.

## Future work (separate issue)

A first-run seed bootstrap experience under the #324 MCP server packaging epic. On startup, if no corpus directory exists, prompt the user (CLI or web) to extract default seed data to a user-editable location. This builds on the infrastructure delivered here but is a distinct UX concern.

## References

- casehubio/connectors#329 — this issue
- casehubio/connectors#324 — MCP server packaging epic
- casehubio/casehub-mosaic#328 — connection homepage (auth management)
- `calendar-ref/src/main/java/.../SeedLoader.java` — representative loading pattern
- `mobile-ref/src/main/java/.../SeedLoader.java` — package-scoped variant
- `health-ref/src/main/java/.../SeedLoader.java` — readTree variant
- `connectors-api` — shared types module where SeedDataResolver will live
- `io.casehub.yaml.jackson.YamlMappers` — shared YAML ObjectMapper factory
