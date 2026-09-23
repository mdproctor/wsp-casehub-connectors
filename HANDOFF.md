# Handover — casehub-connectors

## What Happened

Implemented database-backed consent token persistence for TrueLayer (#107).
Extracted a `ConsentTokenStore` SPI following the platform's persistence
pattern (work/ledger): JPA implementation (Tier 1, EntityManager directly —
no Panache) and in-memory implementation (Tier 3, ConcurrentHashMap). Added
`ConsentCleanupJob` with daily scheduled purge of expired consents. Updated
consumer guide with migration path documentation.

Prior session: designed and implemented BankPlatform SPI with capability
sub-interfaces (#106), TrueLayer provider (AISP + PISP), OAuth2 consent
flow, PSD2 compliance. Earlier: BankFeedPlatform and EmailPlatform SPIs
(#94), simulation framework integration (#102, #104), MCP tools (#103).

## What's Next

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 109 | bank-ref: In-memory reference BankPlatform implementation | M | Low |
| 110 | bank-spi: Simulation corpus YAML for BankPlatform | S | Low |
| 111 | bank-truelayer: Reduce test setup friction for consuming apps | S | Med |
| 108 | TrueLayer: Payment webhook receiver for real-time status updates | M | Med |

#109 is highest-impact — gives developers a working banking backend out of
the box with zero configuration (like chat-ref and calendar-ref). #110 and
#111 complement it for simulation and TrueLayer-specific onboarding.

## Key Artifacts

- Design spec: `docs/specs/issue-107-consent-db-persistence/2026-09-23-consent-db-persistence-design.md` (workspace copy in `specs/`)
- Decisions: `docs/specs/issue-107-consent-db-persistence/decisions.md`
- Plan: `plans/2026-09-23-consent-db-persistence.md`
- #106 design spec: `docs/specs/issue-106-truelayer-bankfeed/2026-09-22-truelayer-bankplatform-design.md`
- Consumer guide: `docs/guides/consumer-guide.md` (consent persistence section added)
