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

The `SimulationDecoratorProcessor` generates CDI `@Decorator` classes that wrap each SPI. It currently iterates `spiClass.methods()` (Jandex — direct methods only) and generates simulation-wrapped overrides. Methods inherited from super-interfaces are NOT currently overridden.

If `as()` is inherited via `PlatformExtensible`, the decorator would inherit the default implementation, which checks `extensionType.isInstance(this)` — but `this` is the decorator, not the real implementation. Result: `as()` always returns `Optional.empty()`.

**Fix:** Change the main generation loop in `generateDecoratorSource()` from `spiClass.methods()` to `collectAllMethods(spiClass, index)` — which already exists in the processor for capability wrapper inner classes. Add method-name deduplication (pattern established in `addParameterEntries()` with `seen.add(method.name())`).

Inherited methods like `as()` go through the standard `generateSimulatedMethod()` path. The generated fallthrough code calls `delegate.as(extensionType)`, where `this` is the real bean — `extensionType.isInstance(this)` correctly evaluates against the concrete implementation.

This is strictly better than name-based detection because:
1. No hardcoded qualified name string — no cross-repo coupling
2. Handles any future inherited default methods automatically
3. Makes inherited methods simulation-configurable (consistent with all other methods)
4. Uses existing, tested infrastructure (`collectAllMethods`, `generateSimulatedMethod` with generic type parameter support)

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

1. **Unit test in simulation-generator:** Verify that when a `@SimulationEligible` interface extends a super-interface with default methods, the generated decorator includes overrides for those inherited methods (via `collectAllMethods`)
2. **Integration test in connectors:** Verify `as()` works through CDI proxy and simulation wrapper layers — inject the SPI, call `as()` with a known extension type, confirm correct Optional result

## References

- `SimulationDecoratorProcessor.java:212-223` — main method generation loop (change from `spiClass.methods()` to `collectAllMethods()`)
- `SimulationDecoratorProcessor.java:594-615` — `collectAllMethods`/`collectMethodsRecursive` (reuse for main loop)
- `SimulationDecoratorProcessor.java:362-457` — `generateSimulatedMethod` with generic type parameter support
- Decision review R1-02 — generic approach vs name-based detection
- `ChatPlatform.java` — representative SPI interface pattern
- `DocumentPlatform.java` — capability sub-interface pattern
- casehubio/connectors#225 — issue specification
