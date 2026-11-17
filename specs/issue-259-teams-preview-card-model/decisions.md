## D1: Builder scope — individual attachment strings

**Choice:** Builder produces individual attachment JSON strings. Consumer collects into `List<String>` and passes to `ComposeResult`.
**Alternatives:**
- Full ComposeResult builder — more ergonomic for the common case but couples builder to response type and prevents mixing attachment types
- Both layers (attachment + response factory) — more API surface for marginal convenience
**Rationale:** Composable. Consumer keeps control of list assembly. Works with mixed attachment types in a single response. Consistent with ComposeResult's existing `List<String>` contract.
**Trade-offs:** Consumer writes one extra line to wrap in `List.of()` and pass to `ComposeResult`. Acceptable — the friction was in building the JSON, not in list construction.
**Sources:** InvokeResponse.ComposeResult, TeamsInboundConnectorTest.java (search result tests)
**Exploration:** quick
**Status:** captured

## D2: Preview card is optional

**Choice:** Preview card is optional on the builder. Calling `.preview(title, subtitle)` adds one; omitting it produces an attachment without preview.
**Alternatives:**
- Required preview — enforces the search-result pattern but blocks non-search use cases (selectItem, link unfurling)
- Two builder types (SearchAttachment + SimpleAttachment) — more type safety but doubles API surface for a rare distinction
**Rationale:** Matches reality — selectItem and queryLink responses don't need a preview card; query search results do. A single builder with optional preview serves all use cases.
**Trade-offs:** No compile-time enforcement that search results include previews. Acceptable — the Teams platform degrades gracefully without previews.
**Sources:** TeamsInboundConnectorTest.java (selectItem test has no preview, query test has preview), Bot Framework composeExtension documentation
**Exploration:** quick
**Status:** captured

## D3: API shape — static factory with builder

**Choice:** `ComposeAttachment.adaptive(cardJson).preview(title, subtitle).build()` — static factory method returning a builder, with optional preview via method chaining.
**Alternatives:**
- Record with `toJson()` and constructor overloads — simpler but positional string ambiguity, doesn't scale if more optional fields appear
- Fully fluent builder with per-field methods including contentType — verbose, re-exposes protocol noise that the builder should eliminate
**Rationale:** Eliminates boilerplate (content type strings are always the same). Handles optional preview cleanly. Static factory `adaptive()` is self-documenting and extensible (room for `thumbnail()`, `hero()` if needed). Follows established builder idiom in this codebase (RichCard.builder()).
**Trade-offs:** Slightly more ceremony than a plain record for the simplest case (no preview). One extra method call: `.build()`. Acceptable for the clarity gained.
**Sources:** RichCard.java builder pattern, InvokeResponse.ComposeResult, issue #259 hypothetical API
**Exploration:** quick
**Status:** captured
