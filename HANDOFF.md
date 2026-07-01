# Handoff — connectors

## Last Session

Shipped #28 — casehub-pages Quinoa UI for chat-demo. Three-column workspace with
dockable side panels (dock bar toggles channels/members), message grouping, scroll
anchoring, presence indicators, dark mode. Wire protocol updated to match casehub-pages
spec (op/columns/seq/string coercion/membershipId/presence snapshot). Profile-gated
build: `-Pdemo` = backend only, `-Pdemo -Pui` = full stack with Quinoa.
Design review ran 4 rounds (26 issues, all resolved). Garden entry GE-20260701-b84dd2
captured Quinoa profile-gating gotcha.

## Immediate Next Step

No blocking issues. Pick from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- ARC42STORIES.MD needs update for RichCard model, send_chat/list_chat_channels tools, Block Kit support, chat-demo UI — L7/L8 + §4/§5/§12 · M · Med
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore from prior commit) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #49 | Thread view / reply UI for chat-demo | S | Low | Threading API exists, UI not surfaced |
| #50 | Reaction UI for chat-demo | S | Low | Backend broadcasts reactions, no UI |
| #51 | Channel creation UI for chat-demo | XS | Low | REST endpoint exists |
| #52 | User identity / login for chat-demo | M | Med | Currently all messages from "ref" |
| #45 | Teams ChatPlatform implementation with Adaptive Cards | M | Med | Requires Teams Bot API client |
| #31 | Multi-guild support for Discord | M | Med | Deferred until real use case |
| #32 | Discord slash commands and interactions | M | Med | |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
