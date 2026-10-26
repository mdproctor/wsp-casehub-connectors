---
layout: post
title: "Filling in the World"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [simulation, seed-data, testing, life-domain]
---

# Filling in the World

A simulation framework is only as good as the data behind it. casehub-life runs entirely on simulated connector APIs for Phase 1 — and up until today, every ref implementation was populated with generic placeholders. Alice at example.com. Acme Corp salary. "Q3 Planning Session." The kind of seed data you write to get the tests green, not to test whether an agent can actually reason about a household.

The fix was straightforward but wide: replace every ref module's seed data with a single coherent household. The Alpha family — Jonny, Sarah, two kids, Grandma Jean in Tunbridge Wells, Dr Patel at the GP surgery, Bob the plumber, a mortgage at Halifax, car insurance approaching renewal at Admiral. A year of bank transactions showing the energy bill creeping up. School events for the kids. A kitchen tap that needs replacing.

What makes this interesting isn't the data itself — it's the cross-platform referential integrity. Dr Patel appears in contacts, calendar (GP appointment for Tom), email (appointment confirmation), location (Elm Street Surgery with opening hours), and bank (prescription payment). Admiral insurance appears in bank transactions, email (renewal quote at £612, up from £576), commerce (comparison products from Direct Line and Churchill), calendar (renewal deadline), and project tracking (a task to compare before the deadline). Bob's Plumbing has a contact card, a calendar visit, an email quote, a bank deposit, and an invoice in document storage.

This matters because the whole point of casehub-life is avatar-driven automation across these platforms. An agent that sees the insurance renewal email should be able to check the bank history for last year's premium, find comparison quotes in commerce, and flag the calendar deadline — all using the same entity. If the seed data doesn't share entities across SPIs, the cross-platform reasoning that makes the product useful can't be tested.

Nine SPIs, forty-five files changed. Calendar and chat needed new SeedLoader infrastructure (YAML files plus Jackson deserialization); the other seven already had SeedLoaders from earlier work and just needed the data replaced. The pattern is identical across all nine: a `SeedLoader` class with a static `ObjectMapper`, records for the seed shape, and classpath resource loading. Mechanical, but the kind of mechanical that compounds — every ref module now boots with a lived-in household instead of a test fixture.

One thing that surfaced late: the original design spec used a real name for the household head. Caught mid-session and renamed to Jonny Alpha across all seed files. The kind of thing that's trivial to fix early and painful to fix after it's merged into three downstream repos.

The ProjectPlatform MCP tools turned out to be already implemented — `ConnectorProjectApi` with twenty endpoints covering issues, labels, milestones, comments, and boards was sitting in the graphql module, fully annotated with `@McpDomain` and wired into the domain filter. The only gap was test coverage, which was a single file following the established mock pattern from the other API tests.

The seed data gap that remains: the design spec calls for simulation corpus entries (search results, directions) that sit in the strategy layer above the ref. That's a separate concern — the ref layer is now complete, and the simulation layer can reference the same household data without duplicating it.
