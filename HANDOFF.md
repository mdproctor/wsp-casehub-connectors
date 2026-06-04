# Handoff — connectors

## Last Session

connectors#1 shipped. Expanded scope from XS doc task to full MCP tool surface:
`casehub-connectors-mcp` submodule with five `@Tool` beans (`send_slack`, `send_teams`,
`send_sms`, `send_whatsapp`, `send_email`), `ConnectorMeshBridge` SPI in `core` (no-op
default, Qhorus bridge deferred to qhorus#249), WhatsApp template + language support.
16 → 7 commits squashed and pushed to casehubio/connectors main.

## Immediate Next Step

Run `work-start` for connectors#14 — convert `singlePlainTextMessage_deliveredWithCorrectFields`
to `deliverDirect()`. XS · Low, quick win before starting #13.

## Cross-Module

**We're blocking** (Qhorus needs this before bridge can land):
- `qhorus` — qhorus#249: `ConnectorMeshBridge` impl in `connector-backend`. SPI is in `core`,
  no-op default ships. Bridge posts `EVENT` to observe channel when case context active. · S · Med

**Peer docs outstanding:**
- parent#168 — sync `casehub-connectors.md` deep-dive + PLATFORM.md for `mcp` module and
  `ConnectorMeshBridge` SPI. Filed this session. · XS · Low

## What's Left

- connectors#14 — convert `singlePlainTextMessage_deliveredWithCorrectFields` to `deliverDirect()` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| connectors#14 | Convert test to deliverDirect() | XS | Low | Start here |
| connectors#13 | Migrate DESIGN.md → ARC42STORIES.MD (foundation tier) | M | Low | devtown ARC42STORIES.MD is structural reference |
| connectors#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |

## References

- Blog: `blog/2026-06-04-mdp06-opening-connectors-to-llm-ecosystem.md`
- Issues filed: connectors#14, qhorus#249, parent#168
- Issues closed: connectors#1
- Garden entries: GE-20260604-81a6a6 (@Unremovable cross-module), GE-20260604-917790 (single-pass sanitizer), GE-20260604-2f0889 (@Alternative @All collision)
- Protocol: PP-20260604-c0a86d (MCP tool exception catch-all)
- ADR: docs/adr/0003-connector-mesh-bridge-spi-in-core.md
