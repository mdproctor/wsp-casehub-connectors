# Design Journal — issue-16-mcp-slack-bot-tools

### 2026-06-08 · §Module Structure

Added `casehub-connectors-slack-bot` to the Module Structure table — new submodule shipping
`SlackBotClient` (Slack Web API HTTP client) and `SlackBotDiscovery` (`ConnectorDiscovery`
implementation). The module is separate from `core` because bot-specific HTTP client code
would pollute the zero-dep `core` module consumed by all downstream repos.

Module Structure table gains a row: `slack-bot | casehub-connectors-slack-bot | SlackBotClient
(chat.postMessage + conversations.list) · SlackBotDiscovery (ConnectorDiscovery SPI impl)`.

The `mcp` row is updated: `casehub-connectors-mcp | ... | send_slack_bot, list_channels` added
to tool surface; depends on `core + email + slack-bot + quarkus-mcp-server-core`.

### 2026-06-08 · §SPI

Two new types added to `core`:

**`DiscoveredTarget`** — top-level record `(id, displayName)` in `io.casehub.connectors`.
Top-level (not nested in `ConnectorDiscovery`) because nested records inside interfaces are
unconventional for public SPIs and force external implementors to qualify every reference
with the interface prefix.

**`ConnectorDiscovery`** — optional SPI in `io.casehub.connectors`. CDI `@ApplicationScoped`
beans; `@All List<ConnectorDiscovery>` collects all implementations (same pattern as
`ConnectorService(@All List<Connector>)`). Method `id()` (not `connectorId()`) — consistent
with `Connector.id()` and `InboundConnector.id()` platform convention. `ChannelDiscoveryMcpTool`
aggregates all registered implementations for `list_channels`. Per-discovery `try/catch` in
the tool prevents a throwing implementation from silencing others.

### 2026-06-08 · §Usage

MCP tool surface expanded. The `casehub-connectors-mcp` module now exposes seven tools (was
five). Two additions:

`send_slack_bot(channel, text, threadTs?)` — posts via `SlackBotClient` using a configured
bot token (`casehub.connectors.slack-bot.token`). Returns `"Posted to <channel> (ts=<ts>)"`
on success; the `ts` enables thread replies in subsequent calls. Does NOT route through
`ConnectorService` because `Connector.send()` is void — routing through the SPI would discard
the `ts`. Bypasses `ConnectorService` directly to `SlackBotClient`; bridge is called on
`ok=true` only.

`list_channels()` — discovers delivery targets across all registered `ConnectorDiscovery` CDI
beans. Returns formatted text (`<connectorId>:\n  <displayName> (<id>)\n...`). Callers use
the IDs with `send_slack_bot`. "No channels discovered." when all beans return empty lists.

`@Blocking` annotation added to all seven `@Tool` methods — existing five tools were missing
it, which would have blocked the Vert.x I/O thread on HTTP calls (GE-20260604-96d82a).
