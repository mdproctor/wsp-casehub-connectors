# Handoff — slot 225 connectors

## Session summary

- **#329 — Seed externalisation**: Landed on main (2 commits after squash). GitHub issue closed.
  - `SeedDataResolver` in `connectors-api` with three-tier fallback
  - All 12 SeedLoaders migrated, 25 YAML files in domain-namespaced test resources
  - MCP server corpus bundling, CDI startup observer, test-jar wiring
  - Critical classpath collision found and fixed during review
  - Blog entry written: "Seed data, your data"
  - Filed #331 (bootstrap UX) and #332 (CWD documentation), both scheduled in queue
- Follow-up checklist complete: code review, blog, issue filing, publishing, promotion

## All branches on main

All 4 repos (connectors, wsp-connectors, platform, wsp-platform) are on main with linear history. All workspace feature branches stamped closed and merged.

## Queue hygiene needed

The queue has stale entries that need cleaning on next session:
- #315 — does not exist on GitHub (deleted or never created)
- #218 — CLOSED (SharePoint multi-site)
- #323 — CLOSED (MCP server packaging)

First real open item: **#326** (stdio transport support)

After cleaning stale entries, run `work start` to scaffold #326.

## Pre-existing test failures (not caused by this session)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests

## Open queue items

| # | Title | Scale |
|---|-------|-------|
| #326 | MCP server: stdio transport support | M |
| #327 | MCP server: replace brittle exclude-types | S |
| #328 | MCP server: domain filtering integration test | S |
| #331 | MCP server: first-run seed corpus bootstrap | S |
| #332 | MCP server: document expected CWD for corpus | XS |
