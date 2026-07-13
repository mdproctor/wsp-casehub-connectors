# Handoff — connectors

## Last Session

Closed branch `issue-63-progressive-disclosure-swipe-emoji` — three
features landed as `cec9ce3` on main: progressive disclosure (#63),
emoji reaction palette (#64), swipe-to-reveal gestures (#55). Design
review (5 rounds, $17.92) drove Popover API for emoji picker, keyboard
accessibility for expand, and snap handoff protocol for swipe. 231 tests,
Maven BUILD SUCCESS. Filed #81, #82, #83 for deferred scope items.

## Immediate Next Step

Pick from What's Next. No outstanding blockers.

## What's Left

- #70 Tech debt — a11y mixins, thread root selection, reaction perf · S · Med
- #81 Emoji picker: recently used persistence · XS · Low
- #82 Emoji picker: skin tone preference · XS · Low
- #83 Swipe-to-close (swiping drawer shut) · S · Med
- Delete overdue closed branches (issue-4, 6, 7, 9, 12) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #62 | Dockable contextual panels (Phase 2) | M | Med | Can build on expanded view infrastructure |
| #45 | Teams ChatPlatform implementation | M | Med | Requires Teams Bot API client |

## References

| Doc | Path |
|-----|------|
| ARC42STORIES.MD | `ARC42STORIES.MD` |
| Design spec | `docs/specs/2026-07-12-progressive-disclosure-emoji-swipe-design.md` |
| Qhorus UI spec | workspace `specs/2026-07-07-qhorus-chat-ui-design.md` |
