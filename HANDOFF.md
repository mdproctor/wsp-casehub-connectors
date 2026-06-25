# Handoff — connectors

## Last Session

Closed #24 — ChatPlatform SPI. Resumed from previous session's handover,
completed work-end: code review (1 NOTE fixed — CopyOnWriteArrayList for
thread-safe InMemoryStore), squashed 12→2 commits, rebased to main, pushed.
Journal merged (§10 slack-bot decisions). Design spec posted to #24.

## Immediate Next Step

No open issues. Check `gh issue list --repo casehubio/connectors` or pick
from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD needs a new chapter for the Chat Platform SPI (L7) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | chat-slack module — SlackChatPlatform adapting SlackBotClient | M | Med | Spec ready; needs new SlackBotClient methods |
| — | chat-discord module — DiscordChatPlatform + DiscordClient | L | Med | New HTTP client, gateway for inbound |
| — | chat-irc module — IrcChatPlatform + IrcClient | M | Med | TCP protocol, Ergo for integration tests |
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
