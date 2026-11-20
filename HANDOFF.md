# Handoff — slot 225 connectors

## Session summary

- **#326 — stdio transport**: Landed on main. GitHub issue ready to close.
  - Platform: extracted `CaseHubMcpTools` to new `casehub-platform-mcp-dispatch` module (3 commits)
  - Platform: moved named server config (`quarkus.mcp.server.casehub.*`) to `mcp-dispatch`
  - Connectors: added `quarkus-mcp-server-stdio` via mutually exclusive Maven profiles (stdio default, `-Dhttp` for SSE)
  - Connectors: removed `quarkus.arc.exclude-types` hack, restored MCP resources
  - Connectors: 10 tests green including regression tests for dispatch tool absence and resource registration
  - Code review complete: critical finding (dual-transport conflict) fixed with profile restructure
  - Blog revised: updated tool count (133), replaced hack description with module split, stdio marked done
  - Filed #333 (modular MCP architecture — future exploration, L/High)
  - Closed #327 as obsolete (replaced by proper module split)

## Cross-repo state

| Repo | Branch | Commits ahead of local/main |
|------|--------|---------------------------|
| connectors | main | 4 (stdio transport + review fixes + blog) |
| platform | main | 3 (mcp-dispatch module + config move + jackson cleanup) |
| wsp-connectors | main | 20 (specs, plans, queue updates) |

Platform commits must land before connectors — connectors depends on the updated `casehub-platform-mcp` (without `CaseHubMcpTools`).

## Next active item

**#328 — domain-filtering-test** (XS/Low). The pipeline offered `branch_name` — next session should provide the branch name and proceed.

## Open queue items

| # | Title | Scale |
|---|-------|-------|
| #328 | MCP server: domain filtering integration test | XS |
| #331 | MCP server: first-run seed corpus bootstrap | S |
| #332 | MCP server: document expected CWD for corpus | XS |

## Pre-existing test failures (not caused by this session)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests
