# HANDOFF — casehub-connectors

## Last Session

Completed 6 issues (#282–#287) — extended the `CardBuilder` fluent API to full parity with the adaptive `Builder`, migrated a ref handler, and closed two defensive-copy gaps.

**#282 — CardBuilder `text()` for body content:**
- Added `text(String)` to `CardBuilder` for hero/thumbnail cards
- Updated shared `cardContent()` helper to accept and emit `text` between subtitle and images

**#283 — Ref handler builder migration:**
- Migrated `RefLinkUnfurlHandler` from raw AdaptiveCard JSON to `hero().text().button()`
- `RefSearchHandler` and `RefFetchTaskHandler` stay on `adaptive()` — they need AdaptiveCard features

**#284 — CardBuilder `preview()` support:**
- Added `preview(title)`, `preview(title, subtitle)` to `CardBuilder`
- Emits separate thumbnail preview block, matching adaptive `Builder.preview()` pattern

**#285 — Defensive metadata copy (InvokeRequest):**
- `InvokeRequest` compact constructor wraps metadata in `Map.copyOf()`

**#286 — CardBuilder `preview(title, subtitle, imageUrl)` overload:**
- Full API symmetry with adaptive `Builder.preview()`
- Existing 1-arg and 2-arg overloads now chain through the 3-arg variant

**#287 — Defensive metadata copy (InboundMessage):**
- `InboundMessage` compact constructor wraps metadata in `Map.copyOf()`, null becomes `Map.of()`
- Matches `InvokeRequest` pattern from #285

**Blog:** "The Builder That Grew" — API design diary covering the builder evolution, shared helper factoring, and defensive copy pattern.

## Branch State

All work on main. No feature branches open. Published to origin.

## Queue

Queue drained — 19/19 items complete. No remaining work.
