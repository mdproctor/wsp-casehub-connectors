# Handoff — connectors

## Last Session

Planning session. Filed connectors#13 (migrate DESIGN.md → ARC42STORIES.MD) and
parent#132 (extend arc42stories spec with Application/Foundation/Extension tier
taxonomy). Branch hygiene: 5 branches stamped closed, main pushed to origin (had
been 2 commits behind since the IMAP work), zombie worktree process killed and
removed. Full build passed clean — all 17 email-inbound tests, no IDLE flakiness.

## Immediate Next Step

Run `work-start` for connectors#13 — migrate DESIGN.md → ARC42STORIES.MD.
Foundation-tier conventions apply (not CaseHub profile); connectors#13 issue body
has the section-by-section scope. devtown ARC42STORIES.MD is the structural reference.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- connectors#12 — IDLE test flakiness — build ran clean this session; may be self-resolved or intermittent · XS · Low
- parent#108 — casehub-qhorus.md deep-dive needs connector-backend module · XS · Low
- parent#109 — casehub-connectors.md fireAsync contract note · XS · Low
- parent#132 — arc42stories spec: add Application/Foundation/Extension tier taxonomy · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| connectors#13 | Migrate DESIGN.md → ARC42STORIES.MD (foundation tier) | M | Low | Immediate next |
| connectors#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |
| connectors#1 | Agent mesh framework conformance | XS | Low | |

## References

- Blog: `blog/2026-06-01-mdp04-arc42stories-three-tiers.md`
- Issues filed this session: connectors#13, parent#132
