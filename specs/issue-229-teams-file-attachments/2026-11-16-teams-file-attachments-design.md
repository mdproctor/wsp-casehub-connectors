# Design: Teams File Attachment Support in toMessageBody

**Issue:** casehubio/connectors#229
**Branch:** issue-229-teams-file-attachments
**Date:** 2026-11-16

---

## Overview

`TeamsMapper.toMessageBody()` currently ignores `ChatContent.attachments()`. Teams Graph API requires uploading files to the channel's SharePoint drive before they can be referenced in a message. This design adds that capability while keeping `TeamsMapper` as a pure mapper.

---

## Architecture

The implementation splits into three layers:

1. **TeamsClient** — new Graph API calls for channel file upload
2. **TeamsChatPlatform** — orchestrates upload before message send
3. **TeamsMapper** — extended to accept pre-uploaded file references

### New types

**`UploadedRef` (package-private record in `chat-teams`)**
```java
record UploadedRef(String name, String contentUrl) {}
```
Carries just what's needed to build the Graph message reference attachment. `contentType` is intentionally absent — Teams reference attachments always use the fixed string `"reference"` as the JSON `contentType`, not the file's MIME type. Internal to `chat-teams` — not part of any SPI.

---

## TeamsClient changes

Two new package-private methods:

### `getChannelFilesFolder(teamId, channelId, token)`
Calls `GET /teams/{teamId}/channels/{channelId}/filesFolder`.

Returns a `JsonObject` (the Graph DriveItem for the folder). The upload methods need:
- **driveId**: extracted from `parentReference.driveId` on the returned item
- **folderId**: the item's own `id` field

> **Implementation note:** verify the actual Graph response shape against the API docs. If `parentReference.driveId` is absent (e.g. the drive is the root drive), fall back to extracting from the `webUrl` or use a separate `/filesFolder?$select=id,parentReference` to ensure the field is present.

### `uploadToChannelFolder(driveId, folderId, filename, contentType, bytes, token)`

**Filename handling (before embedding in URL):**
- If `filename` is null or blank, generate `"attachment-{index}"` with an appropriate extension derived from `contentType` (e.g. `"application/pdf"` → `".pdf"`), or fall back to `"attachment-{index}.bin"`.
- Prefix all filenames with the attachment index to avoid collision when two attachments have the same name: `"{index}-{filename}"` (e.g. `"0-report.pdf"`, `"1-report.pdf"`).
- URL-encode the resulting filename with `URLEncoder.encode(name, StandardCharsets.UTF_8).replace("+", "%20")` before embedding in the Graph path.

**Upload strategy (matches `MicrosoftDriveClient` thresholds):**
- ≤ 4MB: simple PUT to `/drives/{driveId}/items/{folderId}:/{encodedFilename}:/content`
- > 4MB: resumable session via POST to `/drives/{driveId}/items/{folderId}:/{encodedFilename}:/createUploadSession`, then chunked PUTs of 16×320KB each to the returned `uploadUrl`

**Critical — resumable chunk PUTs:** The `uploadUrl` from `createUploadSession` is pre-authenticated. Chunk PUTs must be sent using a **plain `HttpClient`** without an `Authorization` header. Do not route chunks through `MicrosoftGraphClient` (which always adds the Bearer token). Use `HttpHelper.CLIENT` directly with a manually built `HttpRequest`.

Returns a `JsonObject` with `webUrl` and `name` from the Graph drive item response. Use this to construct an `UploadedRef`.

Error responses propagate through `MicrosoftGraphClient`'s existing error mapping (`GraphErrorHandler`) for the session creation step; chunk PUT errors propagate as `ProviderApiException` directly.

---

## TeamsClient permissions

The application token used for `sendMessage` and `replyToMessage` holds `ChannelMessage.Send`. SharePoint/OneDrive uploads via Graph additionally require **`Files.ReadWrite.All`** (or `Sites.ReadWrite.All`) application permission.

> **Deployment requirement:** the Teams app registration must have `Files.ReadWrite.All` application permission granted by an admin. Without this, uploads return 403. The unit tests will pass (stubbed client) but prod will fail. Document this in the CLAUDE.md / consumer guide.

---

## TeamsChatPlatform changes

Both `sendMessage` and `replyToMessage` gain a pre-upload step:

```
1. If content.attachments() is non-empty:
   a. getChannelFilesFolder → driveId, folderId
   b. For each Attachment (with index i):
      - Compute safe filename: index-prefixed, null-safe, URL-encoded
      - uploadToChannelFolder → UploadedRef
   c. If any upload throws → propagate (whole send fails, D3)
2. toMessageBody(content, uploadedRefs)
```

The token used for uploads is the **application token** (same as `sendMessage` and `replyToMessage`).

---

## TeamsMapper changes

`toMessageBody` gains a second parameter:

```java
static JsonObject toMessageBody(ChatContent content, List<UploadedRef> uploadedRefs)
```

The existing zero-attachment call sites pass `List.of()`.

**File reference attachment format** (Teams Graph API `reference` type):
```json
{
  "id": "file-0",
  "contentType": "reference",
  "contentUrl": "https://casehub.sharepoint.com/sites/.../0-filename.pdf",
  "name": "0-filename.pdf"
}
```

Each uploaded ref is added to the `attachments` JSON array with generated ids (`"file-0"`, `"file-1"`, …). The HTML body gets corresponding `<attachment id="file-N"></attachment>` placeholders appended after any existing card placeholders.

If `uploadedRefs` is empty the method behaves identically to today.

---

## Error handling

Upload failures propagate as `ProviderApiException` subclasses from `GraphErrorHandler`:
- 429 → `ProviderRateLimitedException`
- 401/403 → `ProviderAuthException` (most likely cause: missing `Files.ReadWrite.All` permission)
- Other 4xx/5xx → `ProviderApiException`

No partial delivery. If the first of three attachments uploads but the second fails, the third is never attempted and no message is sent.

---

## Testing

**Unit tests in `TeamsMapperTest`:**
- `toMessageBody` with one uploaded ref — verifies `attachments` JSON array and HTML body placeholder
- `toMessageBody` with multiple refs — verifies all ids are unique and all placeholders appear
- `toMessageBody` with empty uploaded refs — no regression

**Unit tests in `TeamsClientTest` (new test class):**
- `getChannelFilesFolder` — verify correct Graph path, driveId and folderId extracted from response
- `uploadToChannelFolder` simple path — ≤4MB stub, verify PUT path (URL-encoded filename), returns webUrl
- `uploadToChannelFolder` resumable path — >4MB, verify session POST then chunk PUTs sent **without** Authorization header
- Null filename fallback — verify generated name used in path
- Duplicate filename deduplication — two attachments same name get different prefixed paths

**Integration-level in `TeamsChatPlatformTest`:**
- `sendMessage` with attachments — stub `TeamsClient`, verify `getChannelFilesFolder` called once, `uploadToChannelFolder` called per attachment, `sendMessage` called with correct body
- Upload failure → entire `sendMessage` throws `ProviderApiException`

---

## Module impact

No new Maven dependencies. All changes within `casehub-connectors-chat-teams`. The `microsoft-graph-client` module already provides `MicrosoftGraphClient`, `GraphErrorHandler`, and `HttpHelper.CLIENT`.

**Consumer guide update required:** document the `Files.ReadWrite.All` application permission prerequisite.

---

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java`
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsMapper.java`
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java`
- `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/MicrosoftDriveClient.java:23` — SIMPLE_UPLOAD_LIMIT=4MB, CHUNK_SIZE=16×320KB constants and resumable pattern
- `connectors-api` — ProviderApiException hierarchy
- `microsoft-graph-client` — MicrosoftGraphClient, GraphErrorHandler, HttpHelper.CLIENT
- casehubio/connectors#229 — issue
- specs/issue-229-teams-file-attachments/decisions.md — D1, D2, D3
