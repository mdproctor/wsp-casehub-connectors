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
    ├── GraphErrorHandler          # HTTP status → ProviderApiException
    ├── TokenRetry                 # 401 → re-resolve token → single retry
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
    public <T> Page<T> getPage(String path, String accessToken,
            Function<JsonObject, T> mapper, String cursor) { ... }
}
```

**Pagination:** Graph API returns `@odata.nextLink` (full URL for next page) and
`value` (array of items). `getPage()` returns `Page<T>` from `connectors-api`
directly — no intermediate type, since `Page<T>` already carries `items`,
`nextCursor`, and `hasMore`. The `@odata.nextLink` value is stored as the cursor.

**Cursor-vs-path behavior:** When `cursor` is non-null, `getPage()` uses `cursor`
directly as the full request URL (it is an `@odata.nextLink` absolute URL) and
ignores the `path` parameter. When `cursor` is null (first page), `getPage()` uses
`BASE_URL + path`. This prevents malformed URL construction like
`BASE_URL + @odata.nextLink`.

On mid-loop failure: return accumulated partial results + WARNING log with
page count, item count, and error string (protocol: PP-20260610-83747b).
`MAX_PAGES = 50` constant caps pagination; cap-hit produces its own WARNING
distinct from error WARNING.

**Request construction:** Each method builds an `HttpRequest` with:
- `BASE_URL + path` for initial requests (or full `cursor` URL for continuation)
- `Authorization: Bearer <accessToken>` header
- `Content-Type: application/json` for POST/PATCH
- `Accept: application/json`

**JSON parsing:** Uses `jakarta.json.JsonObject` (available in Quarkus).
Domain clients parse into their own record types.

### GraphErrorHandler

Maps Graph API HTTP errors to platform exception types. Delegates to
`ProviderErrorMapping.mapStatusCode()` for most status codes, but handles
429 and 403 directly because Graph API uses different conventions than
`ProviderErrorMapping`'s default expectations.

| Graph HTTP Status | Maps to | Handled by |
|---|---|---|
| 401 | `ProviderAuthException(TOKEN_EXPIRED)` | `ProviderErrorMapping` |
| 403 | `ProviderAuthException(INSUFFICIENT_SCOPE)` or `ProviderAuthException(FORBIDDEN)` | `GraphErrorHandler` directly |
| 404 | `NoSuchElementException` | `ProviderErrorMapping` |
| 410 | `SyncTokenExpiredException` (delta queries) | `ProviderErrorMapping` |
| 429 | `ProviderRateLimitedException(retryAfterSeconds)` | `GraphErrorHandler` directly |
| 5xx | `ProviderApiException(statusCode, body, "microsoft")` | `ProviderErrorMapping` |

**403 handling:** `ProviderErrorMapping` checks for the Google-specific string
`"insufficientPermissions"` to distinguish `INSUFFICIENT_SCOPE` from `FORBIDDEN`.
Graph API uses different error codes (`"Authorization_RequestDenied"`,
`"ErrorAccessDenied"`, `"AccessDenied"`). `GraphErrorHandler` maps these Graph
error codes to `INSUFFICIENT_SCOPE` before constructing the exception:

```java
private static AuthFailureReason map403Reason(String graphErrorCode) {
    return switch (graphErrorCode) {
        case "Authorization_RequestDenied",
             "ErrorAccessDenied",
             "AccessDenied" -> AuthFailureReason.INSUFFICIENT_SCOPE;
        default -> AuthFailureReason.FORBIDDEN;
    };
}
```

**429 handling:** `ProviderErrorMapping.mapStatusCode()` hardcodes
`DEFAULT_RETRY_AFTER_SECONDS = 30`. `GraphErrorHandler` bypasses
`ProviderErrorMapping` for 429 and constructs `ProviderRateLimitedException`
directly with the actual `Retry-After` header value:

```java
int retryAfter = parseRetryAfter(response.headers(), 30);
throw new ProviderRateLimitedException(retryAfter, providerId);
```

Graph API error responses include a JSON body with `error.code` and `error.message`.
`GraphErrorHandler.handle(HttpResponse<?> response)` extracts these before mapping.

## email-microsoft — EmailPlatform (Outlook Mail)

### SPI Capabilities

| EmailPlatform method | Graph API endpoint |
|---|---|
| `listMailboxes()` | `GET /me/mailFolders` |
| `listMessages(mailboxId, from, to, pagination)` | `GET /me/mailFolders/{id}/messages?$top=N&$filter=receivedDateTime ge ... and receivedDateTime lt ...` with `@odata.nextLink` continuation |
| `getMessage(mailboxId, messageId)` | `GET /me/messages/{id}` (with `$expand=attachments`) |
| `getAttachmentContent(mailboxId, messageId, attachmentId)` | `GET /me/messages/{id}/attachments/{id}/$value` (returns `byte[]`) |
| `search(query, pagination)` | `GET /me/messages?$search="query"` with `@odata.nextLink` continuation |

### MicrosoftMailClient

Domain client in `email-microsoft` module. Injects `MicrosoftGraphClient`.

```java
public class MicrosoftMailClient {
    private final MicrosoftGraphClient graph;

    public List<JsonObject> listMailFolders(String accessToken) { ... }
    public Page<JsonObject> listMessages(String folderId, String accessToken,
            Instant from, Instant to, int top, String cursor) { ... }
    public JsonObject getMessage(String messageId, String accessToken) { ... }
    public byte[] getAttachmentContent(String messageId, String attachmentId,
            String accessToken) { ... }
    public Page<JsonObject> searchMessages(String query, String accessToken,
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
| `listEvents(calendarId, from, to)` | `GET /me/calendars/{id}/calendarView?startDateTime=...&endDateTime=...` |
| `getEvent(calendarId, eventId)` | `GET /me/events/{id}` |
| `createEvent(calendarId, details)` | `POST /me/calendars/{id}/events` |
| `updateEvent(calendarId, eventId, details)` | `PATCH /me/events/{id}` |
| `deleteEvent(calendarId, eventId)` | `DELETE /me/events/{id}` |
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
| `FileOperations.download(fileId)` | `GET {basePath}/items/{fileId}/content` (302 → manual redirect, see below) |
| `FileOperations.upload(folderId, name, contentType, content)` | Simple: `PUT {basePath}/items/{folderId}:/{name}:/content` (≤4MB); Resumable: upload session (>4MB, see below) |
| `FileOperations.delete(fileId)` | `DELETE {basePath}/items/{fileId}` |
| `FileOperations.listSync(syncRequest)` | `GET {basePath}/root/delta` |
| `FolderOperations.list(parentId)` | `GET {basePath}/items/{parentId}/children` (filter folders, returns `List<Folder>` — no pagination) |
| `FolderOperations.create(parentId, name)` | `POST {basePath}/items/{parentId}/children` (folder facet) |
| `FolderOperations.move(fileId, targetFolderId)` | `PATCH {basePath}/items/{fileId}` (parentReference) |
| `SearchOperations.search(query, pagination)` | `POST {basePath}/root/search(q='query')` |
| `SharingOperations.getShareLink(fileId)` | `POST {basePath}/items/{fileId}/createLink` |

### Delta Sync

Graph Drive delta (`{basePath}/root/delta`) follows the same pattern as calendar:
- `@odata.deltaLink` for incremental token
- HTTP 410 → `SyncTokenExpiredException`
- Deleted items identified by `deleted` facet

### File Download — 302 Redirect Handling

Graph API file content endpoint (`GET {basePath}/items/{fileId}/content`)
returns HTTP 302 with a `Location` header pointing to a pre-authenticated
CDN URL. `HttpHelper.CLIENT` is constructed without `.followRedirects()`
and defaults to `Redirect.NEVER`.

`MicrosoftDriveClient.downloadContent()` handles the redirect explicitly
rather than using a separate `HttpClient` (which would violate protocol
PP-20260607-9794cb) or changing `HttpHelper.CLIENT` globally (which could
have unintended effects on existing connectors):

```java
public byte[] downloadContent(String itemId, String accessToken) {
    var response = graph.getRaw(basePath + "/items/" + itemId + "/content", accessToken);
    if (response.statusCode() == 302) {
        String location = response.headers().firstValue("Location").orElseThrow();
        return graph.getBytesFromUrl(location);
    }
    return response.body();
}
```

This adds `getRaw()` (returns `HttpResponse<byte[]>` without error handler
dispatch) and `getBytesFromUrl()` (GET on an absolute URL without
`BASE_URL` prefix and without `Authorization` header — the CDN URL is
pre-authenticated via query parameters) to `MicrosoftGraphClient`.

### File Upload — Size-Dependent Path

Graph API simple upload (`PUT .../content`) supports files up to 4MB.
Files larger than 4MB require a resumable upload session.

`MicrosoftDriveClient.uploadContent()` detects the boundary:

- **≤4MB:** Simple upload — `PUT {basePath}/items/{folderId}:/{name}:/content`
  with `Content-Type` set to the file's content type and the byte array as body.
- **>4MB:** Resumable upload session:
  1. `POST {basePath}/items/{folderId}:/{name}:/createUploadSession` → returns
     `uploadUrl`
  2. `PUT` byte ranges to `uploadUrl` in chunks (recommended chunk size:
     5–10MB, must be multiples of 320KiB)
  3. Final chunk returns the created `DriveItem` metadata

The 4MB threshold comes from Microsoft's documentation. The `byte[]`
parameter from the SPI supports arbitrarily large files (limited only by JVM
heap), so the upload implementation must handle both paths transparently.

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
    public Page<JsonObject> listChildren(String folderId, String accessToken,
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

**Limitation:** This scopes `SharePointDrivePlatform` to a single SharePoint
site per deployment. Multi-site support (organizations with dozens to hundreds
of sites) is deferred — see casehubio/connectors#218.

## chat-teams — ChatPlatform (Teams)

### Team Discovery

`Discovery.listChannels()` takes no parameters. Graph API requires a teamId
for `GET /teams/{id}/channels`. The approach follows the same pattern as
Discord (which discovers guilds dynamically via `client.listBotGuilds(token)`):

- With the delegated user token: `GET /me/joinedTeams` → list of teams the
  user belongs to
- Then `GET /teams/{id}/channels` for each joined team
- Channels are identified by compound `teamId:channelId` (consistent with
  the Channel Model below)

This scopes discovery to teams the authenticated user has access to — the
same implicit scoping that Discord uses for guilds.

### Capability Mapping

| ChatPlatform capability | Graph API | Permission type | Status |
|---|---|---|---|
| **Messaging** | `POST /teams/{id}/channels/{id}/messages` | Application (`ChannelMessage.Send`) | Native |
| **Threading** | `POST .../messages/{id}/replies` | Application (`ChannelMessage.Send`) | Native |
| **Discovery** | `GET /me/joinedTeams` → `GET /teams/{id}/channels` | Delegated (`Team.ReadBasic.All`, `Channel.ReadBasic.All`) | Native |
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
- Application: `getAccessToken(APPLICATION_ACTOR, "microsoft-app", tenancyId)` — tenant-scoped token

The `"microsoft-app"` provider name signals application-level credentials
(client_credentials grant).

**Application-level actorId:** Application credentials have no user. Rather
than passing `null` as actorId (which propagates null through
`OAuthTokenManagerCore` → `OAuthTokenStore.findByActorId()` and relies on
string concatenation of `"null"` for key construction), use a sentinel
constant:

```java
public static final String APPLICATION_ACTOR = "__application__";
```

Defined on `ServiceConnectionProvider` or in `microsoft-graph-client`. The
`OAuthTokenStore` record for application credentials uses this sentinel as
its actorId. This works with all existing `OAuthTokenStore` implementations
without modification — the sentinel is a normal non-null string.

**Prerequisite — `@RequiresScopes` must be `@Repeatable`:** The current
`@RequiresScopes` annotation takes a single `provider` string and is not
`@Repeatable`. Teams needs scopes registered for both `"microsoft"` (delegated)
and `"microsoft-app"` (application). This requires:
1. Create container annotation `@RequiresScopesList`
2. Add `@Repeatable(RequiresScopesList.class)` to `@RequiresScopes`
3. Update `AuthnBeans.scopeRegistryCore()` to use `getAnnotationsByType()`
4. Update `AuthnSpringAutoConfiguration` similarly

This is a platform-api change — see casehubio/platform#TBD.

### TeamsClient

```java
public class TeamsClient {
    private final MicrosoftGraphClient graph;

    // Application permission operations
    public JsonObject sendMessage(String teamId, String channelId,
            String content, String appAccessToken) { ... }
    public JsonObject replyToMessage(String teamId, String channelId,
            String messageId, String content, String appAccessToken) { ... }
    public Page<JsonObject> listMessages(String teamId, String channelId,
            String appAccessToken, String cursor) { ... }

    // Delegated permission operations
    public List<JsonObject> listJoinedTeams(String userAccessToken) { ... }
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
    "Team.ReadBasic.All", "Channel.ReadBasic.All", "Presence.Read.All",
    "TeamMember.Read.All", "Channel.Create", "Channel.Delete.All",
    "TeamMember.ReadWrite.All"
})
@RequiresScopes(provider = "microsoft-app", scopes = {
    "ChannelMessage.Send", "ChannelMessage.Read.All"
})
```

Requires `@RequiresScopes` to be `@Repeatable` (prerequisite above).

## Auth Model

### ServiceConnectionProvider Integration (D2)

All five providers inject `ServiceConnectionProvider` and call
`getAccessToken(actorId, "microsoft", tenancyId)` at the start of each
SPI method invocation. The returned `ServiceAccessToken.accessToken()` is
passed as the `accessToken` parameter to `MicrosoftGraphClient` methods.

### ActorId Resolution

The SPI methods (`listMailboxes()`, `listCalendars()`, etc.) take no
actorId parameter — this is consistent with all existing providers. The
actorId comes from the platform layer that invokes SPI methods, not from
the SPI itself.

The current Google providers (`GoogleEmailPlatform`, `GoogleCalendarPlatform`)
have the same gap: `GoogleCalendarPlatform.buildService(actorId)` exists but
has no call sites. ActorId resolution is an open platform-level design
problem, not specific to this spec.

For implementation, each Microsoft platform impl follows the same pattern
as the Google providers: inject `ServiceConnectionProvider`, resolve the
token within each SPI method, and pass it down. The mechanism for obtaining
actorId (CDI `@RequestScoped` context, interceptor injection, or another
platform mechanism) will be resolved as a cross-cutting platform concern.

```java
// In each platform impl — actorId source TBD at platform level
private String resolveToken(String actorId) {
    return connectionProvider.getAccessToken(actorId, "microsoft", "default")
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
`ProviderAuthException(TOKEN_EXPIRED)`. All five platform impls need the
same retry pattern: catch `TOKEN_EXPIRED`, re-resolve the token, retry once.

Shared utility in `microsoft-graph-client`:

```java
public class TokenRetry {
    public static <T> T withTokenRetry(Supplier<String> tokenSupplier,
                                        Function<String, T> operation) {
        try {
            return operation.apply(tokenSupplier.get());
        } catch (ProviderAuthException e) {
            if (e.reason() == AuthFailureReason.TOKEN_EXPIRED) {
                return operation.apply(tokenSupplier.get());
            }
            throw e;
        }
    }

    public static void withTokenRetryVoid(Supplier<String> tokenSupplier,
                                           Consumer<String> operation) {
        try {
            operation.accept(tokenSupplier.get());
        } catch (ProviderAuthException e) {
            if (e.reason() == AuthFailureReason.TOKEN_EXPIRED) {
                operation.accept(tokenSupplier.get());
                return;
            }
            throw e;
        }
    }
}
```

`withTokenRetry` for operations that return a value (getMessage, listEvents,
etc.). `withTokenRetryVoid` for void operations (deleteEvent, delete, move).

Platform impls use this uniformly:

```java
// In MicrosoftEmailPlatform
public EmailMessage getMessage(String mailboxId, String messageId) {
    return TokenRetry.withTokenRetry(
        () -> resolveToken(actorId),
        token -> MailMapper.toEmailMessage(
            mailClient.getMessage(messageId, token), mailboxId));
}
```

## Error Handling

### ProviderErrorMapping Reuse

`GraphErrorHandler` extracts HTTP status code, response body, and Graph
error code from responses. For most status codes (401, 404, 410, 5xx),
it delegates to `ProviderErrorMapping.mapStatusCode(statusCode, body,
providerId, errorCode)`. For 403 and 429, it handles mapping directly
(see GraphErrorHandler section above for details and rationale).

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

## Scope Divergence from Issue #211

Issue #211 lists `document-sharepoint` as a separate module. This spec merges
it into `document-microsoft` (D3 revised) — the same Graph API drive operations
with a different base path do not justify a separate module. The issue's
`document-sharepoint` checkbox maps to `SharePointDrivePlatform` within
`document-microsoft`.

Issue #211 does not include `chat-teams`. This spec adds it (D4) because the
shared client and credential infrastructure will be built regardless, and
completing the Microsoft integration in one pass avoids a second context-loading
cost. The issue scope items should be updated accordingly.

## Decisions Summary

| ID | Decision | Status |
|---|---|---|
| D1 | Raw HTTP via `HttpHelper.CLIENT` — no Microsoft Graph SDK | Captured |
| D2 | `ServiceConnectionProvider` for credential resolution | Captured |
| D3 | Single `document-microsoft` module with two CDI beans (revised from separate modules) | Revised |
| D4 | Include `chat-teams` in scope | Captured |
| D5 | Thin base client + domain-specific clients per module | Captured |
| D6 | Composed `MicrosoftDriveClient` with basePath parameter (revised from abstract base) | Revised |
| D7 | Per-provider `@RequiresScopes` declarations | Captured |

Full decision rationale: `wksp/specs/issue-211-microsoft-graph-providers/decisions.md`

## Deliverables

### ARC42STORIES Chapter

Adding 5 new modules and new layers requires a new Chapter entry in
`ARC42STORIES.MD` — following the pattern established by Chapters 1–11.

**Chapter title:** Microsoft Graph Providers
**Layers touched:** Shared HTTP Client, Email, Calendar, Document, Chat
**What this delivers:** Five Platform SPI providers backed by Microsoft Graph
REST API, completing the second provider family (alongside Google) for email,
calendar, and documents, plus the first Teams integration for chat.

## Implementation Order

1. `microsoft-graph-client` — shared HTTP client, pagination, error handling
2. `email-microsoft` — highest n8n deploys (96.3K), exercises all client features
3. `calendar-microsoft` — delta sync validates the sync pattern
4. `document-microsoft` — OneDrive + SharePoint, composed DriveClient
5. `chat-teams` — dual permission model, most complex auth

Each step is independently testable and deployable.

## Prerequisites

1. **`@RequiresScopes` made `@Repeatable`** — required for `chat-teams` dual-provider
   scope declarations (platform-api change)
2. **ActorId resolution mechanism** — platform-level design for how SPI methods
   obtain the current user identity (affects all providers, not just Microsoft)

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
