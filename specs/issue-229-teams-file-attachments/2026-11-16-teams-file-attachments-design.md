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
record UploadedRef(String name, String contentUrl, String contentType) {}
```
Carries just what's needed to build the Graph message reference attachment. Internal to `chat-teams` — not part of any SPI.

---

## TeamsClient changes

Two new package-private methods:

### `getChannelFilesFolder(teamId, channelId, token)`
Calls `GET /teams/{teamId}/channels/{channelId}/filesFolder`.
Returns a `JsonObject` containing `parentReference.driveId` and `id` (the folder's item ID in that drive).

### `uploadToChannelFolder(driveId, folderId, filename, contentType, bytes, token)`
Uploads a file to the channel's SharePoint folder. Returns a `JsonObject` with `webUrl` and `name` from the Graph drive item response.

**Upload strategy (matches `MicrosoftDriveClient` thresholds):**
- ≤ 4MB: simple PUT to `/drives/{driveId}/items/{folderId}:/{filename}:/content`
- > 4MB: resumable session via POST to `/drives/{driveId}/items/{folderId}:/{filename}:/createUploadSession`, then chunked PUTs of 16×320KB each to the returned `uploadUrl`

Error responses propagate as-is through `MicrosoftGraphClient`'s existing error mapping (`GraphErrorHandler`).

---

## TeamsChatPlatform changes

Both `sendMessage` and `replyToMessage` gain a pre-upload step:

```
1. If content.attachments() is non-empty:
   a. getChannelFilesFolder → driveId, folderId
   b. For each Attachment: uploadToChannelFolder → UploadedRef
   c. If any upload throws → propagate (whole send fails, D3)
2. toMessageBody(content, uploadedRefs)
```

The token used for uploads is the **application token** (same as the one already used for `sendMessage` and `replyToMessage` — no new auth complexity).

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
  "contentUrl": "https://casehub.sharepoint.com/sites/.../filename.pdf",
  "name": "filename.pdf"
}
```

Each uploaded ref is added to the `attachments` JSON array with a generated id (`"file-0"`, `"file-1"`, …). The HTML body gets corresponding `<attachment id="file-N"></attachment>` placeholders appended after any existing card placeholders.

If `uploadedRefs` is empty the method behaves identically to today.

---

## Error handling

Upload failures propagate as `ProviderApiException` subclasses from `GraphErrorHandler`:
- 429 → `ProviderRateLimitedException`
- 401/403 → `ProviderAuthException`
- Other 4xx/5xx → `ProviderApiException`

No partial delivery. If the first of three attachments uploads successfully but the second fails, the third is never attempted and no message is sent.

---

## Testing

**Unit tests in `TeamsMapperTest`:**
- `toMessageBody` with one uploaded ref — verifies `attachments` JSON array and HTML body placeholder
- `toMessageBody` with multiple refs — verifies all ids are unique and all placeholders appear
- `toMessageBody` with empty uploaded refs — no regression from existing tests

**Unit tests in `TeamsClientTest`:**
- `getChannelFilesFolder` — verify correct Graph path called, driveId and folderId extracted
- `uploadToChannelFolder` simple path — ≤4MB, verify PUT path and returns webUrl
- `uploadToChannelFolder` resumable path — >4MB, verify upload session POST then chunked PUTs

**Integration-level in `TeamsChatPlatformTest`:**
- `sendMessage` with attachments — stub `TeamsClient`, verify `getChannelFilesFolder` called once, `uploadToChannelFolder` called per attachment, then `sendMessage` called with correct body
- Upload failure → entire `sendMessage` throws

---

## Module impact

No new Maven dependencies. All changes are within `casehub-connectors-chat-teams`. The `microsoft-graph-client` module already provides `MicrosoftGraphClient` and `GraphErrorHandler`, which handle the HTTP mechanics.

---

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsClient.java`
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsMapper.java`
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsChatPlatform.java`
- `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/MicrosoftDriveClient.java:23` — SIMPLE_UPLOAD_LIMIT and CHUNK_SIZE constants
- `connectors-api` — ProviderApiException hierarchy
- `microsoft-graph-client` — MicrosoftGraphClient, GraphErrorHandler
- casehubio/connectors#229 — issue
- specs/issue-229-teams-file-attachments/decisions.md — D1, D2, D3
