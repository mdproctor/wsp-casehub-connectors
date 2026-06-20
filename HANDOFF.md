# Handoff — connectors

*Updated: #21 closed — removed from backlog. qhorus#261 closed — removed from backlog.*

## Last Session

Closed connectors#21 (L): `ARC42STORIES.MD` created — 938 lines, 5 chapters, 5
layers. Migrated from DESIGN.md, 3 ADRs, 8 design specs, and mdp01–mdp11 diary
entries. qhorus#249 stale reference caught and corrected at write time.
`java-update-design` now routes to `ARC42STORIES.MD §10`. Blog: mdp12.

## Immediate Next Step

qhorus#261 (`SlackChannelBackend`) is CLOSED. Check what issues remain open across
connectors and qhorus to determine the next piece of work. Run `gh issue list --repo casehubio/connectors` and `gh issue list --repo casehubio/qhorus`.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## Cleaned up

- `qhorus#261 — feat(slack-channel): implement casehub-qhorus-slack-channel module (SlackChannelBackend)` — closed, removed from backlog

## References

- Blog: `blog/2026-06-17-mdp12-the-library-gets-its-memory.md` (published)
- ARC42STORIES.MD: `ARC42STORIES.MD` (project root — primary design doc)
- Protocols: `docs/protocols/connectors/` (6 total)
