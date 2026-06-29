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
  `casehub-connectors-core`. Version managed by `casehub-parent` BOM
  (`<cloudevents.version>4.0.1</cloudevents.version>`).
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
- Update core `pom.xml` `<description>` — currently reads "No dependencies
  beyond CDI and java.net.http"; must reflect `cloudevents-core` and
  `jackson-databind`.

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

### ARC42STORIES.MD updates

The move changes core's dependency profile and eliminates a module. Several
sections need updating:

**§1 Top Quality Goals** — rewrite "Zero-dep core" row. The name was already
inaccurate (core depends on `quarkus-arc`). The real invariant is: no casehubio
dependencies. Rename to "Lightweight foundation". Mechanism: "CDI + external
standards only (CloudEvents SDK, Jackson); no casehubio dependencies in `core`
or `slack-bot`".

**§2 Constraints** — rewrite "core and slack-bot modules: zero non-JDK runtime
dependencies" row. Same problem — already wrong before this change. Rewrite to:
"no casehubio runtime dependencies; external libraries only (Quarkus CDI,
CloudEvents SDK, Jackson)". This is testable: grep core's dependency tree for
`io.casehub` and verify it returns nothing. Also rewrite header line 8 from
"Depends on: Quarkus BOM + JDK only (core module: JDK only)" to "Depends on:
Quarkus BOM + external standards (core: CDI, cloudevents-core,
jackson-databind; zero casehubio deps)".

**§4 Layer Taxonomy** — L6 row: update description from
"`ConnectorCloudEventAdapter`, `InboundConnectorTypes` — optional submodule" to
"`ConnectorsCloudEventAdapter`, `InboundConnectorTypes` — in core; observes
`InboundMessage`, fires `CloudEvent`".

**§5 Module Structure** — remove `cloud-events` row from the module table. Update
`core` row's "Depends on" from "JDK only" to "JDK, `cloudevents-core`,
`jackson-databind`". Remove the sentence explaining why `cloud-events` is
separate.

**§7 Deployment View** — remove `cloud-events` row. The adapter now activates
by CDI discovery in core; it is a no-op when no `CloudEvent` observer is
present (`fireAsync()` completes with no delivery). Consumers no longer make a
separate inclusion decision — every consumer of core gets the adapter
automatically.

**§13 Glossary** — update `ConnectorCloudEventAdapter` entry: rename to
`ConnectorsCloudEventAdapter`, change "CDI adapter in `cloud-events` module" to
"CDI adapter in `core`".

### Platform doc updates (casehub-parent)

- `docs/repos/casehub-connectors.md` — no `cloud-events` row exists in the
  module table (never added when the module was created in e0aa545). Update
  core row description to mention `ConnectorsCloudEventAdapter`. Update
  "Depends On" section — currently reads "Nothing in the casehubio ecosystem.
  Pure Java (java.net.http.HttpClient)"; must note `cloudevents-core` and
  `jackson-databind` as external deps while preserving the zero-casehubio
  statement.
- `docs/PLATFORM.md` build order — `casehub-connectors` comment is still
  accurate (no casehubio deps); no change needed
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
- Zero casehubio dependencies — preserved (cloudevents-core and jackson-databind
  are external libraries)