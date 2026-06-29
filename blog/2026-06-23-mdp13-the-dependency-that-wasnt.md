---
layout: post
title: "The Dependency That Wasn't"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [cloudevents, dependencies, architecture]
series: issue-23-cloudevent-adapter-alignment
---

## The Dependency That Wasn't

The connectors repo had a `cloud-events` submodule that existed for one reason: to keep `casehub-connectors-core` free of casehubio dependencies. The adapter needed `cloudevents-core` for the CloudEvent types, and the only way to get it was through `casehub-platform-api`. A separate module kept that dependency off core's classpath.

Except the adapter doesn't use anything from `casehub-platform-api`. I went through every import — `CloudEvent` and `CloudEventBuilder` come from `cloudevents-core`, which is an independent CNCF standard library. The CDI annotations come from `quarkus-arc`. Jackson comes from Jackson. The adapter depends on `casehub-platform-api` the way you depend on a shopping bag when what you actually need is the groceries inside it.

The fix was obvious once the problem was framed correctly: core depends on `cloudevents-core` directly. No casehubio artifact involved. The "tension" between adapter-in-core and zero-casehubio-deps dissolves — both hold simultaneously because the dependency was never a casehubio dependency in the first place.

This also forced a correction to ARC42STORIES. The §1 quality goal said "Zero-dep core" and §2's constraint said "zero non-JDK runtime dependencies." Both were already wrong — core has depended on `quarkus-arc` since day one. The real invariant was always *no casehubio dependencies*, not zero dependencies. I rewrote both to name the actual invariant: "Lightweight foundation — CDI + external standards only; no casehubio dependencies." The new constraint is testable: grep core's dependency tree for `io.casehub` and verify it returns nothing.

One thing caught during review: the adapter injects `ObjectMapper` via CDI, but `jackson-databind` alone doesn't provide a CDI producer. The webhook module's `@QuarkusTest` augmentation discovered the adapter through Jandex, tried to resolve the `ObjectMapper` injection point, and failed with a class-loading error in a module that didn't change. The fix was `quarkus-jackson` instead of raw `jackson-databind` — it provides the producer without pulling in REST infrastructure.

The whole change is net negative: four commits, 33 lines added, 98 deleted. One submodule gone, one class renamed and moved, six ARC42STORIES sections updated to stop claiming something that was never true.