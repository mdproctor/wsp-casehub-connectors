# Decisions — #235 Adaptive Card Action Handling (Invoke Activities)

## D1: Invoke subtypes in scope

**Choice:** Action.Submit (Adaptive Card actions) + task modules (task/fetch, task/submit)
**Alternatives:**
- Action.Submit only — covers the core use case but leaves task module interactions unhandled, requiring a separate issue for the same invoke plumbing.
- All invoke subtypes except composeExtension — over-scoped; subtypes beyond card actions and task modules are either messaging extensions (#236) or rarely used.
**Rationale:** Action.Submit and task modules share the same transport mechanism (invoke activity with name + value payload) and the same handler SPI design. Implementing them together avoids revisiting the invoke routing logic. composeExtension/* is explicitly deferred to #236 (messaging extensions) since it has a fundamentally different interaction model (query/response with preview cards).
**Trade-offs:** Larger implementation surface than Action.Submit alone — task module responses have a different JSON schema from card action responses.
**Sources:** casehubio/connectors#235 (issue body), casehubio/connectors#236 (messaging extensions — separate), #220 design spec §Scope Boundary (line 386)
**Exploration:** quick
**Status:** captured

## D2: Response body construction — Handler SPI

**Choice:** InvokeHandler CDI SPI with InvokeService dispatch, following CommandService/CommandHandler pattern
**Alternatives:**
- Static acknowledge + outbound — always return `{ "statusCode": 200 }`, consumers react via InboundMessage events and send follow-ups. Simple but can't update cards in-place — the HTTP response body is the only mechanism for inline card updates.
- CDI event with mutable response holder — fire an event carrying a response holder, observers set the response body. More implicit, harder to reason about ordering, no clear contract on who "wins" if multiple observers set a response.
**Rationale:** Invoke activities are inherently request-response — the HTTP response body determines what happens in Teams (update the card, show a task module, display a message). This is business logic that the connector can't own. The CommandService/CommandHandler pattern already solves the same structural problem for commands: consumers register handlers, the service dispatches by key, the connector uses the return value. InvokeHandler follows the same shape with invoke-specific types.
**Trade-offs:** New SPI in chat-teams. Consumers must implement InvokeHandler beans — more ceremony than reacting to CDI events. Worth it for the explicit contract and type safety.
**Sources:** chat-spi/src/main/java/.../chat/command/CommandHandler.java (prior art), chat-spi/src/main/java/.../chat/command/CommandService.java (dispatch pattern), connectors-api/.../WebhookResult.java (D4 from #220 — Responded variant)
**Exploration:** quick
**Status:** captured

## D3: No-handler fallback behavior

**Choice:** Acknowledge + FINE log — return `{ "statusCode": 200 }` and log the unhandled invoke name at FINE level
**Alternatives:**
- Reject with warning — return `{ "statusCode": 400 }` or error body. Makes missing handlers visible but disruptive — Teams shows an error to the user for a configuration issue on the server side.
- Ignore entirely — return WebhookResult.Ignored(), no InboundMessage. Invoke activities without handlers become invisible — no observability, no event for downstream processing.
**Rationale:** Graceful degradation. The connector always delivers an InboundMessage (delivery infrastructure obligation) regardless of whether a handler exists. The acknowledge response tells Teams the action was received without error. FINE-level logging avoids noise in production where not all invoke types are handled. Consumers who care about unhandled invokes observe the InboundMessage event.
**Trade-offs:** Silent acceptance could mask misconfiguration — a consumer who expects a handler to be wired up won't see an error. FINE log mitigates this for developers who enable trace logging.
**Sources:** TeamsInboundConnector.java:95-98 (existing default branch — same pattern of logging at FINE and returning Ignored)
**Exploration:** quick
**Status:** captured

## D4: InvokeResponse as sealed interface with typed variants

**Choice:** Sealed interface with variants: Accepted, CardUpdate, Message, TaskModule, Error
**Alternatives:**
- Raw JSON string — consumers construct the response body directly. Maximum flexibility but no compile-time safety, easy to produce invalid responses, no self-documenting API.
- Single record with optional fields — `InvokeResponse(int statusCode, String type, String value)`. Flat but lossy — doesn't enforce which combinations are valid.
**Rationale:** Constrains consumers to valid Bot Framework response shapes. Each variant maps to exactly one response schema: `Accepted` → `{ "statusCode": 200 }`, `CardUpdate` → card replacement, `Message` → text message, `TaskModule` → task module continue/message, `Error` → status code + message. The connector handles JSON serialization — consumers work with typed Java, not JSON strings. Pattern-matching in the connector makes the mapping exhaustive.
**Trade-offs:** Adding new response types (if Bot Framework adds them) requires a new sealed variant — additive but forces recompilation. Acceptable since Bot Framework response types change infrequently.
**Sources:** chat-spi/src/main/java/.../chat/command/CommandResponse.java (sealed Immediate/Deferred — same pattern), connectors-api/.../WebhookResult.java (sealed pattern for transport results)
**Exploration:** quick
**Status:** captured

## D5: Handler SPI location — chat-teams module

**Choice:** InvokeHandler, InvokeService, InvokeRequest, InvokeResponse all in chat-teams module
**Alternatives:**
- chat-spi (platform-agnostic) — invoke activities could theoretically be generalized. But no other platform in the codebase has an equivalent mechanism — Discord interactions use a dedicated JAX-RS endpoint (DiscordInteractionEndpoint), Slack slash commands use a dedicated endpoint (SlackCommandEndpoint). Premature generalization.
- connectors-api — too low-level; these types reference Teams-specific response schemas.
**Rationale:** Invoke activities are a Bot Framework / Teams concept. No other ChatPlatform implementation has equivalent invoke semantics. The handler SPI belongs where the connector lives. If a future platform needs similar invoke handling, the types can be extracted to chat-spi at that point — moving types up is straightforward, premature abstraction is not.
**Trade-offs:** Consumers outside chat-teams cannot implement InvokeHandler without depending on chat-teams. This is correct — only code that understands Teams invoke semantics should handle these activities.
**Sources:** chat-discord/src/main/java/.../discord/DiscordInteractionEndpoint.java (Discord: dedicated endpoint, not SPI), chat-slack/src/main/java/.../slack/SlackCommandEndpoint.java (Slack: dedicated endpoint), CLAUDE.md key rule ("pure delivery infrastructure")
**Exploration:** quick
**Status:** captured

## D6: WebhookResult.Responded variant — pre-validated design from #220

**Choice:** `record Responded(String responseBody, String contentType, List<InboundMessage> messages)` — exactly as D4 from #220 decisions specified
**Alternatives:**
- Extend Challenged — semantically wrong; Challenged means platform verification handshake (Slack url_verification, WhatsApp challenge). Invoke responses are functional responses to user actions.
- Separate variant per response type — e.g., RespondedWithCard, RespondedWithMessage. Over-specialized; the transport layer doesn't need to distinguish response shapes — that's the handler SPI's job.
**Rationale:** This decision was pre-validated in the #220 design process (D4, revised after R1-02). The variant handles the unique requirement of invoke activities: both deliver messages AND return a response body. WebhookRouter handles it by calling `service::receive` for each message, then returning `Response.ok(responseBody).type(contentType).build()`.
**Trade-offs:** Additive sealed variant forces exhaustive switch update in WebhookRouter — this is the intended design of the sealed type.
**Depends on:** D4 from #220 decisions (pre-validated)
**Sources:** connectors-api/.../WebhookResult.java (sealed interface), webhook/.../WebhookRouter.java:92-127 (exhaustive switch), #220 decisions D4 (Responded variant specification, revised after R1-02)
**Exploration:** quick (pre-validated in #220)
**Status:** captured
