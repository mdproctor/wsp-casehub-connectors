# Handoff — connectors

## Last Session

Housekeeping session. Resumed from handover, discovered qhorus#261 (`SlackChannelBackend`)
was CLOSED in a separate session — removed from backlog. Attempted work-end: no active
epic branch, nothing to close. Epic hygiene: 14 workspace branches all have
`EPIC-CLOSED.md`; branches issue-4, 6, 7, 9, 12 are past their scheduled deletion dates.
ARC42STORIES.MD stale scan: clean.

## Immediate Next Step

Check open issues across connectors and qhorus to pick the next piece of work:
`gh issue list --repo casehubio/connectors && gh issue list --repo casehubio/qhorus`

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past their 14-day hold · XS · Low
- Commit or discard `specs/2026-06-17-slack-channel-backend-design.md` (untracked in workspace) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
