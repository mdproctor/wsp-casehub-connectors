---
layout: post
title: "Nine SPIs and the Simulation Boundary"
date: 2026-10-05
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [simulation, ref-implementations, audit, seed-data]
---

# Nine SPIs and the Simulation Boundary

The #138 design spec drew a line through every connector SPI: deterministic operations (CRUD, lifecycle, state machines) on one side, interpretive operations (search, routing, recommendations) on the other. The seed data work in #143 shipped YAML files and simulation corpus entries on the strength of that classification. This audit was the verification pass — walk every ref implementation, read the actual code, and check whether the theory holds.

It holds. Across 9 SPIs and roughly 50 capabilities, the classification mapped cleanly to what the code does. ChatPlatform, CalendarPlatform, and BankPlatform are entirely deterministic — every operation is CRUD or lifecycle. LocationPlatform is the opposite: all seven operations are interpretive. The remaining five are mixed, with search as the single interpretive capability in most cases.

The ranking was the interesting part. LocationPlatform stands out not just because every operation is interpretive, but because the ref fallbacks are particularly crude. The directions implementation calculates straight-line distance and divides by a constant speed per travel mode — no actual routing, no waypoints, no traffic. Geocoding does substring matching on addresses. For a demo, "5.2 km straight-line, 6 minutes driving" is visibly wrong in a way that substring-matched search results are not.

CommercePlatform came second. ProductSearch is obviously interpretive — substring match can't handle "noise-cancelling headphones under £300" — but the more interesting finding was about ProductDetails. The ref implementation is a direct map lookup, functionally identical to deterministic CRUD. The "interpretive" classification captures something subtler: a real provider returns recommendations, availability scoring, and dynamic pricing alongside the product record. The ref returns the right shape but none of the enrichment. PlaceDetails in LocationPlatform has the same pattern.

One gap: the design spec classifies email search as interpretive and a corpus file was shipped for it, but `EmailPlatform` has no search method. The corpus references `email-platform.search` which doesn't exist in the interface. Filed as #144.

Three common patterns emerged from looking across all nine implementations. The most extractable is search simulation — five SPIs implement the identical pattern: iterate seed data, substring-match on one to three string fields, return matches. The simulation layer replaces this with corpus-driven responses that add relevance ranking and fuzzy matching. A shared `SubstringSearchFallback` could standardise the ref fallback, though the ref fallback just needs to be structurally valid — the real value is in the corpus.

The demo impact assessment was the most directly useful deliverable. Without simulation, three SPIs are hard to demo convincingly: LocationPlatform (very hard — directions and geocoding are visibly wrong), CommercePlatform (hard — no relevance ranking), and DocumentPlatform (moderate — search only matches filenames, not content). The remaining six produce adequate-to-good demos from seed data alone. BankPlatform in particular looks authentic — UK sort codes, IBANs, realistic merchant names in the transaction history.

The assessment is now a formal document alongside the #138 design spec, covering all four deliverables the issue asked for. The classification was right; the corpus coverage is complete for all six interpretive areas; and the priority ordering gives a clear signal for which SPIs to wire up to the simulation framework first.
