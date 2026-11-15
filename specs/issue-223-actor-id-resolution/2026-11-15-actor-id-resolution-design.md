# Cross-SPI actorId Resolution Mechanism

**Issue:** casehubio/connectors#223
**Date:** 2026-11-15
**Status:** Design

## Problem

Three platform SPIs — CalendarPlatform, DocumentPlatform, EmailPlatform — have flat
method signatures with no user context parameter. Their implementations pass `null` as
actorId to `ServiceConnectionProvider.getAccessToken()`, which blocks multi-user
scenarios where different users have different OAuth tokens.

The platform already has the infrastructure for per-user token resolution:
- `ServiceConnectionProvider.getAccessToken(String actorId, String provider, String tenancyId)` accepts actorId
- `CurrentPrincipal` (request-scoped CDI bean) provides `actorId()` and `tenancyId()`
- Newer SPIs (ContactsPlatform, BankPlatform, ProjectPlatform, CommercePlatform, TravelPlatform, HealthPlatform, MobilePlatform) use user-scoped capability accessors that thread userId through

The gap is that the three older SPIs don't carry user context.

## Solution

Retrofit the three flat SPIs with user-scoped capability accessors, following the
established pattern. Extract `currentPrincipal.actorId()` at the MCP/GraphQL boundary
and pass it as userId to the SPI accessors.

## SPI Changes

### CalendarPlatform

Current flat methods become a `CalendarOperations` inner interface behind a
user-scoped accessor:

```java
@SimulationEligible(name = "calendar-platform")
public interface CalendarPlatform extends PlatformExtensible {

    String id();

    CalendarOperations calendarOps(String userId);

    interface CalendarOperations {
        List<CalendarInfo> listCalendars();
        List<CalendarEvent> listEvents(String calendarId, Instant from, Instant to);
        CalendarEvent getEvent(String calendarId, String eventId);
        CalendarEvent createEvent(String calendarId, EventDetails details);
        CalendarEvent updateEvent(String calendarId, String eventId, EventDetails details);
        void deleteEvent(String calendarId, String eventId);
        SyncResult<CalendarEvent> listEventsSync(String calendarId, SyncRequest request);
    }
}
```

### DocumentPlatform

Already has capability sub-interfaces. Add userId to existing accessors:

```java
@SimulationEligible(name = "document-platform",
    capabilities = {"fileOperations", "folderOperations", "searchOperations", "sharingOperations"})
public interface DocumentPlatform extends PlatformExtensible {

    String id();

    boolean supports(Class<?> capability);

    FileOperations files(String userId);

    FolderOperations folders(String userId);

    SearchOperations search(String userId);

    SharingOperations sharing(String userId);

    // Inner interfaces unchanged
}
```

### EmailPlatform

Current flat methods become an `EmailOperations` inner interface:

```java
@SimulationEligible(name = "email-platform")
public interface EmailPlatform extends PlatformExtensible {

    String id();

    EmailOperations emailOps(String userId);

    interface EmailOperations {
        List<Mailbox> listMailboxes();
        Page<EmailSummary> listMessages(String mailboxId, Instant from, Instant to,
                                        PageRequest pagination);
        EmailMessage getMessage(String mailboxId, String messageId);
        byte[] getAttachmentContent(String mailboxId, String messageId, String attachmentId);
        Page<EmailSummary> search(String query, PageRequest pagination);
    }
}
```

## Implementation Changes

### Google Implementations

**GoogleCalendarPlatform:** Already has `buildService(String actorId)`. The
`calendarOps(userId)` accessor returns a new inner class that calls
`buildService(userId)` once and passes the resulting `Calendar` client to all
operations. Removes the singleton `calendarService` field and `requireClient()`.

**GoogleDocumentPlatform:** Already has inner classes (`GoogleFileOperations`, etc.)
but they reference a shared singleton `driveService`. Change the accessors to accept
userId, call a new `buildService(userId)` method (pattern from GoogleCalendarPlatform),
and construct fresh inner class instances with the per-user Drive client. Removes
`requireClient()`.

**GoogleEmailPlatform:** Same pattern — add `buildService(String actorId)` that
resolves a token via `connectionProvider.getAccessToken(actorId, "google", ...)` and
builds a per-user Gmail client. `emailOps(userId)` returns a new inner class wrapping
the per-user client. Removes `requireClient()`.

### Microsoft Implementations

All three have an identical `resolveToken()` method that passes `null` actorId.

**MicrosoftCalendarPlatform:** Change `resolveToken()` to `resolveToken(String userId)`.
`calendarOps(userId)` returns inner class that passes userId through to token resolution.

**MicrosoftDrivePlatform:** Already has inner classes. Change `resolveToken()` to
`resolveToken(String userId)`. Accessors `files(userId)`, `folders(userId)`, etc.
construct inner classes that capture userId and pass it through to token resolution.

**MicrosoftEmailPlatform:** Change `resolveToken()` to `resolveToken(String userId)`.
`emailOps(userId)` returns inner class that passes userId through.

### Reference Implementations

**RefCalendarPlatform, RefDocumentPlatform, RefEmailPlatform:** Accept userId parameter
on accessors but ignore it — in-memory implementations don't use OAuth.
RefDocumentPlatform already has inner classes; RefCalendarPlatform and RefEmailPlatform
need inner class wrappers.

### NoOp Implementations

**NoOpCalendarPlatform, NoOpDocumentPlatform, NoOpEmailPlatform:** Accept userId and
return NoOp inner implementations. NoOpDocumentPlatform already returns NoOp enum
instances from accessors — just add the userId parameter.

### Simulation Wrappers

The `@SimulationEligible` annotation processor generates simulation wrappers. These
need to thread userId through the accessor methods. Since the processor generates
delegation code, this should be automatic once the SPI interface changes — the
generated wrapper delegates to the real implementation's accessor with the same userId.

### MCP/GraphQL Boundary

**ConnectorOperationsImpl:** Already has `CurrentPrincipal currentPrincipal` injected.
When calling platform SPIs that now require userId, extract
`currentPrincipal.actorId()` and pass it.

**Generated GraphQL resolvers:** The `GraphQLResolverProcessor` generates resolvers
from `ConnectorOperations` SPI. Calendar/Document/Email operations in the generated
code call the platform SPIs. The generated resolvers already have access to
`CurrentPrincipal` via `ConnectorOperationsImpl`. When the generated code calls
e.g. `calendarPlatform.calendarOps(userId).listCalendars()`, the userId comes from
`currentPrincipal.actorId()` resolved in `ConnectorOperationsImpl`.

### CalendarPlatformService / DocumentPlatformService / EmailPlatformService

These are simple registries (`Map<String, Platform>`) that return platform instances
by id. They don't need userId — they just look up the platform. The userId is passed
when calling the user-scoped accessor on the returned platform.

## Modules Affected

| Module | Change |
|--------|--------|
| `calendar-spi` | SPI interface restructure |
| `calendar-ref` | Implement new accessor |
| `calendar-google` | Use buildService(userId) via accessor |
| `calendar-microsoft` | Pass userId to resolveToken |
| `document-spi` | Add userId to accessors |
| `document-ref` | Accept userId (ignore) |
| `document-google` | Build per-user Drive client |
| `document-microsoft` | Pass userId to resolveToken |
| `email-spi` | SPI interface restructure |
| `email-ref` | Implement new accessor |
| `email-google` | Build per-user Gmail client |
| `email-microsoft` | Pass userId to resolveToken |
| `graphql` | Pass currentPrincipal.actorId() to accessors |

Plus test classes in each module.

## Testing

- Unit tests in each implementation module verify that userId flows through to
  `ServiceConnectionProvider.getAccessToken()` — mock the provider, assert the
  actorId parameter.
- Existing functional tests in ref implementations continue to work (pass any userId).
- Integration tests verify the MCP/GraphQL path resolves currentPrincipal.actorId().

## Out of Scope

- ChatPlatform, LocationPlatform, VoicePlatform — these use API keys or bot tokens,
  not per-user OAuth. No change needed.
- MobilePlatform, CommercePlatform, TravelPlatform, HealthPlatform, ProjectPlatform,
  BankPlatform, ContactsPlatform — already have user-scoped accessors.
- `email-google-inbound` — uses `connectionProvider.getAccessToken(emailAddress, ...)`
  directly for Gmail push notifications. This is an inbound connector, not the
  EmailPlatform SPI — it already passes a meaningful actorId (the email address).
- Token caching, connection pooling, or per-user client reuse — separate concern,
  can be added later inside implementations without SPI changes.

## References

- CalendarPlatform.java (calendar-spi) — current flat SPI
- DocumentPlatform.java (document-spi) — current flat-accessor SPI
- EmailPlatform.java (email-spi) — current flat SPI
- ContactsPlatform.java (contacts-spi) — established user-scoped accessor pattern
- ServiceConnectionProvider.java (platform-api) — getAccessToken(actorId, provider, tenancyId)
- CurrentPrincipal.java (platform-api) — request-scoped identity
- ConnectorOperationsImpl.java (graphql) — MCP/GraphQL entry point
- GitHub issue casehubio/connectors#223
