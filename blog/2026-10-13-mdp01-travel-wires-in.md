---
layout: post
title: "Travel wires in"
date: 2026-10-13
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [travel, mcp-domain, wiring, pattern]
---

# Travel wires in

Phase 4 of the Real-World Knowledge Platform epic. TravelPlatform — transport search, accommodation, bookings — wired into the `@McpDomain` surface through `ConnectorTravelApi`. Eleven endpoints covering all four capabilities: TransportSearch, TransportBooking, AccommodationSearch, AccommodationBooking.

The interesting thing about this one is what it didn't require. Commerce (#162) established the wiring pattern two days ago. Travel is the fourth platform to follow it (after contacts, project, and commerce). By now the pattern is mechanical: capability-gated API class, `requireCapability` with `UnsupportedCapabilityException`, `connectorsReport` integration, CDI wiring through `GraphqlBeans`, and the test structure — stub with boolean capability flags, full stub with inline implementations, unsupported-throws and happy-path-delegates.

The one wrinkle was Java records and interface method name collisions. `TravelPlatform` has a `transportSearch()` accessor that returns a `TransportSearch` capability interface. If you name a record component `transportSearch` (the boolean flag), the record's auto-generated accessor clashes with the interface method. Commerce didn't hit this because its capability names (`ProductSearch`, `Cart`, `Checkout`) don't overlap with method names on `CommercePlatform`. The fix was trivial — shorter component names — but it's the kind of thing that only surfaces at compile time.

Nine platforms now report through `connectorsReport` (chat, calendar, bank, email, document, contacts, project, commerce, travel). Location is the tenth SPI but hasn't been wired into the report yet. The `ALL_SCOPES` set has nine entries; CLAUDE.md's "10 platforms" count includes location. That'll be a separate issue.

The pattern maturity is the story here. When we built the first `@McpDomain` API (chat), the shape wasn't obvious — capability introspection, user-scoped accessors, the `requireCapability` helper, the test stub taxonomy. By Phase 4, it's a template. The next platform SPI wiring should take the same shape with no design decisions at all.
