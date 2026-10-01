# Retrofit Sync Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #129 — feat: retrofit sync support for CalendarPlatform and DocumentPlatform
**Issue group:** #129

**Goal:** Add incremental sync to CalendarPlatform and DocumentPlatform using existing `SyncResult<T>`/`SyncRequest`/`SyncTokenExpiredException` primitives from `connectors-api` (#125).

**Architecture:** All changes are additive — existing SPI methods stay unchanged. Each platform gets a new sync method that follows the established pattern from ContactsPlatform: initial full sync (null token) → store token → incremental sync (pass token) → repeat. HTTP 410 / token expiry → throw `SyncTokenExpiredException` → caller falls back to full resync. CalendarPlatform also gets user-scoped accessors to align with ContactsPlatform and BankPlatform patterns.

**Tech Stack:** Java 21, Quarkus 3.32.2, Google Calendar API, Google Drive API, WireMock (tests)

## Global Constraints

- All changes additive — no breaking changes to existing SPI methods
- `SyncResult<T>`, `SyncRequest`, `SyncTokenExpiredException` are in `connectors-api` (already merged via #125)
- Follow `paginating-client-fail-soft` protocol: mid-loop failures return accumulated results + WARNING
- Follow `spi-id-method-naming` protocol: identifier methods are `id()`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Tests: JUnit 5 + AssertJ + WireMock for Google API tests

---

## Batch 1: Calendar sync — SPI + ref implementation

### Task 1: Add `listEventsSync` to CalendarPlatform SPI with ref implementation

**Files:**
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/CalendarPlatform.java:12-27`
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/NoOpCalendarPlatform.java:15-53`
- Modify: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/CalendarBackend.java:10-23`
- Modify: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/InMemoryCalendarBackend.java:19-95`
- Modify: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/RefCalendarPlatform.java:11-53`
- Test: `calendar-ref/src/test/java/io/casehub/connectors/calendar/ref/RefCalendarPlatformTest.java`
- Test: `calendar-spi/src/test/java/io/casehub/connectors/calendar/NoOpCalendarPlatformTest.java`

**Interfaces:**
- Consumes: `SyncResult<T>`, `SyncRequest`, `SyncTokenExpiredException` from `connectors-api`
- Produces: `CalendarPlatform.listEventsSync(String calendarId, SyncRequest request)` returning `SyncResult<CalendarEvent>`

- [ ] **Step 1: Write failing test — initial sync returns all events with token**

Add to `RefCalendarPlatformTest.java`:

```java
@Test
void listEventsSync_initialSync_returnsAllEventsWithToken() {
    var details = new EventDetails("Sync me", null, null,
            new EventTiming.Timed(
                    Instant.parse("2026-07-26T10:00:00Z"),
                    Instant.parse("2026-07-26T11:00:00Z"),
                    ZoneId.of("UTC")),
            List.of());
    platform.createEvent("primary", details);

    var result = platform.listEventsSync("primary", SyncRequest.initial(100));

    assertThat(result.items()).hasSize(1);
    assertThat(result.items().getFirst().summary()).isEqualTo("Sync me");
    assertThat(result.syncToken()).isNotNull();
    assertThat(result.deletedIds()).isEmpty();
    assertThat(result.hasMore()).isFalse();
}
```

- [ ] **Step 2: Write failing test — incremental sync detects new events**

```java
@Test
void listEventsSync_incrementalSync_detectsNewEvents() {
    platform.createEvent("primary", new EventDetails("Initial", null, null,
            new EventTiming.Timed(
                    Instant.parse("2026-07-26T10:00:00Z"),
                    Instant.parse("2026-07-26T11:00:00Z"),
                    ZoneId.of("UTC")),
            List.of()));

    var fullSync = platform.listEventsSync("primary", SyncRequest.initial(100));
    var token = fullSync.syncToken();

    platform.createEvent("primary", new EventDetails("New event", null, null,
            new EventTiming.Timed(
                    Instant.parse("2026-07-27T10:00:00Z"),
                    Instant.parse("2026-07-27T11:00:00Z"),
                    ZoneId.of("UTC")),
            List.of()));

    var incrementalSync = platform.listEventsSync("primary", new SyncRequest(token, 100));
    assertThat(incrementalSync.items()).hasSize(1);
    assertThat(incrementalSync.items().getFirst().summary()).isEqualTo("New event");
}
```

- [ ] **Step 3: Write failing test — incremental sync detects deletes**

```java
@Test
void listEventsSync_incrementalSync_detectsDeletes() {
    var created = platform.createEvent("primary", new EventDetails("Delete me", null, null,
            new EventTiming.Timed(
                    Instant.parse("2026-07-26T10:00:00Z"),
                    Instant.parse("2026-07-26T11:00:00Z"),
                    ZoneId.of("UTC")),
            List.of()));

    var fullSync = platform.listEventsSync("primary", SyncRequest.initial(100));
    var token = fullSync.syncToken();

    platform.deleteEvent("primary", created.id());

    var incrementalSync = platform.listEventsSync("primary", new SyncRequest(token, 100));
    assertThat(incrementalSync.deletedIds()).contains(created.id());
}
```

- [ ] **Step 4: Write failing test — no changes returns empty incremental sync**

```java
@Test
void listEventsSync_noChanges_returnsEmpty() {
    platform.createEvent("primary", new EventDetails("Stable", null, null,
            new EventTiming.Timed(
                    Instant.parse("2026-07-26T10:00:00Z"),
                    Instant.parse("2026-07-26T11:00:00Z"),
                    ZoneId.of("UTC")),
            List.of()));

    var fullSync = platform.listEventsSync("primary", SyncRequest.initial(100));
    var token = fullSync.syncToken();

    var incrementalSync = platform.listEventsSync("primary", new SyncRequest(token, 100));
    assertThat(incrementalSync.items()).isEmpty();
    assertThat(incrementalSync.deletedIds()).isEmpty();
}
```

- [ ] **Step 5: Write failing test — NoOp throws UnsupportedOperationException**

Add to `NoOpCalendarPlatformTest.java`:

```java
@Test
void listEventsSync_throwsUnsupported() {
    var platform = new NoOpCalendarPlatform();
    assertThatThrownBy(() -> platform.listEventsSync("primary", SyncRequest.initial(100)))
            .isInstanceOf(UnsupportedOperationException.class);
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-spi,calendar-ref -Dtest="RefCalendarPlatformTest,NoOpCalendarPlatformTest" -DfailIfNoTests=false`
Expected: compilation error — `listEventsSync` does not exist

- [ ] **Step 7: Add `listEventsSync` to CalendarPlatform SPI**

Add to `CalendarPlatform.java` after `deleteEvent`:

```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
```

```java
SyncResult<CalendarEvent> listEventsSync(String calendarId, SyncRequest request);
```

- [ ] **Step 8: Add `listEventsSync` to NoOpCalendarPlatform**

Add to `NoOpCalendarPlatform.java`:

```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
```

```java
@Override
public SyncResult<CalendarEvent> listEventsSync(final String calendarId,
        final SyncRequest request) {
    throw new UnsupportedOperationException("No calendar provider configured");
}
```

- [ ] **Step 9: Add version tracking to CalendarBackend**

Add to `CalendarBackend.java`:

```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.calendar.model.CalendarEvent;
```

```java
long currentVersion();

List<CalendarEvent> changedSince(String calendarId, long version);

List<String> deletedSince(String calendarId, long version);
```

- [ ] **Step 10: Implement version tracking in InMemoryCalendarBackend**

Replace the `events` field and add version tracking fields. Wrap stored events with version:

```java
import java.util.Map;
import java.util.concurrent.atomic.AtomicLong;

// Add fields:
private final AtomicLong version = new AtomicLong(0);
private final Map<String, Long> deletedVersions = new ConcurrentHashMap<>();
```

Change the storage from `ConcurrentHashMap<String, List<CalendarEvent>>` to `ConcurrentHashMap<String, List<VersionedEvent>>` where:

```java
private record VersionedEvent(CalendarEvent event, long version) {}
```

Update `createEvent` to increment version, `updateEvent` to increment version, `deleteEvent` to record deletion version. Add `currentVersion()`, `changedSince(calendarId, version)`, and `deletedSince(calendarId, version)` implementations.

Key logic for `changedSince`:
```java
@Override
public List<CalendarEvent> changedSince(String calendarId, long sinceVersion) {
    return events.getOrDefault(calendarId, List.of()).stream()
            .filter(ve -> ve.version() > sinceVersion)
            .map(VersionedEvent::event)
            .toList();
}
```

Key logic for `deletedSince`:
```java
@Override
public List<String> deletedSince(String calendarId, long sinceVersion) {
    return deletedVersions.entrySet().stream()
            .filter(e -> e.getValue() > sinceVersion)
            .map(Map.Entry::getKey)
            .toList();
}
```

Existing methods (`listEvents`, `getEvent`) must adapt to read through `VersionedEvent::event`.

- [ ] **Step 11: Implement `listEventsSync` in RefCalendarPlatform**

Add to `RefCalendarPlatform.java`:

```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
```

```java
@Override
public SyncResult<CalendarEvent> listEventsSync(String calendarId, SyncRequest request) {
    long sinceVersion = request.syncToken() != null
        ? Long.parseLong(request.syncToken()) : 0;
    var changed = backend.changedSince(calendarId, sinceVersion);
    var deleted = backend.deletedSince(calendarId, sinceVersion);
    return new SyncResult<>(changed, deleted,
        String.valueOf(backend.currentVersion()), false);
}
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-spi,calendar-ref -Dtest="RefCalendarPlatformTest,NoOpCalendarPlatformTest" -DfailIfNoTests=false`
Expected: all PASS

- [ ] **Step 13: Fix GoogleCalendarPlatform compilation**

`GoogleCalendarPlatform` implements `CalendarPlatform` but doesn't have `listEventsSync` yet. Add a stub that throws `UnsupportedOperationException` to keep the build green:

```java
@Override
public SyncResult<CalendarEvent> listEventsSync(String calendarId, SyncRequest request) {
    throw new UnsupportedOperationException("Google Calendar sync not yet implemented");
}
```

(Task 3 replaces this stub with the real implementation.)

- [ ] **Step 14: Full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 15: Commit**

```bash
git add calendar-spi/ calendar-ref/ calendar-google/
git commit -m "feat(#129): add listEventsSync to CalendarPlatform SPI with ref implementation

Add incremental sync support to CalendarPlatform using SyncResult/SyncRequest
primitives. InMemoryCalendarBackend tracks event versions for change detection.
GoogleCalendarPlatform has a temporary stub pending Task 3 wiring.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Document sync — SPI + ref implementation

### Task 2: Add `listSync` to FileOperations with ref implementation

**Files:**
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java:27-39`
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/NoOpDocumentPlatform.java:49-77`
- Modify: `document-ref/src/main/java/io/casehub/connectors/document/ref/DocumentBackend.java:11-33`
- Modify: `document-ref/src/main/java/io/casehub/connectors/document/ref/InMemoryDocumentBackend.java:21-199`
- Modify: `document-ref/src/main/java/io/casehub/connectors/document/ref/RefDocumentPlatform.java:61-87`
- Test: `document-ref/src/test/java/io/casehub/connectors/document/ref/RefDocumentPlatformTest.java`

**Interfaces:**
- Consumes: `SyncResult<T>`, `SyncRequest`, `SyncTokenExpiredException` from `connectors-api`
- Produces: `DocumentPlatform.FileOperations.listSync(SyncRequest request)` returning `SyncResult<DocumentSummary>`

- [ ] **Step 1: Write failing test — initial sync returns all files with token**

Add to `RefDocumentPlatformTest.java`:

```java
@Test
void listSync_initialSync_returnsAllFilesWithToken() {
    var result = platform.files().listSync(SyncRequest.initial(100));

    assertThat(result.items()).hasSize(7);  // pre-loaded test data
    assertThat(result.syncToken()).isNotNull();
    assertThat(result.deletedIds()).isEmpty();
    assertThat(result.hasMore()).isFalse();
}
```

- [ ] **Step 2: Write failing test — incremental sync detects uploads**

```java
@Test
void listSync_incrementalSync_detectsUploads() {
    var fullSync = platform.files().listSync(SyncRequest.initial(100));
    var token = fullSync.syncToken();

    platform.files().upload("folder-docs", "new-file.txt", "text/plain",
            "hello".getBytes());

    var incrementalSync = platform.files().listSync(new SyncRequest(token, 100));
    assertThat(incrementalSync.items()).hasSize(1);
    assertThat(incrementalSync.items().getFirst().name()).isEqualTo("new-file.txt");
}
```

- [ ] **Step 3: Write failing test — incremental sync detects deletes**

```java
@Test
void listSync_incrementalSync_detectsDeletes() {
    var fullSync = platform.files().listSync(SyncRequest.initial(100));
    var token = fullSync.syncToken();

    platform.files().delete("doc-001");

    var incrementalSync = platform.files().listSync(new SyncRequest(token, 100));
    assertThat(incrementalSync.deletedIds()).contains("doc-001");
}
```

- [ ] **Step 4: Write failing test — no changes returns empty**

```java
@Test
void listSync_noChanges_returnsEmpty() {
    var fullSync = platform.files().listSync(SyncRequest.initial(100));
    var token = fullSync.syncToken();

    var incrementalSync = platform.files().listSync(new SyncRequest(token, 100));
    assertThat(incrementalSync.items()).isEmpty();
    assertThat(incrementalSync.deletedIds()).isEmpty();
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-spi,document-ref -Dtest="RefDocumentPlatformTest" -DfailIfNoTests=false`
Expected: compilation error — `listSync` does not exist

- [ ] **Step 6: Add `listSync` to FileOperations interface**

Add to `DocumentPlatform.FileOperations` after `delete`:

```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.document.model.DocumentSummary;
```

```java
SyncResult<DocumentSummary> listSync(SyncRequest request);
```

- [ ] **Step 7: Add `listSync` to NoOpDocumentPlatform.NoOpFileOperations**

```java
@Override
public SyncResult<DocumentSummary> listSync(SyncRequest request) {
    throw new UnsupportedOperationException(NOT_CONFIGURED);
}
```

- [ ] **Step 8: Add version tracking to DocumentBackend**

Add to `DocumentBackend.java`:

```java
long currentVersion();

List<DocumentSummary> changedSince(long version);

List<String> deletedSince(long version);
```

- [ ] **Step 9: Implement version tracking in InMemoryDocumentBackend**

Add version tracking fields:

```java
import java.util.concurrent.atomic.AtomicLong;

private final AtomicLong version = new AtomicLong(0);
private final Map<String, Long> fileVersions = new LinkedHashMap<>();
private final Map<String, Long> deletedVersions = new LinkedHashMap<>();
```

In `loadData()` — after each `addFile()` call, record the file version:
```java
private void addFile(...) {
    // ... existing code ...
    fileVersions.put(id, version.incrementAndGet());
}
```

In `uploadFile()` — record version on upload:
```java
fileVersions.put(id, version.incrementAndGet());
```

In `deleteFile()` — record deletion version:
```java
deletedVersions.put(fileId, version.incrementAndGet());
fileVersions.remove(fileId);
```

In `moveFile()` — increment version (move is a change):
```java
fileVersions.put(fileId, version.incrementAndGet());
```

Implement the new methods:
```java
@Override
public long currentVersion() {
    return version.get();
}

@Override
public List<DocumentSummary> changedSince(long sinceVersion) {
    return fileVersions.entrySet().stream()
            .filter(e -> e.getValue() > sinceVersion)
            .map(e -> {
                var meta = getFile(e.getKey());
                return new DocumentSummary(meta.id(), meta.name(), meta.folderId(),
                        meta.contentType(), meta.size(), meta.createdAt(), meta.modifiedAt());
            })
            .toList();
}

@Override
public List<String> deletedSince(long sinceVersion) {
    return deletedVersions.entrySet().stream()
            .filter(e -> e.getValue() > sinceVersion)
            .map(Map.Entry::getKey)
            .toList();
}
```

- [ ] **Step 10: Wire `listSync` in RefDocumentPlatform.RefFileOperations**

Add to `RefFileOperations`:

```java
@Override
public SyncResult<DocumentSummary> listSync(SyncRequest request) {
    long sinceVersion = request.syncToken() != null
        ? Long.parseLong(request.syncToken()) : 0;
    var changed = backend.changedSince(sinceVersion);
    var deleted = backend.deletedSince(sinceVersion);
    return new SyncResult<>(changed, deleted,
        String.valueOf(backend.currentVersion()), false);
}
```

- [ ] **Step 11: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-spi,document-ref -Dtest="RefDocumentPlatformTest" -DfailIfNoTests=false`
Expected: all PASS

- [ ] **Step 12: Fix GoogleDocumentPlatform compilation**

Add stub to `GoogleDocumentPlatform.GoogleFileOperations`:

```java
@Override
public SyncResult<DocumentSummary> listSync(SyncRequest request) {
    throw new UnsupportedOperationException("Google Drive sync not yet implemented");
}
```

- [ ] **Step 13: Full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 14: Commit**

```bash
git add document-spi/ document-ref/ document-google/
git commit -m "feat(#129): add listSync to DocumentPlatform.FileOperations with ref implementation

Add incremental sync to FileOperations using SyncResult/SyncRequest primitives.
InMemoryDocumentBackend tracks file versions for change detection.
GoogleDocumentPlatform has a temporary stub pending Task 4 wiring.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Google implementations

### Task 3: Wire syncToken on GoogleCalendarPlatform

**Files:**
- Modify: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java:24-179`
- Test: `calendar-google/src/test/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatformTest.java`

**Interfaces:**
- Consumes: `CalendarPlatform.listEventsSync(String, SyncRequest)` from Task 1
- Produces: Working Google Calendar sync via `events.list` with `syncToken`/`nextSyncToken` parameters

Google Calendar `events.list` supports:
- No syncToken: returns all events + `nextSyncToken` in response
- With syncToken: returns changes since that token (cancelled events appear with `status: "cancelled"`)
- HTTP 410: sync token expired → throw `SyncTokenExpiredException`
- When using syncToken, `timeMin`/`timeMax`/`singleEvents` must NOT be set

- [ ] **Step 1: Write failing test — initial sync returns events with syncToken**

Add to `GoogleCalendarPlatformTest.java`:

```java
@Test
void listEventsSync_initialSync_returnsEventsWithToken() {
    wireMock.stubFor(get(urlPathEqualTo("/calendar/v3/calendars/primary/events"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {
                              "kind": "calendar#events",
                              "items": [
                                {
                                  "id": "evt-1",
                                  "summary": "Standup",
                                  "start": {"dateTime": "2026-07-26T10:00:00Z", "timeZone": "UTC"},
                                  "end": {"dateTime": "2026-07-26T10:30:00Z", "timeZone": "UTC"}
                                }
                              ],
                              "nextSyncToken": "sync-token-1"
                            }
                            """)));

    var result = platform.listEventsSync("primary", SyncRequest.initial(100));

    assertThat(result.items()).hasSize(1);
    assertThat(result.items().getFirst().summary()).isEqualTo("Standup");
    assertThat(result.syncToken()).isEqualTo("sync-token-1");
    assertThat(result.deletedIds()).isEmpty();
}
```

- [ ] **Step 2: Write failing test — incremental sync with token returns changes and cancelled events as deletes**

```java
@Test
void listEventsSync_incrementalSync_returnsCancelledAsDeletes() {
    wireMock.stubFor(get(urlPathEqualTo("/calendar/v3/calendars/primary/events"))
            .withQueryParam("syncToken", WireMock.equalTo("sync-token-1"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {
                              "kind": "calendar#events",
                              "items": [
                                {
                                  "id": "evt-2",
                                  "summary": "New event",
                                  "start": {"dateTime": "2026-07-27T10:00:00Z", "timeZone": "UTC"},
                                  "end": {"dateTime": "2026-07-27T11:00:00Z", "timeZone": "UTC"}
                                },
                                {
                                  "id": "evt-old",
                                  "status": "cancelled"
                                }
                              ],
                              "nextSyncToken": "sync-token-2"
                            }
                            """)));

    var result = platform.listEventsSync("primary",
            new SyncRequest("sync-token-1", 100));

    assertThat(result.items()).hasSize(1);
    assertThat(result.items().getFirst().summary()).isEqualTo("New event");
    assertThat(result.deletedIds()).containsExactly("evt-old");
    assertThat(result.syncToken()).isEqualTo("sync-token-2");
}
```

- [ ] **Step 3: Write failing test — HTTP 410 throws SyncTokenExpiredException**

```java
@Test
void listEventsSync_tokenExpired_throwsSyncTokenExpired() {
    wireMock.stubFor(get(urlPathEqualTo("/calendar/v3/calendars/primary/events"))
            .withQueryParam("syncToken", WireMock.equalTo("expired-token"))
            .willReturn(aResponse().withStatus(410)));

    assertThatThrownBy(() -> platform.listEventsSync("primary",
            new SyncRequest("expired-token", 100)))
            .isInstanceOf(SyncTokenExpiredException.class);
}
```

- [ ] **Step 4: Write failing test — pagination collects all pages**

```java
@Test
void listEventsSync_pagination_collectsAllPages() {
    wireMock.stubFor(get(urlPathEqualTo("/calendar/v3/calendars/primary/events"))
            .withQueryParam("pageToken", WireMock.absent())
            .withQueryParam("syncToken", WireMock.absent())
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {
                              "kind": "calendar#events",
                              "items": [
                                {
                                  "id": "evt-1",
                                  "summary": "Page 1",
                                  "start": {"dateTime": "2026-07-26T10:00:00Z", "timeZone": "UTC"},
                                  "end": {"dateTime": "2026-07-26T11:00:00Z", "timeZone": "UTC"}
                                }
                              ],
                              "nextPageToken": "page2"
                            }
                            """)));

    wireMock.stubFor(get(urlPathEqualTo("/calendar/v3/calendars/primary/events"))
            .withQueryParam("pageToken", WireMock.equalTo("page2"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {
                              "kind": "calendar#events",
                              "items": [
                                {
                                  "id": "evt-2",
                                  "summary": "Page 2",
                                  "start": {"dateTime": "2026-07-27T10:00:00Z", "timeZone": "UTC"},
                                  "end": {"dateTime": "2026-07-27T11:00:00Z", "timeZone": "UTC"}
                                }
                              ],
                              "nextSyncToken": "sync-token-final"
                            }
                            """)));

    var result = platform.listEventsSync("primary", SyncRequest.initial(100));

    assertThat(result.items()).hasSize(2);
    assertThat(result.syncToken()).isEqualTo("sync-token-final");
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google -Dtest="GoogleCalendarPlatformTest" -DfailIfNoTests=false`
Expected: FAIL (stub throws UnsupportedOperationException)

- [ ] **Step 6: Implement `listEventsSync` in GoogleCalendarPlatform**

Replace the stub with:

```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
import io.casehub.connectors.SyncTokenExpiredException;
import com.google.api.client.googleapis.json.GoogleJsonResponseException;
```

```java
@Override
public SyncResult<CalendarEvent> listEventsSync(String calendarId, SyncRequest request) {
    requireClient();
    List<CalendarEvent> items = new ArrayList<>();
    List<String> deletedIds = new ArrayList<>();
    String syncToken = null;
    try {
        String pageToken = null;
        int page = 0;
        while (page < MAX_PAGES) {
            var req = calendarService.events().list(calendarId)
                    .setPageToken(pageToken);
            if (request.syncToken() != null) {
                req.setSyncToken(request.syncToken());
            } else {
                req.setSingleEvents(true);
            }
            if (request.pageSize() > 0) {
                req.setMaxResults(request.pageSize());
            }
            Events response = req.execute();
            if (response.getItems() != null) {
                for (var event : response.getItems()) {
                    if ("cancelled".equals(event.getStatus())) {
                        deletedIds.add(event.getId());
                    } else {
                        items.add(GoogleEventMapper.toCalendarEvent(event, calendarId));
                    }
                }
            }
            if (response.getNextSyncToken() != null) {
                syncToken = response.getNextSyncToken();
            }
            pageToken = response.getNextPageToken();
            if (pageToken == null) { break; }
            page++;
        }
        if (page >= MAX_PAGES) {
            LOG.warnf("listEventsSync hit MAX_PAGES (%d) for calendar '%s' — %d items, %d deletes accumulated",
                      MAX_PAGES, calendarId, items.size(), deletedIds.size());
        }
    } catch (GoogleJsonResponseException e) {
        if (e.getStatusCode() == 410) {
            throw new SyncTokenExpiredException(request.syncToken());
        }
        if (!items.isEmpty()) {
            LOG.warnf(e, "listEventsSync failed mid-pagination for calendar '%s' — returning %d partial items",
                      calendarId, items.size());
            return new SyncResult<>(Collections.unmodifiableList(items),
                    Collections.unmodifiableList(deletedIds), syncToken, false);
        }
        throw new RuntimeException("Google Calendar sync failed for calendar " + calendarId, e);
    } catch (IOException e) {
        if (!items.isEmpty()) {
            LOG.warnf(e, "listEventsSync failed mid-pagination for calendar '%s' — returning %d partial items",
                      calendarId, items.size());
            return new SyncResult<>(Collections.unmodifiableList(items),
                    Collections.unmodifiableList(deletedIds), syncToken, false);
        }
        throw new RuntimeException("Google Calendar sync failed for calendar " + calendarId, e);
    }
    return new SyncResult<>(Collections.unmodifiableList(items),
            Collections.unmodifiableList(deletedIds), syncToken, false);
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google -Dtest="GoogleCalendarPlatformTest" -DfailIfNoTests=false`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add calendar-google/
git commit -m "feat(#129): wire syncToken on GoogleCalendarPlatform.listEventsSync

Google Calendar events.list with syncToken/nextSyncToken for incremental sync.
Cancelled events mapped to deletedIds. HTTP 410 → SyncTokenExpiredException.
Mid-pagination failures return partial results per paginating-client-fail-soft protocol.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: Wire changes.list on GoogleDocumentPlatform

**Files:**
- Modify: `document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java:139-233`
- Test: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleDocumentPlatformTest.java`

**Interfaces:**
- Consumes: `DocumentPlatform.FileOperations.listSync(SyncRequest)` from Task 2
- Produces: Working Google Drive sync via `changes.list` with `startPageToken`/`newStartPageToken`

Google Drive sync uses `changes.list` (not `files.list`):
- Initial sync (null token): call `changes.getStartPageToken()` → list all files via `files.list` → return files + startPageToken
- Incremental sync (with token): call `changes.list(pageToken)` → map changed files + removed files → return with `newStartPageToken`
- HTTP 404 on the page token: treat as expired → throw `SyncTokenExpiredException`

- [ ] **Step 1: Write failing test — initial sync returns files with startPageToken**

Add to `GoogleDocumentPlatformTest.java`:

```java
@Test
void listSync_initialSync_returnsFilesWithToken() {
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes/startPageToken"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {"startPageToken": "page-token-1"}
                            """)));

    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/files"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {
                              "files": [
                                {
                                  "id": "f-1", "name": "Doc.txt",
                                  "mimeType": "text/plain", "size": 100,
                                  "parents": ["folder-1"],
                                  "createdTime": "2026-09-15T10:00:00.000Z",
                                  "modifiedTime": "2026-09-15T10:00:00.000Z"
                                }
                              ]
                            }
                            """)));

    var result = platform.files().listSync(SyncRequest.initial(100));

    assertThat(result.items()).hasSize(1);
    assertThat(result.items().getFirst().name()).isEqualTo("Doc.txt");
    assertThat(result.syncToken()).isEqualTo("page-token-1");
    assertThat(result.deletedIds()).isEmpty();
}
```

- [ ] **Step 2: Write failing test — incremental sync returns changes and deletes**

```java
@Test
void listSync_incrementalSync_returnsChangesAndDeletes() {
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes"))
            .withQueryParam("pageToken", WireMock.equalTo("page-token-1"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                            {
                              "changes": [
                                {
                                  "fileId": "f-2",
                                  "removed": false,
                                  "file": {
                                    "id": "f-2", "name": "NewDoc.txt",
                                    "mimeType": "text/plain", "size": 200,
                                    "parents": ["folder-1"],
                                    "createdTime": "2026-09-16T10:00:00.000Z",
                                    "modifiedTime": "2026-09-16T10:00:00.000Z"
                                  }
                                },
                                {
                                  "fileId": "f-old",
                                  "removed": true
                                }
                              ],
                              "newStartPageToken": "page-token-2"
                            }
                            """)));

    var result = platform.files().listSync(new SyncRequest("page-token-1", 100));

    assertThat(result.items()).hasSize(1);
    assertThat(result.items().getFirst().name()).isEqualTo("NewDoc.txt");
    assertThat(result.deletedIds()).containsExactly("f-old");
    assertThat(result.syncToken()).isEqualTo("page-token-2");
}
```

- [ ] **Step 3: Write failing test — expired token throws SyncTokenExpiredException**

```java
@Test
void listSync_expiredToken_throwsSyncTokenExpired() {
    wireMock.stubFor(get(urlPathEqualTo("/drive/v3/changes"))
            .withQueryParam("pageToken", WireMock.equalTo("expired"))
            .willReturn(aResponse().withStatus(404)));

    assertThatThrownBy(() -> platform.files().listSync(new SyncRequest("expired", 100)))
            .isInstanceOf(SyncTokenExpiredException.class);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -Dtest="GoogleDocumentPlatformTest" -DfailIfNoTests=false`
Expected: FAIL (stub throws UnsupportedOperationException)

- [ ] **Step 5: Implement `listSync` in GoogleDocumentPlatform.GoogleFileOperations**

Replace the stub:

```java
@Override
public SyncResult<DocumentSummary> listSync(SyncRequest request) {
    requireClient();
    if (request.syncToken() == null) {
        return initialSync(request);
    }
    return incrementalSync(request);
}

private SyncResult<DocumentSummary> initialSync(SyncRequest request) {
    try {
        var startToken = driveService.changes().getStartPageToken().execute();
        String token = startToken.getStartPageToken();

        List<DocumentSummary> items = new ArrayList<>();
        String pageToken = null;
        int page = 0;
        while (page < MAX_PAGES) {
            var req = driveService.files().list()
                    .setQ("trashed = false and mimeType != '" + FOLDER_MIME + "'")
                    .setFields("nextPageToken,files(" + FILE_FIELDS + ")")
                    .setPageSize(request.pageSize() > 0 ? request.pageSize() : 100);
            if (pageToken != null) {
                req.setPageToken(pageToken);
            }
            var response = req.execute();
            if (response.getFiles() != null) {
                response.getFiles().stream()
                        .map(GoogleDocumentPlatform::toSummary)
                        .forEach(items::add);
            }
            pageToken = response.getNextPageToken();
            if (pageToken == null) { break; }
            page++;
        }
        if (page >= MAX_PAGES) {
            LOG.warnf("listSync initial hit MAX_PAGES (%d) — %d items accumulated", MAX_PAGES, items.size());
        }
        return new SyncResult<>(Collections.unmodifiableList(items), List.of(), token, false);
    } catch (IOException e) {
        throw new RuntimeException("Google Drive initial sync failed", e);
    }
}

private SyncResult<DocumentSummary> incrementalSync(SyncRequest request) {
    List<DocumentSummary> items = new ArrayList<>();
    List<String> deletedIds = new ArrayList<>();
    String newToken = null;
    try {
        String pageToken = request.syncToken();
        int page = 0;
        while (page < MAX_PAGES) {
            var req = driveService.changes().list(pageToken)
                    .setFields("nextPageToken,newStartPageToken,changes(fileId,removed,file(" + FILE_FIELDS + "))")
                    .setPageSize(request.pageSize() > 0 ? request.pageSize() : 100);
            var response = req.execute();
            if (response.getChanges() != null) {
                for (var change : response.getChanges()) {
                    if (Boolean.TRUE.equals(change.getRemoved()) || change.getFile() == null) {
                        deletedIds.add(change.getFileId());
                    } else if (!FOLDER_MIME.equals(change.getFile().getMimeType())) {
                        items.add(toSummary(change.getFile()));
                    }
                }
            }
            if (response.getNewStartPageToken() != null) {
                newToken = response.getNewStartPageToken();
            }
            pageToken = response.getNextPageToken();
            if (pageToken == null) { break; }
            page++;
        }
        if (page >= MAX_PAGES) {
            LOG.warnf("listSync incremental hit MAX_PAGES (%d) — %d items, %d deletes accumulated",
                      MAX_PAGES, items.size(), deletedIds.size());
        }
    } catch (GoogleJsonResponseException e) {
        if (e.getStatusCode() == 404) {
            throw new SyncTokenExpiredException(request.syncToken());
        }
        if (!items.isEmpty()) {
            LOG.warnf(e, "listSync failed mid-pagination — returning %d partial items", items.size());
            return new SyncResult<>(Collections.unmodifiableList(items),
                    Collections.unmodifiableList(deletedIds), newToken, false);
        }
        throw new RuntimeException("Google Drive sync failed", e);
    } catch (IOException e) {
        if (!items.isEmpty()) {
            LOG.warnf(e, "listSync failed mid-pagination — returning %d partial items", items.size());
            return new SyncResult<>(Collections.unmodifiableList(items),
                    Collections.unmodifiableList(deletedIds), newToken, false);
        }
        throw new RuntimeException("Google Drive sync failed", e);
    }
    return new SyncResult<>(Collections.unmodifiableList(items),
            Collections.unmodifiableList(deletedIds), newToken, false);
}
```

Also add `MAX_PAGES` constant if not already present (it is on `GoogleCalendarPlatform` but check `GoogleDocumentPlatform` — it does not have one currently, so add it):

```java
private static final int MAX_PAGES = 20;
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -Dtest="GoogleDocumentPlatformTest" -DfailIfNoTests=false`
Expected: all PASS

- [ ] **Step 7: Full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add document-google/
git commit -m "feat(#129): wire changes.list on GoogleDocumentPlatform.listSync

Initial sync: getStartPageToken + full files.list. Incremental sync: changes.list
with change/remove mapping. HTTP 404 on expired token → SyncTokenExpiredException.
Mid-pagination failures return partial results per paginating-client-fail-soft protocol.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: GraphQL/MCP sync endpoints

### Task 5: Add sync endpoint to ConnectorCalendarApi

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorCalendarApi.java:29-142`
- Test: create `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorCalendarApiTest.java` (or add to existing test if one exists)

**Interfaces:**
- Consumes: `CalendarPlatform.listEventsSync(String, SyncRequest)` from Task 1
- Produces: REST endpoint `GET /api/connectors/calendar/events/sync`

- [ ] **Step 1: Write failing test — sync endpoint returns SyncResult**

```java
@Test
void syncEvents_returnsSyncResult() {
    // Mock CalendarPlatformService and CalendarPlatform
    // Call api.syncEvents("google", "primary", null, 100)
    // Assert result contains items and syncToken
}
```

(Use the pattern from `ConnectorContactsApiTest` — mock the platform service, verify the API method delegates correctly.)

- [ ] **Step 2: Add syncEvents endpoint to ConnectorCalendarApi**

```java
@PlatformQuery("Incremental sync of calendar events")
@RestPath("/events/sync")
public SyncResult<CalendarEventInfo> syncEvents(
        @QueryParam("platform") String platform,
        @QueryParam("calendarId") String calendarId,
        @QueryParam("syncToken") String syncToken,
        @QueryParam("pageSize") Integer pageSize) {
    CalendarPlatform p = calendarService.platform(platform);
    if (p == null) return new SyncResult<>(List.of(), List.of(), null, false);
    int size = pageSize != null ? pageSize : 100;
    var request = syncToken != null
        ? new SyncRequest(syncToken, size)
        : SyncRequest.initial(size);
    var result = p.listEventsSync(calendarId, request);
    var events = result.items().stream().map(this::toEventInfo).toList();
    return new SyncResult<>(events, result.deletedIds(), result.syncToken(), result.hasMore());
}
```

Add required imports:
```java
import io.casehub.connectors.SyncRequest;
import io.casehub.connectors.SyncResult;
```

- [ ] **Step 3: Run tests and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add graphql/
git commit -m "feat(#129): add calendar sync endpoint to ConnectorCalendarApi

GET /api/connectors/calendar/events/sync with syncToken/pageSize params.
Delegates to CalendarPlatform.listEventsSync, maps results to CalendarEventInfo.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 6: Add sync endpoint for DocumentPlatform

**Files:**
- Create or modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorDocumentApi.java` (check if it exists in ConnectorOperationsImpl — it may be dispatched differently)
- Test: corresponding test class

**Interfaces:**
- Consumes: `DocumentPlatform.FileOperations.listSync(SyncRequest)` from Task 2
- Produces: REST endpoint for document sync (location depends on existing document API pattern)

- [ ] **Step 1: Determine where document sync endpoint belongs**

Check `ConnectorOperationsImpl` for existing document-related methods. If documents are handled in `ConnectorOperationsImpl`, add a method there. If a separate `ConnectorDocumentApi` exists or should be created, follow the `ConnectorCalendarApi` pattern.

- [ ] **Step 2: Write failing test**

```java
@Test
void syncDocuments_returnsSyncResult() {
    // Mock DocumentPlatformService and DocumentPlatform
    // Call api.syncDocuments("google", null, 100)
    // Assert result contains items and syncToken
}
```

- [ ] **Step 3: Add syncDocuments endpoint**

```java
@PlatformQuery("Incremental sync of documents")
@RestPath("/documents/sync")
public SyncResult<DocumentSummary> syncDocuments(
        @QueryParam("platform") String platformId,
        @QueryParam("syncToken") String syncToken,
        @QueryParam("pageSize") Integer pageSize) {
    var p = documentService.platform(platformId);
    int size = pageSize != null ? pageSize : 100;
    var request = syncToken != null
        ? new SyncRequest(syncToken, size)
        : SyncRequest.initial(size);
    return p.files().listSync(request);
}
```

- [ ] **Step 4: Run tests and full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add graphql/
git commit -m "feat(#129): add document sync endpoint to GraphQL/MCP API

Incremental sync endpoint for DocumentPlatform.FileOperations.listSync.
Follows same SyncRequest/SyncResult pattern as contacts and calendar sync.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 5: Calendar user-scoping (additive)

### Task 7: Add user-scoped credential resolution to calendar-google

**Files:**
- Create: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCredentialResolver.java`
- Create: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleOAuthConfig.java`
- Modify: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java`
- Test: `calendar-google/src/test/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatformTest.java`

**Interfaces:**
- Consumes: `GoogleCredentialResolver` pattern from `contacts-google`
- Produces: `GoogleCalendarPlatform` that resolves credentials per-user via `GoogleCredentialResolver` CDI SPI

**Design notes:** This is a Google-provider-level change — the `CalendarPlatform` SPI itself does not change. The existing `GoogleCalendarPlatform` constructor signature changes from `(clientId, clientSecret, refreshToken)` to `(GoogleCredentialResolver)`. The internal test constructor `(Calendar calendarService)` stays for WireMock tests. Since no external consumer constructs `GoogleCalendarPlatform` directly (it's a CDI bean), this is a backward-compatible change within the module.

The `ConnectorCalendarApi` currently calls `calendarService.platform(platform)` → gets a `CalendarPlatform` → calls methods directly (no userId). After user-scoping, `GoogleCalendarPlatform` needs to know which user's credentials to use. Two approaches:

1. **CDI-injected SecurityIdentity** (like `ConnectorContactsApi` does): `GoogleCalendarPlatform` receives `SecurityIdentity` and resolves credentials on each API call
2. **Per-call wrapper**: `GoogleCalendarPlatform` exposes a `forUser(userId)` method that returns a lightweight wrapper configured for that user

Approach 1 is simpler and matches how `ConnectorContactsApi` works — it calls `p.contactRead(userId())`. But CalendarPlatform doesn't have user-scoped sub-interfaces, so the userId must flow differently.

For this task, the simplest additive change:
- `GoogleCalendarPlatform` takes a `GoogleCredentialResolver` and builds `Calendar` service per-call using a cached service per userId
- The SPI methods don't change — `GoogleCalendarPlatform` resolves the "current user" from CDI request context (`SecurityIdentity`)
- If no request context (e.g. background jobs), fall back to a default credential

- [ ] **Step 1: Create `GoogleCredentialResolver` and `GoogleOAuthConfig`**

```java
// GoogleCredentialResolver.java
package io.casehub.connectors.calendar.google;

public interface GoogleCredentialResolver {
    GoogleOAuthConfig resolve(String userId);
}
```

```java
// GoogleOAuthConfig.java
package io.casehub.connectors.calendar.google;

public record GoogleOAuthConfig(String refreshToken, String clientId, String clientSecret) {}
```

- [ ] **Step 2: Refactor GoogleCalendarPlatform to use GoogleCredentialResolver**

Replace the `clientId`/`clientSecret`/`refreshToken` fields with a `GoogleCredentialResolver` field. Change the constructor:

```java
public GoogleCalendarPlatform(GoogleCredentialResolver resolver) {
    this.resolver = resolver;
    try {
        this.transport = GoogleNetHttpTransport.newTrustedTransport();
    } catch (GeneralSecurityException | IOException e) {
        throw new RuntimeException("Failed to initialize HTTP transport", e);
    }
}
```

Build `Calendar` service per userId:

```java
private Calendar buildService(String userId) {
    var config = resolver.resolve(userId);
    var credentials = UserCredentials.newBuilder()
        .setClientId(config.clientId())
        .setClientSecret(config.clientSecret())
        .setRefreshToken(config.refreshToken())
        .build();
    return new Calendar.Builder(transport, GsonFactory.getDefaultInstance(),
            new HttpCredentialsAdapter(credentials))
        .setApplicationName("casehub-connectors")
        .build();
}
```

Inject `SecurityIdentity` via CDI for resolving current userId, or pass userId through the CalendarPlatformService layer.

- [ ] **Step 3: Update tests**

The WireMock test constructor `GoogleCalendarPlatform(Calendar calendarService)` stays unchanged — tests bypass credential resolution entirely. Add a test verifying the resolver-based constructor works:

```java
@Test
void resolverConstructor_callsResolverOnMethods() {
    // Verify GoogleCalendarPlatform uses resolver to build service
}
```

- [ ] **Step 4: Run tests and full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Update ConnectorCalendarApi to pass userId**

Inject `SecurityIdentity` into `ConnectorCalendarApi` (like `ConnectorContactsApi` does). Pass `identity.getPrincipal().getName()` through the platform layer.

This may require either:
- Adding a `CalendarPlatform.forUser(userId)` method (additive default on the interface)
- Or keeping the platform stateless and having `GoogleCalendarPlatform` use request-scoped CDI context

- [ ] **Step 6: Run tests and commit**

```bash
git add calendar-google/ graphql/
git commit -m "feat(#129): add GoogleCredentialResolver for per-user calendar credentials

Refactor GoogleCalendarPlatform from baked credentials to per-user credential
resolution via GoogleCredentialResolver CDI SPI. Aligns with contacts-google pattern.

Refs #129

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [CalendarPlatform.java] — calendar SPI (calendar-spi)
- [DocumentPlatform.java] — document SPI with FileOperations sub-interface (document-spi)
- [ContactsPlatform.java] — reference sync pattern (contacts-spi)
- [RefContactsPlatform.java:56-63] — ref sync implementation pattern
- [GoogleContactsPlatform.java:123-153] — Google sync implementation (People API)
- [InMemoryContactsBackend.java] — version tracking pattern for ref backends
- [SyncResult.java, SyncRequest.java, SyncTokenExpiredException.java] — sync primitives (connectors-api)
- [ConnectorCalendarApi.java] — existing calendar GraphQL/MCP API
- [ConnectorContactsApi.java:50-63] — contacts sync API endpoint pattern
- [GoogleCredentialResolver.java (contacts-google)] — per-user credential resolution pattern
- [paginating-client-fail-soft protocol] — mid-loop failure handling
- [GitHub #129] — focal issue
- [GitHub #125] — merged dependency (sync types)
