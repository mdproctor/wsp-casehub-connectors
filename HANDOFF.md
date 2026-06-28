# Handoff — connectors

## Last Session

Closed #26 — demo chat service. Extended ChatPlatform SPI with 3 new
capabilities (ChannelManagement, MemberManagement, MessageHistory),
extended Reactions with list() and Presence with set(). Added ChatBackend
internal storage in chat-ref, refactored RefChatPlatform to delegate to
it. Built chat-demo module (profile-gated -Pdemo) with SqliteChatBackend,
REST endpoints, WebSocket broadcaster, and seed database. Closed #27
(WebSocket endpoint). Filed 3 pages issues (#52-54). Spec went through
4 review passes with architectural improvements at each round.

## Immediate Next Step

Pages issues casehub-pages#52 (WebSocket dataset provider), #53
(multiplexing), #54 (submit action) are filed. When implemented, the
chat-demo Quinoa UI can be wired up. No open connectors issues.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD L7/L8 layer entries need full §9.4 treatment (currently in §4 taxonomy only) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | chat-slack module — SlackChatPlatform adapting SlackBotClient | M | Med | Spec ready; gains from new capabilities |
| — | chat-discord module — DiscordChatPlatform + DiscordClient | L | Med | New HTTP client, gateway for inbound |
| pages#52-54 | Pages WebSocket dataset provider + multiplexing + submit action | L+S+S | Med+Med+Low | Enables chat-demo Quinoa UI |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
