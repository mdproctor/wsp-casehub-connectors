# Handoff — connectors

*Updated: #28 closed — removed from backlog.*

## Last Session

Investigated what Pages WebSocket completion (pages#52, #53) unlocks.
Found connectors #27 was already done but GitHub missed the close —
sub-issue `Closes` refs lost during git-squash. Fixed two work-end
skill gaps in soredium (c6fce1b): carry sub-issue refs through squash,
stamp project branch on close. Manually closed #27 and stamped
`issue-26-demo-chat-service`. Blog entry mdp16 published.

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
