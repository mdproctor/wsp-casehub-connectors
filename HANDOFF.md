# Handoff — connectors

## Last Session

Closed connectors#14 (XS — convert `singlePlainTextMessage` to `deliverDirect()`).
Then closed connectors#2 (connectors-side of Slack ChannelBackend): `SlackBotClient` in new
`casehub-connectors-slack-bot` module, `InboundConnectorIds.SLACK_INBOUND` constant,
`slack-ts`/`slack-thread-ts` metadata fields in `SlackInboundConnector`. Two protocols
captured: `shared-http-client`, `inbound-connector-id-constants`. Design spec promoted
to `docs/specs/`, blog entry mdp08 published.

## Immediate Next Step

File the qhorus issue for `casehub-qhorus-slack-channel` module (SlackChannelBackend).
It was deferred at work-end and hasn't been filed yet. Then run `work-start` for that issue.

Scope: `SlackBotBinding`, `SlackBotBindingStore`, `SlackThreadCache`, `SlackInboundNormaliser`,
`SlackChannelBackend`, `SlackBindingResource`, Flyway migration, `ConnectorChannelBackend`
WARN→DEBUG change, Flyway location config. Depends on `casehub-connectors-slack-bot` (published).

## Cross-Module

**We're blocking:**
- `qhorus` — qhorus#249: `ConnectorMeshBridge` impl in `connector-backend`. Still open. · S · Med
- `qhorus` — SlackChannelBackend (issue not yet filed — file first). Depends on `connectors-slack-bot` now in GitHub Packages. · L · Med

## What's Left

- parent#191 — sync `casehub-connectors.md` deep-dive for `slack-bot` module · XS · Low
- connectors#15 — minor polish: WireMock version in parent pom, missing Retry-After test, apiBaseUrl comment · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| (new) | casehub-qhorus-slack-channel module — SlackChannelBackend | L | Med | File qhorus issue first |

## References

- Spec: `docs/specs/2026-06-06-slack-channel-backend-design.md`
- Blog: `blog/2026-06-07-mdp08-slack-bot-client.md` (published)
- ARC42STORIES.MD: workspace root (updated: connectors#14 closed, parent#168 replaced with parent#191)
- Protocols: `docs/protocols/connectors/` (2 new: shared-http-client, inbound-connector-id-constants)
