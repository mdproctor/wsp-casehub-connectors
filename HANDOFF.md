# Handoff — connectors

## Last Session

Shipped #54 — responsive layout for chat-demo. Phone mode (slide-in drawers
with header bar, backdrop, inert focus trapping, Escape dismiss, auto-close
on channel selection), tablet mode (single sidebar with Channels/Members tab
switcher), desktop (unchanged three-column). Also: auto-expanding textarea
with Shift+Enter multiline, optimistic channel creation, 44 vitest tests.
Design review ran 5 rounds (13 issues, 13 verified). Garden entry
GE-20260702-b1f919 (flex cross-axis overflow gotcha). Filed #55-59 for
deferred work.

## Immediate Next Step

Pick from What's Next. #53 (ARC42STORIES.MD sync) is immediately actionable
and should capture the responsive layout additions.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- #53 ARC42STORIES.MD sync — needs responsive layout, textarea, breakpoint additions · M · Med
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore) · XS · Low
- #59 responsive minor polish — aria-label, member drawer test, constants dedup, afterEach · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #52 | User identity / login for chat-demo | M | Med | Blocked on casehub-pages#88 (dev-auth) |
| #45 | Teams ChatPlatform implementation with Adaptive Cards | M | Med | Requires Teams Bot API client |
| #55 | Swipe-to-reveal gestures for phone drawers | S | Med | Depends on #54 (done) |
| #57 | Touch-specific message interactions (long-press, swipe-to-reply) | S | Med | |
| #31 | Multi-guild support for Discord | M | Med | Deferred until real use case |
| #32 | Discord slash commands and interactions | M | Med | |

## References

| Doc | Path |
|-----|------|
| Spec | `specs/2026-07-02-chat-demo-responsive-layout-design.md` |
| Plan | `plans/attic/issue-54-responsive-layout/2026-07-02-chat-demo-responsive-layout.md` |
| Garden | `GE-20260702-b1f919` — flex cross-axis overflow gotcha |
| Deferred | #55 (swipe), #56 (emoji overflow), #57 (touch), #58 (platform responsive), #59 (minor polish) |
