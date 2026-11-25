# Manifest Requires — Backward-Chaining Setup Dependencies

## Problem

Connectors have setup prerequisites — Signal needs SMS to receive a verification code, email needs a mail provider. These dependencies are currently implicit, hardcoded into setup playbooks (`connector-status: numbrrs`). The system cannot discover or resolve them automatically.

## Design

### Manifest Schema Changes

Two new optional fields on `ConnectorDeclaration`:

**`requires`** — list of capabilities this connector needs from the ecosystem:

```yaml
requires:
  - capability: sms
    sub: [receive]
    reason: "SMS verification code during registration"
```

- `capability` (required) — the abstract capability category
- `sub` (optional) — list of sub-capability qualifiers to narrow the requirement
- `reason` (optional) — human-readable explanation, shown when presenting resolution options

**`provides`** — list of capabilities this connector makes available:

```yaml
provides:
  - capability: sms
    sub: [receive]
  - capability: number-info
```

- `capability` (required) — the capability category
- `sub` (optional) — sub-capability qualifiers. Unqualified means "full capability"

Both fields accept a mixed form: plain strings for capabilities with no sub-qualifiers, structured objects when sub-qualifiers are needed. The parser normalises both to `CapabilityDeclaration` records.

### Capability Model

Capabilities are convention-based strings — no enum, the vocabulary grows from real needs. Sub-capabilities narrow a category along any dimension (direction, scope, coverage) without baking a taxonomy into the naming scheme.

**Matching rule:** A requirement `sms[receive]` is satisfied by a provider declaring `sms` (unqualified = full capability) or `sms[receive, ...]` (qualified, includes the needed sub-capability). An unqualified requirement `sms` is satisfied by any provider of `sms` regardless of sub-qualifiers.

**Connectors never reference other connectors.** Signal declares "I need sms[receive]" — it has no knowledge of Numbrrs, Twilio, or any provider. Resolution is orthogonal, owned by the lifecycle system.

### Java Records

In `connectors-api`, alongside `ConnectorManifest`:

```java
public record CapabilityDeclaration(String capability, List<String> sub) {
    public CapabilityDeclaration(String capability) {
        this(capability, List.of());
    }
}

public record RequirementDeclaration(String capability, List<String> sub, String reason) {}
```

`ConnectorDeclaration` gains two new fields:

```java
public record ConnectorDeclaration(
        String id,
        String platform,
        String description,
        AuthDeclaration auth,
        List<BeanDeclaration> produces,
        List<RequirementDeclaration> requires,
        List<CapabilityDeclaration> provides,
        @JsonProperty("setup-playbook") String setupPlaybook,
        HealthDeclaration health) {}
```

### DependencyResolver

A new class in `connectors-api` — pure function over manifests and registry status:

```java
public class DependencyResolver {

    public DependencyResolver(List<ConnectorManifest> manifests) { ... }

    /** Requirements not satisfied by any READY provider. */
    public List<UnmetRequirement> unmetRequirements(
            String connectorId, ConnectorRegistry registry);

    /** Options for satisfying a capability requirement. */
    public List<ResolutionOption> resolutionOptions(
            String capability, List<String> sub, ConnectorRegistry registry);
}
```

**`UnmetRequirement`** — a requirement where no provider is READY:

```java
public record UnmetRequirement(
        String capability, List<String> sub, String reason,
        List<ResolutionOption> options) {}
```

**`ResolutionOption`** — sealed hierarchy:

```java
public sealed interface ResolutionOption {
    record ProviderOption(
            String connectorId, ConnectorStatus status,
            String setupPlaybook) implements ResolutionOption {}
    record ManualOption(
            String capability) implements ResolutionOption {}
}
```

- `ProviderOption` — a connector that provides this capability (may be READY, UNCONFIGURED, etc.)
- `ManualOption` — always present as fallback: the user satisfies the requirement themselves

The resolver builds an internal index from `provides` declarations across all manifests. When asked for resolution options, it finds matching providers, checks their status via the registry, and always appends a `ManualOption`.

### Playbook Update

Signal's `signal-registration.yaml` changes from hardcoded connector reference:

```yaml
# Before
- connector-status:
    connector-id: numbrrs
    require: READY
```

to capability-based requirement:

```yaml
# After
- require-capability:
    capability: sms
    sub: [receive]
```

The `require-capability` playbook step uses `DependencyResolver.resolutionOptions()` to find a READY provider. If none is READY, it presents resolution options to the caller.

### Updated Manifests

**Signal:**
```yaml
connector:
  id: signal
  platform: chat
  requires:
    - capability: sms
      sub: [receive]
      reason: "SMS verification code during registration"
```

**Numbrrs:**
```yaml
connector:
  id: numbrrs
  platform: mobile
  provides:
    - capability: sms
      sub: [receive]
    - capability: number-info
```

**Telegram:** No changes — no requirements, no capabilities to advertise beyond its own platform.

### Testing

- **DependencyResolver unit tests:** matching with/without sub-capabilities, unmet requirements, resolution options including manual fallback, unqualified provider satisfies qualified requirement
- **Manifest parser tests:** new `requires` and `provides` fields parse correctly, mixed string/object form, missing fields default gracefully
- **Integration test:** verify resolver works with real manifests from the classpath (extend or complement #363 test)

## References

- `connectors-api/.../ConnectorManifest.java` — existing manifest records
- `connectors-api/.../ConnectorManifestLoader.java` — YAML parser with FAIL_ON_UNKNOWN_PROPERTIES=false
- `connectors-api/.../ConnectorRegistry.java` — runtime status tracking
- `connector-lifecycle/.../ConnectorLifecycleProcessor.java` — build extension that processes manifests
- `mobile-numbrrs/.../connector-manifest.yaml` — existing Numbrrs manifest
- `chat-signal/.../connector-manifest.yaml` — existing Signal manifest
- `playbooks/signal-registration.yaml` — current playbook with hardcoded connector-status: numbrrs
- casehubio/connectors#365 — issue
- D1–D5 in decisions.md — design decisions
