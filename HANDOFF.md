# Handoff — connectors

*Updated: #21 closed — removed from backlog.*

## Last Session

Closed connectors#21 (L): `ARC42STORIES.MD` created — 938 lines, 5 chapters, 5
layers. Migrated from DESIGN.md, 3 ADRs, 8 design specs, and mdp01–mdp11 diary
entries. qhorus#249 stale reference caught and corrected at write time.
`java-update-design` now routes to `ARC42STORIES.MD §10`. Blog: mdp12.

## Immediate Next Step

Confirm qhorus#261 has full scope for `SlackChannelBackend`, then run `work-start`
for qhorus#261 in the qhorus repo.

## Cross-Module

**We're blocking:**
- `qhorus` — qhorus#261: `casehub-qhorus-slack-channel` module (SlackChannelBackend).
  Depends on `casehub-connectors-slack-bot` (in GitHub Packages). · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#261 | casehub-qhorus-slack-channel — SlackChannelBackend | L | Med | Unblocks qhorus#249 (now closed — re-check) |
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

- Blog: `blog/2026-06-17-mdp12-the-library-gets-its-memory.md` (published)
- ARC42STORIES.MD: `ARC42STORIES.MD` (project root — primary design doc)
- Protocols: `docs/protocols/connectors/` (6 total)
