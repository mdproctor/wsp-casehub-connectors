# Decisions — #229: Teams file attachment support

## D1: Upload size strategy

**Choice:** Both simple (≤4MB) and resumable (>4MB)
**Alternatives:**
- Simple only — simpler but silently breaks on larger files
- Simple + throw if exceeded — leaves the door open but unhelpful caller error
**Rationale:** Consistent with how `document-microsoft` already handles Graph drive uploads. Chat attachments can include PDFs or screenshots that exceed 4MB.
**Trade-offs:** Resumable upload adds ~40 lines of chunked PUT logic in TeamsClient.
**Sources:** `MicrosoftDriveClient.java:23` (SIMPLE_UPLOAD_LIMIT = 4MB, CHUNK_SIZE = 16×320KB)
**Exploration:** quick
**Status:** captured

---

## D2: Upload location

**Choice:** Add upload methods directly to TeamsClient
**Alternatives:**
- Depend on `document-microsoft` / `MicrosoftDriveClient` — reuses existing code but `MicrosoftDriveClient` is drive-scoped, not channel-scoped; requires awkward per-channel construction and cross-module dependency
**Rationale:** `TeamsClient` already holds `MicrosoftGraphClient` and owns all Teams Graph API calls. Channel file uploads use the same Graph drive APIs — just routed via the channel's filesFolder driveId. Keeps `chat-teams` self-contained.
**Trade-offs:** Duplicates the simple/resumable upload branching logic from `MicrosoftDriveClient`. Acceptable given the size (~40 lines) and the alternative's coupling cost.
**Sources:** `TeamsClient.java`, `MicrosoftDriveClient.java`
**Exploration:** quick
**Status:** captured

---

## D3: Upload failure handling

**Choice:** Fail the whole send with ProviderApiException
**Alternatives:**
- Best-effort: skip failed uploads, send message without the attachment — silently drops content
**Rationale:** Silent partial delivery is worse than a clean error. Caller can retry or inform the user. Consistent with how other providers handle structured content failures.
**Trade-offs:** A transient Graph error on one of N attachments aborts the whole message.
**Sources:** `ProviderApiException` in `connectors-api`
**Exploration:** quick
**Status:** captured
