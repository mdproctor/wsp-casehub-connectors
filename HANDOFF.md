# Handoff — connectors

## Last Session

Closed branch `issue-39-discord-review-multi-guild` — removed
`casehub.discord.guild-id` config, made DiscordClient a stateless HTTP
transport (guild-id as parameter), added `listBotGuilds` with cursor
pagination, multi-guild support across DiscordChatPlatform,
DiscordDiscovery, and DiscordInboundConnector. Fixed hardcoded guild-id
bug in inbound events. Made CDN hosts configurable. PR #85 to upstream.
Closes #39, #31.

## Immediate Next Step

Pick from What's Next. No outstanding blockers. PR #85 awaits merge.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12, 39) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | Teams ChatPlatform implementation | M | Med | Requires Teams Bot API client |
| #60 | Adopt casehub-pages-push typed protocol SDK | M | Med | — |
| #58 | Responsive layout primitives for pages-runtime | L | High | Cross-module design needed |
| #32 | Discord slash commands and interactions | M | Med | — |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
