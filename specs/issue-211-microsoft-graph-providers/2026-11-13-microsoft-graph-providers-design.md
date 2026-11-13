# Microsoft Graph Providers — Design Spec

**Issue:** casehubio/connectors#211
**Branch:** issue-211-microsoft-graph-providers
**Scale:** L | **Complexity:** Med

## Overview

Five Platform SPI providers backed by Microsoft Graph REST API, sharing one
HTTP client module and one credential resolution path via `ServiceConnectionProvider`:

| Module | SPI | Graph API domain | id() |
|--------|-----|-------------------|------|
| `microsoft-graph-client` | — (shared) | Auth, HTTP, pagination, errors | — |
| `email-microsoft` | `EmailPlatform` | Outlook Mail | `"microsoft"` |
| `calendar-microsoft` | `CalendarPlatform` | Outlook Calendar | `"microsoft"` |
| `document-microsoft` | `DocumentPlatform` | OneDrive + SharePoint | `"microsoft-onedrive"` / `"microsoft-sharepoint"` |
| `chat-teams` | `ChatPlatform` | Teams channels/messages | `"teams"` |

All providers use raw HTTP via `HttpHelper.CLIENT` (D1) — no Microsoft Graph
Java SDK. Credentials resolved via `ServiceConnectionProvider` (D2) with
per-provider `@RequiresScopes` (D7).

## Module Structure

```
microsoft-graph-client/           # Shared module in connectors
  io.casehub.connectors.microsoft
    ├── MicrosoftGraphClient       # Base HTTP client: get/post/patch/delete
    ├── GraphPage<T>               # Paginated response wrapper
    ├── GraphErrorHandler          # HTTP status → ProviderApiException
    └── MicrosoftBeans             # CDI producer for MicrosoftGraphClient

email-microsoft/
  io.casehub.connectors.email.microsoft
    ├── MicrosoftMailClient        # Domain client: mailFolders, messages, attachments
    ├── MicrosoftEmailPlatform     # EmailPlatform impl
    ├── MailMapper                 # Graph JSON → SPI model mapping
    └── EmailMicrosoftBeans        # CDI producer

calendar-microsoft/
  io.casehub.connectors.calendar.microsoft
    ├── MicrosoftCalendarClient    # Domain client: calendars, events, delta sync
    ├── MicrosoftCalendarPlatform  # CalendarPlatform impl
    ├── CalendarMapper             # Graph JSON → SPI model mapping
    └── CalendarMicrosoftBeans     # CDI producer

document-microsoft/
  io.casehub.connectors.document.microsoft
    ├── MicrosoftDriveClient       # Composed drive client (basePath parameter)
    ├── MicrosoftDrivePlatform     # DocumentPlatform impl (OneDrive, /me/drive)
    ├── SharePointDrivePlatform    # DocumentPlatform impl (SharePoint, /sites/{id}/drive)
    ├── DriveMapper                # Graph JSON → SPI model mapping
    └── DocumentMicrosoftBeans     # CDI producer (produces both beans)

chat-teams/
  io.casehub.connectors.chat.teams
    ├── TeamsClient                # Domain client: channels, messages, members, presence
    ├── TeamsChatPlatform          # ChatPlatform impl (8 native capabilities)
    ├── TeamsMapper                # Graph JSON → SPI model mapping
    └── TeamsBeans                 # CDI producer
```

## microsoft-graph-client — Shared HTTP Client

### MicrosoftGraphClient

`@ApplicationScoped` CDI bean. Uses `HttpHelper.CLIENT` (protocol: PP-20260607-9794cb).
Takes credentials at call time (protocol: PP-20260609-0c3e24).

```java
@ApplicationScoped
public class MicrosoftGraphClient {

    private static final String BASE_URL = "https://graph.microsoft.com/v1.0";
    private static final int MAX_PAGES = 50;
    private static final Logger LOG = Logger.getLogger(MicrosoftGraphClient.class);

    // Low-level HTTP operations — all take accessToken at call time
    public JsonObject get(String path, String accessToken) { ... }
    public JsonObject post(String path, JsonObject body, String accessToken) { ... }
    public JsonObject patch(String path, JsonObject body, String accessToken) { ... }
    public void delete(String path, String accessToken) { ... }
    public byte[] getBytes(String path, String accessToken) { ... }
    public void putBytes(String path, byte[] body, String contentType, String accessToken) { ... }

    // Paginated GET — follows @odata.nextLink, fail-soft on mid-loop error
    public <T> GraphPage<T> getPage(String path, String accessToken,
            Function<JsonObject, T> mapper, String cursor) { ... }
}
```

**Pagination:** Graph API returns `@odata.nextLink` (full URL for next page) and
`value` (array of items). `getPage()` uses `@odata.nextLink` as the cursor.
On mid-loop failure: return accumulated partial results + WARNING log with
page count, item count, and error string (protocol: PP-20260610-83747b).
`MAX_PAGES = 50` constant caps pagination; cap-hit produces its own WARNING
distinct from error WARNING.

**Request construction:** Each method builds an `HttpRequest` with:
- `BASE_URL + path` (or the full `@odata.nextLink` URL for pagination continuation)
- `Authorization: Bearer <accessToken>` header
- `Content-Type: application/json` for POST/PATCH
- `Accept: application/json`

**JSON parsing:** Uses `jakarta.json.JsonObject` (available in Quarkus).
Domain clients parse into their own record types.

### GraphPage\<T\>

```java
public record GraphPage<T>(List<T> items, String nextCursor, boolean hasMore) {}
```

Maps to `Page<T>` from `connectors-api` at the platform impl layer.

### GraphErrorHandler

Delegates to `ProviderErrorMapping.mapStatusCode()` from `connectors-api`:

| Graph HTTP Status | Maps to |
|---|---|
| 401 | `ProviderAuthException(TOKEN_EXPIRED)` |
| 403 | `ProviderAuthException(INSUFFICIENT_SCOPE)` |
| 404 | `NoSuchElementException` (SPI-specific handling) |
| 410 | `SyncTokenExpiredException` (delta queries) |
| 429 | `ProviderRateLimitedException(retryAfterSeconds)` — extracts `Retry-After` header |
| 5xx | `ProviderApiException(statusCode, body, "microsoft")` |

Graph API error responses include a JSON body with `error.code` and `error.message`.
`GraphErrorHandler.handle(HttpResponse<?> response)` extracts these before delegating
to `ProviderErrorMapping`.

## email-microsoft — EmailPlatform (Outlook Mail)

### SPI Capabilities

| EmailPlatform method | Graph API endpoint |
|---|---|
| `listMailboxes()` | `GET /me/mailFolders` |
| `listMessages(mailboxId, pagination)` | `GET /me/mailFolders/{id}/messages?$top=N&$skip=M` |
| `getMessage(messageId)` | `GET /me/messages/{id}` (with `$expand=attachments`) |
| `getAttachment(messageId, attachmentId)` | `GET /me/messages/{id}/attachments/{id}` |
| `search(query, pagination)` | `GET /me/messages?$search="query"` |

### MicrosoftMailClient

Domain client in `email-microsoft` module. Injects `MicrosoftGraphClient`.

```java
public class MicrosoftMailClient {
    private final MicrosoftGraphClient graph;

    public List<JsonObject> listMailFolders(String accessToken) { ... }
    public GraphPage<JsonObject> listMessages(String folderId, String accessToken,
            int top, String cursor) { ... }
    public JsonObject getMessage(String messageId, String accessToken) { ... }
    public JsonObject getAttachment(String messageId, String attachmentId,
            String accessToken) { ... }
    public GraphPage<JsonObject> searchMessages(String query, String accessToken,
            int top, String cursor) { ... }
}
```

### MailMapper

Maps Graph API JSON responses to `EmailPlatform` SPI model types (`Mailbox`,
`EmailMessage`, `EmailAttachment`). Handles Outlook-specific fields:
- `wellKnownName` → mailbox type (inbox, sent, drafts, etc.)
- `body.contentType` → text/html discrimination
- `toRecipients`, `ccRecipients`, `from` → address extraction
- `hasAttachments` flag + lazy attachment loading

### OAuth2 Scopes

```java
@RequiresScopes(provider = "microsoft", scopes = {"Mail.Read"})
```

## calendar-microsoft — CalendarPlatform (Outlook Calendar)

### SPI Capabilities

| CalendarPlatform method | Graph API endpoint |
|---|---|
| `listCalendars()` | `GET /me/calendars` |
| `listEvents(calendarId, start, end)` | `GET /me/calendars/{id}/calendarView?startDateTime=...&endDateTime=...` |
| `getEvent(eventId)` | `GET /me/events/{id}` |
| `createEvent(calendarId, event)` | `POST /me/calendars/{id}/events` |
| `updateEvent(eventId, event)` | `PATCH /me/events/{id}` |
| `deleteEvent(eventId)` | `DELETE /me/events/{id}` |
| `listEventsSync(calendarId, syncRequest)` | `GET /me/calendarView/delta` |

### Delta Sync

Graph Calendar delta query (`/me/calendarView/delta`) returns:
- `value[]` — changed events
- `@odata.deltaLink` — token for next incremental sync
- HTTP 410 Gone — delta token expired → `SyncTokenExpiredException`

Maps to existing `SyncResult<CalendarEvent>(items, deletedIds, syncToken, hasMore)`.

Initial sync: call delta with `startDateTime` and `endDateTime`, no token.
Incremental sync: call delta with stored `deltaLink` token.
Deleted events: identified by `@removed` property in delta response.

### CalendarMapper

Maps Graph event JSON to `CalendarEvent` SPI model:
- `start`/`end` → `EventTiming` sealed hierarchy (`Timed` or `AllDay`)
  - `dateTime` + `timeZone` → `Timed`
  - `date` only → `AllDay`
- `attendees[]` → attendee list
- `isAllDay` boolean confirms `AllDay` timing
- `recurrence` → recurrence pattern (if present)

### OAuth2 Scopes

```java
@RequiresScopes(provider = "microsoft", scopes = {"Calendars.ReadWrite"})
```

## document-microsoft — DocumentPlatform (OneDrive + SharePoint)

### Architecture (D3 revised, D6 revised)

Single module, two CDI beans, composed `MicrosoftDriveClient`:

```java
// CDI producer creates both beans
@ApplicationScoped
public class DocumentMicrosoftBeans {

    @Produces @ApplicationScoped
    MicrosoftDrivePlatform oneDrive(MicrosoftGraphClient graph,
            ServiceConnectionProvider connectionProvider) {
        return new MicrosoftDrivePlatform(graph, connectionProvider);
    }

    @Produces @ApplicationScoped
    SharePointDrivePlatform sharePoint(MicrosoftGraphClient graph,
            ServiceConnectionProvider connectionProvider) {
        return new SharePointDrivePlatform(graph, connectionProvider);
    }
}
```

`MicrosoftDrivePlatform` (id = `"microsoft-onedrive"`) uses base path `/me/drive`.
`SharePointDrivePlatform` (id = `"microsoft-sharepoint"`) uses base path `/sites/{siteId}/drive`.

Both create a `MicrosoftDriveClient` with their respective base path — composition,
not inheritance.

### Capability Sub-interfaces

Following the Google pattern (inner classes):

| Capability | Graph API |
|---|---|
| `FileOperations.list(folderId, pagination)` | `GET {basePath}/items/{folderId}/children` |
| `FileOperations.get(fileId)` | `GET {basePath}/items/{fileId}` |
| `FileOperations.download(fileId)` | `GET {basePath}/items/{fileId}/content` (302 redirect → follow) |
| `FileOperations.upload(folderId, filename, content)` | `PUT {basePath}/items/{folderId}:/{filename}:/content` |
| `FileOperations.delete(fileId)` | `DELETE {basePath}/items/{fileId}` |
| `FileOperations.listSync(syncRequest)` | `GET {basePath}/root/delta` |
| `FolderOperations.list(parentId, pagination)` | `GET {basePath}/items/{parentId}/children` (filter folders) |
| `FolderOperations.create(parentId, name)` | `POST {basePath}/items/{parentId}/children` (folder facet) |
| `FolderOperations.move(fileId, newParentId)` | `PATCH {basePath}/items/{fileId}` (parentReference) |
| `SearchOperations.search(query, pagination)` | `POST {basePath}/root/search(q='query')` |
| `SharingOperations.getShareLink(fileId)` | `POST {basePath}/items/{fileId}/createLink` |

### Delta Sync

Graph Drive delta (`{basePath}/root/delta`) follows the same pattern as calendar:
- `@odata.deltaLink` for incremental token
- HTTP 410 → `SyncTokenExpiredException`
- Deleted items identified by `deleted` facet

### MicrosoftDriveClient

```java
public class MicrosoftDriveClient {
    private final MicrosoftGraphClient graph;
    private final String basePath;  // "/me/drive" or "/sites/{siteId}/drive"

    public MicrosoftDriveClient(MicrosoftGraphClient graph, String basePath) {
        this.graph = graph;
        this.basePath = basePath;
    }

    // All drive CRUD operations use basePath
    public GraphPage<JsonObject> listChildren(String folderId, String accessToken,
            int top, String cursor) { ... }
    public JsonObject getItem(String itemId, String accessToken) { ... }
    public byte[] downloadContent(String itemId, String accessToken) { ... }
    // ... etc
}
```

### OAuth2 Scopes

```java
// MicrosoftDrivePlatform (OneDrive)
@RequiresScopes(provider = "microsoft", scopes = {"Files.ReadWrite"})

// SharePointDrivePlatform
@RequiresScopes(provider = "microsoft", scopes = {"Sites.Read.All", "Files.ReadWrite.All"})
```

### SharePoint Site Resolution

`SharePointDrivePlatform` needs a site ID to construct its base path. Options:
- Configuration: `casehub.connectors.sharepoint.site-id` config property
- Discovery: `GET /sites?search=...` at startup to find the default site
- Per-request: site ID passed as part of the folderId convention (e.g., `{siteId}:{folderId}`)

The configuration approach is simplest and consistent with how other providers
handle deployment-specific settings (e.g., Slack workspace, Discord guild).
The site ID is injected into `SharePointDrivePlatform` via the CDI producer.

## chat-teams — ChatPlatform (Teams)

### Capability Mapping

| ChatPlatform capability | Graph API | Permission type | Status |
|---|---|---|---|
| **Messaging** | `POST /teams/{id}/channels/{id}/messages` | Application (`ChannelMessage.Send`) | Native |
| **Threading** | `POST .../messages/{id}/replies` | Application (`ChannelMessage.Send`) | Native |
| **Discovery** | `GET /teams/{id}/channels` | Delegated (`Channel.ReadBasic.All`) | Native |
| **Presence** | `GET /users/{id}/presence` | Delegated (`Presence.Read.All`) | Native |
| **Members** | `GET /teams/{id}/members` | Delegated (`TeamMember.Read.All`) | Native |
| **ChannelManagement** | `POST /teams/{id}/channels`, `DELETE` | Delegated (`Channel.Create`, `Channel.Delete.All`) | Native |
| **MemberManagement** | `POST /teams/{id}/members` | Delegated (`TeamMember.ReadWrite.All`) | Native |
| **MessageHistory** | `GET .../messages` | Application (`ChannelMessage.Read.All`) | Native |
| **Reactions** | Not supported via Graph API | — | Degraded (no-op) |
| **Commands** | Not supported (Bot Framework) | — | Degraded (no-op) |

**8 native capabilities, 2 degraded.** Comparable to Discord (9 native) and
Slack (10 native).

### Dual Permission Model

Teams is unique among providers: it requires **both** delegated and application
permissions. Messaging/Threading/MessageHistory use application permissions
(no user context — the app posts as itself). Discovery/Presence/Members use
delegated permissions (user context required).

`ServiceConnectionProvider` handles this via the provider name convention:
- Delegated: `getAccessToken(actorId, "microsoft", tenancyId)` — user-scoped token
- Application: `getAccessToken(null, "microsoft-app", tenancyId)` — tenant-scoped token

The `"microsoft-app"` provider name signals application-level credentials
(client_credentials grant). This is an extension of the existing SPI —
`getAccessToken` with `null` actorId and a distinct provider name.

If `ServiceConnectionProvider` does not yet support null actorId, this will
need a minor SPI extension or a sentinel actorId value. The implementation
detail is owned by the `ServiceConnectionProvider` implementor, not the
connector module.

### TeamsClient

```java
public class TeamsClient {
    private final MicrosoftGraphClient graph;

    // Application permission operations
    public JsonObject sendMessage(String teamId, String channelId,
            String content, String appAccessToken) { ... }
    public JsonObject replyToMessage(String teamId, String channelId,
            String messageId, String content, String appAccessToken) { ... }
    public GraphPage<JsonObject> listMessages(String teamId, String channelId,
            String appAccessToken, String cursor) { ... }

    // Delegated permission operations
    public List<JsonObject> listChannels(String teamId, String userAccessToken) { ... }
    public JsonObject getPresence(String userId, String userAccessToken) { ... }
    public List<JsonObject> listMembers(String teamId, String userAccessToken) { ... }
    // ... etc
}
```

### Channel Model

Teams channels map to `Channel` SPI model:
- `teamId + channelId` compound identifier (separator: `:`)
- `displayName` → channel name
- `membershipType` → standard / private / shared
- `memberCount` from members list (no direct count API)

### OAuth2 Scopes

```java
@RequiresScopes(provider = "microsoft", scopes = {
    "Channel.ReadBasic.All", "Presence.Read.All",
    "TeamMember.Read.All", "Channel.Create", "Channel.Delete.All",
    "TeamMember.ReadWrite.All"
})
// Application scopes declared separately:
// ChannelMessage.Send, ChannelMessage.Read.All
```

## Auth Model

### ServiceConnectionProvider Integration (D2)

All five providers inject `ServiceConnectionProvider` and call
`getAccessToken(actorId, "microsoft", tenancyId)` at the start of each
SPI method invocation. The returned `ServiceAccessToken.accessToken()` is
passed as the `accessToken` parameter to `MicrosoftGraphClient` methods.

```java
// In each platform impl
private String resolveToken(String userId) {
    return connectionProvider.getAccessToken(userId, "microsoft", "default")
            .accessToken();
}
```

### Entra ID (Azure AD) Configuration

The `ServiceConnectionProvider` implementation handles Entra ID specifics:
- Tenant ID, Client ID, Client Secret — deployment config
- OAuth2 authorization endpoint: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize`
- Token endpoint: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`
- Scope format: `https://graph.microsoft.com/.default` (or individual scopes)

These details are NOT in the connector modules — they're in the
`ServiceConnectionProvider` implementation (platform-api boundary).

### Token Refresh on 401

When `MicrosoftGraphClient` receives a 401, it throws
`ProviderAuthException(TOKEN_EXPIRED)`. The calling platform code can
catch this and request a fresh token from `ServiceConnectionProvider`.
This is a single retry — if the second attempt also fails, propagate
the exception.

## Error Handling

### ProviderErrorMapping Reuse

`MicrosoftGraphClient.GraphErrorHandler` extracts HTTP status code, response
body, and Graph error code from responses, then delegates to
`ProviderErrorMapping.mapStatusCode(statusCode, body, providerId, errorCode)`.

Graph API error response format:
```json
{
  "error": {
    "code": "ErrorItemNotFound",
    "message": "The resource could not be found.",
    "innerError": {
      "request-id": "...",
      "date": "..."
    }
  }
}
```

`GraphErrorHandler` extracts `error.code` and `error.message` before mapping.

### 429 Rate Limiting

Graph API returns `Retry-After` header (seconds). `GraphErrorHandler`
extracts this and creates `ProviderRateLimitedException(retryAfterSeconds)`.
If header is missing, defaults to 30 seconds.

## MCP Surface

These providers do NOT add new MCP tools. Platform SPI providers are exposed
through the existing `ConnectorOperations` dispatch pattern — `casehub_action`
routes to registered platform providers by type. The existing MCP tools
(`list_files`, `get_file`, `calendar_list_events`, `send_chat`, etc.) will
automatically dispatch to Microsoft providers when they are the active/selected
provider for a user's deployment.

## Testing Strategy

Each provider module gets unit tests with mock HTTP responses. Integration
tests in the `integration-tests` module cover end-to-end with real Graph API
calls (gated by `@EnabledIfSystemProperty` for CI with Microsoft 365
developer sandbox credentials).

### Test Structure per Module

1. **MicrosoftGraphClientTest** — base client pagination, error handling,
   header construction
2. **\*MapperTest** — JSON → SPI model mapping for each domain
3. **\*ClientTest** — domain client with mocked `MicrosoftGraphClient`
4. **\*PlatformTest** — platform impl with mocked domain client, verifying
   SPI contract compliance

### Mock HTTP Pattern

`MicrosoftGraphClient` uses `HttpHelper.CLIENT.send()` — mock at the
`HttpClient` level (test constructor) or mock `MicrosoftGraphClient` itself
for domain client tests.

## Implementation Order

1. `microsoft-graph-client` — shared HTTP client, pagination, error handling
2. `email-microsoft` — highest n8n deploys (96.3K), exercises all client features
3. `calendar-microsoft` — delta sync validates the sync pattern
4. `document-microsoft` — OneDrive + SharePoint, composed DriveClient
5. `chat-teams` — dual permission model, most complex auth

Each step is independently testable and deployable.

## References

- `docs/specs/issue-211-microsoft-graph/README.md` — original issue spec
- `docs/specs/2026-11-13-connector-gap-analysis.md` — gap analysis positioning Microsoft as Phase 1
- `connectors-api/.../ProviderErrorMapping.java` — centralised error mapping
- `connectors-api/.../SyncResult.java`, `SyncRequest.java` — sync primitives
- `connectors-api/.../HttpHelper.java` — shared HTTP client singleton
- `platform-api/.../ServiceConnectionProvider.java` — credential resolution SPI
- `platform-api/.../RequiresScopes.java` — OAuth2 scope declaration
- `email-google/` — reference implementation for EmailPlatform provider pattern
- `calendar-google/` — reference implementation for CalendarPlatform + delta sync
- `document-google/` — reference implementation for DocumentPlatform + capability sub-interfaces
- Protocol PP-20260607-9794cb — shared HTTP client (`HttpHelper.CLIENT`)
- Protocol PP-20260609-0c3e24 — credential config ownership (call-time tokens)
- Protocol PP-20260609-e3a2bd — SPI id() naming convention
- Protocol PP-20260610-83747b — fail-soft pagination
- Protocol PP-20260609-0625c9 — `@Blocking` on MCP tool methods
- Decision review: `/Users/mdproctor/reviews/casehub-connectors/issue-211-decision-20261113-134235/`
