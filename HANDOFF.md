# HANDOFF — casehub-connectors

## Last Session

Started Microsoft Graph providers epic (#211) — brainstorming, design spec, plan, and 3 of 6 batches implemented.

**Branch:** `issue-211-microsoft-graph-providers`

**Design phase:**
- Brainstormed 7 decisions (D1-D7), all captured in `specs/issue-211-microsoft-graph-providers/decisions.md`
- D3+D6 revised after light decision review (separate modules → single module with two CDI beans, abstract base → composed DriveClient)
- Full design spec written and reviewed (3-round standard review, 26 issues, 20 verified, spec updated in-place)
- Implementation plan: 6 batches, 6 tasks

**Implementation completed (Batches 1-3):**
- **Batch 1: Foundation** — `microsoft-graph-client` module: `MicrosoftGraphClient` (raw HTTP via `HttpHelper.CLIENT`, `@odata.nextLink`/`@odata.deltaLink` pagination), `GraphErrorHandler` (Graph-specific 403/429 mapping), `TokenRetry` (401 re-resolve pattern). 32 tests.
- **Batch 2: Email** — `email-microsoft` module: `MicrosoftMailClient`, `MailMapper`, `MicrosoftEmailPlatform`. Implements EmailPlatform SPI against Graph Mail API. 10 tests.
- **Batch 3: Calendar** — `calendar-microsoft` module: `MicrosoftCalendarClient`, `CalendarMapper`, `MicrosoftCalendarPlatform`. Implements CalendarPlatform SPI with delta sync support. 13 tests.

**All 55 tests pass. All 3 modules build successfully.**

**Implementation remaining (Batches 4-6):**
- **Batch 4: Document** — `document-microsoft` module: OneDrive + SharePoint in single module with two CDI beans, composed `MicrosoftDriveClient`. Includes SSRF defense on 302 redirects, resumable upload for >4MB files.
- **Batch 5: Chat** — `chat-teams` module: Teams ChatPlatform with 8 native capabilities, dual permission model (delegated + application tokens). Requires `@RequiresScopes` to be `@Repeatable` in platform-api.
- **Batch 6: Integration** — Integration smoke tests, CLAUDE.md update, consumer guide update.

**Key patterns established:**
- `MicrosoftGraphClient.HttpSender` interface (public) for test mocking — same pattern across all modules
- `resolveToken()` returns `"test-token"` when `connectionProvider` is null (test mode)
- `StubHttpResponse` test utility duplicated per module (package-private)
- `toUri()` helper in `MicrosoftGraphClient` for URL encoding query parameters with spaces
- `@odata.deltaLink` support added to `getPage()` — stored in `Page.nextCursor` with `hasMore=false`

**Workspace artifacts:**
- `specs/issue-211-microsoft-graph-providers/2026-11-13-microsoft-graph-providers-design.md` — full design spec (review-updated)
- `specs/issue-211-microsoft-graph-providers/decisions.md` — 7 decisions
- `plans/2026-11-13-microsoft-graph-providers.md` — implementation plan

**To resume:** `work continue` — pick up at Batch 4 (document-microsoft).

## Paused Branch

`issue-206-mobile-platform` is paused on the stack (MobilePlatform SPI work).
