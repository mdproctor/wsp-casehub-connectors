# Design Journal — issue-110-bank-spi-simulation

## 2026-09-25 — Banking queue: corpus, test profile, webhook design

Completed #110 (simulation corpus) and #111 (TrueLayer test profile)
inline — both were small enough to implement directly without plans.

#110: renamed stale `bank-feed-platform.*` qualified names to
`bank-platform.accountInformation.*` / `bank-platform.paymentInitiation.*`
across all corpus YAML, simulation configs, scenario files, and the
`SimulationIntegrationTest`. Added `payments-corpus.yaml` with three
payment lifecycle scenarios.

#111: `TrueLayerTestProfile` implementing `QuarkusTestProfile` —
bundles all 12 lines of OIDC/consent/devservice config into a reusable
test profile. Shipped via test-jar so consuming apps just annotate
`@TestProfile(TrueLayerTestProfile.class)`.

#108 (payment webhook): designed and reviewed. Light design review
caught a showstopper — spec assumed HMAC-SHA512 but TrueLayer uses
JWS (ES512) via `Tl-Signature` header. Revised spec incorporates
JWS verification via `com.truelayer:truelayer-signing:0.3.0`, event
deduplication by `event_id`, `PaymentStatusChanged` record with
`failureReason` and `eventId` fields, and explicit validation-disable
flag. Implementation plan written and committed.
