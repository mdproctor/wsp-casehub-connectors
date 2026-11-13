# HANDOFF — casehub-connectors

## Last Session

Completed Microsoft Graph providers epic (#211) — all 6 batches implemented, code review fixes applied, blog written, 8 follow-up issues filed.

**Branch:** `issue-211-microsoft-graph-providers` — ready for `work end`

**Design phase (prior session):**
- Brainstormed 7 decisions (D1-D7), all captured in `specs/issue-211-microsoft-graph-providers/decisions.md`
- D3+D6 revised after light decision review (separate modules → single module with two CDI beans, abstract base → composed DriveClient)
- Full design spec written and reviewed (3-round standard review, 26 issues, 20 verified, spec updated in-place)
- Implementation plan: 6 batches, 6 tasks

**All 6 batches implemented:**
- **Batch 1: Foundation** — `microsoft-graph-client` module: `MicrosoftGraphClient` (raw HTTP via `HttpHelper.CLIENT`, `@odata.nextLink`/`@odata.deltaLink` pagination), `GraphErrorHandler` (Graph-specific 403/429 mapping), `TokenRetry` (401 re-resolve). Added `send()` for testable custom requests, `BASE_URL` made public. 32 tests.
- **Batch 2: Email** — `email-microsoft` module: `MicrosoftMailClient`, `MailMapper`, `MicrosoftEmailPlatform`. Implements EmailPlatform SPI against Graph Mail API. 10 tests.
- **Batch 3: Calendar** — `calendar-microsoft` module: `MicrosoftCalendarClient`, `CalendarMapper`, `MicrosoftCalendarPlatform`. Implements CalendarPlatform SPI with delta sync via `@odata.deltaLink`. 13 tests.
- **Batch 4: Document** — `document-microsoft` module: `MicrosoftDriveClient` (composed with basePath), `DriveMapper`, `MicrosoftDrivePlatform`. Two CDI beans via `DocumentMicrosoftBeans`: `microsoft-onedrive` (`/me/drive`) and `microsoft-sharepoint` (`/sites/{siteId}/drive`, degraded when unconfigured). SSRF defense on download 302 redirects, resumable upload for >4MB, delta sync. 19 tests.
- **Batch 5: Chat** — `chat-teams` module: `TeamsClient`, `TeamsMapper`, `TeamsChatPlatform`. 8 native capabilities (Messaging, Threading, Discovery, Presence, Members, ChannelManagement, MemberManagement, MessageHistory), Reactions and Commands auto-degraded via Builder. Dual delegated/application token model. Compound `teamId:channelId` channel IDs. 22 tests.
- **Batch 6: Integration** — Cross-module integration test (9 tests), CLAUDE.md updated (5 module rows + description), consumer guide updated (6 module rows + 2 implementation table rows).

**Code review fixes applied:**
- OData injection: escape single quotes in search queries
- URL-encode filenames in upload paths
- NPE guard on validateCdnHost for malformed redirect URLs
- Replaced unreliable `$filter=file ne null` with client-side filtering

**All 105 Microsoft tests pass across 6 modules.** Pre-existing `VoiceRoundTripTest` failures in integration-tests are unrelated (missing kokoro voice platform).

**Blog:** `blog/2026-11-13-mdp02-one-client-four-platforms.md` — "One Client, Four Platforms" diary entry with SVG architecture diagram.

**Issues filed this session:**

| # | Title | Scale |
|---|-------|-------|
| #219 | document-microsoft: configurable SSRF CDN allow-list | XS |
| #220 | chat-teams: Teams inbound via Bot Framework | L |
| #221 | chat-teams: toMessageBody drops markdown and RichCard content | M |
| #222 | chat-teams: ChannelManagement.create() hardcodes first joined team | S |
| #223 | Cross-SPI: actorId resolution for per-user OAuth providers | M |
| #224 | document-microsoft: resumable upload test coverage | S |
| #225 | PlatformExtensible base interface with as() for vendor extensions | M |
| #226 | docs: add Microsoft Graph providers chapter to ARC42STORIES | S |

**Closed:** #45 (superseded by #211 + #221)

**Key patterns established:**
- `MicrosoftGraphClient.HttpSender` interface (public) for test mocking — same pattern across all modules
- `resolveToken()` returns `"test-token"` when `connectionProvider` is null (test mode)
- `StubHttpResponse` test utility duplicated per module (package-private)
- `toUri()` helper in `MicrosoftGraphClient` for URL encoding query parameters with spaces
- `@odata.deltaLink` support in `getPage()` — stored in `Page.nextCursor` with `hasMore=false`
- Composed `MicrosoftDriveClient` with basePath (OneDrive vs SharePoint)
- Dual token resolution for Teams (delegated + application)
- `ChatPlatform.Builder` for Teams (auto-degrades Reactions + Commands)
- Client-side filtering for file/folder separation (Graph `$filter` not reliable on all endpoints)

**Workspace artifacts:**
- `specs/issue-211-microsoft-graph-providers/2026-11-13-microsoft-graph-providers-design.md` — full design spec (review-updated)
- `specs/issue-211-microsoft-graph-providers/decisions.md` — 7 decisions
- `plans/2026-11-13-microsoft-graph-providers.md` — implementation plan

**To resume:** `work end` to close the branch.

## Paused Branch

`issue-206-mobile-platform` is paused on the stack (MobilePlatform SPI work).
