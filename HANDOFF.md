# Handoff — connectors

## Last Session

Shipped #52 — user identity for chat-demo. JWT auth via casehub-pages-auth
dependency, `@Authenticated` REST, `HttpUpgradeCheck` WebSocket auth,
identity-based messaging bypassing Messaging/Threading SPIs, auto-membership,
presence auto-create. Frontend login gate (`<pages-dev-auth>`), identity
widget (`<pages-identity>`), `authenticatedFetch()` with Bearer headers.
Design review ran 2 rounds (14 issues, all resolved). Garden entry
GE-20260703-e4a6b0 (Quarkus WebSockets Next ignores JAX-RS filters gotcha).
Cross-repo: `pages-auth-success` event added to casehub-pages (8a6bc9d).

## Immediate Next Step

Pick from What's Next. #53 (ARC42STORIES.MD sync) is immediately actionable
and should capture the identity additions alongside the responsive layout
work from the previous session.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- #53 ARC42STORIES.MD sync — needs identity, responsive layout, textarea, breakpoint additions · M · Med
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore) · XS · Low
- #59 responsive minor polish — aria-label, member drawer test, constants dedup, afterEach · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | Teams ChatPlatform implementation with Adaptive Cards | M | Med | Requires Teams Bot API client |
| #55 | Swipe-to-reveal gestures for phone drawers | S | Med | Depends on #54 (done) |
| #57 | Touch-specific message interactions (long-press, swipe-to-reply) | S | Med | |
| #31 | Multi-guild support for Discord | M | Med | Deferred until real use case |
| #32 | Discord slash commands and interactions | M | Med | |

## References

| Doc | Path |
|-----|------|
| Spec | `specs/2026-07-03-chat-demo-user-identity-design.md` |
| Garden | `GE-20260703-e4a6b0` — Quarkus WebSockets Next ignores JAX-RS filters |
| Deferred | #55 (swipe), #56 (emoji overflow), #57 (touch), #58 (platform responsive), #59 (minor polish) |
