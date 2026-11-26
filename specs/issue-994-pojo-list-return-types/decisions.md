## D1: Schema generation strategy

**Choice:** Runtime schema generation via PlatformSchemaGenerator
**Alternatives:**
- Compile-time schema from TypeMirror — no runtime dependency, but loses Jackson/Jakarta/sealed hierarchy support
- Hybrid (APT fallback, runtime preferred) — unnecessary complexity for marginal benefit
**Rationale:** PlatformSchemaGenerator already handles all the platform's schema concerns (Jackson annotations, sealed hierarchies, Jakarta validation, enum inlining). Generating schemas at compile time from TypeMirror would require reimplementing all of this. The generated proxy calls `PlatformSchemaGenerator.generate(T.class)` once at construction time and caches the result.
**Trade-offs:** Adds schema-generator as a runtime dependency for consumers of @AgentService proxies
**Sources:** PlatformSchemaGenerator (platform/schema-generator), ResponseFormat.json() API
**Exploration:** quick
**Status:** captured

## D2: ObjectMapper sourcing

**Choice:** Constructor-injected ObjectMapper
**Alternatives:**
- New instance per proxy — ignores app-level Jackson configuration (custom modules, naming strategies)
- Static shared instance — same problem, plus thread-safety concerns with configuration
**Rationale:** CDI wiring passes the application's configured ObjectMapper, ensuring custom serialization settings (naming strategies, modules, mixins) are honoured. Consistent with how ObjectMapper is used elsewhere in the platform.
**Trade-offs:** Adds a third constructor parameter, changing the generated proxy signature
**Sources:** AgentServiceProcessor.java constructor generation
**Exploration:** quick
**Status:** captured

## D3: Collection type support scope

**Choice:** List + Set + Collection (all standard collection interfaces)
**Alternatives:**
- List only — matches issue spec exactly, simpler APT detection
**Rationale:** The APT processor already walks DeclaredType for type argument extraction. Supporting Set and Collection alongside List is minimal additional work in the type classification, and avoids a future issue for a trivial omission.
**Trade-offs:** Slightly more complex type detection in the APT processor; Set/Collection are deserialized as List then converted
**Sources:** Issue platform#994 description
**Exploration:** quick
**Status:** captured
