# HANDOFF — casehub-connectors

## Last Session

Completed #179 — provider-agnostic exception hierarchy and Google provider test coverage. Created `ProviderApiException`, `ProviderRateLimitedException`, `ProviderAuthException` (with `AuthFailureReason` discriminator for 401 vs 403 reason inspection) and `ProviderErrorMapping` in connectors-api. Updated all 4 Google modules (email, document, contacts, location) with typed error handling and 14 new test files. Fixed `initialSync` protocol compliance (partial results on mid-loop failure). Re-parented `SyncTokenExpiredException` under `ProviderApiException`. Closed duplicate #183; #182 remains for TrueLayer migration.

## Immediate Next Step

`work start 182` — migrate TrueLayer's 3 exception types to extend the new provider-agnostic hierarchy (S/Low).

## Cross-Module

- platform#560 — TokenRefreshClient wiring (open, blocks live token refresh)

## References

- `connectors-api/src/main/java/io/casehub/connectors/ProviderErrorMapping.java` — shared status-code mapping
- `specs/issue-179-google-provider-test-coverage/2026-10-27-google-provider-test-coverage-design.md` — design spec
- `plans/2026-10-27-google-provider-test-coverage.md` — implementation plan
- `blog/2026-10-27-mdp02-provider-exception-hierarchy.md` — diary entry
