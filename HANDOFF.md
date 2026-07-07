# Handoff — connectors

## Last Session

Designed and implemented the notification UI component system. Brainstormed
the full notification architecture (platform routing, subscriptions, target
resolution, mute/snooze, channel preferences) → filed epic platform#147 with
7 child issues. Wrote notification UI spec, ran adversarial design review
(4 rounds, 15 issues, all resolved). Built 5 components in blocks-ui:
notification-bell, notification-inbox, subscription-list + types/API/events.
Example showcase page with mock SSE, demo controls, 345+ tests. Fixed
data-table: row-click activates (not selects), zebra striping, column
alignment, mode switcher in dropdown. Squashed and pushed to blocks-ui main.

## Immediate Next Step

Pick from What's Next. #53 (ARC42STORIES.MD sync) should capture the
notification architecture alongside previous responsive layout work.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low
- #53 ARC42STORIES.MD sync — needs identity, responsive layout, notification architecture additions · M · Med
- casehubio.github.io push blocked by pre-push hook (unrelated .gitignore) · XS · Low
- #59 responsive minor polish — aria-label, member drawer test, constants dedup · S · Low
- Notification spec modified on connectors main (design review changes) — commit the final version · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | subscription-editor component (blocks-ui) | M | Med | Event type picker + constraint builder. Depends on platform#155 (EventTypeRegistry) |
| — | subscribe-button component (blocks-ui) | S | Med | Contextual entity/filter subscribe |
| — | pages-summary-bar primitive proposal | S | Low | File issue on casehub-pages, propose API |
| #45 | Teams ChatPlatform implementation | M | Med | Requires Teams Bot API client |
| #55 | Swipe-to-reveal gestures for phone drawers | S | Med | Depends on #54 (done) |
| #57 | Touch-specific message interactions | S | Med | |

## References

| Doc | Path |
|-----|------|
| Notification UI spec | `specs/2026-07-06-notification-ui-components-design.md` |
| Implementation plan | `plans/2026-07-06-notification-inbox-ui.md` |
| Design review | `~/adr/casehub-connectors/notification-ui-components-20260706-102246/` |
| Platform epic | casehubio/platform#147 |
| Pages SSE issue | casehubio/casehub-pages#131 (done) |
| blocks-ui branch | `issue-146-notification-inbox` (closed, landed as 9ac2f80) |
