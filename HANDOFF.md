# Handoff — slot 225 connectors

## Session summary

- **#332 — corpus-cwd-docs**: Was already landed on main from prior session. Marked done, closed on GitHub.
- **#335 — normalise-report-tool-name**: Fixed and landed on main across both repos.
  - Platform: `DynamicToolRegistrar.java` — added `replace("/", "_")` in report registration (line 72), `activateDomain` (line 189). `CaseHubToolCallbackProvider.java` — added `replace("/", "_")` in Spring path (line 138).
  - Connectors: Updated `DomainFilteringTest` assertion from `connectors/voice_report` to `connectors_voice_report`.
  - Added `SlashedDomainReportProvider` and `SlashedDomainQueryResolver` test fixtures with 4 new tests (report, activate, flat mode, negative). All 95 platform mcp tests pass.
  - Filed #338 (extract shared tool name normalisation method) and scheduled in queue.
  - Blog: `2026-11-20-mdp02-the-slash-that-slipped-through.md` — promoted to public.
  - Both issues closed on GitHub.

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors (slot 225) | main | #332 and #335 landed, stamp committed |
| platform (slot 225) | main | #335 landed (squashed, 1 commit) |
| wsp-connectors (slot 225) | main | blog, queue, handoff landed |
| public/connectors | issue-336-docs-device-location-guides | blog promoted, queue synced |

Slot's main is ahead of canonical repos — canonical connectors is on #334 work from another slot.

## Next active item

**#338 — extract-tool-name-normalisation** (XS/Low). Extract shared utility for domain-to-tool-name normalisation.

## Queue

| # | Title | Scale | Status |
|---|-------|-------|--------|
| #338 | extract shared tool name normalisation | XS | Queued |

## Pre-existing test failures (not caused by this session)

- `playbooks`: 4 failures — `ConnectorMobileApi` service registration (CDI wiring)
- `bank-truelayer`: 2 errors — JPA/webhook tests
