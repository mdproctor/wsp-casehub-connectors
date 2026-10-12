# Handover — casehub-connectors

## What Happened

Completed #156 — added `@RequiresConfirmation` to four sensitive SPI methods for out-of-band human-in-the-loop confirmation (platform#561 OperationConfirmation SPI). Annotated:

- `BankPlatform.PaymentInitiation.initiatePayment()` — payment amount, currency, beneficiary
- `CommercePlatform.Checkout.checkout()` — cart ID
- `DocumentPlatform.FileOperations.delete()` — file ID
- `ContactsPlatform.ContactWrite.delete()` — contact ID

Added `casehub-platform-api` dependency to bank-spi, commerce-spi, document-spi, contacts-spi. Reflection-based annotation-presence tests in each module. Landed as PR#161.

Created issues #162 and #163 for Real-World Knowledge Platform (#132) Phases 3 and 4 — now unblocked by platform#509 landing.

## Decisions

- Annotations placed on SPI interfaces (not implementations) — all providers inherit confirmation automatically
- Summary templates use `${param.field}` bean-property syntax for human-readable confirmation prompts
- Decorator generation happens at app build time via platform's ConfirmationDecoratorProcessor — no code gen in SPI modules
- Tests verify annotation presence only — end-to-end confirmation flow tested in platform repo

## Cross-Repo

- platform#561 — OperationConfirmation SPI (closed, landed)
- platform#509 — WorkerAuthorizationPolicy consent tiers (closed, landed)
- platform#560 — TokenRefreshClient wiring (open, blocks live token refresh)
- blocks-ui#232 — Connection status UI component (open)

## Next

- connectors#162 — Wire CommercePlatform to knowledge pipeline (Phase 3 of #132). Unblocked by platform#509.
- connectors#163 — Wire TravelPlatform to knowledge pipeline (Phase 4 of #132). Depends on #162.
- connectors#45 — Teams ChatPlatform implementation. Blocked on Teams Bot credentials.

## Stale design branch

`design/real-world-knowledge-platform` has diverged significantly from main — should be closed or rebased.
