# Handover — casehub-connectors

## What Happened

Designed and implemented BankFeedPlatform and EmailPlatform SPIs (#94),
discovered the platform simulation framework mid-design and pivoted from
hand-written sim modules to `@SimulationEligible` decorator generation.
Created corpus data and example scenario (#102). Added simulation
integration test proving the end-to-end path (#104). Filed and closed
platform#370 (YAML parity gaps — already implemented). MCP tools for
both SPIs landed via #103.

## What's Next

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 105 | Adopt @SimulationEligible on CalendarPlatform and ChatPlatform | M | Med |
| 106 | First real provider — TrueLayer BankFeedPlatform | L | High |

#105 is the interesting one — CalendarPlatform (flat) should be
straightforward. ChatPlatform (capability-based) is the first test of
the simulation framework with non-flat SPIs. See ADR-0011 for the
capability-based SPI discussion.

## Key Artifacts

- Design spec: `docs/specs/issue-94-bankfeed-email-spis/2026-09-20-bankfeed-email-platform-spis-design.md`
- Decisions: `docs/specs/issue-94-bankfeed-email-spis/decisions.md`
- ADR-0011: `docs/adr/0011-simulation-framework-for-connector-spis.md`
- Corpus data: `bank-spi/src/main/resources/simulation/bank-feed/`, `email-spi/src/main/resources/simulation/email/`
- Scenario: `graphql/src/test/resources/scenarios/household-finance/`
- Diary: `docs/blog/2026-09-20-mdp25-what-do-you-call-a-thing-that-isnt-real.md`
