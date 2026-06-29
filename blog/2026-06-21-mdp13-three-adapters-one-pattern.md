---
layout: post
title: "Three Adapters, One Pattern"
date: 2026-06-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors, casehub-iot, casehub-qhorus]
tags: [cloudevents, cdi, cross-repo, adapter-pattern]
---

The task was straightforward: add a CloudEvent adapter to casehub-connectors so
casehub-ras can observe inbound messages without coupling to connector types.
IoT and Qhorus had already shipped theirs. I just needed to follow the pattern.

The pattern had bugs.

## What was actually wrong

Two adapters had already shipped — `IoTCloudEventAdapter` and
`QhorusCloudEventAdapter`. Both observing `@ObservesAsync` domain events and
re-firing them as `Event<CloudEvent>.fireAsync()`. Clean, simple, correct-looking.

Except `fireAsync()` returns a `CompletionStage` that neither adapter handled.
If a downstream observer threw — say, casehub-ras hit a null pointer processing
a temperature event — the exception propagated into the `CompletionStage`, nobody
awaited it, and it vanished into Quarkus's managed executor thread pool. No log,
no alert, the event silently disappeared.

IoT had a second problem: its serialisation error path threw
`UncheckedIOException` from inside the `@ObservesAsync` method. CDI catches
that on the managed executor thread too — same symptom, different mechanism.
The `StateChangeEvent` vanishes without a trace.

And a third: a private static `ObjectMapper` that isolated the adapter from
whatever Jackson config the consuming application had registered. Custom
serializers? Invisible.

## The design question connectors forced

I couldn't just copy the existing pattern — the existing pattern was wrong.
But I also couldn't just fix the bugs. `InboundMessage` didn't carry the
right data for a CloudEvent adapter to work correctly.

CloudEvent `type` is the field downstream systems use for routing and filtering.
It has to be a stable semantic category — "slack", "email", "sms" — not a
mutable instance identifier like "slack-inbound" or "twilio-sms-inbound". IoT
uses `deviceClass` (from the type hierarchy), Qhorus uses `messageType` (from
the speech-act enum). Both are stable fields on the event record.
`InboundMessage` had no equivalent — only `connectorId`, which is an instance
name that could change on rename.

And `tenancyId` wasn't on the record at all. Both IoT and Qhorus read it
directly from their event records — never from `CurrentPrincipal`, which is
`@RequestScoped` and inactive in async observers. Without it on
`InboundMessage`, the connector adapter would produce CloudEvents with no
tenant context — invisible to tenant-scoped RAS processing.

So `InboundMessage` gained two fields: `connectorType` (non-null, validated in
the compact constructor) and `tenancyId` (nullable). Every construction site
across connectors and qhorus broke — which was the point. The compiler forced
every connector to declare its type explicitly and pass tenant context through.

## The canonical pattern

Six rules came out of this, now in the garden as a technique entry:

1. **Inject ObjectMapper** — static instances isolate from app config
2. **Null-safe tenancyId** — the CloudEvents SDK serialises null extensions as the literal string `"null"`
3. **Handle `fireAsync()`** — `.exceptionally()` with a WARN log
4. **Catch serialisation errors** — log at WARN, fire with empty data. Never throw from `@ObservesAsync`
5. **Type from semantic field** — not from instance identifiers
6. **WARN severity** — serialisation failure is degraded, not catastrophic

The new `casehub-connectors-cloud-events` submodule lives outside `core` to
preserve core's zero-external-dep invariant. It activates by classpath presence
— add the dependency and inbound messages start appearing as CloudEvents. Remove
it and nothing changes for existing observers.

The interesting part of this work wasn't the adapter itself — it was that
building the third adapter surfaced bugs in the first two that would have been
invisible until casehub-ras started silently dropping events in production.