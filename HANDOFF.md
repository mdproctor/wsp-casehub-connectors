# HANDOFF — casehub-connectors

## Last Session

Retrofitted three platform SPIs (CalendarPlatform, DocumentPlatform, EmailPlatform) with user-scoped capability accessors so per-user OAuth tokens can be resolved. The same pattern already used by seven other SPIs (ContactsPlatform, BankPlatform, ProjectPlatform, etc.) — CalendarPlatform and EmailPlatform got inner `CalendarOperations`/`EmailOperations` interfaces behind `calendarOps(userId)`/`emailOps(userId)` accessors; DocumentPlatform added `String userId` to its existing `files()`/`folders()`/`search()`/`sharing()` accessors.

All Google implementations build per-user API clients inside the accessor. All Microsoft implementations capture userId and resolve tokens per-call via `TokenRetry`. MCP/GraphQL boundary classes inject `CurrentPrincipal` and pass `actorId()`.

**Branch:** `issue-223-actor-id-resolution` — ready for work-end

**Issues closed this session:**

| # | Title | What changed |
|---|-------|-------------|
| #223 | Cross-SPI actorId resolution for per-user OAuth | 3 SPI interfaces restructured, 6 provider implementations updated (3 Google, 3 Microsoft), 3 NoOp + 3 Ref implementations updated, 3 GraphQL API classes wired with CurrentPrincipal; 42 files changed across 13 modules |

**Issues filed this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #239 | Plugin framework userId resolution for playbook steps | S | Med |
| #240 | Google Document/Email NetHttpTransport reuse | XS | Low |
| #241 | Update consumer and contributor guides for user-scoped accessor pattern | S | Low |

**Queue state after this session:**

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #230 | toMessageBody URL sanitisation for markdown links | S | Low |

**To resume:** `work continue` for #230.
