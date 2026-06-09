# Handoff — connectors

## Last Session

Closed connectors#17 (XS): `parseChannels()` now logs a WARNING when
`response_metadata.next_cursor` is non-empty — workspace with >200 channels
will know the list was cut. Three test cases (truncated, no meta, empty meta).
Also stamped EPIC-CLOSED.md on issue-16 (missed at last work-end). Blog: mdp10.

## Immediate Next Step

File the qhorus issue for the `casehub-qhorus-slack-channel` module
(SlackChannelBackend) — qhorus#261 was filed but is the primary tracker;
confirm it has the full scope, then run `work-start` for qhorus#261.

## Cross-Module

**We're blocking:**
- `qhorus` — qhorus#261: `casehub-qhorus-slack-channel` module (SlackChannelBackend).
  Depends on `casehub-connectors-slack-bot` (now in GitHub Packages). · L · Med
- `qhorus` — qhorus#249: `ConnectorMeshBridge` impl in `connector-backend`. Still open. · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#261 | casehub-qhorus-slack-channel — SlackChannelBackend | L | Med | Unblock qhorus#249 |
| (new) | MCP tools for SlackBot — `list_slack_channels` with pagination | XS | Low | File issue first |
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

- Blog: `blog/2026-06-09-mdp10-logging-the-limit.md` (published)
- Spec: `docs/specs/2026-06-08-mcp-slack-bot-tools-design.md`
- Protocols: `docs/protocols/connectors/` (3 new this epic: spi-id-method-naming, mcp-tool-blocking-annotation, credential-config-ownership)
