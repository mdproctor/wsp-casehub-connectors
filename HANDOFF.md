# Handoff — slot 225 connectors

## Session summary

- **#328 — domain-filtering-test**: Done on `issue-328-domain-filtering-test`. Ready to land.
  - Added `DomainFilteringTest` — `@QuarkusTest` with `@TestProfile` for explicit mode (`casehub.mcp.mode=explicit`, `casehub.mcp.domains=chat,calendar`)
  - 6 assertions: chat/calendar registered, bank/health/commerce excluded, voice report excluded
  - All tests pass. Code review clean.
  - Filed #335 (report tool naming inconsistency — `connectors/voice_report` uses `/` while flat tools use `_`)
  - #335 scheduled in queue

## Cross-repo state

| Repo | Branch | Commits ahead of local/main |
|------|--------|---------------------------|
| connectors | issue-328-domain-filtering-test | 1 (test) |
| wsp-connectors | issue-328-domain-filtering-test | 2 (queue updates, handoff) |

Prior session's connectors and platform commits on main are still ahead of local/main (not yet pushed to origin).

## Next active item

**#331 — seed-bootstrap-ux** (S/Med). First-run seed corpus bootstrap experience.

## Open queue items

| # | Title | Scale |
|---|-------|-------|
| #331 | MCP server: first-run seed corpus bootstrap | S |
| #332 | MCP server: document expected CWD for corpus | XS |
| #335 | normalise report tool name to underscore separator | XS |

## Pre-existing test failures (not caused by this session)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests
