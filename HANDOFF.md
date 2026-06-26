# Handoff — connectors

## Last Session

Closed #25 — chat-irc module. First concrete ChatPlatform adapter (IRC).
Spec went through three review passes (L2 bypass caught, @PostConstruct
blocking caught, RFC 1459 AWAY misconception caught). Implementation via
subagent-driven development — 4 tasks, 41 tests, squashed 9→1 commit.
3 garden entries submitted (no Java IRC server, CompletableFuture collector,
RFC 1459 AWAY).

## Immediate Next Step

No open issues. Check `gh issue list --repo casehubio/connectors` or pick
from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD needs new chapters for Chat Platform SPI (L7) and Chat IRC · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | chat-slack module — SlackChatPlatform adapting SlackBotClient | M | Med | Spec ready; needs new SlackBotClient methods |
| — | chat-discord module — DiscordChatPlatform + DiscordClient | L | Med | New HTTP client, gateway for inbound |
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
