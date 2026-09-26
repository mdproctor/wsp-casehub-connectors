# Handover — casehub-connectors

## What Happened

Completed three banking issues on `issue-110-bank-spi-simulation`:

**#109 (bank-ref):** New `bank-ref` module — in-memory BankPlatform
with 3 UK accounts, 12 transactions, payment lifecycle simulation.
Follows chat-ref/calendar-ref pattern. Merged to main, issue closed.

**#110 (simulation corpus):** Renamed stale `bank-feed-platform.*`
qualified names to `bank-platform.accountInformation.*` /
`bank-platform.paymentInitiation.*` across all corpus YAML, simulation
configs, scenario files, and SimulationIntegrationTest. Added
`payments-corpus.yaml` with 3 payment lifecycle scenarios.

**#111 (TrueLayer test profile):** `TrueLayerTestProfile` implementing
`QuarkusTestProfile` — bundles 12 config lines into a reusable test
profile shipped via test-jar.

**#108 (payment webhook):** Designed and reviewed. Light design review
caught a showstopper — spec assumed HMAC-SHA512 but TrueLayer uses
JWS (ES512). Revised spec and implementation plan committed. Ready for
execution.

## What's Next

#108 is the active issue with a committed plan at
`plans/2026-09-25-payment-webhook.md`. Execute the plan — 2 batches,
2 tasks. The JWS finding from the design review is already incorporated
in the revised spec.

## Key Artifacts

- #108 design spec: `specs/issue-110-bank-spi-simulation/2026-09-25-payment-webhook-design.md`
- #108 decisions: `specs/issue-110-bank-spi-simulation/decisions.md`
- #108 plan: `plans/2026-09-25-payment-webhook.md`
- #108 design review: `/Users/mdproctor/reviews/casehub-connectors/issue-108-payment-webhook-20260925-053043/`
- #109 design spec: `docs/specs/issue-109-bank-ref/2026-09-25-bank-ref-design.md`
- Diary: `blog/2026-09-25-mdp01-bank-ref-pattern-absorption.md`
