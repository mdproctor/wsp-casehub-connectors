## D1: Messaging extension types in scope

**Choice:** All three — search-based (query), action-based (fetchTask/submitAction), and link unfurling (queryLink), plus selectItem
**Alternatives:**
- Search + link unfurling only — covers most common cases but leaves action-based incomplete
- Search only — minimal scope but incomplete user experience
**Rationale:** All three share the same response model and dispatch infrastructure. Incremental cost of each additional type is low once the response variants exist.
**Trade-offs:** Larger scope, but complexity is in the response model (D3), not the type count
**Sources:** Issue #236 body, Bot Framework composeExtension documentation
**Exploration:** quick
**Status:** captured

## D2: Response model — extend existing InvokeResponse

**Choice:** Add new variants to the existing InvokeResponse sealed interface
**Alternatives:**
- Separate MessagingExtensionResponse sealed type — cleaner separation but requires a second dispatch path or wrapping, adds complexity for no functional gain
**Rationale:** composeExtension activities are invoke activities. The InvokeHandler SPI dispatches by name and returns InvokeResponse. A second sealed type would force either dual return types on InvokeHandler or an adapter layer. The existing pipeline handles everything uniformly.
**Trade-offs:** InvokeResponse grows from 6 to 9 variants — still manageable for a sealed interface
**Sources:** TeamsInboundConnector.handleInvoke(), InvokeHandler.java, InvokeService.java, InvokeResponse.java
**Exploration:** quick
**Status:** captured

## D3: Response variant granularity — 3 new variants

**Choice:** Three new InvokeResponse variants: ComposeResult, ComposeAuth, ComposeContinue
**Alternatives:**
- 5+ fine-grained variants (one per use case) — more self-documenting but most share the same JSON shape, adding redundancy
- 1 generic variant ComposeExtension(type, bodyJson) — maximum flexibility but pushes serialisation knowledge onto consumers
**Rationale:** Maps to the 3 distinct JSON response shapes in the Bot Framework protocol. ComposeResult covers search results, link unfurling, selectItem, and submitAction results (all produce `{composeExtension: {type: "result", ...}}`). ComposeContinue covers task module responses. ComposeAuth covers auth flow initiation.
**Trade-offs:** Less self-documenting than 5+ variants — a ComposeResult can be search, unfurl, or action result. Javadoc on the variant clarifies usage.
**Sources:** Bot Framework composeExtension response schema, InvokeResponse.java, InvokeResponseSerializer.java
**Exploration:** quick
**Status:** captured

## D4: Attachment model — raw JSON strings

**Choice:** Raw JSON strings for card attachments, consistent with existing CardUpdate(String cardJson) pattern
**Alternatives:**
- Typed attachment records (ComposeAttachment record) — more structure but couples to Adaptive Card schema
- Builder API — ergonomic but heavy coupling, violates delivery-infrastructure-only rule
**Rationale:** This library is pure delivery infrastructure. Callers build card JSON however they want. Raw strings avoid imposing any opinion on card structure and are consistent with the established pattern.
**Trade-offs:** No compile-time safety on card structure — callers can pass malformed JSON. Acceptable for infrastructure code.
**Sources:** InvokeResponse.CardUpdate, CLAUDE.md key rule ("do not add business logic")
**Exploration:** quick
**Status:** captured

## D5: Auth flow responses in scope

**Choice:** Include ComposeAuth response variant for SSO/OAuth auth flows
**Alternatives:**
- Defer auth to follow-up issue — reduces scope but leaves auth-requiring extensions non-functional
**Rationale:** Without auth responses, messaging extensions that require authentication can't return the proper Bot Framework auth flow. Low incremental cost — it's one additional InvokeResponse variant and serializer case.
**Trade-offs:** None significant — auth response is a simple JSON shape
**Sources:** Bot Framework messaging extension auth documentation
**Exploration:** quick
**Status:** captured
