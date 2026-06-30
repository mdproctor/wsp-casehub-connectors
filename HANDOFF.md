# Handoff — connectors

## Last Session

Implemented three Discord enhancements on one branch (#30, #33, #34):
attachment downloading with SSRF defense and streaming size enforcement,
rich embed outbound support, and send_discord/list_discord_channels MCP tools.
Design-reviewed (19 issues across 6 rounds). Code-reviewed (4 Important fixed,
3 Minor filed as #39). Garden entry GE-20260630-e18bed for BodyHandlers gotcha.
Blog entry mdp17 published.

## Immediate Next Step

No blocking issues. Pick from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD L7/L8 layer entries need full §9.4 treatment · M · Med
- ARC42STORIES.MD needs §4/§5/§12 updates for Discord enhancements (MCP tools, module deps, risks) · S · Low
- Connectors deep-dive (`parent/docs/repos/casehub-connectors.md`) needs Discord module additions · S · Low
- Minor code review findings (#39) — allowedCdnHosts configurability, null URL test, HTTP/1.1 connection reuse · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #36 | Discord message history attachment downloading | S | Low | DiscordMessage already carries metadata |
| #38 | Advanced embed MCP parameters (fields, images, thumbnails) | S | Low | DiscordEmbed model already supports full field set |
| — | chat-slack module — SlackChatPlatform | M | Med | Spec ready |
| #31 | Multi-guild support | M | Med | Deferred until real use case |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
