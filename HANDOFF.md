# Handoff — slot 225 connectors

## Session summary

- **#329 — Seed externalisation**: Landed on main (2 commits after squash). GitHub issue closed.
  - `SeedDataResolver` in `connectors-api` with three-tier fallback
  - All 12 SeedLoaders migrated, 25 YAML files in domain-namespaced test resources
  - MCP server corpus bundling, CDI startup observer, test-jar wiring
  - Critical classpath collision found and fixed during review
  - Blog entry written: "Seed data, your data"
  - Filed #331 (bootstrap UX) and #332 (CWD documentation), both scheduled in queue

## All branches on main

All 4 repos (connectors, wsp-connectors, platform, wsp-platform) are on main with linear history. All workspace feature branches stamped closed and merged.

## Pre-existing test failures (not caused by this session)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests

## Queue

Active: #315 (yaml-contains-normalised-assertion)
Open: #218, #323, #326, #327, #328, #331, #332
