# Handover — casehub-connectors

## What Happened

Implemented `bank-ref` module (#109) — in-memory reference BankPlatform
with pre-loaded UK test data. Follows the chat-ref/calendar-ref pattern:
`BankBackend` → `InMemoryBankBackend` (`@DefaultBean`) → `RefBankPlatform`
→ `BankRefBeans`. Three accounts (current, savings, credit card), 12
transactions with realistic UK merchants, payment lifecycle simulation
(auto-advance on poll: AUTHORIZATION_REQUIRED → EXECUTED → SETTLED).
30 tests. All documentation updated (CLAUDE.md, ARC42STORIES, consumer
guide, contributor guide).

Prior session: consent token persistence (#107), TrueLayer BankPlatform
(#106), BankFeedPlatform and EmailPlatform SPIs (#94), simulation
framework integration (#102, #104).

## What's Next

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 110 | bank-spi: Simulation corpus YAML for BankPlatform | S | Low |
| 111 | bank-truelayer: Reduce test setup friction for consuming apps | S | Med |
| 108 | TrueLayer: Payment webhook receiver for real-time status updates | M | Med |

#110 is the natural follow-on from #109 — packages the same test data as
simulation corpus YAML. #111 reduces onboarding friction for TrueLayer
consumers. #108 adds webhook-driven payment status updates (largest piece).

## Key Artifacts

- Design spec: `docs/specs/issue-109-bank-ref/2026-09-25-bank-ref-design.md` (workspace copy in `specs/`)
- Decisions: `docs/specs/issue-109-bank-ref/decisions.md`
- Plan: `plans/attic/issue-109-bank-ref/2026-09-25-bank-ref.md`
- #106 design spec: `docs/specs/issue-106-truelayer-bankfeed/2026-09-22-truelayer-bankplatform-design.md`
- Consumer guide: `docs/guides/consumer-guide.md`
- Contributor guide: `docs/guides/contributor-guide.md`
