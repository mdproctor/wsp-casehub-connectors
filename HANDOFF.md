# Handoff — connectors

## Last Session

Closed #29 — Discord chat connector. Two new modules: `discord` (shared
HTTP client for REST API v10 + Gateway WebSocket) and `chat-discord`
(ChatPlatform SPI with 8 native capabilities + Gateway-driven inbound).
68 tests. Adversarial design review (4 rounds) refined the spec — caught
forum channel exclusion, blank-config fail-soft, content length limit,
frame accumulation, and message type filtering. Discovered JDK Corretto
WebSocket GUID deviation from RFC 6455 (GE-20260629-c6172a). Filed 5
deferred issues (#30–#34).

## Immediate Next Step

No open connectors issues blocking. Pick from What's Next, or address
the JDK WebSocket GUID production blocker if Discord Gateway deployment
is needed before a JDK fix ships.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD L7/L8 layer entries need full §9.4 treatment · M · Med
- JDK Corretto WebSocket GUID (`...B11` vs RFC `...B63`) blocks production Gateway connections · S · Med
- connectors deep-dive (`parent/docs/repos/casehub-connectors.md`) needs Discord module additions · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #34 | MCP tools — send_discord, list_discord_channels | S | Low | Consumes DiscordClient from discord module |
| #30 | Discord message attachment downloading | S | Med | Separate GET per attachment |
| #33 | Discord rich embed message support | S | Med | DiscordClient API extension |
| — | chat-slack module — SlackChatPlatform adapting SlackBotClient | M | Med | Spec ready; gains from new capabilities |
| #31 | Multi-guild support — one ChatPlatform instance per guild | M | Med | Deferred until real use case |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
