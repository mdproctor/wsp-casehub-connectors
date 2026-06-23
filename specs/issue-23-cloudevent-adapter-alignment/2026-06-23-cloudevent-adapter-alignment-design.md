# CloudEvent Adapter Alignment with IoT Pattern

**Issue:** casehubio/connectors#23
**Date:** 2026-06-23
**Status:** Draft

## Context

Commit `e0aa545` (Refs #20) introduced `ConnectorCloudEventAdapter` in a
separate `cloud-events/` submodule. This was done to avoid adding
`casehub-platform-api` as a dependency to `casehub-connectors-core`, preserving
core's zero-casehubio-dep stance.

The IoT adapter (`IoTCloudEventAdapter`, casehubio/iot#19) subsequently
established the authoritative pattern: adapter lives in the core API module,
with `casehub-platform-api` as a compile dependency. Issue #23 asks the
connectors adapter to follow this pattern.

## Key Insight

The adapter does not use any type from `casehub-platform-api`. Its imports
are `cloudevents-core` (CNCF standard library), CDI annotations, Jackson, JDK
types, and `InboundMessage` from core itself. The `cloud-events` module depends
on `casehub-platform-api` solely as a vehicle to get `cloudevents-core` on the
classpath.

The tension between "adapter in core" and "core stays zero casehubio dep" is
illusory. Core can depend on `cloudevents-core` directly — it is an external
CNCF library, not a casehubio artifact. Both principles are fully satisfied
with no tradeoffs.

## What Changes

### Module structure

- Add `io.cloudevents:cloudevents-core` as a compile dependency to
  `casehub-connectors-core`. This is managed by the `casehub-parent` BOM (same
  version IoT and Qhorus use).
- Add `com.fasterxml.jackson.core:jackson-databind` as a compile dependency to
  core. Required for `ObjectMapper` serialisation (canonical pattern rule 1,
  GE-20260621-629712). Core currently has no Jackson — outbound connectors
  build HTTP bodies with string formatting.
- Add `com.fasterxml.jackson.datatype:jackson-datatype-jsr310` as a test-scope
  dependency. The unit test constructs its own `ObjectMapper` and needs
  `JavaTimeModule` for `Instant` serialisation. Production Quarkus apps register
  it automatically.
- Delete `cloud-events/` submodule entirely — source, `pom.xml`, and `<module>`
  entry in the parent `pom.xml`.

Core gains two external library dependencies. Zero casehubio dependencies
preserved.

### Class placement and naming

| Aspect | Before | After |
|--------|--------|-------|
| Class name | `ConnectorCloudEventAdapter` | `ConnectorsCloudEventAdapter` |
| Package | `io.casehub.connectors.cloudevents` | `io.casehub.connectors` |
| Module | `cloud-events/` submodule | `casehub-connectors-core` |

Naming convention: `IoTCloudEventAdapter` matches `casehub-iot`;
`ConnectorsCloudEventAdapter` matches `casehub-connectors`. The `cloudevents`
sub-package is unnecessary for a single class — the adapter collocates with
`InboundMessage` in the root package.

### Code alignment with IoT reference

Minor style alignment with `IoTCloudEventAdapter`:

- Extract `TYPE_PREFIX` as a `private static final String` constant
- `SOURCE` remains dynamic (`"/casehub-connectors/" + connectorId`) because
  multiple connector implementations share one module — the source URI
  identifies the specific connector, not the subsystem. This is an intentional
  divergence from IoT, where a single subsystem owns all events and a static
  `SOURCE` is correct.
- Use 2-arg `withData("application/json", data)` instead of separate
  `withDataContentType()` + `withData()`

All 7 rules from GE-20260621-629712 are already satisfied. No functional
changes needed — the adapter produces identical CloudEvents.

### CloudEvent mapping (unchanged)

| Field | Value |
|-------|-------|
| `type` | `"io.casehub.connectors.inbound." + connectorType` |
| `source` | `"/casehub-connectors/" + connectorId` |
| `subject` | `"channel/" + externalChannelRef` |
| `id` | `UUID.randomUUID().toString()` |
| `time` | `receivedAt` as `OffsetDateTime` at UTC |
| `datacontenttype` | `application/json` |
| `data` | `InboundMessage` serialised as JSON bytes |
| `tenancyid` ext | From `InboundMessage.tenancyId()`, omitted when null |

### Platform doc updates

- `casehub-parent/docs/repos/casehub-connectors.md` — remove `cloud-events`
  row from module table; note `cloudevents-core` external dep in "Depends On"
- `casehub-parent/docs/PLATFORM.md` build order — `casehub-connectors` comment
  is still accurate (no casehubio deps); no change needed
- GE-20260621-629712 reference implementation list — update class name from
  `ConnectorCloudEventAdapter` to `ConnectorsCloudEventAdapter`

## Testing

Existing `ConnectorCloudEventAdapterTest` moves to core with the adapter.
Renamed to `ConnectorsCloudEventAdapterTest`. Same test structure — unit test
with a capturing `Event<CloudEvent>` mock, no CDI container. Covers:

- Correct `type` field derivation from `connectorType`
- `source` contains `connectorId`
- `subject` contains `externalChannelRef`
- `tenancyid` extension set when present, omitted when null
- Data is JSON-serialised with `application/json` content type
- `time` from `receivedAt`
- `id` is a UUID

## What Does Not Change

- `InboundMessage` record — no changes
- `InboundConnectorService` event firing — still `fireAsync()`
- All existing `InboundMessage` observers — unaffected
- Outbound `Connector` SPI — unaffected
- Every other module in the repo — unaffected
