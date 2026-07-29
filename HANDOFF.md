# Handoff — connectors

## Last Session

Closed branch `issue-90-per-tenant-dedup` — per-tenant destination
deduplication (#90). Cross-repo change: `DestinationScope` enum and
dispatcher dedup logic in casehub-platform, bridge scope + Slack/Teams
opt-in in connectors. Design spec adversarially reviewed (3 rounds, 11
issues). Pushed to both upstream repos.

## Immediate Next Step

Pick up #45 (Teams ChatPlatform) or #32 (Discord slash commands) — both
are independent M/Med features. Or clean up overdue branches (issue-4,
6, 7, 9, 12, 39, 86, 89) and recover 14 unrecovered workspace artifacts.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12, 39, 86, 89) · XS · Low
- 14 unrecovered artifacts on closed workspace branches (specs, blogs) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | Teams ChatPlatform implementation | M | Med | Requires Teams Bot API client |
| #58 | Responsive layout primitives for pages-runtime | L | High | Cross-module design needed |
| #32 | Discord slash commands and interactions | M | Med | — |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
