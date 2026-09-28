# Handover — casehub-connectors

## What Happened

Implemented 5 new modules across 4 issues on 2 branches, adding EmailPlatform
and DocumentPlatform implementations to the connectors repo. Then filed
5 MCP tool coverage issues (#118-122) and 3 platform contract issues
(platform#474-476) to close the gap between SPI capabilities and LLM
discoverability.

**Branch 1 — email-ref + email-google (#113, #114):**
- `email-ref`: In-memory `RefEmailPlatform` — `EmailBackend` interface,
  `InMemoryEmailBackend` (3 mailboxes, 9 messages, 2 attachments,
  cursor-based pagination), CDI producer. 19 tests.
- `email-google`: `GoogleEmailPlatform` — Gmail API with OAuth2, label→mailbox
  mapping, N+1 metadata fetch per page, MIME body parsing (text/plain +
  text/html + multipart), attachment download. 14 WireMock tests.

**Branch 2 — document-spi + document-ref + document-google (#116, #117):**
- `document-spi`: New `DocumentPlatform` SPI with capability sub-interfaces
  following BankPlatform pattern — `FileOperations` (list, get, download,
  upload, delete), `FolderOperations` (list, create, move),
  `SearchOperations` (full-text), `SharingOperations` (share links).
  `supports(Class<?>)` introspection, `@SimulationEligible`. 6 tests.
- `document-ref`: In-memory `RefDocumentPlatform` — 3 folders, 7 files,
  all 4 capabilities, upload/download round-trip, name-based search. 26 tests.
- `document-google`: `GoogleDocumentPlatform` — Google Drive API v3 with
  OAuth2, direct upload, folder management via addParents/removeParents,
  fullText search, anyone-reader share links. 18 WireMock tests.

**MCP coverage gap analysis + issue filing:**
- connectors#118 — EmailPlatform MCP tools (4 tools)
- connectors#119 — DocumentPlatform MCP tools (10 tools)
- connectors#120 — BankPlatform MCP tools (6 tools)
- connectors#121 — ChatPlatform expanded MCP tools (10 tools)
- connectors#122 — `connectors_report(scope=...)` + structured error responses
- platform#474 — Structured error response type (CLOSED — landed)
- platform#475 — Domain report convention (CLOSED — landed)
- platform#476 — Platform aggregator (CLOSED — landed)

## What's Next

`.plan` queued on main with 5 issues — MCP tool coverage for all platform SPIs:

| # | Issue | Scale | Complexity | Depends on |
|---|-------|-------|------------|------------|
| #122 | connectors_report + structured errors | M | Med | platform#474, #475 (landed) |
| #118 | EmailPlatform MCP tools | S | Low | #122 (structured errors) |
| #119 | DocumentPlatform MCP tools | M | Med | #122 (capability-aware tooling) |
| #120 | BankPlatform MCP tools | S | Med | #122 (structured errors) |
| #121 | ChatPlatform expanded MCP tools | M | Med | #122 (capability-aware tooling) |

**Recommended order:** #122 first — it establishes `connectors_report` and the
structured error pattern. Then #118-121 in any order (independent per-SPI tools).

**Design note from this session:** `connectors_report(scope=...)` uses a single
tool with a scope parameter (`all`, `email`, `email,bank`) to control what's
returned. Token cost proportional to scope. One hop. The pattern is standardised
at the platform level (platform#475) — IoT will adopt the same shape.

## Key Artifacts

- Blog entry: `blog/2026-09-28-mdp01-emailplatform-ref-and-gmail.md`
- Design spec (email/bank SPIs): `docs/specs/issue-94-bankfeed-email-spis/2026-09-20-bankfeed-email-platform-spis-design.md`
