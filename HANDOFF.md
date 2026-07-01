# Handoff — connectors

## Last Session

Implemented platform-agnostic rich content model (#37), Slack Block Kit support
(#41), and list_chat_channels with member count (#42) on one branch. RichCard
record in chat-spi with Builder. Discord and Slack translators. MCP tools
consolidated: send_slack_bot + send_discord → send_chat, list_discord_channels →
list_chat_channels. Design-reviewed (19 issues across 5 rounds, $17.77).
Code-reviewed (1 Critical fixed — Slack include_num_members, 3 Important fixed).
Garden entry GE-20260701-280819 for Slack num_members silent absence. Blog entry
mdp18 published. Doc sync issue parent#335 filed.

## Immediate Next Step

No blocking issues. Pick from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD needs update for RichCard model, send_chat/list_chat_channels tools, Block Kit support — L7/L8 + §4/§5/§12 · M · Med
- Connectors deep-dive (`parent/docs/repos/casehub-connectors.md`) needs RichCard, send_chat, Block Kit updates — parent#335 filed · S · Low
- Minor code review findings (#43) — pagination loop DRY, ReactionResult naming, members parsing, isNotBlank pattern · XS · Low
- Minor code review findings (#48) — DiscordGuild int vs Integer, RecordingBridge duplication, send_chat cardColor/cardFields test coverage · XS · Low
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore from prior commit) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #44 | Inbound rich content parsing — surface embeds/blocks as RichCard | S | Med | Deferred from #37 |
| #46 | send_chat multiple cards per message | XS | Low | JSON array parameter for cards |
| #45 | Teams ChatPlatform implementation with Adaptive Cards | M | Med | Requires Teams Bot API client |
| #28 | Chat demo UI (casehub-pages Quinoa) | M | High | Reopened — blocked on casehub-pages WebSocket |
| #31 | Multi-guild support for Discord | M | Med | Deferred until real use case |
| #32 | Discord slash commands and interactions | M | Med | |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
