# Handoff — issue-329-seed-externalisation

## Completed

- **#329 — Seed externalisation**: All implementation complete.
  - `SeedDataResolver` in `connectors-api` with three-tier fallback (file → classpath → empty)
  - All 12 SeedLoaders migrated (including health-ref readTree variant and mobile-ref path normalisation)
  - 25 seed YAML files moved from `src/main/resources/` to `src/test/resources/seed/<domain>/`
  - MCP server corpus bundling via maven-resources-plugin (package phase)
  - `SeedDataResolverConfig` CDI startup observer reads `casehub.ref.seed-dir`
  - Test-jar dependencies for all consumer modules (spring, integration-tests, playbooks)
  - Domain-aware classpath resolution to prevent `messages.yaml` collision between email-ref and mobile-ref
  - Code review completed — critical classpath collision found and fixed
  - Squash review: 12 commits → 2
  - Filed #331 (first-run seed bootstrap UX) as follow-up

## Follow-up needed

A comprehensive follow-up checklist is saved to session memory (`seed-externalisation-followup`):
1. Comprehensive code review (deeper than the automated one done here)
2. Write/revise blog entry about seed externalisation
3. File issues for loose ends
4. Schedule filed issues in plan
5. Work end with full promotion and publishing
6. Promote workspace files and publish blogs from all branches
7. Mark finished issues done, scaffold next queue item

## Pre-existing test failures (not caused by this branch)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests

## Queue

Next open items: #315, #218, #323, #326, #327, #328
