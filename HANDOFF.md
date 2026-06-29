# Handoff — connectors

## Last Session

Closed #29 (Discord chat connector — 2 modules, 8 native capabilities,
Gateway inbound, 68 tests) and #35 (replaced JDK WebSocket with Vert.x
for RFC 6455 GUID compliance). Garden entry GE-20260629-c6172a documents
the JDK Corretto WebSocket GUID deviation. Filed 5 deferred issues
(#30–#34). Blog entry mdp15 published.

## Immediate Next Step

No blocking issues. Pick from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD L7/L8 layer entries need full §9.4 treatment · M · Med
- Connectors deep-dive (`parent/docs/repos/casehub-connectors.md`) needs Discord module additions · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #34 | MCP tools — send_discord, list_discord_channels | S | Low | Consumes DiscordClient |
| #30 | Discord message attachment downloading | S | Med | Separate GET per attachment |
| #33 | Discord rich embed message support | S | Med | DiscordClient API extension |
| — | chat-slack module — SlackChatPlatform | M | Med | Spec ready |
| #31 | Multi-guild support | M | Med | Deferred until real use case |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
