# PlatformExtensible Base Interface with as()

**Issue:** casehubio/connectors#225
**Branch:** issue-225-platform-extensible
**Date:** 2026-11-14

## Problem

Microsoft Graph providers (and future vendor-specific providers) expose capabilities that don't belong in the general platform SPI — SharePoint site discovery, Teams 1:1 chats, Adaptive Cards, Focused Inbox. Consumers need a type-safe way to access these without coupling to concrete implementation classes.

`instanceof` checks against implementation classes are broken in this architecture: CDI proxies implement the SPI interface but are not instances of the concrete class, and `@SimulationEligible` wrappers have the same problem.

## Design

### PlatformExtensible interface

Add to `connectors-api` (`io.casehub.connectors`):

```java
public interface PlatformExtensible {
    default <T> Optional<T> as(Class<T> extensionType) {
        if (extensionType.isInstance(this)) {
            return Optional.of(extensionType.cast(this));
        }
        return Optional.empty();
    }
}
```

The default implementation uses `isInstance(this)` — correct for the real bean and for CDI proxies (which delegate the call to the contextual instance).

### SPI changes

All 13 platform SPIs extend `PlatformExtensible`:

| SPI | Module |
|-----|--------|
| ChatPlatform | chat-spi |
| DocumentPlatform | document-spi |
| CalendarPlatform | calendar-spi |
| BankPlatform | bank-spi |
| EmailPlatform | email-spi |
| ContactsPlatform | contacts-spi |
| LocationPlatform | location-spi |
| CommercePlatform | commerce-spi |
| TravelPlatform | travel-spi |
| ProjectPlatform | project-spi |
| HealthPlatform | health-spi |
| VoicePlatform | voice-spi |
| MobilePlatform | mobile-spi |

Each SPI adds `extends PlatformExtensible` to its interface declaration. No other changes needed — no existing code breaks since this is purely additive.

WebPlatform (web-spi) also extends it if present (found in `@SimulationEligible` search).

### Consumer pattern

```java
platform.as(MicrosoftDriveExtensions.class).ifPresent(msft ->
    msft.sharePoint().listSites()
);
```

The extension type is an interface (e.g., `MicrosoftDriveExtensions`), not the concrete class. The provider implements both the SPI and the extension interface. Extension interfaces live in the provider module (e.g., `document-microsoft`), not in the SPI module.

### Simulation decorator processor update

**Repo:** casehubio/platform (simulation-generator module)

The `SimulationDecoratorProcessor` generates CDI `@Decorator` classes that wrap each SPI. It iterates `spiClass.methods()` (Jandex — direct methods only) and generates simulation-wrapped overrides. Methods inherited from super-interfaces are NOT currently overridden.

If `as()` is inherited via `PlatformExtensible`, the decorator would inherit the default implementation, which checks `extensionType.isInstance(this)` — but `this` is the decorator, not the real implementation. Result: `as()` always returns `Optional.empty()`.

**Fix:** After the existing `supports()` special-case handling (line 216-219), add detection for `PlatformExtensible` as a super-interface. When found, generate a simple delegation override:

```java
@Override
public <T> Optional<T> as(Class<T> extensionType) {
    return delegate.as(extensionType);
}
```

This is NOT wrapped with simulation logic — it's pure delegation. The precedent is the `supports()` method which also gets special handling rather than standard simulation wrapping.

The detection uses Jandex interface traversal: check if any super-interface in the hierarchy has the qualified name `io.casehub.connectors.PlatformExtensible`.

### CDI proxies

No change needed. CDI proxies intercept all method calls and delegate to the contextual instance. The default `as()` implementation on `PlatformExtensible` will be called on the actual bean, where `this` is the real implementation — `isInstance(this)` works correctly.

### NoOp / Default implementations

Each SPI's `NoOp*` or `Default*` implementation does not implement any extension interfaces, so `as()` returns `Optional.empty()` by default. No changes needed.

## Scope

### In scope
- `PlatformExtensible` interface in `connectors-api`
- All 13+ platform SPIs extend it
- `SimulationDecoratorProcessor` updated to generate `as()` delegation
- Processor test coverage for the new delegation
- `connectors-api` dependency added to `simulation-generator` test scope (for Jandex index)

### Out of scope
- Concrete vendor extension interfaces (e.g., `MicrosoftDriveExtensions`) — separate issues per the issue's "Vendor extension candidates" table
- SPI promotion candidates (free/busy, meeting responses, version history, granular permissions) — separate issues

## Testing

1. **Unit test in simulation-generator:** Verify that when a `@SimulationEligible` interface extends `PlatformExtensible`, the generated decorator includes the `as()` delegation override
2. **Integration test in connectors:** Verify `as()` works through CDI proxy and simulation wrapper layers — inject the SPI, call `as()` with a known extension type, confirm correct Optional result

## References

- `SimulationDecoratorProcessor.java:212-223` — main method generation loop
- `SimulationDecoratorProcessor.java:216-219` — `supports()` special-case precedent
- `ChatPlatform.java` — representative SPI interface pattern
- `DocumentPlatform.java` — capability sub-interface pattern
- casehubio/connectors#225 — issue specification
