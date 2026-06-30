# Handoff — connectors

## Last Session

Implemented Discord attachment fixes (#36), advanced embed MCP parameters (#38),
and SlackChatPlatform with all 9 native capabilities (#40) on one branch.
Design-reviewed (16 issues across 3 rounds, $13.61). Code-reviewed (1 Important
fixed, 5 Minor filed as #43). Garden entry GE-20260630-d0ea16 for Jakarta JSON
isNull NPE. Blog entry mdp18 published.

## Immediate Next Step

No blocking issues. Pick from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD needs update for chat-slack module, SlackBotClient expansion, Discord attachment/embed enhancements — L7/L8 + §4/§5/§12 · M · Med
- Connectors deep-dive (`parent/docs/repos/casehub-connectors.md`) needs Discord + Slack module additions · S · Low
- Minor code review findings (#43) — pagination loop DRY, ReactionResult naming, members parsing, isNotBlank pattern · XS · Low
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore from prior commit) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #41 | send_slack_bot Block Kit support | S | Med | Slack-native rich content in MCP |
| #42 | list_slack_channels tool with rich detail | XS | Low | Topic, purpose, member count |
| #37 | Platform-agnostic rich content model for ChatPlatform SPI | M | High | Cross-platform abstraction for embeds/blocks/cards |
| #31 | Multi-guild support for Discord | M | Med | Deferred until real use case |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
