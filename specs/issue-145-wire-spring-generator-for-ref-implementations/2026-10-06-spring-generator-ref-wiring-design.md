# Design: Wire spring-generator for platform SPI ref implementations

**Issue:** casehubio/connectors#145
**Date:** 2026-10-06
**Scope:** casehubio/connectors (`connectors-spring` module)

## Problem Statement

The connectors repo maintains 9 platform SPI ref implementations with consistent CDI wiring (#141). The hand-written `connectors-spring` module covers core outbound connectors (Slack, Teams, SMS, WhatsApp) but not the platform SPIs. Spring consumers have no auto-configuration for `ChatPlatform`, `BankPlatform`, or any of the other 7 ref implementations.

The `spring-generator` Maven plugin exists in the platform repo and already handles the annotation mappings needed. It runs in `platform-spring`, `agent-spring`, and other modules. Connectors has never wired it in.

## Architecture

Extend `connectors-spring` with the `spring-generator` plugin. The plugin scans Jandex indexes from all 9 ref modules and generates `*AutoConfiguration.java` classes. Each generated class carries `@ConditionalOnClass` on its implementation type — consumers include only the ref modules they need, and auto-configs activate selectively.

The existing hand-written `ConnectorsAutoConfiguration` is untouched. Generated and manual configs coexist in the same module.

## Changes

### 1. `connectors-spring/pom.xml`

**Dependencies (optional):** Add all 9 ref modules and their SPI modules as `<optional>true</optional>` dependencies. The `@ConditionalOnClass` annotations on generated auto-configs mean these are not transitive — consumers declare whichever refs they want explicitly.

```
chat-spi, chat-ref
bank-spi, bank-ref
calendar-spi, calendar-ref
commerce-spi, commerce-ref
contacts-spi, contacts-ref
document-spi, document-ref
email-spi, email-ref
location-spi, location-ref
project-spi, project-ref
```

**Plugin:** Add `casehub-platform-spring-generator` with two executions:

| Execution | Goal | Phase | Purpose |
|-----------|------|-------|---------|
| `generate` | `generate` | `generate-sources` | Scan Quarkus module Jandex indexes, produce `*AutoConfiguration.java` |
| `verify-drift` | `verify` | `verify` | Compare Quarkus `@Produces` return types against Spring `@Bean` types; fail on drift |

Configuration: `<quarkusModules>` lists all 9 ref module directories (`${project.basedir}/../chat-ref`, etc.).

### 2. Generated auto-configuration classes (10 beans from 8 modules)

The generator produces one `@AutoConfiguration` class per scanned package. Each `@Produces` method with concrete bean parameters becomes a `@Bean @ConditionalOnMissingBean` method.

| Module | Generated class | Beans |
|--------|----------------|-------|
| chat-ref | `ChatRefAutoConfiguration` | `refChatPlatform(ChatBackend)`, `refInboundTranslator()` |
| bank-ref | `BankRefAutoConfiguration` | `refBankPlatform(BankBackend)` |
| calendar-ref | `CalendarRefAutoConfiguration` | `refCalendarPlatform(CalendarBackend)` |
| commerce-ref | `CommerceRefAutoConfiguration` | `refCommercePlatform(CommerceBackend)` |
| contacts-ref | `ContactsRefAutoConfiguration` | `refContactsPlatform(ContactsBackend)` |
| document-ref | `DocumentRefAutoConfiguration` | `refDocumentPlatform(DocumentBackend)` |
| email-ref | `EmailRefAutoConfiguration` | `refEmailPlatform(EmailBackend)` |
| location-ref | `LocationRefAutoConfiguration` | `refLocationPlatform(LocationBackend)` |

The `@DefaultBean` annotation on each backend class (e.g. `InMemoryChatBackend`) is also scanned — the generator produces a `@Bean @ConditionalOnMissingBean` for each, allowing Spring consumers to override backends just as Quarkus consumers can.

### 3. Hand-written `ProjectRefManualConfig.java` (1 CDI-dependent bean)

`ProjectRefBeans` has two producers. `refProjectPlatform(ProjectBackend)` is generatable. `projectPlatformService(Instance<ProjectPlatform>)` uses CDI `Instance<T>` — the generator cannot map this automatically.

```java
@AutoConfiguration
@ConditionalOnClass(ProjectPlatformService.class)
public class ProjectRefManualConfig {

    @Bean
    @ConditionalOnMissingBean
    ProjectPlatformService projectPlatformService(ObjectProvider<ProjectPlatform> platforms) {
        return new ProjectPlatformService(platforms.orderedStream().toList());
    }
}
```

Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` alongside the generated entries.

### 4. Verification

`SpringVerifyMojo` at the `verify` phase compares all Quarkus `@Produces` return types (from Jandex) against all Spring `@Bean` return types (from generated + manual sources). Any Quarkus producer with no corresponding Spring bean fails the build. This runs on every `mvn verify` — regressions are caught automatically when new producers are added to any ref module.

## What doesn't change

- `ConnectorsAutoConfiguration.java` — existing hand-written core connector auto-config is untouched
- The 9 ref modules — already normalised in #141, no further changes needed
- The SPI interfaces — framework-agnostic, no CDI or Spring annotations
- `connectors-api` — framework-agnostic shared types

## Key constraint

The `@ConditionalOnClass` on each generated auto-config is the selectivity mechanism. A Spring consumer that depends on `chat-ref` and `bank-ref` (but not the other 7) gets only `ChatRefAutoConfiguration` and `BankRefAutoConfiguration` activated. The optional dependencies in `connectors-spring` are not transitive.

## References

- casehubio/parent#469 — dual-framework support epic ("Generate, Don't Write" principle)
- casehubio/connectors#99 — prior closed issue (core connector extraction only)
- casehubio/connectors#141 — ref implementation CDI normalisation (prerequisite)
- `platform-spring/pom.xml` — spring-generator plugin configuration pattern
- `connectors-spring/ConnectorsAutoConfiguration.java` — existing hand-written CDI→Spring mappings
- `SpringGeneratorMojo` / `SpringVerifyMojo` — generator and verify implementations
