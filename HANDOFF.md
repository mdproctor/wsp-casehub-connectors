# HANDOFF — casehub-connectors

## Last Session

Adaptive Card action handling (#235) — invoke activities (Adaptive Card `Action.Submit`, task module `task/fetch`, `task/submit`) now have full support through three layers: a new `WebhookResult.Responded` transport variant, an `InvokeHandler` CDI SPI for response construction, and invoke routing in `TeamsInboundConnector`.

**Key architectural addition:** `WebhookResult.Responded` is the sixth sealed variant — the only one that both delivers `InboundMessage`s AND returns an HTTP response body. The router handles it in a single operation: deliver messages to observers, return body to caller.

**InvokeHandler SPI (chat-teams):**
- `InvokeHandler` — CDI bean interface: `name()` returns the invoke activity name, `handle(InvokeRequest)` returns `InvokeResponse`
- `InvokeService` — discovers handlers via `@All List<InvokeHandler>`, dispatches by name, catches exceptions (falls back to `Accepted()`), detects duplicate handler names at startup
- `InvokeResponse` — sealed interface with 6 variants constraining consumers to valid Bot Framework response schemas: `Accepted`, `CardUpdate`, `Message`, `TaskModuleContinue`, `TaskModuleMessage`, `Error`
- `InvokeResponseSerializer` — package-private, maps variants to Bot Framework JSON (two different envelopes: `statusCode/type/value` for card actions, `task` wrapper for task modules)

**TeamsInboundConnector changes:**
- New `case "invoke"` in the activity type switch
- `handleInvoke` constructs InboundMessage (always delivered), dispatches to InvokeService, serializes response with guard try-catch (falls back to Accepted on serialization failure)
- 6-arg constructor with `InvokeService` (5-arg chains to it with null for backward compat)

**TeamsBeans changes:**
- New `InvokeService` producer via `@All List<InvokeHandler>` CDI discovery
- `teamsInboundConnector` producer now injects `InvokeService`

**Commits (3, squashed from 6):**
- `d0bc67a6` — WebhookResult.Responded variant + WebhookRouter handling
- `45dd2338` — InvokeHandler SPI with typed response serialization
- `bcdb85cb` — wire invoke activity handling in TeamsInboundConnector
- `ba2b27f1` — docs: add Responded HTTP mapping to WebhookResult Javadoc

**Issues closed this session:**

| # | Title | What changed |
|---|-------|-------------|
| #235 | Adaptive Card action handling invoke activities | WebhookResult.Responded, InvokeHandler/InvokeService SPI, InvokeResponse sealed variants, TeamsInboundConnector invoke routing |

**Issues filed this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #254 | docs: update CLAUDE.md, ARC42STORIES, guides for Responded and InvokeHandler | S | Low |
| #255 | chat-teams: InvokeHandler integration test | S | Low |

**Queue state after this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #236 | Messaging extensions support | L | High |

**To resume:** `work continue` for #236 (next in queue). #236 builds on the `WebhookResult.Responded` and invoke infrastructure from #235, extending it to `composeExtension/*` activity subtypes.

**Blog:** "When the Card Talks Back" — covers the invoke activity design, the tension between pure delivery infrastructure and request-response semantics, and why Discord solved it differently.
