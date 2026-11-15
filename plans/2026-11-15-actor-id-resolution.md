# Cross-SPI actorId Resolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #223 — Cross-SPI: actorId resolution mechanism for per-user OAuth providers
**Issue group:** #223

**Goal:** Retrofit CalendarPlatform, DocumentPlatform, and EmailPlatform SPIs with user-scoped
capability accessors so per-user OAuth tokens can be resolved.

**Architecture:** Each flat SPI gets a user-scoped accessor (e.g. `calendarOps(String userId)`)
that returns a capability interface. Implementations build per-user API clients inside the
accessor. The MCP/GraphQL boundary (`ConnectorCalendarApi` etc.) injects `CurrentPrincipal`
and passes `actorId()` as userId.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, ServiceConnectionProvider

## Global Constraints

- Java source level 21, built on JVM 26
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All casehubio artifacts: `0.2-SNAPSHOT`
- SPI changes must preserve `@SimulationEligible` annotations
- `PlatformExtensible` must remain as super-interface on all three SPIs
- NoOp implementations use `@DefaultBean @ApplicationScoped`
- Use `ide-tooling` for all structural editing (never bash for .java files)

---

## Batch 1: CalendarPlatform — SPI + all implementations + callers

### Task 1: CalendarPlatform SPI, NoOp, and Ref

**Files:**
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/CalendarPlatform.java`
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/NoOpCalendarPlatform.java`
- Modify: `calendar-ref/src/main/java/io/casehub/connectors/calendar/ref/RefCalendarPlatform.java`
- Modify: `calendar-ref/src/test/java/io/casehub/connectors/calendar/ref/RefCalendarPlatformTest.java`
- Test: `calendar-spi/src/test/java/io/casehub/connectors/calendar/NoOpCalendarPlatformTest.java`

**Interfaces:**
- Produces: `CalendarPlatform.calendarOps(String userId)` returning `CalendarPlatform.CalendarOperations`
- Produces: `CalendarPlatform.CalendarOperations` with all existing flat methods

- [ ] **Step 1: Write failing test for new CalendarPlatform shape**

In `RefCalendarPlatformTest`, add a test that calls the new accessor:

```java
@Test
void calendarOpsReturnsOperations() {
    var ops = platform.calendarOps("user-1");
    assertThat(ops).isNotNull();
    assertThat(ops.listCalendars()).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-ref -Dtest=RefCalendarPlatformTest#calendarOpsReturnsOperations -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `calendarOps` method does not exist

- [ ] **Step 3: Restructure CalendarPlatform SPI**

Replace the interface body in `CalendarPlatform.java` using `ide_edit_member` (member = `CalendarPlatform`):

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

- [ ] **Step 4: Update NoOpCalendarPlatform**

Replace the class body to implement the new interface. Use `ide_edit_member`:

```java
@DefaultBean
@ApplicationScoped
public class NoOpCalendarPlatform implements CalendarPlatform {

    private static final CalendarOperations NOOP = new CalendarOperations() {
        @Override public List<CalendarInfo> listCalendars() { return List.of(); }
        @Override public List<CalendarEvent> listEvents(String c, Instant f, Instant t) { return List.of(); }
        @Override public CalendarEvent getEvent(String c, String e) { return null; }
        @Override public CalendarEvent createEvent(String c, EventDetails d) { return null; }
        @Override public CalendarEvent updateEvent(String c, String e, EventDetails d) { return null; }
        @Override public void deleteEvent(String c, String e) {}
        @Override public SyncResult<CalendarEvent> listEventsSync(String c, SyncRequest r) {
            return new SyncResult<>(List.of(), List.of(), null, false);
        }
    };

    @Override public String id() { return "noop"; }
    @Override public CalendarOperations calendarOps(String userId) { return NOOP; }
}
```

- [ ] **Step 5: Update RefCalendarPlatform**

Wrap existing logic in a `CalendarOperations` inner class. The Ref implementation
has in-memory state — `calendarOps(userId)` returns an instance that delegates to
the existing methods (ignoring userId since there's no OAuth):

```java
public class RefCalendarPlatform implements CalendarPlatform {
    // ... existing fields and constructor unchanged ...

    @Override
    public CalendarOperations calendarOps(String userId) {
        return new RefCalendarOperations();
    }

    private class RefCalendarOperations implements CalendarOperations {
        // move all existing method bodies here unchanged
    }
}
```

Use `ide_edit_member` with `member = RefCalendarPlatform` to restructure.

- [ ] **Step 6: Update RefCalendarPlatformTest**

All existing test calls like `platform.listCalendars()` become
`platform.calendarOps("test-user").listCalendars()`. Use `ide_structural_search_replace`
with pattern `platform.$method$($args$)` for each SPI method to add the accessor.

- [ ] **Step 7: Update NoOpCalendarPlatformTest**

Similarly update calls to use the new accessor.

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-spi,calendar-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add calendar-spi/ calendar-ref/
git commit -m "feat(#223): restructure CalendarPlatform SPI with user-scoped accessor"
```

### Task 2: Google and Microsoft Calendar implementations

**Files:**
- Modify: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java`
- Modify: `calendar-google/src/test/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatformTest.java`
- Modify: `calendar-microsoft/src/main/java/io/casehub/connectors/calendar/microsoft/MicrosoftCalendarPlatform.java`
- Modify: `calendar-microsoft/src/test/java/io/casehub/connectors/calendar/microsoft/MicrosoftCalendarPlatformTest.java`

**Interfaces:**
- Consumes: `CalendarPlatform.calendarOps(String userId)` → `CalendarOperations`
- Produces: Google and Microsoft implementations that thread userId to `ServiceConnectionProvider.getAccessToken()`

- [ ] **Step 1: Write failing test for GoogleCalendarPlatform actorId threading**

In `GoogleCalendarPlatformTest`, add a test that verifies actorId is passed through:

```java
@Test
void calendarOpsPassesActorIdToConnectionProvider() {
    var mockProvider = mock(ServiceConnectionProvider.class);
    when(mockProvider.getAccessToken("user-42", "google", "default"))
        .thenReturn(new ServiceAccessToken("token", Instant.now().plusSeconds(3600)));
    var platform = new GoogleCalendarPlatform(mockProvider);
    // The accessor should build a service using the provided userId
    var ops = platform.calendarOps("user-42");
    assertThat(ops).isNotNull();
    verify(mockProvider).getAccessToken("user-42", "google", "default");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google -Dtest=GoogleCalendarPlatformTest#calendarOpsPassesActorIdToConnectionProvider -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement GoogleCalendarPlatform with user-scoped accessor**

Restructure the class:
- Remove the `calendarService` field and `requireClient()` method
- `calendarOps(String userId)` calls `buildService(userId)` and returns a new
  `GoogleCalendarOperations` inner class wrapping the per-user `Calendar` client
- Move all existing method bodies into the inner class
- The inner class captures the `Calendar` service as a constructor parameter

```java
@Override
public CalendarOperations calendarOps(String userId) {
    return new GoogleCalendarOperations(buildService(userId));
}

private class GoogleCalendarOperations implements CalendarOperations {
    private final Calendar service;
    GoogleCalendarOperations(Calendar service) { this.service = service; }
    // existing method bodies moved here, using this.service instead of calendarService
}
```

- [ ] **Step 4: Run Google Calendar tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS (update existing tests to use `calendarOps("test-user")`)

- [ ] **Step 5: Write failing test for MicrosoftCalendarPlatform actorId threading**

```java
@Test
void calendarOpsPassesUserIdToResolveToken() {
    var mockProvider = mock(ServiceConnectionProvider.class);
    when(mockProvider.getAccessToken("user-42", "microsoft", "default"))
        .thenReturn(new ServiceAccessToken("token", Instant.now().plusSeconds(3600)));
    var platform = new MicrosoftCalendarPlatform(mockCalendarClient, mockProvider);
    var ops = platform.calendarOps("user-42");
    assertThat(ops).isNotNull();
}
```

- [ ] **Step 6: Implement MicrosoftCalendarPlatform with user-scoped accessor**

- Change `resolveToken()` to `resolveToken(String userId)` — passes userId instead of null
- `calendarOps(String userId)` returns a new inner class that captures userId
  and calls `resolveToken(userId)` for each operation

```java
@Override
public CalendarOperations calendarOps(String userId) {
    return new MicrosoftCalendarOperations(userId);
}

private class MicrosoftCalendarOperations implements CalendarOperations {
    private final String userId;
    MicrosoftCalendarOperations(String userId) { this.userId = userId; }
    // existing method bodies moved here, calling resolveToken(userId)
}

private String resolveToken(String userId) {
    if (connectionProvider == null) return "test-token";
    return connectionProvider.getAccessToken(userId, "microsoft", "default").accessToken();
}
```

- [ ] **Step 7: Run Microsoft Calendar tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-microsoft -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add calendar-google/ calendar-microsoft/
git commit -m "feat(#223): thread userId through Google and Microsoft Calendar implementations"
```

### Task 3: Calendar GraphQL/MCP callers

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorCalendarApi.java`
- Modify: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorCalendarApiTest.java`

**Interfaces:**
- Consumes: `CalendarPlatform.calendarOps(String userId)` → `CalendarOperations`
- Consumes: `CurrentPrincipal.actorId()` → `String`

- [ ] **Step 1: Write failing test for ConnectorCalendarApi userId threading**

Update `StubCalendarPlatform` in `ConnectorCalendarApiTest` to implement the new interface,
and add a test that verifies `CurrentPrincipal` is used:

```java
@Test
void listCalendarsUsesCurrentPrincipal() {
    var api = apiWith(fullPlatform());
    // inject a mock CurrentPrincipal
    api.currentPrincipal = () -> "test-actor";
    var result = api.listCalendars("test");
    assertThat(result).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorCalendarApiTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL

- [ ] **Step 3: Update ConnectorCalendarApi**

Add `@Inject CurrentPrincipal currentPrincipal;` field.
Change every method from `p.listCalendars()` to `p.calendarOps(currentPrincipal.actorId()).listCalendars()`.

Example for `listCalendars`:
```java
public List<CalendarInfo> listCalendars(@QueryParam("platform") String platform) {
    CalendarPlatform p = calendarService.platform(platform);
    if (p == null) return List.of();
    return p.calendarOps(currentPrincipal.actorId()).listCalendars();
}
```

Apply the same pattern to all 7 methods: `listCalendars`, `listEvents`, `getEvent`,
`createEvent`, `updateEvent`, `deleteEvent`, `syncEvents`.

- [ ] **Step 4: Update StubCalendarPlatform in test**

Implement `calendarOps(String userId)` returning a stub that delegates to the
existing method implementations:

```java
static class StubCalendarPlatform implements CalendarPlatform {
    // ... existing fields ...
    @Override public CalendarOperations calendarOps(String userId) {
        return new CalendarOperations() {
            // move existing stub methods here
        };
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorCalendarApiTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add graphql/
git commit -m "feat(#223): thread currentPrincipal.actorId() through calendar API callers"
```

---

## Batch 2: DocumentPlatform — SPI + all implementations + callers

### Task 4: DocumentPlatform SPI, NoOp, and Ref

**Files:**
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java`
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/NoOpDocumentPlatform.java`
- Modify: `document-ref/src/main/java/io/casehub/connectors/document/ref/RefDocumentPlatform.java`
- Modify: `document-ref/src/test/java/io/casehub/connectors/document/ref/RefDocumentPlatformTest.java`

**Interfaces:**
- Produces: `DocumentPlatform.files(String userId)`, `folders(String userId)`,
  `search(String userId)`, `sharing(String userId)` — same return types as before

- [ ] **Step 1: Write failing test for new DocumentPlatform accessor signatures**

```java
@Test
void filesAcceptsUserId() {
    var ops = platform.files("user-1");
    assertThat(ops).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-ref -Dtest=RefDocumentPlatformTest#filesAcceptsUserId -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL

- [ ] **Step 3: Add userId parameter to DocumentPlatform accessors**

In `DocumentPlatform.java`, change:
- `FileOperations files();` → `FileOperations files(String userId);`
- `FolderOperations folders();` → `FolderOperations folders(String userId);`
- `SearchOperations search();` → `SearchOperations search(String userId);`
- `SharingOperations sharing();` → `SharingOperations sharing(String userId);`

Use `ide_edit_member` for each method.

- [ ] **Step 4: Update NoOpDocumentPlatform**

Add `String userId` parameter to each accessor. The NoOp returns are unchanged.

- [ ] **Step 5: Update RefDocumentPlatform**

Add `String userId` parameter to each accessor (ignored — in-memory impl).

- [ ] **Step 6: Update RefDocumentPlatformTest**

All calls to `platform.files()` become `platform.files("test-user")`, etc.

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-spi,document-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add document-spi/ document-ref/
git commit -m "feat(#223): add userId parameter to DocumentPlatform capability accessors"
```

### Task 5: Google and Microsoft Document implementations

**Files:**
- Modify: `document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java`
- Modify: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleDocumentPlatformTest.java`
- Modify: `document-microsoft/src/main/java/io/casehub/connectors/document/microsoft/MicrosoftDrivePlatform.java`
- Modify: `document-microsoft/src/test/java/io/casehub/connectors/document/microsoft/MicrosoftDrivePlatformTest.java`

**Interfaces:**
- Consumes: `DocumentPlatform.files(String userId)` etc.

- [ ] **Step 1: Write failing test for GoogleDocumentPlatform actorId threading**

```java
@Test
void filesPassesActorIdToConnectionProvider() {
    var mockProvider = mock(ServiceConnectionProvider.class);
    when(mockProvider.getAccessToken("user-42", "google", "default"))
        .thenReturn(new ServiceAccessToken("token", Instant.now().plusSeconds(3600)));
    var platform = new GoogleDocumentPlatform(mockProvider);
    var ops = platform.files("user-42");
    assertThat(ops).isNotNull();
    verify(mockProvider).getAccessToken("user-42", "google", "default");
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `files()` doesn't accept userId

- [ ] **Step 3: Implement GoogleDocumentPlatform with per-user client construction**

Key changes:
- Remove singleton `driveService`, `fileOps`, `folderOps`, `searchOps`, `sharingOps` fields
- Remove `requireClient()`
- Add `buildService(String actorId)` method (same pattern as GoogleCalendarPlatform)
- Each accessor creates a fresh inner class with the per-user Drive client:

```java
@Override
public FileOperations files(String userId) {
    return new GoogleFileOperations(buildService(userId));
}
@Override
public FolderOperations folders(String userId) {
    return new GoogleFolderOperations(buildService(userId));
}
// ... same for search and sharing

private Drive buildService(String actorId) {
    if (overrideService != null) return overrideService;
    var token = connectionProvider.getAccessToken(actorId, "google", DEFAULT_TENANCY);
    var credentials = GoogleCredentials.create(
        new AccessToken(token.accessToken(), Date.from(token.expiresAt())));
    return new Drive.Builder(transport, GsonFactory.getDefaultInstance(),
            new HttpCredentialsAdapter(credentials))
        .setApplicationName("casehub-connectors")
        .build();
}
```

Each inner class takes `Drive service` as a constructor parameter instead of
referencing the enclosing `driveService` field.

- [ ] **Step 4: Run Google Document tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-google -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS (update existing tests to use `files("test-user")` etc.)

- [ ] **Step 5: Implement MicrosoftDrivePlatform with userId**

Change `resolveToken()` to `resolveToken(String userId)`.
Add `String userId` to each accessor. Each inner class captures userId:

```java
@Override
public FileOperations files(String userId) {
    requireConfigured();
    return new MicrosoftFileOperations(userId);
}

private class MicrosoftFileOperations implements FileOperations {
    private final String userId;
    MicrosoftFileOperations(String userId) { this.userId = userId; }
    // existing methods unchanged, calling resolveToken(userId)
}

private String resolveToken(String userId) {
    if (connectionProvider == null) return "test-token";
    return connectionProvider.getAccessToken(userId, "microsoft", "default").accessToken();
}
```

- [ ] **Step 6: Run Microsoft Document tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-microsoft -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add document-google/ document-microsoft/
git commit -m "feat(#223): thread userId through Google and Microsoft Document implementations"
```

### Task 6: Document GraphQL/MCP callers

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorDocumentApi.java`
- Modify: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorDocumentApiTest.java`
- Modify: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorOperationsImplTest.java`

**Interfaces:**
- Consumes: `DocumentPlatform.files(String userId)` etc.
- Consumes: `CurrentPrincipal.actorId()`

- [ ] **Step 1: Update ConnectorDocumentApi**

Add `@Inject CurrentPrincipal currentPrincipal;` field.
All calls: `platform.files()` → `platform.files(currentPrincipal.actorId())`, etc.

- [ ] **Step 2: Update test stubs**

Update `StubDocumentPlatform` in `ConnectorOperationsImplTest` to accept userId on
accessors. Update `ConnectorDocumentApiTest` similarly.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest="ConnectorDocumentApiTest,ConnectorOperationsImplTest" -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add graphql/
git commit -m "feat(#223): thread currentPrincipal.actorId() through document API callers"
```

---

## Batch 3: EmailPlatform — SPI + all implementations + callers

### Task 7: EmailPlatform SPI, NoOp, and Ref

**Files:**
- Modify: `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatform.java`
- Modify: `email-spi/src/main/java/io/casehub/connectors/email/spi/NoOpEmailPlatform.java`
- Modify: `email-ref/src/main/java/io/casehub/connectors/email/ref/RefEmailPlatform.java`
- Modify: `email-ref/src/test/java/io/casehub/connectors/email/ref/RefEmailPlatformTest.java`

**Interfaces:**
- Produces: `EmailPlatform.emailOps(String userId)` → `EmailPlatform.EmailOperations`

- [ ] **Step 1: Write failing test**

```java
@Test
void emailOpsReturnsOperations() {
    var ops = platform.emailOps("user-1");
    assertThat(ops).isNotNull();
    assertThat(ops.listMailboxes()).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Restructure EmailPlatform SPI**

Same pattern as CalendarPlatform — extract an `EmailOperations` inner interface:

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

- [ ] **Step 4: Update NoOpEmailPlatform**

Same pattern as NoOpCalendarPlatform.

- [ ] **Step 5: Update RefEmailPlatform**

Wrap existing methods in an inner class returned by `emailOps(userId)`.

- [ ] **Step 6: Update RefEmailPlatformTest**

All calls through `platform.emailOps("test-user").method(...)`.

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-spi,email-ref -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add email-spi/ email-ref/
git commit -m "feat(#223): restructure EmailPlatform SPI with user-scoped accessor"
```

### Task 8: Google and Microsoft Email implementations

**Files:**
- Modify: `email-google/src/main/java/io/casehub/connectors/email/google/GoogleEmailPlatform.java`
- Modify: `email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailPlatformTest.java`
- Modify: `email-microsoft/src/main/java/io/casehub/connectors/email/microsoft/MicrosoftEmailPlatform.java`
- Modify: `email-microsoft/src/test/java/io/casehub/connectors/email/microsoft/MicrosoftEmailPlatformTest.java`

**Interfaces:**
- Consumes: `EmailPlatform.emailOps(String userId)` → `EmailOperations`

- [ ] **Step 1: Write failing test for GoogleEmailPlatform**

```java
@Test
void emailOpsPassesActorIdToConnectionProvider() {
    var mockProvider = mock(ServiceConnectionProvider.class);
    when(mockProvider.getAccessToken("user-42", "google", "default"))
        .thenReturn(new ServiceAccessToken("token", Instant.now().plusSeconds(3600)));
    var platform = new GoogleEmailPlatform(mockProvider);
    var ops = platform.emailOps("user-42");
    assertThat(ops).isNotNull();
    verify(mockProvider).getAccessToken("user-42", "google", "default");
}
```

- [ ] **Step 2: Implement GoogleEmailPlatform with user-scoped accessor**

Same pattern as GoogleCalendarPlatform:
- Remove `gmailService` field and `requireClient()`
- Add `buildService(String actorId)` that creates per-user Gmail client
- `emailOps(userId)` returns `GoogleEmailOperations` inner class with per-user client

- [ ] **Step 3: Run Google Email tests**

Expected: PASS

- [ ] **Step 4: Implement MicrosoftEmailPlatform with user-scoped accessor**

- Change `resolveToken()` to `resolveToken(String userId)`
- `emailOps(userId)` returns `MicrosoftEmailOperations` inner class capturing userId

- [ ] **Step 5: Run Microsoft Email tests**

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add email-google/ email-microsoft/
git commit -m "feat(#223): thread userId through Google and Microsoft Email implementations"
```

### Task 9: Email GraphQL/MCP callers

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorEmailApi.java`
- Modify: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorEmailApiTest.java`

**Interfaces:**
- Consumes: `EmailPlatform.emailOps(String userId)` → `EmailOperations`
- Consumes: `CurrentPrincipal.actorId()`

- [ ] **Step 1: Update ConnectorEmailApi**

Add `@Inject CurrentPrincipal currentPrincipal;` field.
All calls: `p.listMailboxes()` → `p.emailOps(currentPrincipal.actorId()).listMailboxes()`, etc.

- [ ] **Step 2: Update StubEmailPlatform in test**

Implement `emailOps(String userId)` returning a stub.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorEmailApiTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add graphql/
git commit -m "feat(#223): thread currentPrincipal.actorId() through email API callers"
```

---

## Batch 4: Full build verification and docs

### Task 10: Full build + ConnectorOperationsImpl + guide updates

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java` (if needed for connectorsReport)
- Modify: `docs/guides/consumer-guide.md`
- Modify: `docs/guides/contributor-guide.md`

- [ ] **Step 1: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

Fix any compilation failures in modules not yet touched (playbooks, integration-tests,
connectors-spring, etc.) — these will be callers that use the old flat API.

- [ ] **Step 2: Check ConnectorOperationsImpl.connectorsReport**

Read `ConnectorOperationsImpl.connectorsReport()` (lines 198-345). If it calls
platform methods directly (not just `ids()` and `supports()`), update to use
`calendarOps`/`emailOps`/`files` with a system actor ID or skip user-context-dependent
operations in the report.

- [ ] **Step 3: Update consumer guide**

In `docs/guides/consumer-guide.md`, update the CalendarPlatform, DocumentPlatform,
and EmailPlatform API examples to show the new user-scoped accessor pattern.

- [ ] **Step 4: Update contributor guide**

In `docs/guides/contributor-guide.md`, document the user-scoped accessor pattern
as the standard for per-user OAuth SPIs.

- [ ] **Step 5: Final full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS with all tests passing

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat(#223): full build green, update consumer and contributor guides"
```

## References

- [2026-11-15-actor-id-resolution-design.md] — design spec this plan implements
- CalendarPlatform.java:15 — current flat SPI interface
- DocumentPlatform.java:18 — current flat-accessor SPI interface
- EmailPlatform.java:15 — current flat SPI interface
- ContactsPlatform.java:17 — established user-scoped accessor pattern (reference implementation)
- GoogleCalendarPlatform.java:61 — existing buildService(actorId) pattern
- GoogleDocumentPlatform.java:40 — existing inner class structure
- MicrosoftCalendarPlatform.java:100 — resolveToken() passing null actorId
- ConnectorCalendarApi.java:31 — primary caller injecting CalendarPlatformService
- ConnectorDocumentApi.java:32 — primary caller injecting DocumentPlatformService
- ConnectorEmailApi.java:26 — primary caller injecting EmailPlatformService
- CurrentPrincipal.java:24 — request-scoped identity interface
- GitHub casehubio/connectors#223
