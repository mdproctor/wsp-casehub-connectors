# Handoff — connectors

## Last Session

Shipped #49, #50, #51 — chat-demo interactive features. Channel create/delete
with cascade (SPI addition across all platforms), emoji reactions with palette
and pills, Discord-style inline reply threading. Design review ran 4 rounds
(14 issues, 13 verified, 1 accepted). Blog entry mdp19 published.
casehub-pages#88 filed for dev-auth (SmallRye JWT login gate).

## Immediate Next Step

Pick from What's Next. #52 (user identity) is ready once casehub-pages#88 ships.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- #53 ARC42STORIES.MD sync — ChannelManagement.delete, reactions snapshot, threading UI, RichCard, send_chat, Block Kit · M · Med
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore from prior commit) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #52 | User identity / login for chat-demo | M | Med | Blocked on casehub-pages#88 (dev-auth) |
| #45 | Teams ChatPlatform implementation with Adaptive Cards | M | Med | Requires Teams Bot API client |
| #31 | Multi-guild support for Discord | M | Med | Deferred until real use case |
| #32 | Discord slash commands and interactions | M | Med | |

## References

| Doc | Path |
|-----|------|
| Spec | `specs/2026-07-02-chat-demo-interactive-features-design.md` |
| Blog | `blog/2026-07-02-mdp19-making-the-demo-talk-back.md` |
| Pages issue | `casehubio/casehub-pages#88` — dev-auth JWT module |
| ARC42 sync | `casehubio/connectors#53` |
