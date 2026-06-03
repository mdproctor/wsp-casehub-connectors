# Handoff — connectors

## Last Session

connectors#12 fixed and shipped. Hybrid approach: five content/parsing tests converted to
`deliverDirect()` + start-after (deterministic Path A); remaining SMTP-after-start tests
use Awaitility `atMost(5s)` with `@Timeout(10)` headroom. All 17 tests deterministic.
Squashed to 2 commits on casehubio/connectors main. Filed connectors#14 (Minor finding:
convert `singlePlainTextMessage_deliveredWithCorrectFields` to `deliverDirect()`).

## Immediate Next Step

Run `work-start` for connectors#13 — migrate DESIGN.md → ARC42STORIES.MD.
Foundation-tier conventions apply (not CaseHub profile); connectors#13 issue body
has the section-by-section scope. devtown ARC42STORIES.MD is the structural reference.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- connectors#14 — convert `singlePlainTextMessage_deliveredWithCorrectFields` to `deliverDirect()` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| connectors#13 | Migrate DESIGN.md → ARC42STORIES.MD (foundation tier) | M | Low | Immediate next |
| connectors#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |
| connectors#1 | Agent mesh framework conformance | XS | Low | |

## References

- Blog: `blog/2026-06-02-mdp05-fixing-idle-timing-properly.md`
- Issues filed this session: connectors#14
- Issues closed: connectors#12
