# Handoff — connectors

## Last Session

Closed connectors#16 (MCP tools for SlackBot): `ConnectorDiscovery` SPI + `DiscoveredTarget`
record in `core`; `SlackBotClient.ID` + `listChannels()` in `slack-bot`; `SlackBotDiscovery`
CDI bean; `SlackBotMcpTool` (`send_slack_bot`) + `ChannelDiscoveryMcpTool` (`list_channels`)
in `mcp`; `@Blocking` fixed on all 7 `@Tool` methods. 190 tests, 0 failures. 5 review rounds
on the spec. 3 new protocols captured. 2 garden entries submitted. Blog: mdp09.

## Immediate Next Step

File the qhorus issue for the `casehub-qhorus-slack-channel` module
(SlackChannelBackend) — qhorus#261 was filed but is the primary tracker;
confirm it has the full scope, then run `work-start` for qhorus#261.

## Cross-Module

**We're blocking:**
- `qhorus` — qhorus#261: `casehub-qhorus-slack-channel` module (SlackChannelBackend).
  Depends on `casehub-connectors-slack-bot` (now in GitHub Packages). · L · Med
- `qhorus` — qhorus#249: `ConnectorMeshBridge` impl in `connector-backend`. Still open. · S · Med

## What's Left

- parent#197 — sync PLATFORM.md capability table for `ConnectorDiscovery` + new tools · XS · Low
- parent#198 — sync casehub-connectors.md deep-dive for slack-bot module + tools · XS · Low
- connectors#17 — log warning when conversations.list is truncated at limit=200 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#261 | casehub-qhorus-slack-channel — SlackChannelBackend | L | Med | Unblock qhorus#249 |
| (new) | MCP tools for SlackBot — `list_slack_channels` with pagination | XS | Low | connectors#17 prerequisite |
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

- Blog: `blog/2026-06-09-mdp09-tools-for-the-bot.md` (published)
- Spec: `docs/specs/2026-06-08-mcp-slack-bot-tools-design.md`
- Protocols: `docs/protocols/connectors/` (3 new: spi-id-method-naming, mcp-tool-blocking-annotation, credential-config-ownership)
- ARC42STORIES.MD: workspace root (no stale entries; §9.2 C3 and §10 will need updating when next epic closes)
