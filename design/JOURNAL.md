# Design Journal — issue-2-slack-channel-backend

### 2026-06-07 · §10

Three architectural decisions captured in this commit:

**casehub-connectors-slack-bot as a separate submodule, not an extension of core.** The Slack Bot API client (`SlackBotClient`) lives in its own Maven module rather than in `casehub-connectors-core`. The `core/` module is a dependency of every downstream repo; adding a bot-specific HTTP client there would pollute the classpath of all consumers with a capability most of them don't use. The `email-inbound/` module is the precedent — heavier, purpose-specific inbound infrastructure lives in its own module and activates by classpath presence. Future growth of the bot client (Block Kit, reactions, file uploads) belongs in this module, not in `core/`.

**Shared HttpHelper.CLIENT for bot HTTP calls.** `SlackBotClient` uses `HttpHelper.CLIENT` (the shared `java.net.http.HttpClient` instance defined in `connectors-core`) rather than creating its own. Creating a second singleton would produce two independent connection pools, two connect-timeout configurations, and inconsistent behaviour across connector types. The shared client already has a 5 s connect timeout and is designed for reuse across all connectors.

**Jakarta JSON builder for payload construction.** `SlackBotClient.buildPayload()` uses `Json.createObjectBuilder()` from the Jakarta JSON API (already on the classpath via `core/`) rather than `StringBuilder` with `HttpHelper.jsonQuote()`. Manual string building is fragile under charset edge cases and diverges from the pattern used in the rest of the codebase (MCP tools, SlackInboundConnector). The builder approach is consistent, safe, and removes the `HttpHelper` import from `SlackBotClient` entirely — the bot client has no reason to depend on webhook utility code.
