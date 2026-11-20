# Handoff — slot 225 connectors

## Session summary

- **#335 — normalise-report-tool-name**: Done on `issue-335-normalise-report-tool-name`.
  - Fix in platform repo: `DynamicToolRegistrar.java` line 72 — added `replace("/", "_")` to report tool name construction so slashed domains (e.g. `connectors/voice`) produce `connectors_voice_report` instead of `connectors/voice_report`
  - Added `SlashedDomainReportProvider` test fixture (`@McpDomain("test/nested")`) and test `reportTool_slashedDomainNormalisesToUnderscore`
  - Updated `DomainFilteringTest` in connectors to use normalised tool name `connectors_voice_report`
  - All 93 platform mcp tests pass, all 6 connectors domain filtering tests pass
- **#332 — corpus-cwd-docs**: Was already landed on main from prior session. Marked done in queue.

## Cross-repo state

| Repo | Branch | Commits ahead of main |
|------|--------|-----------------------|
| platform | issue-335-normalise-report-tool-name | 1 (fix + test) |
| connectors | issue-335-normalise-report-tool-name | 1 (test assertion update) |
| wsp-connectors | issue-335-normalise-report-tool-name | queue + handoff |

## Queue

| # | Title | Status |
|---|-------|--------|
| #332 | corpus-cwd-docs | Done (landed on main) |
| #335 | normalise-report-tool-name | Done on branch, ready to land |

## Pre-existing test failures (not caused by this session)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests
